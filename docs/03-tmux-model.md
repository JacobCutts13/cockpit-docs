# 3. The multiplexer model

This is the chapter that explains most of cockpit's other decisions.

## Why the multiplexer owns the terminals

The obvious way to build a terminal UI is to own the pseudo-terminals yourself: spawn a
PTY per session with a native binding, hold the file descriptors, render the bytes. Most
terminal applications do exactly that.

Cockpit does not own a single PTY. The multiplexer already running on the machine owns
them all, and cockpit attaches as an additional viewer. Four consequences follow, and
together they are worth far more than the control you give up:

**Agents survive the app.** Quit cockpit, crash it, rebuild it mid-task — nothing dies.
The processes belong to a server that was running long before the app started. For a tool
whose job is supervising long-running work, an architecture where the supervisor's crash
kills the work is not viable. Recovery is free for the same reason: there is no state to
rebuild, because none was ever held.

**Two viewers, one truth.** You can be attached from a regular terminal simultaneously.
Both see identical bytes. This is not a synchronisation feature that had to be built — it
falls out of not owning the terminal. The GUI never traps you.

**No native modules.** No PTY binding means no compiled dependency: no rebuild per
Electron version, no architecture-specific build matrix, no packaging archaeology.

**Agents can name their own location.** Each pane has a stable identifier, exposed to
processes inside it as an environment variable. An agent can read its own pane ID and
stamp it onto anything it reports — which is the only reason chapter 6's hook system is
possible. Without it, correlating "an agent said something" with "which pane" would be
guesswork.

What you give up is fine rendering control and anything the multiplexer cannot express.
That has cost approximately nothing.

## Shelling out, safely

Every interaction is a subprocess call with an argument array — never a shell string:

```ts
const exec = promisify(execFile);

async function mux(args: string[]): Promise<string> {
  try {
    const { stdout } = await exec('tmux', args, { maxBuffer: 64 * 1024 * 1024 });
    return stdout;
  } catch (err) {
    // Surface the tool's own stderr rather than the generic wrapper message.
    const detail = String(err?.stderr ?? '').trim();
    throw new Error(detail || String(err));
  }
}
```

Three things here earn their place. Using the argument-array form means no value is ever
parsed by a shell, so quoting and injection simply do not arise. The large output buffer
is because capturing a pane's scrollback can return several megabytes. And promoting the
subprocess's own stderr is the difference between a useful error and "Command failed with
exit code 1".

## Parsing structured output

The multiplexer's list commands accept a format string, which turns them into a
structured query interface. The field delimiter is the ASCII unit separator, `\x1f`:

```ts
const SEP = '\x1f';
const fmt = (fields: string[]) => fields.map((f) => `#{${f}}`).join(SEP);

const out = await mux(['list-panes', '-a', '-F',
  fmt(['pane_id', 'window_id', 'pane_current_path', 'pane_pid', 'pane_width'])]);

