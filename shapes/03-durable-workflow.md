# Shape 3: Durable workflow / lifecycle execution

## Definition

A durable workflow is a multi-step process whose progress must survive crashes, retries, timers, human input, or long pauses. The hard problem is not naming a state; it is preserving which steps already happened, which side effects are safe to repeat, and where execution resumes after failure. Mature teams stop reconstructing that from ad hoc rows, jobs, and crons and model the execution itself as durable state.

## Where it occurs

- Payment and refund flows
- Approval processes
- Provisioning and onboarding
- Long-running agent tasks
- Retry-heavy integrations

## Key invariant

Progress is durable and resumable, and external side effects are replay-safe or compensated.

## Incident: usage reset while the Inbox stayed queued

On November 6, 2026, Tejas saw available Codex usage after an external reset while his Inbox remained queued: *“the moment you see the usage is left, you need to update your state of the system. And that should start flowing the request in.”* The first theory was that Concierge did not poll CodexBar. It did poll both saved accounts every three minutes and read the reset account at 0% use. The actual defect was a retained refusal attached to that **inactive** account: the reader reconciled old refusals only for the active login, so the account switch still treated the saved account as unavailable. Claude had no proven usable login, so a provider handoff could not recover the queue either.

When a workflow waits on external capacity, treat the fresh observation and every account-specific refusal that gates the next transition as one reconciliation boundary. Match the account and source time, retire only a refusal the new observation truly supersedes, then reevaluate the existing queued work and guarded switch. Do not clear unrelated refusal state or replay an accepted request. A periodic timer proves only that the read ran; it does not prove the waiting workflow was released. The [Concierge failure record](https://github.com/tejasdc/slack-concierge/blob/main/docs/incidents/2026-11-06-inactive-codex-reset-hidden-from-switch.md) contains the acceptance sequence; the Concierge reader and guarded switch enforce the transition.

## Mental model, in teams' own words

- Temporal: "Workflow Executions are to distributed systems what transactions are to databases." Source: https://web.temporal.io/blog/workflow-introduction
- Temporal: "Replace your brittle state machines." Source: https://temporal.io/
- Temporal: "The full running state of a Workflow is durable and fault tolerant by default." Source: https://temporal.io/
- AWS names the "Lambda orchestrator anti-pattern" when orchestration hides inside handlers. Source: https://aws.amazon.com/blogs/compute/streamlining-aws-serverless-workflows-from-aws-lambda-orchestration-to-aws-step-functions/

## When to use this shape

- A process must wait on time, humans, or external systems.
- Retry safety matters because side effects are expensive or irreversible.
- The same execution may resume hours or days later.
- The design keeps asking "which step already completed?" or "what do we do after restart?"

## When NOT to use this shape

- One coordinator can handle the whole entity in one short mutation.
- The core problem is ordering within a single stream.
- The state is awareness or presence that can be dropped without consequence.

## Compatible libraries / patterns

| Library or pattern | Adopt when... |
| --- | --- |
| Idempotent step log + saga compensation | The workflow is modest and the team wants a lightweight discipline first. |
| Temporal / Cadence | Retries, timers, signals, and recovery are central to the feature. |
| Step Functions / Conductor | The team prefers explicit orchestration with visible execution history. |
| DBOS / durable execution | Postgres-backed durable replay is a good fit for the stack. |
| Plain `status` field | Overkill warning: never use this alone once you need resume points or replay-safe side effects. |

## TypeScript/Node applicable example

```ts
type Step = 'charge' | 'send_receipt' | 'done';

type Workflow = {
  id: string;
  step: Step;
  history: string[];
  idempotencyKey: string;
};

export async function resumeWorkflow(
  workflow: Workflow,
  effects: { charge: (k: string) => Promise<void>; sendReceipt: () => Promise<void> }
): Promise<Workflow> {
  switch (workflow.step) {
    case 'charge':
      await effects.charge(workflow.idempotencyKey);
      return { ...workflow, step: 'send_receipt', history: [...workflow.history, 'charged'] };
    case 'send_receipt':
      await effects.sendReceipt();
      return { ...workflow, step: 'done', history: [...workflow.history, 'receipt_sent'] };
    case 'done':
      return workflow;
  }
}
```

## References

- https://web.temporal.io/blog/workflow-introduction
- https://cadenceworkflow.io/docs/concepts/workflows
- https://aws.amazon.com/documentation-overview/step-functions/
- https://aws.amazon.com/blogs/compute/streamlining-aws-serverless-workflows-from-aws-lambda-orchestration-to-aws-step-functions/
- https://www.dbos.dev/blog/what-is-lightweight-durable-execution
- https://stripe.com/blog/idempotency
