# Research Preview Features

These three features require:
1. Access request: https://claude.com/form/claude-managed-agents
2. Additional beta header on all requests:
   ```
   anthropic-beta: managed-agents-2026-04-01-research-preview
   ```

---

## Outcomes

Elevates a session from "conversation" to "autonomous work toward a goal."
You define what done looks like. A separate grader (independent context window) evaluates
the output against your rubric and tells the agent what to fix. Agent iterates until satisfied.

### Setup

```python
# 1. Create session normally
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    title="Build DCF model for Costco",
)

# 2. Send define_outcome — agent starts immediately, no separate user.message needed
client.beta.sessions.events.send(session.id, events=[{
    "type": "user.define_outcome",
    "description": "Build a DCF model for Costco in .xlsx format",
    "rubric": {
        "type": "text",
        "content": RUBRIC_MARKDOWN,
        # or: "type": "file", "file_id": "file_abc"  (upload via Files API for reuse)
    },
    "max_iterations": 5,   # optional, default 3, max 20
}])
```

### Writing effective rubrics

Rubrics must be explicit, gradeable criteria — not vague quality assessments.
The grader scores each criterion independently.

```markdown
# DCF Model Rubric

## Revenue Projections
- Uses historical revenue data from the last 5 fiscal years
- Projects revenue for at least 5 years forward
- Growth rate assumptions are explicitly stated

## Output Quality
- All figures are in a single .xlsx file with clearly labeled sheets
- Key assumptions are on a separate "Assumptions" sheet
- Sensitivity analysis on WACC and terminal growth rate is included
```

**Tip:** Give Claude a known-good example and ask it to analyze what makes it good,
then turn that analysis into rubric criteria. Better than writing from scratch.

### Outcome events to handle

```python
for event in stream:
    match event.type:
        case "span.outcome_evaluation_start":
            print(f"[Grader evaluating iteration {event.iteration}...]")

        case "span.outcome_evaluation_end":
            match event.result:
                case "satisfied":
                    print(f"[Done: {event.explanation}]")
                    # session goes idle — fetch outputs
                case "needs_revision":
                    print(f"[Revising... iteration {event.iteration}]")
                    # agent continues automatically
                case "max_iterations_reached":
                    print("[Max iterations hit — final revision running]")
                case "failed":
                    print("[Rubric fundamentally incompatible with task]")

        case "session.status_idle":
            break
```

### Retrieving output files

The agent writes deliverables to `/mnt/session/outputs/` inside the container.
Fetch them via Files API scoped to the session:

```python
files = client.beta.files.list(scope_id=session.id)
for f in files.data:
    print(f"{f.id}: {f.filename} ({f.size_bytes} bytes)")

# Download
content = client.beta.files.download(files.data[0].id)
content.write_to_file("output.xlsx")
```

### Checking outcome status without streaming

```python
session = client.beta.sessions.retrieve(session.id)
for outcome in session.outcome_evaluations:
    print(f"{outcome.outcome_id}: {outcome.result}")
    # outc_01a...: satisfied
```

### Notes
- Only one active outcome at a time
- To chain outcomes: send a new `user.define_outcome` after the previous one ends
- A `user.interrupt` stops the current outcome
- After outcome completes, session can continue as a normal conversational session

---

## Multiagent

One **orchestrator agent** coordinates specialist **subagents**. Each agent has isolated context
but they all share the same container filesystem.

**Only one level of delegation** — subagents cannot call further subagents.

### Setup

```python
# 1. Create specialist agents first
reviewer = client.beta.agents.create(
    name="Code Reviewer",
    model="claude-sonnet-4-6",
    system="You review code for correctness, security, and style. Be thorough and specific.",
    tools=[{"type": "agent_toolset_20260401", "default_config": {"enabled": False},
            "configs": [{"name": "read", "enabled": True}, {"name": "grep", "enabled": True}]}],
)

test_writer = client.beta.agents.create(
    name="Test Writer",
    model="claude-sonnet-4-6",
    system="You write comprehensive test suites. Focus on edge cases and coverage.",
    tools=[{"type": "agent_toolset_20260401"}],
)

# 2. Create orchestrator with callable_agents list
orchestrator = client.beta.agents.create(
    name="Engineering Lead",
    model="claude-sonnet-4-6",
    system="""You coordinate engineering work.
    Delegate code review to the reviewer agent and test writing to the test writer.
    Summarize their findings and give a final recommendation.""",
    tools=[{"type": "agent_toolset_20260401"}],
    callable_agents=[
        {"type": "agent", "id": reviewer.id, "version": reviewer.version},
        {"type": "agent", "id": test_writer.id, "version": test_writer.version},
    ],
)

# 3. Create session referencing orchestrator only
session = client.beta.sessions.create(
    agent=orchestrator.id,
    environment_id=environment.id,
)
```

### Session threads

The top-level session stream shows a condensed view. Each subagent runs in its own thread.

