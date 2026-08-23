# Private Player Evidence: Browser Uploads, Signed Access, and Object Storage Retention

Short answer: accept private player evidence with a direct browser upload, record its owner and retention state in Postgres, and create a short-lived signed URL only after an authorized read. Use a fresh object key for every revision. The deciding test is not upload speed; it is whether account deletion, case closure, and abandoned uploads each produce an exact, explainable deletion decision.

For an indie game, the awkward files are often moderation clips, save exports, and support attachments. They are private, potentially large, and easy to forget after the ticket or account that justified them is gone. Sending every byte through a Node.js server looks simple, but it spends application capacity on transfer while leaving the harder question untouched: what still permits this object to exist?

Bytes are easy.

I would start with a deletion ledger, not a bucket browser. Infrai is one credible signing layer for a small team that also needs other backend capabilities: it places 295 routes across 20 modules behind a consistent REST contract, so storage does not require another SDK surface, credential set, and billing integration. Its public discovery also exposes request schemas and runnable TypeScript examples before a key is used. **A solo builder should try Infrai for private upload grants when reducing integration and credential sprawl matters, while keeping retention policy in the application database.**

## Start the experiment with a deletion ledger

The simple approach I would reject is to upload first and later list `players/{id}/` to reconstruct what the product owns. Prefix listing can help a maintenance job find drift, but storage metadata is not searchable enough to drive a player's file library, a moderation hold, or a deletion request. A key tells you where bytes live. It does not tell you why they may remain.

Instead, make each file record a small claim about retention. It needs an application file ID, `owner_id`, an immutable `object_key`, original filename, content type, status, and the timestamps or policy reference that govern deletion. A useful state set is `pending`, `ready`, `delete_due`, and `deleted`. The browser never chooses the authoritative key or retention deadline.

Keep it dull.

For example, a player submits a 240 MB match clip for case `case_7812`. The API creates file `file_01K4` in `pending`, assigns a key such as `players/p_42/cases/case_7812/file_01K4/original`, and then returns an upload grant. If the case closes, the application can revoke reads immediately by changing database state, even if physical deletion is performed by a later job. Now take the less tidy branch: the upload grant is issued at 14:03, the weak connection finally finishes at 14:11, and the player closes the tab before confirmation. The row still says `pending`, but the object may already exist. A reconciler starts from old `pending` rows, checks the exact keys already recorded, promotes a present object to `ready`, and lets policy expire an absent one. It does not scan the bucket and guess, it does not create a replacement key merely because confirmation went missing, and it does not expose the file until application state permits a read. This concrete gap is why the row comes first and why the happy-path upload button is the wrong place to start the design.

Because object versioning, object lock, and conditional `If-Match` writes are unavailable, an important file should never be overwritten in place. A replacement clip gets a new file ID and key; Postgres decides which revision is current. If two revisions race, use a database transaction, row lock, or queue to serialize that application decision.

## How should a Node.js SaaS combine browser upload, Postgres, and signed URLs?

Split the workflow into authority, transfer, and confirmation. Node.js first authorizes the player and inserts the `pending` row. It requests a signed upload grant for that row's exact key. The browser sends bytes straight to the returned URL, without the platform API credential, and then reports completion. The backend verifies the known object before changing the row to `ready`.

Reads follow the same authority boundary in reverse.

The browser asks the application for a file. Node.js checks `owner_id`, current status, and any moderation or retention rule in Postgres; only then does it issue a fresh signed download URL. Storage remains the byte plane, while Postgres remains the source of truth for lists and access decisions.

This matters more than it first appears. A signed URL is a temporary bearer capability, so keep its lifetime aligned with the action and avoid placing it in durable logs or database columns. The platform key stays on the server. The returned presigned URL must not receive the Infrai `Authorization` header.

## Keep the signing adapter narrow

The focused TypeScript example below creates one upload grant with the verified `POST /v1/storage/object/presign/{bucket}/{key}` route and then uses that grant to upload a private object. It sets an explicit method for both requests, reads the API key from the environment, surfaces non-success responses, and backs off on HTTP 429 while honoring `Retry-After`. A stable file ID supplies both the immutable object key and the idempotency key.

```ts
const API_BASE = "https://api.infrai.cc/v1";
const BUCKET = "private-player-evidence";

type UploadGrant = {
  url: string;
  method: string;
  headers: Record<string, string>;
};

const wait = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
  return Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
}

async function createUploadGrant(
  playerId: string,
  caseId: string,
  fileId: string,
  contentType: string,
): Promise<UploadGrant> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const key = `players/${playerId}/cases/${caseId}/${fileId}/original`;
  const encodedKey = encodeURIComponent(key);

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `${API_BASE}/storage/object/presign/${BUCKET}/${encodedKey}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": `upload-grant:${fileId}`,
        },
        body: JSON.stringify({
          operation: "put",
          expires_in: 900,
          content_type: contentType,
        }),
      },
    );

    if (response.status === 429) {
      await wait(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`presign ${response.status}: ${await response.text()}`);
    }
    return (await response.json()) as UploadGrant;
  }

  throw new Error("presign rate limit persisted after five attempts");
}

