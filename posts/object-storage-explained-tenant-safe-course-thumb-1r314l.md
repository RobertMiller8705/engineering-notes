# Object Storage Explained: Tenant-Safe Course Thumbnails, Resizing, and Signed Downloads

Short answer: use private object storage for course originals and generated thumbnails, resize in a worker or dedicated image service, and issue short-lived signed URLs only after the SaaS application checks the tenant and user.

For an edtech product, the least complex useful flow is also the safer one. The browser asks the application for permission to upload, then sends the large media bytes to a presigned storage URL instead of pushing them through the application server. A worker reads the private original, creates the requested image variants, and writes each thumbnail as another private object. The application database remains the source of truth for tenant ownership, dimensions, variant state, and access policy.

Storage is not the resizer.

## How should a SaaS app isolate private originals and signed thumbnail downloads?

Treat tenant isolation as a data-model decision before comparing vendors. Give each tenant a private bucket when the operational model permits it, or use an immutable tenant prefix in a shared private bucket when bucket-per-tenant management would be excessive. In either case, derive the bucket or prefix from the authenticated tenant record on the server. Never accept a bucket name or tenant prefix from the browser as proof of ownership.

A useful object layout is `originals/course_91/asset_7/source.png` beside `thumbs/course_91/asset_7/320x180.webp`. Keep `tenant_id`, `course_id`, the original key, width, height, format, and variant status in the database. Object listing can filter by prefix, but it cannot search server-side metadata, so a storage scan is the wrong authorization index and the wrong catalog for a thumbnail UI.

The upload sequence is small: authorize the instructor, allocate an asset ID, build its tenant-bound key, mint a presigned PUT, and return that temporary capability to the browser. The browser uploads directly. Once the application confirms the upload state, it queues resizing; the worker writes generated variants under `thumbs/`, and a later read request receives a presigned GET only after the same tenant check. Don't store the signed URL as durable state. Store the key.

This distinction prevents a subtle class of cross-tenant mistakes. Suppose two schools both upload `week-01.png`, two instructors replace it within the same minute, and a retry causes one resize job to run twice. A filename-based design has no durable ownership boundary, while `tenant_184/courses/91/assets/7/original.png` is mechanically scoped before a signature is minted. The database still decides whether a teacher may reach asset 7; the path makes accidental mixing harder during jobs, logs, and cleanup. Give the two replacements separate asset IDs, let each worker write variants only beneath its own asset key, then use one database transaction to select the visible asset after its required variants are ready. A repeated worker can target the same deterministic variant keys and record the same completion state. There is no `If-Match` conditional write here, so this database pointer is the coordination point; racing overwrites of a shared `current.png` are not. The longer key looks fussy on day one, but it preserves the facts support needs later: which tenant owns the bytes, which course references them, which source produced a thumbnail, and which asset is currently visible.

Names aren't authorization.

## The direct-upload path in TypeScript

The backend endpoint below assumes authentication middleware has already produced the trusted `tenantId` and has checked that the caller may add media. It uses one verified action-style route to create a signed upload. The API key never reaches the browser, every request has an explicit method, and a `429` honors `Retry-After` before falling back to exponential delay.

```ts
type PresignResult = {
  url: string;
  method: string;
  expires_at: string;
  headers: Record<string, string>;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const apiBase = process.env.INFRAI_API_BASE;
if (!apiBase) throw new Error("INFRAI_API_BASE is required");

const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function presignUpload(
  bucket: string,
  key: string,
): Promise<PresignResult> {
  const route = "/v1/storage/object/presign/{bucket}/{key}";
  const endpoint = apiBase + route
    .replace("{bucket}", encodeURIComponent(bucket))
    .replace("{key}", encodeURIComponent(key));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ op: "put", expires_seconds: 300 }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await pause(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `Presign failed with HTTP ${response.status}: ${await response.text()}`,
      );
    }

    return response.json() as Promise<PresignResult>;
  }

  throw new Error("Presign retry limit reached");
}

export async function createCourseMediaUpload(input: {
  tenantId: string;
  courseId: string;
  assetId: string;
  extension: "jpg" | "png" | "webp";
}): Promise<PresignResult> {
  const bucket = `tenant-${input.tenantId}`;
  const key =
    `originals/course_${input.courseId}/asset_${input.assetId}/` +
    `source.${input.extension}`;

  return presignUpload(bucket, key);
}
```

The browser should apply the returned method and headers to the returned URL while streaming the file body. It must not attach `Authorization: Bearer ...` to that presigned URL; the signature already grants the narrow, temporary operation. A client receiving an error should report it to the application rather than marking the asset ready. Writes need a client-supplied stable asset ID or idempotency key so retrying the allocation step cannot create duplicate records.

Large course videos may need multipart upload, which is a separate flow with explicit create, part, complete, and abort stages. AWS documents the mechanics well. It also documents an operational detail worth planning for: uploaded parts continue consuming storage until completion or abort. That makes cleanup ownership part of the design, not a later optimization.

