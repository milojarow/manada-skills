# Personalization — tuning a single lobo

Every subagent is configured by its frontmatter (markdown agents) or its `AgentDefinition` (SDK). Same fields either way.

## Frontmatter fields

| Field | Values | Notes |
|---|---|---|
| `name` | lowercase-hyphen | identity; drives the invocation id |
| `description` | prose | **when** to dispatch it (auto-match). Keep it scoped to its domain |
| `model` | `sonnet` · `opus` · `haiku` · full id · `inherit` | default `inherit` (same model as the parent). Route cheap jobs to `haiku` |
| `tools` | array | what it may use; **omit = inherits all** parent tools |
| `disallowedTools` | array | subtract from the inherited/listed set |
| `skills` | array of skill names | **preloads the full skill content** into the agent at startup (not just the description). It can still invoke other skills via the Skill tool |
| `memory` | `user` · `project` · `local` | **persistent, cross-session memory** (see below). Omit = no persistence |
| `effort` | `low` · `medium` · `high` · `xhigh` · `max` | overrides the session effort for this agent |
| `color` | `blue` · `cyan` · `green` · `yellow` · `magenta` · `red` | UI tag |
| `maxTurns` | number | cap the agent's agentic turns |
| `background` (SDK) | bool | run as a non-blocking background task |
| `permissionMode` · `hooks` · `mcpServers` | — | supported in project/user/SDK; **ignored for plugin agents** (see [scopes.md](scopes.md)) |

## Persistent memory (the lobo that remembers)

By default a subagent is ephemeral — it does its job and the context evaporates. Set `memory` to make it accumulate learnings across runs:

- `memory: user` → `~/.claude/agent-memory/<name>/` — persists across **all** projects (e.g. recurring anti-patterns, conventions it keeps re-deriving).
- `memory: project` / `local` → narrower scope.
- Omit → no persistence (most workers stay here).

Use it for an agent whose value compounds with experience (a reviewer that learns your recurring issues), not for a pure one-shot.

## What a subagent inherits (it starts blind)

Each subagent runs in a **fresh conversation**. It does NOT see the parent's chat history or the parent's system prompt. What it gets:

| Receives | Does NOT receive |
|---|---|
| Its own system prompt (the `.md` body / `AgentDefinition.prompt`) | The parent conversation history or tool results |
| The **dispatch prompt** (the Agent tool's prompt string) — the only channel from parent to worker | The parent's system prompt |
| Project `CLAUDE.md` (when `settingSources` includes `project`) | Preloaded skills, unless listed in `skills` |
| Tool definitions (inherited, or the subset in `tools`) | |

**Consequence:** put everything the worker needs — file paths, the error text, the expected output shape, the decision so far — **in the dispatch prompt**. A thin prompt makes it work in the dark.

The parent gets the subagent's **final message** back (verbatim as the Agent tool result), and may summarize it. To preserve it verbatim in the user-facing reply, say so in the parent's prompt.

## Portability rule

Write the system prompt **self-contained** — inline the rules it needs rather than pointing at an absolute path it must read at runtime. Then the agent behaves identically on any machine it travels to (see [creating-deploying.md](creating-deploying.md)).
