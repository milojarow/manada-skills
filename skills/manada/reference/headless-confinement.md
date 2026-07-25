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

## Fail closed

If the guard module is missing, the `import` throws and the lobo **does not run** — failing closed,
the right direction. But make sure the installer deploys the guard alongside the lobo: a guard that
only exists on the development machine leaves a fresh install with a broken import.
