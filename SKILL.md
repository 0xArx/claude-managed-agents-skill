---
name: claude-managed-agents
description: >
  Build applications with the Claude Managed Agents API — Anthropic's managed infrastructure
  for running Claude as an autonomous agent in a cloud container. Trigger this skill whenever
  the user wants to build with managed agents, asks about agent sessions, wants to stream
  agent events, uses client.beta.agents / client.beta.sessions / client.beta.environments,
  asks about the managed-agents-2026-04-01 beta, wants Claude to autonomously run tools in
  a sandbox, or asks about outcomes, multiagent orchestration, or persistent agent memory.
  Also trigger when the user asks how to run Claude for long-running tasks, async work, or
  when they want Claude to execute bash commands, write files, or browse the web autonomously.
---

# Claude Managed Agents

Managed Agents is Anthropic's hosted infrastructure for running Claude as an autonomous agent.
Instead of building your own agent loop, you get a cloud container where Claude can run bash,
read/write files, search the web, and execute code — all managed by Anthropic.
Built-in prompt caching, context compaction, and performance optimizations are included.

**Official API documentation (always up to date):**
- Overview & concepts: https://platform.claude.com/docs/en/managed-agents/overview
- Quickstart: https://platform.claude.com/docs/en/managed-agents/quickstart
- API reference: https://platform.claude.com/docs/en/managed-agents/api-reference
- Events reference: https://platform.claude.com/docs/en/managed-agents/events

If you need the latest details on any endpoint, parameter, or behavior not covered in this
skill's reference files, fetch the relevant documentation page above.

**Beta header required on every request:**
```
anthropic-beta: managed-agents-2026-04-01
```
The SDK sets this automatically. Raw HTTP calls must include it manually.

---

## The 4 Core Concepts

```
Agent       → reusable config: model + system prompt + tools + skills
Environment → reusable container template: packages + networking
Session     → live running instance of Agent inside Environment
Events      → the communication layer: you send events in, agent sends events out
```

Create **Agent** and **Environment** once. Reuse them across many **Sessions**.
Each session gets its own isolated container — no shared filesystem between sessions.

---

## Setup

```bash
pip install anthropic          # Python
npm install @anthropic-ai/sdk  # TypeScript
```

```python
from anthropic import Anthropic
client = Anthropic()  # uses ANTHROPIC_API_KEY env var
```

**`ant` CLI** — Anthropic's official CLI for interactive agent/environment/session management.
Install: `brew install anthropics/tap/ant` (macOS) or download from GitHub releases.
The Python/TS SDK is sufficient for application code.

---

## Step 1 — Create an Agent

```python
agent = client.beta.agents.create(
    name="Coding Assistant",
    model="claude-sonnet-4-6",          # claude-opus-4-6 or claude-haiku-4-5 also valid
    system="You are a helpful coding assistant. Write clean, well-documented code.",
    tools=[{"type": "agent_toolset_20260401"}],  # enables all 8 built-in tools
)
# Save agent.id — you'll reference it in every session
```

**Agent fields:**

| Field | Notes |
|---|---|
| `model` | Required. All Claude 4.5+ models. Fast mode: `{"id": "claude-opus-4-6", "speed": "fast"}` |
| `system` | Agent persona/behavior. Distinct from user messages (which describe the task). |
| `tools` | `agent_toolset_20260401` = all built-in tools. See `references/tools.md` for fine-grained control. |
| `skills` | Anthropic pre-built: `xlsx`, `pdf`, `docx`, `pptx`. Or custom skills by ID. Max 20/session. |
| `callable_agents` | Research preview. Other agent IDs this agent can delegate to. |
| `metadata` | Arbitrary key-value pairs for your own tracking. |

**Attaching skills:**

```python
agent = client.beta.agents.create(
    name="Financial Analyst",
    model="claude-sonnet-4-6",
    system="You are a financial analysis agent.",
    skills=[
        {"type": "anthropic", "skill_id": "xlsx"},               # Anthropic pre-built
        {"type": "custom", "skill_id": "skill_abc123", "version": "latest"},  # your own
    ],
)
```

**Versioning:** Every `update()` call increments `version`. Always pass the current version back:

```python
agent = client.beta.agents.update(agent.id, version=agent.version, system="new prompt")
```

Omitted fields are preserved. Arrays (tools, skills) are fully replaced on update. Archive = read-only.

---

## Step 2 — Create an Environment

