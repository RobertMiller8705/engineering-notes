# Live Webinar Poll Results API: Sequence Checkpoints for Viewer Recovery

Use monotonic sequence checkpoints and bounded backfill for every webinar room. The deciding constraint is reconnect convergence: thousands of viewers can tolerate a briefly stale tally, but they can't safely recover from a gap if the server only pushes the newest delta.

**The practical answer is a snapshot-plus-tail protocol.** A Node.js service should keep vote acceptance separate from result distribution, publish aggregate snapshots rather than individual votes, and require every reconnect request to declare the last sequence the viewer applied. The same rule lets a media chat room recover missed messages without pretending that a live connection is a durable log.

## How should a live webinar poll results API recover viewers?

A connection proves reachability now. It doesn't prove that the browser received every earlier result, applied each one once, or retained state while a laptop slept. Mobile handoffs, background tabs, overloaded clients, and process restarts all create the same application-level question: what state does this viewer have?

Gaps happen.

The tempting first design broadcasts an increment after each accepted vote. It looks efficient. Miss one increment and the local tally stays wrong; replay one after reconnect and it becomes too high. A stream of perfectly valid messages can therefore produce an invalid projection.

Do less. The write path validates and deduplicates votes, updates an authoritative aggregate, then appends a result event with a poll-local sequence. The fan-out path distributes that aggregate. No viewer needs voter identity, and reconnecting doesn't rerun vote processing.

## The focused protocol

The experiment compares two recovery rules under forced disconnects. Rule A resumes with the latest live message and accepts whatever arrives next. Rule B sends the client's last applied sequence, then receives either the missing tail or a fresh aggregate snapshot. The pass condition isn't delivery count; it's eventual equality between the viewer tally and the authoritative tally.

That's the contract.

| Condition | Server response | Client action |
| --- | --- | --- |
| No checkpoint | Current snapshot | Replace local state |
| Checkpoint retained | Ordered tail | Apply in sequence |
| Checkpoint expired | Current snapshot | Replace local state |
| Duplicate sequence | No recovery needed | Ignore event |
| Sequence gap | Snapshot or complete tail | Pause and recover |

A compact event shape is enough:

```ts
type Tallies = Record<string, number>;

type ResultEvent = {
  roomId: string;
  pollId: string;
  sequence: number;
  tallies: Tallies;
  emittedAt: string;
};

type ResumeRequest = {
  roomId: string;
  pollId: string;
  afterSequence: number | null;
};

type ResumeReply =
  | { mode: "tail"; events: ResultEvent[] }
  | { mode: "snapshot"; event: ResultEvent };
```

The sequence belongs to one poll, not to a server process or a socket. Persist it with the aggregate update so a restart can't move the tally forward while leaving its checkpoint behind. A transaction or atomic datastore operation is the useful primitive; the database brand is secondary.

On the client, reject duplicates, detect gaps, and replace state on a snapshot. Keep that logic independent of WebSocket, server-sent events, long polling, or a WebRTC data channel. Transport adapters should only deliver bytes and expose connection state.

```ts
class PollProjection {
  private sequence = 0;
  private tallies: Tallies = {};

  apply(event: ResultEvent): "applied" | "duplicate" | "gap" {
    if (event.sequence <= this.sequence) return "duplicate";
    if (event.sequence !== this.sequence + 1) return "gap";

    this.tallies = { ...event.tallies };
    this.sequence = event.sequence;
    return "applied";
  }

  replace(event: ResultEvent): void {
    this.tallies = { ...event.tallies };
    this.sequence = event.sequence;
  }

  checkpoint(): number {
    return this.sequence;
  }
}
```

There is a specific trap here. Suppose the viewer last applied sequence `41`, disconnected, and then reconnects after the retained tail expired. The server returns a snapshot at sequence `57`. Calling `apply` rejects it as a gap because the reducer expects `42`, even though `57` is the correct authoritative replacement; calling `replace` moves both the tally and checkpoint together. Treating snapshots and tail events as interchangeable makes a clean-looking reducer reject the exact recovery message it requested, so keep the two operations distinct in code and tests.

## Bound the backfill on purpose

Retaining an infinite result log buys little for a short poll. Choose a window from observed disconnect duration and event rate, then make expiration explicit. If `afterSequence` is retained, return the ordered tail. If it is older than the first retained event, return one current snapshot. Never return a partial tail silently because the result may look plausible while remaining wrong.

This is also the main cost trade-off. Broadcasting one aggregate update at a controlled cadence makes outbound work depend on connected viewers and update frequency, rather than on every raw vote. A fast trivia round may justify more updates than a long panel survey. Don't encode that interval into correctness; recovery should work at any cadence.

Backpressure needs a policy. Give each connection a bounded outbound queue. When a slow viewer falls behind, discard superseded aggregate messages and send the newest snapshot with its sequence. Chat is different: each message may matter, so use retained backfill instead of coalescing. One room can share checkpoint machinery while applying different retention rules to poll aggregates and chat history.

The limitation is real: snapshot-plus-tail uses more storage and application logic than fire-and-forget broadcast. It is the wrong choice for disposable reactions where losing a few events has no user-visible consequence. It also doesn't create exactly-once delivery. It creates an idempotent projection that converges despite duplicate or missing transport deliveries.

## Failure injection is the useful test

Serializer tests are cheap, but they don't answer the selection question. Run a deterministic test that accepts votes while selected clients are offline, reconnects them from several checkpoints, and compares every final projection with the authoritative aggregate. Repeat with a server restart around aggregate publication.

Test five cases: a duplicate event, a one-event gap, a checkpoint inside retention, a checkpoint older than retention, and a slow consumer whose queue reaches its bound. Record convergence time from reconnect to matching state, snapshot frequency, retained events per active poll, bytes sent per viewer, queue depth, and reconnect attempts. Those measurements reveal whether a configuration fits the audience shape.

Thousands of simulated connections help only if their behavior resembles the event. Stagger reconnects, pause some clients, and create a burst near poll close. A test that opens every connection at once and keeps every reader perfect mostly measures the harness.

## Choosing the delivery layer

Start from directionality. Poll results are predominantly server-to-viewer, while vote submission is a separate authenticated request. A one-way streaming transport can fit that split. Bidirectional chat may make a full-duplex connection convenient. WebRTC can carry arbitrary peer data through `RTCDataChannel`, but peer topology, signaling, and recovery remain application decisions; the transport doesn't remove sequence and backfill requirements.

The selection checklist is short. Can the layer carry the checkpoint on reconnect, expose enough connection state to observe churn, enforce bounded buffering, and run through the deployment path already in use? Verify authorization at room join and again when serving backfill. A stale connection credential must not become permanent access to later poll results.

Ship the protocol before optimizing the pipe. Measure reconnect convergence, gap rate, snapshots per reconnect, queue pressure, and bytes per viewer in a production-shaped load test. Those numbers determine retention and publication cadence. A transport benchmark by itself doesn't.

## References

- https://www.w3.org/TR/webrtc/
