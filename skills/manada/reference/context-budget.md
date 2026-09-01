# The budget of a lobo — what it costs before it does anything

A subagent's first API call already carries its whole startup context. Measured on one machine
(Claude Code 2.1.257, a large always-on `CLAUDE.md` + memory index, ~110 skills listed), by
sending each type a one-word task and reading the `usage` of its first assistant message:

| Agent type | Startup context | What it carries |
|---|---|---|
| `general-purpose` (all tools, custom-agent defaults) | **113,654 tok** | system prompt, every tool incl. `Skill` + the listing, the full `CLAUDE.md` hierarchy, git snapshot |
| a custom agent with 4 tools and **no `Skill`** | **99,505 tok** | same minus the listing and most tool descriptions |
| `Explore` (built-in, read-only) | **37,781 tok** | system prompt + tools incl. `Skill`; **no `CLAUDE.md`, no git snapshot** |

Two deltas fall out: the skill listing plus the extra tool descriptions are worth **~14k**, the
inherited `CLAUDE.md` hierarchy **~75k**. On a machine with a small `CLAUDE.md` the second number
shrinks; the first does not, it scales with how many skills are installed and enabled.

A fan-out multiplies this: eight `general-purpose` lobos ≈ 900k tokens of context before any tool
call; eight `Explore` lobos ≈ 300k.

## Where the tokens go, and the lever for each

| Line item | Lever |
|---|---|
| **`CLAUDE.md` hierarchy** (user + project + nested) + git snapshot — inherited by every custom agent, no field turns it off | Use **`Explore` / `Plan`** for read-only jobs (they skip both). For headless lobos, `settingSources: []` in the SDK ([harness-vs-sdk.md](harness-vs-sdk.md)). For everything else the lever is the size of the always-on files themselves: `/doctor` finds unused skills/plugins vs their context cost and migrates always-loaded guidance into on-demand skills |
| **Skill listing** (name + description of every enabled skill, inside the `Skill` tool) | Declare `tools:` without `Skill`, or `disallowedTools: ["Skill"]`, on any lobo that never invokes skills. Globally: gate whole plugins per repo (`claude plugin disable <p> --scope user`, then `enable --scope project` where they belong), set rarely-triggered non-plugin skills to `name-only` in `skillOverrides`, mark operator-only skills `disable-model-invocation: true` (they leave the listing entirely). `claude plugin details <plugin>` prints each plugin's always-on cost; the Skills row of `/context` shows the listing as the model receives it |
| **Preloaded skills** (`skills:`) — the whole body of each, at startup | Preload **≤1**; put the rest one Skill-tool call away. `claude plugin details` prints the on-invoke size of each skill (3k–14k tokens is typical) |
| **Agent descriptions** — the combined descriptions of all custom agents | Kept under ~15k tokens or the harness truncates; detail goes in the body |
| **Tool descriptions** | `tools:` declared narrow; a lobo with `Read`+`Grep` carries two descriptions, not forty |
| **Per-turn work** | `maxTurns`, `effort: low` for mechanical jobs, `model: haiku` where judgment is cheap |

## Measure your own

The transcript of a lobo is JSONL; the first assistant message's `usage` is its startup:

```bash
jq -c 'select(.type=="assistant") | .message.usage
       | {in:.input_tokens, cache_create:.cache_creation_input_tokens, cache_read:.cache_read_input_tokens}' \
   <transcript.jsonl> | head -1
```

Interactive lobos: the Agent tool result names the task's output file (a symlink to the
transcript). SDK lobos: the `init`/`result` messages carry `session_id`, and the transcript lives
under `~/.claude/projects/<encoded-cwd>/`. Dedupe by `message.id` before summing over a whole run
([cost-accounting.md](cost-accounting.md)).

A control before believing any of these numbers: send the same one-word task to two types and
confirm the difference has the sign you expect. The three rows above were taken that way, in one
session, minutes apart.

## The three rules, restated

1. **Declare `tools:`** on every custom lobo; omit `Skill` unless it must invoke skills.
2. **Preload ≤1 skill**; a lobo that needs five skills is a job for the main session or a Workflow with specialized, narrow lobos.
3. **Read-only → `Explore`/`Plan`**, never a custom agent that inherits the whole `CLAUDE.md` for a `grep`.
