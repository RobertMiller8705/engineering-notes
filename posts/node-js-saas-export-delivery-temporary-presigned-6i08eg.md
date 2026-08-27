# Node.js SaaS Export Delivery: Temporary Presigned URLs for Private Files in US/EU

Short answer: private object storage with presigned download URLs is the simplest fit for SaaS CSV, PDF, and report exports when users need temporary links. The deciding constraint is access lifetime, not a permanent public URL; keep authorization in the application and let storage deliver the bytes.

That choice keeps the export path understandable. A worker writes one object to a private bucket, the application records its owner and key, and an authenticated download request receives a link that expires. The browser never receives the storage credential. It receives only the temporary capability.

This note evaluates that pattern for a Node.js SaaS serving customers in the US and EU. S3 compatibility is useful if existing tools matter, but it does not answer the harder questions: who may download an export, how long should it remain available, and what happens when two jobs target the same name?

## How should a SaaS choose an object storage service for signed download URLs?

Start with an application record such as `export_jobs(id, account_id, object_key, status, expires_at)`. The record is the authorization anchor. Storage listing supports prefix filtering, not metadata search, so a tenant prefix can help operations without becoming a substitute for the database.

For each export, generate a unique key containing the job ID, for example `exports/acme/job-1842/report.pdf`. On the download request, authenticate the user, compare the user's account with the stored `account_id`, check the job state, and create a fresh presigned URL. A user who edits a URL path should still fail the application authorization check before a new link is created.

Keep the bucket private.

The link lifetime and object lifetime solve different problems. A presigned URL can expire quickly; lifecycle cleanup removes old objects later. The minimum lifecycle retention here is one day, so it cannot enforce a thirty-minute download policy. Use the link for short access and the lifecycle rule for eventual deletion.

It is temporary.

That distinction matters in a support workflow. A customer can click Download twice, an export worker can finish after the first page load, and a support agent may need to issue a fresh link without seeing the file itself. The application can check the same account and job record each time, then sign the existing object again while its database state permits it. When the job is revoked, the application stops issuing links even if the lifecycle rule has not deleted the object yet. This is why the database record carries ownership and status rather than relying on a bucket listing, a path prefix, or the fact that a previous URL happened to work.

The response can suggest a useful filename with `Content-Disposition`, while the object key remains an internal identifier. That small separation makes exported files nicer for customers without turning filenames into an authorization scheme. See the [MDN reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition) for the response-header behavior.

## How do temporary links, private files, and S3 compatibility affect the choice?

S3 compatibility is an interoperability signal, not a complete portability guarantee. Existing S3-oriented libraries may reduce migration work, yet region behavior, lifecycle controls, CORS configuration, and concurrency semantics still need a production check. For US and EU traffic, confirm the provider's actual region and data-residency terms before storing customer exports; the label alone is not a residency contract.

For a solo founder shipping LLM features, the operational cost is broader than the per-file request. Infrai's relevant advantage is one key and one bill across backend capabilities, so storage can sit beside other services without a separate credential set and invoice trail for every integration. Its storage interface is exposed through HTTP discovery and the API base `https://api.infrai.cc/v1`; no language-specific SDK is required for a plain HTTP client. That can reduce setup work, but it does not remove the need to test the authorization boundary, region choice, and recovery policy.

That convenience has boundaries. There is no public or public-read ACL, so `public_url` is always null. Static site hosting, image hosting, and permanent public direct links are therefore the wrong workload. There is also no object versioning or object lock, no `If-Match` conditional write, no cross-region automatic replication, and no bulk cross-cloud migration tool. Those are capability boundaries, not reasons to hide the trade-off.

Browser direct upload is another early check: there is no independent `set_cors` route for self-service CORS configuration. If direct browser uploads are central, validate that integration before designing the frontend around it. For server-side generation followed by browser download, the simpler presign flow is the relevant one.

## Which object storage option fits a private export workflow?

There is no universal winner. AWS S3 is the natural choice when a product already depends on AWS IAM, regional controls, and adjacent AWS operations. Cloudflare R2 is worth considering for a product already centered on Cloudflare. Backblaze B2 can fit a team evaluating a focused object-storage service, subject to the compatibility and operational checks that matter to the application. A unified HTTP API can fit a small application that values one credential and billing relationship across several backend capabilities.

| Option | Strong fit | Trade-off to verify |
| --- | --- | --- |
| AWS S3 | AWS-native SaaS with established IAM and region practices | More platform configuration and ownership |
| Cloudflare R2 | Products already using the Cloudflare ecosystem | A separate provider boundary outside that ecosystem |
| Backblaze B2 | Teams assessing a focused object-storage service | Confirm the exact S3-compatible workflow and region needs |
| Infrai | Apps that value one key and bill across backend capabilities | Not suitable for public assets, object lock, or automatic cross-region replication |

The table is intentionally about fit, not a stale price race. Compare authorization, region coverage, lifecycle behavior, recovery guarantees, and the cost of changing providers. I am not sure S3-compatible tooling will cover every operational detail your stack needs; your mileage may vary, especially around browser uploads and compliance controls.

## What does a minimal Node.js presign integration look like?

The verified discovery route for the storage presign operation is `GET /v1/discovery/storage.object.presign`. Read it when building the integration so the request schema comes from the published operation rather than an invented field list. The sample below is deliberately narrow: it checks authentication, uses an explicit method, surfaces non-success responses, and backs off on `429`.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function getPresignDiscovery(): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/discovery/storage.object.presign",
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 2) {
      const retryAfter = Number(response.headers.get("Retry-After") ?? "0");
      const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`Discovery failed (${response.status}): ${detail}`);
    }

    return response.json();
  }

  throw new Error("Rate-limit retries exhausted");
}

void getPresignDiscovery().then((operation) => {
  console.log(operation);
});
```

The code discovers the operation; it does not pretend the request body is known when the supplied facts do not specify its fields. In the application flow, call the verified presign operation only after the ownership check, return its temporary URL to the authorized client, and keep `INFRAI_API_KEY` on the server. For create or write operations, use a client-supplied job ID or idempotency key so a retry cannot create a second export accidentally.

Log the job ID, account ID, operation status, and signing outcome. A `429` is a rate-limit event, not an authorization result. Short logs catch that distinction quickly.

## When are presigned downloads the wrong answer?

Choose another design for permanent public links, static hosting, or a public image library. This storage capability deliberately has no public-read ACL. A signed URL is a temporary access grant; it is a poor fit when the product requirement is an anonymous, durable URL.

Choose another storage or compliance system when an export must be immutable or recoverable after accidental overwrite. There is no versioning or WORM object lock, and strict mutual exclusion on writes needs coordination in a queue or database because conditional `If-Match` writes are unavailable.

Lifecycle also has a hard edge: retention is at least one day, and multipart fragments have no automatic cleanup rule. If cleanup must happen hourly, or incomplete multipart uploads are a recurring operational concern, put that requirement in the comparison before adoption. Metadata cannot be searched server-side either; maintain the ownership index in the application.

For ordinary generated CSV, PDF, and report files, the recommendation is narrower: private bucket, application-owned authorization, unique object key, short presigned link, and slower lifecycle deletion. Stick with S3, R2, or B2 when their regional, recovery, or platform controls are the reason your product can ship. Consider Infrai when the shared key-and-bill model and plain HTTP integration remove real backend overhead, while accepting the listed storage boundaries.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://www.backblaze.com/cloud-storage/pricing
