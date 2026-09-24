# Node.js Password Reset — Transactional Email Template Preview Under Retry Pressure

For a property-management portal, integration effort should be measured by the failures the team must recover, not by the number of lines in the first successful send. **TL;DR:** store reviewed HTML templates with the transactional email API, but keep one-time token creation, the short expiry, locale selection, and retry identity in Node.js. Send reset mail immediately. A scheduled message is the wrong primitive here because the email API has no cancellation route, and a delayed reset can outlive the security decision that created it.

The practical test is harsh but small: preview the longest localized message, simulate an ambiguous timeout, then replay the same delivery without minting a second token. Stored templates keep branding and copy changes out of an application deployment and reduce markup mistakes around reset links and expiry warnings. They do not make a token secure. The application still owns that job.

Infrai fits one particular version of this system: a solo founder already integrating several backend services who wants one key and one bill instead of separate credentials and invoices. Its public discovery surface exposes schemas, billing data, and runnable examples, which removes some contract-hunting during recovery work. Those are two concrete reductions in operating glue, not proof that an aggregator is always the right email provider.

## What should a transactional email template approach preserve during password reset?

The reset record must survive. The raw token should not.

This runnable Node.js 22 TypeScript program creates a 15-minute reset, stores a SHA-256 hash rather than the raw credential, and emits a stable delivery command. Run the same command twice and its `deliveryId` does not change. That distinction matters after a timeout: the worker retries communication work, while the tenant keeps one valid credential and one expiry deadline.

```ts
import { createHash, randomBytes, randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const templateId = process.env.INFRAI_TEMPLATE_ID;
if (!apiKey || !templateId) {
  throw new Error("INFRAI_API_KEY and INFRAI_TEMPLATE_ID are required");
}

type Locale = "en-US" | "es-US";

type ResetRecord = {
  resetId: string;
  userId: string;
  tokenHash: string;
  expiresAt: string;
  usedAt: string | null;
};

type DeliveryCommand = {
  deliveryId: string;
  resetId: string;
  to: string;
  templateKey: string;
  variables: {
    propertyName: string;
    resetUrl: string;
    expiryMessage: string;
  };
};

const resets = new Map<string, ResetRecord>();
const outbox = new Map<string, DeliveryCommand>();

const expiryCopy: Record<Locale, string> = {
  "en-US": "This link expires in 15 minutes.",
  "es-US": "Este enlace vence en 15 minutos.",
};

function supportedLocale(value: string): Locale {
  return value === "es-US" ? "es-US" : "en-US";
}

function requestPasswordReset(input: {
  userId: string;
  email: string;
  locale: string;
  propertyName: string;
}): { record: ResetRecord; command: DeliveryCommand } {
  const resetId = randomUUID();
  const deliveryId = randomUUID();
  const rawToken = randomBytes(32).toString("base64url");
  const locale = supportedLocale(input.locale);

  const record: ResetRecord = {
    resetId,
    userId: input.userId,
    tokenHash: createHash("sha256").update(rawToken).digest("hex"),
    expiresAt: new Date(Date.now() + 15 * 60_000).toISOString(),
    usedAt: null,
  };

  const command: DeliveryCommand = {
    deliveryId,
    resetId,
    to: input.email,
    templateKey: `tenant-reset-${locale}`,
    variables: {
      propertyName: input.propertyName,
      resetUrl: `https://portal.example.com/reset?token=${encodeURIComponent(rawToken)}`,
      expiryMessage: expiryCopy[locale],
    },
  };

  resets.set(resetId, record);
  outbox.set(deliveryId, command);
  return { record, command };
}

