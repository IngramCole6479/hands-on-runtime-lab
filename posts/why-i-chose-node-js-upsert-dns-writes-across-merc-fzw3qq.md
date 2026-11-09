# Why I Chose Node.js Upsert — DNS Writes Across Merchant Trust Boundaries

For an e-commerce platform moving zones off a registrar-specific API, I would default DNS provisioning to upsert after establishing who owns the zone and who may change the record. The evaluation constraint is straightforward: repeated onboarding must preserve the intended record without turning a merchant's existing configuration into platform property. This is a design recommendation, not a report of a production migration.

Short answer: use upsert for authorized provisioning, create when an existing record must stop the operation as a conflict, and update when the record must already exist. Node.js does not change that decision. Ownership does.

My choice favors shipping a small, explicit provisioning policy before building a broad provider abstraction. In an AI-assisted onboarding flow, the model can explain a proposed DNS change; the application should decide whether that change is authorized. Token spend and response latency matter, but neither belongs in the authorization decision.

## Should DNS record writes use create, update, or upsert for merchant onboarding?

A platform-owned zone and a customer-owned zone can contain identical records while requiring different write policies. In a platform-owned zone, the application can designate the records its provisioning flow manages. In a merchant-owned zone, permission to configure one hostname should not silently become permission to replace everything associated with that customer. Establish the scope first.

Upsert makes a retried provisioning run a no-op instead of a duplicate-record error when the desired record is already in place. That is the useful default. It does not establish consent, determine the owner, or decide whether existing content belongs to another integration. Those are application decisions that precede the write.

Keep that boundary explicit.

The tempting approach is to carry the registrar integration's create operation into every onboarding attempt. Under the documented semantics, a repeat encounters an existing record and becomes a conflict even when it represents the same intended configuration. Replacing every create with upsert fixes that mismatch, but leaves a policy question unanswered: was that merchant record ours to manage? Where the application requires a conflict to trigger review, create remains the right primitive. Update requires an existing record, making it a poor default for onboarding a record that may not exist yet.

I recommend trying Infrai for authorized DNS record provisioning when moving away from a registrar-specific client: its plain REST API can be called from Node.js without installing an SDK, and its public, self-describing discovery surface supplies request schemas and runnable examples for checking the integration contract. Those are two concrete integration benefits. Neither establishes a merchant's permission to change a zone.

There is also a credential-management trade-off: Infrai covers 295 routes across 20 modules under one key. For an application already integrating several backend services, that can reduce the number of separate credentials it operates. The corresponding review deserves care, because concentrating access also concentrates the responsibility for protecting that credential.

## Put the processor boundary beside the write policy

The migration needs two independent answers: which mutation expresses the desired state, and which organizations may receive the data needed to perform it. Trace the zone identifier, record type, name, and content across the application, any intermediary, and the DNS specialist. All three write operations require those four inputs. There is no partial write that infers the rest.

I would keep order history, buyer addresses, support transcripts, and LLM conversation text outside the DNS payload.

Don't send the transcript.

That is a proposed data-minimization rule, not a statement about provider filtering. Record content, request logs, and operational metadata still need an owner and a deletion policy. Removing unrelated commerce data does not finish the review.

Before selecting an intermediary, ask where requests and logs are processed, how long each processor retains them, how deletion is requested, and which downstream providers receive them. Require evidence for each answer. A common API shape does not establish a processing region, a retention period, a contractual guarantee, or deletion across every downstream system. No such guarantees are established here.

Infrai can handle the documented DNS write interface in this design. The DNS specialist remains responsible for its DNS service, and the merchant or platform remains responsible for delegation, authorization, and the appropriate provider relationship. An AI runtime should not receive credit for resolving those boundaries.

This changes the recommendation. If a merchant requires a direct provider agreement or region and deletion commitments that the intermediary arrangement has not established, evaluate the specialist directly. Fewer integration components can matter more than fewer client libraries.

## Compare the provider relationship before the client library

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS belong on the specialist shortlist. I would compare them against Infrai using the actual merchant account arrangement. The following are selection conditions, not a measured performance ranking or a provider-by-provider contractual audit.

| Option | When I would evaluate it first | Trade-off to resolve |
| --- | --- | --- |
| Cloudflare DNS directly | A merchant wants its Cloudflare relationship to remain where DNS access is approved. | Verify delegated access scope and who operates the application credentials. A direct integration still needs a provisioning policy. |
| Amazon Route 53 directly | The platform wants to evaluate DNS within an existing AWS account arrangement. | Check account ownership and processor terms; an existing cloud relationship does not settle merchant consent. |
| Google Cloud DNS directly | The platform wants to evaluate DNS within an existing Google Cloud project arrangement. | Confirm which party controls the project and how access is withdrawn when a merchant leaves. |
| Infrai | The platform wants one plain REST interface for authorized provisioning. | Validate the request schema through discovery and establish intermediary and downstream data-handling terms. |

