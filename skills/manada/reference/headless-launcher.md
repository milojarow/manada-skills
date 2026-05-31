# Headless launcher — running a pack from a script, safely

When the pack runs with no interactive session (a hook, a timer, a fire-and-forget dispatch), a thin **launcher** wraps each `query()` call. These are the patterns that keep that launcher cheap, safe, and from melting down.

## The four anti-fork-bomb locks

A headless agent that can re-trigger itself (a hook spawning a `query()` that emits an event that fires the same hook) is a fork bomb. Every headless lobo carries **all four** locks:

1. **`settingSources: []`** — don't load the user's filesystem settings. Two payoffs: the lobo runs without the operator's `CLAUDE.md`/persona/auto-memory (no bias from the host environment), and it **does not inherit the user's hooks**, so finishing can't re-fire the dispatcher.
2. **No `Agent` in `allowedTools`** — a lobo that can't summon lobos can't multiply. The pack stays flat.
3. **`CLAUDE_HEADLESS=1`** exported — a guard the dispatcher checks before launching, so a headless run never spawns another headless run.
4. **Capped `maxTurns` / budget** — a hard ceiling on a single runaway lobo.

Drop any one and the others may not save you. Treat them as a set.

## Split the work: IO in bash, judgment in the LLM

A lobo asked to triage/dedup over a large backlog will **time out** if it `Read`s every file one by one — the round-trips dominate. Push the heavy IO into the launcher (plain bash/Node) and let the lobo do only the part that needs judgment:

- The launcher builds an inline **digest** — one line per item, `name: short description` — and passes it in the prompt.
- The lobo triages on the digest and only `Read`s the handful it flags as suspicious.

Measured effect: a triage that died at the turn/time ceiling over 100+ files dropped to a handful of turns once it read a digest instead of the raw files. The rule generalizes — **the launcher does the bulk scan, the lobo does the call.**

## Cap the input — but calibrate the cap

Feeding an unbounded backlog into a lobo wastes tokens and can blow the context window, so the launcher caps the input/delta it forwards. The cap is a real knob, not a safe default:

- Too tight starves the lobo — a ~100 KB cap lost roughly half of a real session (~190 KB of text).
- Size it to one realistic unit of work. A ~300 KB cap covers a full session-up-to-compaction (~75 K tokens, comfortably under a Sonnet-class context window) and the lobo still digests it.

Pick the cap from the *largest input you actually expect*, and log when you truncate so silent loss is visible.

## Dispatch: gate → lock → fire-and-forget

For a launcher hooked into session close or compaction, don't block the host while the lobo runs:

- **Gate** — bail early if there's nothing to do (and respect the `CLAUDE_HEADLESS` guard above).
- **Lock** — a single-instance lock so overlapping triggers don't stack N copies.
- **Fire-and-forget** — run the work in a detached subshell, `( … ) & disown`. It doesn't hang the close/compaction, and `disown` detaches it from the parent so the background job **survives the client exiting**.

## Tune model and effort per lobo, by env var

Not every lobo needs the same horsepower. Drive `model` and `effort` per lobo from an env var so the launcher sets them per role:

- **opus / high** for irreversible decisions — editing indexes, deleting files, scrubbing into public repos.
- **sonnet** (lower effort) for everything else — triage, routing, summaries.

This keeps the expensive tier on the few calls that can't be undone, and the cheap tier on the bulk.

See [harness-vs-sdk.md](harness-vs-sdk.md) for the SDK-side gotchas (Node launch, stdin, the result envelope, subscription auth) and [scopes.md](scopes.md) for why a headless run uses inline `agents`.
