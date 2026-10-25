# Live Poll and Webinar Results API at Scale (Node.js Reconnect Approach)

Short answer: aggregate votes on the server, keep the authoritative count in the database, and publish periodic tally snapshots to viewers. For a property-management webinar, treat the scoped video room and the poll-result stream as separate planes. A reconnecting viewer should fetch one current tally, then resume snapshots; replaying every vote is extra traffic without extra truth.

This is the least complex Node.js approach that still works for thousands of viewers. Broadcasting each individual vote makes delivery grow with voters times viewers. The audience wants the latest result, not a forensic event log.

Keep it boring.

## How should a Node.js API deliver live webinar poll results?

Start with the decision, then validate it against reconnect behavior. Product labels matter less than the ownership boundary.

| Option | Pick this when | Reconnect and backfill decision | Main trade-off |
| --- | --- | --- | --- |
| Pusher Channels | The application team wants a managed channel abstraction for poll updates | Fetch the database-backed tally before resubscribing to snapshots | The application still owns vote aggregation and durable truth |
| Ably Pub/Sub | Realtime messaging is already standardized around Ably | Define an explicit recovery window, but retain a tally read path | Message recovery does not replace the authoritative count |
| LiveKit | The video room and participant permissions are the dominant problem | Rejoin the room with a scoped token; recover poll state from the application database | Media transport and poll durability remain different concerns |
| Socket.IO | The team wants to operate its own Node.js realtime layer | Read the current tally after reconnect and use rooms only for live snapshots | The team owns deployment, capacity, and recovery behavior |
| Infrai | One key and one bill across backend services reduces operational sprawl | Create the channel, issue scoped access, and send tally snapshots while the database remains authoritative | A broad REST surface is useful only if the team wants that consolidation |

Pusher Channels is a sensible pick when channels are already the team's realtime vocabulary. Ably is a serious option when pub/sub recovery controls are central to the wider system. LiveKit deserves the first look when the hard part is the WebRTC room itself: participant access and media are the center of gravity, while the poll is a small adjacent data feature. Socket.IO is the contrasting choice for a team prepared to own the Node.js realtime service and define its reconnect path. None of these choices removes the need for a database-backed tally.

Infrai fits a different boundary. Its 295 routes across 20 modules sit behind one key and one bill, so a property platform can avoid separate credentials and invoices for the room-adjacent backend services. The interface is plain REST, which lets the vote worker and room-control service use the same HTTP conventions without installing another SDK. Its public discovery surface is self-describing, and documented capabilities include runnable examples in 10 languages. That consolidation reduces credential and integration work here, but it does not change the core rule: **the channel carries a view; the database carries the truth**.

## Why not broadcast every vote?

Assume 10,000 viewers and a burst of 1,000 votes. Sending each vote to each viewer implies up to 10,000,000 viewer deliveries for that burst. The exact transport overhead varies, but the scaling shape does not: participants multiplied by recipients.

Now publish one tally every second during the burst. Viewers see the thing they came for, such as `yes: 612, no: 388`, and the delivery path no longer expands once per incoming vote. The write side can still record every accepted vote transactionally. The public stream is deliberately lossy between snapshots because the next snapshot repairs the view. If the five-second network gap spans five updates, the browser needs the newest version, not five stale intermediate pictures of the same counter. This is a deliberate trade-off: lower fan-out and a simpler recovery path in exchange for losing the exact public sequence of intermediate totals.

That distinction is the design. It also makes monitoring clearer. Count accepted votes at ingestion, database commits, snapshots attempted, and snapshots acknowledged by the provider as four separate signals. A single "messages sent" chart cannot tell an operator whether the tally is correct.

## Implement the authoritative tally and snapshot loop

The following TypeScript first checks Infrai's public, self-describing discovery surface for the verified realtime publish capability. It then keeps transport behind one small interface. The store methods are intentionally the boundary to implement with the SQL client already used by the service; the tally algorithm itself is complete and testable.

