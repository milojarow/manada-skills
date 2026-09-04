---
name: manada
description: Use when creating, configuring, scoping, deploying, or dispatching Claude Code subagents (plugin, project or user-global agents, or Agent SDK agents), when choosing between the interactive harness and the headless SDK, when deciding whether a fan-out, a pipeline, or a deterministic Workflow pays off, when a lobo must be confined or its startup cost bounded, or when running agents on a Claude subscription without an API key.
when_to_use: Trigger phrases — "subagente", "lobo", "manada", "fan-out", "lanza N agentes", "Agent SDK", "query()", "AgentDefinition", "el agente no aparece", "el lobo hace cosas que no debería", "cuánto cuesta lanzar un agente", "ultracode", "workflow de agentes", "agent teams".
---

# Manada — Claude Code subagents

Build and dispatch a pack of specialized subagents: workers you send to do one focused job in
their own context, then collect the result.

> **🐺 ACTIVE-SKILL MARKER:** Prefix your reply with 🐺 **only on turns where the work touches the `manada` domain** — Claude Code subagents: plugin/project/user scoping, harness vs SDK, fan-out and workflow patterns, lobo confinement and cost — regardless of the layer/project (frontend, backend, a local script — all count); what matters is whether *this turn* touches the domain. On turns that do NOT touch it (typecheck, build, deploy, git ops, editing or curl in other domains), **omit 🐺** even if the skill loaded earlier in the session. If other active skills also apply to the same turn, **stack their emojis** in the prefix.

## Overview

A subagent (a "lobo" of the pack) runs in its **own fresh context**, with its **own system
prompt, tools, and model**, and returns only its final message to the parent. Use one to keep
heavy work out of the main conversation, to run several focused jobs in parallel, or to apply
specialized instructions without bloating the main prompt.

Two runtimes, same agent definition:
- **Harness (interactive):** the parent session dispatches subagents with the **Agent tool**. Agents are markdown files. This is the default when you're working in a Claude Code session.
- **SDK (headless):** a program calls `query()` and defines agents with the `agents` / `AgentDefinition` option. This is for scripts, hooks, cron — anything with no interactive session.

**Three questions decide everything:** *where does the agent live* (scope), *what can it do and
how is it tuned* (customization), and *what does it cost to start* (budget). A lobo is not free
before it does anything: measured on this harness, a `general-purpose` lobo starts with
**~114k tokens** of context, an `Explore` lobo with **~38k**. The difference is the `CLAUDE.md`
hierarchy that every custom agent inherits and the built-in read-only types skip.

## When to use

- Creating, configuring, or deploying a subagent in any scope (plugin / project / user-global) or via the SDK.
- Deciding **scope** so an agent is available only where intended — and won't fire elsewhere.
- Customizing an agent: `model`, `tools`, `skills`, `memory`, `effort`, `permissionMode`, `hooks`, `mcpServers`, `maxTurns`, `background`, `isolation`, `color`.
- Giving an agent **cross-session memory**, or wiring agents into a skill's workflow.
- Choosing **harness vs SDK**, running agents on a **Claude subscription** (no API key), or deciding when **agent teams** or a **Workflow** (ultracode) fit instead of plain fan-out.
- Bounding what a lobo **costs to start** and what it may **touch** (confinement).

