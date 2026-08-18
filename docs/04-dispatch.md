# 4. Dispatch

Dispatch is the feature the rest of the application exists to support: turn a piece of
work into a running agent, correctly, in one action.

## The shape

Six entry points — an issue from either of two trackers, an item from a third-party inbox, a
freeform chat message, and two flavours of pull-request agent — all reduce to one
orchestrator. They differ in exactly three ways: how repositories are resolved, how each
worktree gets onto its branch, and which command is handed to the agent.

```ts
type DispatchPlan = {
  containerDirName: string;
  repos: Array<{ clonePath: string; shortName: string }>;
  prepareWorktree: (clonePath: string, worktreePath: string) => Promise<PrepareResult>;
  buildCommand: (worktreePaths: string[], branches: string[]) => string;
  spawnInWorktree?: boolean;
  onDispatched?: () => Promise<void>;
};
```

Everything variable is a field or a function. The orchestrator itself has no conditionals
about where the work came from. Adding a seventh source means writing an adapter, not
editing the core.

## The sequence

Every dispatch creates a *container* directory, with one worktree per repository inside
it. That layout is what makes multi-repository work uniform — a task spanning two
repositories is the same shape as one spanning a single repository, just with two
subdirectories.

```
<projects-root>/<container-name>/
  ├── service-a/     ← worktree, on the task branch
  └── service-b/     ← worktree, on the same task branch
```

The steps, in order:

1. **Resolve the container conflict.** If the directory already exists, do nothing and
   return a typed "already exists" result. The UI then offers remove, reuse, or cancel.
   Nothing is ever silently destroyed.
2. **Create each worktree.** Sequentially, recording each success. If one fails, roll back
   every worktree created so far and return the failure.
3. **Exclude the agent's scratch files.** The task and goal files an agent writes are
   added to the worktree's per-checkout git exclude list, so they never show as changes
   and never make the worktree look dirty to the cleanup flow.
4. **Build the command and copy it to the clipboard.** Belt and braces — if the automated
   paste fails for any reason, the human can still paste it manually.
5. **Spawn a terminal window** in the container (or inside the single worktree, for
   pull-request flows where the tooling infers the PR from the branch).
6. **Start the agent and hand over the task**, detached — see below.

Steps 1 to 5 are awaited, so the UI can select the new pane the moment it exists. Step 6
is fire-and-forget, because it involves waiting.

## The handover problem

You cannot start an agent CLI and immediately paste a task into it. Its interface takes
several seconds to initialise, and anything sent before it is listening is silently
discarded.

There is no ready signal, so cockpit polls for one:

```ts
const WELCOME_MARKERS = ['Welcome back', 'Welcome to', ' v'];  // illustrative

async function waitForReady(paneId: string, timeoutMs = 20_000): Promise<boolean> {
  const deadline = Date.now() + timeoutMs;
  while (Date.now() < deadline) {
    await sleep(500);                                  // sleep first: a new pane is empty
    let buf: string;
    try {
      buf = await capturePane(paneId, { lines: 200, withEscapes: false });
    } catch {
      return false;                                    // pane died
    }
    if (WELCOME_MARKERS.some((m) => buf.includes(m))) return true;
  }
  return false;
}
```

Two details matter. It sleeps before the first capture, because a freshly spawned pane has
nothing in it yet. And it strips escape sequences before matching, because colour codes
will otherwise split a marker string in half and the match silently never fires.

Then — and this is the part that is only learnable by getting it wrong — **the banner
appearing is not the same as the input loop being ready.** There is a further settling
window of a few hundred milliseconds after the welcome text renders. Paste inside it and
the content is accepted and discarded, which looks exactly like the paste never happening.

So the handover is: wait for ready, rename the session to match the container, settle,
then paste.

All of this Claude-specific knowledge lives in one small module, by explicit rule. If
cockpit ever dispatches to a different agent CLI, that gets its own module alongside —
the file is not generalised into a plugin interface for a second implementation that does
not exist yet.

## Who writes the task file

This is the design decision that changed, and it is more interesting than where it landed.

Originally cockpit wrote the task file itself, parsed a "definition of done" section out
of the issue body, and injected a goal command into the pane. That put cockpit in the
business of understanding issue formats — every change to how tasks were specified meant
changing the desktop application.

Now cockpit pastes a single command and stops. The agent's own skill does the rest: it
interviews the human, writes its own task and goal files into the worktree, and injects
its own goal command into its own pane using the pane identifier it can read from its
environment.

The division that emerged is worth stating as a principle:

> **Cockpit owns orchestration. The agent's tooling owns every word the agent reads.**

Cockpit creates conditions — a worktree, a branch, a pane, a running process. It does not
compose prompts beyond handing over the source text, and it does not define what "done"
means. Changing the working contract is now editing a skill file, not shipping a desktop
release.

The one residual involvement is step 3 above: cockpit still pre-excludes the filenames the
skill will write, because it is the thing that made the worktree.

## Failure handling

The rule is a clean split by phase.

**Before anything is created**, failures are typed values, not exceptions:

```ts
type DispatchResult =
  | { ok: true; containerPath: string; paneId: string; branch: string }
  | { ok: false; reason: 'container-exists' | 'no-local-clone' | 'lookup-failed' };
```

Every repository is validated before the first worktree is created, so a partially-set-up
multi-repository dispatch is not reachable.

**After the pane exists**, failures are logged and swallowed. The pane is real, the
command is on the clipboard, and a human can finish the job in two seconds. Tearing down
a working terminal because an automated paste failed would be worse than the failure.

The one exception is the chat-triggered path, where no human is watching. That posts an
explicit failure notice back into the thread asking for a retry, because the normal
reporting channel is the agent itself, and the agent is what failed to start.
