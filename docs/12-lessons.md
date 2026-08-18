# 12. What worked, what didn't

## What worked

Each of these is argued where it belongs; this is the index, not the argument.

- **Not owning the terminals** (3) — the single best decision, and the cost never arrived.
  If you build something in this space, start here.
- **Push over poll for agent state** (5, 6) — generic across tools, and fails loudly.
- **Typed failures before mutation, swallowed failures after** (4).
- **Deterministic first, model second** (9) — turned an unfalsifiable output into a
  reproducible one.
- **Cache-as-state with unconditional reconciliation** (2) — no rollback logic, and no way
  to stay wrong for more than one poll.
- **Deriving types from each other** (5, 10) — a missing label or icon is a compile error,
  not a runtime gap.

## What didn't

**Building agent-specific inference first.** The session-file-and-process-walk approach for
identifying agents was real work, shipped, then deleted entirely. The generic replacement
was shorter than the thing it replaced. The tell was there from the start: it encoded one
vendor's directory layout.

**Streaming keystrokes into an interface whose state I couldn't see.** Driving a
multi-question picker incrementally desynchronised the moment input arrived out of order,
and the failure mode was answering the wrong question rather than erroring. The fix —
accumulate, then submit once as an atomic burst — should have been the first design.

**Trusting a measurement that said zero.** Two separate counting bugs reported a tool as
unused: one missed slash-command invocations, one skipped symlinked directories. Both
looked like findings rather than defects, and one of them nearly got a heavily-used tool
deleted. *Anything reporting "unused" deserves suspicion before action.*

**A hardcoded number nobody checked.** An efficiency figure was repeated for a long
time before anyone computed it. Building the deterministic cost layer was largely a
reaction to that. An unverified number repeated often enough becomes a fact.

**Documentation drifting from code.** Preparing these chapters turned up several places
where the internal README described a design that had been replaced — a deleted module,
a responsibility that had moved to a skill, a field that does not exist in current
releases. Nothing enforced the relationship, so nothing maintained it.

## What a second attempt would change

**Own less, earlier.** The best decisions all pushed responsibility outward — to the
multiplexer, to git, to the agent's own tooling. The worst all pulled it in. The
task-file responsibility moving out to a skill should have happened far sooner.

**Design the multi-instance story up front.** Support for several concurrent copies was
retrofitted, and it shows: two independent port mechanisms with different constraints,
and several files needing per-instance isolation.

**Write the trust model down on day one.** The grants in chapter 11 are all defensible and
none was ever a decision. They accumulated. Writing them down at the start would not have
changed most of them, but it would have made the one or two worth reconsidering visible
while they were still cheap to change.

## Prior art

Cockpit is one point in a space several people are exploring independently. The nearest
sibling is a native terminal built around the same premise — agent multitasking as the
organising principle, workspace-per-task, a review queue — reaching similar conclusions
from a different starting point, which is usually a sign the problem is real rather than
the solution clever.

The obvious next step, and the one cockpit does not take, is multiplayer: several humans
and several agents coordinating as a team rather than one person supervising a fleet.
Everything here is single-player by construction. That is a limit of the design, not an
oversight — but it is the limit that will matter.
