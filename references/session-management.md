# Session Management Reference

## Session Object Fields

A retrieved session (`client.beta.sessions.retrieve(session_id)`) includes:

```python
session.id             # ses_...
session.status         # "idle" | "running" | "rescheduling" | "terminated"
session.agent_id       # agent it was created with
session.environment_id
session.title
session.created_at
session.updated_at

# Cumulative token usage across ALL turns in the session
session.usage.input_tokens
session.usage.output_tokens
session.usage.cache_read_input_tokens
session.usage.cache_creation_input_tokens
session.usage.server_tool_use_input_tokens

# Timing stats
session.stats.active_seconds    # time the agent was actually processing
session.stats.duration_seconds  # wall clock time from creation to now
```

`usage` is cumulative — add up multiple turns. Useful for billing estimation or enforcing budgets.

---

## Session CRUD

```python
# List all sessions (paginated)
sessions = client.beta.sessions.list()
for s in sessions.data:
    print(f"{s.id}: {s.status} — {s.title}")

# Retrieve one
session = client.beta.sessions.retrieve(session_id)

# Update metadata/title (cannot change agent or environment)
client.beta.sessions.update(session_id, title="New title")

# Archive (read-only, history preserved)
client.beta.sessions.archive(session_id)

# Delete (permanent — interrupts first if needed)
# Cannot delete a running session — interrupt first, then delete
client.beta.sessions.delete(session_id)
```

---

## Resources API — Add / Update / Remove Mid-Session

Resources (files, GitHub repos, memory stores) can be added, updated, or removed while a session is active.

### Add a resource

```python
resource = client.beta.sessions.resources.add(
    session_id,
    resource={
        "type": "file",
        "file_id": "file_abc123",
        "mount_path": "/data/new_file.csv",
    }
)
# resource.id — use this for update/delete
```

### List resources

```python
resources = client.beta.sessions.resources.list(session_id)
for r in resources.data:
    print(f"{r.id}: {r.type} at {r.mount_path}")
```

### Retrieve a resource

```python
resource = client.beta.sessions.resources.retrieve(resource_id, session_id=session_id)
```

### Update a resource (e.g. rotate a GitHub token without stopping the session)

```python
client.beta.sessions.resources.update(
    resource_id,
    session_id=session_id,
    resource={
        "type": "github_repository",
        "authorization_token": "ghp_new_token_here",
    }
)
```
This is the intended pattern for GitHub token rotation mid-session — no need to recreate the session.

### Remove a resource

```python
client.beta.sessions.resources.delete(resource_id, session_id=session_id)
```

---

## Files API — Retrieve Session Outputs

Files written by the agent to `/mnt/session/outputs/` inside the container are retrievable:

```python
# List output files for a session
files = client.beta.files.list(scope_id=session_id)
for f in files.data:
    print(f"{f.id}: {f.filename} ({f.size_bytes} bytes)")

# Download a file
content = client.beta.files.download(files.data[0].id)
content.write_to_file("output.xlsx")
```

Files are NOT deleted when a session is deleted — they persist as independent resources.
