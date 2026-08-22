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

## One run per job: systemd template units, and why `%I` is the trap

When each dispatch is its own unit instance (`lobo@<job-id>.service`), the two specifiers that pass the instance name into `ExecStart` are **not** interchangeable — and the one that looks more robust is the dangerous one:

- **`%i`** — the raw instance name. Spaces arrive as `\x20`.
- **`%I`** — the *unescaped* name. `\x20` becomes a space again, but **every `-` becomes `/`**, because in systemd's escaping a hyphen encodes a path separator.

Measured: a unit with `ExecStart=… run.mjs "%I"` and instance `2026-08-20_1344_informe-mensual-1` received the path `2026/08/20_1344_informe/mensual/1`. Switching `%i` → `%I` "for robustness" turned a spaces bug into an invented-paths bug — and the launcher then writes its output into directories nobody asked for.

**Rule: if the job id is slugified (`[a-z0-9._-]`, no spaces), use `%i`.** The raw string already *is* the value. `%I` only helps when the id can contain spaces **and** contains no hyphens, which in practice is almost never.

Cheaper than fighting the escaping: **guarantee the id is safe at the source** — slugify when the job is created. One space in an id breaks three layers at once: systemd, the filesystem paths the launcher derives, and any `^[A-Za-z0-9._-]+$` id validator on the API or UI that later reads the run.

## The launcher must not name a default that belongs to the worker

Found in a coordinated two-machine deploy (caught by the *receiving* agent
verifying against the EFFECTIVE model in use, not the literal in the code —
that verification discipline is half the lesson).

A bash launcher for headless lobos logged:

```
log "LAUNCH gardener ... model=${GARDENER_MODEL:-opus}"
```

while the worker `.mjs` resolved:

```js
const model = process.env.GARDENER_MODEL || "sonnet";
```

Same env var, two languages, two defaults. The day the worker's default changed
(opus → sonnet), the launcher's did not — and from that deploy on, the log said
`model=opus` on every run while the pack actually ran sonnet. Nothing breaks
functionally, which is what makes it worse: the log is exactly where you go to
ask "which model ran this?" when auditing spend or debugging a strange result.

Minimal proof (same env, two paths, two answers):

```bash
bash -c 'echo "${VAR:-opus}"'                      # -> opus
node -e 'console.log(process.env.VAR || "sonnet")' # -> sonnet
```

**The fix is not syncing the literals.** Changing `:-opus` to `:-sonnet` leaves
the factory intact — they will drift apart again. The fix is that **the launcher
only asserts what it controls**:

```bash
log "LAUNCH gardener ... ${GARDENER_MODEL:+model-override=$GARDENER_MODEL}"
```

- If there's an env override, log it AS an override (that part the launcher does
  own).
- If not, log nothing — the authoritative line is whatever the worker itself
  prints at startup (e.g. `[gardener] init ... model=claude-...` on stderr,
  which the launcher already redirects into the same log). That comes from the
  process that actually ran.

**General rule, plus the widening it took a second finding to reach:** in any
launcher/worker pair (bash→node, systemd→binary, wrapper→CLI), a value with a
default has exactly ONE owner — the process that consumes it. Every other place
that wants to mention it either reads the effective value, logs only the
override, or stays silent. A second literal is debt with a detonation date.

The widening (same deploy, an hour later): the first patch searched for the
measured PATTERN of the bug (`:-opus` in a log line), not the CLASS (any
surface that names the value). Running an unfiltered `grep -rn 'opus'` over the
whole repo turned up six more assertions of the old default: the README (in the
billing sentence — the worst possible place), the tuning table (the canonical
surface for "what actually runs"), and three header comments — one twenty lines
from the `const` that contradicted it. The complete rule: the default has ONE
owner, and everything else — log, README, table, comment — either points at the
owner or mirrors it with the owner as tiebreaker. When patching: sweep the
CLASS (`grep -rn` for the old value across the whole repo), not the instance.
Historical mentions ("opus's premium used to be 37%") stay — they describe the
past, not a current default.

## A surface can lie about a BEHAVIOUR, not just a value

The same failure class as the section above, one level up — and the expensive one.

A launcher's header comment stated that it "scrub-gates the outgoing diff". No code applied
the deny-list: the gate existed only as an instruction in the **prompt** of the model it
launched. On the strength of that sentence, the lobo that pushes to public repos with a live
`GH_TOKEN` was moved down to a cheaper tier. It was caught only because the peer agent on the
other machine asked one question — *"is your gate code, or is it the same comment I just
measured in my own clone?"*. The tier was reverted, but the lobo had already run three times
at the lower tier and pushed to three public repos. The mechanical audit afterwards returned
0 matches — that time.

**A surface can misstate a VALUE (a default) or a BEHAVIOUR (a gate, a retry, a backup).** A
wrong value corrupts a log; a wrong behaviour corrupts a risk decision, because "there is a
gate" is exactly the premise on which someone lowers a safeguard. The only source for either
is the code that executes it: open the path that runs and find the enforcement, or treat the
behaviour as absent. Citing a comment as evidence is memory, not measurement.

## Tune model and effort per lobo, by env var

Not every lobo needs the same horsepower. Drive `model` and `effort` per lobo from an env var so the launcher sets them per role:

- **opus / high** for irreversible decisions — editing indexes, deleting files, scrubbing into public repos.
- **sonnet** (lower effort) for everything else — triage, routing, summaries.

This keeps the expensive tier on the few calls that can't be undone, and the cheap tier on the bulk.

### A bare model alias is a hidden `:latest`

`model: "sonnet"` / `model: "opus"` in code or a launcher's default (`|| "sonnet"`) is not a
pinned choice — the alias is a moving pointer the provider can repoint without anyone touching a
line of config. Measured in production: a fleet jumped from one model generation to the next
overnight with zero commits, because the alias moved under it. Two costs: it can't be audited
(no config anyone reviewed names the version actually running), and it breaks any cost/latency
series measured across machines or over time — the pointer can move mid-series on one host and
not another, and the two halves of the series stop being comparable.

**Rule: code and agent config always name the exact ID** (`claude-sonnet-5`, never `sonnet`) —
same doctrine as never using a container's `:latest` tag. This applies to what ships; an
operator typing an alias at an interactive prompt is a conscious, one-off choice and is fine.

The four locks keep the pack from multiplying; they do **not** bound what a single lobo can do to
the machine. For that, see [headless-confinement.md](headless-confinement.md) — note especially that
`settingSources: []` (lock 1) also means filesystem hooks don't load, so the guard hook has to be
passed programmatically.

See [harness-vs-sdk.md](harness-vs-sdk.md) for the SDK-side gotchas (Node launch, stdin, the result envelope, subscription auth) and [scopes.md](scopes.md) for why a headless run uses inline `agents`.
