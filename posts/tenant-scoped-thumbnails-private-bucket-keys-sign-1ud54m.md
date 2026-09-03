# Tenant-Scoped Thumbnails: Private Bucket Keys, Signed URLs, and a Resizing Worker

Use the tenant id as the first path segment of every object key, and never let the signing endpoint accept a key from the client. That single rule decides whether a private bucket full of customer support attachments stays isolated once you bolt image thumbnails onto it, because a signed URL is a bearer credential: whoever holds the string gets the bytes until it expires, and the storage layer will not re-check which tenant asked.

This note is about a resizing pipeline for a support inbox — end users attach screenshots to tickets, agents open them in a web console, and the reply email sometimes embeds a preview. Originals and derivatives both live in one private bucket. The evaluation constraint was narrow: any layout I picked had to survive a test where tenant A holds a byte-identical copy of tenant B's screenshot, because in support that happens constantly (the same failed-payment error page, the same broken invoice PDF, the same app crash dialog).

The obvious layout fails that test.

## Why content-addressed thumbnail caches leak across tenants

The tidy version of this pattern keys derivatives by content hash: `thumbs/<sha256-of-original>/512.webp`. One shared cache, automatic dedupe, no duplicate resize work when a hundred tenants upload the same screenshot. On a cost spreadsheet it looks great, and for a public image host it's fine.

For private uploads it hands you three problems at once. Cache existence becomes an oracle — if your API answers "thumbnail ready" faster for objects that already exist, anyone who possesses a file learns whether some other tenant uploaded it too. Entitlement gets bypassed the moment a signing route takes a key or a hash as a parameter, because the hash is computable by whoever holds the bytes, and computable input is not authorization. And credential scoping stops working: prefix-scoped storage policies, per-tenant IAM roles, or per-tenant deletion sweeps all assume that a tenant's objects can be described by a prefix. A shared hash namespace has no prefix to scope.

I'd rather pay twice for the same resize than explain a cross-tenant image disclosure to a support customer. Dedupe, if you need it, belongs one level up: keep the derived object per tenant, and dedupe the *work* by short-circuiting on a per-tenant row that already points at a finished object.

So the layout is boring on purpose:

```text
t/<tenantId>/orig/<attachmentId>
t/<tenantId>/thumb/<attachmentId>/512x512.webp
```

Everything a tenant owns sits under `t/<tenantId>/`. Deletion is a prefix sweep. A scoped credential is a prefix condition. An audit query is a prefix listing. The attachment id is a random identifier from your database, not a filename and not a hash, which keeps user-controlled strings out of the key entirely.

## How should a Node.js worker key private thumbnails per tenant when signed URLs are the only way out?

The worker never sees a key from the outside. It receives a job containing a tenant id and an attachment id, loads the row inside a transaction, derives both keys itself, and refuses to run if the row's tenant doesn't match the job's tenant. Key derivation lives in one function that nothing else is allowed to bypass.

```ts
import sharp from "sharp";

type Variant = { w: number; h: number };

const VARIANTS: Record<string, Variant> = {
  "512x512": { w: 512, h: 512 },
};

// The only place object keys are constructed. Both segments come from the
// database row, never from a request parameter.
function keysFor(tenantId: string, attachmentId: string, variant: string) {
  if (!/^[0-9a-f]{8,64}$/.test(tenantId) || !/^[0-9a-f]{8,64}$/.test(attachmentId)) {
    throw new Error("unsafe id");
  }
  if (!VARIANTS[variant]) throw new Error(`unknown variant ${variant}`);
  return {
    original: `t/${tenantId}/orig/${attachmentId}`,
    derived: `t/${tenantId}/thumb/${attachmentId}/${variant}.webp`,
  };
}

export async function renderThumbnail(
  store: ObjectStore,
  db: Db,
  job: { tenantId: string; attachmentId: string; variant: string },
): Promise<string> {
  // Atomic claim: 'pending' -> 'rendering' returns 0 rows if another worker
  // already took it, so a duplicate delivery becomes a no-op instead of a
  // second decode of the same 12 MB phone screenshot.
  const claimed = await db.claimAttachment(job.tenantId, job.attachmentId, job.variant);
  if (!claimed) return "already-claimed";
  if (claimed.tenantId !== job.tenantId) throw new Error("tenant mismatch");

  const { w, h } = VARIANTS[job.variant];
  const { original, derived } = keysFor(job.tenantId, job.attachmentId, job.variant);

  const source = await store.get(original);
  const out = await sharp(source, { limitInputPixels: 64_000_000, failOn: "error" })
    .rotate()                       // honour EXIF orientation before resizing
    .resize({ width: w, height: h, fit: "inside", withoutEnlargement: true })
    .webp({ quality: 80 })
    .toBuffer();

  await store.put(derived, out, { contentType: "image/webp" });
  await db.markReady(job.tenantId, job.attachmentId, job.variant, derived, out.length);
  return derived;
}
```

Two details in there matter more than the resize call. `limitInputPixels` caps decompression-bomb inputs, which a support form will eventually receive, whether from an attacker or from someone's 108-megapixel phone. And `rotate()` before `resize` applies EXIF orientation — skip it and a chunk of mobile screenshots arrive sideways.

