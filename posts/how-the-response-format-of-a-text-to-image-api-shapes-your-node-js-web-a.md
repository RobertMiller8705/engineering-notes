# How the response format of a text-to-image API shapes your Node.js web app

**Short answer:** choose the text-to-image API whose response format matches how your web app already stores files — raw bytes if the image lands in your own bucket, an expiring URL if you only need a browser preview — and let docs and SDK quality break ties rather than decide for you.

That's the whole call. The rest of this is why I moved that one criterion above latency and price per image, after a detour I could have skipped.

## The field I assumed was there

I had a prototype running in an afternoon. A user types a prompt, my Node.js service calls the image endpoint, I push the result to object storage, and the web app renders a signed URL from my own CDN. Then I swapped in a second provider — same request body, same size, a config change and nothing else — and the upload path blew up with `TypeError: Cannot read properties of undefined (reading 'startsWith')`, thrown three frames deep inside my own helper. No status code in the message. No provider error. Nothing in the log that named the actual problem. The first API had handed me `data[0].url`; the second returned `data[0].b64_json` and no URL at all, because its default response format is base64 and I'd never read that line. One 1024×1024 PNG arrived as roughly 1.9 MB of base64 sitting inside a JSON body, and my code walked straight past the missing field into a null dereference. About 40 minutes, most of it spent staring at my own stack trace instead of at the response.

The response format is the integration. Everything else you can swap later.

I've since stopped treating it as a detail you discover at implementation time — it's the first thing I read in a provider's docs, before the model list and before the price table.

## How do I choose a text-to-image API for a Node.js web app?

Rank candidates on four axes, in this order: response format, error shape, how they handle slow generations, and only then developer experience.

Response format first, because it decides where the bytes live and who pays for the extra hop. Base64 inside JSON gives you the file immediately, at roughly a third more payload than the image itself — fine if you were going to re-upload it to your own storage anyway, wasteful if the browser was the only consumer. An expiring URL is cheaper on the wire and terrible if you cache the URL instead of the file, because you'll serve dead links a few hours later. Neither is wrong. They're just different assumptions about who owns the asset.

Error shape is the second axis and the one most quickly skipped. I want a stable machine-readable `code` field, ideally something shaped like RFC 9457 problem details, so my retry logic branches on a value instead of on English prose. A safety refusal, a bad prompt, an over-quota response and a transient server condition all need different handling, and if the API flattens them into one string message, you end up writing substring matching that quietly stops working when someone rewords the copy.

Third: what happens when generation takes 30 seconds. Some APIs hold the connection open, some hand back a job id you poll, and a few offer a separate batch lane with a completion window measured in hours. The catch is that an async job handle moves the state machine into your code — you now own polling, dead jobs, and the "user closed the tab" case. For an interactive preview that isn't a good fit; for a nightly render of 5,000 catalogue variants, it's exactly right, and batch lanes are usually priced below the interactive path.

Developer experience comes last on purpose. Good docs save you a day. The wrong response contract costs you every time you change providers.

## A generation call that survives both response formats

```ts
type ImageResponse = { data?: Array<{ b64_json?: string; url?: string }> };

const BASE = process.env.IMAGE_API_BASE ?? "https://api.example-provider.test";

export async function generateImage(prompt: string): Promise<Buffer> {
  const res = await fetch(`${BASE}/v1/images/generations`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${process.env.IMAGE_API_KEY}`,
      "content-type": "application/json",
    },
    body: JSON.stringify({ prompt, size: "1024x1024", n: 1, response_format: "b64_json" }),
    signal: AbortSignal.timeout(60_000),
  });

  if (res.status === 429) {
    // Back off for exactly as long as the API asks, then let the caller retry.
    const wait = Number(res.headers.get("retry-after") ?? 5);
    throw Object.assign(new Error("rate limited"), { retryAfter: wait, retryable: true });
  }

  if (!res.ok) {
    // Read the body before throwing: the status alone never says which field was rejected.
    const detail = await res.text();
    throw new Error(`image generation ${res.status}: ${detail.slice(0, 300)}`);
  }

  const first = ((await res.json()) as ImageResponse).data?.[0];
  if (!first) throw new Error("image generation returned no data entries");

  if (first.b64_json) return Buffer.from(first.b64_json, "base64");
  if (first.url) {
    const img = await fetch(first.url);
    if (!img.ok) throw new Error(`image fetch ${img.status}`);
    return Buffer.from(await img.arrayBuffer());
  }
  throw new Error(`unrecognised image payload: ${Object.keys(first).join(",")}`);
}
```

Thirty lines, no dependencies, `fetch` and `AbortSignal.timeout` are built into Node. The part that matters isn't the happy path — it's the last `throw`, which prints the keys the provider actually sent me. Had that line existed the first time, my 40 minutes would have been 40 seconds.

## Docs, SDKs, and the developer experience that actually costs you

| Response shape | What you get back | Where it hurts | Fits when |
| --- | --- | --- | --- |
| Base64 in the JSON body | Bytes, immediately | Payload about a third larger than the file, held in memory | You re-upload to your own bucket anyway |
| Expiring URL | A link you must fetch before it expires | An extra round trip, plus dead links if you cache the URL | The browser renders it once and nothing is archived |
| Async job handle | A job id and your own polling state | You own retries, timeouts and abandoned jobs | Bulk or non-interactive generation |

An SDK is a dependency, and I keep forgetting to price that in. It's code you pin, upgrade, and occasionally fight when a major version reshapes the client, and on serverless it's also cold-start weight. What you get in return is real: typed responses, retry defaults, multipart handling you'd otherwise write yourself. My rule is boring — take the SDK for the provider I've committed to, and keep the second provider on plain HTTP behind the same interface, so the abstraction is mine rather than theirs.

The documentation signal I trust isn't the quickstart. It's whether the reference page prints a complete response body, including the error envelope, for every parameter combination that changes the shape. Docs that show eight language tabs of request code and one truncated `{ "data": [ ... ] }` are telling you the response contract was an afterthought, and that's the exact gap I fell into. As far as I can tell there's no correlation between how polished a docs site looks and how honest its response examples are — I'm not sure why, though I'd guess response examples get generated once and never re-checked against a schema.

## What I check before shipping

One contract test per provider, asserting the response shape against a recorded fixture, and the same assertions running nightly against the live endpoint so a silent default change shows up as a red build rather than as a support ticket. Strip the base64 blob out of the fixture first, or your repo grows by a megabyte per test.

Then the boring operational layer. Every image row stores the prompt, the model id, the size and the provider request id, because "why did this one come out different" is unanswerable without the inputs. Retries branch on the error code and cap at two, since image endpoints bill per generation and a retry storm is a bill, not just noise. Timeouts are explicit everywhere, both on the generation call and on the follow-up fetch of an expiring URL. Cost is a counter I can query per user per day, wired up on day one, because retro-fitting it after the first surprise invoice takes longer than adding it now.

Last, keep the provider behind an interface of about twenty lines: generate, and a normalise step that returns a `Buffer`. That interface is the only reason my second swap took an hour instead of a weekend, and it's the piece of this I'd keep even if I never changed vendor again.

## References

- Node.js global `fetch` — https://nodejs.org/api/globals.html#fetch
- RFC 4648, Base64 data encoding — https://datatracker.ietf.org/doc/html/rfc4648#section-4
- RFC 9457, Problem Details for HTTP APIs — https://www.rfc-editor.org/rfc/rfc9457
- Using the Fetch API (MDN) — https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch
- Batch API guide — https://platform.openai.com/docs/guides/batch
- Prompt Engineering Guide — https://www.promptingguide.ai
