# Personalization — tuning a single lobo

Every subagent is configured by its frontmatter (markdown agents) or its `AgentDefinition` (SDK). Same fields either way.

## Frontmatter fields

| Field | Values | Notes |
|---|---|---|
| `name` | lowercase-hyphen, no `:` | identity; drives the invocation id. **Required** |
| `description` | prose | **when** to dispatch it (auto-match). Keep it scoped to its domain. **Required**. The harness truncates the combined descriptions of all custom agents past ~15k tokens: put detail in the body, not here |
| `model` | `sonnet` · `opus` · `haiku` · `fable` · full id · `inherit` | default `inherit` (same model as the parent). Route cheap jobs to `haiku`. A bare alias is a hidden `:latest` — pin the full id when the behaviour must not drift |
| `tools` | list | what it may use; **omit = inherits every parent tool**, including `Skill` (and with it the whole skill listing). Declare it |
| `disallowedTools` | list | subtract from the inherited/listed set. `disallowedTools: ["Skill"]` drops the listing from a lobo that never invokes skills |
| `skills` | list of skill names | **preloads the full body** of each at startup (not the description). Cannot preload a skill with `disable-model-invocation: true`; missing ones are skipped with a warning. The agent can still invoke other skills via the Skill tool unless `Skill` is removed |
| `memory` | `user` · `project` · `local` | **persistent, cross-session memory** (below). Omit = no persistence |
| `effort` | `low` · `medium` · `high` · `xhigh` · `max` | overrides the session effort for this agent |
| `color` | `red` · `blue` · `green` · `yellow` · `purple` · `orange` · `pink` · `cyan` | UI tag |
| `maxTurns` | number | cap the agent's agentic turns; it returns partial output when hit |
| `background` | bool | `true` keeps the lobo in the background even where the harness would otherwise run it in the foreground. Background lobos get a **narrower built-in tool set** (see dispatching.md) |
| `isolation` | `worktree` | runs in a temporary git worktree branched from the default branch, auto-cleaned if it made no changes; Bash/Monitor commands are checked to stay inside it |
| `permissionMode` | `default` · `acceptEdits` · `auto` · `dontAsk` · `bypassPermissions` · `plan` | project/user/SDK only — **ignored for plugin agents** (see [scopes.md](scopes.md)). A `fork` always inherits the parent's mode |
| `hooks` | map | lifecycle hooks scoped to this subagent, same event map as `settings.json` (`PreToolUse`, `PostToolUse`, …) written in YAML; the exact shape is in the official hooks doc ("hooks in skills and agents"). Project/user/SDK only |
| `mcpServers` | map | MCP servers available to this subagent. Project/user/SDK only |
| `initialPrompt` | string | auto-submitted as the first turn when the agent definition runs as a **main session** |
| `experimental` | map | e.g. `cacheTtl: 5m` or `1h` — prompt-cache TTL for the lobo's prefix |

## Persistent memory (the lobo that remembers)

By default a subagent is ephemeral — it does its job and the context evaporates. Set `memory` to make it accumulate learnings across runs:

- `memory: user` → `~/.claude/agent-memory/<name>/` — persists across **all** projects (e.g. recurring anti-patterns, conventions it keeps re-deriving).
- `memory: project` → `.claude/agent-memory/<name>/` — shareable through version control (the recommended default when the knowledge is about the project).
- `memory: local` → `.claude/agent-memory-local/<name>/` — project-specific, not committed.
- Omit → no persistence (most workers stay here).

On enable, the lobo's system prompt gains memory instructions, the first 200 lines / 25 KB of its `MEMORY.md` are injected with curation instructions, and Read/Write/Edit are auto-enabled for the memory dir. Tell it to check its memory before starting and to save what it learned when done — write that into its markdown prompt, not only into dispatch prompts.

Use it for an agent whose value compounds with experience (a reviewer that learns your recurring issues), not for a pure one-shot.

## What a subagent inherits (it starts blind)

Each non-fork subagent runs in a **fresh conversation**. It does NOT see the parent's chat history or the parent's system prompt. What it gets at startup:

| Receives | Does NOT receive |
|---|---|
| Its own system prompt (the `.md` body / `AgentDefinition.prompt`) + environment details | The parent conversation history or tool results |
| The **dispatch prompt** (the Agent tool's prompt string) — the only channel from parent to worker | The parent's system prompt |
| **Every level of `CLAUDE.md`** (user, project, nested) and a git status snapshot — **except** the built-in `Explore` and `Plan` types, which skip both | Skills, unless preloaded in `skills` (or invoked later through the Skill tool) |
| Tool definitions (inherited, or the subset in `tools`) — and with `Skill` among them, the whole skill listing | |
| Preloaded skills, full body | |
| The roster of sibling named agents (when it has `SendMessage`) | |

That inherited `CLAUDE.md` hierarchy is the single largest line item in a custom lobo's startup — measured at ~75k tokens on a machine with a large always-on memory index. Numbers and levers: [context-budget.md](context-budget.md).

**Consequence:** put everything the worker needs — file paths, the error text, the expected output shape, the decision so far — **in the dispatch prompt**. A thin prompt makes it work in the dark. When the worker needs the whole conversation, dispatch a **fork** (`subagent_type: "fork"`) instead of re-explaining: it inherits the system prompt, tools, model and full history, and only its result comes back.

The parent gets the subagent's **final message** back (verbatim as the Agent tool result), and may summarize it. To preserve it verbatim in the user-facing reply, say so in the parent's prompt.

## Portability rule

Write the system prompt **self-contained** — inline the rules it needs rather than pointing at an absolute path it must read at runtime. Then the agent behaves identically on any machine it travels to (see [creating-deploying.md](creating-deploying.md)).
