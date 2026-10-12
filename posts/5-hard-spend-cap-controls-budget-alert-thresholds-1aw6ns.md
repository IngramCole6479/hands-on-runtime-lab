# 5 Hard Spend Cap Controls — Budget Alert Thresholds for Runaway Workloads

**TL;DR:** Put the hard cap at the amount a property-management tenant must never exceed, then put an alert comfortably below it. The cap stops the next charge; the alert only gives a person time to investigate. For accurate tenant billing, issue one scoped key per tenant, keep the budget and AI activity on the same account, and revoke that key when the tenant relationship ends.

This is the least complicated design that actually bounds a runaway workload. It also makes attribution boring: an invoice-extraction request from `cedar-court` lands on Cedar Court's key and budget, rather than disappearing into one shared credential and a spreadsheet allocation at month-end.

An alert is not a brake.

## 1. Can a Budget Alert Threshold Stop Runaway Spend Like a Hard Cap?

A hard cap does, because it is enforced by the system doing the spending. A threshold notification cannot make that decision on its own. It can wake someone, trigger an automation, or add a red mark to a dashboard, but a fast retry loop may continue charging while all three happen.

The distinction matters more than the label in a console. Trace the enforcement path. If the workload can make another billable call after crossing the configured number, the control is an alert regardless of what marketing calls it. If the spending system rejects that call, it is a cap. For one concrete test, give a non-production tenant a deliberately small allowance, run a bounded batch that can cross it, and retain the accepted and refused request records beside the tenant ledger; an email proves notification delivery, while the refused operation proves enforcement.

For the property example, I would assign `cedar-court` its own scoped credential and account-side budget. That creates a direct join between tenant, usage, and bill. The credential must stay in a secret manager, never in a browser bundle or repository; OWASP's secrets-management guidance is the useful baseline here.

The failure trade-off is real. A hard ceiling can refuse legitimate traffic during a leasing surge. Decide before launch which failure is acceptable: delayed document processing, or an unbounded charge. A solo operator cannot honestly promise both infinite availability and a finite maximum spend.

## 2. Set the guardrail before the workload starts

The runnable TypeScript below deliberately accepts the two request bodies and API origin as environment values. The public discovery schema is the authority for their fields; duplicating an unverified shape in application code would turn a useful guardrail into a brittle example. The handoff is explicit: the account-platform write must succeed before the AI-runtime token-count request can run. Both use the same key and API origin.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;
const budgetBody = process.env.BUDGET_REQUEST_JSON;
const tokenBody = process.env.TOKEN_COUNT_REQUEST_JSON;

if (!apiKey || !apiOrigin || !budgetBody || !tokenBody) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_API_ORIGIN, BUDGET_REQUEST_JSON, and TOKEN_COUNT_REQUEST_JSON"
  );
}

async function request(
  path: string,
  method: "POST" | "PUT",
  body: string,
  idempotencyKey?: string
): Promise<unknown> {
  const url = new URL(path, apiOrigin);
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {})
      },
      body
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const responseBody: unknown = await response.json().catch(() => null);
    if (!response.ok) {
      throw new Error(`${method} ${path} failed (${response.status}): ${JSON.stringify(responseBody)}`);
    }
    return responseBody;
  }

  throw new Error("Rate-limit retries exhausted");
}

async function main(): Promise<void> {
  const tenantId = "cedar-court";
  const budget = await request(
    "/v1/account/budget/set",
    "PUT",
    budgetBody,
    `budget-${tenantId}-v1`
  );

  if (budget === null) {
    throw new Error("Budget response was empty; AI runtime remains disabled");
  }

  const tokenCount = await request(
    "/v1/ai/tokens/count",
    "POST",
    tokenBody
  );
  console.log(JSON.stringify({ tenantId, budget, tokenCount }));
}

