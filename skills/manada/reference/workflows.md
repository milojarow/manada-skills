# Workflows (ultracode) — the deterministic graph of agents

A plain fan-out is one stage: N Agent calls in one message, N results back. The **Workflow tool**
runs a *graph*: a JavaScript script that dispatches agents, fans out over their outputs, pipes
one stage into the next and collects typed results — deterministically, with the orchestration in
code instead of in the orchestrating model's judgment. This is what the literature calls "prompt
graph engineering": prompts as nodes of an explicit, executable graph that is itself the
engineering artifact.

## The opt-in rule

The tool exists, but it runs **only when the user explicitly asked for multi-agent
orchestration**: the keyword `ultracode` in the prompt, `/effort ultracode` for the session, the
user's own words ("use a workflow", "fan out agents", "orchestrate this with subagents"), a skill
whose instructions call it, or a named saved workflow. A task that would merely *benefit* from a
workflow does not count — describe what it would do and roughly cost, and ask. Workflows can
spawn dozens of agents.

## Shape of a script

Load the `workflow-authoring` skill before writing one; it holds the API and its gotchas. The
skeleton:

```js
export const meta = {                       // pure literal: no variables, no calls
  name: 'review-changes',
  description: 'Review changed files across dimensions, verify each finding',
  phases: [{ title: 'Review' }, { title: 'Verify' }],
}
const DIMENSIONS = [{ key: 'bugs', prompt: '...' }, { key: 'perf', prompt: '...' }]
const results = await pipeline(
  DIMENSIONS,
  d => agent(d.prompt, { label: `review:${d.key}`, phase: 'Review', schema: FINDINGS_SCHEMA }),
  review => parallel(review.findings.map(f => () =>
    agent(`Adversarially verify: ${f.title}`, { label: `verify:${f.file}`, phase: 'Verify', schema: VERDICT_SCHEMA })
      .then(v => ({ ...f, verdict: v }))
  ))
)
return { confirmed: results.flat().filter(Boolean).filter(f => f.verdict?.isReal) }
```

- `agent(prompt, opts)` — one lobo; `label`, `phase`, and a JSON `schema` for a typed result.
- `parallel([...thunks])` — run thunks concurrently; `pipeline(items, mapFn, thenFn)` — each item's
  second stage starts as soon as its first stage completes (no wasted wall-clock).
- `phase()` / `log()` for structure and progress; `args` is the input the caller passed.
- The script is plain JavaScript, passed inline; each run persists it under the session directory
  and returns the path, so iterating means editing that file and re-invoking with `scriptPath`.

## Caps and lifecycle

| Limit | Value |
|---|---|
| Concurrent agents | 16 |
| Agents per run | 1,000 |
| Items per `parallel()` / `pipeline()` call | 4,096 |
| Session guideline (default "medium") | keep it under ~15 agents; the user can raise it |

Runs go to the background: the tool returns a task id and a notification arrives on completion;
`/workflows` watches live progress. `resumeFromRunId` re-runs a stopped run and returns cached
results for every `agent()` call whose prompt and options did not change — only edited or new
calls run again (same session only). Saved workflows live in `.claude/workflows/` (project) or
`~/.claude/workflows/` (user); `/deep-research` is the bundled one. Prompt caching across the
fan-out defaults to 5 minutes (`subagentPromptCacheTtl: 1h` to widen).

## When it beats a plain fan-out

- Two or more **stages with data flowing between them**, or a per-item follow-up (verify each
  finding, re-generate each rejected asset) — the pipeline overlaps stages automatically.
- Results that must come back **typed** (`schema`) and be filtered in code, not eyeballed.
- Enough agents that keeping the orchestration in the main model's context would itself be the
  cost.

A single stage of independent work stays a plain fan-out: several Agent calls in one message.
Every lobo in a workflow still pays its startup ([context-budget.md](context-budget.md)); a
workflow does not make agents cheaper, it makes their wiring deterministic.