```python
environment = client.beta.environments.create(
    name="my-env",
    config={
        "type": "cloud",
        "networking": {"type": "unrestricted"},  # or "limited" — see below
        "packages": {
            "pip": ["pandas", "numpy", "requests"],
            "npm": ["express"],
            "apt": ["ffmpeg"],
        },
    },
)
# Save environment.id
```

**Networking modes:**
- `unrestricted` — full outbound access (fine for dev)
- `limited` — allowlist only (recommended for production)

```python
# Production-safe networking
"networking": {
    "type": "limited",
    "allowed_hosts": ["api.example.com", "storage.googleapis.com"],
    "allow_mcp_servers": True,       # lets MCP server endpoints through
    "allow_package_managers": True   # lets pip/npm registries through at runtime
}
```

**Supported package managers:** `apt`, `cargo`, `gem`, `go`, `npm`, `pip`
Packages declared here are cached across all sessions sharing this environment.

For container specs, pre-installed runtimes, and filesystem layout, see `references/container-reference.md`.

---

## Step 3 — Create a Session

```python
session = client.beta.sessions.create(
    agent=agent.id,                # uses latest version
    # agent={"type": "agent", "id": agent.id, "version": 2},  # pin a specific version
    environment_id=environment.id,
    title="My task",               # optional, human-readable label
    vault_ids=["vault_abc"],       # optional — attach vaults for MCP auth credentials
    resources=[                    # optional file/repo mounts
        {
            "type": "github_repository",
            "url": "https://github.com/org/repo",
            "authorization_token": "ghp_...",
            "checkout": {"type": "branch", "name": "main"},
            "mount_path": "/repo",
        },
        {"type": "file", "file_id": "file_abc123", "mount_path": "/data/input.csv"},
    ],
)
# Save session.id
```

**Session statuses:**

| Status | Meaning |
|---|---|
| `idle` | Waiting for input, or between turns |
| `running` | Actively processing |
| `rescheduling` | Transient error; auto-retry in progress |
| `terminated` | Unrecoverable error — stop your loop |

**Important:** You cannot delete a running session — interrupt first, then delete.
Files, environments, and agents are NOT deleted when a session is deleted — only the session
history and container are gone. Use archive to preserve history in read-only form.

For MCP setup with vaults, see `references/vaults-and-mcp.md`.
For session CRUD, usage/stats, and mid-session resource management, see `references/session-management.md`.
For production patterns, cost monitoring, debugging, and testing strategies, see `references/production-patterns.md`.

---

## Step 4 — The Event Loop

**Open the stream first, then send the message.**
If you send before opening, the API buffers events and you may miss the start. In Python,
always open the stream first to eliminate any race conditions.

```python
with client.beta.sessions.events.stream(session.id) as stream:
    client.beta.sessions.events.send(
        session.id,
        events=[{
            "type": "user.message",
            "content": [{"type": "text", "text": "Write a fibonacci script and test it"}],
        }],
    )

    for event in stream:
        match event.type:
            case "agent.message":
                for block in event.content:
                    print(block.text, end="", flush=True)

            case "agent.tool_use":
                print(f"\n[{event.name}]", flush=True)

            case "agent.custom_tool_use":
                # You must execute this and send back the result — agent is waiting
                result = my_execute_tool(event.name, event.input)
                client.beta.sessions.events.send(session.id, events=[{
                    "type": "user.custom_tool_result",
                    "custom_tool_use_id": event.id,
                    "content": [{"type": "text", "text": str(result)}],
                    "is_error": False,
                }])

            case "session.status_idle":
                # Could be done, or could need your input — always check stop_reason
                break

            case "session.status_terminated":
                raise RuntimeError("Session terminated")
```

**TypeScript:**

```typescript
const stream = await client.beta.sessions.events.stream(session.id);

await client.beta.sessions.events.send(session.id, {
  events: [{ type: "user.message", content: [{ type: "text", text: "..." }] }],
});

for await (const event of stream) {
  if (event.type === "agent.message") {
    for (const block of event.content) process.stdout.write(block.text);
  } else if (event.type === "session.status_idle") {
    break;
  }
}
```

---

## Steering Mid-Execution

```python
# Send a follow-up while the agent is running (or when idle)
client.beta.sessions.events.send(session.id, events=[{
    "type": "user.message",
    "content": [{"type": "text", "text": "Also add unit tests"}],
}])

# Stop the agent immediately
client.beta.sessions.events.send(session.id, events=[{
    "type": "user.interrupt",
}])
```

Sessions are stateful — history is persisted server-side. You can reconnect to an idle session
and continue without recreating anything.