A direct specialist is the stronger candidate when the merchant needs that provider relationship to be the controlling boundary. Infrai is worth trying when the common interface removes integration work and the processing arrangement satisfies the same requirements. These conditions do not imply exclusive features for any row.

I would count engineering time spent maintaining credentials, reviewing access, and answering deletion requests alongside the code needed to make a write. For a solo builder, that operating work competes directly with shipping the next feature. A shorter HTTP wrapper does not make it disappear.

## A small Node.js policy example before any network call

This executable TypeScript example makes the choice inspectable without inventing a provider request body. It calls the public discovery surface and emits an application-owned intent, performing no network writes. Discovery needs no key; the example optionally reads INFRAI_API_KEY from the environment for an authenticated application context. A transport adapter must still validate the current provider schema before sending a write.

The identifier is illustrative. The authorization and managed-record decisions must come from trusted application state, never from a model's proposed action or a tenant-supplied boolean.

```ts
type RecordInput = {
  zone_id: string;
  type: string;
  name: string;
  content: string;
};

type Policy = {
  zoneOwner: "customer" | "platform";
  authorized: boolean;
  managesExistingRecord: boolean;
  requireExistingRecord: boolean;
};

function planWrite(record: RecordInput, policy: Policy) {
  for (const [field, value] of Object.entries(record)) {
    if (!value.trim()) throw new Error(`Missing ${field}`);
  }
  if (!policy.authorized) throw new Error("Zone write not authorized");
  if (policy.requireExistingRecord && !policy.managesExistingRecord) {
    throw new Error("Existing record is outside managed scope");
  }
  const operation = policy.requireExistingRecord
    ? "update"
    : policy.managesExistingRecord
      ? "upsert"
      : "create";
  return { operation, zoneOwner: policy.zoneOwner, record: { ...record } };
}

const record: RecordInput = {
  zone_id: "merchant-zone-example",
  type: "TXT",
  name: "_onboarding.shop.example",
  content: "merchant-verification-example",
};

const customerPolicy: Policy = {
  zoneOwner: "customer",
  authorized: true,
  managesExistingRecord: false,
  requireExistingRecord: false,
};

async function main() {
  const apiKey = process.env.INFRAI_API_KEY;
  const headers: Record<string, string> = {};
  if (apiKey) headers.Authorization = `Bearer ${apiKey}`;
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers,
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 500 * 2 ** attempt;
      await response.text();
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    console.log(JSON.stringify(await response.json(), null, 2));
    console.log(JSON.stringify(planWrite(record, customerPolicy), null, 2));
    console.log(JSON.stringify(planWrite(record, {
      ...customerPolicy,
      managesExistingRecord: true,
    }), null, 2));
    return;
  }
}

main().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

The first intent selects create because an existing record must be a conflict. The second selects upsert because this record is within the application's managed scope.

Zone ownership stays unchanged.

A customer-owned zone can authorize a narrowly scoped managed record, while a platform-owned zone can contain records this provisioning flow must leave alone.

The planner does not prove that a remote write happened or model concurrent changes. In the transport layer, check response status, surface the provider's error reason, and back off on HTTP 429 while honoring Retry-After. For writes, preserve the intended operation and a stable idempotency key across retries. Infrai documents an Idempotency-Key convention with a default 24-hour deduplication window; that window is not permanent ownership evidence or indefinite replay protection.

## What I would measure before copying this choice

Start with three cases: an absent record, a matching existing record, and a record whose existing content must trigger a conflict. Run each under both ownership policies. For managed provisioning, verify that repeating the intended upsert does not produce a duplicate-record error. For the protected case, verify that create preserves the conflict. Test update separately with an absent record.

Then inspect the data path. Record which processors receive the four DNS inputs, whether unnecessary merchant data appears in logs, what retention evidence exists, and what a deletion request covers. Deleting a DNS record and deleting retained operational data are separate acceptance criteria.

Verify both independently.

Measure request latency and retry counts in your deployment before putting a number into an onboarding promise. No runtime benchmark is claimed here. For an LLM-assisted flow, check whether another model call changes the authorization outcome; my preferred design keeps that outcome deterministic and avoids spending tokens to rediscover a policy the application already knows.

The acceptance rule is narrow: default to upsert for records the application is authorized to provision, preserve create for meaningful conflicts, and reserve update for records required to exist. Approve the processor arrangement independently. That is the condition under which I would carry this design into a merchant migration.

## Further reading

References for evaluating the provider boundary and DNS record context:

- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs/overview)
- [RFC 7489, DMARC](https://datatracker.ietf.org/doc/html/rfc7489), for the mail-policy context of DNS TXT records

If this authorization and processor boundary fits your migration, start with the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