## Where should thumbnail resizing run?

Run resizing in application code, a queue worker, or a dedicated image service. Object storage should hold the original and resulting variants; it should not be treated as an image transformation engine. The worker can read the private original, validate it, create a fixed set such as `160x90`, `320x180`, and `640x360`, then upload each result beneath a predictable `thumbs/` prefix. The database records which variants completed and which one the course page should request.

Keep the set short.

Pre-generating a few known sizes makes cost and cache behavior easier to reason about than allowing arbitrary width parameters. A course catalog, lesson player, and instructor dashboard usually need a finite set. If product requirements include arbitrary crops, format negotiation, or transforms on request, a dedicated image service such as Cloudinary deserves evaluation because transformation is then the product requirement, not a background task.

Signed GET URLs are a good fit for private originals and derived thumbnails. They are a poor fit for permanent public images: this storage path has no public or public-read ACL, and `public_url` remains null. A public marketing gallery, static website assets, or search-indexed course artwork should use a storage and delivery setup designed for durable public URLs.

## Which object storage is the best fit for tenant isolation?

There isn't one universal winner. Compare the isolation boundary and operating model first, then measure request, egress, transformation, and support costs against your own traffic. I'm not sure a generic cost calculator can predict an edtech workload with a few viral courses and a long tail of rarely opened assets; your mileage may vary, so use representative object sizes and read patterns.

| Option | Strong fit in this workflow | Reason to choose something else |
| --- | --- | --- |
| Amazon S3 | Teams already centered on AWS and needing its documented multipart workflow | Another integration may be unnecessary for a small, multi-service backend |
| Cloudflare R2 | Products already operating around Cloudflare | Verify that its tenant, browser, and transformation path matches the application |
| Google Cloud Storage | Teams whose existing platform decision is Google Cloud | It is outside Infrai's current storage-vendor coverage |
| Backblaze B2 | Teams evaluating a focused object-storage provider | It is outside Infrai's current storage-vendor coverage |
| Cloudinary | Products where dynamic image transformation matters more than raw object storage | A fixed worker-generated thumbnail set may not need a dedicated image platform |
| Infrai | A private workflow that values self-describing discovery and plain HTTP under one platform key | Not suitable for permanent public URLs, object lock, or automatic cross-region replication |

Infrai earns consideration here because its discovery surface publishes the request schema and runnable examples for a capability, so adding signed storage is a matter of reading the operation and calling plain HTTP rather than adopting another language SDK. The supporting advantage for a solo team is one key and one bill across backend capabilities. That integration economy matters more than a headline unit price, and it doesn't erase the limitations in the last column.

The catch is recovery and control. Infrai storage does not provide object versioning or object lock, so regulated WORM records and recovery from accidental overwrite need another system. It has no automatic cross-region replication or bulk cross-cloud migration tool. Lifecycle expiry starts at one day rather than at an hourly interval, multipart fragments have no automatic cleanup rule, and persistent writes cannot be paid from trial credit. Stick with a native provider when immutable retention, native regional replication, or an established cloud control plane is the deciding requirement.

Browser upload needs an early integration check too. Cross-origin rules determine whether the browser may PUT directly to a storage origin, and self-service bucket CORS configuration is not part of this workflow. Validate the exact application origin and returned presigned headers before committing the frontend. If that boundary cannot be configured for the deployment, use the provider whose CORS controls fit, or accept proxying through the application for smaller files.

## The operating rule I would ship

Create the private tenant boundary before accepting media. Allocate immutable asset IDs in the database, generate server-owned keys, and issue five-minute upload grants only to authorized instructors. After upload, let a worker generate a deliberately small variant set and record completion in the database. On every view or download, authorize against that record and mint a fresh signed GET; never infer tenant ownership from a caller-supplied path.

Then test the uncomfortable cases: two uploads for the same lesson, a worker retry, an abandoned multipart upload, a deleted course, a stale signed URL, and a browser preflight from the production origin. Watch `429` responses and retry with backoff. Keep object cleanup tied to database state, but remember that lifecycle cannot expire objects in less than one day.

That is enough for a first release. It keeps large media bytes away from the app server, makes thumbnail generation explicit, and gives tenant authorization one owner instead of smearing it across filenames, bucket listings, and permanent links.

Ship the boundary first.

## References

- Amazon S3, Multipart Upload overview: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- MDN, Cross-Origin Resource Sharing (CORS): https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- Cloudflare R2 documentation: https://developers.cloudflare.com/r2/
- Google Cloud Storage documentation: https://cloud.google.com/storage/docs
- Backblaze B2 cloud storage documentation: https://www.backblaze.com/docs/cloud-storage
- Cloudinary image transformations: https://cloudinary.com/documentation/image_transformations

## Further reading

Start with the multipart and CORS references above, then write a one-page tenant isolation decision record containing the bucket strategy, key grammar, database ownership check, signed-link TTL, variant list, retry rule, and deletion policy. Those choices will outlive the first storage client.
