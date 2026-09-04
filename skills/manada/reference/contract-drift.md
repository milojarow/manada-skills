# A closed enum against server-controlled values is a blast radius on the other side

Pattern measured across two sessions working the two sides of the same API contract.

**The failure:** the client enumerated every state the server could send. The server added one
member to its union type — from that side it reads as purely additive: it compiles, the server's
own tests pass, nothing breaks in that repo. But the client's parser hit a value outside its
closed list, returned `null`, and the **entire screen** went blank — not just the field the value
belonged to. The user lost real working time to it.

## Why it's invisible from the side that broke it

There is no way for the server side to see that adding a word to a union type silences a screen
on the other side of the wire. The change looks safe by every signal available where it was made.
This is exactly the failure mode two independently-dispatched sessions produce when each only
verifies its own half of a shared contract.

## The rule that discriminates — not "accept everything"

For every closed comparison against a server-controlled value, ask:

> **When this parser rejects, what does the person looking at the screen lose?**

- **A label, a status that only informs** → tolerate **by name**: render "server says
  `<raw value>`", keep the screen alive, forward the raw value to diagnostics.
- **The truth about money, quantities, or who owes what** → keep rejecting. Swallowing a
  contradiction about inventory or a balance is how a discrepancy goes unnoticed.

**Tolerance must not spill into coherence checks.** An unknown value that arrives **alone** gets
rendered. The same unknown value **paired with a list or an amount that contradicts it** gets
rejected.

A guard that stays strict should carry a one-line comment stating what would be lost if it were
relaxed — an unstated guard is the one someone deletes next year "because it's in the way."

## The agreement between the two sides isn't enough by itself

Standing agreement, both directions: notify before adding a state to a response; notify before
starting to REQUIRE a new field. Treat it as a net, not the fix — if the notice were the only
protection, the day someone forgets it repeats. What actually closes the gap is the reader
classifying **by behavior** ("arrived with no list and no count ⇒ this is a place that doesn't
count") instead of by a name list — a behavior-based reader survives a value it has never seen.

## Auditing a codebase for this — rarely done

The defect never lives in one file. Measure two counts per module: how many comparisons it closes
against server literals, and **how many surfaces consume it** (the blast radius) — then sort by
the second number. A reader nothing calls yet gets logged as debt and left alone: fixing what
nothing calls spends review risk without buying anything.
