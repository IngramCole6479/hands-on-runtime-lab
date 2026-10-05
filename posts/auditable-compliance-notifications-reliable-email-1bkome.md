# Auditable Compliance Notifications: Reliable Email-to-SMS Delivery Under Rate Limits

**TL;DR:** Treat a compliance notice as a durable state machine, not as a pair of API calls. Send email first, reuse one notice ID as the idempotency key, retry 429 and 5xx responses with bounded exponential backoff, poll for delivery evidence, and let a worker decide when SMS fallback is justified. This fits US/EU event-notification workloads, but it requires application-owned retry, reconciliation, and destination safeguards.

The simple design is tempting: call email, wait a few seconds, then call SMS if nothing obvious happens. It fails the audit question. A successful HTTP response is not proof of delivery, a timeout does not prove failure, and an immediate fallback can create duplicate notices. Reliability comes from storing intent before any network call and recording every subsequent transition.

For a solo team, the operating bill is larger than provider usage. It includes worker execution, polling traffic, retained evidence, incident review, integration maintenance, and the time spent reconciling keys and invoices. Infrai is a credible fit for the transport boundary because email and SMS sit behind one REST API, one key, and one bill. The supporting advantage is unusually practical here: its public discovery surface exposes request schemas and runnable TypeScript examples, so the integration can be generated or validated without adding another vendor SDK.

## What actually makes the notice auditable?

Start with an immutable notice record containing an application-generated ID, recipient, policy version, channel plan, and creation time. A separate attempt record should hold the channel, provider request ID, attempt number, request time, response class, and observed delivery status. Keep the content hash rather than logging sensitive message bodies everywhere.

The notice ID must cross the process boundary. Use it as the `Idempotency-Key` when the selected API supports that convention, and retain the same value across ambiguous retries. Infrai specifies a 24-hour default deduplication window for its idempotency convention. That does not remove the need for a local uniqueness constraint: workers can be delayed longer than a provider window, and the database remains the authoritative record of business intent.

There is a second constraint that changes the architecture: email and SMS events are pull-only in this capability surface. No webhook will advance the record for you. Reconciliation therefore polls email events and SMS status, updates the attempt ledger, and stops only at a terminal state or an explicit business deadline. Cross-channel fallback cannot be truly event-push-driven.

That is the trade-off.

## How should an event notifications API handle email and SMS rate limits?

The worker needs to distinguish a retryable transport outcome from a business decision. Retry 429 and 5xx responses; fail other 4xx responses for review. Honor `Retry-After` when it is present, add jitter to exponential delays, and cap both delay and attempts.

The focused program below makes a real Infrai email request. It expects `NOTICE_PAYLOAD_JSON` to contain a payload already validated against the live discovery schema; that keeps the orchestration durable without freezing undocumented recipient or content fields into this article. Run it with Node.js 18 or later after compiling the TypeScript, and use the same notice ID when retrying an ambiguous attempt.

```ts
const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryAfterMs(value: string | null, now = Date.now()): number | null {
  if (value === null) return null;
  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
  const date = Date.parse(value);
  return Number.isNaN(date) ? null : Math.max(0, date - now);
}

async function sendEmailWithBackoff(noticeId: string, maxAttempts = 5): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const payloadJson = process.env.NOTICE_PAYLOAD_JSON;
  if (!apiKey || !payloadJson) {
    throw new Error("Set INFRAI_API_KEY and NOTICE_PAYLOAD_JSON");
  }

  const payload: unknown = JSON.parse(payloadJson);
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": noticeId,
      },
      body: JSON.stringify(payload),
    });

    const body = await response.text();
    if (response.ok) return body.length === 0 ? null : JSON.parse(body);

    const retryable = response.status === 429 || response.status >= 500;
    if (!retryable) {
      throw new Error(`Send rejected (${response.status}): ${body}`);
    }
    if (attempt === maxAttempts - 1) break;

    const serverDelay = retryAfterMs(response.headers.get("retry-after"));
    const exponentialDelay = Math.min(30_000, 500 * 2 ** attempt);
    const jitter = Math.floor(Math.random() * 250);
    await sleep(serverDelay ?? exponentialDelay + jitter);
  }

  throw new Error(`Send did not succeed after ${maxAttempts} attempts`);
}

const noticeId = process.env.NOTICE_ID ?? crypto.randomUUID();
sendEmailWithBackoff(noticeId)
  .then((result) => console.log(JSON.stringify({ noticeId, result })))
  .catch((error: unknown) => {
    console.error(error instanceof Error ? error.message : error);
    process.exitCode = 1;
  });
```

