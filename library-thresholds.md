# Library-adoption thresholds

## The wrong question: "do we have 30 states yet?"

The count heuristic is misleading because mature teams do not adopt heavier primitives when diagrams get crowded; they adopt them when the failure mode changes shape. Figma avoided full CRDT machinery in a high-complexity product because its authority model let it simplify. Temporal adopts durable execution for workflows with modest label counts because retries, waits, and crash recovery change the problem class. Cloudflare uses actor-style coordination for chat-room-like entities because serialization is the pressure, not state-count.

Evidence: [Figma multiplayer](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/), [Temporal workflow introduction](https://web.temporal.io/blog/workflow-introduction), [Cloudflare Durable Objects](https://developers.cloudflare.com/durable-objects/concepts/what-are-durable-objects/), [Linear sync engine](https://linear.app/now/scaling-the-linear-sync-engine)

## The right question: has the failure mode changed shape?

Ask what is actually breaking:

- Ordering ambiguity for one entity
- Concurrent edits to one shared object
- Progress that must survive crashes, timers, or pauses
- Many writers mutating one entity at once
- Product quality depending on instant local reads and optimistic writes
- Invariants failing only after long sequences of operations

That framing points to the right primitive much faster than label-counting.

## Actor / coordinator systems

Pressure pattern: many concurrent code paths want to mutate one entity, and correctness depends on serialized ownership more than on global joins.

Who adopted it and why: Slack routes channels to stateful channel servers. Cloudflare Durable Objects and Orleans make one logical owner addressable by key. OTP and Akka provide the mailbox vocabulary for the same idea.

Threshold in evidence terms: adopt actor-like coordination when "who owns entity X right now?" is the real correctness question. This can be true with five states or fifty.

Anti-pattern warning: actors are a poor fit for truly shared transactional consensus across many entities. Do not create one giant actor for the whole system.

Sources: [Slack](https://slack.engineering/real-time-messaging/), [Cloudflare Durable Objects](https://developers.cloudflare.com/durable-objects/concepts/what-are-durable-objects/), [Orleans](https://learn.microsoft.com/en-us/dotnet/orleans/overview), [OTP](https://www.erlang.org/docs/26/design_principles/users_guide.html), [Akka transactors warning](https://doc.akka.io/libraries/akka/1.3.1/java/transactors.html)

## Statechart libraries

Pressure pattern: the state space inside one module becomes hard to reason about because hierarchical regions, orthogonal concerns, and guarded transitions are all real.

Who adopted it and why: the external-patterns report places statecharts higher on the heaviness ladder because they make explicit states, transitions, guards, and visualization first-class.

Threshold in evidence terms: adopt XState or a similar statechart tool when ad hoc unions are no longer readable because concurrency is hierarchical, not because the raw number of states feels embarrassing.

Anti-pattern warning: do not use a statechart library for every toggle or three-state UI flow.

Sources: [David Harel statecharts](https://dubroy.com/refs/Statecharts_a_visual_formalism_for_complex_systems.pdf), [Tim Deschryver on XState](https://timdeschryver.dev/blog/my-love-letter-to-xstate-and-statecharts), [Barr Group on hierarchical state machines](https://barrgroup.com/embedded-systems/how-to/introduction-hierarchical-state-machines)

## Document-sync systems

Pressure pattern: multiple people or replicas edit the same logical object, and merge intent matters.

Who adopted it and why: Figma narrowed the merge unit because the server is authoritative. Yjs and Automerge target the opposite case: network-agnostic or offline-capable convergence across replicas.

Threshold in evidence terms: adopt document-sync machinery when the same object is concurrently editable and users care how conflicts merge. The threshold is merge pressure, not the size of the object graph.

Anti-pattern warning: do not buy document-sync complexity for append-only chat, form submissions, or server-owned objects that one coordinator can serialize.

Sources: [Figma](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/), [Yjs](https://docs.yjs.dev/), [Automerge](https://automerge.org/docs/hello/)

## Durable workflow systems

Pressure pattern: progress must survive crashes, retries, timers, human waits, or long gaps between steps.

Who adopted it and why: Temporal, Cadence, Step Functions, Conductor, and DBOS all stop reconstructing process state from jobs and rows. Stripe's idempotency model shows the same lesson at the API boundary: retries need stable operation identity.

Threshold in evidence terms: adopt durable workflow machinery the moment the hard question becomes "what already happened, what is outstanding, and where do we resume?" This can happen with four states and can stay unnecessary with forty if everything is synchronous and single-owner.

Anti-pattern warning: AWS explicitly calls hidden orchestration inside handlers an anti-pattern. A `status` field is not workflow history.

Sources: [Temporal](https://web.temporal.io/blog/workflow-introduction), [Cadence](https://cadenceworkflow.io/docs/concepts/workflows), [AWS Step Functions](https://aws.amazon.com/documentation-overview/step-functions/), [AWS anti-pattern post](https://aws.amazon.com/blogs/compute/streamlining-aws-serverless-workflows-from-aws-lambda-orchestration-to-aws-step-functions/), [Conductor](https://docs.conductor-oss.org/devguide/concepts/workflows.html), [DBOS](https://www.dbos.dev/blog/what-is-lightweight-durable-execution), [Stripe](https://stripe.com/blog/idempotency)

## Sync engines

Pressure pattern: the product promise depends on instant local reads, optimistic local writes, and later reconciliation.

Who adopted it and why: Linear treats the sync engine as product infrastructure. Tuomas Artman's retrospective is explicit that polling-and-patching approaches fail to deliver the product feel.

Threshold in evidence terms: adopt a real sync layer when "snappy local feel" is part of the product contract. The threshold is not offline purity or state count; it is whether incidental cache invalidation has already become dishonest.

Anti-pattern warning: bolting sync on late with polling produces pseudo-sync and expensive rewrites.

Sources: [Linear sync engine](https://linear.app/now/scaling-the-linear-sync-engine), [localfirst.fm transcript](https://www.localfirst.fm/15/transcript), [Linear docs](https://linear.app/docs/get-the-app)

## Property-based testing

Pressure pattern: the bugs only appear after long or surprising sequences of operations, and example tests keep missing the invariant violation.

Who adopted it and why: the external-patterns report recommends `fast-check` model-based testing when the real question is "for every sequence, does this invariant still hold?"

Threshold in evidence terms: adopt property-based testing when you can name an invariant but cannot trust yourself to enumerate the breaking sequences. This is often the cheapest next step before a heavier framework.

Anti-pattern warning: it is overkill for pure utilities and wasteful if the team cannot state the invariant clearly.

Sources: [fast-check](https://fast-check.dev/), [fast-check model-based testing](https://fast-check.dev/docs/advanced/model-based-testing/), [Hillel Wayne](https://www.hillelwayne.com/post/using-formal-methods/)

## CRDT / OT systems

Pressure pattern: multiple replicas must merge concurrent edits without a central authoritative writer, and users care about preserved edit intent.

Who adopted it and why: Yjs, Automerge, and the Google Docs OT lineage exist for dense overlapping concurrent edits. Figma is the counterexample that proves the rule: if the server is authoritative and the merge unit can be narrowed, full CRDT cost may be unnecessary.

Threshold in evidence terms: adopt CRDT or OT when you genuinely have concurrent editable shared objects across replicas. Do not use state-count or "future-proofing" as the threshold.

Anti-pattern warning: buying CRDT or OT because collaboration sounds modern is cargo cult. Append-only message feeds almost never need it.

Sources: [Yjs](https://docs.yjs.dev/), [Automerge](https://automerge.org/docs/hello/), [Google OT background](https://research.google/pubs/pub41926), [Ellis and Gibbs](https://dl.acm.org/doi/10.1145/72981.72982), [Figma](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/)