for (const line of out.split('\n').filter(Boolean)) {
  const [id, windowId, path, pid, width] = line.split(SEP);
  // ...
}
```

The delimiter choice is the point. Session names, window names, working directories and
pane titles are all user-controlled and can contain spaces, tabs, colons and pipes. A unit
separator cannot appear in any of them, so `split` is unambiguous without escaping or
quoting rules.

Every list call swallows its error and yields an empty result. A multiplexer server that
isn't running should render as "no sessions", not as an error dialog.

## Identifying what a pane is running

There is one genuinely non-obvious problem here, and it took a rewrite to get right.

The multiplexer exposes a field for a pane's foreground command. It reads the process
*title*, which a process may overwrite. Some tools do — one popular coding agent sets its
title to its own version number. So the field that should say `claude` says something like
a bare version string, and every heuristic built on it breaks the next time the tool
ships a release.

The first implementation worked around this specifically: read the agent's own session
files from disk and walk the process table upward from each recorded process ID until
reaching a pane's shell. It worked, and it was wrong in shape — it knew about one
particular agent, needed a hop-capped ancestor walk, and would need extending for every
future tool with the same habit.

The replacement is generic and shorter. One process-table read per snapshot, then a
three-tier fallback:

```ts
// One `ps -A -o pid=,ppid=,comm=` per snapshot.
// Prefer the first child of the pane's shell; fall back to the shell; then to
// whatever the multiplexer reported.
function resolveCommand(panePid: string): string {
  const child = firstChildOf.get(Number(panePid));
  return (child && commByPid.get(child))
      ?? commByPid.get(Number(panePid))
      ?? muxReported;
}
```

The line parser is deliberately a regex capturing two leading numeric fields and keeping
the rest intact, rather than a whitespace split — process names can themselves contain
spaces.

This resolves correctly for anything that overwrites its title, needs no per-tool
knowledge, and makes `pane.command === 'claude'` a dependable test everywhere else in the
application.

## Writing into a pane

Three tiers: **named keys** (`Enter`, `Escape`, `C-c`) for interface navigation;
**literal text**, flagged so it is never interpreted as key names; and **raw bytes**, the
keystroke path from the terminal view.

Raw bytes need exact fidelity — every keypress, escape sequence and paste arrives here.
Small payloads are hex-encoded, one byte per argument:

```ts
const bytes = [...Buffer.from(data, 'utf8')].map((b) => b.toString(16).padStart(2, '0'));
await mux(['send-keys', '-t', paneId, '-H', ...bytes]);
```

That guarantees fidelity and breaks at scale. Each byte costs roughly eleven bytes of
argument-vector space, so the operating system's argument limit caps this at about 90 KB
— which a large paste exceeds easily. Above a threshold, the payload spills to a temporary
file and is delivered through the multiplexer's own buffer mechanism instead.

### The bracketing trap

There are two paste paths and they make *opposite* decisions about bracketed paste. This
is the single easiest thing to get wrong.

Bracketed paste wraps content in markers so the receiving program knows it was pasted
rather than typed. Interactive programs use it to avoid executing a multi-line paste line
by line.

- **The keystroke path must not add the markers.** The terminal emulator in the renderer
  has *already* wrapped the content before it arrives. Adding them again double-wraps, and
  literal escape markers appear in the pasted text.
- **The programmatic path must add them.** Here cockpit is the originator and nobody else
  has wrapped anything. Without the markers, a multi-line command is submitted one line at
  a time.

Same underlying mechanism, opposite correct answer, decided by who wrapped it first.

The programmatic path also allocates a *named* buffer per call. The default buffer is
global to the multiplexer server, so two concurrent programmatic pastes will overwrite
each other between load and paste — and, memorably, deliver each other's content to the
wrong pane.

## Streaming a pane's output

Live output uses the multiplexer's pipe facility, which copies everything written to a
pane into a shell command. Cockpit points that at an append to a log file and tails it:

```
pane output ──▶ pipe ──▶ append to log file
                             │
                       drain loop (~16 ms) + filesystem watcher
                             │
                       refcounted subscribers ──▶ IPC ──▶ terminal view
```

One stream per pane, however many subscribers. The last subscriber to leave tears the
whole thing down: stop draining, stop watching, close the descriptor, stop the pipe,
unlink the log.

Draining reads positionally against a held descriptor, tracking its own offset, so
concurrent appends are safe. Both a filesystem watcher and a fixed interval trigger it —
the watcher for latency, the timer because watch events are coalesced and unreliable
depending on the filesystem.

Two subtleties are worth naming.

**The log file is the buffer.** There is no backpressure signal from the renderer. The
multiplexer writes regardless of whether anyone is keeping up, and the drain reads at
most a fixed chunk per iteration but loops until caught up. This is a deliberate trade:
an unbounded, OS-paged buffer that cannot stall the writer, bounded in practice by
unlinking the log when the last subscriber leaves.

**Subscription ordering avoids a gap.** The pipe starts *before* the scrollback snapshot
is captured, but the subscriber is registered *after*. Anything emitted during the capture
window accumulates in the log and is delivered on the first drain. Nothing is lost; the
cost is a small duplicate window where content appears in both the snapshot and the live
stream. For a redrawing interface this is invisible.

The snapshot needs one piece of surgery the live stream does not. Scrollback capture
returns one logical line per terminal row including trailing blanks; writing those
newlines would advance the cursor past the last row and scroll the content you just
captured off the screen. So trailing newlines are stripped and the remainder converted to
CRLF. Live bytes are raw terminal output that already carries CRLF and is written through
untouched.

## Resizing

The terminal view measures its container, emits a resize event, and cockpit forwards it to
the multiplexer with clamped bounds. The multiplexer reflows, the program repaints, and
the new bytes arrive through the stream that is already open.

Two details. The listener must be registered *before* the first fit, or the initial resize
away from the default 80×24 never fires. And the fit is triggered from four independent
places — an animation frame, a resize observer, a window listener, a delayed timeout —
because on first paint the container can briefly report zero dimensions. All are
idempotent; at least one always lands.

One consequence: this resizes the *window*, not the pane, so a second attached viewer gets
resized too. Cockpit does not own the terminal, and that cuts both ways.