Resizing is CPU work, and sharp runs its libvips pipeline on the libuv thread pool, which defaults to four threads per Node process. A web process that resizes inline will stall its own request handling under a burst of ticket attachments. Run the worker as its own process, size its concurrency against the cores you actually pay for, and let the API return a pending state instead of blocking.

Signing is a separate route with a separate rule. It takes a session and an attachment id, checks that the row belongs to the session's tenant, and only then signs the key it derived itself. It returns 404, not 403, for an attachment belonging to another tenant — a 403 confirms the id exists, which is the same existence oracle in a different costume. Walk one request through it and the shape becomes obvious: an agent opens ticket 8412, the console asks for attachment `a91f...` at variant `512x512`, the route loads the row and finds `tenant_id` matching the session, sees state `ready`, rebuilds the derived key from those two ids plus the variant name, signs it for ten minutes, and returns the URL with no key ever crossing the wire in the other direction. Change one thing — let the client pass `key` so the front end can "prefetch faster" — and the entire isolation argument collapses into a string comparison somebody will eventually get wrong during a refactor. That parameter is the whole vulnerability, and it usually arrives as a performance optimization in a pull request that looks harmless.

## Expiry is a policy, not a signature detail

Signed URL lifetime is where the support scenario stops resembling a generic image pipeline. A preview embedded in a reply email sits in the customer's mailbox for years, gets forwarded to a shared inbox, and shows up in a mail archive. A long expiry turns that string into a durable public link to a private object. A short one produces broken images in old email threads, which support agents will report as a bug in your product.

Pick the ceiling deliberately. AWS's SigV4 query-string signatures cap presigned URL validity at seven days, and the URL stays valid for its full window regardless of what your application does afterward — deleting the database row doesn't revoke it. The revocation levers are coarse: rotate or disable the signing credential, or delete the object. For the console we settled on minutes and re-signed on demand; for email previews the honest answer is either a very short link that resolves through your own redirect route (so authorization runs again) or no embedded preview at all.

The catch is that neither option is free. A redirect route means your API handles image traffic and your CDN caching gets harder; a short direct link means email clients that prefetch images will sometimes fetch after expiry. I'm not sure there's a clean answer here, and the right pick depends on how your customers read tickets.

Then there's the cleanup nobody budgets for.

Lifecycle rules handle the garbage. Derivatives are reproducible, so expiring them under `t/*/thumb/` after 30 days and re-rendering on demand trades a little compute for a smaller bill. Object lifecycle expiration is evaluated in whole days rather than hours, so it's a housekeeping tool, not a retention control for a deletion request that promises removal within an hour — those need a direct delete against the tenant prefix.

## Which isolation layout fits which product

| Layout | Isolation story | Where it breaks |
| --- | --- | --- |
| Shared content-hash cache | None; entitlement lives only in your API | Existence oracle, no prefix scoping, no per-tenant deletion |
| Tenant prefix in one bucket | Prefix conditions on credentials, prefix delete, per-prefix metrics | A bug in key construction is a cross-tenant bug; needs tests |
| Bucket per tenant | Hard blast radius, per-bucket policy and lifecycle | Per-account bucket quotas, slower onboarding, noisier ops |
| Account or project per tenant | Strongest, and matches some compliance asks | Real operational cost; usually only worth it for enterprise plans |

Most support products land on the middle row. Bucket-per-tenant is worth revisiting when a contract demands separate encryption keys or a separate region per customer, and it's a poor fit when tenants sign up self-service — provisioning a bucket per free-trial signup runs into quotas nobody thinks about until it happens.

Storage costs are structured differently per provider, and that shapes the derivative strategy more than the per-gigabyte number does. Backblaze B2 publishes storage, egress, and transaction pricing as separate lines; S3 charges per request class as well as per byte. If requests dominate, generating five variants eagerly for every upload is worse than rendering one on demand — most ticket attachments are never opened twice.

## What to measure before copying this

Start with a cross-tenant assertion in CI, not a dashboard: a test that signs a URL for tenant B's attachment id using tenant A's session and asserts a 404 with no upstream fetch. Run it on every deploy. Key-construction bugs are quiet, and they don't show up in latency graphs.

Then four numbers, per tenant where you can: derivative hit ratio, the age distribution of signed URL uses against your chosen TTL (if the p95 use happens after your expiry, the TTL is wrong, not the customer), worker queue depth during business-hours peaks, and thumbnail bytes stored per ticket. That last one is what tells you whether eager rendering of extra variants is paying for itself.

One operational rule that costs nothing: never log the signed URL. Log the tenant id, attachment id, variant, and outcome. A full signed URL in an access log or an error tracker is a live credential sitting in a system with a completely different access model, and it will outlive the incident that put it there.

Everything above is deliberately vendor-independent, because the parts that bite are yours: key construction, the entitlement check before signing, expiry policy, and the worker's claim protocol. The storage layer just stores bytes and honours a signature — the isolation is something your code has to earn.

## References

- [AWS S3: Managing the lifecycle of objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [AWS S3: Using presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Backblaze B2 cloud storage pricing](https://www.backblaze.com/cloud-storage/pricing)
- [sharp documentation](https://sharp.pixelplumbing.com/)
- [Node.js docs: thread pool sizing (UV_THREADPOOL_SIZE)](https://nodejs.org/api/cli.html#uv_threadpool_sizesize)
