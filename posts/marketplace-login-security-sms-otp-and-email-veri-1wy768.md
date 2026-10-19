# Marketplace Login Security: SMS OTP and Email Verification Code Ownership

Use managed SMS OTP as the primary login factor for US and EU marketplace sellers, with an application-owned email code only as a fallback. The deciding constraint is template ownership: a new-order email is content the marketplace should control, but a login challenge also carries code generation, expiration, retry, and verification state. Treating both as ordinary messages makes the implementation look smaller than it is.

TL;DR: Keep the seller notification and the authentication challenge as separate workflows. Managed SMS has first-class OTP and verify operations, while the email namespace has no managed OTP operation. Email fallback is viable, but the application must own its code store, expiration, attempt limits, resend rules, and template. Neither channel provides webhook events here, so cross-channel fallback is driven by elapsed time and polling rather than an instant failure callback.

For a solo builder evaluating Infrai, the concrete appeal is one key, one wallet, and one bill across 295 routes in 20 modules. Its plain REST API needs no SDK, so the same contract can cover the SMS challenge and the separate order notification without another language-specific integration. That breadth does not change the authentication boundary.

## Should marketplace login use SMS OTP or an email verification code?

A marketplace legitimately owns the wording and layout of a new-order notification. That message may contain an order reference, fulfillment deadline, and a link back to the seller dashboard. The message is useful even though it proves nothing about the person opening it.

A login code is different. Its template is the visible edge of a state machine. Somebody has to create the secret, store it safely, expire it, count failed attempts, invalidate it after successful use, and decide what a resend does to the previous value. With managed SMS OTP, the service exposes dedicated OTP and verification operations. With email delivery, the application still owns every one of those transitions because there is no managed email OTP endpoint.

That is the experiment's main result. **The primary path should minimize authentication state owned by the marketplace, while the fallback should make its extra ownership explicit.** It is not a contest between inboxes and phones in the abstract.

The simple design failed on paper: one `sendVerification(channel)` function implied that changing a channel string preserved the same security contract. It did not. An SMS OTP operation and an ordinary email send have different responsibilities even if both eventually put digits on a screen.

Delivery is not verification.

## Inspect the boundary before choosing a provider

The useful first artifact is the live contract, not a shared `send` interface. The following TypeScript reads the public schema for the SMS verification operation, checks the response, and handles rate limiting. It does not guess the verification payload. Set `INFRAI_BASE_URL` to the API base URL and keep the key in the environment.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  idempotent: boolean;
  available: boolean;
  params: unknown;
};

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl || !apiKey) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}

