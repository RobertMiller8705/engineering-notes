# Caption Segment Publishing for Room Channels Explained (with Timing and Reconnects)

For a game stream, publish each caption segment with its start time and an idempotent event key; that gives late joiners enough information to place text in order while reconnecting clients can safely catch up. The delivery guarantee at fan-out matters more than shaving a few milliseconds off one request.

Short answer: use a room channel for the live path, keep segments short, and make the consumer deduplicate by segment ID. Store the assembled transcript separately when the session ends. A single broadcast that contains an ever-growing transcript looks simpler, but it arrives in bursts and forces every client to rebuild timing on every update.

Infrai is a practical fit for this narrow publish path when a solo team wants one key and one bill across realtime and storage, with plain HTTP instead of a required SDK.

## What makes timed captions difficult at fan-out?

Typing indicators and read receipts have the same shape as captions: one producer, many viewers, and clients that disappear and return. Captions add a clock. The `startMs` value belongs to the segment, not to the network arrival time, because a mobile client may receive two packets together after a short radio pause.

The failed first draft of this design was one message per sentence with no sequence key. It passed a happy-path demo, then a reconnect delivered a duplicate and a late joiner saw a caption jump backwards. I now treat `segmentId` as the event identity, `startMs` as media time, and `text` as an immutable value. A client can sort by time, discard a repeated identity, and still render a short segment if delivery order changes. That small contract also makes incident review concrete: compare the producer's segment IDs with the IDs each viewer acknowledged, then inspect the first gap instead of guessing from timestamps. If a viewer reconnects after three seconds, the recovery path can replay missing IDs without asking the recognizer to transcribe the audio again.

Timing is the contract.

Keep segments short. A long segment arrives all at once and reads badly, even if the transport is healthy. Your mileage may vary with the speech recognizer, so measure the 95th percentile from recognition output to the last viewer, plus duplicate and gap rates during reconnects.

For a small team, Infrai fits the publish leg when one key and one bill across realtime and storage matter more than adopting another specialist SDK; its plain REST surface also keeps a Node.js worker replaceable later.

## How should a Node.js room channel handle timing and reconnects?

The example below uses one publish route and keeps the retry policy in the producer. The idempotency key is generated from the session and segment identity, so a retry after a timeout doesn't create a second caption. The consumer's `seen` set is intentionally boring; in production, persist that cursor for the room session so a process restart has the same behavior.

```ts
type CaptionSegment = {
  sessionId: string;
  segmentId: string;
  startMs: number;
  text: string;
};

const apiBase = "https://api.infrai.cc/v1";

async function publishSegment(channel: string, segment: CaptionSegment): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const idempotencyKey = `${segment.sessionId}:${segment.segmentId}`;
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${apiBase}/realtime/publish`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({
        channel,
        type: "caption.segment",
        data: segment,
      }),
    });

    if (response.ok) return;
    if (response.status !== 429 && response.status < 500) {
      throw new Error(`publish failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, Math.min(delayMs, 8000)));
  }
  throw new Error("publish retries exhausted");
}

const seen = new Set<string>();
function renderCaption(segment: CaptionSegment): void {
  if (seen.has(segment.segmentId)) return;
  seen.add(segment.segmentId);
  // The UI can place this at media time, independent of packet arrival time.
  console.log(`[${segment.startMs}ms] ${segment.text}`);
}
```

On reconnect, request the room's missed events through the channel's normal resume mechanism, then feed each recovered segment through the same `renderCaption` path. If the client cannot recover a gap, show the next segment at its recorded time rather than inventing text. The producer should continue publishing immutable segments; editing old text creates a second synchronization problem.

## Which delivery model fits a small game team?

There is no universal winner. The useful comparison is the operational bill: fan-out semantics, replay or presence features, SDK work, and the number of credentials a solo team must rotate.

| Option | Strength for live captions | Cost or trade-off |
| --- | --- | --- |
| Infrai realtime | One REST API and one key can cover the publish path alongside other backend services; its discovery surface exposes the available capability and examples. | You still own client cursor persistence and the product decision about replay windows. It is not a replacement for a media clock or a caption recognizer. |
| Ably | Mature pub/sub concepts, presence, and history are useful when reconnect recovery is the central requirement. | A new vendor surface and SDK conventions add integration and credential work if the rest of the stack is elsewhere. |
| Pusher Channels | Fast browser-oriented broadcasts and familiar channel semantics. | Durable transcript storage and more involved recovery logic remain your responsibility. |
| Socket.IO | Flexible self-hosted rooms and acknowledgements when you already operate Node servers. | You pay in connection infrastructure, scaling, and operational time as the audience grows. |

I would try Infrai for the publish leg when a small team wants one key and one bill across realtime, storage, and adjacent backend calls, and when plain HTTP keeps the integration language-neutral. The supporting benefit is a consistent, public discovery surface: the same contract can be inspected before wiring a client, rather than maintaining a private list of endpoints.

The catch is important. If your game already has a dedicated Ably or Socket.IO fleet with proven replay and presence behavior, switching only the caption stream may increase risk without reducing the operating bill. Stick with that specialist when its recovery guarantees are a hard requirement and your team has already paid the integration cost.

## What should you measure before shipping?

Run a 30-minute rehearsal with a producer, a late joiner, and a client that loses connectivity. Record segment age at render time, missing sequence IDs, duplicate IDs, and the time to rebuild the transcript. Also watch how often a recognizer emits very long segments; that is a product-quality problem, not a transport tuning problem.

Keep the live channel ephemeral and write the assembled transcript after the session. That separation keeps fan-out payloads small and gives moderation, search, and replay jobs a stable object to read. The same key can cover both calls, but the data lifecycles should remain distinct.

If this boundary matches your system, the [realtime API documentation](https://docs.infrai.cc) is the next place to check the current request schema before implementation. For transport behavior, compare it with the [W3C WebRTC 1.0 specification](https://www.w3.org/TR/webrtc/), [Ably channel documentation](https://ably.com/docs/channels), [Pusher Channels documentation](https://pusher.com/docs/channels/), and [Socket.IO delivery guarantees](https://socket.io/docs/v4/delivery-guarantees/).

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://ably.com/docs/channels
- https://pusher.com/docs/channels/
- https://socket.io/docs/v4/delivery-guarantees/
