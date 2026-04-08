# Production Patterns & Best Practices

Things you learn after actually building with managed agents that aren't in the API docs.

---

## Writing System Prompts for Agents

Prompting an autonomous agent is fundamentally different from prompting a chatbot. An agent
will be making many decisions on your behalf without checking in. The system prompt is your
only chance to shape all of them at once.

### Tell the agent where to put deliverables

If the agent writes a file but doesn't know you need it, it might write to `/tmp` (lost on
session end) instead of `/mnt/session/outputs/` (retrievable via Files API).

```
Always write final output files to /mnt/session/outputs/.
Write intermediate/scratch files to /tmp/.
```

### Define what "done" means

Agents without a clear definition of done tend to keep polishing indefinitely, or stop too
early. Be explicit:

```
You are done when:
1. The script runs without errors
2. All tests pass
3. The output file is written to /mnt/session/outputs/

Do NOT ask for confirmation. Proceed until these conditions are met.
```

### Handle uncertainty explicitly

If you don't tell the agent what to do when it's uncertain, it will improvise — sometimes
well, sometimes not. Give it a clear rule:

```
If you are uncertain about a requirement, make a reasonable assumption, document it
in a comment or README, and proceed. Do not stop to ask.
```

Or, if you want it to ask:
```
If you are uncertain about a critical requirement, send a message asking for
clarification before proceeding.
```

### Scope the tools

An agent with access to bash, web_fetch, and file writes can do a lot of damage if it
misinterprets a task. If your task is read-only analysis, say so:

```
You are a read-only analysis agent. You may read files and search the web,
but you must NOT write, delete, or modify any files.
```

And enforce it with tool configuration — don't rely on system prompt alone for security
boundaries. See `references/tools.md` for `always_ask` and whitelist mode.

### Be specific about output format

Agents take "write a report" very literally. Specify the format, length, and structure:

```
Write the analysis as a markdown file with these sections:
- Executive Summary (3-5 sentences)
- Findings (bullet points)
- Recommendations (numbered list)
- Appendix (raw data tables)
```

---

## Robust Event Loop for Production

The minimal event loop in the quickstart will break under real conditions. Here's a
production-grade version that handles reconnects, budget limits, and all stop_reason types.

```python
import time
import json
from anthropic import Anthropic

client = Anthropic()

def run_agent_task(session_id: str, message: str, max_tokens: int = 100_000) -> str:
    """
    Run a task and return when complete. Handles all stop_reason types,
    reconnects on stream drops, and enforces a token budget.
    """
    total_tokens_used = 0
    result_text = []

    # Open stream FIRST, then send message
    with client.beta.sessions.events.stream(session_id) as stream:
        client.beta.sessions.events.send(session_id, events=[{
            "type": "user.message",
            "content": [{"type": "text", "text": message}],
        }])

        for event in stream:
            match event.type:
                case "agent.message":
                    for block in event.content:
                        if block.type == "text":
                            result_text.append(block.text)
                            print(block.text, end="", flush=True)

                case "agent.tool_use":
                    print(f"\n[→ {event.name}]", flush=True)

                case "agent.custom_tool_use":
                    result = execute_custom_tool(event.name, event.input)
                    client.beta.sessions.events.send(session_id, events=[{
                        "type": "user.custom_tool_result",
                        "custom_tool_use_id": event.id,
                        "content": [{"type": "text", "text": json.dumps(result)}],
                        "is_error": False,
                    }])

                case "span.model_request_end":
                    # Track cumulative token spend — enforce budget
                    usage = event.model_usage
                    total_tokens_used += usage.input_tokens + usage.output_tokens
                    if total_tokens_used > max_tokens:
                        client.beta.sessions.events.send(session_id, events=[{
                            "type": "user.interrupt",
                        }])
                        raise RuntimeError(f"Budget exceeded: {total_tokens_used} tokens used")

                case "session.status_idle":
                    sr = event.stop_reason
                    match sr.type:
                        case "end_turn":
                            return "".join(result_text)
                        case "interrupted":
                            return "".join(result_text)  # partial result
                        case "retries_exhausted":
                            raise RuntimeError("Session retries exhausted")
                        case "requires_action" | "custom_tool_use":
                            # Confirm pending tool calls and keep looping
                            for event_id in sr.event_ids:
                                client.beta.sessions.events.send(session_id, events=[{
                                    "type": "user.tool_confirmation",
                                    "tool_use_id": event_id,
                                    "result": "allow",
                                }])
                            # Stream continues — don't break

                case "session.status_terminated":
                    raise RuntimeError(f"Session terminated: {event.error}")

                case "session.error":
                    if event.error.retry_status == "terminal":
                        raise RuntimeError(f"Fatal session error: {event.error.message}")
                    # retrying/exhausted — session handles it, keep listening

                case "session.deleted":
                    raise RuntimeError("Session was deleted mid-stream")

    return "".join(result_text)
```

---

## Cost Monitoring

### Per-turn cost from the stream

`span.model_request_end` fires after each inference call with token counts for that turn:

```python
case "span.model_request_end":
    u = event.model_usage
    print(f"Turn cost: {u.input_tokens} in / {u.output_tokens} out")
    print(f"Cache hits: {u.cache_read_input_tokens} read / {u.cache_creation_input_tokens} created")
```

A high `cache_read_input_tokens` ratio means caching is working well — the system prompt and
conversation history are being reused. If it's near zero, your sessions are too short to
benefit from caching, or the system prompt changes too frequently.

### Cumulative cost from the session object

After the session, check `session.usage` for the total across all turns:

