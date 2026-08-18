# 2. Architecture

Cockpit is an Electron application with the standard three-process split, and almost all
of its interesting behaviour lives in one of them.

```
┌─────────────────────────────────────────────────────────┐
│  main process (Node)                                     │
│                                                           │
│   multiplexer control   git / worktrees   issue trackers  │
│   pane streaming        agent hooks       scheduler       │
│   chat socket           telemetry         secrets         │
└───────────────────────────┬─────────────────────────────┘
                            │  contextBridge
┌───────────────────────────┴─────────────────────────────┐
│  preload — a typed, flat API object. No logic.           │
└───────────────────────────┬─────────────────────────────┘
                            │  window.<api>
┌───────────────────────────┴─────────────────────────────┐
│  renderer (React) — sidebar tree, terminal view, panels  │
└─────────────────────────────────────────────────────────┘
```

The renderer never touches the filesystem, never shells out, and never talks to a network
service. Everything it needs arrives over one typed bridge. That is not a stylistic
preference — it is the containment boundary the security model rests on, and chapter 11
returns to it.

## No backend, no database, no HTTP

There is no server component. There is no persistent store beyond a handful of small
JSON files in a config directory. The application does not expose a port.

This is possible because cockpit is a *viewer* rather than a *system of record*. The
authoritative state already exists on the machine:

| State | Owned by |
|---|---|
| Which sessions, windows and panes exist | the multiplexer |
| What each pane has printed | the multiplexer's scrollback |
| Which branch a worktree is on | git |
| What issues are open | the issue tracker's API |
| What an agent is doing right now | the agent, which reports it |

Cockpit's job is to read those, join them, and render the result. It stores only the
things nobody else does: which pane is mirrored to which chat thread, when each scheduled
task last ran, and a few user preferences. Every one of those is a small JSON file.

The practical consequence is worth stating plainly: **quitting the app does not stop
anything.** Agents keep running. Restarting cockpit reconstructs its entire view from the
machine in a few hundred milliseconds, because it never held the truth in the first place.

## The IPC contract

One shared TypeScript module defines every type that crosses the bridge, and the renderer
and main process both import it. It is the only thing preventing the two halves from
drifting.

The API surface is deliberately flat — no namespacing, no nesting. Several dozen
request/response channels and a handful of push channels. The distinction between the two
is carried entirely by naming convention rather than by any runtime registry:

```ts
interface CockpitApi {
  // Request/response: returns a promise.
  getSnapshot(): Promise<Snapshot>;
  sendRaw(paneId: string, data: string): Promise<void>;

  // Push: takes a handler, returns an unsubscribe function.
  onPaneBytes(handler: (msg: PaneBytes) => void): () => void;
}
```

Every subscription returns its own disposer. That convention matters more than it looks:
React effects clean up by calling the returned function, so a component that subscribes
correctly cannot leak a listener, and the pattern is uniform enough that it is hard to get
wrong.

The core domain types are small and mirror the multiplexer's own model:

```ts
type Pane = {
  id: string;          // the multiplexer's own pane identifier
  windowId: string;
  sessionName: string;
  command: string;     // resolved process name — see chapter 3
  path: string;        // working directory
  active: boolean;
  width: number;
  height: number;
};

type Snapshot = { sessions: Session[]; windows: Window[]; panes: Pane[] };
```

A `Snapshot` is the whole tree in one object. The renderer polls for it on a short
interval and treats the result as the single source of truth for the sidebar.

## One shared cache, used as application state

The renderer uses a query cache (React Query) rather than hand-rolled state, and leans on
it harder than is typical. The snapshot query *is* the sidebar's state. Optimistic edits
are written directly into the cache:

```ts
// Killing a window: prune it from the cache immediately, then reconcile.
queryClient.setQueryData<Snapshot>(['snapshot'], (prev) =>
  prev && {
    ...prev,
    windows: prev.windows.filter((w) => w.id !== windowId),
    panes: prev.panes.filter((p) => p.windowId !== windowId),
  }
);
try {
  await api.killWindow(windowId);
} finally {
  await refetch();   // re-adds the window if the kill actually failed
}
```

The `finally` is the whole design. Optimism gives an instant UI; the unconditional
refetch means a failed operation self-corrects within one poll interval. There is no
rollback logic to write and no way for the cache to stay wrong.

Selection follows the same principle — it is derived rather than stored:

```ts
const selectedPane = snapshot?.panes.find((p) => p.id === selectedPaneId) ?? null;
```

When a pane dies, it vanishes from the next snapshot, `find` returns `undefined`, and the
viewer disappears on its own. No cleanup code, no dangling reference, no stale view of a
pane that no longer exists.

## Three things about startup

Most of the boot sequence is unremarkable. Three steps are not.

**A guard on the standard streams, installed first.** Without a listener, Node escalates a
broken-pipe error to a fatal uncaught exception. A routine log line written while nothing
was reading stdout was enough to kill the process.

**A power-save blocker.** Without it the OS suspends timers whenever the app is in the
background — which defeats the entire point of a tool that monitors work while you are
looking at something else.

**A prune of persisted state against a fresh snapshot.** Pane identifiers are ephemeral
and are recycled across multiplexer restarts, so every file keyed by one is swept against
the live set at boot. This recurs throughout the application and is worth carrying
forward: **any state keyed by a pane identifier is temporary and must be reconciled
against reality.**
