# SendGrid, Resend, and Postmark Alternatives: Transactional Email API in Node.js

Short answer: for a property-management app sending a short-lived password-reset message, choose the provider whose domain verification, template, and suppression workflow you can operate. SendGrid is the broadest ecosystem, Resend is the most modern developer experience, Postmark is tightly focused on transactional delivery, and an aggregator such as Infrai fits when one API key and one bill across backend services matter more than SMTP or push webhooks.

The decision rule is practical: if the app already speaks SMTP, keep that compatibility; if the team wants a small Node.js surface, pick a focused API; if email is one piece of a wider backend, consider a unified gateway and accept its event and channel limits.

## What does the reset flow actually need?

The happy path is short. A tenant requests a reset, the service creates a random token that expires quickly, and the mail provider renders a template with a link. Before sending, the service checks that the sender domain is verified and that the recipient is not suppressed. Afterward, a worker records the provider message ID and periodically pulls delivery events for reconciliation. For a concrete property-management flow, use a 10-minute expiry, make the link single-use, and keep the building name in the template while leaving unit and lease details out. Those are application decisions, but the provider choice determines how much glue surrounds them.

Keep it dull.

That last step is easy to skip. It is also where operational differences show up. A reset email is security-sensitive, so the application owns token expiry and must avoid putting secrets in logs. The provider owns transport, domain authentication, and suppression hygiene.

Here is a minimal TypeScript sender. The request is deliberately small: one template payload, one explicit expiry value, and an idempotency key derived from the reset request. The retry loop honors `Retry-After` on rate limits and surfaces non-success bodies. Set `INFRAI_BASE_URL` to the service base URL in deployment configuration, alongside the API key, so neither value lands in source control.

```ts
type ResetMail = {
  requestId: string;
  recipient: string;
  resetUrl: string;
  expiresInMinutes: number;
};

export async function sendResetMail(input: ResetMail) {
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  const body = {
    to: input.recipient,
    subject: "Reset your property portal password",
    template: "password-reset",
    variables: {
      reset_url: input.resetUrl,
      expires_in_minutes: input.expiresInMinutes,
    },
  };

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `reset-${input.requestId}`,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return (await response.json()) as { id: string };

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const detail = await response.text();
    throw new Error(`email send failed (${response.status}): ${detail}`);
  }

  throw new Error("email send exhausted retries");
}
```

The same integration pattern applies when using a provider SDK: keep the token lifecycle in your service, pass a stable request identifier, and store the response ID. Verify the sending domain before production traffic; DKIM rotation belongs in the domain maintenance runbook, not in the reset handler.

## Should I use SendGrid, Resend, Postmark, or another transactional email API?

SendGrid is the sensible choice for teams that need a mature marketing-and-transactional ecosystem, SMTP relay, and a large set of integrations. That breadth adds configuration surface area. A small onboarding service may use only a fraction of it, while still inheriting more dashboards and concepts.

Resend favors a compact, developer-oriented API and React-email-style workflows. It is attractive when templates live close to application code. Check the exact event and retention behavior you need before committing; a reset-monitoring job still needs a clear polling or webhook plan.

Postmark is deliberately transactional. Its message streams and delivery focus make it easy to reason about password resets and welcome mail separately from bulk campaigns. The tradeoff is a narrower ecosystem and fewer general-purpose messaging features than SendGrid.

An aggregator can be a better fit for a solo founder who is already wiring storage, scheduling, and AI calls. Infrai presents those backend capabilities behind one REST credential and a consolidated bill, so a reset service does not acquire another vendor account just for email. Its email routes cover sending, templates, domain verification, DKIM rotation, and suppression lists. The tradeoff is concrete: there is no SMTP relay, and event retrieval is pull-only, so instant trigger workflows need an application poller or another event system.

No provider erases that decision.

| Option | Strong fit | Boundary to verify |
| --- | --- | --- |
| SendGrid | SMTP plus broad ecosystem | More operational surface for a small app |
| Resend | API-first, code-adjacent templates | Event and retention details for your audit needs |
| Postmark | Transactional streams and focused delivery | Smaller ecosystem outside transactional email |
| Unified gateway | One credential across backend services | No SMTP; pull-only events; channel readiness varies |

This is not a performance ranking. Provider defaults, regional availability, and policy change. I would choose the smallest integration that meets the event requirement, then test the exact sender domain and recipient mix the app will operate. Adding a broad platform for one reset message creates work; choosing a narrow API when the roadmap already needs several backend services creates a different kind of work.

## Where does domain and suppression work belong?

Treat domain verification as a deployment prerequisite. Publish the provider's DNS records, verify the domain, and schedule DKIM rotation. Google’s sender guidance is a useful baseline for authentication and complaint handling, even when your provider has its own checklist.

Suppression handling should be boring and explicit. Before a reset send, check the recipient against the suppression list; after a hard bounce or complaint, add it. A periodic event pull can reconcile provider state into your own audit table. Pull-only events are adequate for a dashboard that refreshes every few minutes, but they are a poor foundation for a workflow that must react within seconds.

There is no hosted email OTP fallback in this capability, and scheduled email cancellation is not available. If the product later adds email verification codes or delayed campaigns, those controls belong in application code or a different provider. SMS has its own cancellation route, but that does not change the email design.

## Operational checklist before shipping

Set the reset token TTL in the application and invalidate it on first use. Use a generic response for unknown email addresses so the endpoint does not reveal which tenants exist. Keep the template free of unnecessary personal data, and make the reset URL single-use. Record a request ID, provider message ID, template version, and send outcome; never record the token itself.

Run a staging send after DNS verification, then exercise a suppressed recipient and a forced 429 response. Confirm that a retry with the same idempotency key does not create a second message. Finally, decide how often the reconciliation job runs and what happens when the provider is unavailable. Those choices matter more than a feature checklist.

## References

- https://docs.sendgrid.com/for-developers/sending-email/api-getting-started
- https://resend.com/docs/send-with-nodejs
- https://postmarkapp.com/developer/api/email-api
- https://support.google.com/a/answer/81126
