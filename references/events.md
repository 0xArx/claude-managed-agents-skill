# Complete Event Type Reference

All events include `id`, `type`, and `processed_at` (RFC 3339 timestamp, null if queued).
Events flow over SSE. Stream via `GET /v1/sessions/:id/stream`.

---

## User Events (you → agent)

### user.message
Sends a message to the agent. Starts the agent loop or continues it after idle.
```json
{
  "type": "user.message",
  "content": [
    {"type": "text", "text": "Your task here"},
    {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "..."}},
    {"type": "document", "source": {"type": "file", "file_id": "file_abc"}}
  ]
}
```
Content can be: `text`, `image`, `document` blocks.

### user.interrupt
Stops the agent mid-execution immediately.
```json
{"type": "user.interrupt"}
```
Session transitions to `idle` after interrupt is processed.

### user.tool_confirmation
Approve or deny a tool call when the tool's permission policy is `always_ask`.
Sent in response to `session.status_idle` with `stop_reason.type = "requires_action"`.
```json
{
  "type": "user.tool_confirmation",
  "tool_use_id": "evt_abc123",     // the agent.tool_use event id that needs confirmation
  "result": "allow",               // or "deny"
  "deny_message": "Too risky"      // optional, only meaningful on deny
}
```
For multiagent: include `session_thread_id` if the event came from a subagent thread.

### user.custom_tool_result
Your response to an `agent.custom_tool_use` event. Agent is paused waiting for this.
```json
{
  "type": "user.custom_tool_result",
  "custom_tool_use_id": "evt_xyz789",   // matches agent.custom_tool_use.id
  "content": [{"type": "text", "text": "result data here"}],
  "is_error": false                      // set true if your tool failed
}
```
For multiagent: include `session_thread_id` if the event came from a subagent thread.

### user.define_outcome
Research preview. Define what "done" looks like with a rubric.
See `references/research-preview.md`.

---

## Agent Events (agent → you)

### agent.message
Claude's text response.
```json
{
  "type": "agent.message",
  "id": "evt_...",
  "content": [{"type": "text", "text": "I'll write a script for that..."}],
  "processed_at": "2026-04-08T12:00:00Z"
}
```
Iterate `event.content` and print `block.text` for each text block.

### agent.thinking
Extended thinking content. Emitted separately from messages.
```json
{
  "type": "agent.thinking",
  "content": [{"type": "thinking", "thinking": "Let me analyze this..."}]
}
```
Usually just log this; not needed for end-user display.

### agent.tool_use
Agent invoked a built-in tool (bash, read, write, etc). Runs automatically.
```json
{
  "type": "agent.tool_use",
  "id": "evt_...",
  "name": "bash",
  "input": {"command": "python fibonacci.py"},
  "processed_at": "2026-04-08T12:00:01Z"
}
```
No action needed from you — the tool runs in the container.

### agent.tool_result
Result of a built-in tool execution.
```json
{
  "type": "agent.tool_result",
  "tool_use_id": "evt_...",
  "content": [{"type": "text", "text": "0 1 1 2 3 5 8..."}]
}
```

### agent.mcp_tool_use
Agent invoked an MCP server tool.
```json
{
  "type": "agent.mcp_tool_use",
  "id": "evt_...",
  "name": "tool_name",
  "mcp_server_name": "my-mcp-server",
  "input": {...},
  "evaluated_permission": "allow"   // "allow", "ask", or "deny"
}
```

### agent.mcp_tool_result
Result of an MCP tool execution.
```json
{
  "type": "agent.mcp_tool_result",
  "mcp_tool_use_id": "evt_..."
}
```

### agent.custom_tool_use
**Action required.** Agent wants to call one of your custom tools.
You must execute the tool and send `user.custom_tool_result`.
```json
{
  "type": "agent.custom_tool_use",
  "id": "evt_...",           // use this as custom_tool_use_id in your response
  "name": "get_weather",
  "input": {"location": "San Francisco"},
  "processed_at": "2026-04-08T12:00:02Z"
}
```

### agent.thread_context_compacted
Conversation history was compacted to fit the context window. Normal, no action needed.

