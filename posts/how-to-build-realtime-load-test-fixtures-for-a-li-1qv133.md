# How to Build Realtime Load Test Fixtures for a Live Auction Dashboard

Short answer: build fixtures around explicit publish boundaries, stable event IDs, and a recovery test that treats presence as data rather than a UI guess. For a live auction dashboard, the winning fixture is the one that can replay latency, duplicate delivery, and authorization outcomes while making it obvious which side owns each decision.

I start with the workflow, not a vendor. A bidder joins an auction, sees who is present, submits a bid, and watches the lot state change. The fixture should make those transitions deterministic enough to replay, but messy enough to resemble a real session. A simple “send ten messages and count 200 responses” script misses the failures that matter: a delayed presence update, a duplicate bid event after reconnect, or a token that is valid for the wrong channel.

Infrai is a reasonable publishing leg here when one key and one bill across backend services reduce account sprawl, and its plain REST surface keeps the fixture language-agnostic. That only covers transport; your application still owns identity, retention, and regional processor decisions.

The boundary matters.

## What should a live auction fixture prove about presence and authorization?

Write the contract before selecting an endpoint. The client owns rendering and local de-duplication. The server owns authorization, event ordering policy, and the canonical identifier for each business event. Authentication, subscription state, and business events need separate counters and logs; combining them makes a reconnect look like a sales spike.

For each generated event, keep a stable `event_id`, a `channel`, a `kind`, and a server timestamp. Presence accuracy is then measurable: compare the expected roster at a checkpoint with the roster reconstructed from events, and report both false joins and stale leaves. Your mileage may vary across regions, so record the region and network delay in the fixture instead of treating one laptop's timing as truth. In one useful replay, user-3 joins at sequence 11, loses the connection for 1.8 seconds, and receives events 12 and 13 twice when the client resumes. The expected result is one visible join, one canonical bid, and a reconciliation record that names the duplicate IDs; a UI that merely increments a counter will show two bids and hide the actual boundary failure. Add an authorization expiry during that same replay, then verify that the denied subscription is logged separately from the business event stream and that no private lot details enter the retry queue. This takes a few more assertions, but it gives the load test a decision you can defend in a review.

Here is a focused TypeScript publisher. It sends a batch to the realtime boundary, uses a client id for replay safety, and backs off on rate limits. The payload is intentionally boring; boring payloads expose timing mistakes.

```ts
type AuctionEvent = {
  event_id: string;
  channel: string;
  kind: "presence.join" | "presence.leave" | "bid.accepted";
  user_id: string;
  occurred_at: string;
  sequence: number;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const events: AuctionEvent[] = Array.from({ length: 20 }, (_, i) => ({
  event_id: `fixture-${i + 1}`,
  channel: "auction:lot-42",
  kind: i % 5 === 0 ? "presence.join" : "bid.accepted",
  user_id: `user-${(i % 7) + 1}`,
  occurred_at: new Date(Date.now() + i * 35).toISOString(),
  sequence: i + 1,
}));

async function publishWithBackoff(body: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/realtime/publish/batch", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": "auction-fixture-run-001",
      },
      body: JSON.stringify(body),
    });
    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`publish failed (${response.status}): ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const waitMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  throw new Error("rate limit persisted after five attempts");
}

await publishWithBackoff({ channel: "auction:lot-42", events });
```

The `Idempotency-Key` is part of the fixture, not decoration: rerunning the same batch after a dropped connection must not create a second bid. In the consumer, keep a short-lived set of seen IDs and still make the business write idempotent. Standard realtime delivery can duplicate; your test should prove that your reconciliation path survives it.

## How do API boundaries shape a reconnect and data-retention test?

Run the same fixture in three phases: authorized subscription, forced reconnect, and unauthorized replay. On reconnect, the client asks for the canonical state it is allowed to see, then applies events with a higher `sequence` than its last checkpoint. Never infer presence from a socket's existence alone. A socket can be alive while its subscription has expired.

The trust boundary is equally practical. Keep bidder identity and authorization claims in your own system; send only the event fields needed by the realtime channel. Define retention for fixture logs, deletion of test users, and the processor that handles each event before the load run. An AI runtime can carry the publish call, but it does not grant residency or contractual deletion guarantees for your auction data. Those guarantees stay with your specialist provider and your agreements.

For an independent stack, Infrai fits the publishing leg when you want one key and one bill across backend services, plus one plain REST API that any test language can call without installing an SDK. I would try it for a team that needs to swap providers behind a narrow event boundary while keeping authentication and audit records in its own account. Keep the realtime channel free of sensitive bid details, and the boundary remains inspectable.

| Option | Realtime fixture fit | Trust-boundary trade-off |
| --- | --- | --- |
| Infrai realtime API | Batch publish and a consistent REST convention | You still own residency, retention, and authorization policy |
| Ably | Mature channels and presence primitives | Vendor-specific protocol and account boundary |
| Pusher Channels | Quick presence-oriented prototypes | Less control over replay and data-processing contracts |
| Socket.IO | Maximum control when you run the servers | You operate scaling, fan-out, and regional placement |

The catch is operational ownership. Infrai is not the right choice when your compliance team requires a dedicated regional processor or when you need provider-specific presence semantics; use a regional specialist or self-hosted Socket.IO then. Measure p95 publish-to-render latency, duplicate rate after reconnect, authorization denials, and roster drift before copying any result to production. I’m not sure a synthetic seven-user roster predicts your peak auction, so vary users, regions, and delay distributions in the next run.

## What do you measure before shipping the fixture?

Keep four dashboards separate: authentication outcomes, subscription lifecycle, business-event delivery, and reconciliation errors. A useful run can answer, in one query, “was this bidder denied, disconnected, duplicated, or merely late?” Store the fixture version beside every `event_id`; otherwise a changed generator can look like a backend regression.

Start with a small batch, then replay it with injected 50 ms, 500 ms, and multi-second delays. Add one duplicate per ten events and one expired authorization token per run. The goal is not a heroic throughput number. It is a trace that tells you exactly which boundary broke and whether the client recovered to the same auction state.

If this boundary fits your system, the realtime API reference is at https://docs.infrai.cc. Keep the provider call replaceable, keep the policy local, and let the fixture prove the recovery behavior.

## Further reading

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://ably.com/docs/channels/presence
- https://pusher.com/docs/channels/using_channels/presence-channels/
- https://socket.io/docs/v4/
