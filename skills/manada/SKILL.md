---
name: manada
description: Use when creating, configuring, deploying, or invoking Claude Code subagents — agents shipped inside a plugin/skill, project agents in `.claude/agents/`, user-global agents in `~/.claude/agents/`, or agents built programmatically with the Claude Agent SDK (`query()` / `AgentDefinition`). Use when picking a subagent's scope so it stays available only where intended; customizing its model, tools, preloaded skills, persistent memory, effort, or color; giving a subagent cross-session memory; wiring subagents into a skill's workflow; choosing the interactive harness (Agent tool) vs the headless SDK; running agents on a Claude Pro/Max subscription without an API key; or deciding when parallel subagents (fan-out) pay off vs add overhead. Also covers the security cap on plugin-shipped agents (hooks/mcpServers/permissionMode ignored) and how to lift it. Not for authoring the skill that agents live in — that is `forjador-de-skills` / `superpowers:writing-skills`.
---

# Manada — Claude Code subagents

Build and dispatch a pack of specialized subagents — workers you send to do one focused job in their own context, then collect the result.

> **🐺 ACTIVE-SKILL MARKER:** Prefija tu reply con 🐺 **solo en turnos donde el trabajo toca el dominio de `manada`** — subagents en Claude Code — scoping plugin/project/user, harness vs SDK, fan-out patterns. La **capa/proyecto da igual** (frontend, backend, n8n, script local — todos valen): lo que importa es si *este turno* toca el dominio. En turnos que NO lo tocan (typecheck, build, deploy, git ops, edición o curl de otros dominios), **omite 🐺** aunque la skill se haya cargado antes en la sesión. Si otras skills activas también aplican al mismo turno, **apila sus emojis** en el prefijo.

## Overview

A subagent (a "lobo" of the pack) runs in its **own fresh context**, with its **own system prompt, tools, and model**, and returns only its final message to the parent. Use one to keep heavy work out of the main conversation, to run several focused jobs in parallel, or to apply specialized instructions without bloating the main prompt.

Two runtimes, same agent definition:
- **Harness (interactive):** the parent session dispatches subagents with the **Agent tool**. Agents are markdown files. This is the default when you're working in a Claude Code session.
- **SDK (headless):** a program calls `query()` and defines agents with the `agents` / `AgentDefinition` option. This is for scripts, hooks, cron — anything with no interactive session.

**Two questions decide everything:** *where does the agent live* (scope) and *what can it do / how is it tuned* (customization). The reference files below split along those lines.

## When to use

- Creating, configuring, or deploying a subagent in any scope (plugin / project / user-global) or via the SDK.
- Deciding **scope** so an agent is available only where intended — and won't fire elsewhere.
- Customizing an agent: `model`, `tools`, preloaded `skills`, persistent `memory`, `effort`, `color`.
- Giving an agent **cross-session memory**, or wiring agents into a skill's workflow.
- Choosing **harness vs SDK**, or running agents on a **Claude subscription** (no API key).
- Deciding when **parallel subagents (fan-out)** pay off vs add overhead.

**Not for:** authoring the skill/plugin the agents ship in (that's `forjador-de-skills` + `superpowers:writing-skills`), or general prompt writing unrelated to subagents.

## Where things live

| Topic | Reference |
|---|---|
| The 3 scopes (plugin/project/user) + the two locks (location + description); the plugin-agent security cap on `hooks`/`mcpServers`/`permissionMode` and **how to lift it**; why the `~/.claude/projects/<slug>/` dir is NOT where agent definitions live (config vs data) | [reference/scopes.md](reference/scopes.md) |
| Per-agent customization — frontmatter fields (`model`, `tools`/`disallowedTools`, `skills`, `memory`, `effort`, `color`); what a subagent inherits (fresh context) | [reference/personalization.md](reference/personalization.md) |
| Harness (Agent tool) vs headless SDK (`query()`); subscription auth; sessions (resume/continue/fork); the Node-version launch gotcha | [reference/harness-vs-sdk.md](reference/harness-vs-sdk.md) |
| Dispatching — fan-out vs pipeline vs overhead; how to send N agents in parallel and collect them | [reference/dispatching.md](reference/dispatching.md) |
| Creating, deploying & integrating an agent into each scope; portability (the agent travels with the plugin) | [reference/creating-deploying.md](reference/creating-deploying.md) |

## Quick reference

Agent definition (markdown frontmatter — `.claude/agents/<name>.md`, `~/.claude/agents/<name>.md`, or a plugin's `agents/<name>.md`):

```yaml
---
name: image-reviewer
description: Use this agent to <when>. Only for <domain>.   # description = when it's chosen
model: sonnet            # sonnet | opus | haiku | full id | inherit (default)
tools: ["Read", "Grep"]  # omit = inherit all; this is read-only
memory: project          # user | project | local → persistent, cross-session learning
effort: medium           # low | medium | high | xhigh | max
color: cyan              # blue | cyan | green | yellow | magenta | red
---
You are <role>. <system prompt — self-contained so the agent is portable>.
```

- **Dispatch (harness):** Agent tool with `subagent_type: "<plugin>:<name>"` (plugin) or `<name>` (project/user); the dispatch prompt is the ONLY channel to the worker — put every path/decision/context it needs there.
- **Dispatch (SDK):** `query({ prompt, options: { agents: { name: {...} }, allowedTools: ["Agent", ...] } })`.
- **Fan-out:** send N agents in one batch for independent work; collect each final message.

## Common mistakes

- **Meeseek of decoration.** Spawning a subagent for sequential work the main session could do directly — pure overhead. Fan-out pays only when work is independent (parallel + context isolation + specialization).
- **Expecting `hooks`/`mcpServers`/`permissionMode` from a plugin agent.** They're ignored for plugin-shipped agents (security). Move the agent to `.claude/agents/`, `~/.claude/agents/`, or the SDK to get them. See [reference/scopes.md](reference/scopes.md).
- **Dispatching with a thin prompt.** The subagent starts blind — no view of this conversation. If you don't put the file path, the error, the expected shape in the dispatch prompt, it works in the dark.
- **`~/.claude/agents/` for something meant to be scoped.** That's user-global — visible in every session. For "only this project" use `.claude/agents/`; for "ships with the skill" use the plugin's `agents/`.
- **Filesystem agent edited mid-session and expected to load.** Agents load at startup; restart (or reinstall the plugin) to pick up a new/edited agent.
- **Vague `description`.** Auto-dispatch matches the description. "reviews things" leaks; "validate the generated images for a short, only for that project" stays in its lane.
