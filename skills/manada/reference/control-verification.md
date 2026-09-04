# Verifying a control delivered by an implementer — breakage alone doesn't discriminate

When an implementer (human or lobo) hands back code **with its own test/control**, that control
is suspect by construction: whoever wrote the fix has no incentive to write the check that
contradicts it. The standard verification — "break it and watch it go red" — is necessary and
**not sufficient**.

A control that goes red on ANY edit to the file looks identical to one that actually measures the
rule. So test it in **both** directions, and report the number, not just pass/fail:

1. **Breakage:** reintroduce the defect in the **production** code, never touching the control
   itself. Report **how many assertions turned red** — not just "it failed." The count says *what
   coverage was actually lost*: "6 red, and one of them is the case that matters most" says
   something a bare "failed" doesn't.
2. **Positive control:** make an **innocuous** change in the same area (a comment, `rows={3}` →
   `rows={4}`, a style class) that should **NOT** turn the control red. If it does, the control is
   measuring "nobody touched this file," not the rule it claims to defend.

## Three shapes of a hollow control, measured in the field

- **Lives on an `import` or a mention.** The name still appears in the import line even though
  nothing calls the function. Detect by deleting the real call site and checking whether the
  control still reports green.
- **Fabricates its own input.** It hands the pure function an object that already carries the
  field it claims to guard — so it exercises the function (which is fine) and can never see
  whether the caller actually wires it through. Measured case: deleting the one call site line
  left the control reporting PASS.
- **Isn't registered.** The control's file exists, reads like it protects something, and the
  runner never executes it — typically when a task scopes the implementer to N files and the
  registration lives in a different one. **An unregistered control does not exist** — checking
  that it's wired into the runner is part of reviewing the merge, not an afterthought.

## The sibling rule: when the RULE changes, the control moves — it doesn't get deleted

A control whose rule was deliberately revoked is **not deleted: it gets re-pointed at the new
boundary**. Deleting it throws away the protection along with the old restriction, and the gap
goes unnoticed because the file that would have flagged it no longer exists. Worked example: a
control once blocked overwriting a signed document because the server discarded the previous
version with no backup; once the server started keeping revisions, the reason the control was
born was still alive in a different shape — so it was rewritten to require a stated reason for
the replacement and to assert the prior version stays retrievable.

Signal that a control should move rather than be deleted: **the sentence it defends is still
true** — only *where* it's enforced moved.
