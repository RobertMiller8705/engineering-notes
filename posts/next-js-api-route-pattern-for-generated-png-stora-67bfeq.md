# Next.js API Route Pattern for Generated PNG Storage and Signed Links

**Short answer:** A Next.js API route should store each generated PNG as a private object in its tenant's assigned region, persist the object key and region, then create a signed download link only after an authorized request.

The least complex version has three durable facts: an image record, an object key, and its assigned region. The PNG bytes live in a private object store; the database stores the key and region, not a signed URL. A later download request checks access against that record and creates a new temporary URL.

Keep the browser out of it.

## The data flow should make retries boring

An API route is a policy boundary, not merely a byte relay. It authenticates the caller, gets the tenant's approved US or EU data region from trusted application state, validates the generated image, writes a private object under a server-derived key, then commits a record that ties the object to the tenant. A separate route signs a read only after it has checked the same tenant boundary.

The ordering matters because image generation is not a transaction with object storage. A process can be retried after the object write and before the database commit. A database row can exist while a queued cleanup is still pending. Treat those as explicit states instead of assuming a single request succeeds forever. For example, an idempotent retry may find a database row that already owns the request, return its image ID, and skip a second logical creation; a periodic reconciler can separately find private objects that have no matching committed row after a defined age and remove them under the same tenant-and-region policy. The reverse case needs equal care: if deletion begins, the application should mark the image unavailable before background cleanup so it does not issue another signed link merely because the bytes have not yet been removed. These are small state transitions, but they prevent the usual confusion where a retry looks like duplication, a cleanup looks like data loss, or a temporary URL appears to be permanent application state.

Use an idempotency key for the logical generation request and a content digest for the bytes. They answer different questions: one says "have we handled this request?" and the other says "are these bytes identical?". A unique database constraint on `(tenantId, requestId)` is the part that makes a retry safe; a random filename alone does not.

## How should a Next.js API route store an AI-generated PNG for a US/EU SaaS?

Keep provider syntax behind a small storage interface. The following App Router example deliberately uses application-owned adapters: it does not imply a commercial endpoint, SDK, or storage protocol. The `putPrivate` implementation must enforce private access, while `createReadUrl` must limit the capability to one object and a bounded lifetime.

```ts
import { createHash, randomUUID } from "node:crypto";
import { imageRows, tenants } from "@/lib/db";
import { storageForRegion } from "@/lib/storage";

type Region = "us" | "eu";

function hasPngSignature(bytes: Uint8Array): boolean {
  const signature = [137, 80, 78, 71, 13, 10, 26, 10];
  return signature.every((byte, index) => bytes[index] === byte);
}

export async function POST(request: Request): Promise<Response> {
  const tenantId = request.headers.get("x-tenant-id");
  const requestId = request.headers.get("idempotency-key") ?? randomUUID();
  if (!tenantId) return Response.json({ error: "unauthorized" }, { status: 401 });

  const existing = await imageRows.findByRequestId(tenantId, requestId);
  if (existing) return Response.json(existing, { status: 200 });

  const bytes = new Uint8Array(await request.arrayBuffer());
  if (!hasPngSignature(bytes)) {
    return Response.json({ error: "invalid_png" }, { status: 415 });
  }

  const tenant = await tenants.require(tenantId);
  const region: Region = tenant.dataRegion;
  const digest = createHash("sha256").update(bytes).digest("hex");
  const key = `${tenantId}/generated/${digest}.png`;
  const storage = storageForRegion(region);

  await storage.putPrivate(key, bytes, { contentType: "image/png" });
  const image = await imageRows.insertIfAbsent({
    id: randomUUID(),
    tenantId,
    requestId,
    key,
    digest,
    region,
  });

  return Response.json({ id: image.id }, { status: 201 });
}
```

Put a body-size limit in front of this handler. Checking the eight-byte PNG signature is useful, but it is not a complete image-safety policy; decode limits and content scanning depend on the application's threat model. A route should also derive the object key from trusted identifiers or a digest, never from a browser-provided filename. One short route can carry a surprising amount of authority.

