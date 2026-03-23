# Task Planning — Technical Report

## Overview

Planning in DeepAgents is **not a separate execution mode** but a **tool-driven task management system** called `write_todos`, provided by LangChain's `TodoListMiddleware`. It enables agents to:

- Break complex tasks into discrete, trackable steps
- Signal progress to users in real time (pending → in_progress → completed)
- Maintain a persistent scratchpad visible across model turns
- Revise plans as new information is discovered

The system has two distinct rendering layers: a **CLI layer** (Rich-based terminal UI) and an **ACP layer** (Agent Client Protocol, for programmatic clients). Both are driven by the same underlying tool.

---

## Key Files and Responsibilities

| File | Lines | Role |
|------|-------|------|
| `libs/deepagents/deepagents/graph.py` | 198, 229, 258 | Registers `TodoListMiddleware` in every agent's middleware stack |
| `libs/cli/deepagents_cli/widgets/messages.py` | 784–887 | Formats `write_todos` output as a styled checklist in the terminal |
| `libs/cli/deepagents_cli/tool_display.py` | 236–239 | Renders the tool-call header line for `write_todos` |
| `libs/deepagents/deepagents/server.py` | 109, 184–267, 333 | ACP server: tracks per-session plan state, sends `AgentPlanUpdate` |
| `libs/deepagents/deepagents/subagents.py` | 336 | Excludes `todos` from the state passed to subagents |
| `libs/cli/deepagents_cli/system_prompt.md` | 224–238 | System-prompt guidance instructing the model on how to use `write_todos` |
| `libs/deepagents/deepagents/textual_adapter.py` | 717–724 | ACP Textual adapter — receives plan updates (rendering not yet implemented) |
| `tests/test_todo_middleware.py` | 17–102 | Unit tests: validates single-call constraint and tool behavior |
| `tests/test_agent.py` | 412–466 | Integration test: verifies plan clearing at session end |
| `tests/test_end_to_end.py` | 127–138 | End-to-end test: basic `write_todos` execution |

---

## Data Structures

### Todo Item

Each todo is a plain Python dictionary:

```python
{
    "content":    str,   # Task description — max 70 chars for display
    "status":     str,   # "pending" | "in_progress" | "completed"
    "activeForm": str,   # (optional) present-tense form, e.g. "Fixing bug"
}
```

### LangGraph State

The `TodoListMiddleware` stores the current todo list under the key `todos` in the LangGraph agent state. This field is:

- Checkpointed between turns
- Visible in the agent's system prompt on every model call
- **Excluded** from state propagated to subagents (`subagents.py:336`)

### ACP Plan State (`server.py:109`)

```python
class AgentServerACP:
    _session_plans: dict[str, list[dict[str, Any]]] = {}
```

Per-session dictionary mapping `session_id` → current todo list. Updated on every `write_todos` call and cleared when the session ends or all tasks complete.

### ACP Wire Format

```python
PlanEntry(
    content:  str,               # Task description
    status:   str,               # "pending" | "in_progress" | "completed"
    priority: str = "medium",    # Hardcoded in server.py:245
)

AgentPlanUpdate(
    session_update: "plan",
    entries: list[PlanEntry],    # Empty list = clear plan
)
```

---

## Middleware Registration

`TodoListMiddleware` is added at position 0 (first) in every agent's middleware stack in `graph.py`:

```python
middleware = [
    TodoListMiddleware(),               # line 258 — planning
    MemoryMiddleware(sources=[...]),
    SkillsMiddleware(sources=[...]),
    FilesystemMiddleware(backend=backend),
    SubAgentMiddleware(subagents=subagents),
    SummarizationMiddleware(trigger=...),
    AnthropicPromptCachingMiddleware(),
    PatchToolCallsMiddleware(),
    *user_middleware,
    HumanInTheLoopMiddleware(interrupt_on=...),
]
```

The same registration is applied to:
- The main agent (line 258)
- Each subagent definition (lines 198, 229)

---

## Execution Flow

```
User provides a multi-step task
        │
        ▼
Agent decides to plan (LLM output)
        │
        ▼
AIMessage with tool_call:
  write_todos(todos=[
    {"content": "Step 1", "status": "pending"},
    {"content": "Step 2", "status": "pending"},
  ])
        │
        ▼
TodoListMiddleware validates and stores todos in state
        │
        ├─── CLI path ──────────────────────────────────────────────────────┐
        │    messages.py renders checklist:                                  │
        │      "0 active | 2 pending | 0 done                               │
        │       ○ todo   Step 1                                              │
        │       ○ todo   Step 2"                                             │
        │                                                                    │
        └─── ACP path ──────────────────────────────────────────────────────┘
             server.py calls _handle_todo_update()
             → sends AgentPlanUpdate(entries=[PlanEntry(...), ...])
        │
        ▼
Agent starts executing Step 1, calls write_todos again to update status:
  write_todos(todos=[
    {"content": "Step 1", "status": "completed"},
    {"content": "Step 2", "status": "in_progress"},
  ])
        │
        ▼
Display updates; agent continues
        │
        ▼
All todos reach "completed"
        │
        ├─── ACP: _clear_plan() → AgentPlanUpdate(entries=[])
        └─── CLI: checklist shows "N done"
```

