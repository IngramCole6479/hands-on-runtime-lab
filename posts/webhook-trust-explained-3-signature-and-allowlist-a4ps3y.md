# Webhook Trust Explained: 3 Signature and Allowlisting Controls for Platform Intake

**Short answer:** For inbound marketplace platform events, use webhook signature verification as the identity check, treat IP allowlisting as an optional early filter, and accept an event only after it reaches durable storage. That combination survives sender address changes without turning an outage into silent data loss.

The least complex useful path is `receive -> limit -> verify -> deduplicate -> persist -> acknowledge -> process`. Signature verification answers who could have created the message and whether its bytes changed. An IP rule answers where the connection appeared to originate. Those are different questions, so forcing one control to do both jobs creates brittle failure modes.

One constraint drives the rest of the design: a marketplace can cap intake spend or refuse more traffic, but it can't promise unlimited acceptance during an unbounded outage. Pick that boundary deliberately.

## A minimal signed intake path

Assume a marketplace receives `listing.updated` and `order.created` events from an account platform. The edge applies a coarse request-size and rate limit, then the application verifies a signature over the timestamp and exact raw request bytes. A replay key is claimed atomically, the verified envelope is appended to durable storage, and only then does the endpoint return success. Workers can be unavailable for hours without changing the authentication path.

Keep the raw bytes.

Parsing JSON before verification can change insignificant-looking bytes such as whitespace or key order. The following TypeScript defines its own header contract rather than pretending every sender uses the same one: `x-market-signature` contains `t=<unix-seconds>,v1=<hex-hmac>`, the signed value is `<timestamp>.<raw-body>`, and the receiver allows a configured clock window. The replay store must implement an atomic claim, not a read followed by a write.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

type ReplayStore = {
  claim(key: string, ttlSeconds: number): Promise<boolean>;
};

type VerifiedEvent = {
  id: string;
  type: "listing.updated" | "order.created";
  accountId: string;
  occurredAt: string;
};

function verifySignature(
  rawBody: Buffer,
  header: string,
  secret: string,
  nowSeconds: number,
  toleranceSeconds: number,
): { timestamp: number; digest: string } {
  const fields = new Map(
    header.split(",").map((part) => {
      const [key, ...rest] = part.trim().split("=");
      return [key, rest.join("=")];
    }),
  );
  const timestamp = Number(fields.get("t"));
  const suppliedHex = fields.get("v1") ?? "";

  if (!Number.isSafeInteger(timestamp)) throw new Error("invalid timestamp");
  if (Math.abs(nowSeconds - timestamp) > toleranceSeconds) {
    throw new Error("timestamp outside acceptance window");
  }
  if (!/^[a-f0-9]{64}$/i.test(suppliedHex)) throw new Error("invalid signature");

  const signed = Buffer.concat([Buffer.from(`${timestamp}.`), rawBody]);
  const expected = createHmac("sha256", secret).update(signed).digest();
  const supplied = Buffer.from(suppliedHex, "hex");

  if (supplied.length !== expected.length || !timingSafeEqual(supplied, expected)) {
    throw new Error("invalid signature");
  }
  return { timestamp, digest: suppliedHex.toLowerCase() };
}

export async function authenticateEvent(
  rawBody: Buffer,
  signatureHeader: string,
  secret: string,
  replayStore: ReplayStore,
  nowSeconds = Math.floor(Date.now() / 1000),
): Promise<VerifiedEvent> {
  const verified = verifySignature(rawBody, signatureHeader, secret, nowSeconds, 300);
  const replayKey = `${verified.timestamp}:${verified.digest}`;
  const firstDelivery = await replayStore.claim(replayKey, 300);
  if (!firstDelivery) throw new Error("duplicate signed delivery");

  const event: unknown = JSON.parse(rawBody.toString("utf8"));
  if (!isVerifiedEvent(event)) throw new Error("invalid event schema");
  return event;
}

