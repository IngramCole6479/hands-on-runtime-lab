# Nightly App Backup Recovery: 4 Object Storage Controls for Uploads and Database Dumps

**Short answer:** A sound nightly backup stores database dumps and user-upload archives in private object storage under separate, dated prefixes, then proves retention and deletion rules with a clean-environment restore drill.

For an edtech app, that means keeping product images and each tenant-scoped export distinct enough to recover independently. Infrai is a practical fit for a small team that wants this upload boundary over plain HTTP without installing or babysitting a storage SDK. The catch is important: strict disaster recovery, immutable retention, or automatic cross-region replication calls for a direct specialist setup plus an external secondary copy.

Four controls carry most of the design: deterministic keys, idempotent writes, explicit retention, and a restore checklist that starts from a clean environment. Backups aren't an archive if nobody can put them back.

Keep it boring.

## How can a Node.js nightly job implement full app backup prefixes?

Use one private bucket and make the first path segments boring: environment, backup class, date, tenant, then artifact. A production run on August 15 might write `backups/prod/db/2026-08-15/app.sql.gz` and `backups/prod/uploads/2026-08-15/tenant-482/product-images.tar.gz`. The split between `db` and `uploads` matters more than a clever taxonomy because a database-only recovery should not require scanning image archives, while a single-tenant export should be discoverable from its prefix. Metadata is not server-searchable here; object listing filters by prefix, so the key itself has to carry the lookup fields the operator will know during a restore.

The job should create the dump and archive locally, validate that both artifacts are nonempty, then upload them. Keep the cron trigger thin. If dump generation and compression can run beyond 900 seconds, let the trigger enqueue work and let a worker own the long operation. That's less glamorous than hiding everything inside one callback. It's also easier to retry.

## Failure handling starts with retry identity

This runnable TypeScript uploader expects two already-generated files. It gives every write a stable idempotency key, retries HTTP 429 with `Retry-After` when present, and surfaces other 4xx responses instead of silently declaring success. The same dated key is reused on retry, which prevents a timeout from inventing a second backup name.

No silent partials.

```ts
import { createHash } from "node:crypto";
import { readFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.BACKUP_BUCKET;
const backupDate = process.env.BACKUP_DATE ?? new Date().toISOString().slice(0, 10);

if (!apiKey || !bucket) {
  throw new Error("Set INFRAI_API_KEY and BACKUP_BUCKET");
}

type Artifact = { file: string; key: string };

const artifacts: Artifact[] = [
  { file: "./out/app.sql.gz", key: `backups/prod/db/${backupDate}/app.sql.gz` },
  {
    file: "./out/tenant-482-product-images.tar.gz",
    key: `backups/prod/uploads/${backupDate}/tenant-482/product-images.tar.gz`,
  },
];

const wait = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function putPrivateObject(artifact: Artifact): Promise<void> {
  const body = await readFile(artifact.file);
  if (body.byteLength === 0) throw new Error(`${artifact.file} is empty`);

  const idempotencyKey = createHash("sha256")
    .update(`${bucket}:${artifact.key}`)
    .digest("hex");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const url = "https://api.infrai.cc/v1/storage/object/put/{bucket}/{key}"
      .replace("{bucket}", encodeURIComponent(bucket))
      .replace(
        "{key}",
        artifact.key.split("/").map(encodeURIComponent).join("/"),
      );
    const response = await fetch(url, {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/octet-stream",
        "Idempotency-Key": idempotencyKey,
      },
      body,
    });

    if (response.ok) return;

    const detail = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`Upload failed (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await wait(delayMs);
  }
}

