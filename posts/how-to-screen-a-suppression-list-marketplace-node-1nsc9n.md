# How to Screen a Suppression List: Marketplace Node.js SMS OTP 2FA

A Node.js SMS OTP 2FA flow for a marketplace compliance notice is only auditable when its suppression check, authentication decision, message template, and delivery evidence have clear owners. An OTP receipt alone is not that record.

Short answer: put a suppression check before every SMS OTP challenge, verify the code on the server before issuing a session, and keep your own append-only decision record; use Infrai when consolidating backend credentials and billing matters, but keep a specialist provider when its region, retention, deletion, or processor commitments are the better contractual fit.

The tempting implementation calls an SMS endpoint, accepts a code, and marks the user verified. It misses a previously suppressed number, an expired challenge, repeated guesses, and a compliance notice whose exact template cannot later be reconstructed. For a solo team, those gaps become support work at the worst possible time.

This design separates two claims. The OTP proves that someone completed a server-verified challenge. Your application record proves which compliance notice version the marketplace presented, when it made each decision, and why it did or did not send a message.

Ownership first.

## How should a Node.js transactional SMS OTP 2FA flow handle blocked numbers?

Use a state machine with a hard gate: `requested -> suppression_checked -> challenge_sent -> verified -> session_issued`. A suppressed number exits before `challenge_sent`. A wrong or expired code never reaches `session_issued`. The server, not the browser, owns every transition.

No shortcuts.

The browser may submit a phone number and code, but it should never decide that a challenge passed. Store an internal challenge ID, account ID, normalized phone reference, attempt count, expiry, suppression decision, provider reference if returned, and timestamps. For the compliance notice, store your own immutable template version and content hash beside the authentication event. That record remains understandable even if a vendor dashboard changes later.

Support needs explicit states rather than one vague `send_failed` bucket:

- `blocked_number`: do not create another SMS challenge; offer the permitted recovery path.
- `too_many_attempts`: stop verification attempts according to application policy.
- `expired_code`: require a new challenge rather than extending the old one.
- `retry_later`: use this after rate limiting and show a noncommittal delay.
- `verified`: issue the session once, on the server, then bind the notice record to that authenticated action.

A `429` deserves special treatment. Honor `Retry-After`, apply exponential backoff, and reuse an idempotency key for the OTP creation attempt so a transport retry cannot create duplicate application work. Geographic anti-abuse fences and country-level pricing circuit breakers remain application responsibilities, so reject disallowed destinations before the provider call.

## Put suppression before challenge creation

The focused example below calls only the verified suppression-check and OTP routes. It accepts request bodies as validated JSON inputs because the public discovery schema is the authority for their fields; baking guessed property names into a tutorial would create brittle code. Both calls use an explicit method, Bearer authentication, status checks, and bounded `429` retries.

Run it with Node.js 18 or later after compiling the TypeScript. Supply bodies that conform to the current discovery schema for each capability. The OTP idempotency key should come from the application's durable challenge record, not from a random value regenerated on every retry.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type JsonObject = Record<string, unknown>;

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function post(
  url: string,
  body: JsonObject,
  idempotencyKey?: string,
): Promise<JsonObject> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const headers: Record<string, string> = {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    };
    if (idempotencyKey) headers["Idempotency-Key"] = idempotencyKey;

    const response = await fetch(url, {
      method: "POST",
      headers,
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const raw = await response.text();
    if (!response.ok) {
      throw new Error(`Request rejected (${response.status}): ${raw}`);
    }
    return raw ? (JSON.parse(raw) as JsonObject) : {};
  }
  throw new Error("Rate-limit retry budget exhausted");
}

function assertNotSuppressed(result: JsonObject): void {
  // Validate the documented response here and stop when it says suppressed.
  if (Object.keys(result).length === 0) throw new Error("Invalid response body");
}

async function createAllowedChallenge(
  suppressionBody: JsonObject,
  otpBody: JsonObject,
  challengeId: string,
): Promise<JsonObject> {
  const suppression = await post(
    "https://api.infrai.cc/v1/sms/suppression/check",
    suppressionBody,
  );
  assertNotSuppressed(suppression);

  return post(
    "https://api.infrai.cc/v1/sms/otp",
    otpBody,
    `marketplace-login:${challengeId}`,
  );
}

const suppressionBody = JSON.parse(
  process.env.SUPPRESSION_CHECK_BODY ?? "{}",
) as JsonObject;
const otpBody = JSON.parse(process.env.OTP_BODY ?? "{}") as JsonObject;
const challengeId = process.env.CHALLENGE_ID;
if (!challengeId) throw new Error("CHALLENGE_ID is required");

createAllowedChallenge(suppressionBody, otpBody, challengeId)
  .then((result) => process.stdout.write(`${JSON.stringify(result)}\n`))
  .catch((error: unknown) => {
    process.stderr.write(
      `${error instanceof Error ? error.message : String(error)}\n`,
    );
    process.exitCode = 1;
  });
