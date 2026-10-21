# Shape 7: Durable conversation / agent thread

## Definition

A durable agent thread is a conversational or task execution that carries more than chat text. It accumulates tool calls, checkpoints, human interventions, forks, and resumable progress across turns. The key design move is to treat the thread or session as the unit of continuity rather than the latest message or prompt string.

## Where it occurs

- Anthropic Claude Agent SDK sessions
- OpenAI conversation or response chains
- LangGraph state graphs with checkpointers
- Modern tool-using agent runtimes

## Key invariant

The unit of continuity is the thread or execution, not just the latest message.

## Incident: a held group message was declared delivered before it was saved

On 2026-10-13 Tejas rejected a confident explanation of his Cortex group chat failure: "Uh, get your thing reviewed by someone else here. Get a fresh pair of eyes, maybe a fable agent, to make sure we don't ever see any fucking cause issues here. for whatever reason, the tagging reason, whatever encryption you're talking about, because I don't understand what, I don't think you're explaining it to me correct or like you don't understand anything about what's going on. So I don't trust your judgment or whatever your fix here." The initial diagnosis correctly separated tag recognition from damaged model continuation data, but the first review of the release missed a separate custody gap. The group gateway moved untagged messages into a tagged turn and marked them handed off before the agent runtime durably saved that turn. On restart it deleted those rows. A crash between handoff and save could erase context. A fresh reviewer found the concrete path; the source repair changed restart recovery to reoffer uncertain messages, accepting that a crash after save may repeat context.

For any chat or agent thread that moves context between stores, identify the exact durable acceptance point. A callback that starts a run, or a lease marked handed off, is not proof of acceptance. Review the producer, the destination's persistence boundary, restart settlement, and the user-visible reply boundary together. When the runtime offers no acceptance receipt, state the at-least-once or at-most-once tradeoff explicitly and choose against silent loss of human context. An archive is reference custody, not proof that the next model turn has loaded it. This rule applies independently of why a particular provider call failed. The general form, with the other incidents that hit it, is [guideline 16](../guidelines.md#16-a-success-signal-proves-only-what-it-measures).

## Mental model, in teams' own words

- Anthropic: "How sessions persist agent conversation history, and when to use continue, resume, and fork." Source: https://code.claude.com/docs/en/agent-sdk/sessions
- OpenAI uses conversation objects or `previous_response_id`, and conversations store messages, tool calls, and tool outputs as items. Source: https://platform.openai.com/docs/guides/conversation-state and https://platform.openai.com/docs/api-reference/conversations?api-mode=responses
- LangGraph: "If you are using LangGraph with a checkpointer, you already have durable execution enabled." Source: https://docs.langchain.com/oss/python/langgraph/durable-execution
- LangGraph interrupts "pause graph execution at specific points and wait for external input." Source: https://docs.langchain.com/oss/python/langgraph/human-in-the-loop

## When to use this shape

- An agent can call tools, wait, resume, or fork.
- Human approval or intervention must pause and later continue execution.
- The runtime needs to remember more than plain text history.
- Tool outcomes and checkpoints need durable auditability.

## When NOT to use this shape

- The feature is a plain append-only conversation with no resumable execution.
- The hard part is ordering one room's messages, not preserving agent lineage.
- The right abstraction is a durable workflow with minimal conversational context.

## Compatible libraries / patterns

| Library or pattern | Adopt when... |
| --- | --- |
| Session or conversation object | The main need is durable conversational continuity and tool trace. |
| LangGraph | The agent needs explicit graph state, checkpoints, and interrupts. |
| Temporal / DBOS | The agent is embedded inside a heavier durable workflow. |
| Append-only thread plus checkpoint rows | A lightweight runtime needs explicit resume points without full framework adoption. |
| Plain chat transcript only | Overkill warning in reverse: this breaks once tools, forks, or waits are first-class. |

## TypeScript/Node applicable example

```ts
type ThreadItem =
  | { kind: 'user'; text: string }
  | { kind: 'tool_call'; tool: string; status: 'running' | 'done' }
  | { kind: 'checkpoint'; node: string; state: Record<string, unknown> }
  | { kind: 'interrupt'; reason: string };

type AgentThread = { id: string; cursor: number; items: Array<ThreadItem & { cursor: number }> };

export function appendItem(thread: AgentThread, item: ThreadItem): AgentThread {
  const cursor = thread.cursor + 1;
  return {
    ...thread,
    cursor,
    items: [...thread.items, { ...item, cursor }],
  };
}
```

## References

- https://code.claude.com/docs/en/agent-sdk/sessions
- https://docs.anthropic.com/en/docs/claude-code/sdk
- https://platform.openai.com/docs/guides/conversation-state
- https://platform.openai.com/docs/api-reference/conversations?api-mode=responses
- https://docs.langchain.com/oss/python/langgraph/durable-execution
- https://docs.langchain.com/oss/python/langgraph/human-in-the-loop