### agent.thread_message_sent / agent.thread_message_received
Multiagent only. Agent communicated with another thread.
```json
{
  "type": "agent.thread_message_sent",
  "to_thread_id": "thread_abc",
  "content": [...]
}
```

---

## Session Status Events (agent → you)

### session.status_idle
Agent finished its current work and is waiting for input.
**Always check `stop_reason.type`** — it's either done or needs your input.

```json
{
  "type": "session.status_idle",
  "stop_reason": {
    "type": "end_turn"         // done naturally → break your loop
  }
}
```
```json
{
  "type": "session.status_idle",
  "stop_reason": {
    "type": "requires_action",
    "event_ids": ["evt_abc"]   // tool confirmation needed → send user.tool_confirmation
  }
}
```
```json
{
  "type": "session.status_idle",
  "stop_reason": {
    "type": "custom_tool_use", // custom tool needed → execute and send user.custom_tool_result
    "event_ids": ["evt_xyz"]
  }
}
```
```json
{
  "type": "session.status_idle",
  "stop_reason": {
    "type": "interrupted"      // user.interrupt was processed
  }
}
```
```json
{
  "type": "session.status_idle",
  "stop_reason": {
    "type": "retries_exhausted"  // all automatic retries failed — treat as terminal
  }
}
```

### session.status_running
Agent started a new turn of processing. Informational.

### session.status_rescheduled
A transient error occurred; the session is retrying automatically. No action needed.

### session.status_terminated
Session ended due to an unrecoverable error. Stop your loop and handle the error.
```json
{
  "type": "session.status_terminated",
  "error": {"type": "...", "message": "..."}
}
```

### session.error
An error occurred during processing.
```json
{
  "type": "session.error",
  "error": {
    "type": "overloaded_error",
    "message": "...",
    "retry_status": "retrying"   // "retrying" | "exhausted" | "terminal"
  }
}
```
- `retrying` — session will auto-retry, keep listening
- `exhausted` — retries used up; session goes to `retries_exhausted` idle stop_reason
- `terminal` — unrecoverable; session will terminate

### session.deleted
The session was deleted while the stream was active. All subsequent events are dropped.
```json
{"type": "session.deleted"}
```
Break your event loop immediately when you receive this.

---

## Span Events (observability)

### span.model_request_start / span.model_request_end
Wraps each model inference call.
```json
{
  "type": "span.model_request_end",
  "model_usage": {
    "input_tokens": 1200,
    "output_tokens": 450,
    "cache_read_input_tokens": 800,
    "cache_creation_input_tokens": 0
  }
}
```
Useful for tracking per-turn token costs.

### span.outcome_evaluation_start / ongoing / end
Research preview. Emitted during outcome evaluation loops. See `references/research-preview.md`.

---

## Complete Event Loop Pattern (with all stop_reason handling)

```python
def run_session(client, session_id):
    with client.beta.sessions.events.stream(session_id) as stream:
        for event in stream:
            match event.type:
                case "agent.message":
                    for block in event.content:
                        if block.type == "text":
                            print(block.text, end="", flush=True)

                case "agent.tool_use":
                    print(f"\n[tool: {event.name}]", flush=True)

                case "agent.custom_tool_use":
                    result = execute_my_tool(event.name, event.input)
                    client.beta.sessions.events.send(session_id, events=[{
                        "type": "user.custom_tool_result",
                        "custom_tool_use_id": event.id,
                        "content": [{"type": "text", "text": json.dumps(result)}],
                        "is_error": False,
                    }])

                case "session.status_idle":
                    sr = event.stop_reason
                    if sr.type == "end_turn":
                        print("\n")
                        return "done"
                    elif sr.type == "interrupted":
                        return "interrupted"
                    elif sr.type in ("requires_action", "custom_tool_use"):
                        for event_id in sr.event_ids:
                            client.beta.sessions.events.send(session_id, events=[{
                                "type": "user.tool_confirmation",
                                "tool_use_id": event_id,
                                "result": "allow",
                            }])
                        # Loop continues — agent will resume

                case "session.status_terminated":
                    raise RuntimeError(f"Session terminated: {event.error}")

                case "session.error":
                    if event.error.retry_status == "non_retryable":
                        raise RuntimeError(f"Non-retryable error: {event.error.message}")
                    # retryable errors — session handles retry automatically
```