```python
session = client.beta.sessions.retrieve(session_id)
u = session.usage
print(f"Total: {u.input_tokens + u.output_tokens} tokens")
print(f"Active time: {session.stats.active_seconds}s")
print(f"Wall time: {session.stats.duration_seconds}s")
```

`active_seconds` vs `duration_seconds` — the gap between them is time the agent spent idle
(waiting for your input, or between tasks). Useful for understanding actual compute cost.

### Rough cost estimation by model (April 2026 pricing — verify at console.anthropic.com)

| Model | Input | Output | Notes |
|---|---|---|---|
| claude-haiku-4-5 | cheapest | cheapest | Good for simple, fast tasks |
| claude-sonnet-4-6 | mid | mid | Best balance for most tasks |
| claude-opus-4-6 | expensive | expensive | Complex reasoning, architecture |

Managed agents sessions typically use 3-10x more tokens than a single API call for the same
task, because of context compaction overhead and multi-turn tool loops.

---

## Session Reuse vs New Session

**Reuse an existing session** when:
- You're continuing a task that was interrupted
- You want the agent to build on previous work (e.g., write tests for code it just wrote)
- You're having a multi-turn conversation about the same project

**Create a new session** when:
- The task is completely unrelated to the previous one
- The previous session accumulated a lot of context that would confuse the new task
- You need a clean container state (fresh filesystem)
- An error left the session in an uncertain state

```python
# Check if session is reusable before creating a new one
session = client.beta.sessions.retrieve(existing_session_id)
if session.status in ("idle", "running"):
    session_id = existing_session_id   # reuse
else:
    session = client.beta.sessions.create(agent=agent.id, environment_id=env.id)
    session_id = session.id            # fresh start
```

---

## Debugging a Session That Went Wrong

When an agent does something unexpected, replay the event history:

```python
def audit_session(session_id: str):
    events = client.beta.sessions.events.list(session_id)
    for event in events.data:
        match event.type:
            case "agent.message":
                print(f"[AGENT] {event.content[0].text[:200]}")
            case "agent.tool_use":
                print(f"[TOOL] {event.name}: {json.dumps(event.input)[:200]}")
            case "agent.tool_result":
                print(f"[RESULT] {str(event.content)[:200]}")
            case "user.message":
                print(f"[USER] {event.content[0].text[:200]}")
            case "session.status_idle":
                print(f"[IDLE] stop_reason={event.stop_reason.type}")
            case "session.status_terminated":
                print(f"[TERMINATED] {event.error}")
```

**Common things to look for:**
- Agent called a tool with wrong parameters → your tool description needs improvement
- Agent stopped at `requires_action` and your loop didn't handle it → check stop_reason handling
- Agent wrote to `/tmp` instead of `/mnt/session/outputs/` → update system prompt
- Lots of `agent.thinking` with no `agent.tool_use` → agent confused about which tool to use
- Token spike on a single turn → agent likely got a huge tool result back; trim your tool responses

---

## Testing Agent Applications

Full end-to-end sessions are expensive and slow to run in CI. Layer your tests:

### Layer 1: Unit test tool handlers (fast, free)

```python
def test_query_database_handler():
    result = execute_custom_tool("query_database", {
        "sql": "SELECT COUNT(*) FROM users",
        "database": "analytics"
    })
    assert "count" in result
    assert isinstance(result["count"], int)
```

### Layer 2: Test the event loop logic with a mock stream (fast, free)

```python
class MockStream:
    def __init__(self, events):
        self._events = events
    def __iter__(self):
        return iter(self._events)
    def __enter__(self): return self
    def __exit__(self, *args): pass

def test_event_loop_handles_requires_action(monkeypatch):
    mock_events = [
        MockEvent(type="session.status_idle", stop_reason=MockReason(
            type="requires_action", event_ids=["evt_123"]
        )),
        MockEvent(type="session.status_idle", stop_reason=MockReason(type="end_turn")),
    ]
    # Verify your loop sends tool_confirmation and then terminates cleanly
```

### Layer 3: Integration tests against real sessions (slow, run manually or nightly)

Keep a dedicated test environment and test agent. Use deterministic tasks with verifiable
outputs (e.g., "write a Python script that prints 42 and run it; the output must be '42'").

---

## Avoiding Silent Failures

These issues won't raise exceptions — the agent just silently does the wrong thing:

**1. Custom tool hang** — If you define a custom tool and your event loop doesn't handle
`agent.custom_tool_use`, the agent waits forever. Your loop appears stuck. Always log
unhandled event types in development:

```python
case _:
    print(f"[unhandled event: {event.type}]")
```

**2. Writing to wrong path** — Agent writes deliverables to `/tmp` or the working directory
instead of `/mnt/session/outputs/`. Files exist but are unretrievable after session ends.
Explicitly state output paths in your system prompt and verify with `files.list()` afterward.

**3. Wrong stop_reason handling** — If you break on any `session.status_idle` without
checking `stop_reason.type`, you'll exit the loop when the agent is waiting for tool
confirmation. The session stays open and idle, burning time. Always match on `stop_reason.type`.

**4. Stale agent version** — If you create a session with `agent=agent.id`, it pins to
the latest version at session creation. If you then update the agent, existing sessions
don't see the change. Create a new session if you need the updated agent.

**5. `limited` networking blocking runtime installs** — If your environment uses `limited`
networking and your system prompt tells the agent to `pip install` something, it will fail
silently (or with a confusing error) unless `allow_package_managers: true` is set. Pre-declare
heavy packages in the environment config instead.
