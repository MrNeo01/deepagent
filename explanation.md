# DeepAgents Framework — Technical Explanation

A deep-dive into how the `deepagents` library works: its execution model, context/memory management, tool system, planning/reflection, and overall orchestration design.

---

## Table of Contents

1. [How the Agent Works](#1-how-the-agent-works)
2. [Context and Memory Management](#2-context-and-memory-management)
3. [Tool Management](#3-tool-management)
4. [Planning and Reflection](#4-planning-and-reflection)
5. [Framework Orchestration and Design](#5-framework-orchestration-and-design)

---

## 1. How the Agent Works

### Entry Point

Everything starts with a single factory function:

```python
# libs/deepagents/deepagents/graph.py:82
agent = create_deep_agent(
    model="claude-sonnet-4-6",
    tools=[...],
    system_prompt="...",
    backend=FilesystemBackend(root_dir="/workspace"),
)
```

This function builds a **compiled LangGraph state graph** — a directed graph whose nodes are LLM calls and tool executions, and whose edges represent the agent's decision flow.

### The Core Execution Loop

The agent runs in a ReAct-style loop managed by LangGraph:

```
User Message
    ↓
[LLM Call] → decides to call tools
    ↓
[Tool Execution] → results appended to message history
    ↓
[LLM Call] → may call more tools, or produce final answer
    ↓
... repeat until no more tool calls ...
    ↓
Final Response
```

The recursion limit is set to **1,000 iterations** (`graph.py:306`), giving agents ample room for complex multi-step tasks.

### System Prompt

Every agent receives a base system prompt (`graph.py:36–68`) that instructs it to:

- Act directly without announcing intent ("Don't say 'I'll now do X' — just do it")
- Read and understand existing code/files before modifying them
- Implement, then verify, then iterate
- Stop and analyze instead of blindly retrying on failure
- Provide brief progress updates for longer tasks

The user's custom `system_prompt` is prepended, so user instructions always take precedence.

### Model Resolution

`libs/deepagents/deepagents/_models.py` handles flexible model specifications:

- String `"claude-sonnet-4-6"` → resolves to `ChatAnthropic`
- String `"openai:gpt-4o"` → resolves to `ChatOpenAI`
- Pre-instantiated `BaseChatModel` → used as-is
- Default: Claude Sonnet 4.6

---

## 2. Context and Memory Management

Deep agents use two orthogonal mechanisms: **long-term memory** (persistent across sessions) and **in-context summarization** (managing token limits within a session).

### 2a. Long-Term Memory — `MemoryMiddleware`

**File:** `libs/deepagents/deepagents/middleware/memory.py`

Memory is loaded from **AGENTS.md files** on disk — markdown files that store facts about the user, project conventions, preferences, and recurring context.

**Loading flow (`before_agent()`, line 238):**

1. At the start of each agent run, `MemoryMiddleware` reads all configured source files (e.g., `~/.deepagents/AGENTS.md`, `./AGENTS.md`).
2. Contents are cached in `MemoryState.memory_contents` — a private field in the LangGraph state, so it is never exposed to the user.
3. If a file is missing, the agent continues silently. Real I/O errors are raised.

**Injection into every LLM call (`wrap_model_call()`, line 322):**

Before every LLM call, the loaded memory is injected into the system message, wrapped in XML tags:

```
<agent_memory>
  [contents of AGENTS.md files]
</agent_memory>
```

This means the model always sees the memory, even mid-conversation, without it appearing in the human/assistant message history.

**What gets stored in memory:**
- User preferences and role descriptions
- Project-specific conventions and tool usage notes
- Recurring context that would otherwise be re-explained each session

**What must NOT be stored:**
- API keys, access tokens, passwords, or credentials
- Transient/session-specific state
- One-off requests

**Memory sources** support layering — base → user → project → team — with later sources overriding earlier ones.

### 2b. In-Session Context Compaction — `SummarizationMiddleware`

**File:** `libs/deepagents/deepagents/middleware/summarization.py`

As a long conversation grows, token usage increases. `SummarizationMiddleware` automatically compacts old messages:

1. **Trigger:** When token/message count exceeds a configurable threshold.
2. **Compaction:** An LLM call summarizes the oldest messages into a condensed narrative.
3. **History offload:** Full verbatim history is saved to the backend at `/conversation_history/{thread_id}.md`, creating a permanent audit log.
4. **Pre-compaction optimization:** Large `write_file`, `edit_file`, and `execute` outputs are truncated first (before the LLM summary call), reducing the cost of the summary itself.

A `compact_conversation` tool is also exposed to the agent, with system prompt guidance on when to use it proactively (e.g., when switching to a new sub-task).

### 2c. Large Tool Result Eviction

**File:** `libs/deepagents/deepagents/middleware/filesystem.py:277–347`

When a tool returns a very large result (default threshold: 20,000 tokens), it is **evicted from the message history**:

- The full result is saved to `/large_tool_results/{tool_call_id}` in the backend.
- The message history entry is replaced with a **preview** (head + tail + truncation notice).
- The agent can re-read the full result via the `read_file` tool if needed.

This prevents tool outputs (e.g., large grep results or file reads) from flooding the context window.

---

## 3. Tool Management

### 3a. Tool Architecture

Tools are **not registered globally** — they are created fresh by each middleware component and injected into the agent at construction time. This keeps tools composable and backend-aware.

### 3b. Filesystem Tools — `FilesystemMiddleware`

**File:** `libs/deepagents/deepagents/middleware/filesystem.py`

The core tool set every agent gets:

| Tool | Description |
|------|-------------|
| `ls` | List directory contents |
| `read_file` | Read file with `offset`/`limit` pagination |
| `write_file` | Create or overwrite a file |
| `edit_file` | String-replacement edits (exact match required) |
| `glob` | Pattern-match filenames |
| `grep` | Search file contents via regex |
| `execute` | Run shell commands (only if backend supports it) |

Each tool has both **sync and async implementations**, created via `StructuredTool.from_function()`.

Results are typed dataclasses (`ReadResult`, `WriteResult`, `EditResult`, `LsResult`, `GrepResult`, `GlobResult`) defined in `backends/protocol.py`.

### 3c. The Backend Protocol

**File:** `libs/deepagents/deepagents/backends/protocol.py`

All file I/O goes through a **backend abstraction layer** (`BackendProtocol`), not directly to the filesystem. This makes the tool system portable:

```
FilesystemMiddleware (tools)
        ↓
   BackendProtocol
   ┌──────────────────────────────────┐
   │ StateBackend    – ephemeral, in-memory (default)      │
   │ FilesystemBackend – real disk, root_dir-isolated      │
   │ StoreBackend    – LangGraph persistent store          │
   │ CompositeBackend – routes paths to different backends │
   │ LocalShellBackend – adds execute() support            │
   └──────────────────────────────────┘
```

The `CompositeBackend` is particularly powerful: it routes reads/writes to different backends based on path prefixes, enabling fine-grained isolation (e.g., `/secrets/` → read-only backend, `/workspace/` → writable backend).

### 3d. The Task Tool — Subagent Delegation

**File:** `libs/deepagents/deepagents/middleware/subagents.py:374–471`

The `task` tool is how the main agent delegates to subagents:

```python
task(
    description="Search for all Python files containing 'TODO' and summarize them",
    subagent_type="general-purpose"
)
```

- The subagent runs in an **isolated LangGraph graph** with its own state.
- It inherits the parent's tools and model unless overridden.
- The result is returned as a `ToolMessage` to the parent agent.
- The parent never sees the subagent's intermediate messages — only the final result.

A default **general-purpose subagent** is always included, suitable for research, file search, and multi-step exploration tasks.

### 3e. Custom Tools

Users can pass arbitrary tools to `create_deep_agent(tools=[...])`. These are injected alongside the built-in filesystem tools.

### 3f. Error Recovery — `PatchToolCallsMiddleware`

**File:** `libs/deepagents/deepagents/middleware/patch_tool_calls.py`

Handles a subtle failure mode: if the agent emits an AI message with tool calls but the conversation is interrupted before the tool responses arrive, the next run would have "dangling" tool calls with no matching `ToolMessage`.

This middleware scans the message history before each agent run and **synthetically completes** any dangling tool calls with a cancellation notice, keeping the message history valid.

---

## 4. Planning and Reflection

### 4a. Todo List / Task Planning — `TodoListMiddleware`

Every deep agent automatically gets a `write_todos` tool from LangChain's `TodoListMiddleware`. This enables the agent to:

1. Break a complex task into discrete sub-steps.
2. Track which steps are pending, in-progress, or complete.
3. Revise the plan as new information is discovered.

The todo list is stored in the agent's state and visible in the system prompt, giving the model a persistent scratchpad for multi-step reasoning.

### 4b. Reflection via Iterative Verification

Reflection is not a separate tool — it is **baked into the base system prompt** (`graph.py:52–60`):

```
1. Understand — Read relevant files, understand patterns, check conventions
2. Act — Implement the solution
3. Verify — Check work against requirements; run tests if available; iterate
```

The agent is explicitly instructed to:
- Keep working until the task is **fully complete** (not just "first draft complete").
- **Stop and analyze** when the same failure repeats, rather than blindly retrying.
- Ask clarifying questions if the task is ambiguous before starting.

This forces a natural reflection loop: implement → verify → identify gap → fix → verify again.

### 4c. Skills System — `SkillsMiddleware`

**File:** `libs/deepagents/deepagents/middleware/skills.py`

Skills are **pre-written, reusable agent procedures** stored in the backend:

```
/skills/web-research/
├── SKILL.md       # YAML frontmatter (name, description) + markdown instructions
└── helper.py      # Optional supporting files
```

When a skill is relevant, it is injected into the agent's system prompt at runtime. Skills can be layered (base → user → project → team), enabling organizations to build reusable knowledge on top of the base agent.

---

## 5. Framework Orchestration and Design

### 5a. The Middleware Pattern

The entire framework is built around a **middleware stack**. Each middleware is a composable unit that can:

- Add new tools to the agent
- Modify the system prompt
- Wrap LLM calls (pre/post processing)
- Run hooks before/after agent execution
- Extend the agent's state schema

Middleware is applied in a defined order, with each layer wrapping the next — similar to HTTP middleware in web frameworks.

```python
# graph.py:256–283 (simplified)
middleware = [
    TodoListMiddleware(),
    MemoryMiddleware(sources=["~/.deepagents/AGENTS.md"]),
    SkillsMiddleware(sources=["~/.deepagents/skills"]),
    FilesystemMiddleware(backend=backend),
    SubAgentMiddleware(subagents=subagents),
    SummarizationMiddleware(trigger=...),
    AnthropicPromptCachingMiddleware(),
    PatchToolCallsMiddleware(),
    *user_middleware,
    HumanInTheLoopMiddleware(interrupt_on=...),
]
```

### 5b. State Management via LangGraph

The agent's state is a typed Python `TypedDict` that accumulates across the execution loop:

- `messages`: The full conversation history (HumanMessage, AIMessage, ToolMessage)
- `files`: Virtual filesystem state (for `StateBackend`)
- `memory_contents`: Loaded AGENTS.md content (private)
- `summarization_events`: History of compaction events
- Custom fields added by user middleware

LangGraph's **reducers** handle how state is merged after each node execution (e.g., list-append for messages, dict-merge for files).

### 5c. Subagent Architecture

```
Main Agent (LangGraph Graph)
    │
    ├── Middleware stack (tools, memory, summarization, ...)
    │
    └── task(subagent_type="X") tool
            │
            └── Subagent Graph (isolated LangGraph Graph)
                    ├── Own middleware stack
                    ├── Own state (no shared state with parent)
                    └── Returns final text result → parent ToolMessage
```

Subagents run **synchronously** (blocking the parent) by default. `AsyncSubAgentMiddleware` enables **remote execution** via LangGraph SDK, allowing subagents to run on separate servers and be polled for completion.

### 5d. Backend Abstraction Layer

The backend layer decouples the agent's logic from its storage mechanism:

```
Agent Logic (middleware, tools)
        ↓
   BackendProtocol (abstract interface)
        ↓
   ┌─────────────────────────────────────────┐
   │  StateBackend     — LangGraph state dict │
   │  FilesystemBackend — OS filesystem       │
   │  StoreBackend      — LangGraph Store     │
   │  CompositeBackend  — path-based routing  │
   │  LocalShellBackend — shell execution     │
   └─────────────────────────────────────────┘
```

This means the same agent code works whether files live in memory (testing), on disk (development), or in a distributed store (production).

### 5e. Human-in-the-Loop

`HumanInTheLoopMiddleware` intercepts specified tool calls before execution and **pauses the graph**, requesting human approval. The agent state is checkpointed, and execution resumes once approval is granted. This is configured per-tool via `interrupt_on`:

```python
create_deep_agent(
    interrupt_on={"write_file": True, "execute": True}
)
```

### 5f. Prompt Caching

`AnthropicPromptCachingMiddleware` adds Anthropic's prompt caching headers to LLM calls. Because the system prompt and tool definitions are large and repeated every turn, caching them reduces both latency and cost significantly on long agentic runs.

---

## Complete Execution Flow

```
create_deep_agent() → compiled LangGraph graph
         │
agent.invoke({"messages": [HumanMessage("...")]})
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  BEFORE AGENT HOOKS                                  │
│  • MemoryMiddleware: load AGENTS.md files            │
│  • PatchToolCallsMiddleware: fix dangling tool calls │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  LLM CALL (with injected context)                   │
│  • System prompt = base + user custom + memory XML  │
│  • Tools = filesystem + task + write_todos + custom │
│  • AnthropicPromptCaching headers applied           │
└─────────────────────────────────────────────────────┘
         │
    ┌────┴────────────────────────────────┐
    │ No tool calls                       │ Tool calls
    ▼                                     ▼
Final Response             ┌─────────────────────────────┐
                           │  TOOL EXECUTION             │
                           │  • FilesystemMiddleware      │
                           │    ls/read/write/edit/...   │
                           │  • SubAgentMiddleware        │
                           │    → spawn subagent graph   │
                           │  • TodoListMiddleware        │
                           │    write_todos              │
                           │  • HumanInTheLoopMiddleware │
                           │    → pause for approval     │
                           └─────────────────────────────┘
                                         │
                                         ▼
                           ┌─────────────────────────────┐
                           │  RESULT PROCESSING          │
                           │  • Large results → evict    │
                           │    to /large_tool_results/  │
                           │  • SummarizationMiddleware  │
                           │    → compact if over limit  │
                           │  • ToolMessage → history    │
                           └─────────────────────────────┘
                                         │
                                         └──────────────► LLM CALL (loop)
```

---

## Key Design Principles

| Principle | How It's Expressed |
|-----------|-------------------|
| **Composability** | Middleware stack — add/remove capabilities without touching core |
| **Backend agnosticism** | `BackendProtocol` abstracts all I/O — same agent, different storage |
| **Context efficiency** | Summarization + large result eviction + prompt caching |
| **Delegation** | Subagent system isolates context-heavy work from the main agent |
| **Persistence** | LangGraph checkpointing + conversation history offload |
| **Safety** | `interrupt_on` for human approval; credential-free memory guidelines |
| **Iterative correctness** | Base prompt enforces verify-then-complete loop |

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `libs/deepagents/deepagents/graph.py` | `create_deep_agent()` — wires everything together |
| `deepagents/middleware/filesystem.py` | File operation tools (ls, read, write, edit, glob, grep) |
| `deepagents/middleware/memory.py` | AGENTS.md loading and injection |
| `deepagents/middleware/subagents.py` | `task` tool and subagent graph construction |
| `deepagents/middleware/async_subagents.py` | Remote subagent execution via LangGraph SDK |
| `deepagents/middleware/summarization.py` | Context compaction and history offload |
| `deepagents/middleware/skills.py` | Skill discovery and injection |
| `deepagents/middleware/patch_tool_calls.py` | Dangling tool call recovery |
| `deepagents/backends/protocol.py` | `BackendProtocol` interface + result types |
| `deepagents/backends/state.py` | Ephemeral in-memory backend |
| `deepagents/backends/filesystem.py` | Disk-based backend with root isolation |
| `deepagents/backends/composite.py` | Path-routing multi-backend |
| `deepagents/_models.py` | Model spec resolution |
| `deepagents/base_prompt.md` | Base system prompt template |
