# Onboarding a new adversarial reviewer — never trust its first "found nothing"

When a NEW agent/model is put in the role of adversarial reviewer (a different provider, a
different CLI, a different version than the one already trusted for that job), its first task
must never be the real diff. A "found nothing" over real code is indistinguishable from: the
reviewer is broken · the prompt didn't arrive complete · the CLI truncated the input · the
default model isn't the one you assumed. **Zero findings without a positive control is not
evidence — it's an unmeasured instrument.**

## The method (~5 minutes)

1. Write a small file (≤30 lines) with **N defects planted on purpose**, of distinct classes,
   all falsifiable. Include at least one **wiring defect** (silent, doesn't crash) and at least
   one **contract defect** (a function promises something the caller doesn't honor).
2. Run the candidate reviewer **and the reviewer that already works** with the SAME prompt, in
   parallel.
3. Score by: how many of the N it found · whether it invented one that isn't there · whether it
   names the dimensions where it found nothing (vs. staying silent) · whether it cites
   `file:line` or just prose.
4. Only then hand it the real diff.

## Planted defects that work well as a control

- A handler passed raw to `onClick={submit}` whose first parameter is a boolean flag: the
  synthetic event arrives as the argument, is truthy, and the confirmation branch goes dead.
  Silent, doesn't throw, and disables a safety gate.
- An early `return` placed after `isSubmitting=true` and before the line that clears it: an
  eternal lock, the button goes dead with no visible error.
- `await fetch(...)` with no `try/catch`: any throw skips the line that clears the flag.
- `fetch` without checking `res.ok`: a 402/500 with a JSON body is processed as success and the
  callback receives `undefined`. The failure disappears with no exception.

## What tells apart a good reviewer

Beyond finding the N: that it says explicitly which dimension it attacked and found nothing on,
instead of staying silent — and that it refuses to fabricate a plausible-but-false defect when a
dimension is genuinely clean.

## The harness that measures the reviewer can itself be broken

The measurement rig fails before the thing it measures does — treat that as a first-class
failure mode, not noise:

- A missing dependency of the harness (e.g. a timing tool that isn't part of the base install on
  a given distro) can make BOTH runs exit immediately with an empty/near-empty output — which
  reads exactly like "both reviewers failed." Verify the harness actually ran (non-trivial
  output, exit code, elapsed time) before concluding anything about the reviewers.
- A one-shot agent run can take several minutes. A tool timeout shorter than that kills the run
  and returns a 0-byte file — another false negative that looks like the reviewer, not the
  harness, failed.

Same principle as the outgoing-diff gate in
[headless-confinement.md](headless-confinement.md#the-outgoing-gate-belongs-to-the-launcher-never-to-the-model):
a zero result is only meaningful once the instrument producing it is proven live.

## A live reviewer can look dead: check the process, not the launcher's name for it

A watchdog script that decides "the reviewer died, relaunch it" on weak signals can declare a
perfectly healthy run dead — and the fix (relaunching) burns the budget on a duplicate run
racing the original. Two false-negatives cause this, and they stack.

**Process check: match `comm`, not the `exe` basename.** Some CLIs install a thin launcher at
`/usr/bin/<name>` that execs a separately-versioned binary under `~/.<name>/downloads/...`, so
`/proc/<pid>/exe` resolves to a path whose basename doesn't match `<name>` at all — a watchdog
that filters `case "$exe" in */<name>) ...` misses a process that has been running for minutes.
`/proc/<pid>/comm` (the kernel's name for the main thread, 15 chars) still reports the tool's
short name correctly regardless of which binary is actually loaded. `pgrep -x <name>` matches on
`comm` and, unlike `-f`, never self-matches the watchdog's own command line. Use `/proc/<pid>/cwd`
only to tell apart *which* of several live instances you're looking at; if you must match on
`exe`, match by basename **prefix**, never equality.

**Output check: file size is not a certificate of death.** A process mid-run can leave its
output file at a few hundred bytes for a long time when the bulk of the answer only lands at the
end — a short file plus "process not found" reads as "abandoned," when the process was simply one
correct pgrep filter away from being found. The only signal that actually distinguishes "still
writing" from "gave up" is who holds the file descriptor:

    readlink /proc/<pid>/fd/1     # is <pid> still writing to this fd?
    fuser <file> 2>/dev/null      # or, from the file's side: who has it open?

**Renaming the file does not redirect the writer.** The fd follows the inode, not the path. If a
still-running process's output gets renamed "because it looked abandoned," the writer keeps
appending to the renamed file under its old fd, and anything later created at the old path starts
empty and stays empty. Recoverable by renaming back — same inode, the writer never noticed.

Cost of getting both checks wrong together, measured once: a reviewer already several minutes
into a real run, declared dead by basename-matching plus a short-output heuristic, and relaunched
twice — three identical review runs stacked on the same diff, burning three times the budget for
one answer.
