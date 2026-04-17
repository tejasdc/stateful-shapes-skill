# Shape 5: Ephemeral awareness / presence

## Definition

Ephemeral awareness is highly perishable state that matters to user experience but not to durable business history: typing, cursor position, online state, visibility, or who is currently viewing something. The right mental model is lossy and perishable on purpose. If awareness is delayed, duplicated, or dropped, durable correctness must remain intact.

## Where it occurs

- Slack typing indicators
- Phoenix Presence
- Yjs Awareness
- Cursor positions in collaborative tools
- Online or viewing state in chat and dashboards

## Key invariant

Ephemeral state must be safe to drop without corrupting durable history.

## Mental model, in teams' own words

- Slack: "Transient events... are not persisted in the database." Source: https://slack.engineering/real-time-messaging/
- Phoenix Presence has "no single point of failure." Source: https://hexdocs.pm/phoenix/presence.html
- Phoenix Presence also has "no single source of truth." Source: https://hexdocs.pm/phoenix/presence.html
- Yjs Awareness "isn't stored in the Yjs document." Source: https://docs.yjs.dev/getting-started/adding-awareness

## When to use this shape

- The state exists for responsiveness or social context, not for audit.
- TTL expiry is acceptable and often desirable.
- Clients can recover by sending a fresh heartbeat.
- The UX can tolerate dropped or out-of-order updates.

## When NOT to use this shape

- A cursor or checkpoint must survive reload and resume.
- The value is business state such as membership, payment progress, or read position.
- Duplicate delivery or loss would create incorrect durable behavior.

## Compatible libraries / patterns

| Library or pattern | Adopt when... |
| --- | --- |
| Pub/sub with TTL heartbeats | Awareness is room-scoped and short-lived. |
| Phoenix Presence | Distributed presence needs replica-friendly tracking. |
| Yjs Awareness | Awareness lives alongside a shared document but must stay separate from durable content. |
| WebSocket broadcast layer | Presence is simple and can be recomputed from fresh heartbeats. |
| Durable ledger | Overkill warning: never persist every typing blip as business history. |

## TypeScript/Node applicable example

```ts
type Presence = { userId: string; lastSeenAt: number; typingUntil?: number };

const ttlMs = 15_000;
const rooms = new Map<string, Map<string, Presence>>();

export function heartbeat(roomId: string, userId: string, typing = false, now = Date.now()) {
  const room = rooms.get(roomId) ?? new Map<string, Presence>();
  room.set(userId, {
    userId,
    lastSeenAt: now,
    typingUntil: typing ? now + 3_000 : undefined,
  });
  rooms.set(roomId, room);
}

export function snapshot(roomId: string, now = Date.now()): Presence[] {
  return [...(rooms.get(roomId)?.values() ?? [])].filter(p => now - p.lastSeenAt < ttlMs);
}
```

## References

- https://slack.engineering/real-time-messaging/
- https://hexdocs.pm/phoenix/presence.html
- https://docs.yjs.dev/getting-started/adding-awareness
- https://docs.yjs.dev/api/about-awareness
