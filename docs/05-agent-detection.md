# 5. Knowing what an agent is doing

There are two questions to answer about any pane, and they need completely different
mechanisms:

1. **Is this pane running an agent?** Answered by inspection — chapter 3's process
   resolver.
2. **What is that agent doing right now?** Answered by the agent telling you.

The second is chapter 6's subject. This chapter covers why the split exists and what
happens to panes that aren't agents at all.

## The pull model, and why it was abandoned

Chapter 3 covers the abandoned identity mechanism — session files on disk plus a process
walk — and why it was replaced. The status half was wrong for a further reason worth
stating on its own: **it inferred rather than observed.** A status field on disk is
whatever the tool last chose to write, at whatever cadence it chose. There is no event, so
there is no moment at which you know something changed.

The lesson generalises past this codebase: **when a tool can tell you what it is doing, do
not deduce it.** Deduction gives you something that works today and breaks on the next
release without saying so.

## Status as a small closed set

The statuses an agent pane can be in are deliberately few:

```ts
const PANE_STATUS_LABELS = {
  working:       'working',
  'needs-input': 'needs you',
  done:          'done',
  error:         'error',
} as const;

type PaneStatusKind = keyof typeof PANE_STATUS_LABELS;
```

The union is derived from the label map so that the set of statuses and the set of
human-readable strings cannot drift apart. Adding a status without a label is a
compile error, and — because the sprite set is keyed by the same union (chapter 10) —
so is adding one without an icon.

Mapping an incoming event to a status is one exhaustive switch with a `never` guard in the
default branch. That guard is doing real work: a new event type is a compile error until
somebody makes an explicit decision about what it means visually, rather than silently
falling through to "working".

Two properties of this set are non-obvious and both were deliberate.

**`done` is sticky.** It has no expiry. It means "this agent produced a completed turn",
not "this agent finished recently". An agent that ended its turn yesterday still reads as
done.

**There is a fifth, unreported state.** The backend only holds statuses for panes that
have actually emitted one, so a freshly-started agent has no status at all. Rendering that
as `done` would be a lie — it claims a turn ended when none has. It gets its own visual
treatment, sharing the resting pose with `done` but distinctly faded. Every agent pane
gets an icon; none of them claims something untrue.

## Non-agent panes

Not everything in the tree is an agent. Builds, test watchers and plain shells also go
quiet, and they have no hooks to report with. Those get the fallback: a poll of the
multiplexer's per-*window* activity timestamp. There is no pane-scoped equivalent — the
field that looks like one is empty in current releases — which is acceptable here because
dispatched work is one pane per window.

The state machine is small and every constant in it is defensive. First sighting seeds
state without firing, or launching the app notifies you about every pane that already
exists. A minimum busy duration filters incidental output, so the signal is "something did
work and then stopped" rather than "something printed a line". And agent panes are skipped
*explicitly*: their interfaces redraw continuously so the transition would never fire
anyway, but skipping them by name keeps the accounting honest instead of accumulating
meaningless entries.

## The notification bus

Everything that observes an agent publishes to one small bus:

```ts
type NotifyEvent =
  | 'pane.idle'
  | 'agent.turn-end'
  | 'agent.tool-call'
  | 'agent.user-prompt'
  | 'agent.needs-input'
  | 'agent.forward-progress';

type Sink = (event: NotifyEvent, params: NotifyParams) => void | Promise<void>;
```

Sinks are awaited individually inside their own error handling, so one misbehaving
consumer cannot block the others. Two are registered: one drives the sidebar status, one
mirrors to chat.

The sidebar sink broadcasts the *whole* status map rather than a delta, which lets every
consumer in the renderer mirror it with one shared hook and no reconciliation logic. It
also reports whether a status actually changed, so the common case — a burst of tool calls
that all map to `working` — does not trigger a re-broadcast per event.

Both sinks drop some events deliberately. The sidebar ignores idle notifications for
non-agent panes because that surface does not show them, and ignores the internal
forward-progress signal because it exists to resolve a pending question elsewhere and must
not clobber a displayed status.

## A note on what got removed

An earlier version badged the dock with a count of agents waiting for input. When the
underlying trigger became generic it was deleted rather than migrated, because the right
semantics stopped being obvious — a count of *what*? Everything that fired? Everything
unread? Most candidates need read/unread tracking that does not exist.

Shipping a badge with defensible-but-arbitrary semantics would have been easier. Removing
it and leaving a note saying why is better.
