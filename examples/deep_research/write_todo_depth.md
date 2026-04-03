# `write_todos` — Deep Dive

---

## Q1 — All Files and Lines Where `write_todo` Is Defined, Used, or Related

### Definition (the tool itself)

| File | Lines | What happens |
|------|-------|-------------|
| `.venv/lib/python3.12/site-packages/langchain/agents/middleware/todo.py` | 125–135 | Module-level `@tool` definition of `write_todos` (standalone function, not used at runtime — overridden by the class instance below) |
| `.venv/…/todo.py` | 185–199 | **Authoritative runtime definition**: `TodoListMiddleware.__init__` creates a *new* `@tool`-decorated `write_todos` closure with the custom `tool_description`, stores it in `self.tools = [write_todos]` |
| `.venv/…/todo.py` | 29–47 | `Todo` TypedDict and `PlanningState` — the data structures the tool works with |
| `.venv/…/todo.py` | 50–108 | `WRITE_TODOS_TOOL_DESCRIPTION` — the long instruction string the LLM sees in its tool schema |
| `.venv/…/todo.py` | 110–122 | `WRITE_TODOS_SYSTEM_PROMPT` — injected into every model call by `wrap_model_call` / `awrap_model_call` |

### Middleware class (contains the tool, enforces constraints, injects prompts)

| File | Lines | What happens |
|------|-------|-------------|
| `.venv/…/todo.py` | 138–327 | `TodoListMiddleware` class: `__init__`, `wrap_model_call`, `awrap_model_call`, `after_model`, `aafter_model` |
| `.venv/…/__init__.py` (langchain middleware package) | 21, 69 | Re-exports `TodoListMiddleware` so consumers can do `from langchain.agents.middleware import TodoListMiddleware` |

### Registration (where the middleware — and thus the tool — is wired into agents)

| File | Lines | What happens |
|------|-------|-------------|
| `libs/deepagents/deepagents/graph.py` | 7 | `from langchain.agents.middleware import … TodoListMiddleware` |
| `libs/deepagents/deepagents/graph.py` | 197–198 | Registered in the **general-purpose subagent** middleware stack |
| `libs/deepagents/deepagents/graph.py` | 227–229 | Registered in each **custom subagent** middleware stack |
| `libs/deepagents/deepagents/graph.py` | 256–258 | Registered in the **main agent** middleware stack (position 0, first in chain) |

### Rendering / display (consumes tool output, does not call the tool)

| File | Lines | What happens |
|------|-------|-------------|
| `libs/cli/deepagents_cli/widgets/messages.py` | 784–887 | `_format_todos_output` — parses the tool result string and renders a Rich checklist in the terminal |
| `libs/cli/deepagents_cli/tool_display.py` | 236–239 | Renders the call-header line `(*) write_todos(N items)` |
| `libs/deepagents/deepagents/server.py` | 109, 184–267 | ACP server tracks per-session plan state (`_session_plans`), converts tool results to `AgentPlanUpdate` wire messages |

### State plumbing

| File | Lines | What happens |
|------|-------|-------------|
| `.venv/…/todo.py` | 46 | `todos` field declared as `Annotated[NotRequired[list[Todo]], OmitFromInput]` in `PlanningState` — checkpointed between turns, excluded from model input |
| `libs/deepagents/deepagents/middleware/subagents.py` | 336 | `todos` explicitly excluded from state passed to subagents |

### System-prompt guidance (in the deep_research example)

| File | Lines | What happens |
|------|-------|-------------|
| `examples/deep_research/research_agent/prompts.py` | 7 | RESEARCH_WORKFLOW_INSTRUCTIONS step 1: "Create a todo list with `write_todos`…" |

### Tests

| File | Lines | What it verifies |
|------|-------|-----------------|
| `libs/deepagents/tests/unit_tests/test_todo_middleware.py` | 1–102 | Single-call constraint + error content/status |
| `libs/deepagents/tests/unit_tests/test_end_to_end.py` | 127–138 | Basic `write_todos` execution in a full agent loop |
| `libs/acp/tests/test_agent.py` | 412–466 | ACP plan clearing at session end |

### How to import and call

```python
from langchain.agents.middleware import TodoListMiddleware
from langchain.agents import create_agent

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",
    middleware=[TodoListMiddleware()],  # tool is registered automatically
)
result = agent.invoke({"messages": [HumanMessage("Refactor my codebase")]})
print(result["todos"])  # → list[Todo]
```

The agent (LLM) calls `write_todos` by name; the middleware routes the call to the closure stored in `self.tools`. You never call `write_todos` directly from Python.

---

## Q2 — Does the Todo List Obligate the Agent to Complete All Tasks?

**Short answer: soft obligation via system-prompt instructions, not a hard runtime guard.**

