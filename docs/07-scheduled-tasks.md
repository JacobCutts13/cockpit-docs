# 7. Scheduled agent dispatch

Cockpit has a scheduler. What it schedules is unusual: mostly not work, but *agents*.

## The inversion

A conventional scheduled job does the thing. It runs, computes, writes, finishes.

The dominant pattern here is different. A scheduled task is a **deterministic launcher
whose entire job is to bring an agent into existence and then get out of the way.** The
task returns in under a minute. The real work — a long session of an agent reading,
editing, testing, and opening a draft pull request — happens after the task has already
reported success.

```
timer fires
   │
   ├─▶ create an isolated worktree for this run
   ├─▶ open a terminal window in it
   ├─▶ start the agent CLI
   ├─▶ paste one slash command
   └─▶ return { dispatched: true }        ← under 60 seconds
                                             the agent runs on far longer
```

The skill named by that slash command is the program. The task file is the loader. This is
the same division as chapter 4: **cockpit owns orchestration, the agent's tooling owns
every word the agent reads.**

The task contract is correspondingly tiny:

```ts
type ScheduledTask = {
  key: string;          // kebab-case; must match its own filename
  displayName: string;
  cron: string;         // standard 5-field, local timezone
  run: () => Promise<{ dispatched: boolean; reason: string | null }>;
};
```

There is deliberately no factory abstraction over dispatch, because the tasks differ in
their orchestration more than they share it. A registry is one explicit array. There is no
hot reload and no dynamic discovery — a task that is not imported does not exist.

## Why 60 seconds

`run()` is wrapped in a hard timeout. That constraint is not incidental; it is what
enforces the pattern. A task that tries to do the work itself hits the timeout and is
recorded as a failure, which is the correct outcome — it should have dispatched an agent
instead.

Failure states are deliberately not modelled separately. A failure is an ordinary
completed row carrying an error reason. Background work that fails *after* `run()` has
returned reports itself through a separate call that writes the same field.

## Scheduling semantics

Two decisions here are worth copying.

**The next run is computed from the last run, not from wall-clock.** A task overdue by
several intervals fires exactly once on the next tick, never N times to catch up. Nobody
wants a week of missed runs to arrive at once.

**First sighting seeds a baseline and persists it.** An unseen task records "last run =
now" and marks the row as a baseline so the UI still reports "never run". Holding that
baseline only in memory would recompute it as *now* on every tick, and the task would
never fire at all.

Startup also resets any row still marked as running, since a hard kill skips the cleanup
handler that would otherwise clear it.

## Unattended agents, and what makes that acceptable

Scheduled agents run with permission prompts disabled. There is no human awake to approve
a file write, so an agent that stalls on a confirmation dialog achieves nothing.

That is a real grant of authority. What makes it defensible — worktree isolation, the
draft pull request as the human gate, a wall-clock cap — is chapter 11's subject; it is
the security model, not a detail of scheduling. One bound is specific to this path though:

- **Panes start muted.** A scheduled agent's mid-run chatter would flood a chat channel,
  so its pane is created with mirroring off but its destination already pinned. When the
  agent's first turn ends, its final summary is posted and the pane is unmuted.

Muting is a per-pane flag, never a disconnection — tearing down the shared socket would
take every other integration dark at the same time.

There is one ordering subtlety: the completion listener is registered *before* the agent
starts, so an unusually fast first turn cannot end before anything is listening.

## The subprocess exit trap

There is a command-line runner for triggering a task manually, and it carries a warning
that generalises well beyond this codebase.

The runner exits as soon as `run()` resolves. Since the entire design puts the real work in
detached promises that outlive `run()`, **the runner kills exactly the thing you were
trying to test.** A task that sequences a second agent after the first appears to work
perfectly and silently never starts the second one.

Testing these tasks means triggering them inside the live application process, over the
debugging port, where the detached work survives. This is a direct and unavoidable
consequence of "dispatch, don't do": any runner whose lifetime is `run()`'s lifetime
truncates the task.

## Anti-patterns, recorded

The task directory carries its own house rules, and the reasoning behind two of them is
more broadly useful:

**Do not introduce OS-level schedulers.** No system launch agents, no crontab. Scheduling
stays inside the application so that a scheduled agent appears in the sidebar like every
other pane. A task running somewhere you cannot see it is a task you will not notice
failing — the entire value of the tool is that work is visible.

**Do not swallow errors.** Let them propagate so the scheduler records them. A task that
catches its own failure and returns success is invisible until somebody wonders why
nothing has happened for a long time.
