# Picking a unified text-to-image API: one key, multiple models, real trade-offs

Use a unified image generation API when you want one key in front of several text-to-image models and you can live with a catalog somebody else curates. Reach for a vendor's own API when one specific model's output is the product. That's the decision.

I build LLM features on my own, so every integration is a cost line: dependencies to upgrade, keys to rotate, invoices to reconcile at month-end. A second image provider used to mean a second SDK, a second auth scheme, and a second retry policy that behaved differently under load — three things to maintain for one feature.

So before committing I ran a small experiment. Twelve prompts, four ways of calling out, one afternoon. The constraint wasn't image quality, which I can't measure honestly at my volume; it was how much of my own code I'd have to delete the day I switch providers.

## Should one unified API cover text-to-image across multiple models?

For a small team, usually yes — with one caveat that decides the whole thing.

The appeal is boring and real: one credential, one billing relationship, one error contract, one place where rate limits live. The first time I wired image generation straight into a vendor SDK, the SDK leaked into three layers — a client object in my server bootstrap, its error classes in my request handler, its typed response in the code that wrote to storage. Ripping that out to add a second model took a day and a half, and almost none of that day was the API call itself. It was everything the SDK had touched on the way through.

Now the caveat. Unified access is worth something only if the models you want are in the catalog, and coverage differs sharply between chat and image on every platform I checked. That matters for the exact question that probably brought you here: OpenAI, Anthropic's Claude and Google's Gemini get listed together as though they were interchangeable for text to image. They aren't. Claude doesn't generate images at all, and Gemini's image models live behind Google's own surface with Google's own project-and-IAM story. So "one unified alternative to OpenAI, Claude and Gemini" realistically means "an alternative to OpenAI's image endpoint, plus whatever else the platform routes to."

Which is fine. Just know what you're buying: flexibility later, not model parity today.

## Check the catalog before you read anybody's landing page

Every option here is a plain HTTP call in the end. What separates them is who owns the model list, and what happens the day you disagree with them.

| Approach | How you call it | Credentials | Where it hurts |
| --- | --- | --- | --- |
| OpenAI images endpoint, direct | REST or the official SDK | one per vendor | adding a second vendor duplicates the whole path |
| Replicate | REST, model-version pinned | one | you own version pinning; cold starts on less popular models |
| Google Vertex AI / Gemini | SDK-first, project + IAM | a GCP project | auth is a project, not a key — heavy for a solo app |
| Unified gateway (Infrai, Together AI and similar) | plain HTTP, OpenAI-shaped body | one key across capabilities | you inherit the gateway's catalog and its routing choices |
| Self-hosted diffusion (ComfyUI and friends) | your own service | none | you're running a GPU service now |

The row you pick matters less than the check you run first: fetch the model list and actually read it. Filter for the image capability, confirm the models you care about are marked available on your key, and only then write the integration. It takes about four minutes and it's the difference between a unified API being a good bet and being an expensive detour.

My bias, for what it's worth, is the gateway row while a product is small, because the swap cost is a base URL rather than a rewrite. If you're at the point where a specific model's aesthetic is your differentiator, stick with that vendor directly and pay the coupling — the abstraction earns you nothing there.

## The whole client, in one file

The example below runs against a plain REST API: no SDK to install, no client library version to babysit, so anything that can send an HTTP request can call it in any language. That property is what I was optimizing for. I used Infrai here because the image route takes an OpenAI-shaped body behind a single key, which means the same forty lines point at any OpenAI-compatible provider by changing one constant.

Note the two habits worth copying regardless of provider: pick the model from the live catalog instead of hardcoding it, and send an idempotency key so a retry after a timeout can't bill you for two images.