function isVerifiedEvent(value: unknown): value is VerifiedEvent {
  if (typeof value !== "object" || value === null) return false;
  const event = value as Record<string, unknown>;
  return (
    typeof event.id === "string" &&
    (event.type === "listing.updated" || event.type === "order.created") &&
    typeof event.accountId === "string" &&
    typeof event.occurredAt === "string"
  );
}
```

The `300`-second window is an example policy, not a universal constant. I'm not sure a fixed five-minute window fits every marketplace; clock quality, retry behavior, and the acceptable replay surface determine the real value. Record rejections by reason, then adjust from evidence. Don't expand the window merely to hide clock drift.

Secret handling deserves its own lifecycle — generation, restricted access, rotation, revocation, and audit — because an HMAC check is only as meaningful as control of the shared secret. The OWASP Secrets Management Cheat Sheet is a useful baseline for that operational work. During rotation, accept the current and next secret for a bounded overlap, identify which one verified the request, and remove the old one when the sender transition is complete.

## Should inbound platform events use webhook signature verification or IP allowlisting?

Use signatures for message authenticity and integrity. Add an allowlist only when the sender publishes a dependable address contract and your edge can update it without a risky manual deployment. If only one control is feasible, signatures usually preserve the more important property: possession of the signing secret travels with the message, while a source address does not prove who formed its body.

| Decision point | Signature verification | IP allowlisting |
|---|---|---|
| Question answered | Was this exact payload signed by a secret holder? | Did the connection arrive from an approved address? |
| Change dependency | Secret rotation | Network range changes |
| Replay handling | Needs timestamp plus atomic deduplication | Does not identify a repeated payload |
| Earliest rejection point | Application or capable edge | Network edge |
| Main operating risk | Mishandled raw bytes or secret lifecycle | Stale ranges refusing legitimate deliveries |

Neither control validates the business meaning of an event. After authentication, check the schema, event type, account scope, identifier lengths, and state transition before any side effect. A correctly signed `order.created` payload can still be unacceptable for an unknown account or duplicate order ID. Authentication grants entry to validation; it doesn't grant permission to mutate everything.

There is a catch on each side. Signature verification is not suitable when the sender provides no stable signing contract or there is no safe way to distribute and rotate a secret; in that case, use a private network path or a tightly managed allowlist while you establish a stronger message-level control. Stick with IP allowlisting as the primary gate when both parties control fixed network boundaries and rejecting traffic after an unannounced address change is an accepted business risk. Conversely, skip the allowlist when senders use changing shared egress ranges and rapid range updates would create more refused traffic than the filter prevents.

Use both when defense in depth is worth the operational coupling. The order matters: reject obviously disallowed sources and oversized bodies at the edge, verify the signed raw body, atomically claim the replay key, validate the event, and append it. An allowlist must never become a reason to omit signature checking for internet-delivered messages.

## Spend ceilings versus refused traffic

Outages turn a security choice into a capacity choice. Suppose workers stop but the intake edge and durable append remain healthy. Verified events can queue, and recovery is governed by backlog depth and drain rate. If durable storage is unavailable, returning success would create silent loss; refuse the delivery so a sender with retries can try again. This is blunt, but honest.

A solo operator needs a hard admission policy before that day arrives. Set maximum body size, per-sender rate limits, total accepted bytes, replay-store capacity, durable backlog capacity, and a maximum event age. These limits should be observable and configurable without editing authentication code. Once the storage ceiling is near, reserve capacity for high-value event classes only if the product contract explicitly permits prioritization; otherwise apply the same refusal rule consistently.

The economic distinction is small but useful. IP filtering can discard unwanted connections before signature computation and storage, so it may reduce work at the edge. Signature verification costs compute and requires secret operations, while accepted traffic consumes replay and queue capacity. Yet optimizing away one HMAC operation while allowing unbounded payloads or an unbounded durable backlog aims at the cheap part of the system. Body limits and admission control define the spend ceiling. Identity controls decide which requests may consume it.

Refused traffic needs a budget too. Track refused requests by cause: source filter, body size, malformed signature header, timestamp window, signature mismatch, replay, schema, rate limit, or capacity. Don't collapse them into one authentication counter. A sudden rise in source-filter refusals suggests a network contract change; a rise in old timestamps points toward clock or delivery delay; capacity refusals say the outage plan has reached its declared edge. Those interpretations are operational hypotheses, not proof, so correlate them with sender notices and internal saturation signals.

Small systems benefit from a narrow interface between intake and processing. The receiver writes an immutable envelope containing the verified event, receipt time, authentication key version, and processing status. Workers consume by event ID and make side effects idempotent. This separation lets the public endpoint stay available while search indexing, notifications, or model-backed enrichment is paused to protect a spending limit. It also avoids re-verifying transformed JSON later, when the original signed bytes may no longer exist.

Stop accepting before the bill or disk is unbounded.

## Run the outage drill before launch

Test the contract with generated keys and synthetic events in an isolated environment. Cover one valid request, a one-byte body change, malformed hexadecimal input, an expired timestamp, a future timestamp, a duplicate delivery, an unknown event type, an oversized body, and both secrets during rotation. Then stop the worker, keep intake running, fill the queue to its warning threshold, and measure whether processing resumes without duplicate side effects. The test is about state transitions, not a pretty dashboard.

Deployment should separate edge policy from application policy. Roll out a new sender address range before enforcing it, but don't leave temporary ranges without an expiry owner. Roll secrets with a bounded two-key verification period. Deploy schema readers before a sender begins emitting a new event variant. If your mileage may vary anywhere, it will be the sender's retry schedule: confirm it directly, because an outage design that assumes retries without a contract is merely hopeful.

The operational checklist is short enough to keep in prose. Confirm that alerts distinguish rejected, queued, and processed events; the queue reports age as well as count; every accepted response follows a durable append; replay claims are atomic; workers use event IDs for idempotency; secrets can rotate without downtime; address changes have an owner; storage limits trigger before exhaustion; and the recovery rate is greater than the expected arrival rate. Rehearse key revocation and queue saturation separately — they fail for different reasons and need different decisions.

The final decision is deliberately unglamorous: signatures establish trust in the payload, allowlists reduce the network surface when their maintenance contract is credible, and durable admission control determines whether the marketplace spends more or refuses traffic during an outage. Write those three policies independently. Then test their intersections.

## References

- OWASP, Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html

## Further reading

The OWASP checklist above is the source to keep beside the secret generation, storage, rotation, revocation, and audit design.
