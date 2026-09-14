# Photo Contest Submissions in Express: Record One Transform Version Per Entry

Pick one named transformation, run every entry through it at intake, and write that transformation's version into the row next to the photo. For a Node.js/Express photo contest that is the entire design: identical processing is what makes judging comparable, and the recorded version is what keeps the entries you took in March comparable to the ones you take in June, after someone edits the recipe.

The originals never get touched. Winners need print files.

Take a fintech app running an in-app contest — customers upload one shot of the small business they're funding with the card, and a five-person panel scores the gallery on their phones. Every image a judge sees has been resized and compressed by the same recipe, so the thing that varies between two entries is the photograph, not the encoder. Quality against bandwidth is the axis you're tuning, and you decide it once, for all entries, before submissions open.

## How do I normalise every photo contest submission identically?

Three steps, and the order is the whole trick. Accept the upload and keep the original bytes untouched. Hand a copy to one named transformation. Store the render you get back along with the transformation's name and version, then serve only that render to judges.

Nothing in the request path chooses settings per photo. No "this one came in dark, bump the quality", no per-device branch, no second attempt at a smaller size because an upload looked heavy. The moment a per-entry decision creeps into intake, the gallery stops being a comparison and you can't say which entries were affected.

The version string is the part that gets skipped, and it's the part that costs you later. Six weeks into a contest someone drops quality from 80 to 72 because the gallery feels slow on a train, ships it on a Tuesday, and now entry 1 and entry 900 were judged under different rules — with no column in the database that says so. If every row carries `contest-entry-v3`, the fix is boring: re-run the older entries through the new recipe before scoring, or scope a judging round to a single version. Without that column you're guessing, and in a contest with a prize attached, guessing is the thing you get asked about.

## The Express intake route

One POST handler, one named recipe, one column. The example below is ESM on Node 20, and it stores the whole processed response rather than picking fields out of it — the provider's schema is the authority on field names, not my memory.

```ts
import express from "express";
import { randomUUID } from "node:crypto";

const BASE = process.env.INFRAI_BASE_URL!;   // the provider's /v1 root
const KEY = process.env.INFRAI_API_KEY!;     // never inline a key
const TRANSFORM = "contest-entry-v3";        // the one recipe every entry gets

type Json = Record<string, unknown>;
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

async function post(path: string, body: Json, idemKey: string): Promise<Json> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE}${path}`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "idempotency-key": idemKey,          // a retry must not produce a second render
      },
      body: JSON.stringify(body),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after")) || 0;
      await sleep(retryAfter * 1000 || 400 * 2 ** attempt);
      continue;
    }

    const json = (await res.json()) as Json;
    if (!res.ok) throw new Error(`${path} ${res.status}: ${JSON.stringify(json)}`);
    return json;
  }
  throw new Error(`${path}: rate limited on all 4 attempts`);
}

// POST /v1/image/transformation/create — run once at deploy, never per request.
async function ensureTransform(): Promise<void> {
  await post("/image/transformation/create", {
    name: TRANSFORM,
    operations: [
      { type: "resize", width: 2000, height: 2000, fit: "inside" },
      { type: "compress", format: "webp", quality: 80 },
    ],
  }, `transform:${TRANSFORM}`);
}

const entries = new Map<string, Json>();
const app = express();

app.post("/entries", express.json(), async (req, res) => {
  const entryId = String(req.body.entryId ?? randomUUID());
  const originalUrl = String(req.body.imageUrl);
  try {
    // POST /v1/image/process — the same named transform, every single entry.
    const render = await post("/image/process", {
      image_url: originalUrl,
      transformation: TRANSFORM,
    }, `entry:${entryId}`);

    entries.set(entryId, {
      originalUrl,                              // kept for the winners' print files
      render,
      transform: TRANSFORM,                     // the column judging depends on
      processedAt: new Date().toISOString(),
    });
    res.status(201).json({ entryId, transform: TRANSFORM });
  } catch (err) {
    res.status(502).json({ error: (err as Error).message });
  }
});

await ensureTransform();
app.listen(3000);
```

Two details in there matter more than the transform settings. The idempotency key is derived from the entry id, so a submitter who double-taps on a flaky train connection gets one render instead of two. And a 429 backs off and retries instead of hammering — deadline night is when a photo contest gets its traffic, in one ugly spike, usually in the last 90 minutes.

## Quality against bandwidth: where to put the line

Judging happens on phones, so the render budget is the scroll budget. Forty thumbnails in a judging round at 250 KB each is a 10 MB scroll, and a panel on mobile data will feel that; the same round at 80 KB each is 3.2 MB and nobody notices. That's the arithmetic behind the setting, and it's worth doing before you argue about encoders.

Long edge at 2000 px with WebP around quality 80 is a defensible starting point for a judged gallery: it survives a full-screen look on a laptop, and it's small enough to scroll. Go lower and JPEG artefacts start showing up in exactly the places photographers get judged on — sky gradients, skin, shadow detail. Go higher and you're paying bandwidth for detail that the panel's screen can't render anyway. MDN's format guide is the honest reference for what each codec costs you; your own entries probably want one round of eyeballing before you freeze the number.

Keep the original in cold storage regardless of what you pick. You can always re-derive a smaller render from a full-resolution file, and you can never recover the detail you threw away at intake — that's what the winners' print files come out of.

## Which tool fits which contest

| Option | Where processing runs | How you pin the version | The catch |
|---|---|---|---|
| sharp (libvips) | In your Node process | Your lockfile plus your own code | You own the CPU and the memory ceiling; a deadline spike competes with your API traffic |
| Cloudinary | Vendor, named transformations | Named transformations are editable in place, so the version belongs in the name | Deep URL/transform model to learn for a contest that needs one recipe |
| imgix | Vendor, URL parameters | The parameter string itself, centralised in your code | No stored recipe — a shared link with different params renders differently |
| Cloudflare Images | Vendor, named variants | Variant definitions, versioned by hand | Variants apply at delivery, so editing one changes how past entries look |
| Infrai | Vendor, named transformations | The transformation name, created once at deploy | General backend API rather than an image CDN |

If your contest is a few thousand entries and you already run a worker, sharp on libvips is the least complex thing that works, and it keeps the transform definition in the same repo as the code that calls it. Stick with it unless intake bursts are the problem.

Infrai fits when the image step is one of several backend jobs in the same app — named transformations sit behind the same key and the same request conventions as its other modules, 295 routes across 20 modules, so adding moderation or OCR to the intake path later is another endpoint rather than another vendor contract. That breadth is the argument, and it has a boundary — it's a backend API, not a delivery network, so for millions of edge-served renders a day an image CDN like imgix or Cloudflare Images is the better fit.

## Before you open submissions

Freeze the recipe before the first entry lands, and treat a change to it the way you'd treat a schema migration: new name, new version, backfill decided deliberately. Write the transform name into the entry row in the same transaction as the entry itself, so there is never a render whose provenance you have to reconstruct. Keep originals in private storage with signed URLs and leave them out of the judging path entirely. Run 20 real submissions through the pipeline the week before launch and look at them on the device the judges will actually use — not on your monitor. And when someone asks mid-contest whether the gallery can load faster, the answer is a new transform version applied to everything, not a quiet tweak applied to what's left.

## Further reading

- [MDN — Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [sharp — high performance Node.js image processing](https://sharp.pixelplumbing.com/)
- [Cloudinary — named transformations](https://cloudinary.com/documentation/image_transformations#named_transformations)
- [imgix — rendering API reference](https://docs.imgix.com/apis/rendering)
- [Cloudflare Images — variants](https://developers.cloudflare.com/images/manage-images/create-variants/)