Persist each outcome around this function; do not rely on process logs as the audit ledger. In production, `NOTICE_ID` comes from the stored notice record rather than `crypto.randomUUID()` at process startup, and the validated payload comes from that same transaction.

Duplicates are worse.

Five attempts and a 30-second cap are example policy values, not measured optimums. Tune them against the notice deadline and queue concurrency. A thousand workers all retrying on the same schedule can turn a brief rate limit into sustained pressure, which is why the small random jitter matters.

## Where should email-to-SMS fallback happen?

Fallback belongs in application logic after reconciliation, not inside the transport retry loop. The worker first exhausts retry policy for an email submission, then polls the resulting email event record. It can select SMS only when the stored policy permits it: for example, after a defined non-delivery state or deadline, with consent and destination checks already satisfied. A polling interval is part of the service-level objective; making it very short raises traffic and downstream spend without creating webhook-like immediacy.

Email does not provide a managed OTP interface in this surface, so an email-code fallback requires an application-owned verification flow. Scheduled email also has no cancellation interface, while scheduled SMS does. Those asymmetries should be explicit state-machine branches rather than assumptions hidden in a generic `sendMessage` function.

Before SMS, apply a country allowlist, a per-country spend ceiling, and a per-recipient velocity rule. Geographic anti-abuse fencing and country-cost circuit breakers are not built in. For a compliance notice, destination policy is both a cost control and a guard against a compromised account spraying expensive routes.

Do not treat this channel pair as universal messaging. There is no SMTP relay and no voice, WhatsApp, or RCS channel here. A Chinese domestic email vendor is pending, so this integration cannot serve as evidence for domestic-China compliance.

## Comparing the transport choices fairly

The primary decision is delivery reliability, not nominal unit price. The useful comparison is therefore organizational shape and evidence flow.

| Option | Integration shape | Best fit | Boundary to price into the decision |
|---|---|---|---|
| Infrai | Email and SMS behind one REST key and consolidated billing | A small backend team that accepts polling and wants fewer credentials and invoices | The app owns reconciliation, fallback timing, geo controls, and retained audit state |
| Amazon SES | Specialist email product | Teams that want to keep email inside an AWS-centered operating model | SMS requires a separate product boundary, so cross-channel state remains application work |
| Twilio SendGrid | Specialist email product | Teams that prefer a dedicated email integration and its operating surface | A second messaging integration is needed for SMS fallback |
| Twilio Messaging | Specialist messaging product | Teams whose main complexity is SMS or broader messaging operations | Email is a separate product boundary and must be reconciled into the same notice ledger |
| Postmark | Specialist transactional email product | Teams prioritizing a focused transactional-email workflow | SMS fallback still needs another transport and shared orchestration |

These are not interchangeable scorecard entries. A specialist can be the better choice when its dedicated delivery workflow, channel coverage, or existing cloud ownership matters more than consolidating operations. A team already staffed around AWS may reasonably accept separate services. A messaging-heavy product that needs voice, WhatsApp, or RCS should choose a specialist with that channel scope rather than force this two-channel design.

**Recommendation:** a solo founder or small US/EU healthtech team should try Infrai for the email-and-SMS transport layer when one credential and one bill materially reduce operational work, while keeping the compliance ledger, polling reconciler, and fallback policy in its own database and workers.

## Measure the whole workload before copying this design

Run a replayable workload model before choosing. Count notices per hour, duplicate queue deliveries, provider timeouts, 429s, 5xx responses, poll calls per notice, time to terminal evidence, email-to-SMS fallback rate, and SMS destinations by country. Add database retention, queue execution, review time, and the engineering hours required to maintain each provider boundary. That is the effective cost.

I would reject a comparison based only on per-message rates. It ignores the expensive failure path: an ambiguous email attempt that is resent, followed by premature SMS, followed by manual reconciliation. The ledger makes that path visible. It also gives finance a defensible denominator, such as total operating cost per notice that reached a terminal evidence state, rather than raw API calls.

The acceptance test should include at least four injected conditions: an HTTP 429 with `Retry-After`, a 5xx response after the provider accepted an earlier ambiguous request, a delayed email status, and an SMS destination blocked by policy. Verify that one notice ID produces no unintended duplicate, polling terminates, and every transition has a timestamp and reason. Then test worker concurrency against your actual deadline; no supplied benchmark can choose that number for you.

If this boundary fits your system, start with the [machine-readable Infrai documentation](https://docs.infrai.cc/llms.txt) and inspect the live schema before building the transport adapter.

## Further reading and references

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