**Not for:** authoring the skill/plugin the agents ship in (that's `forjador-de-skills` + `superpowers:writing-skills`), or general prompt writing unrelated to subagents.

## Where things live

| Topic | Reference |
|---|---|
| The 3 scopes (plugin/project/user) + the two locks (location + description); the plugin-agent security cap on `hooks`/`mcpServers`/`permissionMode` and **how to lift it**; why `~/.claude/projects/<slug>/` is NOT where definitions live | [reference/scopes.md](reference/scopes.md) |
| Every frontmatter field (`model` incl. `fable`, `tools`/`disallowedTools`, `skills`, `memory`, `effort`, `color`, `permissionMode`, `hooks`, `mcpServers`, `maxTurns`, `background`, `isolation`, `initialPrompt`, `experimental`); what a subagent inherits at startup and what it does not | [reference/personalization.md](reference/personalization.md) |
| **The budget of a lobo** — measured startup sizes per agent type, where the tokens go (CLAUDE.md hierarchy, the skill listing, preloaded skills), the levers that cut it, and how to measure your own | [reference/context-budget.md](reference/context-budget.md) |
| Dispatching — fan-out vs pipeline vs overhead; foreground vs background; `fork` (inherit the conversation); `isolation: worktree`; continuing a running lobo with `SendMessage`; nesting depth and concurrency limits | [reference/dispatching.md](reference/dispatching.md) |
| **Workflows (ultracode)** — the deterministic graph of agents (`agent()` / `parallel()` / `pipeline()` / `phase()`), when it beats a plain fan-out, its caps, and the opt-in rule | [reference/workflows.md](reference/workflows.md) |
| Harness (Agent tool) vs headless SDK (`query()`); subscription auth; sessions (resume/continue/fork); agent teams (experimental) and skills with `context: fork`; the Node-version launch gotcha | [reference/harness-vs-sdk.md](reference/harness-vs-sdk.md) |
| Creating, deploying & integrating an agent into each scope; portability (the agent travels with the plugin) | [reference/creating-deploying.md](reference/creating-deploying.md) |
| Running a pack headless from a launcher script — the four anti-fork-bomb locks; splitting IO (bash) from judgment (LLM) with a digest; calibrating the input cap; gate→lock→fire-and-forget dispatch; per-lobo model/effort; why a comment claiming a BEHAVIOUR is not evidence the behaviour exists | [reference/headless-launcher.md](reference/headless-launcher.md) |
| Confining a headless lobo — why `allowedTools` without `Edit`/`Write` is **not** read-only (`Bash` subsumes them); enforcing with a `PreToolUse` hook; allowlist design; canary verification; failing closed; gating what a lobo pushes outward | [reference/headless-confinement.md](reference/headless-confinement.md) |
| Rewriting a tool's input from a `PreToolUse` hook (`updatedInput`) — bounding an unbounded `Read`, the runtime's own output caps, why a hook is not live in the session that registered it | [reference/pretooluse-input-rewriting.md](reference/pretooluse-input-rewriting.md) |
| Onboarding a NEW adversarial-reviewer agent/model — why a "found nothing" over real code is not evidence without a positive control first, the planted-defect method, how the measurement harness itself can be the thing that's broken, and why a watchdog must check `/proc` (`comm`, fd holder) rather than an exe path or output size before declaring a live reviewer dead | [reference/reviewer-calibration.md](reference/reviewer-calibration.md) |
| What a run costs — `total_cost_usd`/`num_turns` on the result message; telling SDK agents from interactive sessions via `entrypoint`; deduplicating `usage` by `message.id`; cache-bucket multipliers | [reference/cost-accounting.md](reference/cost-accounting.md) |

## Quick reference

Agent definition (markdown frontmatter — `.claude/agents/<name>.md`, `~/.claude/agents/<name>.md`, or a plugin's `agents/<name>.md`):

```yaml
---
name: image-reviewer
description: Use this agent to <when>. Only for <domain>.   # description = when it's chosen
model: sonnet            # sonnet | opus | haiku | fable | full id | inherit (default)
tools: ["Read", "Grep"]  # DECLARE it. Omit = inherits everything, Skill tool + its listing included
disallowedTools: ["Skill"]   # subtract from the inherited set (a listing you do not pay for)
skills: []               # preload = the FULL body of each, at startup; keep it to 0-1
memory: project          # user | project | local → persistent, cross-session learning
effort: medium           # low | medium | high | xhigh | max
maxTurns: 30             # returns partial output when hit
color: cyan              # red | blue | green | yellow | purple | orange | pink | cyan
# project/user scope only (ignored in a plugin): permissionMode, hooks, mcpServers
---
You are <role>. <system prompt — self-contained so the agent is portable>.
```

- **Dispatch (harness):** Agent tool with `subagent_type: "<plugin>:<name>"` (plugin), `"<name>"` (project/user), a built-in (`Explore`, `Plan`, `general-purpose`), or `"fork"` (inherits the whole conversation). Optional `model`, `isolation: "worktree" | "remote"`. The dispatch prompt is the ONLY channel to a non-fork worker — put every path/decision/context it needs there.
- **Dispatch (SDK):** `query({ prompt, options: { agents: { name: {...} }, allowedTools: ["Agent", ...] } })`.
- **Fan-out:** send N Agent calls in one message for independent work; collect each final message. **Continue** a lobo with `SendMessage` to its name or id; **stop** one with `TaskStop`.
- **Read-only job? Use `Explore` or `Plan`**: they skip `CLAUDE.md` and the git snapshot, and start ~3× lighter than a custom agent.

## The budget of a lobo, in three rules

1. **Declare `tools:`.** An agent that omits it inherits every tool, including `Skill` with the
   full skill listing. A lobo that will never invoke a skill does not carry the listing.
2. **Preload at most one skill.** `skills:` injects each skill's whole body at startup
   (`claude plugin details <plugin>` prints the on-invoke size per skill: 3k–14k tokens each).
   Five preloaded skills on eight lobos is hundreds of thousands of tokens before any work.
3. **Read-only work goes to `Explore`/`Plan`** (or an SDK lobo with `settingSources: []`); a custom
   harness agent cannot opt out of the `CLAUDE.md` hierarchy, and that is ~75k tokens here.

Measurements, the full lever list and the one-line command that measures a lobo's own startup:
[reference/context-budget.md](reference/context-budget.md).

## Common mistakes

- **Meeseek of decoration.** Spawning a subagent for sequential work the main session could do directly — pure overhead. Fan-out pays only when work is independent (parallel + context isolation + specialization).
- **Launching lobos with every tool and a stack of preloaded skills.** Each one starts with the listing plus every preloaded body; a fan-out multiplies it. Declare `tools`, preload ≤1 skill, pick `Explore`/`Plan` for reading. See the three rules above.
- **Expecting `hooks`/`mcpServers`/`permissionMode` from a plugin agent.** They're ignored for plugin-shipped agents (security). Move the agent to `.claude/agents/`, `~/.claude/agents/`, or the SDK to get them. See [reference/scopes.md](reference/scopes.md).
- **Dispatching with a thin prompt.** A non-fork subagent starts blind — no view of this conversation. If you don't put the file path, the error, the expected shape in the dispatch prompt, it works in the dark. When the worker genuinely needs everything you know, dispatch a `fork` instead of re-explaining.
- **Assuming the pack is flat by nature.** Nesting is allowed by default (three layers below the main session, `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`). A flat pack is a **policy** you enforce: keep `Agent` out of a lobo's `tools`, and set the depth to `1` where a fork-bomb would be catastrophic (headless launchers). See [reference/dispatching.md](reference/dispatching.md).
- **`~/.claude/agents/` for something meant to be scoped.** That's user-global — visible in every session. For "only this project" use `.claude/agents/`; for "ships with the skill" use the plugin's `agents/`.
- **Filesystem agent edited mid-session and expected to load.** Agents load at startup; restart (or reinstall the plugin) to pick up a new/edited agent.
- **Calling a lobo "read-only" because it has no `Edit`/`Write`.** `Bash` subsumes both (`sed -i`, `>`, `tee`) plus service stops, deletions and `rm -rf`. Real confinement is a `PreToolUse` hook with a deny-by-default allowlist — see [reference/headless-confinement.md](reference/headless-confinement.md).
- **Writing a Workflow because a task "feels big".** The Workflow tool runs only on the user's explicit opt-in (`ultracode`, "use a workflow"); without it, plain Agent calls in one message are the fan-out. See [reference/workflows.md](reference/workflows.md).
- **Vague `description`.** Auto-dispatch matches the description. "reviews things" leaks; "validate the generated images for a short, only for that project" stays in its lane. Combined custom-agent descriptions past ~15k tokens are truncated by the harness — keep each one tight.