The `WRITE_TODOS_TOOL_DESCRIPTION` (`.venv/…/todo.py:84–99`) tells the model:

> - Mark tasks `in_progress` **before** beginning work
> - Mark tasks `completed` **immediately** after finishing
> - ONLY mark a task `completed` when you have FULLY accomplished it
> - Unless all tasks are completed, you should **always have at least one task `in_progress`**
> - If you encounter errors/blockers, keep the task `in_progress` and create a new task for the blocker

These are **natural-language instructions** baked into the tool description and system prompt. They constrain the LLM's behavior through the prompt, not through runtime code.

There is **no code that**:
- prevents the agent from terminating while tasks remain `pending` or `in_progress`
- inspects the todo list before returning a final response
- forces another agent turn if incomplete tasks exist

The only runtime enforcement is on the **ACP rendering side**: `_all_tasks_completed()` (`server.py:184–196`) sends a clear-plan signal when all statuses are `"completed"`, and `_clear_plan()` is also called at session end regardless. This is purely for UI hygiene — it has no effect on agent execution.

**In summary**: the todo list is a structured scratchpad with instructional discipline, not a completion gate.

---

## Q3 — How Is the One-Call-Per-Turn Limit Enforced?

The enforcement is in `TodoListMiddleware.after_model` (`.venv/…/todo.py:255–305`).

### Mechanism

Every time the LLM produces an `AIMessage`, the LangGraph runtime calls `after_model` **before** executing any of the tool calls. The method:

1. Finds the last `AIMessage` in state (`line 279`)
2. Counts tool calls whose `name == "write_todos"` (`line 284`)
3. If the count is **> 1** (`line 286`), it generates one `ToolMessage` per offending call, all with `status="error"` and the message:

   > `"Error: The write_todos tool should never be called multiple times in parallel. Please call it only once per model invocation to update the todo list."`

4. Returns `{"messages": error_messages}` (`line 303`) — this **short-circuits** tool execution: the `Command` inside `write_todos` is never run, so `state["todos"]` is **not updated**

5. The error `ToolMessage`s are added to the message history, the agent gets another model turn, and is expected to retry with a single call

### Why this design?

`write_todos` uses a LangGraph `Command(update={"todos": todos})` which **replaces the entire list** on each call. Two parallel calls would produce two competing replacements with no defined order. Rejecting both and forcing a single retry eliminates the ambiguity.

### Async version

`aafter_model` (`line 307–327`) is the async hook — it simply delegates to the sync `after_model`. Both sync and async agent runtimes are covered.

### Test

`test_todo_middleware.py:17–102` exercises this precisely: a fake model emits an `AIMessage` with two `write_todos` tool calls; the test asserts that both `ToolMessage`s have `status="error"` and that `result["todos"] == []`.

---

## Q4 — What Is the Return Type of `write_todos` and Is It Saved?

### Return type

```python
@tool(description=WRITE_TODOS_TOOL_DESCRIPTION)
def write_todos(
    todos: list[Todo], tool_call_id: Annotated[str, InjectedToolCallId]
) -> Command[Any]:
    return Command(
        update={
            "todos": todos,
            "messages": [ToolMessage(f"Updated todo list to {todos}", tool_call_id=tool_call_id)],
        }
    )
```

The function returns a **`langgraph.types.Command[Any]`**. This is not a plain value — it is a LangGraph primitive that instructs the graph runtime to apply `update` to the agent state dict.

### What gets saved

| Target | What | Where |
|--------|------|-------|
| `state["todos"]` | The full `list[Todo]` passed by the LLM | LangGraph agent state, checkpointed by the configured `Checkpointer` between turns |
| `state["messages"]` | A `ToolMessage` with the string `"Updated todo list to [...]"` | Same LangGraph state / message history |

### Persistence

- `todos` is declared `Annotated[NotRequired[list[Todo]], OmitFromInput]` in `PlanningState`. `OmitFromInput` means it is **excluded from the messages fed to the model** on the next turn, but it **is** checkpointed (serialized) by LangGraph's state machinery if a `Checkpointer` is provided.
- The `ToolMessage` string representation (`"Updated todo list to [...]"`) **is** included in the message history fed to the model, giving the LLM confirmation the update succeeded.
- In the ACP server (`server.py:109`), a parallel copy is maintained in `AgentServerACP._session_plans: dict[str, list[dict]]` — a simple in-memory dict keyed by `session_id`. This is used exclusively for sending `AgentPlanUpdate` events to connected clients; it is not persisted to disk.

### Nothing is written to disk by default

Unless the application configures a LangGraph `Checkpointer` (e.g., `SqliteSaver`, `PostgresSaver`), the `todos` live only in memory for the duration of the agent session.
