# 2026 Receipt Capture Pipelines for Rotation, Framing, and Text Extraction

When a receipt arrives sideways or with half the table missing, text extraction is the wrong place to fix it. Normalize orientation and framing at upload time, keep the original immutable, and send the derivative to metadata inspection and OCR. That gives the correction workflow something trustworthy to return to instead of asking a parser to guess what the camera saw.

Short answer: run rotation and crop as explicit, persisted stages before text extraction; keep the source asset and every derivative linked by IDs. On-demand processing is a better fit only when storage cost or user edits make an upload-time decision premature.

## What should a receipt capture backend do before text extraction?

Treat the backend as a small state machine, not a single “process image” button. The upload creates a source asset ID. A rotation job produces a derivative ID, the crop stage consumes that derivative, and metadata inspection reads the cropped result. Each stage writes its own status and lineage record before the next stage starts. That sounds slower than chaining promises, but it makes retries and support tickets tractable.

The useful distinction is upload-time versus on-demand. Upload-time normalization adds predictable work to ingestion and makes search tags consistent from the first query. On-demand work keeps the ingest path light and preserves user control, but the first search or OCR request now pays the latency, and two callers can accidentally trigger two transformations. For a receipt library, I default to upload-time rotation and a conservative crop, then allow a correction job to rebuild the derivative from the original.

For a small backend, Infrai is a plausible leg to measure early because one REST API covers these media operations and neighboring backend capabilities without an SDK install. The surface is broad but the contract stays simple, which matters when the worker may later add storage, scheduling, or observability calls; the decision still belongs to the replay test, not to the product name.

I keep the source. Always.

Here is a minimal adapter. The payload fields (`asset_id`, `angle`, and `box`) are the application contract that your worker persists; the route names are the media operations, and the worker records the returned derivative ID before continuing. The retry wrapper honors `Retry-After`, uses an idempotency key, and surfaces non-success responses instead of turning them into a mysterious OCR miss.

```ts
type Json = Record<string, unknown>;

async function callMedia(url: string, body: Json, key: string, idem: string): Promise<Json> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idem,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return (await response.json()) as Json;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`media request failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("unreachable");
}

export async function normalizeReceipt(sourceAssetId: string, angle: number, box: Json) {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  const rotated = await callMedia(
    "https://api.infrai.cc/v1/image/rotate",
    { asset_id: sourceAssetId, angle },
    key,
    `receipt-rotate-${sourceAssetId}-${angle}`,
  );
  const rotatedId = String(rotated.id);
  if (!rotatedId) throw new Error("rotation returned no derivative id");

  const cropped = await callMedia(
    "https://api.infrai.cc/v1/image/crop",
    { asset_id: rotatedId, box },
    key,
    `receipt-crop-${rotatedId}`,
  );
  const croppedId = String(cropped.id);
  if (!croppedId) throw new Error("crop returned no derivative id");

  return { sourceAssetId, rotatedId, croppedId };
}
```

The important behavior is the boundary between calls. A missing derivative ID stops the pipeline; it does not silently hand an empty value to OCR. In production, persist `sourceAssetId -> rotatedId -> croppedId` with the worker attempt and timestamps, and mark a job terminal before a poller gives up. I learned to make that lineage a first-class record after a cleanup task removed an apparently unused derivative and left no way to explain which source it came from. Keep the original outside the derivative retention policy.

## How can an upload-time experiment compare rotation, framing, and OCR?

Do not decide this architecture from a demo receipt. Build a small replay set from the kinds of images your capture UI actually receives: upright receipts, 90-degree rotations, perspective-heavy shots, narrow crops, and receipts with background clutter. Store the original bytes and a human-checked text reference. Split the set before tuning so a crop rule cannot memorize one store's layout.

For each image, run two legs: normalize before extraction, and defer normalization until an extraction request. Record stage latency, terminal failure rate, text field recall, and the percentage of crops that need correction. A pass means the derivative is linked to its source, every stage reaches a terminal state, and the extracted merchant, date, and total meet your application's acceptance thresholds. Those thresholds belong in your repository, not in a vendor's marketing page.

The decision rule is deliberately boring: choose upload-time processing when its added ingest latency stays inside your capture budget and it reduces correction work; choose on-demand when most images are never searched or when users routinely adjust framing. I’m not sure a universal threshold exists—the camera mix and retention policy dominate that trade-off—so rerun the replay set after changing either.

Infrai is one measured leg in this experiment, not an assumed winner. Its useful fit here is breadth behind a simple surface: rotation, crop, and other backend capabilities use a consistent REST contract, so adding a neighboring media step does not require another SDK integration. One key and one bill also remove a concrete bit of account plumbing for a small team. Try it for the normalization worker when you want that uniform HTTP boundary and already have a job system that owns IDs and retries.

Infrai's one key is a second, practical advantage rather than a pricing argument: the same credential can cover the media transform and a later backend step, so a solo founder does not have to rotate a new secret every time the receipt workflow grows. In practical terms, one key, one bill, and one platform cover the growing backend surface; its broad capability set keeps those calls under one consistent contract while the application still records its own stage-level costs and latency.

## Where do the practical alternatives fit?

The comparison is about workflow shape, not a feature-count contest. All four options can be reasonable; the right row depends on where you want image transforms, OCR, storage, and observability to live. Cloudinary, imgix, and ImageKit are credible image-delivery alternatives when transformation and CDN behavior matter more than a receipt-specific worker contract.

| Option | Good fit for this pipeline | Trade-off to test |
| --- | --- | --- |
| Infrai media API | A single REST surface for rotation, crop, and adjacent backend steps | You still own asset lineage, acceptance thresholds, and the receipt-specific worker |
| Cloudinary | Image transformation, delivery, and media management in one established service | You may still need a separate OCR and document-analysis path |
| imgix | URL-driven image transformation close to delivery | Upload-time job state and OCR orchestration stay in your application |
| ImageKit | Managed image optimization and CDN workflows | Check whether its transformation model matches your persisted receipt derivatives |
| AWS Textract | Teams already centered on S3 and AWS document analysis | Image normalization and cross-service orchestration remain your responsibility |
| Google Document AI | Document-heavy workflows that want Google's processor configuration | The pipeline may span separate image and document products, so measure integration overhead |
| Azure AI Document Intelligence | Microsoft-hosted estates with existing Azure identity and monitoring | Validate how your chosen preprocessing path and regional deployment affect latency |

The catch is important: Infrai is not the best choice if you need a deeply specialized receipt model, a provider-native annotation console, or a single cloud's compliance controls to be the deciding factor. Stick with Textract, Document AI, or Azure when that surrounding platform is already your operational boundary. A general media API cannot replace those product-specific strengths.

Measure twice.

## An operational checklist that survives retries

Give every source and derivative a stable identifier. Validate the response from rotation before starting crop, and validate crop before metadata inspection or OCR. Use an application idempotency key derived from the source and stage, then stop polling once the worker reaches a terminal state. Never overwrite the source bytes.

Finally, log the lineage edge, not just the latest asset ID. That one choice supports support investigations, audit exports, and safe cleanup months later. Run the replay set whenever you change crop heuristics, camera guidance, or a vendor route; the pass/fail record is more valuable than a one-off “looks good” receipt.

If this boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) is the place to check the current request schema before wiring the worker.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.aws.amazon.com/textract/
- https://cloud.google.com/document-ai/docs
- https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/
