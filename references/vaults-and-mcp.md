# Vaults and MCP Connector

Vaults are Anthropic-managed credential stores for MCP authentication.
They decouple server URLs (on the agent) from credentials (on the session),
so you can rotate tokens without touching agent definitions.

---

## Creating a Vault

```python
vault = client.beta.vaults.create(
    name="Production MCP Credentials",
    description="OAuth tokens for GitHub and Notion MCP servers",
)
# Save vault.id — attach it to sessions via vault_ids
```

Vaults are workspace-scoped (shared across all agents in your workspace).

---

## Adding Credentials

Two credential types: **`mcp_oauth`** (for OAuth-based MCP servers) and **`static_bearer`**
(for API keys / PATs stored as bearer tokens).

### static_bearer — for API keys and PATs

```python
credential = client.beta.vaults.credentials.create(
    vault_id=vault.id,
    type="static_bearer",
    mcp_server_url="https://mcp.github.com",   # must match agent's mcp_servers[].url
    token="ghp_your_token_here",
)
```

### mcp_oauth — for OAuth flows

```python
credential = client.beta.vaults.credentials.create(
    vault_id=vault.id,
    type="mcp_oauth",
    mcp_server_url="https://mcp.notion.com",
    # OAuth tokens are obtained via the OAuth flow separately and stored here
    access_token="...",
    refresh_token="...",
    expires_at="2026-12-01T00:00:00Z",
)
```

---

## Attaching a Vault to a Session

```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    vault_ids=["vault_abc123"],   # agent's mcp_servers will be authenticated from this vault
)
```

The API matches credentials by `mcp_server_url` — the URL in the vault credential must exactly
match the URL in the agent's `mcp_servers` array.

**Invalid vault auth does NOT block session creation** — the session starts, but MCP tools
from that server will fail with a `session.error` event. Authentication retries happen
on the next `session.status_idle` → `session.status_running` transition. You can decide
to block further interactions on this error, trigger a credential update, or allow the
session to continue without that MCP server.

---

## Rotating Credentials

```python
# Add new credential (old one stays active until you archive it)
new_cred = client.beta.vaults.credentials.create(
    vault_id=vault.id,
    type="static_bearer",
    mcp_server_url="https://mcp.github.com",
    token="ghp_new_token",
)

# Archive the old credential
client.beta.vaults.credentials.archive(old_credential.id, vault_id=vault.id)
```

**Constraint:** Only 1 active credential per `mcp_server_url` per vault.
If you create a second credential for the same URL, archive the first one first.

---

## Vault Constraints

| Constraint | Limit |
|---|---|
| Active credentials per `mcp_server_url` per vault | 1 |
| Max credentials per vault | 20 |
| Vault scope | Workspace-wide (shared across all agents) |

Archiving a vault cascades — all credentials inside are also archived.

---

## Managing Credentials via API

```python
# List all credentials in a vault
for cred in client.beta.vaults.credentials.list(vault.id).data:
    print(f"{cred.id}: {cred.mcp_server_url} ({cred.type}) — {cred.status}")

# Retrieve a specific credential
cred = client.beta.vaults.credentials.retrieve(credential_id, vault_id=vault.id)

# Archive (soft delete — reversible)
client.beta.vaults.credentials.archive(credential_id, vault_id=vault.id)

# Archive the vault itself (cascades to all credentials)
client.beta.vaults.archive(vault.id)
```

---

## Full MCP Setup Example

```python
# 1. Create vault and add credentials (once)
vault = client.beta.vaults.create(name="My MCP Creds")
client.beta.vaults.credentials.create(
    vault_id=vault.id,
    type="static_bearer",
    mcp_server_url="https://mcp.github.com",
    token="ghp_...",
)

# 2. Define agent with server URL and mcp_toolset tool entry (once)
agent = client.beta.agents.create(
    name="Dev Agent",
    model="claude-sonnet-4-6",
    tools=[
        {"type": "agent_toolset_20260401"},
        {
            "type": "mcp_toolset",
            "mcp_server_name": "github",  # must match mcp_servers[].name
        },
    ],
    mcp_servers=[
        {"name": "github", "url": "https://mcp.github.com"},
    ],
)

# 3. Create session with vault attached (every time)
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    vault_ids=[vault.id],
)
```
