# Tools Reference

## Built-in Agent Toolset

Enable with `{"type": "agent_toolset_20260401"}` on your agent. All 8 tools are on by default.

| Tool | Name | What it does |
|---|---|---|
| Bash | `bash` | Execute shell commands in a persistent shell session |
| Read | `read` | Read a file from the container filesystem |
| Write | `write` | Write a file to the container filesystem |
| Edit | `edit` | Perform exact string replacement in a file |
| Glob | `glob` | Fast file pattern matching (e.g. `**/*.py`) |
| Grep | `grep` | Regex text search across files |
| Web Fetch | `web_fetch` | Fetch content from a URL |
| Web Search | `web_search` | Search the web for information |

---

## Disabling Specific Tools

```python
agent = client.beta.agents.create(
    tools=[{
        "type": "agent_toolset_20260401",
        "configs": [
            {"name": "web_fetch", "enabled": False},
            {"name": "web_search", "enabled": False},
        ],
    }]
)
```

## Enable Only Specific Tools (whitelist mode)

Start with everything off, then enable only what you need:

```python
agent = client.beta.agents.create(
    tools=[{
        "type": "agent_toolset_20260401",
        "default_config": {"enabled": False},
        "configs": [
            {"name": "bash", "enabled": True},
            {"name": "read", "enabled": True},
            {"name": "write", "enabled": True},
        ],
    }]
)
```

## Tool Permission Policies

Two levels of control: toolset-wide default, then per-tool overrides.

```python
agent = client.beta.agents.create(
    tools=[{
        "type": "agent_toolset_20260401",
        # Toolset-level default — applies to all tools not explicitly configured:
        "default_config": {
            "permission_policy": {"type": "always_ask"},   # require approval for everything
        },
        # Per-tool overrides — take priority over default_config:
        "configs": [
            {
                "name": "bash",
                "enabled": True,
                "permission_policy": {"type": "always_ask"},   # pause and ask each time
                # or:
                # "permission_policy": {"type": "always_allow"}, # run automatically (default)
            },
            {
                "name": "read",
                "enabled": True,
                "permission_policy": {"type": "always_allow"},  # read is safe — auto
            },
        ],
    }]
)
```

**Confirmation flow when `always_ask`:**
1. Agent wants to run a tool
2. API emits `session.status_idle` with `stop_reason.type = "requires_action"` and `event_ids`
3. You send `user.tool_confirmation` with `tool_use_id` and `result: "allow"` or `"deny"`
4. On deny, include optional `deny_message` explaining why — Claude sees this and adapts

```python
client.beta.sessions.events.send(session.id, events=[{
    "type": "user.tool_confirmation",
    "tool_use_id": event_id,       # from stop_reason.event_ids
    "result": "deny",
    "deny_message": "Don't delete files — only read and analyze",  # optional on deny
}])
```

**Note:** Custom tools are NOT governed by permission policies — they always pause for your
`user.custom_tool_result`. MCP tools default to `always_ask` — see `references/vaults-and-mcp.md`.

**When to use `always_ask`:** Production systems where you want human-in-the-loop for destructive
commands (bash, write, edit). Development: `always_allow` is fine.

---

## Custom Tools

Custom tools extend Claude's capabilities to your own systems. **Claude never executes custom tools** —
it emits a structured request and your code runs the operation.

### Define the tool on your agent

```python
agent = client.beta.agents.create(
    name="Data Agent",
    model="claude-sonnet-4-6",
    tools=[
        {"type": "agent_toolset_20260401"},   # keep built-in tools
        {
            "type": "custom",
            "name": "query_database",
            "description": """Query the internal analytics database using SQL.
                Use this when the user asks about metrics, user counts, revenue data,
                or any question that requires querying stored business data.
                Returns rows as JSON. Limit queries to 1000 rows max.
                Do NOT use for write operations — read-only.""",
            "input_schema": {
                "type": "object",
                "properties": {
                    "sql": {
                        "type": "string",
                        "description": "The SQL SELECT query to execute"
                    },
                    "database": {
                        "type": "string",
                        "enum": ["analytics", "users", "billing"],
                        "description": "Which database to query"
                    }
                },
                "required": ["sql", "database"],
            },
        },
    ],
)
```

### Handle custom tool calls in your event loop

