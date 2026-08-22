# Rewriting a tool's input from `PreToolUse` — `updatedInput`

A `PreToolUse` hook is not limited to allow / deny / ask. It can hand back a **replacement input**
for the tool, which the runtime uses instead of the one the model produced. That turns the hook from
a gate into a *shaper*: a call that would be too expensive can be made cheap instead of being
blocked.

## The output contract

```json
{"hookSpecificOutput": {
   "hookEventName": "PreToolUse",
   "permissionDecision": "allow|deny|ask|defer",
   "permissionDecisionReason": "…",
   "updatedInput": { },
   "additionalContext": "…"
}}
```

`updatedInput` replaces the tool input before execution; the CLI logs the substitution
(`modified tool input keys: [ … ]`). The programmatic permission handler has the same lever in its
own shape: `{ behavior: "allow", updatedInput?: object }` / `{ behavior: "deny", message: string }`.

**There is no `PostToolUse` equivalent** — no `updatedOutput`. If the goal is to keep something out
of the context window, `PreToolUse` is the *only* control point; once the tool has run, the result
is already in the transcript.

## The motivating case: bounding an unbounded read

An unbounded `Read` of a large file is a **rent, not a purchase** — the result stays in the history
and is re-sent with every later turn of the session. Measured over a large corpus of real `Read`
calls, only about a fifth carried `offset`/`limit`, and the minority of unbounded calls that hit
large files accounted for the overwhelming majority of all bytes read.

The guard: when neither `offset` nor `limit` is present and the file exceeds the budget, compute a
`limit` from **that file's own average line length** (a 4,000-line config and a 4,000-line minified
bundle are not the same read) and return it in `updatedInput`.

### Four rules that make an input rewrite safe

1. **Explicit intent wins.** If the model passed `offset` or `limit`, change nothing. That escape
   hatch is checked *first*, before any budget logic.
2. **Never silently.** Announce the truncation with the real line count and an explicit "you did NOT
   see the whole file — do not conclude something is missing." A truncation that reads like a
   complete read is exactly how an agent concludes a function does not exist from half a file.
3. **Floor and no-op guard.** A `limit` below ~50 lines is not worth serving, and if the computed
   `limit` already covers the file, rewrite nothing.
4. **Fail open, always.** Any error in the guard must emit an empty decision and exit 0
   (`trap 'echo "{}"; exit 0' ERR`). A guard that can break a read is worse than no guard. Skip
   binaries and render-sensitive types (`.png`, `.pdf`, `.ipynb`) — a line limit means nothing there
   and only breaks the rendering.

Same discipline as a confinement guard ([headless-confinement.md](headless-confinement.md)): a pure,
unit-testable decision function, and a wiring check that is separate from it.

## The caps the runtime already ships

Set in the `env` block of `settings.json`:

| variable | default | ceiling |
|---|---|---|
| `BASH_MAX_OUTPUT_LENGTH` | 30,000 chars | 150,000 |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | exists; default not exposed | — |
| `MAX_MCP_OUTPUT_TOKENS` | exists | — |
| `TASK_MAX_OUTPUT_LENGTH` | exists | — |

**Exceeding a cap does not lose the output.** The overflow is written to the session's
`tool-results/*.txt` and the tool returns a short preview plus the path — observed on results ranging
from tens of kilobytes to tens of megabytes. Lowering a cap therefore converts the tail of a noisy
command into "preview + a path you can grep", not into missing data. That is what makes tightening
these safe.

## 🔴 A newly registered hook is not live in the session that registered it

`settings.json` is read at session start. A hook added mid-session passes a CLI unit test and **does
not fire** in that session — the read comes back unbounded and the guard's log stays empty. Split the
verification in two:

- **Logic** — feed the event JSON to the hook on stdin from the shell and inspect the returned
  decision.
- **Registration** — a *fresh* session, with an observable effect (the truncation notice, the guard's
  own log line).

Concluding "the hook doesn't work" from the session that installed it is the same false negative as
editing an agent definition mid-session and expecting it to load.