```python
# List all threads in a session
for thread in client.beta.sessions.threads.list(session.id):
    print(f"[{thread.agent_name}] {thread.status}")

# Stream a specific subagent's thread for detailed trace
with client.beta.sessions.threads.stream(thread.id, session_id=session.id) as stream:
    for event in stream:
        if event.type == "agent.message":
            for block in event.content:
                print(block.text, end="")
        elif event.type == "session.thread_idle":
            break

# List past events for a thread
for event in client.beta.sessions.threads.events.list(thread.id, session_id=session.id):
    print(f"[{event.type}] {event.processed_at}")
```

### Routing tool calls from subagents

When a subagent needs your input (custom tool result or tool confirmation), the event on the
**session stream** includes a `session_thread_id` field. Echo it back in your response.

```python
for event in stream:
    if event.type == "agent.custom_tool_use":
        result = execute_my_tool(event.name, event.input)
        send_params = {
            "type": "user.custom_tool_result",
            "custom_tool_use_id": event.id,
            "content": [{"type": "text", "text": json.dumps(result)}],
        }
        # Critical: echo session_thread_id if present (routes to correct subagent)
        if hasattr(event, 'session_thread_id') and event.session_thread_id:
            send_params["session_thread_id"] = event.session_thread_id

        client.beta.sessions.events.send(session.id, events=[send_params])
```

### Multiagent event types on session stream

| Event | Description |
|---|---|
| `session.thread_created` | Coordinator spawned a new subagent thread |
| `session.thread_idle` | A subagent thread finished its work |
| `agent.thread_message_sent` | Agent sent a message to another thread |
| `agent.thread_message_received` | Agent received a message from another thread |

---

## Memory Stores

Memory stores are persistent document collections that survive across sessions.
The agent reads and writes them automatically using memory tools.

### Create and seed a store

```python
# Create the store
store = client.beta.memory_stores.create(
    name="User Preferences",
    description="Per-user preferences, past decisions, and project conventions.",
)

# Optionally pre-load it with reference material
client.beta.memory_stores.memories.write(
    memory_store_id=store.id,
    path="/standards/formatting.md",
    content="Always use GAAP formatting. Dates are ISO-8601. Currency in USD.",
)
```

Memory files are capped at **100KB (~25K tokens)** each. Structure as many small focused
files, not a few large ones. Organize with paths like a filesystem: `/user/prefs.md`,
`/project/conventions.md`, `/history/past_decisions.md`.

### Attach to a session

```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    resources=[{
        "type": "memory_store",
        "memory_store_id": store.id,
        "access": "read_write",   # or "read_only"
        "prompt": "User preferences and project context. Check before starting any task.",
    }],
)
```

Max **8 memory stores per session**. Use multiple stores to separate:
- Shared read-only reference material (one store for many sessions)
- Per-user or per-project read-write learnings

### Memory tools (automatically available when store is attached)

| Tool | Description |
|---|---|
| `memory_list` | List documents, optionally filtered by path prefix |
| `memory_search` | Full-text search across document contents |
| `memory_read` | Read a document's full content |
| `memory_write` | Create or overwrite a document at a path |
| `memory_edit` | Modify an existing document |
| `memory_delete` | Remove a document |

Memory tool calls appear as `agent.tool_use` events in the stream.

### Managing memories via API

```python
# List all memories in a store
page = client.beta.memory_stores.memories.list(store.id, path_prefix="/")
for memory in page.data:
    print(f"{memory.path} ({memory.size_bytes} bytes)")

# Read a specific memory
mem = client.beta.memory_stores.memories.retrieve(memory_id, memory_store_id=store.id)
print(mem.content)

# Write/upsert by path (creates or replaces)
client.beta.memory_stores.memories.write(
    memory_store_id=store.id,
    path="/user/prefs.md",
    content="Updated preferences...",
)

# Update by ID (rename, edit content, or both)
client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id=store.id,
    content="New content",
    # Optimistic concurrency — only update if content hasn't changed:
    precondition={"type": "content_sha256", "content_sha256": mem.content_sha256},
)

# Create-only guard (fail if path already exists)
client.beta.memory_stores.memories.write(
    memory_store_id=store.id,
    path="/user/prefs.md",
    content="Initial prefs",
    precondition={"type": "not_exists"},
)

# Delete
client.beta.memory_stores.memories.delete(mem.id, memory_store_id=store.id)
```

### Memory versions (audit + rollback)

Every write creates an immutable version (`memver_...`). List versions, retrieve past content,
or redact sensitive content from history (removes content while preserving audit trail).

```python
# List version history for a memory
for v in client.beta.memory_stores.memory_versions.list(store.id, memory_id=mem.id):
    print(f"{v.id}: {v.operation}")   # created / modified / deleted

# Retrieve a specific version's content
version = client.beta.memory_stores.memory_versions.retrieve(version_id, memory_store_id=store.id)
print(version.content)

# Redact for compliance (PII, leaked secrets, GDPR deletion)
client.beta.memory_stores.memory_versions.redact(version_id, memory_store_id=store.id)
# Clears: content, content_sha256, content_size_bytes, path
# Preserves: actor, timestamps, operation — audit trail intact
```