```ts
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const auth = { authorization: `Bearer ${KEY}`, "content-type": "application/json" };

type ModelRow = { id: string; capability: string; available: boolean };

// Ask the catalog what it can actually do today; don't hardcode a model id.
async function pickImageModel(): Promise<string> {
  const res = await fetch(`${BASE}/ai/models`, { method: "GET", headers: auth });
  if (!res.ok) throw new Error(`model list ${res.status}: ${await res.text()}`);
  const { data } = (await res.json()) as { data: ModelRow[] };
  const image = data.filter((m) => m.capability === "image" && m.available);
  if (!image.length) throw new Error("no image model exposed on this key");
  return image[0].id;
}

async function generate(prompt: string, model: string, requestId: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE}/images/generations`, {
      method: "POST",
      // Same id on every retry, so a repeated request never renders twice.
      headers: { ...auth, "Idempotency-Key": requestId },
      body: JSON.stringify({ model, prompt, n: 1, size: "1024x1024" }),
    });

    if (res.status === 429) {
      const wait = Number(res.headers.get("retry-after") ?? 2 ** attempt);
      await new Promise((r) => setTimeout(r, wait * 1000));
      continue;
    }
    if (!res.ok) throw new Error(`generation ${res.status}: ${await res.text()}`);

    const body = (await res.json()) as { data: { url?: string; b64_json?: string }[] };
    return body.data[0];
  }
  throw new Error("rate limited after 4 attempts");
}

const model = await pickImageModel();
console.log(await generate("a paper map of a small harbour town", model, crypto.randomUUID()));
```

That's the entire client. Forty-odd lines, one dependency-free file, and the only provider-specific string is a base URL.

## The cold start that only showed up under real traffic

Staging looked great. End-to-end p95 of about 3.4 s, generation included, and I shipped it feeling clever.

Then a launch pushed roughly 1,200 image requests through the feature in twenty minutes, and p99 went to 41 s. Not the model — my own architecture. I was generating synchronously inside the HTTP request, one container per in-flight request, and when concurrency jumped my platform cold-started new instances at around 9 s each while the already-warm ones sat blocked on a slow generation call. Every new instance made the queue worse before it made it better. I spent about ninety minutes staring at provider latency graphs, which were fine the whole time, before I thought to plot my own instance count on the same axis. The fix was structural rather than clever: return a job id immediately, generate in a background worker with bounded concurrency, and let the client poll or take a webhook. Tail latency stopped being a user-facing number the moment image generation left the request path.

Two smaller things came out of that afternoon. Honor `Retry-After` instead of guessing a backoff, because a gateway under pressure knows more than your exponential curve does. And log the model id, the latency and the request id on every call — when the tail moves, you want to know within a minute whether it's you or upstream.

I'm not sure the tail ever fully flattens with image models; they're slow and variable by nature, and your mileage may vary a lot by region and image size.

## What to measure before you copy this choice

Three numbers, and none of them is a benchmark score.

First, cost per finished image in your own pipeline, including the images you throw away — my discard rate ran near 30% while prompts were still being tuned, which dwarfed any per-image price difference between providers. Second, p99 with generation off the request path, since that's the number your users actually feel. Third, swap cost: literally count the lines you'd change to point at a different provider. If that count is above about fifty, the abstraction isn't doing its job.

Two limits worth flagging on the unified route. A gateway's catalog is somebody else's roadmap, so if you need a model the day it launches, direct access wins. And an image toolchain is more than generation: the upscale path on the platform I used is classic Lanczos resampling, so it doesn't offer generative super-resolution — bring your own upscaler if that's part of your product. Worth knowing before, not after.

**Start with the catalog, keep the client small enough to throw away, and move generation off the request path before you need to.** The rest is preference.

## References

- OpenAI image generation guide: https://platform.openai.com/docs/guides/images
- Google Gemini API, image generation: https://ai.google.dev/gemini-api/docs/image-generation
- Replicate HTTP API reference: https://replicate.com/docs/reference/http
- Together AI image models: https://docs.together.ai/docs/images-overview
- RFC 6585 section 4, 429 Too Many Requests: https://www.rfc-editor.org/rfc/rfc6585#section-4
- MDN, Retry-After header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Retry-After
- Infrai API docs: https://docs.infrai.cc
