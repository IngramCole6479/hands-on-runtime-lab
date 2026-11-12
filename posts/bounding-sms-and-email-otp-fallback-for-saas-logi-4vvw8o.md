# Bounding SMS and Email OTP Fallback for SaaS Login (Before Report Release)

A financial report can be ready while its recipient is still waiting for a login code. Starting another send doesn't establish what happened to the first one. TL;DR: for SMS OTP versus email OTP in US and EU SaaS login, choose the channel and fallback policy by authorized completion under delayed delivery. Keep one shared retry budget, explicitly retire superseded challenges, and measure report attachment delivery separately. Neither an accepted send nor an email open is sufficient evidence of the finished journey.

The result here is a proposed acceptance rule, not a measured channel winner. The experiment must preserve the same account-access requirements while injecting delayed and ambiguous outcomes. Otherwise, faster completion could mean that fallback weakened the release policy. For a fintech application sending a generated report as an email attachment, that would be the wrong result to optimize.

## How should SMS and email OTP fallback handle delayed codes during SaaS login?

Start the experiment with the awkward sequence: an SMS challenge is delayed, the customer requests email fallback, and then both codes arrive. The simple approach gives each channel its own resend counter and accepts whichever code comes back. That leaves two questions unanswered: how much sending can one login trigger, and which challenge still has authority? For this note, choose one active challenge per login journey and a shared send budget. Fallback replaces the challenge; it doesn't create a fresh allowance.

That choice has a cost. A customer who enters the earlier code after fallback must request or use the active challenge instead. The interface needs to explain that the earlier code has been replaced. The benefit is a testable rule about late arrivals, with no need to infer which message the customer saw first. Both SMS and email must obey it.

Keep the attachment out of these messages.

The report release is a separate authorized action tied to an immutable report version and an approved recipient address. Successful login alone shouldn't authorize an arbitrary document or a newly typed destination. In the test fixture, use report `statement-042`, recipient `member@example.test`, and a regional cohort label of either `US` or `EU`. These are illustrative inputs, not production observations. Hold the generated artifact constant while exercising retries so that model latency and token consumption don't contaminate the comparison.

This is a failure-injection experiment. It starts with the sequence most likely to expose a policy ambiguity, then works outward to ordinary delivery. It doesn't begin by averaging successful sends.

## Make the failure schedule the experiment input

Use the following acceptance matrix for both channel orders. SMS followed by email and email followed by SMS must face the same account authorization, destination enrollment, and retry rules. Otherwise, the experiment compares different security policies under the same label.

| Injected condition | Required application outcome | Evidence to retain |
| --- | --- | --- |
| First challenge arrives after fallback | Superseded challenge is rejected | Challenge identity and replacement event |
| Customer clicks resend repeatedly | Shared send allowance is enforced | Attempt count across both channels |
| Code arrives after its expiry | Verification fails; attachment stays unreleased | Expiry decision and release state |
| Recipient changes before release | Earlier destination approval is insufficient | Recipient approval checked at release |
| Attachment submission has an unknown outcome | Journey remains unresolved pending reconciliation | Submission attempt identity and uncertainty |

For a concrete fixture, configure 3 challenge sends within 10 minutes across the entire login journey. These numbers are test parameters, not a security standard or a production recommendation. Exercise the limit concurrently as well as sequentially: a counter checked independently by two workers isn't enough to establish a shared allowance. Require atomic enforcement in the storage contract. Add account, destination, and network-source controls according to the abuse model; don't let a new journey identifier reset the account's budget.

The security comparison needs an explicit boundary. Offering SMS and email as alternatives doesn't by itself establish two-factor authentication. Define what a completed challenge proves, what other factor the account policy requires, and which previously enrolled destination may receive recovery messages. Don't let a delay authorize enrollment of a new phone number or mailbox. This experiment can check those application rules; the two references below don't establish a universal ranking of SMS and email authentication security.

For US and EU cohorts, keep sample counts alongside elapsed-time distributions and completion rates. A region label is a grouping for observations, not a delivery guarantee or a compliance conclusion. If a cohort has too little evidence, leave its channel choice unresolved. Don't manufacture a winner from the overall average.

