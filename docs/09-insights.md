# 9. Telemetry from transcripts

Agent CLIs write their sessions to disk as newline-delimited JSON — one event per line,
one file per session. That corpus is the only honest record of what your tooling actually
does, and it is sitting there unread.

Cockpit reads it to answer three questions: which of my tools get used, where do they
cause friction, and what does this cost.

## Deterministic first, model second

The design rule that makes this work: **compute everything computable in code; use a model
only for what genuinely requires judgement.**

The first attempt did the opposite. It handed transcripts to an agent and asked what it
noticed. The output was plausible, unfalsifiable, and different every run — the failure
mode of asking a model to do arithmetic.

The split now:

| Question | Method |
|---|---|
| How often was each tool invoked? | Counted in code |
| What did it cost? | Arithmetic over recorded token usage |
| Was a tool invoked and then corrected? | Pattern match over adjacent turns |
| Which prompts recur across sessions? | Clustering in code |
| Was the user actually satisfied? | Model, one call per session |

Everything in the first four rows is a pure function over synthetic input, so it is
unit-testable and identical on every run.

## Counting invocations two ways

A worked example of why deterministic does not mean easy.

Invocations were counted by looking for the model's tool-call events. A tool that is
almost always triggered by the human typing a slash command therefore read as **zero
uses** — and it was one of the most-used tools in the system.

Both forms have to be counted: the structured tool-call event, and the slash-command
marker recorded in a user message. Built-in commands share the marker syntax, so a
slash-command hit only counts when the name matches a tool that actually exists on disk.

A second bug in the same area: the on-disk tool list was built with a plain directory
check, which skips symlinks. Every symlinked tool silently read as unused — exactly the
signal that would get one deleted.

Both bugs had the same signature. **The measurement said zero, and zero looked like a
finding rather than a bug.** Anything reporting "unused" deserves suspicion before action.

## Filtering the noise

Two filters carry most of the value, and both exist because the first real run produced
garbage without them.

**Scaffolding.** Transcripts are full of injected harness text — system reminders, task
notifications, interruption markers, structured-output instructions. Left in, these
dominate every "repeated prompt" cluster, because they genuinely are the most repeated
text present. They are just not *prompts*.

**Environmental friction.** Rate limits, timeouts, connection resets, server errors. Left
in, these dominate every tool's friction score, and the ranking becomes a measure of which
tool ran during the worst network conditions. A network failure is not a design flaw in
the tool that happened to be running.

Both filters are deliberately broad. Under-filtering produces confidently wrong
conclusions; over-filtering produces a slightly thin dataset.

## Signals worth stealing

**Invoked-then-corrected.** A tool fires, and the *immediately following* human turn
matches a correction pattern — "that's wrong", "try again", "undo that". Attribution
breaks if another tool fires in between. Crude, and far better than anything self-reported.

**Repeated-prompt clustering.** Group session-opening prompts by their first few
normalised words; a cluster above a threshold is a candidate for a tool that should exist.

**Never model-triggered.** A tool invoked only by explicit command, never chosen by the
model, is under-triggering — its description is not pulling it in. That distinguishes
"nobody needs this" from "nobody can find this", which call for opposite actions.

## Cost

Pure arithmetic over the token counts already recorded in each transcript, against a
published price table fetched from a maintained open dataset. Models absent from the table
contribute zero rather than being guessed at from a family prefix.

The output carries a standing caveat: these are list prices, a pay-as-you-go equivalent,
not what a subscription actually charges. The number is for *comparison between* runs, not
for reconciling a bill.

This replaced a hardcoded efficiency figure that had been repeated for a long time
without anyone checking it. That is the real argument for this whole subsystem: **an
unverified number repeated often enough becomes a fact.**

## Measuring the loop

Each run appends a small record to an append-only log, so the next run can compute a
per-tool delta against the last. That turns "this got better" from an opinion into a
measured drop.

Trends are plotted by the *window* they describe, not by when the run happened, so
backfilled windows land in the right place on the axis rather than bunching at the
right-hand edge.

The related finding, which generalises to any dashboard: **show the trend, not the total.**
A cumulative count answers no question anybody has.
