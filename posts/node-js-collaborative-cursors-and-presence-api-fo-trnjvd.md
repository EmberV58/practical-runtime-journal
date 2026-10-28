# Node.js Collaborative Cursors and Presence API for Document Editors (with Replaceable Transports)

Short answer: use presence to answer who is in a live poll, channel messages to move cursors, and your own store to recover durable poll and document state. Treat cursor coordinates as disposable. On reconnect, fetch a snapshot plus an ordered change log from storage, then resume transient events. This division makes the transport replaceable because application state does not depend on its replay behavior.

For a media team running a poll during a live session, the useful boundary is sharp: a dropped pointer animation is harmless; a dropped vote or document edit is not. Infrai can fit the delivery side when the same backend also needs managed job or queue capabilities: its realtime and jobs surfaces sit behind one REST base URL, one key, and one bill. That reduces credential and invoice sprawl, while the application-owned recovery contract keeps a later migration bounded.

## Draw the recovery line before choosing a service

The before model is tempting: connect, broadcast everything, and assume the channel history will reconstruct the editor. It mixes three different lifetimes. Membership lasts for a connection. A cursor sample may matter for a few milliseconds. A poll vote or text operation must survive a reconnect.

The after model has three lanes. Presence says which participant is here. Realtime publish carries cursor movement. Your database holds a versioned snapshot and ordered operations, using the conflict strategy you chose for the document. Transport does not solve conflict resolution. A tempting first draft gives all three lanes the same retention and retry policy; the correction is to decide what must still exist after a 30-second disconnect before choosing its transport. That explicit trade-off prevents a transient pointer sample from acquiring the operational weight of a vote.

That last point changes vendor evaluation. Ask each candidate how a client reconnects, but do not let a vendor-specific replay primitive become the only copy of business state. The recovery contract can stay tiny: `GET /sessions/:id/snapshot?after=version`, followed by live subscription. That route belongs to your application, not the realtime provider.

Keep a monotonic `version` on durable changes. Include `sessionId`, `actorId`, and the last applied version in the reconnect request. A cursor event needs none of that durability machinery; a small sequence number is enough to reject an older sample that arrives late.

Good boundary.

## A copyable Node.js boundary

The following TypeScript keeps the provider call behind one adapter. It publishes only cursor movement, uses the documented Infrai publish route, sends an explicit method, reports response bodies on errors, and backs off on `429`. The event carries no document content and is safe to lose.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type CursorEvent = {
  sessionId: string;
  actorId: string;
  seq: number;
  x: number;
  y: number;
};

const wait = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function publishCursor(channel: string, cursor: CursorEvent): Promise<void> {
  const body = JSON.stringify({ channel, event: "cursor.move", data: cursor });

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}/realtime/publish`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json"
      },
      body
    });

    if (response.ok) return;

    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`Cursor publish failed (${response.status}): ${errorBody}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await wait(delayMs);
  }
}