```python
with client.beta.sessions.events.stream(session.id) as stream:
    client.beta.sessions.events.send(session.id, events=[{
        "type": "user.message",
        "content": [{"type": "text", "text": "How many users signed up last week?"}],
    }])

    for event in stream:
        if event.type == "agent.custom_tool_use":
            # event.name = "query_database"
            # event.input = {"sql": "SELECT COUNT(*) ...", "database": "users"}

            try:
                result = run_my_query(event.input["sql"], event.input["database"])
                client.beta.sessions.events.send(session.id, events=[{
                    "type": "user.custom_tool_result",
                    "custom_tool_use_id": event.id,
                    "content": [{"type": "text", "text": json.dumps(result)}],
                    "is_error": False,
                }])
            except Exception as e:
                client.beta.sessions.events.send(session.id, events=[{
                    "type": "user.custom_tool_result",
                    "custom_tool_use_id": event.id,
                    "content": [{"type": "text", "text": str(e)}],
                    "is_error": True,
                }])

        elif event.type == "agent.message":
            for block in event.content:
                print(block.text, end="")

        elif event.type == "session.status_idle":
            break
```

### Writing good custom tool descriptions (critical for performance)

The description is the primary signal Claude uses to decide when and how to call your tool.
A weak description = Claude calling the wrong tool or not calling it when it should.

**Good description structure:**
1. What the tool does (1 sentence)
2. When to use it — specific triggers and contexts
3. What parameters mean and their valid values
4. Important constraints and caveats (what NOT to do, limits, errors to expect)

**Example — weak vs strong:**

```
# Weak (too vague):
"description": "Get weather data"

# Strong:
"description": """Get current weather conditions and a 5-day forecast for any city.
    Use this when the user asks about weather, temperature, rain, or outdoor conditions.
    The 'location' parameter should be a city name (e.g. 'San Francisco', 'London') —
    not coordinates or zip codes. Returns JSON with temp_celsius, conditions, and
    forecast array. If the city is not found, returns an error — suggest alternatives."""
```

**Consolidate related operations into one tool with an action parameter:**
```python
# Instead of: create_ticket, update_ticket, close_ticket (3 tools, selection ambiguity)
# Do:
{
    "name": "manage_ticket",
    "description": "...",
    "input_schema": {
        "properties": {
            "action": {"type": "string", "enum": ["create", "update", "close"]},
            "ticket_id": {"type": "string"},
            "data": {"type": "object"}
        }
    }
}
```

**Namespace tool names when spanning multiple services:**
`db_query`, `db_write`, `storage_read`, `storage_write` — not just `query`, `read`.

**Design tool responses to return only high-signal information.** Return semantic, stable
identifiers (slugs or UUIDs) rather than opaque internal references. Include only the fields
Claude needs to reason about its next step. Bloated responses waste context window and make it
harder for Claude to extract what matters.

---

## MCP Servers (Two-Step Setup)

MCP auth is intentionally split: the server URL goes on the agent, credentials go in a Vault
attached to the session. This lets you rotate credentials without touching agent definitions.

### Step 1 — Define server on agent (URL only, NO auth here)

```python
agent = client.beta.agents.create(
    name="Research Agent",
    model="claude-sonnet-4-6",
    tools=[
        {"type": "agent_toolset_20260401"},
        # mcp_toolset REQUIRED — this is what actually exposes MCP tools to Claude
        {
            "type": "mcp_toolset",
            "mcp_server_name": "github",   # must match mcp_servers[].name exactly
            # optional: restrict which MCP tools are available
            # "allowed_tools": ["create_issue", "list_prs"],
        },
    ],
    mcp_servers=[
        {
            "type": "url",                 # required field
            "name": "github",              # referenced by mcp_toolset above
            "url": "https://mcp.github.com",
            # NO authorization_token here — auth goes in Vault
        }
    ],
)
```

### Step 2 — Attach vault with credentials at session creation

```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    vault_ids=["vault_abc123"],   # vault must contain credential for mcp.github.com
)
```

See `references/vaults-and-mcp.md` for full vault setup (creating vaults, credential types,
rotation, and constraints).

**MCP tool events:**
- `agent.mcp_tool_use` — agent called an MCP tool
- `agent.mcp_tool_result` — result returned from MCP server

**MCP permission policy:** Defaults to `always_ask` — you'll get `requires_action` events
unless you override to `always_allow` in the `mcp_toolset` tool entry.

**Networking note:** If environment uses `limited` mode, add `"allow_mcp_servers": true`
so MCP server requests aren't blocked.
