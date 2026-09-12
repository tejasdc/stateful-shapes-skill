---
name: stateful-shapes
description: Use when designing or reviewing any stateful subsystem - chat, messaging, workflows, agent threads, collaborative documents, session management, lifecycle state, or any code where a state machine/FSM, status field, concurrency, ordering, races, mutation ownership, single writer, actor/coordinator, sync engine, idempotency, workflow framework, XState, Temporal, or Durable Objects might be relevant.
---

# Stateful Shapes

## Overview

Stateful complexity is not one thing. Mature systems fall into a small number of recognizable shapes. The mistake is usually not "we forgot a retry" - it is "we used the wrong shape." This skill names the shapes and the threshold for when to reach for a heavier library.

## When to use

- Designing a new stateful subsystem.
- Reviewing code with concurrency, ordering, retries, or multiple writers.
- Deciding whether to adopt XState, Temporal, Durable Objects, CRDTs, or a sync engine.

## Core principle

Shape, not count. Teams reach for heavier primitives when the failure mode changes shape, not when state labels cross a threshold.

## The seven shapes

| Shape | Where it occurs | Key invariant | Canonical team | Detail file |
| --- | --- | --- | --- | --- |
| Ordered event stream per entity | Chat rooms, issue histories, payments | One authoritative ordering boundary per stream | Slack / Discord / Telegram | [01](./shapes/01-ordered-event-stream.md) |
| Replicated collaborative document | Shared docs, design files, block graphs | Concurrent edits converge at the right merge unit | Figma / Notion / Yjs | [02](./shapes/02-collaborative-document.md) |
| Durable workflow / lifecycle execution | Payments, approvals, retries, timers | Progress survives crashes, waits, and retries | Temporal / Cadence / Stripe | [03](./shapes/03-durable-workflow.md) |
| Single coordinator / actor per entity | Rooms, carts, sessions, queue shards | Exactly one live mutation coordinator per entity | Durable Objects / Orleans / OTP | [04](./shapes/04-actor-per-entity.md) |
| Ephemeral awareness / presence | Typing, cursors, online state, visibility | Loss or delay must not corrupt durable history | Slack / Phoenix / Yjs | [05](./shapes/05-ephemeral-awareness.md) |
| Sync engine / local-first cache | Instant local reads with later reconciliation | Local state stays usable now and converges later | Linear | [06](./shapes/06-sync-engine.md) |
| Durable conversation / agent thread | Tool-using agents, resumable runs, human approval | Continuity lives in the thread, not the latest message | Anthropic / OpenAI / LangGraph | [07](./shapes/07-agent-thread.md) |

## How to use this skill

1. Identify the entity: room, session, workspace, document, payment, agent run, or similar.
2. Identify the failure mode: races, lost messages, duplicate effects, merge conflicts, stale reads, or crash recovery gaps.
3. Match to the shape or shapes. Multiple shapes often compose.
4. For user-visible workflows, define the requested effect, known facts, and visible transitions before wiring handlers; use [guideline 11](./guidelines.md#11-preserve-user-action-meaning-across-requests-and-refreshes).
5. Read the matching detail files and [guidelines](./guidelines.md), then choose the lightest tool that honestly solves the pressure.

## Decision: when to adopt a library

Use [library-thresholds.md](./library-thresholds.md). Do not reach for XState because you have many states; reach for it when the failure mode demands hierarchical concurrency. Do not reach for Temporal for a few labels; reach for it when progress must survive crashes, timers, or retries.

## Guidelines

See [guidelines.md](./guidelines.md) for the practical rules.

## Common mistakes

- Picking a shape by state count rather than failure mode.
- Collapsing multiple shapes into one aggregate.
- Persisting ephemeral awareness as durable business state.
- Using a `status` column as a substitute for full workflow history.
- Retrying ambiguous operations without idempotency keys.
- Assuming replicas can "just take the latest" without sequence or state-resolution rules.
- Buying CRDT or OT complexity before there are concurrent editable shared objects.