## A small replay test catches the policy mistake

The focused example below checks the late-arrival rule through an application-owned test interface. It's a TypeScript contract for a deterministic fixture, not a messaging SDK or a complete verifier. The fixture controls its clock, isolates one enrolled account, and implements replacement atomically. The opaque codes exist only in this test boundary; production diagnostic events should exclude codes and financial contents.

```ts
type Channel = "sms" | "email";
type Challenge = { id: string; code: string };

interface LoginFixture {
  issue(channel: Channel): Promise<Challenge>;
  verify(id: string, code: string): Promise<boolean>;
  advanceBy(milliseconds: number): Promise<void>;
  challengeSendCount(): Promise<number>;
  reportReleaseCount(): Promise<number>;
}

function expect(condition: boolean, message: string): void {
  if (!condition) throw new Error(message);
}

async function checkLateArrival(fixture: LoginFixture): Promise<void> {
  const earlier = await fixture.issue("sms");
  await fixture.advanceBy(30_000);
  const replacement = await fixture.issue("email");

  expect(
    !(await fixture.verify(earlier.id, earlier.code)),
    "The superseded SMS challenge must fail"
  );
  expect(
    await fixture.verify(replacement.id, replacement.code),
    "The active email challenge must succeed"
  );
  expect(
    await fixture.challengeSendCount() === 2,
    "Fallback must count against the shared send allowance"
  );
  expect(
    await fixture.reportReleaseCount() === 0,
    "Login verification alone must not release the report"
  );
}
```

The 30-second advance is a controlled test delay. It isn't an observed delivery time, an expiry recommendation, or a suggested fallback timeout. Configure the fixture so the replacement remains valid at verification. Repeat with the channel order reversed, then vary arrival order and expiry independently. A real-channel probe can measure delivery timing later; this test first establishes what the application must do regardless of that timing.

Ship the smallest slice: one report type, one approved destination, and the replacement rule. Keep the channel adapter behind an application-owned interface so the same failure schedule can exercise either transport. This requires a little fixture work up front, but it makes changing a transport less likely to change account policy accidentally.

## Measure uncertainty before expanding the rollout

There are two mail questions in this workflow: can a login message reach the customer, and can the authorized report attachment reach its approved address? RFC 7489 describes DMARC policy and identifier alignment involving SPF or DKIM. Inspect that evidence for the sending domains. It doesn't certify inbox placement, code entry, or report receipt. The Mail Privacy Protection guide describes protection of mail activity information, which is another reason to use an explicit application action as the login-completion signal instead of treating passive mail activity as proof.

Accepted isn't received.

For each channel order and regional cohort, measure time to verified login, rejection of superseded codes, abandonment, fallback use, and sends per completed login. Follow the same journey through report authorization, attachment queueing, submission, and whatever delivery evidence the mail path exposes. Preserve unknown outcomes in the denominator and report them separately. A worker that loses the submission response needs reconciliation; blindly retrying can produce another attachment even when the first submission succeeded. Internal job deduplication alone doesn't prove exactly-once external delivery.

Before rollout, run the failure schedule against the deployed storage and worker configuration, especially concurrent sends and uncertain attachment submissions. Keep an explicit path for unresolved journeys rather than silently recording success. The operational trade-off is more state and reconciliation work in exchange for a completion metric that means something.

Track challenge attempts per verified login, attachment attempts per authorized release, and generation tokens per report journey. Retaining the same generated artifact during retries is a design choice that avoids asking the model to repeat already completed work; its retention policy still needs to match the report's sensitivity. These counters expose avoidable work without making changing unit prices the decision criterion.

Copy the policy only after choosing an acceptable completion deadline, an allowance for unresolved deliveries, and recovery requirements for the report workflow. Then check whether each cohort meets them without loosening authorization. The useful result is a customer receiving the correct attachment after an authorized login, with delayed codes and uncertain submissions still accounted for.

## Sources

References:

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- Mail Privacy Protection guide: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
