# Confining a headless lobo — `allowedTools` is not a sandbox

A headless lobo that runs unattended, as the machine's operator user, is only as confined as its
*enforcement point*. The common pattern below looks read-only and is not:

```js
allowedTools: ["Read", "Grep", "Glob", "Bash"],   // "READ-ONLY: no Edit/Write/Agent"  ← false
permissionMode: "bypassPermissions",
```

**`Bash` subsumes `Edit` and `Write`** (`sed -i`, `>`, `tee`) **and everything else**: stopping
services, deleting containers, `rm -rf`. If the user it runs as has passwordless sudo — common on a
single-operator host — the lobo effectively has root. The only thing standing between it and the
machine is a sentence in the system prompt.

The tell-tale asymmetry: the *deterministic* component that can kill processes sits behind an
explicit gate, while the LLM with full root runs with none. The piece with a real guarantee is
disabled; the piece without one is live.

## The threat model is not an attacker

It isn't someone with an interactive shell. It's two far more likely things:

1. **The model deciding on its own** that stopping or deleting something "helps the diagnosis."
2. **Prompt injection from the content the lobo reads.** An auditing lobo reads journals, logs and
   process names — all untrusted text that someone else may have written.

Against both, deny-by-default with a read allowlist is the answer. A prompt is not.

## Where enforcement goes: a `PreToolUse` hook, not `canUseTool`

With `permissionMode: "bypassPermissions"` the permission layer is skipped, so **`canUseTool` may
never be consulted**. The SDK is explicit that a `PreToolUse` hook deny outranks bypass — the hook
applies even when permissions are bypassed. That is the correct control point.

Also: with `settingSources: []` (the isolation every headless lobo wants — see
[headless-launcher.md](headless-launcher.md)) **filesystem hooks are not loaded**. The hook must be
passed programmatically in the options:

```js
hooks: {
  PreToolUse: [{
    hooks: [async (input) => {
      const v = check(input.tool_name, input.tool_input);   // pure function, unit-testable
      if (v.allow) return { continue: true };
      return { hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason: `guard: ${v.reason}. You are read-only by construction.`,
      }};
    }],
  }],
},
```

The `permissionDecisionReason` reaches the model, so use it to redirect the lobo to an allowed
command instead of leaving it flailing.

## Designing the allowlist

A list of permitted binaries is not enough. What the guard actually has to cover:

- **Interpreters denied outright** (`sh`, `bash`, `python`, `perl`, `node`, `ruby`, and also
  `xargs`, `timeout`, `nohup`, `env` with a command): arbitrary execution by definition, and a
  reading lobo never needs them.
- **Command substitution denied** (`$(...)`, backticks, process substitution) — the channel for
  smuggling anything inside a command that looks innocent from the outside.
- **Redirection only to `/dev/null`.** `2>/dev/null` is idiomatic and writes nothing; any other
  destination is a write.
- **Pipes and chaining allowed, but validate EVERY segment.** An `&&` joining one allowed and one
  forbidden command is denied because of the forbidden one.
- **Per-command rules, not just per-binary:** `sed` without `-i`, `find` without `-delete`/`-exec`,
  `awk` without `>` or `system()` (an awk program can write files from the inside), the write verbs
  of `systemctl`, mutating subcommands of the container runtime, `tee` only to `/dev/null`, `sudo`
  restricted to a short list of concrete reads and always with `-n`.
- **Secret-bearing paths denied in `Read`/`Grep`/`Glob` too** (keyrings, `settings.json`, private
  keys, `shadow`). A headless lobo's output usually lands in a log or a notification channel, so a
  leak there travels on its own.

## Verify with a canary, not only unit tests

Unit tests over the pure `check()` function are necessary but prove nothing about the hook being
*wired in*. The test that counts: **run the real lobo** and ask it for one allowed command and one
forbidden command whose effect is observable and inert — e.g. `echo X > /tmp/canary.txt`. If the
guard is not connected, the file appears.

The run must end with: the allowed command executed, the forbidden one denied with the reason
reaching the model, the canary file **absent**, and a normal exit.

Also worth sabotaging the guard (make it allow everything) to confirm the tests then fail. A test
that doesn't fail when you break what it tests isn't testing anything.

## The outgoing gate belongs to the launcher, never to the model

When a lobo's job ends in an irreversible outward action — pushing to a public repo, posting,
sending — the check in front of that action must live in the process that performs it.
**Whoever pushes is whoever gates, and the gater is not the model.** A lobo holding
`Bash(git:*)` also holds `push --no-verify`, so a pre-push hook is not a gate against prompt
injection: take the push away from the model and give it to the launcher, which greps the
outgoing diff against a deny-list *before* pushing.

Five requirements — R1–R3 measured against real machines, R4–R5 structural:

- **R1 — pin the instrument's binary.** In Claude Code's Bash tool, `grep` is a **function**
  from the shell snapshot that executes ugrep (`ARGV0=ugrep`). That is the norm, not a local
  quirk, and ugrep dies on a BRE `\|` coming from a deny-list: `error: empty (sub)expression`.
  A gate written with a bare `grep -f` dies on its first run. Use `/usr/bin/grep`, or validate
  the patterns against the chosen binary when installing the gate.
- **R2 — match the flag to the deny-list's dialect.** A deny-list written in ERE (`{n}`, `+`,
  `?`, `|`, `()`) and read by a plain `grep -f` is searched as **literal text**. Measured with
  six planted fake secrets (a token prefix, a JWT, a connection URI, a cloud key id, an IP, a
  phone number): `grep -f` found 0 of 6, `grep -Ef` found all 6. Pass `-E`, or declare the
  dialect in the deny-list header and marry it to the flag.
- **R3 — positive control on EVERY run, over a regex-only pattern.** Feed the instrument a
  string that must match and require a non-zero count; a run that cannot prove its instrument
  works cannot report a zero. The control string must exercise a pattern that **only a live
  regex** can catch (a fake token-shaped literal). A control that happens to hit a literal
  pattern of the deny-list does not discriminate — it passes identically with every regex in
  the file dead, which is how a first audit reported "0 matches" with a broken instrument.
- **R4 — fail closed.** Any instrument failure — exit status > 1, invalid pattern, deny-list
  file absent — means **no push, plus a log entry**; never "nothing matched, go ahead". A
  corrupt deny-list must not be able to turn into a green light.
- **R5 — the gate runs outside the model's reach**: in the launcher, after the model's work is
  committed and before anything leaves the machine.

## `git show` on the outgoing diff produces a false positive, per commit

Auditing an outgoing push by piping `git show <sha>` (or a range) into the deny-list grep also
feeds the grep `git show`'s own header — `Author: Name <email>` — and an email-shaped pattern in
the deny-list matches it every time. That is public commit metadata, not the content of the
change, and it makes the gate look like it is blocking a real leak on every single commit.

Use `git diff <sha>^..<sha>` (or `git diff @{u}..HEAD` for a range) instead — it carries only the
changed lines, no author/commit headers. Measured against two real pushes: `git show` produced
one phantom hit, `git diff` produced zero. If a gate is built on `git show`, it is not broken —
it is measuring the wrong thing.

## Fail closed

If the guard module is missing, the `import` throws and the lobo **does not run** — failing closed,
the right direction. But make sure the installer deploys the guard alongside the lobo: a guard that
only exists on the development machine leaves a fresh install with a broken import.