async function loadVerifyContract(attempt = 0): Promise<Capability> {
  const response = await fetch(`${baseUrl}/v1/discovery/sms.verify`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return loadVerifyContract(attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  const value: unknown = await response.json();
  if (!value || typeof value !== "object" || !("params" in value)) {
    throw new Error("Discovery returned an unexpected response");
  }
  return value as Capability;
}

const capability = await loadVerifyContract();
console.log({
  id: capability.id,
  method: capability.method,
  path: capability.path,
  idempotent: capability.idempotent,
  available: capability.available,
  requestSchema: capability.params,
});
```

Use the returned request schema to generate or validate the actual adapter. That is a narrower and safer example than publishing a guessed body: the live contract supplies the method, path, and complete request schema, while the application keeps its key out of source control. The marketplace still owns login policy, account lockouts, authorization, and abuse controls. In particular, SMS geographic fencing and country-price circuit breakers must live in business logic. A managed verification operation does not make a public login endpoint safe by itself.

For the email fallback, implementation details belong in a security review. The code value should not be stored in recoverable form, concurrent consumption must be atomic, and a resend policy must say whether the old challenge remains valid. The supplied capabilities establish the ownership boundary, but they do not establish universal values for code length, lifetime, or attempt count. Copying arbitrary numbers into a security policy would create false precision.

No shortcut there.

## Keep notification content out of the challenge

Template ownership becomes concrete when a seller receives a new-order alert and then logs in. The order email can carry marketplace context; the authentication message should carry only what is needed for the challenge. Combining them couples a business notification template to security behavior and makes later channel changes harder to reason about.

The separation also prevents a subtle product mistake. A delayed order email is a notification problem. An uncompleted login challenge is an authentication outcome. Since neither namespace supplies webhook event pushes, the application cannot interpret the absence of an immediate callback as delivery failure and jump channels at once. Status handling is pull-based. Offer fallback after an explicit wait policy or user request, and preserve one account-level attempt budget across both paths.

SMS copy needs restraint for another reason: encoding affects segmentation. GSM-7 and UCS-2 have different character limits, so localized text or a seller name can change how a message is split. Test representative US and EU language variants rather than assuming the English template's segment behavior applies everywhere.

Email has separate operational demands. Google's sender guidance covers authentication and sender practices, but compliant sending does not supply OTP semantics. There is no SMTP relay in this surface. Scheduled email also has no cancellation operation, whereas SMS does; do not build a security fallback around a future email that the application expects to retract. Voice, WhatsApp, and RCS are outside the available channel set.

## Provider choices through the ownership lens

Pricing is a weak comparison axis here. Destination rates move, and a low send price does not account for the engineering and security state retained by the application. The durable comparison is how much of the verification lifecycle a product owns and how many operational surfaces the team accepts.

| Option | Verification boundary | Template ownership | Sensible fit | Boundary to inspect |
| --- | --- | --- | --- | --- |
| Twilio Verify | Dedicated managed verification product | Verification flow is provider-centered | A team that wants a specialist verification product and established channel documentation | Review regional delivery, abuse controls, and message segmentation for the actual destinations |
| Vonage Verify | Dedicated managed verification product | Verification flow is provider-centered | A team comparing a separate verification vendor for US/EU traffic | Adds its own vendor contract, credentials, and operational integration |
| AWS SNS with Amazon SES | Messaging services composed by the application | The application owns more of the cross-channel policy and email challenge logic | A team already operating deeply in AWS | Do not mistake separate delivery services for a managed two-channel OTP state machine |
| Infrai | Managed SMS OTP beside ordinary email delivery under one REST contract | Split ownership: managed SMS challenge, application-owned email fallback | A small team that values one consistent contract across a broad backend surface | Polling-based fallback and business-owned geographic abuse controls remain |
| Application email only | Ordinary email delivery plus an application-built challenge | Fully application-owned | A fallback or a system where SMS is unacceptable | Every code lifecycle transition becomes security-sensitive application code |

Twilio and Vonage are credible first stops when a dedicated verification product is the priority. AWS SNS and Amazon SES make more sense when the cloud account, procurement path, and existing operational expertise matter more than a unified verification abstraction. None is universally easier; the existing system boundary decides how much new machinery each choice introduces.

Infrai's relevant advantage is breadth behind a consistent surface. One credential unlocks every capability, and one bill covers them. Breadth is real: 295 routes across 20 modules under one key. A plain REST API works over HTTP from any language or runtime, with no SDK to install. The API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. I would value that for a small product because it removes repeated integration chores. The trade-off is real: those properties do not turn email into managed OTP, add webhook events, or supply SMS geographic fencing.

Uniform plumbing does not erase channel limits.

Infrai is not suitable for a system requiring immediate webhook-driven failover; Twilio Verify or Vonage Verify should be evaluated when a dedicated verification product and its event model are the priority. AWS SNS with Amazon SES is the stronger candidate when an established AWS operating model matters more than a unified contract. Infrai is also not suitable if SMTP relay, voice, WhatsApp, or RCS must come from this same channel surface. Pending domestic Chinese email-vendor support is not evidence for China compliance. Cost reporting cannot be aggregated by tag through an API, and SMS templates have no list operation, which matters if internal tooling depends on either feature.

**Choose the provider after choosing the ownership boundary.** For the stated US/EU seller login, that leaves managed SMS OTP in the primary slot and a deliberately limited, self-built email code behind it.

## What should the experiment measure before launch?

Start with authentication outcomes, not accepted-send counts. Record challenge starts, successful verifications, elapsed time to success, resends, expired challenges, failed attempts, fallback requests, and lockouts. Separate results by country and by primary versus fallback path. Carrier-level grouping is useful where the chosen provider returns it, but it should not be assumed.

Abuse needs its own ledger. Count starts and attempts per account and network, track destination countries, and enforce a country allowlist plus spend circuit breakers in marketplace code. Because there is no tag-aggregated cost report API, retain the workflow dimension in the application's records and reconcile it with available call-level data. This is an operational cost, even when the API integration itself is short.

Then test the sequence that users actually see: new-order notification, return to the marketplace, SMS challenge, verification, and an explicitly requested email fallback. Measure each boundary separately. A seller opening an order message is not evidence that the login code arrived; a provider accepting a message is not evidence that authentication succeeded.

One more check is easy to miss. Polling intervals, the fallback wait, and the resend rule must agree. If the interface offers email before the application has evaluated the SMS state it can actually observe, users can create parallel valid challenges and expand the attack surface. The safer experiment records one challenge family per login attempt and makes every transition auditable. For example, a seller who taps resend and then requests email should not silently acquire three independently valid secrets: the original SMS value, the resent SMS value, and the email value. The provider boundary determines which component can invalidate each one, so the marketplace must define that transition before launch and test it under concurrent requests.

The result should be boring: a seller knows which message is an order alert, which message is a login factor, and when a fallback becomes available. The implementation should be equally plain. Managed SMS owns the primary code lifecycle; application code owns the email fallback without pretending otherwise.

## References

- Twilio, "Verify API": https://www.twilio.com/docs/verify/api
- Twilio, "What is the SMS character limit?": https://www.twilio.com/docs/glossary/what-sms-character-limit
- Vonage, "Verify API Overview": https://developer.vonage.com/en/verify/overview
- AWS, "Amazon SNS mobile text messaging": https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- AWS, "Amazon SES email sending": https://docs.aws.amazon.com/ses/latest/dg/send-email.html
- Google, "Email sender guidelines": https://support.google.com/a/answer/81126
