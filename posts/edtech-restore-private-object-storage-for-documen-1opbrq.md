# EdTech Restore: Private Object Storage for Document Retention and Lifecycle Delete

Short answer: choose private object storage only after you can explain who may restore a tenant snapshot, which snapshot they may see, and what happens when the document is past its retention window. For an edtech app, the least complex design is a private bucket, opaque tenant-scoped keys, application-owned retention records, and a restore job that writes a new version instead of replacing the current document in place.

That decision is about access control and delivery simplicity. The storage service holds bytes; the application decides identity, authorization, retention class, and restore destination. A lifecycle rule can clean up ordinary temporary exports on a day boundary, but it should not be the only record of a deletion promise or the only copy of a backup plan. The rule needs a human-readable policy beside it: which event starts the clock, which tenant exceptions are allowed, how a restore is authorized, and what the application records after the bytes are gone. Without that companion record, a support engineer can see an object and its age but cannot tell whether keeping it is correct, whether a hold applies, or whether a failed restore has already created a replacement.

The bucket is only the byte store.

## Break the tenant boundary before choosing storage

Start with the restore request, not the upload form. A teacher or school administrator asks for snapshot `snapshot-7` belonging to tenant `school-204`. The API authenticates the person, checks the tenant membership and permission, verifies that the snapshot is eligible for restore, and creates a short-lived job. The browser never gets to turn a tenant ID and object key into authority by itself.

The object key should be an implementation detail, such as `tenants/school-204/snapshots/snapshot-7/doc-991`. It is useful for partitioning and operations, but it is not an access-control policy. Store the tenant ID, document ID, snapshot ID, retention class, creation time, deletion state, and restore state in a database row. Keep the authorization decision tied to those fields. A prefix check alone is too weak: a path can be well-formed while its owner is wrong.

The restore job should read the source snapshot, validate its manifest, and write to a new destination key. That gives the application a place to compare the restored document with the current one before publishing it. It also makes retries less surprising. A retry can see the same idempotency key and return the existing job rather than creating two restore operations. In a concrete flow, a request for `school-204/snapshot-7` first creates `restore-118` in `pending`, then claims the source row and writes `staging/restore-118`. The worker records the source snapshot and checksum with the job, so a restart cannot silently switch to a newer snapshot. After validation, one database transition makes the staged object the current document; a later cleanup removes the old staging key. If authorization is revoked while the job is pending, the worker rechecks it before publishing. That extra check is cheap compared with explaining to a school why a valid-looking restore came from the wrong tenant.

That is the boundary.

Private delivery is the useful default for student work, assessment material, and staff exports. Public links, static-site delivery, and permanent download URLs are different requirements. They need an explicit public-delivery design, with a separate authorization review.

## Write retention as data, not as a bucket setting

Give each object class a different policy even when all of them use the same storage service. A source document may be retained while the account is active. A generated export may be disposable. A backup snapshot may outlive the source for recovery, but it should not quietly outlive the product's stated policy. A one-day minimum is a floor for a daily lifecycle action, not proof that an object disappears exactly 24 hours after creation.

I use a retention record like this because it makes the clock inspectable:

```ts
type StoredItem = {
  itemId: string;
  tenantId: string;
  kind: "document" | "backup" | "export";
  objectKey: string;
  createdAt: string;
  expiresAt: string;
  legalHold: boolean;
  deletionState: "active" | "queued" | "deleted";
};

function isEligibleForDelete(item: StoredItem, now: Date): boolean {
  if (item.legalHold || item.deletionState !== "active") return false;
  const expiresAt = Date.parse(item.expiresAt);
  if (!Number.isFinite(expiresAt)) {
    throw new Error(`Invalid expiry for ${item.itemId}`);
  }
  return expiresAt <= now.getTime();
}

const now = new Date("2026-08-11T12:00:00Z");
const queue = items
  .filter((item) => isEligibleForDelete(item, now))
  .map((item) => ({ itemId: item.itemId, objectKey: item.objectKey }));
```

The example leaves the transport out on purpose. The storage adapter should expose a small interface such as `readSnapshot`, `writeObject`, and `deleteObject`; the policy worker should not know which provider implements those operations. A transaction or queue claim must move a row from `active` to `queued` before the delete is attempted. On success, record `deleted` and the reason. On a retryable failure, retain the job with an attempt count and next-run time. Don't let a scheduled scan issue the same deletion concurrently.