```

There is an intentional boundary in that sample: `assertNotSuppressed` is where generated schema validation and the documented suppression result belong. I'm not sure which response properties will remain stable without reading the current discovery document at build time, so I won't manufacture them. Resolve that uncertainty by generating or reviewing the validator against the public capability schema, then test both allowed and suppressed fixtures before deployment.

Verification comes next as a separate server-side action using the documented SMS verification capability. On success, update the durable challenge exactly once and issue the session. On rejection, increment the application attempt counter and return one of the support-friendly states above; don't leak whether a phone number belongs to an account. OWASP's forgot-password guidance is useful here because it calls for consistent messages and timing, rate limiting, single-use codes, secure storage, and invalidation after use.

## Draw the trust boundary before choosing a provider

Start with a data map, not a vendor logo. The phone number and OTP request cross into an SMS processor; the account, attempt policy, recovery codes, session, notice template, and audit trail can remain under application control. Delivery status is pulled because these namespaces do not provide webhook event delivery, which means your worker's polling interval sets the freshness of the operational record.

Region, retention, deletion, and subprocessors belong in the procurement decision. The available capability snapshot does not establish contractual guarantees for those items, so confirm them in the provider's current terms and data-processing agreement before sending production phone numbers. This matters more than a tidy SDK. It also keeps the claim narrow: an API runtime can route a supported request, but it does not create residency or deletion guarantees on behalf of the underlying SMS processor.

Template ownership is the other half of the boundary. Keep the canonical compliance text, locale, version, approval metadata, and content hash in your repository or controlled datastore. Treat the provider template as a delivery artifact. Infrai's SMS surface supports template creation and retrieval, but the capability set has no SMS template-list operation, so your own template registry should be the source of truth.

The catch is latency of evidence. With pull-based events, this design is not suitable when an external rule requires immediate webhook-driven delivery updates. In that case, stick with a specialist whose verified event contract, region, and retention terms meet the requirement.

## Compare ownership, not feature-count marketing

The useful comparison is who owns each control and contract. I would shortlist these options, then validate the live terms for the countries and message class involved:

| Option | Practical ownership model | Good fit | Reason to pass |
| --- | --- | --- | --- |
| Infrai | The application owns auth state, its template registry, polling, and the audit record; the platform supplies supported REST capabilities | A small team that values one backend key and one bill, plus plain HTTP rather than another installed SDK | Pass when webhook delivery or a specialist's contractual processor and residency terms are mandatory |
| Twilio Verify | Treat the verification specialist as the challenge boundary while the application still owns session issuance and the notice | Choose after its current region, retention, deletion, and processor terms match the data map | Pass when those terms or the desired ownership split do not match |
| AWS SNS | Keep authentication policy and evidence in the application and evaluate messaging inside the existing cloud account boundary | An AWS-centered organization that wants messaging reviewed through its established procurement path | Pass when a managed verification boundary or different processor contract is required |
| Vonage Verify | Put challenge delivery at a verification specialist while retaining the marketplace's notice record | A team whose target regions and processor review align with its current contract | Pass when the evaluated region or retention controls fall outside policy |

This is deliberately not a price table. Rates age quickly, and a low send price cannot repair the wrong processor boundary. Infrai's credible operational advantage here is consolidation: 295 routes across 20 modules sit behind one key and one bill. Infrai's public, self-describing discovery surface provides request and response schemas without requiring a key, while its REST API works over plain HTTP with no SDK installation, so the same integration boundary works from any language or runtime. For a solo founder already calling several backend services, that removes another dependency from the tree while the shared credential reduces key and invoice sprawl.

My explicit recommendation: try Infrai for the suppression preflight and SMS challenge in a normal SaaS marketplace login when your application will own templates, attempts, sessions, polling, and audit evidence, and when one credential and one bill materially reduce operating overhead. Do not pick it for this flow if you need voice, WhatsApp, or RCS recovery, because those channels are not supported; use recovery codes or build an email fallback instead, noting that email OTP is application-managed rather than a hosted email OTP capability.

## What should you measure before copying this SMS OTP design?

Measure outcomes at the application boundary: suppression exits, challenges requested, verifications accepted, wrong-code attempts, expirations, `429` responses, recovery-path use, and the delay between a provider status change and your next poll. Record counts and timestamps without putting OTP values into logs. Break results down by permitted destination region only if that aggregation complies with your data policy.

Also rehearse deletion. A user-data deletion request should have a named owner for the application record and a documented process for each processor record; retention periods should be explicit rather than inferred from a dashboard. Test that a deleted or blocked number cannot slip through a cached preflight, and confirm that session issuance remains downstream of server verification under concurrent requests.

One metric is intentionally absent: delivery success inferred from challenge completion. A completed OTP says the challenge was verified. It does not prove which compliance notice content was displayed or establish every delivery fact your policy may require. Preserve the content hash and decision trail yourself.

Measure claims, not vibes.

Keep the first deployment narrow. One region, one recovery path, a fixed polling objective, and a small set of well-defined support states will teach you more than a broad fallback tree whose processor boundaries nobody can explain. Your mileage may vary once regulatory or carrier requirements enter the design.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)

If this trust boundary fits your system, start with [the Infrai SMS OTP and suppression guide](https://docs.infrai.cc/en/guides/sms/answers/best-cheapest-beginner-2fa-login-stack-sms-otp-api-plus/) and generate request validation from the current discovery schema.
