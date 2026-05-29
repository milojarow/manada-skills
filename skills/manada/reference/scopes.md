# Scopes — where an agent lives decides who can use it

A subagent is controlled by **two independent locks**:

1. **Location (physical scope)** — *where the agent file lives* decides **who can see/invoke it**.
2. **`description` (semantic scope)** — auto-dispatch matches the description, so it decides **when** the agent is chosen, even among visible ones.

You need both. A perfectly-scoped location with a vague description still fires in the wrong place; a sharp description in a global location is still globally available.

## The locations

| Location | Available in | Precedence (name clash) | Travels with |
|---|---|---|---|
| `.claude/agents/` (project) | only that project — discovered by walking up from cwd | higher | the project repo |
| `~/.claude/agents/` (user) | **every** session, all projects | middle | nothing (your machine) |
| `<plugin>/agents/` (plugin) | wherever the plugin is enabled | lower | **the plugin** (marketplace) |
| `query({ agents: {…} })` (SDK inline) | only that `query()` call | n/a | your code |
| managed (org policy dir) | org-managed sessions | highest | org settings |

Notes:
- **Auto-discovery:** every `.md` in an `agents/` dir loads; `plugin.json` does **not** enumerate them.
- **Namespacing:** a plugin agent is invoked as `pluginname:agentname` (subfolders add colons: `plugin:sub:agent`). Project/user agents are just `agentname`. Name comes from the `name:` field; keep names unique per scope.
- **The slug dir is NOT for agent definitions.** `~/.claude/projects/<slug>/` stores session/subagent **transcripts** (and, by convention, memory files) — Claude Code does not scan it for agent `.md` files. Definitions = config (`.claude/agents/`, `~/.claude/agents/`, plugin); the slug dir = data.

## Pick a scope

- **Only this project** → `.claude/agents/` (physical lock; doesn't exist elsewhere).
- **Ships with a skill, portable to any machine** → plugin `agents/` (travels via marketplace, namespaced).
- **Every session, all your work** → `~/.claude/agents/` (use sparingly; this is the broad one).
- **Inside a headless program** → SDK inline `agents` (scoped to that run).

## The plugin-agent security cap — and how to lift it

**A plugin agent does NOT get `hooks`, `mcpServers`, or `permissionMode`** — those frontmatter fields are **ignored** when an agent loads from a plugin.

This is **a security restriction by provenance, not a capability ceiling.** A plugin comes from a marketplace (third party); letting it ship hooks (run code on events), MCP servers (connect external services), or `bypassPermissions` (skip your approvals) would mean installing arbitrary code/connections/permissions without your control.

**The same agent gets all three in any scope you author yourself:**

| Origin of the agent | `hooks` · `mcpServers` · `permissionMode` |
|---|---|
| **Plugin** (marketplace) | ❌ ignored (security) |
| **Project** `.claude/agents/` | ✅ supported — you wrote it, it's trusted |
| **User** `~/.claude/agents/` | ✅ supported |
| **SDK** `query({ agents: {…} })` / `AgentDefinition` | ✅ supported (also at the top-level `query()` options) |

So **a Claude agent absolutely can use MCP, hooks, and `permissionMode`.** The capability is first-class in the agent system; only the *plugin* delivery path caps it. If a worker needs an MCP server (e.g. to reach an API), lifecycle hooks, or its own permission mode, give it **project/user scope** (or define it via the **SDK**) instead of shipping it in a plugin. The trade-off: you gain the capability, you lose "travels-with-the-plugin." Choose scope by what the worker needs.

The doc says it plainly: *"if you need them, copy the agent file into `.claude/agents/` or `~/.claude/agents/`."*