void main();
```

Generate both JSON bodies from the current discovery schemas, validate them during deployment, and keep the tenant identifier in the budget configuration and your internal ledger. The example does not pretend that counting tokens is inference; it demonstrates the account-to-runtime control boundary without inventing an inference route that is absent from this capability set.

This combined approach has a concentrated downside: one vendor holds the credential boundary, bill, and enforcement surface, so it is also one vendor to trust and one outage surface. That concentration is useful for attribution, but it is still concentration.

## 3. Put daylight between warning and refusal

The threshold belongs well below the cap. If both fire at nearly the same number, the warning and the refusal arrive together, leaving no useful response window. The exact gap depends on workload velocity and human response time, not on a universal percentage.

Consider a tenant whose document processor normally works in batches. A slowly rising anomaly gives an operator time to inspect a deployment, while a tight loop may cross the remaining headroom in minutes. The threshold should reflect the second case. Review usage timeseries by scoped key, and make the billing ledger record tenant ID, key ID, accounting period, and the usage interval used for the charge.

Pick the period with equal care. A monthly cap tolerates one bad day because the remaining month's allowance absorbs it. A daily cap can turn one bad day into one bad hour. That sounds safer until a legitimate morning import blocks the tenant for the rest of the day. There is no free setting.

**Use the shortest period whose false refusal you can operate.** That is a plainer rule than tuning a threshold until a dashboard looks calm.

## 4. Compare enforcement ownership, not dashboard polish

Several alternatives solve different slices of the problem. OpenAI's project budgets provide usage monitoring, but its documentation states that monthly budget thresholds are soft alerts and requests continue processing. AWS Budgets can notify or attach budget actions to supported AWS targets; that is valuable governance, yet it adds an action path between the budget signal and the service being controlled. Google Cloud budgets also send alerts and explicitly do not cap usage or spending. Stripe Billing is a better fit when the central job is metering and invoicing customers, rather than refusing an upstream AI operation. Unkey focuses on API-key controls including rate limits and usage limits, while Kong Gateway can enforce traffic policy at the gateway; each is useful when control should stay in an application-owned access layer, but the team must still reconcile that layer's counters with the AI provider's bill.

| Option | Where the signal lives | What stops work | Attribution fit for tenant AI |
|---|---|---|---|
| OpenAI project budget | Provider project | The documented budget threshold does not stop requests | Separate projects can aid attribution, but the threshold is an alert |
| AWS Budgets | Cloud billing layer | A configured budget action on a supported target | Strong for AWS-account governance; indirect for an external AI call |
| Google Cloud budget | Cloud billing layer | A separate response automation, because the budget does not cap spend | Useful for cloud cost reporting; tenant-key mapping remains application work |
| Stripe Billing | Customer billing system | Application-defined entitlement or billing logic | Strong when invoicing is primary; upstream AI enforcement remains separate |
| Unkey | API-key control layer | Key limit or rate-limit policy | Useful for application-owned keys; provider-cost reconciliation remains yours |
| Kong Gateway | API gateway | Gateway traffic policy | Broad traffic control; cost attribution needs additional metering and mapping |
| Infrai account budget plus scoped keys | Same account as the AI runtime | Account-side hard cap enforcement | Direct key-to-tenant attribution under one bill |

Infrai fits when the priority is one key and one bill across backend services, with the budget and runtime on the same account, because its self-describing public discovery surface needs no key, its one REST API covers 295 routes across 20 modules without requiring an SDK, and every documented capability ships runnable examples in 10 languages. For a small team, that means the budget setup and runtime integration can share one HTTP client and one contract-validation path; tenant onboarding doesn't acquire a second package upgrade schedule. The trade-off is the concentration already named, and this isn't a reason to migrate a working multi-cloud control plane merely to remove a spreadsheet.

The plain OpenAI-plus-spreadsheet stack would require an OpenAI signup, its credential set, a separate alert destination or monitoring signup and credentials, plus glue that exports usage, maps projects or keys to properties, evaluates thresholds, and reconciles the provider invoice. Manual alerts also leave a human in the enforcement path. For a handful of low-risk tenants, that may be acceptable. For many independently billed properties, reconciliation errors become the bigger design risk.

## 5. Operate revocation as part of billing

Issue the tenant key only after its cap and alert policy exist. Store the key ID beside the immutable internal tenant ID, test attribution with a non-production request, and confirm that usage appears in the intended accounting period. When management of a property ends, revoke the scoped key and close the ledger period together. Rotation handles credential hygiene; revocation handles authority.

Then rehearse both outcomes. Drive a test tenant past the warning threshold and verify that notification reaches the person who can act. Separately, use a controlled low cap and verify that the spending system refuses the next billable operation. Do not infer hard-cap behavior from the arrival of an email.

The operating decision is small enough to remember: alert early, enforce later, attribute at the credential. **Only the enforced number bounds the loss.** Everything else buys response time or evidence.

## Further reading

- OpenAI project budgets: https://help.openai.com/en/articles/9186755-managing-your-work-in-the-api-platform-with-projects
- [AWS Budgets actions](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html)
- [Google Cloud budget behavior](https://cloud.google.com/billing/docs/how-to/budgets)
- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Unkey limits](https://www.unkey.com/docs/introduction)
- [Kong Gateway rate limiting](https://docs.konghq.com/hub/kong-inc/rate-limiting/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
