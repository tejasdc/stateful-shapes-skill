---
name: stateful-shapes
description: Use when designing, changing or reviewing anything that keeps state over time - chat, messaging, queues, workflows, agent threads and handoffs, sync between devices or machines, session or lifecycle status, timers and progress, retries, single writer or actor/coordinator, state machine, XState, Temporal, Durable Objects or a sync engine - and when something stateful went wrong, in the words people use for it - "it got sent twice", "the answer never came back", "final reply will follow", "the restart killed the work", "it says queued/waiting but it's done", "the timer is literally broken", "How difficult is it to make a timer work?", "it jumps back after a save", "my change disappeared", "it shows as my message but I didn't write it", "the page stopped syncing after the update", "the notification went to the wrong account", "the scheduled run skipped", "why are we just telling the other machine your list may be out of date", "sync this across both machines", "design the protocol", "how long will you wait, what's the protocol there". Not for copy, styling or formatting changes.
---

# Stateful Shapes

## Overview

Stateful complexity is not one thing. Mature systems fall into a small number of recognizable shapes, and the usual mistake is the wrong shape, not a forgotten retry. This skill names the shapes, the rules our own incidents taught, and the threshold for a heavier library.

The October 21 retrospective of every session that opened this skill (384 sessions, both machines) found it helps most when an agent turns a shape into one traced question ("which entity owns this transition, and where does its event become durable?") and least when it is read and cited without changing anything. Repeated loading still let boundary bugs ship, and several rules entered only after users hit them. So the skill now asks for a short written state card and a check that would fail, not a reading. Evidence: agent-ecology `docs/retrospectives/2026-10-21-stateful-shapes-retrospective.md`.

## First: is this worth a state review?

Name one of these in a sentence before going further: an observed failure (lost, duplicated, reordered, stale, regressed, stuck), or an invariant the change must keep. If you cannot, stop; this is not a stateful change. Copy, styling, formatting and pure presentation tweaks never need it.

State the operating profile next: one person, a few devices, two machines, unless the project says otherwise. Stripe- or Temporal-grade durability is a reference for the shape, not a bar to meet. In August 2026 this skill's vocabulary was stamped onto cosmetic state in Concierge (a table, nine functions and a worker to remove an emoji) and helped one plan grow from 344 to 2,825 lines over fourteen review rounds (slack-concierge `docs/retrospectives/2026-08-25-overengineering-retrospective.md`). That retrospective asked for this gate and profile; they were added only on October 21.

## The state card (write this; one screen)

Put it in the design, the plan or the review. It is this skill's deliverable.

1. **Entity** that owns ordering, and the **one writer** for it. If two writers touch one record, which fields each owns or how they merge ([guideline 6](./guidelines.md#6-if-many-code-paths-mutate-one-entity-make-one-coordinator-instead-of-more-rules)).
2. **Operation identity** that survives retries, refreshes and restarts: one per attempt or run, never a calendar date ([guidelines 4](./guidelines.md#4-do-not-use-one-status-field-as-workflow-history) and [5](./guidelines.md#5-add-idempotency-at-the-boundary-as-soon-as-retries-can-duplicate-effects)).
3. **Acceptance ladder**: recorded by us → admitted by the transport → saved durably by the receiver → shown to the person. For each success signal the code checks, the rung it proves ([guideline 16](./guidelines.md#16-a-success-signal-proves-only-what-it-measures)).
4. **Provisional and retained state**: what the screen shows before the durable answer, and how the two merge by identity ([guidelines 11](./guidelines.md#11-preserve-user-action-meaning-across-requests-and-refreshes) and [15](./guidelines.md#15-merge-progress-by-identity-before-aggregating)).
5. **Lifetime**: which process holds the work, what restarts it, and whether that is the thing updated most often ([guideline 14](./guidelines.md#14-separate-the-lifetime-of-what-does-the-work-from-the-lifetime-of-what-coordinates-it)).
6. **Old readers**: the older installed page, worker or peer that will meet the new record or message ([guideline 17](./guidelines.md#17-an-older-installed-reader-will-meet-every-new-record)).
7. **Failure trace** in five steps: before the effect, effect committed, response lost, refresh or restart, what the person sees.
8. **One falsifying sequence and the check that runs it** (a test, a fault injection, an installed-build check). A card without a check is context, not prevention.

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

1. Pass the gate and state the profile.
2. Identify the entity and the failure mode; match one or more shapes (they compose). Read only the matching shape files and the guidelines the card points to.
3. Write the state card, then choose the lightest tool that honestly solves the pressure.
4. **When you cite this skill in a review or report**, name the guideline, the design or code choice it changed, and the check that fails without that choice. Shapes listed with no changed choice are background; say so. Of 177 Concierge sessions that opened the skill, 73 never mentioned it afterwards, and most others could not be tied to any change.

## Decision: when to adopt a library

Use [library-thresholds.md](./library-thresholds.md). Do not reach for XState because you have many states; reach for it when the failure mode demands hierarchical concurrency. Do not reach for Temporal for a few labels; reach for it when progress must survive crashes, timers, or retries. The retrospective found no case where this skill pushed a heavy library in; keep it that way.

## Common mistakes

- Picking a shape by state count rather than failure mode.
- Collapsing multiple shapes into one aggregate.
- Persisting ephemeral awareness as durable business state.
- Using a `status` column as a substitute for workflow history, or a hand-written status that outlives the work it describes.
- Retrying ambiguous operations without idempotency keys.
- Treating "the call returned success" as "the receiver has it" ([guideline 16](./guidelines.md#16-a-success-signal-proves-only-what-it-measures)).
- Shipping a new record kind with no check that an older open page can read or pass it through ([guideline 17](./guidelines.md#17-an-older-installed-reader-will-meet-every-new-record)).
- Assuming replicas can "just take the latest" without sequence or state-resolution rules.
- Buying CRDT or OT complexity before there are concurrent editable shared objects.
- Refetching a whole, growing collection whenever one item in it changes ([guideline 12](./guidelines.md#12-make-every-read-cost-what-changed-not-how-much-history-exists)).
- Subtracting provisional progress from a total that may exclude its entity ([guideline 15](./guidelines.md#15-merge-progress-by-identity-before-aggregating)).
- Applying Stripe-grade durability to cosmetic or single-user state.
