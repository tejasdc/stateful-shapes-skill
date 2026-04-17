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