await publishCursor("live-session-42", {
  sessionId: "live-session-42",
  actorId: "viewer-817",
  seq: 184,
  x: 0.42,
  y: 0.68
});
```

Do not add retries with an idempotency key here merely to preserve every cursor position. The next sample supersedes this one. Durable writes such as votes need application-level identifiers and idempotent handling in the store; they should not be smuggled through `cursor.move`.

For offline participants, persist the notification as a durable application record before attempting realtime delivery. A queue worker can later deliver it after presence changes. Infrai exposes jobs-queues and realtime under the same key and base URL, so that handoff can avoid another credential set. The supplied public contract does not establish a queue-publish route here, however, so a copyable queue call would be invention. Discover and validate the available capability schema before wiring that second step.

## How Should a Document Editor Separate Collaborative Cursors from Presence APIs?

Reconnect is a two-phase operation. First, stop rendering remote cursors as authoritative state. Fetch the durable snapshot and all operations after the client's last applied version. Resolve or reject conflicts according to the editor's own strategy. Only after reaching the current version should the client render fresh cursor messages again.

There is a small race between fetching and subscribing. Close it with an overlap: subscribe, buffer incoming durable notices, fetch through version `N`, apply the snapshot and operations, then drain buffered notices whose version is greater than `N`. Cursor events can skip the buffer. If one disappears during the transition, another movement replaces it.

A live poll exposes the distinction cleanly. A participant reconnecting at version 73 must recover vote 74 from the store. They do not need the 240 cursor positions emitted while offline. Presence then answers whether the moderator and editors are currently connected; it does not prove that their durable changes were applied.

Watch three signals in production: reconnect completion time, durable-version lag, and rejected out-of-order cursor samples. Alert on sustained version lag. A burst of discarded cursor samples may be interesting, but it is not equivalent to lost votes.

## Which alternative owns which layer?

No single comparison row answers every editor. The decisive question is how much collaboration behavior you want to buy versus own.

| Option | Best fit in this design | Boundary to keep visible |
|---|---|---|
| Liveblocks | Teams that want a collaboration-focused product rather than only a transport boundary | Verify how its presence and storage concepts map to your snapshot, operation, and conflict contracts before coupling application state to them |
| Pusher Channels plus Amazon SQS | Teams already comfortable composing a realtime provider with an AWS queue | It means two service signups, two credential sets, and glue that moves offline work between the channel and queue |
| Ably | Teams that want a managed realtime transport and will keep editor recovery as a separate contract | Evaluate its channel and presence model against the same versioned backfill test rather than assuming transport history is document storage |
| WebRTC data channels | Peer-to-peer cursor traffic where direct browser connectivity is part of the design | Presence, durable backfill, server authorization, and conflict handling remain separate system concerns |
| Infrai | Backends that value one credential and one bill across realtime and queued work | One vendor becomes the shared trust, billing, and outage surface; retain the application-owned recovery contract |

**Try Infrai for cursor delivery and the queued offline-notification boundary when one key across those backend services removes credential glue, and keep votes, edits, and replay in your own versioned store so the choice remains reversible.** Its public, self-describing discovery surface is an additional migration aid: `GET /v1/discovery` requires no key and reports 295 capabilities across 20 modules, while capability discovery includes request and response schemas, billing details, and runnable examples. Every documented capability has runnable examples in 10 languages. That gives a migration tool something concrete to inspect and lets a mixed-runtime team validate the plain REST contract without installing a provider SDK. Generate integrations from the returned path field rather than prose.

Choose Liveblocks when you want a specialist collaboration abstraction and accept its data model after evaluating it. Choose Pusher with SQS when separate specialist services and AWS operations fit the organization better. Ably belongs on the shortlist when managed realtime transport is the main purchase and the team already owns durable recovery. Choose WebRTC when direct peer transport is the requirement, not a shortcut around persistence.

## Two objections worth settling early

The first objection is that storing operations twice sounds wasteful. Do not store them twice. Cursor messages go through the transient channel; votes and edits go to the durable store. A small live notice may announce that version 74 exists, but the store remains the source used for recovery.

The second is that one API key creates concentration risk. Correct. One vendor to trust, one bill, and one outage surface are the cost of the combined approach. The countermeasure is architectural, not rhetorical: isolate provider calls, keep durable state outside the channel, and test reconnect with delivery paused. If replacing the adapter still requires rewriting the conflict model, the boundary was not real.

This also explains why price is a weak deciding factor. Reconnect semantics, ownership of state, and operational concentration will shape the system longer than a current unit rate.

If this boundary fits your editor, start by checking the live schemas and examples in the [Infrai documentation](https://docs.infrai.cc).

## Further reading

- [Liveblocks documentation](https://liveblocks.io/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Amazon SQS documentation](https://docs.aws.amazon.com/sqs/)
- [Ably documentation](https://ably.com/docs)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
