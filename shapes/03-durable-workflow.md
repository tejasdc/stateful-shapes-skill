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