---

## Tool Permission Handling

Tools have two permission modes: `always_allow` (auto-runs) or `always_ask` (pauses for your approval).
When a tool needs approval, the agent emits `session.status_idle` with `stop_reason.type = "requires_action"`.

```python
for event in stream:
    if event.type == "session.status_idle":
        if hasattr(event, "stop_reason") and event.stop_reason.type == "requires_action":
            for event_id in event.stop_reason.event_ids:
                client.beta.sessions.events.send(session.id, events=[{
                    "type": "user.tool_confirmation",
                    "tool_use_id": event_id,
                    "result": "allow",          # or "deny"
                    # "deny_message": "...",    # optional explanation on deny
                }])
        else:
            break  # genuinely done
```

For fine-grained tool configuration (disable tools, whitelist mode, per-tool policies), see `references/tools.md`.

---

## Session Event Quick Reference

Full reference with all fields and patterns: `references/events.md`

| Event | Key Fields | What to do |
|---|---|---|
| `agent.message` | `content[].text` | Print the text |
| `agent.tool_use` | `name`, `input` | Observe — runs automatically |
| `agent.custom_tool_use` | `id`, `name`, `input` | **Execute it**, send `user.custom_tool_result` |
| `agent.thinking` | `content` | Extended thinking — log or ignore |
| `session.status_idle` | `stop_reason` | Check `stop_reason.type` before acting |
| `session.status_running` | — | New turn started — informational |
| `session.status_terminated` | `error` | Unrecoverable — raise and stop |
| `session.error` | `error.retry_status` | Check if `retrying` / `exhausted` / `terminal` |
| `session.deleted` | — | Stream is over — break immediately |
| `span.model_request_end` | `model_usage` | Token counts per inference call |

---

## Common Patterns

### Reconnect to an existing session

```python
# Fetch full event history
events = client.beta.sessions.events.list(session.id)
for event in events.data:
    print(event.type, event.processed_at)

# Then open a new stream and continue
with client.beta.sessions.events.stream(session.id) as stream:
    client.beta.sessions.events.send(session.id, events=[{
        "type": "user.message",
        "content": [{"type": "text", "text": "Continue where you left off"}],
    }])
    ...
```

### Retrieve output files

```python
files = client.beta.files.list(scope_id=session.id)
for f in files.data:
    content = client.beta.files.download(f.id)
    content.write_to_file(f.filename)
```

### Archive vs delete

```python
client.beta.sessions.archive(session.id)  # read-only, history preserved
client.beta.sessions.delete(session.id)   # permanent — cannot delete a running session
```

---

## Research Preview Features

Three features require a separate beta header and access approval:

```
anthropic-beta: managed-agents-2026-04-01-research-preview
```

- **Outcomes** — Define a rubric; a grader evaluates the output and the agent iterates until done.
- **Multiagent** — One orchestrator delegates work to specialist subagents (one level deep).
- **Memory Stores** — Persistent document collections that survive across sessions.

See `references/research-preview.md` for full setup and code examples.
[Request access →](https://claude.com/form/claude-managed-agents)

---

## Rate Limits

| Operation type | Limit |
|---|---|
| Create (agents, sessions, environments, vaults, etc.) | 60 req/min |
| Read (retrieve, list, stream, etc.) | 600 req/min |

Organization-level spend limits and tier-based rate limits also apply.

---

## Branding Guidelines

When building a product powered by Claude Managed Agents:

**Allowed:** "Claude Agent", "{YourName} Powered by Claude"
**Not permitted:** "Claude Code", "Claude Code Agent", or any branding that mimics Claude Code products

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Missing beta header in raw HTTP | Add `anthropic-beta: managed-agents-2026-04-01` to every request |
| Sending message before opening stream | Open stream first — race conditions are real in Python |
| Not checking `stop_reason.type` on idle | `session.status_idle` fires for both completion AND required actions |
| Not handling `agent.custom_tool_use` | Agent hangs indefinitely waiting for `user.custom_tool_result` |
| Updating agent without passing `version` | Will fail — always pass the current `version` |
| Assuming environments share filesystems | Each session gets its own container — no shared state |
| `limited` networking without `allow_package_managers` | Pip/npm installs at runtime will fail |
| Putting MCP auth token on the agent | Wrong — URL only on agent; auth goes in a Vault via `vault_ids` on the session |
| Deleting a running session | Interrupt first, wait for `idle`, then delete |
| Expecting session delete to clean up files/agents/envs | Those are separate resources — session delete only removes history and container |
