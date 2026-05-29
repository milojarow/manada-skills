# Dispatching — fan-out, pipeline, or don't

## The two shapes

- **Fan-out** — many workers in **parallel** from one point. Requires **independence**: no worker needs another's result. Wall-clock = the slowest single worker, not the sum. Plus context isolation (each branch is separate) and specialization (each can differ).
- **Pipeline** — **sequential** stages where each feeds the next (script → voice → captions). The opposite of fan-out; there's nothing to parallelize. An orchestrator runs it in order.

A task often has both: a pipeline overall, with a fan-out *inside* one stage (e.g. "generate N images in parallel" is fan-out; "script → images → compose" is pipeline).

## When fan-out pays vs when it's overhead

| Pays (independent + parallel) | Overhead (don't spawn) |
|---|---|
| Exhaustive mapping / reading many files at once | Sequential work where stage N needs N-1 |
| Multi-angle research (by name, by content, by layer) | A task the parent can just do directly |
| Adversarial verification (N skeptics refute a claim) | Synthesizing one doc from findings |
| Per-item validation (N items → N validators) | Only a handful of known files (~5) |
| A/B generation (N variants → pick best) | "I have a powerful tool, let's go big" |

The trap: spawning workers because you *can*. A subagent for sequential work the main session could do is pure overhead — another process, another round of tokens, latency. Architecture follows what the task needs, not how capable the agent feels.

## How to dispatch

- **Harness, parallel:** issue **several Agent tool calls in one message** — they run concurrently. Collect each final message.
- **Harness, single:** one Agent call with `subagent_type: "<plugin>:<name>"` (plugin) or `"<name>"` (project/user).
- **SDK:** `query({ prompt, options: { agents: {…}, allowedTools: ["Agent", …] } })` lets the main agent delegate by description; or run independent `query()` calls under `Promise.all` for raw fan-out.
- Every dispatch's prompt is the **only channel** to the worker — see [personalization.md](personalization.md).

## Limits

- **How many at once:** no fixed SDK cap. Concurrency is bounded by your **token pool + rate limits + RAM** (each is a `claude` process). The harness's workflow runner caps concurrency around `min(16, cores-2)`.
- **One level only:** a subagent **cannot spawn its own subagents**. Don't give a subagent the `Agent` tool. The pack is flat: you (the parent) summon the lobos; lobos don't summon lobos.
- **Cost:** each worker spends tokens. A huge pack for a small job is waste — match the pack size to the work.
