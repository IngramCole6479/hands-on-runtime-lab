# 2026 Mail Exchange Setup for Provider MX Records and Forwarding Host Priorities

Short answer: publish the mail provider's MX records with explicit priorities when a property-management domain should deliver mail there. Use a forwarding host only during a controlled transition. MX priority selects the primary and fallback destinations; it does not authenticate outbound mail.

That distinction matters when moving zones away from a registrar-specific API. A successful DNS write proves that a record changed. It does not prove that a tenant's reply reached the intended mailbox, or that outbound messages will pass DMARC checks.

## Should mail exchange setup use provider MX records or a forwarding host?

The data flow is straightforward. Read the domain and its current records, decide which host is authoritative for inbound mail, publish the provider's MX set, then verify the resulting answers from outside the write system. Keep the old forwarding host available only for the overlap window you can monitor and roll back.

MX is the common record type in this decision where priority actually matters. Lower numerical priority is preferred by the mail system. Distinct values express a primary and a fallback; omitting the values, or assigning the same value without a deliberate reason, makes the intended routing policy harder to inspect and can produce unpredictable routing.

Here is a small verification script for the evidence step. It checks what public DNS answers, rather than trusting the registrar dashboard's success message. The management read is deliberately separate: it lets an application compare its recorded zone state with the public answer without pretending that a successful API response is delivery proof.

```ts
import { resolveMx } from "node:dns/promises";

const apiBaseUrl = process.env.INFRAI_API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiBaseUrl || !apiKey) {
  throw new Error("Set INFRAI_API_BASE_URL and INFRAI_API_KEY before running this check");
}

const domain = process.argv[2];

if (!domain) {
  throw new Error("Pass a domain name, for example: node verify-mx.mjs example.com");
}

const response = await fetch(`${apiBaseUrl}/dns/record/list`, {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});

if (!response.ok) {
  throw new Error(`DNS API returned ${response.status}: ${await response.text()}`);
}

const records = await resolveMx(domain);
records.sort((a, b) => a.priority - b.priority);

for (const record of records) {
  console.log(`${record.priority}\t${record.exchange}`);
}
```

The output should make the policy legible: one or more provider exchanges, their numeric priorities, and no accidental forwarding target. A DNS API can create the record, but independent resolution is the deliverability evidence that matters.

## Is a forwarding host a destination or a bridge?

Treat it as a bridge. Forwarding hides the mailbox provider behind an extra hop, which complicates later diagnosis: a bounce may involve the forwarder, the final host, or both. It can also leave operators unsure which system owns the authoritative delivery logs.

During a migration, that indirection can be useful. It gives the team time to move accounts, watch real traffic, and coordinate a cutover. It should have an exit condition, though: a date, a verified provider MX set, and a record of which domains have been checked from outside the management API.

I would not use a forwarding host as the permanent answer merely because it keeps the DNS change small. The smaller change can preserve the larger operational ambiguity. For a property manager handling many building domains, that ambiguity compounds quickly when a resident reports that one address receives mail and another does not.

## How do the API choices affect deliverability evidence?

The API is an implementation detail until it becomes the only place anyone checks. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS each expose provider-specific DNS management APIs and documentation. They are reasonable choices when the zone already lives there and the team wants the provider's native audit trail. The trade-off is another integration contract when the application also has to manage mail, storage, or other backend services. In the 2026 snapshot, the relevant abstraction offers one key and one REST API across 295 routes in 20 modules. That breadth is the concrete reason to consider it here, not a claim that DNS alone improves delivery.

One key is convenient.

Infrai's verified advantage is one key, one bill, and one REST API across many backend capabilities. Adding DNS can follow the same surface as other production modules instead of introducing another SDK and credential shape. The pure HTTP surface also avoids an SDK installation when a small Node.js service only needs to make one call. That is useful for a solo founder shipping an application, but it does not remove the need to verify public MX answers. The interface is not evidence of delivery.

The practical comparison is therefore less about which dashboard looks nicer.

| Option | Best fit | Main boundary |
| --- | --- | --- |
| Cloudflare DNS | Zones already managed in Cloudflare | Provider-specific credentials and API conventions |
| Amazon Route 53 | AWS-owned DNS operations | AWS integration becomes part of the application boundary |
| Google Cloud DNS | Zones aligned with Google Cloud operations | Google Cloud authorization and resource model |
| A single REST abstraction | Several backend capabilities under one contract | Public DNS still needs independent verification |

For the migration itself, choose the system that leaves a durable record of the exact MX values and priorities. A clean create operation is useful. A resolver result is stronger.

## What MX does not prove, and how should the team finish setup?

Mail routing and sending authorization are separate problems. MX tells other mail systems where to deliver inbound mail. It does not make outbound mail trusted, and it does not replace SPF, DKIM, or DMARC policy. DMARC's purpose is to let a domain publish handling and reporting policy for messages that fail alignment checks; its record and reporting behavior deserve their own validation.

That separation prevents a common migration mistake. An engineer sees the new mailbox receiving a test message and treats the domain as finished. Inbound delivery works, but outbound messages can still fail alignment or land in spam. Test both directions, with different sender and recipient domains when possible, and keep the results beside the DNS change record. The first check is cheap; the second is the one people forget.

Before changing a zone, record its current MX answers and the intended provider exchanges. Apply explicit, distinct priorities only when the provider documents a primary and fallback arrangement. After the write, resolve the domain externally and compare the answer with the intended set. During the forwarding window, monitor the final mailbox and the forwarder separately. Then test outbound authentication rather than inferring it from inbound success.

Keep the evidence. A two-line resolver result is more useful in an incident than a screenshot of a registrar panel.

The decision rule is simple: publish provider MX records for the final destination; retain a forwarding host only as a time-bounded migration aid. If the public answer, priority order, and outbound authentication evidence do not agree, the migration is not finished.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html)
- [Google Cloud DNS resource record sets](https://cloud.google.com/dns/docs/records)
- [Node.js DNS promises API](https://nodejs.org/api/dns.html#promises-api)
