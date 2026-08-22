# Cost accounting for a pack — the SDK already tells you, most launchers throw it away

Every `query()` closes with a `{type: "result"}` message that carries **`total_cost_usd`** and **`num_turns`**. The launcher usually parses `.result` and drops the rest, so a fleet of headless lobos can run for months with **zero token record** while the number flows through the pipe untouched.

One command tells you whether that is happening:

```bash
grep -c 'total_cost_usd' <launcher-log>     # 0 = you are throwing it away
```

The fix is one line per launcher:

```bash
cost=$(printf '%s' "$out" | jq -r '.total_cost_usd // "?"' 2>/dev/null)
turns=$(printf '%s' "$out" | jq -r '.num_turns // "?"' 2>/dev/null)
log "DONE rc=$rc cost=$cost turns=$turns result=$result"
```

Emit it from the `.mjs` **and** parse it in the wrapper. A launcher that emits the field into an envelope nobody reads is the same as not emitting it.

## Telling SDK agents apart from interactive sessions, after the fact

Transcripts under `~/.claude/projects/**/*.jsonl` carry an **`entrypoint`** on every `assistant` record. It is the clean discriminator:

| value | who |
|---|---|
| `cli` | interactive Claude Code session |
| `sdk-ts` | Agent SDK agent (`query()` from Node) |
| `sdk-cli` | headless run in print-mode |

A lobo **does leave a transcript** even with `settingSources: []`, and it lands in the slug of the **cwd you launched it from**, not a directory of its own. If you want to attribute runs later, launch each kind of agent from a distinct cwd.

## 🔴 The trap that double-counts everything

One API response is written as **several JSONL lines** — one per content block (thinking, text, tool_use) — and **every line carries the same `usage` object**. Summing per line inflates the total several times over. **Deduplicate by `message.id` before summing.**

Positive control: the first response of a session must show `cache_read_input_tokens = 0` and a `cache_creation_input_tokens` equal to the startup context. If you see two consecutive lines with an identical `usage` and the same `requestId`, that is the double count.

## Weigh the buckets — raw tokens lie about proportion

A cache read is worth about 1/50 of an output token, so a raw token count misrepresents where the money went. Multipliers over the model's base input price:

| bucket | multiplier |
|---|---|
| cache read | 0.1× |
| cache write, 5-minute TTL | 1.25× |
| cache write, 1-hour TTL | 2.0× |

`usage.cache_creation` carries the `ephemeral_5m_input_tokens` vs `ephemeral_1h_input_tokens` split — without it you cannot pick the right multiplier, and on 1-hour TTL the error is a factor of two.

## Where the money actually is — measure before optimizing the pack

In one fleet measured across ~1,200 transcripts / ~50,000 API responses, the headless lobos were **5.8%** of the total weight. The other 94% was interactive sessions, whose cost is dominated by **context size** (a median of ~380K tokens re-read per response), not by the number of agents.

Two conclusions worth carrying:

- **Before optimizing a pack for cost, measure — it is probably not there.** Cutting agents to save money usually attacks the small half.
- **Do audit each lobo's model and effort.** In that same fleet, 3 of 5 launchers ran opus / effort `high` with `maxTurns` 30–80 out of inherited habit, and those 3 concentrated **37%** of the pack's spend. See the per-lobo tuning section in [headless-launcher.md](headless-launcher.md).