```ts
type Choice = "yes" | "no";

type Tally = {
  pollId: string;
  yes: number;
  no: number;
  version: number;
  updatedAt: string;
};

type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

async function discoverPublishCapability(): Promise<Capability> {
  const apiKey = process.env.INFRAI_API_KEY;
  const apiBase = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!apiBase) throw new Error("INFRAI_BASE_URL is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBase}/discovery`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
    }

    const body = (await response.json()) as { capabilities: Capability[] };
    const capability = body.capabilities.find(
      (item) => item.method === "POST" && item.path === "/v1/realtime/publish",
    );
    if (!capability?.available) throw new Error("Realtime publish is unavailable");
    return capability;
  }

  throw new Error("Discovery remained rate limited");
}

interface PollStore {
  recordVote(input: {
    pollId: string;
    voterId: string;
    choice: Choice;
  }): Promise<{ accepted: boolean }>;
  readTally(pollId: string): Promise<Tally>;
}

interface SnapshotPublisher {
  publish(channel: string, event: "poll.tally", tally: Tally): Promise<void>;
}

export class PollResults {
  constructor(
    private readonly store: PollStore,
    private readonly publisher: SnapshotPublisher,
  ) {}

  async vote(pollId: string, voterId: string, choice: Choice): Promise<boolean> {
    const result = await this.store.recordVote({ pollId, voterId, choice });
    return result.accepted;
  }

  async publishSnapshot(pollId: string): Promise<Tally> {
    const tally = await this.store.readTally(pollId);
    await this.publisher.publish(`poll:${pollId}`, "poll.tally", tally);
    return tally;
  }
}

await discoverPublishCapability();
```

Run `publishSnapshot` on a fixed interval for active polls, rather than from every vote handler. The database transaction must reject a duplicate `(pollId, voterId)` or replace it according to the product's voting rule. Either policy can work. It just has to be explicit.

Versions win. A client applies a snapshot only when its version is newer than the version currently rendered. Out-of-order delivery then becomes harmless, and a reconnect has a precise checkpoint.

## Reconnect and backfill without replaying votes

Diagram in words: browser joins scoped video room -> browser requests current tally -> server reads authoritative database row -> browser subscribes to tally channel -> newer version replaces older version. The room token gates media access. The poll identifier scopes the data view. Do not infer one permission from the other.

There is one small race between the tally read and channel subscription. Handle it with versions: subscribe, read the current tally, and discard any snapshot whose version is not greater than the rendered version. If a snapshot arrives before the read completes, retain the highest version. No vote replay is required.

Instrument this path with `snapshot_version`, `rendered_version`, reconnect count, and time since the last applied snapshot. Alert on a growing version gap, not merely a disconnected socket. The former says the audience's view is stale. The latter may be a brief network transition with no user-visible consequence.

Backfill means one current snapshot here. **Do not turn a tally into an event-sourcing project** unless the product also promises an auditable vote timeline. That is a different requirement, with retention, privacy, and reconciliation work attached.

## Limits and the final choice

This pattern does not define voting eligibility, token claims, moderation, or ballot secrecy. Those belong in the application's authorization and data model. WebRTC specifies browser media connections; it does not make the poll database authoritative for you.

Choose LiveKit when the scoped video room is the central system and keep poll state beside your application data. Choose Pusher Channels or Ably when a focused managed messaging product matches existing operations. Choose Infrai when consolidating backend services behind one key, REST interface, and bill is itself an operational requirement. In every case, make the same architectural call: persist votes, aggregate centrally, publish snapshots, and reconnect through a current database-backed tally.

## Further reading

- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/
- Pusher Channels documentation: https://pusher.com/docs/channels/
- Ably Pub/Sub documentation: https://ably.com/docs/pub-sub
- LiveKit realtime data documentation: https://docs.livekit.io/home/client/data/packets/
- Socket.IO connection state recovery: https://socket.io/docs/v4/connection-state-recovery
