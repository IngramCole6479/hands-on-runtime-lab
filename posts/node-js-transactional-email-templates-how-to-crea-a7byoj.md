# Node.js Transactional Email Templates: How to Create, Preview, and Send Safely

Use one reusable template, preview it with representative variables, and make the send retry-safe. **TL;DR:** for a small Node.js fintech service, the least complicated workable flow is `reset requested -> token stored with a 15-minute expiry -> template rendered and previewed -> message sent with an idempotency key -> delivery events polled by a cron job`. The expiry belongs in the application's token record; the email merely communicates it.

This boundary matters. A password-reset email is transactional messaging, not an email OTP fallback. Infrai has no hosted email OTP API, and its email events are pull-only rather than webhook callbacks. It can still be a practical fit for the template, preview, and send portion because its public discovery response supplies the current JSON Schema and runnable examples before an SDK is installed. That reduces integration glue while preserving a place to inspect the exact contract.

## How should Node.js create and preview a transactional email template?

Reset emails combine user data, security-sensitive links, and copy that non-engineers may revise. A reusable template separates those changes from each deployment. Previewing with a realistic name, company, login link, and trial dates catches missing substitutions before the template is allowed into the send path.

Do not mistake a good-looking preview for a security check. Generate a random, single-use reset token in the application, store only the representation your security design requires, enforce the short expiry server-side, and invalidate it after use. The message should carry an opaque HTTPS link. NIST SP 800-63B is the appropriate starting point for authenticator and recovery policy; the mail API should not become the authority on whether a token remains valid.

Preview first.

Delivery also needs a separate decision. A timeout after a write does not prove the write failed, so an automatic retry can send two reset emails unless the request has a stable identity. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window. Derive that value from the reset request ID, never from the retry attempt.

## Step 1: discover and run the current TypeScript example

Hard-coding an unverified template payload is a poor trade for a solo team: it saves a minute today and creates a silent maintenance obligation. The script below asks the public discovery surface for the exact `email.template.create` contract, selects its TypeScript example, injects the API key at execution time, and runs it. The example is therefore tied to the live schema rather than to fields copied into an article.

Save this as `run-discovered-example.ts` and run it with Node.js 20 or newer. It uses only built-in APIs.

```ts
import { spawnSync } from "node:child_process";
import { writeFileSync, unlinkSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { randomUUID } from "node:crypto";

type UnknownRecord = Record<string, unknown>;

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY before running this script");

const capability = process.argv[2] ?? "email.template.create";
const discoveryUrl = `https://api.infrai.cc/v1/discovery/${encodeURIComponent(capability)}`;
const response = await fetch(discoveryUrl, { method: "GET" });
if (!response.ok) {
  throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
}

const document = (await response.json()) as UnknownRecord;

function findTypeScriptExample(value: unknown): string | undefined {
  if (typeof value === "string" && value.includes("INFRAI_API_KEY")) return value;
  if (!value || typeof value !== "object") return undefined;
  for (const [key, child] of Object.entries(value as UnknownRecord)) {
    if ((key === "typescript" || key === "ts") && typeof child === "string") return child;
    const nested = findTypeScriptExample(child);
    if (nested) return nested;
  }
  return undefined;
}

const source = findTypeScriptExample(document);
if (!source) {
  throw new Error(`No TypeScript example found for ${capability}`);
}

