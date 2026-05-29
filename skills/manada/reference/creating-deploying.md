# Creating, deploying & integrating an agent

## 1. Create the agent file

An agent is a markdown file: frontmatter (see [personalization.md](personalization.md)) + a **self-contained system prompt**. Inline the rules it needs; don't make it read an absolute path at runtime (that breaks portability). Read-only worker? `tools: ["Read"]`. Needs to act? add `Edit`/`Write`/`Bash` (and pick the scope that allows the permission model you need — see [scopes.md](scopes.md)).

## 2. Deploy by scope

| Scope | Where to put the file | Activation | Travels |
|---|---|---|---|
| **Project** | `<project>/.claude/agents/<name>.md` | restart the session (agents load at startup) | with the project repo (if committed) |
| **User** | `~/.claude/agents/<name>.md` | restart the session | nowhere (your machine) |
| **Plugin** | `<repo>/agents/<name>.md` | bump version → push → `/plugin marketplace update <repo>` (then it's namespaced `repo:name`) | **with the plugin** |
| **SDK** | inline in `query({ agents: {…} })` | next run of your program | with your code |

**Startup-only gotcha:** filesystem agents load when the session starts. Create/edit one mid-session and it won't appear until you restart (or, for a plugin, reinstall/update + restart).

## 3. Ship a plugin agent (the portable path)

The agent lives in the plugin repo's `agents/` and rides the normal skill release:

1. Write `agents/<name>.md` in the repo (`agents/` sits at the repo root, sibling of `.claude-plugin/`; auto-discovered).
2. Bump the version in **both** `.claude-plugin/plugin.json` and `marketplace.json`.
3. `git add` + commit + push.
4. `/plugin marketplace update <repo>` (or restart) → the agent is now `repo:name`, available wherever the plugin is enabled.

**Portability checklist** (so it works 100% on another machine):
- `agents/` is **not** gitignored → the `.md` is committed and travels (`git ls-files agents/` shows it).
- System prompt is **self-contained** — no machine-specific paths; inputs arrive in the dispatch prompt at runtime.
- `model`/`tools` are built-in/aliases (resolve anywhere). Secrets (API keys) are per-machine and are NOT part of the agent — a worker that only reads/judges needs none.

## 4. Integrate into a skill's workflow

A plugin agent travels with the skill, but it won't insert itself into the work — the **SKILL.md must tell Claude when to dispatch it**. In the relevant step, name the agent and the trigger, e.g.:

> After generating the batch, dispatch one `repo:thing-validator` per item, in parallel; regenerate anything it flags before continuing.

Add a short "Subagents" section to the SKILL.md listing each agent, what it does, and **when not to** (fan-out pays with several items; for one, do it inline — see [dispatching.md](dispatching.md)).
