# Node.js Screen Sharing Sessions — API Permissions for Clinical Cursor Recovery in 2026

Short answer: keep screen capture in the WebRTC media path, but authorize publishing with a server-issued capability and a monotonically increasing revision. A participant can publish only when the current session snapshot grants `screen:publish`. On reconnect, fetch that snapshot first, backfill cursor presence second, and resume incremental events only after the client has accepted the latest revision.

That order matters in a healthtech editor. A collaborative cursor is ephemeral editor state; a shared screen may expose a patient chart. Membership in the room is not proof that a participant may publish a screen, and a cursor-presence message must not become an accidental media permission.

## How should a Node.js screen sharing session API handle reconnects?

Use one authoritative snapshot instead of unrelated booleans. It should identify the session, authenticated participant, grant revision, and exact capabilities. A clinician may have `cursor:write`; a participant approved for a review call may also have `screen:publish`. Keep those grants separate.

The revision is the recovery mechanism. If an old WebSocket message arrives after a revoke, it must not make a newer decision older. This Node.js example keeps the policy at the application boundary and leaves signaling transport implementation-specific. The boundary is deliberate.

The awkward part is the handoff.

For a concrete recovery trace, consider a reviewer who was granted screen publishing at revision 41, then lost connectivity while an administrator revoked it at revision 42. The reconnecting browser may still hold an old capability token, an open peer connection, and a cursor watermark from before the revoke. Those three pieces of local state cannot be trusted equally: the watermark is useful for requesting presence deltas, the peer connection needs cleanup, and the capability must be revalidated. Fetching one authoritative snapshot lets the client make those different decisions in one ordered transition instead of letting whichever callback runs first decide what the participant may do. The same transition also prevents a cursor backfill from being mistaken for permission to resume media.

```ts
type Capability = "cursor:write" | "screen:publish";

type GrantSnapshot = {
  sessionId: string;
  participantId: string;
  revision: number;
  capabilities: ReadonlySet<Capability>;
};

type PublishRequest = {
  sessionId: string;
  participantId: string;
  grantRevision: number;
};

function approveScreenPublish(
  grant: GrantSnapshot,
  request: PublishRequest,
): { ok: true; revision: number } | { ok: false; code: "STALE_GRANT" | "FORBIDDEN" } {
  if (grant.revision !== request.grantRevision) {
    return { ok: false, code: "STALE_GRANT" };
  }

  const matchesIdentity =
    grant.sessionId === request.sessionId &&
    grant.participantId === request.participantId;

  if (!matchesIdentity || !grant.capabilities.has("screen:publish")) {
    return { ok: false, code: "FORBIDDEN" };
  }

  return { ok: true, revision: grant.revision };
}
```

The authenticated server identity must supply `participantId`; accepting it from an untrusted browser would defeat the check. This function also does not replace authentication, authorization at the API boundary, or media encryption. It is the narrow stale-grant rule.

## How does reconnecting preserve the right state?

Imagine a reviewer reconnecting after grant revision 41 while an administrator has revoked screen publishing at revision 42. The client must not renegotiate from its local state. The recovery sequence is:

1. Establish authenticated transport and identify the session.
2. Fetch or receive the complete grant snapshot.
3. Stop any local screen sender that the new snapshot does not permit.
4. Backfill current cursor presence and receive its event watermark.
5. Resume cursor deltas from that watermark.
6. Offer a new capture track only after the state is current.

Presence is ephemeral. Return each active participant's latest cursor position and a watermark; do not replay an unbounded history of pointer movements. The cursor endpoint should not carry document content merely because it is convenient. A reviewer needs the current collaboration state, not a reconstruction of every pointer position since the disconnect.

```ts
type SyncState =
  | { kind: "offline" }
  | { kind: "authorizing" }
  | { kind: "synced"; grantRevision: number; cursorWatermark: string };

function acceptSnapshot(
  state: SyncState,
  snapshot: { grantRevision: number; cursorWatermark: string },
): SyncState {
  if (state.kind === "synced" && snapshot.grantRevision < state.grantRevision) {
    return state;
  }

  return {
    kind: "synced",
    grantRevision: snapshot.grantRevision,
    cursorWatermark: snapshot.cursorWatermark,
  };
}

function shouldStartCapture(
  state: SyncState,
  grant: GrantSnapshot,
): boolean {
  return (
    state.kind === "synced" &&
    state.grantRevision === grant.revision &&
    grant.capabilities.has("screen:publish")
  );
}
```

The browser's `getDisplayMedia()` call should happen after this transition. It prompts the user and returns a media stream; it does not create the application's durable authorization policy. When the user stops a track, publish a stop event and release session-side screen state. A revoke should trigger the same cleanup even if the browser track has not ended.

Start from current state.

## What failure does revisioning prevent?

It separates transport delivery from authority. Messages can be delayed, duplicated, or observed after reconnect. Every grant update carries an ordering key, so a client can ignore an older update without guessing which event arrived first. Logs can join `sessionId`, `participantId`, and `grantRevision` to show which decision was evaluated.

Revisioning does not eliminate the race between an authorization check and media negotiation. A revoke can occur after the check and before the peer connection finishes. The service that controls membership and signaling therefore needs a revocation signal, and the client must stop the sender when it receives one. Browser UI is a useful second line of defense, not the authority.

Log decisions, not pixels: decision code, authenticated participant, session ID, grant revision, reconnect count, and cursor watermark. Keep document text and captured-frame metadata out of routine logs. Test an old revision after revoke, duplicate grant updates, reconnect without screen capability, participant removal during capture, and a browser track ending without a server event. Add a property that client revisions never move backward.

Let the Node.js service own identity, snapshots, capability revisions, signaling messages, and audit decisions. Let WebRTC own media transport and negotiation. Let the editor protocol own cursor snapshots and deltas. This boundary keeps a cursor update from becoming media authority.

The operational checklist is concrete: authenticate before resolving the participant; issue a complete snapshot; require its revision on publish requests; make revoke idempotent; stop or replace a sender when capabilities change; backfill cursors before consuming deltas; and measure stale-grant rejections, reconnect duration, snapshot age, and capture-stop events. Verify both directions: a participant can recover cursor visibility without screen permission, and screen permission does not grant document-editing authority.

The trade-off is real. This split adds a snapshot and a recovery state machine, so it is more work than treating room membership as permission. That extra state is a poor fit for a throwaway internal demo with no sensitive media, but it is a reasonable cost when a clinical editor must revoke screen sharing predictably. A single media service can simplify negotiation; it cannot remove the need to define who may publish or what happens after a disconnect.

That is the rule I would ship in 2026: permission is a versioned session capability, screen capture is a user-mediated WebRTC operation, and reconnect recovery starts from authoritative state. The media plumbing stays separate from the security decision, while the clinical editor gets a deterministic path back to the present.

## References

- https://www.w3.org/TR/webrtc/
- https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia
- https://www.rfc-editor.org/rfc/rfc6455
- https://www.rfc-editor.org/rfc/rfc8445