---

## Constraint: Single Call Per Turn

The `TodoListMiddleware` enforces that `write_todos` may only be called **once per model invocation** (i.e., one tool call per AIMessage). Parallel or multiple calls in the same response are rejected with:

```
Error: The `write_todos` tool should never be called multiple times in parallel.
Please call it only once per model invocation to update the todo list.
```

The tool message status is set to `"error"` and the todos are **not written**.

Tested in `tests/test_todo_middleware.py:17–102`.

---

## CLI Display Implementation

### Entry Point (`messages.py:784`)

```python
def _format_todos_output(self, output: str, *, is_preview: bool = False) -> FormattedOutput:
```

### Parsing (`messages.py:816–832`)

Two strategies are attempted in order:

1. Regex: `r"\[(\{.*\})\]"` — matches a JSON-like list literal
2. `ast.literal_eval()` — direct Python evaluation

Falls back to plain text if both fail.

### Stats Bar (`messages.py:834–855`)

```
3 active | 2 pending | 1 done
```

Counts are derived from `status` values and rendered with color (yellow = active, dim = pending/done).

### Item Format (`messages.py:857–887`)

| Status | Symbol | Style |
|--------|--------|-------|
| `completed` | `✓ done  ` | green, dimmed text |
| `in_progress` | `● active` | yellow |
| `pending` | `○ todo  ` | dim |

Maximum content length: **70 characters** (`messages.py:115`).

In preview mode: at most 4 items shown with a truncation notice.

### Tool Header (`tool_display.py:236–239`)

```python
elif tool_name == "write_todos":
    if "todos" in tool_args and isinstance(tool_args["todos"], list):
        count = len(tool_args["todos"])
        return f"(*) write_todos({count} items)"
```

---

## ACP Plan Management (`server.py`)

### `_handle_todo_update` (`server.py:216–267`)

Called whenever a `write_todos` tool result is observed in the event stream.

```python
async def _handle_todo_update(
    self,
    session_id: str,
    todos: list[dict[str, Any]],
    *,
    log_plan: bool = True,
) -> None:
```

Steps:
1. Validates `status` values — only `"pending"`, `"in_progress"`, `"completed"` allowed
2. Converts each dict to a `PlanEntry(content=..., status=..., priority="medium")`
3. Sends `AgentPlanUpdate(session_update="plan", entries=[...])`
4. Optionally logs the plan as a visible text message for the user

### `_clear_plan` (`server.py:198–214`)

```python
async def _clear_plan(self, session_id: str) -> None:
```

Sends `AgentPlanUpdate(entries=[])` to signal to the client that the plan should be removed from display.

Triggered when:
- All todos are `"completed"` (`_all_tasks_completed()`, `server.py:184–196`)
- The agent session ends

### `_all_tasks_completed` (`server.py:184–196`)

```python
def _all_tasks_completed(self, plan: list[dict[str, Any]]) -> bool:
    if not plan:
        return True
    return all(todo.get("status") == "completed" for todo in plan)
```

---

## System-Prompt Guidance (`system_prompt.md:224–238`)

The model is instructed to:

1. Use todos for any task with **2 or more steps**
2. Mark a task `in_progress` **before** starting it
3. Mark a task `completed` **immediately** after finishing it — no batching
4. Add sub-tasks discovered mid-execution right away
5. Skip todos for simple single-step tasks
6. On first plan creation: **present the plan to the user and wait for approval** before starting work
7. Update status promptly after each item is finished

---

## Planning vs. Subagent Tasks

The codebase uses "task" in two distinct, independent senses:

| Concept | Mechanism | File |
|---------|-----------|------|
| **Todo task** (planning) | `write_todos` tool via `TodoListMiddleware` | `graph.py`, `messages.py` |
| **Subagent task** (delegation) | `task()` tool via `SubAgentMiddleware` | `subagents.py:374–471` |

Todo tasks track progress within a single agent. Subagent tasks spawn isolated child agent graphs and return a result. The two systems do not interact: `todos` state is explicitly excluded from the state passed to subagents (`subagents.py:336`).

---

## Not Yet Implemented

The Textual UI adapter (`textual_adapter.py:717–724`) receives `AgentPlanUpdate` messages but does **not yet render** them into a widget. The code contains a `pass` statement and a comment marking it as future work.

---

## Testing

| Test | File | What it verifies |
|------|------|-----------------|
| Single-call constraint | `test_todo_middleware.py:17–102` | Middleware rejects parallel `write_todos` calls |
| ACP plan clear | `test_agent.py:412–466` | `AgentPlanUpdate(entries=[])` is sent at session end |
| End-to-end execution | `test_end_to_end.py:127–138` | `write_todos` executes and returns correct output |