## Regional placement is a record, not a request hint

For a US/EU SaaS, select the storage region from the tenant's durable policy and write that region beside the object key. Do not choose a location from a request header, current IP address, or a browser preference. Travel and VPNs make those signals unstable, and a user-controlled header is not a placement policy.

There are a few reasonable topologies. A single regional bucket reduces operational surface when every tenant has the same placement terms. Separate regional buckets make the selected location visible and easy to audit. Replication can improve delivery reach, but it creates another copy that must be included in deletion, access control, retention, logging, and contractual review. No topology turns a bucket location into a legal conclusion on its own.

The catch is that replication is not suitable when an image must remain in one jurisdiction. Conversely, one shared bucket is a weak fit when tenant agreements require different locations. Keep the selected region immutable for an image unless a deliberate migration copies the object, validates the copy, changes the record, and cleans up under an audited workflow. The application should sign from the recorded region, even after the tenant's default changes.

## Signed downloads are capabilities, not authorization

The download route should first load the image record, verify that the caller is permitted to access its tenant, and then sign the exact stored key. A signed URL is usually a bearer capability: a recipient who has it can use it until expiry subject to the storage service's policy. It cannot replace application authorization.

```ts
import { imageRows } from "@/lib/db";
import { requireSession } from "@/lib/session";
import { storageForRegion } from "@/lib/storage";

export async function GET(
  request: Request,
  context: { params: Promise<{ imageId: string }> },
): Promise<Response> {
  const session = await requireSession(request);
  const { imageId } = await context.params;
  const image = await imageRows.require(imageId);

  if (image.tenantId !== session.tenantId) {
    return Response.json({ error: "forbidden" }, { status: 403 });
  }

  const downloadUrl = await storageForRegion(image.region).createReadUrl(image.key, {
    expiresInSeconds: 300,
  });

  return Response.json(
    { downloadUrl },
    { headers: { "Cache-Control": "private, no-store" } },
  );
}
```

Five minutes in this example is an application choice, not a universal number. Long downloads and asynchronous consumers may need another expiry, while sensitive material may need a shorter one. Expiry is also not instant revocation. If access must end immediately, stop minting new URLs and use a delivery path or storage policy that can deny the old capability. Don't log signed query strings; logs are often copied to systems with broader access than the image itself.

MDN defines `Cache-Control` as the HTTP response header for controlling caching behavior. The response above contains a temporary capability, so `private, no-store` is a sensible starting point. The image response is separate. Its cache directive needs a conscious decision about shared caches, cache keys, expiry, and whether tenant content may be retained outside the application path.

## Ship with tests for the states between requests

The happy path proves very little. Test the same idempotency key twice and assert that the application produces one logical image. Submit a non-PNG payload and expect `415`. Attempt to sign an image from another tenant and expect `403`. Exercise US and EU fixtures and assert that each read and write uses the image record's region rather than a freshly computed preference.

Also test cleanup. When a user deletes an image, deny new signatures as soon as the record becomes unavailable, then let an idempotent worker remove the object by immutable key. This avoids treating the database and object store as if they shared a transaction manager. Track a request ID, tenant ID, region, byte count, digest prefix, and separate generation, upload, and download timings. Keep secrets, prompts, and signed URLs out of those events.

Costs have the same shape as the architecture: generation, stored bytes, writes, reads, and transfer are distinct counters. Without that split, it is easy to optimize model output when retention or repeated downloads are the actual driver. I'm not sure a CDN belongs in a first release; it adds cache and regional-copy policy that a small image workflow may not need. Measure the delivery path first.

Before deploying, verify that public access is denied, runtime credentials are scoped to their regional namespace, retention rules match the product promise, and backups retain the database mapping from image to object key. Then run a synthetic upload and authorized download in each region using production-like policy. Small checks. Large payoff.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://cloud.google.com/storage/docs
