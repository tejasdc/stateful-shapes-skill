# Practical guidelines

These ten rules come directly from the "A concrete set of guidelines the team can adopt" section of the source report. The examples are generic, but the guidance is grounded in the cited teams.

## 1. Pick the entity that owns ordering before writing code

Rule: name the coordination key first.

Slack and Discord stay tractable because they make the ordering boundary explicit: channel first, everything else second. If the system cannot answer "who owns ordering for this entity?" it usually starts compensating with retries, timestamps, and after-the-fact repair.

Source teams: [Slack](https://slack.engineering/real-time-messaging/), [Discord](https://discord.com/blog/how-discord-stores-trillions-of-messages)

```ts
// Bad: two writers decide order independently.
saveMessage(roomId, message, Date.now());

// Good: one room stream assigns sequence.
stream.append(roomId, { kind: 'message.sent', message });
```

## 2. Separate durable history from ephemeral awareness

Rule: typing, presence, and cursors are not message history.

Slack, Phoenix Presence, and Yjs all split durable state from perishable awareness. This keeps loss, duplication, or delay in the awareness layer from corrupting the durable record.

Source teams: [Slack](https://slack.engineering/real-time-messaging/), [Phoenix Presence](https://hexdocs.pm/phoenix/presence.html), [Yjs Awareness](https://docs.yjs.dev/getting-started/adding-awareness)

```ts
// Bad: typing mutates durable history.
events.push({ kind: 'typing.started', userId });

// Good: typing is a TTL-based broadcast surface.
presence.heartbeat(roomId, userId, { typing: true });
```

## 3. Treat per-user read state as its own subsystem

Rule: read cursors are not room metadata.

Discord gives read state a dedicated service because it changes at a different rate and has different ownership than room history. Keeping read state separate avoids locking room writes behind per-user cursor churn.

Source team: [Discord](https://discord.com/blog/why-discord-is-switching-from-go-to-rust)

```sql
-- Bad: one row tries to hold both room state and every user's read position.
UPDATE rooms SET last_read_seq = 42 WHERE id = $1;

-- Good: one monotonic cursor per (room, user).
UPDATE read_state
SET last_read_seq = $new
WHERE room_id = $room AND user_id = $user AND last_read_seq < $new;
```

## 4. Do not use one status field as workflow history

Rule: lifecycle progress needs durable steps, not just a label.

Temporal, Cadence, Conductor, Step Functions, and DBOS all exist because long-running progress is richer than `status = 'running'`. Once retries, timers, or approvals appear, the durable question becomes "what already happened?" not merely "what label are we in?"

Source teams: [Temporal](https://web.temporal.io/blog/workflow-introduction), [Cadence](https://cadenceworkflow.io/docs/concepts/workflows), [Conductor](https://docs.conductor-oss.org/devguide/concepts/workflows.html), [AWS Step Functions](https://aws.amazon.com/documentation-overview/step-functions/), [DBOS](https://www.dbos.dev/blog/dbos-transact-open-source-typescript-framework)

```ts
// Bad
workflow.status = 'processing';

// Good
workflow.history.push({ step: 'charged_card', at: now });
workflow.nextStep = 'send_receipt';
```

## 5. Add idempotency at the boundary as soon as retries can duplicate effects

Rule: ambiguous retries need operation identity.

Stripe treats the API boundary as part of the state machine: the retried operation has a stable identity. Without that, the system cannot distinguish "retry the same effect" from "perform a new effect."

Source team: [Stripe](https://stripe.com/blog/idempotency)

```ts
// Bad: duplicate retry can repeat the business effect.
await chargeCard(input);

// Good: correlate retries by boundary identity.
await idempotency.run(key, () => chargeCard(input));
```

## 6. If many code paths mutate one entity, make one coordinator instead of more rules

Rule: serialize ownership before inventing more guards.

Cloudflare, Orleans, and OTP make correctness come from one owner per entity. That is usually simpler than allowing many writers and trying to reconstruct order after the fact.

Source teams: [Cloudflare Durable Objects](https://developers.cloudflare.com/durable-objects/concepts/what-are-durable-objects/), [Orleans](https://learn.microsoft.com/en-us/dotnet/orleans/overview), [OTP](https://www.erlang.org/docs/26/design_principles/users_guide.html)

```ts
// Bad: any caller can mutate session state directly.
session.state = nextState;

// Good: all mutations pass through one keyed mailbox.
await mailbox.dispatch(sessionId, () => sessionActor.apply(command));
```

## 7. If the product promise is instant local feel, invest in sync early

Rule: sync is product infrastructure, not a later patch.

Linear's lesson is that local-first responsiveness becomes much harder if the system starts with polling and incidental cache invalidation. If the UX promise depends on immediate reads and optimistic writes, the sync layer deserves first-class design.

Source teams: [Linear](https://linear.app/now/scaling-the-linear-sync-engine), [localfirst.fm transcript](https://www.localfirst.fm/15/transcript)

```ts
// Bad: UI waits for round-trip before showing the change.
await api.updateIssue(input);
setState(await api.fetchIssue(id));

// Good: local write now, reconcile later.
store.applyOptimistic(mutation);
sync.enqueue(mutation);
```

## 8. Do not buy CRDT or OT complexity until the product truly needs concurrent shared editing

Rule: start from the authority model, not the trend.

Figma is the canonical warning against cargo-culting CRDTs. They studied the space, then deliberately used the smallest merge boundary their centralized model allowed.

Source team: [Figma](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/)

```ts
// Bad: full document CRDT for a server-owned append-only feed.
const doc = automerge.change(roomDoc, ...);

// Good: property-level or stream-level coordination when one authority is enough.
roomStream.append(roomId, event);
```

## 9. Model the agent as a durable thread or workflow

Rule: agent progress is not "just another chat message."

Anthropic, OpenAI, and LangGraph all preserve more than plain text: they preserve session continuity, tool-call trace, checkpoints, and resume points. Once an agent can pause, fork, or call tools, its execution lineage deserves its own durable surface.

Source teams: [Anthropic](https://code.claude.com/docs/en/agent-sdk/sessions), [OpenAI](https://platform.openai.com/docs/guides/conversation-state), [LangGraph](https://docs.langchain.com/oss/python/langgraph/durable-execution)

```ts
// Bad: hide tool progress inside a normal chat message.
messages.push({ role: 'assistant', text: 'calling tool...' });

// Good: persist execution state separately.
thread.items.push({ kind: 'tool_call', tool: 'search', status: 'running' });
```

## 10. Use Hickey's test when the design feels muddy

Rule: ask which concerns are being braided together.

Rich Hickey's framing is useful because it catches the root mistake early: durable history, ephemeral awareness, workflow progress, and read state often look related in the UI while needing different invariants underneath. If one table or reducer is trying to do all of them, the design is probably mixing shapes.

Source thinker: [Rich Hickey, "Simple Made Easy"](https://www.infoq.com/presentations/Simple-Made-Easy/)

```ts
// Bad: one aggregate mixes unrelated concerns.
type RoomState = { messages: Message[]; typing: string[]; workflowStatus: string };

// Good: one join point, separate surfaces.
type Room = { stream: Stream; presence: Presence; workflow: AgentThread };
```