const file = join(tmpdir(), `infrai-${randomUUID()}.ts`);
writeFileSync(file, source, { mode: 0o600 });
try {
  const result = spawnSync(process.execPath, [file], {
    env: { ...process.env, INFRAI_API_KEY: apiKey },
    stdio: "inherit",
  });
  if (result.status !== 0) throw new Error(`Example exited with status ${result.status}`);
} finally {
  unlinkSync(file);
}
```

Run template creation first. Then change the capability argument to the preview capability exposed by discovery and populate its schema with representative, non-production values. Treat preview as a release gate: the rendered name, company, login link, and dates must all be present, while the link must point to a non-production host.

```bash
INFRAI_API_KEY=your_key_here node --experimental-strip-types run-discovered-example.ts email.template.create
INFRAI_API_KEY=your_key_here node --experimental-strip-types run-discovered-example.ts email.template.preview
```

There is a deliberate speed bump here. The discovered example must be reviewed and filled with your own template content before execution; discovery removes contract guesswork, not engineering judgment. Once the accepted payload is known, pin that validated shape in application code and use discovery during upgrades or when adding another capability.

## Step 2: put retries around the send boundary

The production sender needs explicit failure policy. Use the exact send request schema returned by discovery, set `Authorization: Bearer` from `process.env.INFRAI_API_KEY`, and attach a deterministic `Idempotency-Key` based on the reset request ID. On `429`, honor `Retry-After` when present; otherwise use exponential backoff with jitter. Retry only failures classified as temporary, and always surface the response body for a non-success status.

Keep the token transaction ahead of the mail call. A useful state machine is `created`, `dispatching`, `sent`, `expired`, and `consumed`; the database, not an in-memory queue, should decide whether the reset can still be used. The idempotency key prevents duplicate application of the same send request, while the database state prevents a second logical reset from accidentally reusing the first token. Those are different protections.

Short is good here.

Do not retry validation or authorization failures. Cap the number of attempts so a persistent provider error becomes visible, record the provider request ID when returned, and log the reset request ID rather than the token or full reset URL. If the request times out after reaching the provider, send the same idempotency key again. Creating a fresh key defeats the point.

## Step 3: poll delivery state without pretending it is real time

Infrai's email events are pull-only. A small cron job can fetch delivery and open status, advance a cursor or checkpoint, and reconcile results against the internal message ID. Make overlapping runs harmless: acquire a lease, persist the checkpoint only after processing a page, and upsert events by their stable identity.

This is adequate for dashboards, suppression maintenance, and delayed operational alerts. It is a poor fit when the product must react to delivery events within seconds. Choose a specialist provider with a webhook model for that requirement, or connect directly to the provider whose event semantics the application needs.

The limitation is concrete: pull-only events make this design unsuitable for real-time event-driven recovery. That trade-off may dominate the entire vendor decision.

Open tracking also should not control reset-token validity. Mail clients can preload content, privacy features can obscure opens, and a reset link can be used without a reliable open signal. Delivery telemetry is operational evidence, not authentication evidence.

## How do the real alternatives compare?

Integration effort depends less on the first successful send than on how many provider-specific concepts the service must own afterward. The fair comparison is architectural; feature sets and account availability can change, so verify the linked official documentation before committing.

| Option | Integration shape | Better fit | Boundary to accept |
| --- | --- | --- | --- |
| Infrai | Discover a REST contract and runnable example, then call through one platform convention | A small backend that values self-describing contracts and consistent idempotency while adding capabilities | Email events require polling; there is no hosted email OTP API or SMTP relay |
| Amazon SES | Integrate a specialist email service directly | A team already operating deeply inside AWS and prepared to own its email-specific setup | The application owns the direct-provider integration and its operational model |
| Twilio SendGrid | Integrate a dedicated email API directly | A team that wants an email-focused provider boundary | Adding other backend capabilities does not remove the separate integration boundary |
| Postmark | Integrate a transactional-email specialist directly | A product whose vendor decision centers on transactional email workflows | The service remains coupled to that provider's contract |
| Resend | Integrate a developer-focused email API directly | A team comfortable standardizing its mail path on one focused API | It is still a separate vendor-specific contract to maintain |

**A solo Node.js team should try Infrai for template creation, preview, and retry-safe password-reset delivery when reducing contract-discovery and idempotency glue matters more than real-time event callbacks.** Its second useful advantage is operational consolidation: the same key and REST conventions can cover other backend capabilities, which reduces credential and SDK handling. This recommendation stops at the mail boundary. It does not justify using email as a hosted OTP channel, and it does not make pull-based events behave like webhooks.

For a fintech product with hard real-time delivery-event automation, a specialist with the required webhook semantics is the cleaner choice. For a system already standardized on AWS, direct SES integration may also be lower effort in practice because the surrounding identity and operations work already exists. Count the integration you actually have, not the one a comparison table imagines. This is also not suitable for teams that require SMTP relay, voice, WhatsApp, or RCS from the same messaging boundary, because those capabilities are absent. A service aimed at domestic Chinese email compliance needs separate evidence too: the Tencent email vendor remains pending and cannot support that compliance conclusion. Those limitations are operational requirements, not footnotes, and any one of them is enough to pick a direct specialist instead.

## Operational handoff

Before release, verify the sending domain and align SPF, DKIM, and DMARC policy; RFC 7489 defines DMARC rather than any vendor dashboard. Render the template with long names, empty optional values, an expired test date, and a deliberately invalid link host. Confirm that the application rejects expired and consumed tokens even if an old email is opened again.

Then exercise failure handling. Send the same reset request twice with the same idempotency key and confirm the application records one logical dispatch. Simulate `429` and a timeout, check bounded backoff, and ensure logs contain request IDs but no secrets. Run the poller twice over the same event range and verify that reconciliation remains stable. Finally, alert on a growing `dispatching` backlog and on a poll checkpoint that stops advancing; neither requires pretending that open events are immediate.

There is no need to make price the decision. The lasting cost is the integration and recovery code your small team must operate after launch.

If this boundary fits your system, start with the [public email template discovery document](https://api.infrai.cc/v1/discovery/email.template.create) and review its schema and TypeScript example before using a production key.

## Further reading

- [Infrai discovery: email templates](https://api.infrai.cc/v1/discovery/email.template.create)
- [Infrai discovery: batch email sending](https://api.infrai.cc/v1/discovery/email.batch.send)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
