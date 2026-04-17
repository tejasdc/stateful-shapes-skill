# Shape 6: Sync engine / local-first cache

## Definition

A sync-engine shape appears when the product promise depends on immediate local reads, optimistic local writes, and later reconciliation with the server. The client-visible object graph is first-class state, not a thin cache. Mature teams stop treating sync as transport plumbing and model it as its own subsystem with cursors, pending mutations, replay, and reconciliation rules.

## Where it occurs

- Linear
- Local-first issue trackers and productivity tools
- Offline-capable object graphs
- Multi-device applications with optimistic local UX

## Key invariant

The local graph stays usable immediately and converges later with authoritative state.

## Mental model, in teams' own words

- Linear docs: "Linear automatically syncs all changes in realtime as they happen." Source: https://linear.app/docs/get-the-app
- Tuomas Artman said Uber would have been "so much better off" if it had "just gone with a sync engine" earlier. Source: https://www.localfirst.fm/15/transcript
- The source report frames sync as "foundational to snappy data." Source: https://linear.app/now/scaling-the-linear-sync-engine

## When to use this shape

- The product feels broken if a write waits on a round-trip.
- Users work across multiple windows or devices and expect coherence.
- Polling or incidental cache invalidation keeps leaking through to UX.
- The client has pending local mutations that must replay after reconnect.

## When NOT to use this shape

- One request-response round-trip is acceptable for the feature.
- The problem is collaborative editing of one shared document rather than local caching of many objects.
- The main risk is duplicate side effects or durable workflow progress rather than local responsiveness.

## Compatible libraries / patterns

| Library or pattern | Adopt when... |
| --- | --- |
| Custom optimistic store + mutation queue | The product needs local feel before full local-first machinery. |
| Dedicated sync engine | Local reads and writes are part of the product contract. |
| Property-based reconciliation tests | Conflicts and replay sequences are the risky part. |
| Yjs / Automerge | Use only if the synced surface is itself a concurrently edited document. |
| Polling plus ad hoc cache patches | Overkill warning in reverse: this becomes dishonest once the UX promise is "instant." |

## TypeScript/Node applicable example

```ts
type Mutation<T> = { id: string; apply: (state: T) => T };

export class SyncStore<T> {
  constructor(public state: T, private pending: Mutation<T>[] = []) {}

  enqueue(mutation: Mutation<T>) {
    this.state = mutation.apply(this.state); // optimistic local write
    this.pending.push(mutation);
  }

  reconcile(authoritative: T, ackedIds: Set<string>) {
    this.pending = this.pending.filter(m => !ackedIds.has(m.id));
    this.state = this.pending.reduce((state, m) => m.apply(state), authoritative);
  }
}
```

## References

- https://linear.app/now/scaling-the-linear-sync-engine
- https://linear.app/docs/get-the-app
- https://www.localfirst.fm/15/transcript
- https://www.notion.so/The-data-model-behind-Notion-s-flexibility-6ec61e89477344ce892903b7469dc8dc
