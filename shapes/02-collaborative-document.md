# Shape 2: Replicated collaborative document

## Definition

A collaborative-document shape appears when multiple users or replicas can edit the same logical object and the system must merge concurrent edits into one coherent state. The core question is not ordering alone; it is how to preserve user intent when edits overlap. Mature teams win by choosing the smallest honest merge unit: property, block, node, text span, or shared type.

## Where it occurs

- Figma files
- Notion block trees
- Yjs shared documents
- Automerge-backed local-first documents
- Google Docs style editors

## Key invariant

Concurrent edits must converge at the chosen merge boundary without losing intended changes.

## Mental model, in teams' own words

- Figma: "No more complex than necessary." Source: https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
- Figma: "A simpler system is easier to reason about." Source: https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
- Figma: "Figma isn't using true CRDTs" because the server is "the central authority." Source: https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
- Figma: "Changes are atomic at the property value boundary." Source: https://www.figma.com/blog/how-figmas-multiplayer-technology-works/

## When to use this shape

- Several users can edit the same object at the same time.
- Offline edits may later merge back into a shared object.
- Last-writer-wins on the whole object would lose intent users care about.
- The hard question is "how do these concurrent edits merge?" not "who owns this room?"

## When NOT to use this shape

- The entity is an append-only feed such as chat history or audit history.
- One authoritative writer can serialize mutations cheaply.
- The state is presence or typing information that should disappear when the client disconnects.

## Compatible libraries / patterns

| Library or pattern | Adopt when... |
| --- | --- |
| Central-authority merge logic | One server is authoritative and can narrow conflicts to a small merge unit. |
| Yjs | Offline-capable replicas need convergent shared editing. |
| Automerge | Network-agnostic CRDT merging is more important than keeping the model minimal. |
| Property-based merge tests | Merge rules are custom and need invariant testing. |
| Ordered stream only | Overkill warning: a pure event log is not enough once overlapping edits must converge by intent. |

## TypeScript/Node applicable example

```ts
type Node = { id: string; version: number; props: Record<string, string> };
type Doc = { nodes: Record<string, Node> };
type Patch = { nodeId: string; baseVersion: number; props: Record<string, string> };

export function applyPatch(doc: Doc, patch: Patch): Doc {
  const current = doc.nodes[patch.nodeId];
  if (!current) throw new Error('unknown node');

  // Central-authority simplification: merge at the property boundary.
  const merged: Node = {
    ...current,
    version: Math.max(current.version, patch.baseVersion) + 1,
    props: { ...current.props, ...patch.props },
  };

  return {
    ...doc,
    nodes: { ...doc.nodes, [patch.nodeId]: merged },
  };
}
```

## References

- https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
- https://www.notion.so/The-data-model-behind-Notion-s-flexibility-6ec61e89477344ce892903b7469dc8dc
- https://docs.yjs.dev/
- https://automerge.org/docs/hello/
- https://research.google/pubs/pub41926
