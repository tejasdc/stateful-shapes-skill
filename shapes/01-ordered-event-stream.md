# Shape 1: Ordered event stream per entity

## Definition

An ordered event stream is an append-oriented history for one entity: a room, channel, issue, payment, or similar object. The hard problem is not "what status is this object in?" but "what happened next, and have we already applied it?" Mature systems make the ordering boundary local to the entity instead of pretending the whole product has one global sequence.

## Where it occurs

- Slack channels and threads
- Discord channels and message history
- Telegram chats and update streams
- Stripe object lifecycles and retried operations
- Linear issue history

## Key invariant

There must be one authoritative ordering boundary per stream.

## Mental model, in teams' own words

- Slack: "Channel Servers (CS) are stateful and in-memory." Source: https://slack.engineering/real-time-messaging/
- Slack: "Every CS is mapped to a subset of channels based on consistent hashing." Source: https://slack.engineering/real-time-messaging/
- Discord: messages are partitioned by "the channel they're sent in, along with a bucket." Source: https://discord.com/blog/how-discord-stores-trillions-of-messages
- Stripe: "The server receives the ID and correlates it with the state of the request on its end." Source: https://stripe.com/blog/idempotency

## When to use this shape

- The hardest questions sound like "what happened next in this room?"
- Duplicate delivery or replay would create duplicate business effects.
- Devices or transports may reconnect and need a catch-up cursor.
- The entity mostly grows by appending new facts rather than by concurrent shared editing.

## When NOT to use this shape

- Several people are concurrently editing the same object and merge intent matters. That is a collaborative-document shape.
- The real problem is retries, timers, and crash-safe progress across steps. That is a durable-workflow shape.
- The state is perishable UX awareness like typing or cursor position. That is an ephemeral-awareness shape.

## Compatible libraries / patterns

| Library or pattern | Adopt when... |
| --- | --- |
| Append-only event log + single writer | One owner per entity can assign order. Start here. |
| Keyed actor / mailbox | Many callers race to append to the same stream. |
| Idempotency keys | Retries can duplicate effects at the API or queue boundary. |
| Property-based invariant tests | The risk is out-of-order, duplicate, or skipped application across long sequences. |
| CRDT / OT | Usually overkill for append-only streams. Use only if the stream's entries themselves become collaboratively edited objects. |

## TypeScript/Node applicable example

```ts
type StreamEvent =
  | { kind: 'message.sent'; seq: number; body: string }
  | { kind: 'message.edited'; seq: number; targetSeq: number; body: string };

type StreamState = { nextSeq: number; events: StreamEvent[] };

export class StreamRepo {
  #streams = new Map<string, StreamState>();

  append(streamId: string, event: Omit<StreamEvent, 'seq'>): StreamEvent {
    const state = this.#streams.get(streamId) ?? { nextSeq: 1, events: [] };
    const stored = { ...event, seq: state.nextSeq++ } as StreamEvent;

    state.events.push(stored);
    this.#streams.set(streamId, state);
    return stored;
  }

  readFrom(streamId: string, afterSeq = 0): StreamEvent[] {
    return (this.#streams.get(streamId)?.events ?? []).filter(e => e.seq > afterSeq);
  }
}
```

## References

- https://slack.engineering/real-time-messaging/
- https://discord.com/blog/how-discord-stores-trillions-of-messages
- https://core.telegram.org/api/updates
- https://stripe.com/blog/idempotency
- https://martinfowler.com/eaaDev/EventSourcing.html
