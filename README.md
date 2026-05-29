# manada-skills

**Your pack of specialized Claude Code subagents — built, scoped, and dispatched on purpose.**

## What is this?

`manada` is the knowledge for creating, customizing, deploying and invoking Claude Code **subagents** — the workers you dispatch to do a focused job in their own context and report back. It covers all three places an agent can live (inside a plugin/skill, a project, or user-global) plus building agents programmatically with the **Claude Agent SDK**, and how to tune each one (model, tools, preloaded skills, persistent memory, effort, color).

### Why this skill exists

- **Scope is two locks, not one.** *Where* the agent file lives decides who can see it; its *description* decides when it's chosen. Get either wrong and an agent fires in the wrong place — or never fires.
- **Plugin-shipped agents are capped for security.** `hooks`, `mcpServers` and `permissionMode` are ignored when an agent ships in a plugin. The capability is real — you get it back in project/user scope or via the SDK. Knowing this up front saves a confusing debug.
- **Most agents should be ephemeral; some should remember.** Persistent `memory` turns a one-shot worker into one that learns across sessions — powerful, and opt-in.
- **Fan-out pays only when work divides.** Parallel agents shine for independent work (isolation + concurrency); for sequential work they're pure overhead.
- **Subagents start blind.** Fresh context, no view of the parent conversation — whatever the worker needs goes in the dispatch prompt.

## The skill

| Skill | Description |
|-------|-------------|
| **manada** | Create, scope, customize, deploy and dispatch Claude Code subagents (harness + Agent SDK). |

## Installation

Add this marketplace in Claude Code:

```
/plugin → Marketplaces → Add Marketplace → milojarow/manada-skills
```

Then install:

```
/plugin → Discover → manada-skills → Install
```

## Requirements

- Claude Code (subagents + plugin support).
- For the headless path: Node.js + `@anthropic-ai/claude-agent-sdk` (optional — only if you build agents programmatically).

## License

MIT