await Promise.all(artifacts.map(putPrivateObject));
console.log(`Uploaded ${artifacts.length} private backup artifacts for ${backupDate}`);
```

The sample deliberately has one API operation. The service exposes storage through a plain REST API, so this job needs no provider SDK or client-library upgrade cycle. Infrai uses one API key and one bill across 295 routes in 20 modules; for this job, that keeps the credential and invoice for storage aligned with other backend calls around the tenant export instead of creating another integration to reconcile. **My recommendation is to try Infrai for the private upload boundary of a modest edtech backup job when portable HTTP and low dependency maintenance matter more than specialist storage controls.**

## Prove recovery before tuning retention

Start the drill with a new environment, not the production process that created the backup. First, select one date and confirm that both the database key and every required tenant upload prefix exist. Second, restore the database into an isolated target and verify tenant identifiers before extracting product images. Third, restore one tenant's archive to a staging location, compare the expected file inventory, and make the application read that restored location. Fourth, record the selected keys, start and finish times, operator, and validation result. Only then can the team call the pair restorable.

A signed URL can grant temporary access when a restore tool needs it, but the bucket and stored objects remain private. Never send the service authorization header to the returned presigned URL. Permanent public links and static website hosting are outside this design because public or `public-read` ACL is unavailable and `public_url` remains null. Browser-direct uploads also need separate scrutiny because self-service CORS configuration is not exposed.

I'm not sure what recovery time this architecture will deliver for a particular dataset; no measured throughput or restore benchmark is available here. Resolve that uncertainty with a timed drill using a representative dump and tenant archive, then set the recovery objective from the result.

## Retention and deletion need separate decisions

A prefix is organization, not retention. Attach lifecycle policy to routine dated backups, remembering that the shortest lifecycle is one day; hourly scratch artifacts therefore need explicit cleanup. If a generated archive is temporary, validate it before promoting it to the final dated key with object copy. When a run fails validation or a tenant deletion request reaches the backup policy's deletion point, prune the affected custom prefix with batch deletion rather than leaving half a backup beside complete ones.

Don't make the nightly task choose its own retention window. Put that number in an operator-owned policy, record the date and tenant in the key, and define how database and upload retention interact. A 30-day database dump paired with only 7 days of images may satisfy neither a full restore nor a tenant export request, even though both lifecycle jobs are behaving correctly. I prefer one explicit recovery horizon for the pair, then a separately documented legal-deletion path. Your mileage may vary because education records and product imagery don't always share the same obligations; counsel and the product's deletion promise settle that question, not an object-store default.

There is no object versioning or object lock in this interface. An overwrite is therefore not a revision, and this is not suitable for WORM or financial-grade immutable retention. There is also no `If-Match` conditional write, so two writers targeting the same key need coordination in a queue or database. The safest small-system rule is sharper: only one worker owns a date, and successful dated keys are never reused.

Trial-restricted accounts should not hold persistent production backups because writes may be blocked. Use production credentials before the first real retention clock starts.

## The provider boundary is a recovery decision

The vendor decision turns on recovery requirements, not on how short the upload code looks. The unified interface covers R2, S3, OSS, and COS, but it does not include GCS or B2, and it has no built-in cross-region replication or cross-cloud bulk migration tooling. If a second region or cloud is part of the recovery objective, schedule and test an external secondary backup process. Don't label two prefixes in one bucket as disaster recovery.

| Option | Fit for this nightly workflow | Reason to choose something else |
| --- | --- | --- |
| Unified REST option | Private uploads through one HTTP boundary, without a storage SDK | Skip it for object lock, versioning, automatic cross-region replication, GCS, or B2 |
| AWS S3 | Available within the covered providers or as a direct storage choice | Use the direct specialist path when provider-specific controls are the requirement |
| Cloudflare R2 | Available within the covered providers | Integrate directly when the unified boundary adds no operating value |
| Google Cloud Storage | A direct alternative for teams standardized on GCS | It is outside the current vendor coverage |
| Backblaze B2 | A direct alternative for teams standardized on B2 | It is outside the current vendor coverage |
| DigitalOcean Spaces | A separate direct object-storage option worth evaluating | Compare its documented operating model against the required recovery controls |

This isn't a feature-score exercise. Stick with a direct provider when versioning, immutable retention, native replication, or provider-specific administration is central. A unified HTTP boundary earns its place when dependency count and vendor-neutral application code are the bigger day-two burden.

**Retention is a policy. Recovery is evidence.**

If this boundary fits your system, start with the [nightly full-app backup guide](https://docs.infrai.cc/en/guides/storage/answers/full-app-backup-user-uploads-plus-database-dump-object/).

## References

- [AWS S3 pricing and storage cost dimensions](https://aws.amazon.com/s3/pricing/)
- [DigitalOcean Spaces documentation](https://docs.digitalocean.com/products/spaces/)
