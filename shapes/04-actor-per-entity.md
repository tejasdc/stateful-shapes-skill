# Shape 4: Single coordinator / actor per entity

## Definition

This shape assigns one logical owner to one mutable entity: room, cart, session, queue shard, or user. The coordinator serializes mutations through a mailbox or keyed execution path so correctness comes from ownership before it comes from retries or locks. The central question is "who is allowed to mutate this entity right now?"

## Where it occurs

- Cloudflare Durable Objects
- Orleans grains
- Erlang or OTP processes
- Akka actors
- Chat rooms, carts, and session coordinators

## Key invariant

For a given entity, there is exactly one live mutation coordinator at a time.

## Mental model, in teams' own words

- Cloudflare: "Use Durable Objects for stateful coordination, not stateless request handling." Source: https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/
- Cloudflare: "Each Durable Object has a globally-unique name." Source: https://developers.cloudflare.com/durable-objects/concepts/what-are-durable-objects/
- Cloudflare: "Strong consistency - operations must be serialized to avoid race conditions." Source: https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/
- Orleans actors are always addressable by key. Source: https://learn.microsoft.com/en-us/dotnet/orleans/overview

## When to use this shape

- Many callers want to mutate one entity concurrently.
- The simplest honest rule is "all writes for entity X go here."
- Most invariants are entity-local rather than cross-system consensus problems.
- Async re-entrancy or stale captured state keeps causing bugs.

## When NOT to use this shape

- Several replicas must merge concurrent edits to one shared document.
- The hard part is durable multi-step progress with timers and retries.
- The system needs shared transactional consensus across many entities at once.

## Compatible libraries / patterns

| Library or pattern | Adopt when... |
| --- | --- |
| In-process keyed mailbox | One process or service can own coordination today. |
| Durable Objects | The runtime can host globally addressable single-threaded coordinators. |
| Orleans / OTP / Akka | The system already embraces actor runtime semantics. |
| OCC plus lease | You need serialization pressure relief before a full actor runtime. |
| One global actor | Overkill warning: never funnel the whole system through one mailbox. |

## TypeScript/Node applicable example

```ts
export class KeyedMailbox<K> {
  #queues = new Map<K, Promise<unknown>>();

  dispatch<T>(key: K, op: () => Promise<T>): Promise<T> {
    const prev = this.#queues.get(key) ?? Promise.resolve();
    const next = prev.then(op, op);
    this.#queues.set(key, next.catch(() => undefined));
    return next;
  }
}

const rooms = new KeyedMailbox<string>();

export function mutateRoom(roomId: string, op: () => Promise<void>) {
  return rooms.dispatch(roomId, async () => {
    await op(); // exactly one mutator for this room runs at a time
  });
}
```

## References

- https://developers.cloudflare.com/durable-objects/concepts/what-are-durable-objects/
- https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/
- https://learn.microsoft.com/en-us/dotnet/orleans/overview
- https://www.erlang.org/docs/26/design_principles/users_guide.html
- https://doc.akka.io/guide/concepts/akka-actor.html
