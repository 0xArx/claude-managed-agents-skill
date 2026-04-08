# Claude Managed Agents — Claude Code Skill

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that gives Claude deep, always-available knowledge of the **Claude Managed Agents API** — Anthropic's hosted infrastructure for running Claude as an autonomous agent in a cloud container.

Instead of you having to paste documentation or explain the API every time, this skill loads the right context automatically whenever you're building with managed agents.

> **Official documentation:** [platform.claude.com/docs/en/managed-agents/overview](https://platform.claude.com/docs/en/managed-agents/overview)
> This skill is a curated companion to those docs — focused on patterns, gotchas, and production usage.
> Always refer to the official docs for the latest API changes, new features, and full reference material.

---

## What This Skill Covers

| Topic | Where |
|---|---|
| Agent / Environment / Session creation | `SKILL.md` |
| The event loop, stream ordering, stop_reason handling | `SKILL.md` + `references/events.md` |
| All event types (user ↔ agent ↔ session ↔ span) | `references/events.md` |
| Built-in tools, custom tools, tool permissions | `references/tools.md` |
| MCP server integration (two-step auth pattern) | `references/tools.md` + `references/vaults-and-mcp.md` |
| Vaults and credential management | `references/vaults-and-mcp.md` |
| Container specs, pre-installed runtimes, filesystem layout | `references/container-reference.md` |
| Session CRUD, usage/stats fields, mid-session resource management | `references/session-management.md` |
| Outcomes, Multiagent orchestration, Memory Stores (research preview) | `references/research-preview.md` |
| System prompt engineering for agents, cost monitoring, debugging, testing, silent failures | `references/production-patterns.md` |
| Rate limits, branding guidelines, common mistakes | `SKILL.md` |

---

## Installation

### Option 1 — Clone directly into skills folder

```bash
git clone https://github.com/YOUR_USERNAME/claude-managed-agents-skill \
  ~/.claude/skills/claude-managed-agents
```

### Option 2 — Copy manually

Download or clone this repository, then copy the folder:

```bash
cp -r claude-managed-agents-skill ~/.claude/skills/claude-managed-agents
```

### Option 3 — One-liner (curl)

```bash
mkdir -p ~/.claude/skills && \
  curl -L https://github.com/YOUR_USERNAME/claude-managed-agents-skill/archive/refs/heads/main.tar.gz \
  | tar -xz -C ~/.claude/skills --strip-components=1 \
    --one-top-level=claude-managed-agents
```

That's it. Claude Code picks up skills automatically from `~/.claude/skills/` — no restart needed.

---

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — any recent version
- An `ANTHROPIC_API_KEY` with access to the `managed-agents-2026-04-01` beta

> **Note:** The managed agents beta is not universally available yet. If you don't have access,
> [request it here](https://claude.com/form/claude-managed-agents).

---

## How It Works

Claude Code uses a **progressive disclosure** system for skills:

1. The skill `name` and `description` are always in context — Claude uses these to decide when to activate the skill.
2. `SKILL.md` loads in full whenever the skill triggers — covers the complete build flow.
3. Reference files (`references/`) load on demand when Claude needs deeper detail on a specific topic.

This means the skill costs essentially zero context when you're not using it, and loads exactly what's needed when you are.

### When does it trigger?

The skill activates automatically when you:

- Ask about `client.beta.agents`, `client.beta.sessions`, or `client.beta.environments`
- Want to stream agent events or build an event loop
- Ask about tool permission policies, custom tools, or MCP connectors
- Mention outcomes, multiagent orchestration, or memory stores
- Ask how to run Claude for long-running or async tasks
- Want Claude to execute bash, write files, or browse the web autonomously in a container

---

## File Structure

```
claude-managed-agents/
├── SKILL.md                          # Core skill — always loaded when triggered
└── references/
    ├── events.md                     # Complete event type reference + full event loop pattern
    ├── tools.md                      # Built-in tools, custom tools, MCP setup
    ├── vaults-and-mcp.md             # Vault lifecycle, credential types, MCP auth
    ├── container-reference.md        # Container specs, runtimes, filesystem layout
    ├── session-management.md         # Session CRUD, usage/stats, mid-session resources
    ├── research-preview.md           # Outcomes, Multiagent, Memory Stores
    └── production-patterns.md        # System prompts, cost monitoring, debugging, testing
```

---

## Key Things Claude Will Know

Most managed agents bugs come from a handful of non-obvious patterns. This skill teaches Claude all of them:

**Stream ordering** — Open the event stream *before* sending the message, not after. The API buffers events but race conditions in Python are real.

**`stop_reason` on every idle** — `session.status_idle` fires both when the agent is done *and* when it needs your input (tool confirmation, custom tool result). Always check `stop_reason.type`.

**MCP auth split** — The server URL goes on the agent definition. Credentials go in a Vault attached at session creation. Never put auth tokens directly on the agent.

**Agent version locking** — Every `agent.update()` increments the version. You must pass the current `version` back or the update will fail.

**Custom tool hang** — If you define a custom tool and don't handle `agent.custom_tool_use` in your event loop, the agent silently waits forever.

**GitHub token rotation** — Update a GitHub resource mid-session via `sessions.resources.update()`. No need to delete and recreate the session.

**`session.deleted` terminates the stream** — If the session is deleted while you're listening, you'll get a `session.deleted` event and no further events will arrive.

---

## What This Skill Is NOT

This is a **Claude Code skill** — documentation that helps Claude write code using the Managed Agents API. It is not:

- An agent itself
- The Anthropic pre-built agent skills (`xlsx`, `pdf`, `docx`, `pptx`) — those are document-processing capabilities agents use during tasks
- A wrapper or SDK of any kind

---

## API Coverage

Built against the `managed-agents-2026-04-01` beta header (April 2026).

Research Preview features (`outcomes`, `multiagent`, `memory_stores`) require:
```
anthropic-beta: managed-agents-2026-04-01-research-preview
```
and separate access approval.

For the full, always-up-to-date API reference, visit the official docs:
**[platform.claude.com/docs/en/managed-agents/overview](https://platform.claude.com/docs/en/managed-agents/overview)**

---

## Contributing

Found something missing, outdated, or wrong? PRs welcome.

The skill is intentionally focused on **patterns and gotchas** — the things that aren't obvious from reading the API types. When contributing, ask: *would a developer hit this on their first or second day with the API?* If yes, it belongs here.

**Adding content:**
- Core concepts and the build flow → `SKILL.md`
- New event types or stop_reason variants → `references/events.md`
- Tool configuration patterns → `references/tools.md`
- Keep each reference file under ~300 lines; split if needed

---

## License

MIT — use it, fork it, remix it.