Lifecycle cleanup remains valuable for routine aging. Configure it for objects whose product promise tolerates a day-level boundary, and test it against a clock that crosses the boundary. It does not replace an account-closure workflow, a legal hold, or a restore test. It also does not tell support why a particular object remains. That explanation belongs in the application record and audit log.

## Test the restore state machine at its dangerous edges

The first failure mode is confused identity. A restore endpoint accepts `tenantId` in the request body, trusts it, and then fetches a key assembled from that value. A caller who can change one string may now read another school's snapshot. Derive the tenant from the authenticated session, then authorize the requested document and snapshot against that tenant. Treat every client-provided identifier as a lookup value, never as proof of ownership.

The second is publishing partial state. Large documents can require several writes, and a worker can stop after the new object is present but before the database marks the restore complete. Use a staging key, a manifest or checksum supplied by the application, and an explicit publish transition. A reader should see either the previous complete document or the new complete document, not an object whose metadata says “restored” while its content is incomplete.

The third is retention drift. A database job may mark an export expired while a lifecycle rule still has time left, or a lifecycle rule may remove bytes before the application has cleared the corresponding row. The application must define which state is authoritative for each class. Reconcile both directions, emit a metric for orphaned rows and orphaned objects, and make deletion explainable.

The fourth is calling a copy a backup without testing recovery. A second object is useful only if the team can locate it, authorize it, read it, and restore it into a clean destination. Schedule a small restore test with synthetic tenant data. Verify that a user from `school-204` cannot request `school-205` data, and verify that an expired export is not offered as a restore source. The test should exercise the policy, not just the storage SDK.

I'm not sure a one-day minimum is acceptable for every export. That depends on the promise made to schools and on whether a support case needs a recovery window. Your mileage may vary, but the answer should be written in the retention class rather than left to an undocumented default.

## How should an edtech team compare private object storage, backups, and lifecycle delete?

The comparison should be framed around guarantees and delivery work. Storage capacity is only one line item. A private object store with a simple application adapter may be easier to ship, while a provider-specific integration may expose controls the application genuinely needs. Neither choice removes the need for tenant authorization and restore testing.

| Design choice | Useful when | Reconsider it when |
| --- | --- | --- |
| Private object storage behind an application API | The product needs controlled document delivery and an uncomplicated upload path | Users need permanent public URLs, fine-grained archive controls, or provider-native replication |
| Direct browser upload with signed requests | Large files should avoid passing through the application server | The team cannot enforce size, content type, tenant binding, and expiry before issuing the request |
| Application-managed retention records | Support, audit, and restore decisions need an explainable source of truth | The team is unwilling to run reconciliation and deletion jobs |
| Lifecycle rules for temporary exports | The retention promise tolerates a daily cleanup boundary | The promise requires hourly precision, legal holds, or an exact deletion timestamp |

The catch is operational ownership. A direct upload path reduces delivery work but moves validation and authorization into the signing service. A server-mediated path is easier to reason about for a small document, but it adds latency and bandwidth handling. A lifecycle rule reduces recurring cleanup code, but it cannot represent a school-specific exception by itself. Choose the boundary your team can observe and test.

For this edtech scenario, I would keep the public API small: create an upload intent, create a restore job, read job status, and issue a short-lived download only after authorization. The storage adapter can change behind those commands. The conclusion follows from the workflow, not a vendor label: use private storage for ordinary tenant documents, keep retention and authorization in application data, and select a different system when the product promises immutable archives or public publishing.

## Set release gates for delete and restore jobs

Before launch, write down the clock origin for documents, backups, and exports; the minimum retention for each; the actor allowed to restore; the destination state used during restore; and the rule for account closure. Add tenant IDs to logs without logging document contents or bearer tokens. Give every restore and delete job an idempotency key, an owner, a final state, and a reason.

Then test the awkward paths. Restore an old snapshot into a staging destination. Attempt a cross-tenant read. Hold an item and confirm that cleanup skips it. Run cleanup twice. Remove the database row and the object in opposite orders and confirm reconciliation reports the mismatch. Test a clock boundary, a malformed expiry, and a worker restart between the object write and the publish transition.

Keep the routine path boring. Private keys, server-side authorization, application-owned retention, day-level lifecycle cleanup for disposable exports, and a tested restore job are enough for many teams. When the requirement changes to exact deletion, immutable retention, public delivery, or cross-region recovery, treat that as a new systems decision rather than adding one more flag to the bucket.

## References

- https://aws.amazon.com/s3/pricing/
- https://docs.digitalocean.com/products/spaces/

## Further reading

- https://aws.amazon.com/s3/pricing/
- https://docs.digitalocean.com/products/spaces/
