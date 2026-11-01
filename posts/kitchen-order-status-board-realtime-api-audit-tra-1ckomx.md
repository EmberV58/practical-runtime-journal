# Kitchen Order Status Board Realtime API Audit Trail for Reconnecting Kiosks

For a kitchen order status board, the realtime API should make each order state auditable when a kiosk reconnects. The same contract works for an education chat room: retain the event, assign a room sequence, and show whether the display applied it in order. A connection count cannot prove that.

Short answer: commit each message to room history before sending it, give it a monotonically increasing room sequence, and make a reconnecting display resume from its last contiguous applied sequence. A live socket is a delivery attempt. The cursor is the recovery contract. The board needs the current ordered state, not a comforting count of open connections.

Before, the broadcaster loops through sockets and calls the job done when the loop finishes. After, the room can answer three separate questions: was the event accepted, was it sent to a live subscriber, and was it applied in order? Those are different facts. Keep them different in code, logs, and dashboards.

That distinction matters.

Not enough.

## How should a kitchen order status board use a realtime API?

Start with the vocabulary. A message is accepted when authorization passes and the room writer assigns its sequence. It is live-sent when a transport accepts the send operation. It is applied when a client has processed the event and advanced its cursor without a gap. "Delivered" is too vague for an incident review.

The flow looks like this in words: room writer creates sequence 18, history stores 18, fan-out attempts 18 for eligible subscribers, one client applies 18 and acknowledges 18, another reconnects at 17 and receives 18 through repair. The second client was not live-sent, but it can still reach the same state. That is the useful outcome.

I initially expected the transport choice to decide reliability. It does not. Transport changes how bytes move. It does not create room history, define authorization, or tell the server whether a display applied a message after a reconnect.

Use small, bounded metric dimensions. `room_type`, `transport`, and `outcome` can support useful cuts. Room IDs, student IDs, and message bodies do not belong in metric labels. They belong in carefully controlled diagnostic records, with message content redacted. A classroom can produce enough unique identifiers to make an otherwise good dashboard difficult to store and query.

The minimum useful signals are committed room sequences, live-send outcomes, replay counts, repair age for active clients, and the highest contiguous cursor per subscriber. Add a correlation ID to the logs for one repair. That lets an operator follow a gap without turning the observability system into a copy of the chat database.

## What does a reconnecting client actually promise?

The promise should be narrow and testable: a client that presents cursor 17 receives the authorized records after 17, applies them in sequence, ignores duplicates it has already applied, and acknowledges only the highest contiguous sequence. It does not promise exactly-once network delivery. Network frames can be delayed, duplicated, or dropped.

Here is the protocol shape without tying it to a commercial service or a particular route:

```ts
type RoomEvent = {
  roomId: string;
  seq: number;
  id: string;
  senderId: string;
  body: string;
  createdAt: string;
};

type ResumeRequest = {
  roomId: string;
  lastAppliedSeq: number;
  clientId: string;
};

function applyEvent(state: RoomEvent[], event: RoomEvent): RoomEvent[] {
  const lastSeq = state.at(-1)?.seq ?? 0;

  if (event.seq <= lastSeq) return state;
  if (event.seq !== lastSeq + 1) {
    throw new Error("ordered repair required");
  }

  return [...state, event];
}
```

The client-side rule is deliberately boring. It makes the failure visible. If sequence 19 arrives while 18 is missing, the display must stop advancing its rendered history and ask for the missing range. Applying 19 immediately produces a plausible screen with a hidden hole, which is much harder to diagnose than a visible repair state.

A scoped subscriber token should carry enough authorization context for the server to check the room, role, client identity, and expiry. The server must enforce that scope during both live join and history repair. A token that protects the socket handshake but permits arbitrary replay has left the recovery path weaker than the live path.

The token is not an acknowledgement. It proves what the server is willing to let the client request. The client cursor proves what that client says it has applied. Keep those meanings separate.

## How do you test fan-out without mistaking activity for delivery?

Test the timeline, not only the happy-path broadcast. Create three subscribers. Publish a burst. Disconnect one after sequence 2. Reconnect it with cursor 2. The expected repair contains every authorized event from sequence 3 through the room head, in order. Then replay the same response twice and verify that application remains unchanged.

Add the awkward cases that expose a weak contract: expiry between live delivery and repair, two reconnects using the same client identifier, a subscriber outside the room scope, and a gap followed by a later live event. The outside subscriber must receive neither live events nor replay data. The client with a gap must not advance its contiguous cursor merely because a newer event arrived.

The test should assert outcomes at each boundary. A stored event with no live-send record is not automatically a loss. A live-send record with no applied acknowledgement is not proof that the display rendered it. Those assertions force the team to name the guarantee it actually operates.

In a 2026 Node.js deployment, test the board, order sequence, and kiosk reconnect path as one workflow. Set repair-age objectives from that workflow and its traffic profile. Do not copy a latency number from another system. A kitchen board may tolerate a different repair window from an active classroom discussion, and both may need a separate rule for a display that has been offline for longer than retained history.

## Does retained history make the system too heavy?

It adds real work: a retention policy, an index suitable for ordered reads, cleanup, and a decision about what happens when a cursor is older than the retained range. That cost is visible. The alternative is also a decision, just a less useful one: after a missed broadcast, the client has no authoritative events from which to reconstruct its state.

The trade-off is explicit. This design is a poor fit for a transient signal that can be safely recomputed and does not need ordered recovery. For a chat message or an order transition that changes what a person sees, the retained event is part of the correctness story. For a disposable presence hint, retaining every update may add complexity without adding user value. The limitation is intentional: replayable history serves correctness, while disposable hints may not justify it.

Keep the writer and fan-out worker independent. Commit the event once, then let sends proceed without making room progress wait for the slowest subscriber. Record a lagging cursor and repair it. This avoids turning one poor network path into back pressure for every student in the room.

There is a second boundary around privacy. Logs can carry message ID, room sequence, and correlation ID while omitting message bodies. The same identifiers are enough to trace a repair. Classroom content is not necessary diagnostic metadata, and a kitchen order payload can carry operational details that do not belong in a metric system either.

## Which transport belongs underneath the contract?

Choose the transport after defining the application evidence. WebSocket provides a full-duplex browser connection. WebRTC data channels provide another browser networking model with different setup and connectivity considerations. Neither one supplies room history, scoped authorization, or contiguous application acknowledgements.

That separation keeps a migration practical. The room writer, cursor rules, repair behavior, and metrics can remain stable while the transport adapter changes. It also improves developer experience: a test can inject a delayed, duplicated, or dropped transport event without pretending that a particular network library is the source of truth.

The decision rule is concrete. When a student loses connectivity at sequence 42, the system should be able to show that the client reconnected with cursor 42, passed the room-scope check, applied the missing ordered range, and advanced its acknowledgement only through the last contiguous message. If the same sentence works for a kitchen status display, the design is describing a delivery contract rather than a product feature.

That is the dashboard change worth making: replace "connected sockets" and "broadcast attempts" with committed sequences, live-send outcomes, repair age, replay counts, and contiguous cursors. A display can be temporarily absent without becoming mysteriously wrong. The system has a record of the gap, a bounded way to repair it, and an operator-facing account of each boundary.

## Sources

References:

- https://www.w3.org/TR/webrtc/
- https://www.rfc-editor.org/rfc/rfc6455
- https://www.rfc-editor.org/rfc/rfc7519
- https://developer.mozilla.org/en-US/docs/Web/API/WebSocket
