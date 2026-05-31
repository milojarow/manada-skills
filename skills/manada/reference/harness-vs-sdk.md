# Harness vs SDK — two ways to run the pack

Same agent definitions; two runtimes.

| | Harness (interactive) | SDK (headless) |
|---|---|---|
| What | The Claude Code session dispatches subagents with the **Agent tool** | A program calls **`query()`** from `@anthropic-ai/claude-agent-sdk` |
| Agents from | filesystem (`.claude/agents/`, `~/.claude/agents/`, plugin) | `agents` option (inline `AgentDefinition`) or filesystem |
| Use when | you're working in a session; a skill dispatches workers | scripts, hooks, cron, services — no interactive session |
| Capability cap | plugin agents capped (see [scopes.md](scopes.md)) | full (inline agents support `mcpServers`, `permissionMode`, etc.) |

Rule of thumb: **interactive work → harness** (the Agent tool already gives you subagents; you don't need the SDK). **Automation with no session → SDK.**

## Running on a Claude subscription (no API key)

The Agent SDK is `@anthropic-ai/claude-agent-sdk` (Anthropic's own). It can run against a **Claude Pro/Max subscription** — it inherits Claude Code's auth, so `apiKeySource` is `none` and it falls back to the OAuth credentials from `/login`.

- **Interactive / already logged in:** nothing to do — it uses the keychain from your login.
- **Unattended (timer/cron):** run `claude setup-token` (needs a browser once), export the result as `CLAUDE_CODE_OAUTH_TOKEN`.
- **Precedence gotcha:** if `ANTHROPIC_API_KEY` is set in the env, it **wins** and bills API credits instead of the subscription. Unset it to stay on the sub. Confirm you landed on the sub by checking `apiKeySource === "none"` on the `init` event — anything else means a key leaked into the env and you're paying API.
- **Billing:** before ~15 Jun 2026, SDK/subscription usage drew from the same pool as interactive Claude Code; from 15 Jun 2026 it moves to a **separate monthly credit** (Pro $20 / Max-5x $100 / Max-20x $200). Verify current terms.

## Node launch gotcha

On bleeding-edge Node (e.g. Node 26 on a rolling-release distro), the SDK's bundled native binary may fail to launch (`native binary exists but failed to launch`). Fix: point the SDK at the **system `claude`** binary —

```js
options: { pathToClaudeCodeExecutable: "claude" }   // or an absolute path
```

## Reading the prompt from stdin

A common launch shape is `printf '%s' "$prompt" | node lobo.mjs`. **Do not read stdin with `readFileSync(0)`** — when fd0 is non-blocking (typical under a pipe) it throws `EAGAIN` intermittently and drops the prompt. Read it as an async stream instead:

```js
let s = "";
process.stdin.setEncoding("utf8");
for await (const c of process.stdin) s += c;
```

## The result envelope

Each `query()` run ends in a result message whose `subtype` is one of: `success`, `error_exception`, `error_max_turns`, `error_max_budget`. Branch on it — `error_max_turns`/`error_max_budget` are caps you set (raise them or accept the partial), while **`error_exception: "socket connection closed unexpectedly"` is a transient network failure, not a bug** — retry it and it generally passes. Don't treat every non-`success` as a defect to debug.

## What the SDK loads by default (isolation)

- **`settingSources`** — omitting it loads user + project + local filesystem settings, **including `CLAUDE.md`** (matches the CLI). To run isolated (no `CLAUDE.md`, no settings) pass `settingSources: []`. Load only what you want, e.g. `["project"]`. (A brief v0.1.0 default of "load nothing" was reverted — use a recent version.)
- **`systemPrompt`** — defaults to **minimal**, NOT Claude Code's. For the full CLI behavior: `systemPrompt: { type: "preset", preset: "claude_code" }`. Independent of `settingSources`.
- **`effort`** — `low|medium|high|xhigh|max` (works with adaptive `thinking`); `model` per call. Set them low for cheap automation.

## Sessions (continuing context)

The SDK persists each conversation to disk and can resume it — context = the re-sent transcript, not model memory.

| Option | Does |
|---|---|
| `continue: true` | resume the **most recent** session in the cwd (no id) |
| `resume: "<session_id>"` | resume a **specific** session (read `session_id` off the result/init message) |
| `forkSession: true` (with `resume`) | branch a copy into a new session; original untouched |
| `persistSession: false` | stateless; nothing written to disk |

Transcripts live at `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`; a `resume` that returns a blank session is usually a **cwd mismatch**. Resuming re-materializes the full transcript, so token cost grows with history (prompt caching amortizes it).