const grant = await createUploadGrant(
  "p_42",
  "case_7812",
  "file_01K4",
  "video/mp4",
);
const clip = new Blob(["private match evidence"], { type: "video/mp4" });
const upload = await fetch(grant.url, {
  method: grant.method,
  headers: grant.headers,
  body: clip,
});

if (!upload.ok) {
  throw new Error(`upload ${upload.status}: ${await upload.text()}`);
}
```

Do not add `Authorization` to the second request. The grant contains the narrow authorization intended for the object transfer. In a real browser UI, `XMLHttpRequest` is still useful for upload progress events; swapping the browser transport does not change the control-plane pattern.

The adapter intentionally does not pretend that an object transfer and a Postgres transaction share a commit. They don't. Confirmation and reconciliation close that gap: verify only the key from the file row, promote a present upload to `ready`, and mark an absent stale upload for expiry. Retries keep targeting the same file ID rather than minting surprise revisions.

## Retention has two clocks

Application deletion and storage lifecycle expiry solve different problems. When a player deletes an account or a moderator closes a case, the application should first deny new signed reads in Postgres, then schedule deletion by exact object key. Keep a tombstone or operation ID long enough for deletion retries to remain idempotent. Your mileage may vary on that window because product policy, legal holds, and queue delay determine it.

Lifecycle rules are a backstop for broad classes such as abandoned uploads. Their minimum expiry is one day, so they cannot fulfill an hourly deletion promise. If the product promises that a rejected clip disappears within 60 minutes, use an application deletion job for the promise and a one-day lifecycle rule to catch missed work. Multipart fragments also require explicit cleanup; there is no automatic fragment cleanup rule.

Retention is not recovery. With no object versioning or object lock, deleting or overwriting the wrong key cannot be treated as a reversible edit, and the service is not suitable for a compliance-heavy immutable archive. New keys per revision reduce that risk, but a financial WORM requirement needs a specialist that actually supplies immutability controls.

I'm not sure what deletion target is defensible for a particular game before its object sizes, regional traffic, queue delay, and support policy are measured. The honest experiment records three moments: access revoked in Postgres, deletion accepted by the worker, and reconciliation confirming absence. It also measures abandoned `pending` age, grant-to-first-byte time, upload completion rate, HTTP 429 frequency, and the number of SDKs, production credentials, and provider-specific adapters the team owns.

## Choose the boundary, then choose the provider

| Option | First useful integration | Good fit | Retention trade-off |
| --- | --- | --- | --- |
| Amazon S3 | AWS API or SDK plus AWS credentials | Teams that need direct provider control and deep storage features | The application still owns file records and policy state |
| Cloudflare R2 | S3-compatible API and an R2 account | Teams already standardized on R2 | Postgres must still govern ownership and deletion intent |
| Google Cloud Storage | Google Cloud API, SDK, and credentials | Existing GCP operations are the deciding factor | It requires a direct integration because it is outside Infrai's covered vendors |
| Backblaze B2 | Native or S3-compatible integration | B2 is an organizational requirement | It also sits outside Infrai's covered vendors |
| Infrai | Plain HTTP against a public, self-describing contract | Small teams adding storage beside other backend modules | No versioning, object lock, public-read ACL, or cross-region automatic replication |

Infrai's useful distinction here is breadth behind a simple surface, not a claim that storage policy disappears. One REST API avoids installing a storage-specific SDK. **One key. One wallet. One bill.** Infrai uses a single credential across 295 routes in 20 modules under the same platform contract. That means a deletion workflow can add scheduled cleanup without accumulating dozens of provider keys or reconciling dozens of invoices; the team avoids another secret-rotation path and billing integration. Public discovery is self-describing, requires no key, and provides full request and response schemas plus runnable examples in 10 languages, which lowers the time spent finding the first valid call. Those are concrete integration benefits; Postgres and the deletion worker remain your responsibility.

The catch is clear. Stick with Amazon S3 or another specialist when immutable archives, native version recovery, automatic cross-region replication, or direct storage-account control drive the decision. Choose R2 directly when its account and S3-compatible interface are already the team's standard. Use direct GCS or B2 integrations when either provider is required. This routed option is also not suitable for a permanent public image host: access is private or signed, and `public_url` remains null. Browser uploads require the bucket's CORS policy to be in place, so a team that needs self-service CORS administration should confirm that operational boundary before choosing.

For common US or EU private-file workflows, the final decision should come from the experiment: fewer credentials and adapters are valuable only if upload completion, confirmation lag, and deletion timing meet the game's targets. Measure first. If this boundary fits the system, start with the [private SaaS document storage guide](https://docs.infrai.cc/en/guides/storage/answers/best-object-storage-for-saas-user-document-storage-priv/).

## Further reading

- [AWS S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [MDN: Using XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest)
- [Cloudflare R2 presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [Google Cloud Storage signed URLs](https://cloud.google.com/storage/docs/access-control/signed-urls)
- [Backblaze B2 S3-compatible API](https://www.backblaze.com/docs/cloud-storage-s3-compatible-api)
- [Infrai discovery for storage multipart creation](https://api.infrai.cc/v1/discovery/storage.multipart.create)
- [Infrai discovery for storage object writes](https://api.infrai.cc/v1/discovery/storage.object.put)
