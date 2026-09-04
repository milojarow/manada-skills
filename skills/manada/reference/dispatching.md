# Dispatching — fan-out, pipeline, or don't

## The two shapes

- **Fan-out** — many workers in **parallel** from one point. Requires **independence**: no worker needs another's result. Wall-clock = the slowest single worker, not the sum. Plus context isolation (each branch is separate) and specialization (each can differ).
- **Pipeline** — **sequential** stages where each feeds the next (script → voice → captions). The opposite of fan-out; there's nothing to parallelize. An orchestrator runs it in order.

A task often has both: a pipeline overall, with a fan-out *inside* one stage (e.g. "generate N images in parallel" is fan-out; "script → images → compose" is pipeline). When the graph has several stages **and** a per-item follow-up (review each finding, verify each claim), that is the shape the **Workflow tool** runs deterministically — see [workflows.md](workflows.md); it needs the user's explicit opt-in.

## When fan-out pays vs when it's overhead

| Pays (independent + parallel) | Overhead (don't spawn) |
|---|---|
| Exhaustive mapping / reading many files at once | Sequential work where stage N needs N-1 |
| Multi-angle research (by name, by content, by layer) | A task the parent can just do directly |
| Adversarial verification (N skeptics refute a claim) | Synthesizing one doc from findings |
| Per-item validation (N items → N validators) | Only a handful of known files (~5) |
| A/B generation (N variants → pick best) | "I have a powerful tool, let's go big" |

The trap: spawning workers because you *can*. A subagent for sequential work the main session could do is pure overhead — another process, another round of tokens, latency. Architecture follows what the task needs, not how capable the agent feels. And every worker pays its startup before doing anything: eight `general-purpose` lobos are ~900k tokens of context before the first tool call, eight `Explore` lobos ~300k ([context-budget.md](context-budget.md)).

## How to dispatch

- **Harness, parallel:** issue **several Agent tool calls in one message** — they run concurrently. Collect each final message.
- **Harness, single:** one Agent call with `subagent_type: "<plugin>:<name>"` (plugin), `"<name>"` (project/user), a built-in (`Explore`, `Plan`, `general-purpose`), or `"fork"`.
- **Harness, fork:** `subagent_type: "fork"` inherits the **entire conversation** — system prompt, tools, model, history, permission mode — and returns only its result. Use it when the worker needs everything you know; a `model` override is ignored for a fork. The `/subtask` command is the same thing from the prompt.
- **Harness, isolated checkout:** `isolation: "worktree"` gives the lobo its own git worktree (branched from the default branch, not your HEAD; auto-cleaned if unchanged). `isolation: "remote"` runs it in a cloud environment when that is enabled.
- **SDK:** `query({ prompt, options: { agents: {…}, allowedTools: ["Agent", …] } })` lets the main agent delegate by description; or run independent `query()` calls under `Promise.all` for raw fan-out.
- Every dispatch's prompt is the **only channel** to a non-fork worker — see [personalization.md](personalization.md).

## Verify a worktree before dispatching — "the directory exists" isn't "it's the right tree"

`git worktree add -b <branch> <dir> <base>` **fails if `<dir>` already exists** — typically a
leftover from an earlier dispatch in the same session — and the intuitive check lies about it:

```bash
test -d "$dir/src" && echo "OK"    # true even for a worktree left over from a stale earlier task
```

A worker dispatched into a stale worktree starts on a dead branch: its diff either resurrects old
code or conflicts with everything on merge. Worse, `add -b` **creates the branch before it fails**
on the existing directory, so a retry fails a *different* way ("a branch named X already exists")
and reads like an unrelated error to whoever's watching.

The check that actually discriminates — expected branch **and** HEAD equal to the base's — run
from inside the worktree, before dispatching anything into it:

```bash
cd "$dir" \
  && [ "$(git branch --show-current)" = "$branch" ] \
  && [ "$(git rev-parse HEAD)" = "$(git rev-parse "$base")" ] \
  && echo FRESH || { echo "DO NOT DISPATCH"; exit 1; }
```

**The cleanup that prevents the whole class:** once a worker's diff is merged, remove BOTH the
worktree (`git worktree remove --force`) and the branch (`git branch -D`). Leaving only one turns
the next `add` with that name into the failure above. Printing `git worktree list` in the same
turn as a dispatch is worth doing on its own — a long list of stale worktrees is itself the sign
that cleanup isn't happening.

## Foreground vs background

In an interactive session lobos run in the **background** by default: you keep working, the result arrives as a task notification, and permission prompts surface in the main session naming the lobo. A background lobo gets a **narrower built-in tool set** (Read, Grep, Glob, Bash, Edit, Write, NotebookEdit, WebFetch, WebSearch, TodoWrite, Skill, ToolSearch, Monitor, TaskStop, SendMessage, Artifact, worktree enter/exit) — no `Agent` tool among them. `background: true` in the definition pins a lobo to the background; `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` forces everything to the foreground. Before v2.1.186 background lobos auto-denied permission prompts.

## Continuing, stopping, waiting

- **Continue** a running or finished lobo with `SendMessage` to its name or id: a completed lobo auto-resumes in the background (v2.1.191+); one you stopped yourself does not. Messages from the launching agent count as task direction (mid-task corrections are fine); nothing sent this way changes its permissions or configuration.
- **Stop** one with `TaskStop`. Do not poll a lobo's output file: the harness re-invokes you when it finishes.
- A lobo you named can be reached again by that name only while no newer lobo took it (v2.1.199+); use the id from the spawn to reach an earlier one.

## Limits

- **How many at once:** the harness caps concurrent subagents at **20** by default (`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`, v2.1.217+; not enforced under ultracode). The Workflow tool caps at 16 concurrent. The SDK has no fixed cap: concurrency is bounded by your **token pool + rate limits + RAM** (each is a `claude` process).
- **Nesting:** allowed by default, **three layers below the main session** (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`). At the depth limit the `Agent` tool is withheld from lobos, except forks. **A flat pack is a policy, not a platform fact**: keep `Agent` out of a lobo's `tools`, and set the depth to `1` in any headless launcher, where a lobo re-spawning lobos is the fork-bomb described in [headless-launcher.md](headless-launcher.md).
- **Cost:** each worker spends tokens — its startup first, then its turns. A huge pack for a small job is waste — match the pack size to the work, and the agent **type** to the job (read-only → `Explore`/`Plan`).