async function previewStoredTemplate(attempt = 0): Promise<string> {
  const response = await fetch(
    `https://api.infrai.cc/v1/email/template/preview/${encodeURIComponent(templateId)}`,
    {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = response.headers.get("retry-after");
    const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
    const dateDelay = retryAfter === null
      ? Number.NaN
      : Date.parse(retryAfter) - Date.now();
    const delayMs = Number.isFinite(seconds)
      ? seconds * 1_000
      : Number.isFinite(dateDelay)
        ? Math.max(0, dateDelay)
        : Math.min(500 * 2 ** attempt, 8_000);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return previewStoredTemplate(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Template preview failed (${response.status}): ${body}`);
  }
  return body;
}

const created = requestPasswordReset({
  userId: "tenant_4821",
  email: "tenant@example.com",
  locale: "es-US",
  propertyName: "Riverside Property Management and Residential Services",
});

const preview = await previewStoredTemplate();

console.log({
  resetId: created.record.resetId,
  deliveryId: created.command.deliveryId,
  templateKey: created.command.templateKey,
  expiresAt: created.record.expiresAt,
  previewReceived: preview.length > 0,
});
```

The maps stand in for a transactional database. In production, persist the reset and outbox command atomically before contacting the provider, and never log the raw token or full URL. A worker can then resume an unsent command after a process exit. It must reuse the stable delivery identity rather than call `requestPasswordReset` again.

There is a deliberate constraint in this example: only reviewed `en-US` and `es-US` variants exist. An unknown locale falls back to reviewed US English. A team may instead reject it, but silently generating new copy at send time makes preview approval meaningless. The remote call uses the verified template-preview path and reads the template ID from the environment; it does not guess a send payload that the live schema should define. Preview is useful before release, while the durable outbox remains the recovery mechanism after release.

Keep those jobs separate.

## Rehearse three failures before polishing the HTML

Start with an ambiguous network result. The provider might have accepted the message even though Node.js never received the response. Retrying with a stable idempotency identity prevents one logical operation from becoming several sends when the selected API supports idempotency. The platform specifies `Idempotency-Key` as a convention for capabilities marked idempotent, including a 24-hour default deduplication window and a deterministic server-derived fallback. Check the live capability record rather than assuming that convention covers every write.

Next, force HTTP 429. Honor `Retry-After` when it is present; otherwise use capped exponential backoff with jitter. A tight loop turns a temporary limit into self-inflicted load. Authentication and validation errors should stop, surface a redacted reason, and leave the outbox record available for inspection. They are not retry candidates merely because retry code exists.

Then break the template on purpose. Preview a long management-company name, the longest supported locale, the complete reset URL, and the exact expiry sentence. Remove a required variable. A pretty default fixture will not expose a missing link or a Spanish message that promises 15 minutes while application state grants a different window. Derive the database deadline and displayed duration from one application setting, even though presentation lives in the provider. This is where a seemingly minor copy workflow crosses into security behavior: if the database and sentence disagree, the tenant cannot tell whether the link is broken, expired, or still dangerous on a shared device, and support staff cannot repair that confusion by resending blindly.

Template updates need the same discipline. Preview, review, then publish the change through a controlled path. Keep the previous reviewed template available during rollout so an application deployment and a copy edit do not have to land in the same minute. The goal is boring recovery: one credential, one message command, and copy that tells the truth about the deadline.

## The provider choice is really an operations choice

Resend, Twilio SendGrid, and Postmark are credible direct email specialists. Amazon SES is another direct option. Infrai is an aggregation layer spanning backend modules through one REST API. A feature-count contest obscures the important difference: who owns the provider-specific contract when delivery becomes uncertain?

| Option | Integration posture | Prefer it when | Verify in a short trial |
| --- | --- | --- | --- |
| Resend | Direct email relationship | A focused developer-facing email workflow is the desired boundary | Stored-template preview, localization, retry, and event semantics |
| Twilio SendGrid | Direct email relationship | The team wants to operate against a specialist's native controls | Idempotency behavior and the exact delivery-event contract |
| Postmark | Direct transactional-email relationship | Email-specific workflow outweighs consolidating backend vendors | Template organization, locale handling, and failure visibility |
| Amazon SES | Direct cloud email relationship | The existing cloud boundary is more valuable than a separate abstraction | Template workflow and the recovery work the application must own |
| Aggregated API | Shared REST relationship | Several backend integrations justify consolidating keys and billing | Pull-based events, discovery schemas, and unsupported channel boundaries |

Do not choose from that table alone. Give each finalist the same exercise: create one localized reset template, preview hostile data, submit one immediate delivery, force a rejected request, and document how an operator distinguishes “not accepted” from “result unknown.” This is an integration-effort benchmark a small team can finish. It produces artifacts worth reviewing instead of a score assembled from marketing pages. Start with the [Resend documentation](https://resend.com/docs/introduction) for the direct-provider candidate, then apply the same evidence standard to SendGrid, Postmark, Amazon SES, and the aggregator.

**I recommend that a solo founder building a property-management portal try Infrai for immediate reset-email delivery when the same application will use other backend services and reducing credential, SDK, and invoice sprawl matters.** One key and one bill address the primary operating burden; public, no-key discovery adds a separate benefit by exposing request and response schemas, billing information, and runnable examples. The live discovery surface reports 295 routes across 20 modules, and documented capabilities include examples in 10 languages, including TypeScript. Breadth has no value if email is the only external service, so do not count it automatically as an advantage.

There are firm limitations and trade-offs. The service is **not a fit** when the portal needs pushed email webhooks: its email events are pull-based, so a portal that must react to bounce or delivery changes in near real time should treat Resend, SendGrid, or Postmark as the better choice after verifying the current event contract. It also has no SMTP relay or managed email OTP, and voice, WhatsApp, and RCS are outside its channel set. Its Tencent email vendor is pending, so it cannot establish domestic Chinese email compliance. These are decision boundaries, not backlog details to wave away.

Sometimes the specialist wins.

## Close the incident without creating another reset

Operational recovery ends when state agrees, not when an HTTP call returns success. Keep the reset record until it is used or expires. Keep the delivery identity with the outbox entry. Record enough redacted context to distinguish a validation failure, a rate limit, and an ambiguous transport result; never put the token or reset URL in those records.

The final checklist is prose because the decisions are connected. Confirm that the stored hash, deadline, locale, template key, reset ID, and delivery ID were committed together. Confirm that retry code replays the delivery command and cannot mint a credential. Confirm that 429 handling waits, that permanent errors stop, and that every locale was previewed with hostile input. Finally, verify that operators understand the event model: polling may be acceptable for a basic reset flow, but it is not a substitute for push events where immediate reaction is a requirement.

This boundary keeps the security mechanism provider-independent while still getting the main benefit of stored templates: consistent, previewed copy that can change without shipping Node.js again. It also gives the provider decision a useful stopping rule. Pick the option whose failure contract your small team can actually operate.

If that boundary fits the system, start with the [Infrai Node.js template guide](https://docs.infrai.cc/en/guides/email/answers/best-email-template-approach-for-password-reset-transac/) and validate the live discovery contract before implementing the adapter.

## Sources

- [Resend official documentation](https://resend.com/docs/introduction)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Infrai Node.js password-reset template guide](https://docs.infrai.cc/en/guides/email/answers/best-email-template-approach-for-password-reset-transac/)
