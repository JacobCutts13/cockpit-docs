# 6. Agent hooks

This is how the application knows what an agent is doing, and it contains the two most
interesting problems in the codebase.

## The transport

The agent CLI supports hooks: shell commands it runs at defined lifecycle points, handed
a JSON payload on standard input. Cockpit installs one script that handles every hook
type. The script does almost nothing — it stamps in the pane identifier from its own
environment and writes the payload to a file in a drop directory:

```
agent lifecycle event
      │
      ▼
  hook script  ──▶  writes <drop-dir>/<event>.json   (temp file, then rename)
                            │
                            ▼
                    filesystem watcher in the main process
                            │
                            ▼
                    parse ─▶ claim ─▶ dispatch by event type
```

A file drop rather than a socket, for three reasons: the hook script stays trivial and has
no dependency on cockpit running; events survive the app being restarted mid-turn; and
the whole thing is inspectable with `ls` when it misbehaves.

The write is temp-then-rename because the watcher fires on file creation and would
otherwise read a half-written file.

At startup the drop directory is swept clean rather than replayed. A turn that ended
yesterday is not news today, and replaying a backlog on launch would produce a burst of
stale notifications.

## Two instances, one directory

Several copies of the application can run simultaneously (chapter 2), and they all watch
the same drop directory. Without coordination each would process every event and post
duplicate messages.

The fix is an atomic claim: rename the file to include the claiming process ID before
reading it. Rename is atomic, so exactly one process wins, and the loser gets a
"no such file" error it can ignore.

```ts
function claimEventFile(src: string, pid: number): string | null {
  const dest = `${src}.${pid}.taken`;
  try {
    renameSync(src, dest);      // atomic: exactly one caller wins
    return dest;
  } catch {
    return null;                // somebody else got there first
  }
}
```

This turned out to solve a second problem for free. Filesystem watchers on some platforms
fire duplicate events for a single rename, so even a *single* instance would double-process
without it. One mechanism, two races.

Everything read out of these files is validated before use. The pane identifier is
regex-checked against the multiplexer's format; the event name is narrowed to a known
union; a parse failure returns a discriminated result carrying a specific reason and a
truncated preview of the raw content, so a malformed event produces a diagnosable log line
rather than a silent skip.

## Problem one: telling a human keystroke from an injection

This is the subtle one.

Cockpit mirrors an agent's conversation to a chat thread, so you can follow a run from
your phone. Human-typed prompts are mirrored too — you want to see what you asked.

But cockpit *also* types into panes. Chat replies get delivered by injection. Dispatch
pastes commands. Answers to interactive questions are driven as keystrokes. Every one of
those fires the same "user submitted a prompt" hook as a real keystroke, and the hook
payload cannot distinguish them — from the agent's perspective they are identical.

Mirror them all and you get: every chat reply echoed back into the chat thread it came
from, and every dispatch command posted as though you had typed it.

The fix is to have the writer declare its own writes. Anything cockpit injects records
what it sent; the hook handler consumes a matching record and suppresses that mirror:

```ts
// Written by every cockpit-originated injection, before sending.
recordInjectedPrompt(paneId, text);

// Read by the hook handler when a prompt-submitted event arrives.
if (consumeInjectedPrompt(paneId, event.prompt)) return;   // ours — don't mirror
await mirrorToChat(paneId, event.prompt);                  // human — mirror it
```

Three properties make it correct rather than merely working:

**Consume-once.** A matched record is deleted. If you genuinely type the same thing twice,
the second one mirrors.

**A short expiry.** Some injections never produce a matching event at all — the line that
launches the agent, for instance, is consumed by the shell. Without expiry that record
would sit there forever and suppress an identical human prompt an hour later. A window of
tens of seconds covers the real gap, which is sub-second in practice.

**Human input records nothing.** Keystrokes from the terminal view go through a different
function that writes no record, so they always mirror. The default is "mirror"; suppression
is the exception that has to be actively claimed.

The general pattern is worth extracting: **when you cannot distinguish your own writes
from a user's at the observation point, make the writer declare them.** Trying to infer it
downstream — by timing, by content heuristics — does not work.

## Problem two: answering a multiple-choice question remotely

When an agent asks a structured question, the notification carries the questions and their
options, and cockpit renders them as interactive controls in the chat thread. Tapping
options on a phone and hitting submit drives the agent's terminal interface.

The obvious implementation — send keystrokes as each option is tapped — was built, shipped,
and was wrong.

The agent's picker is a stateful tabbed form: several questions, a submit tab, navigation
between them. Cockpit cannot see which tab it is currently on. Replaying keys per tap
desynchronises the moment anything arrives out of order — and taps *do* arrive out of
order, because people answer question two before question one. The observed failure was
answering the wrong question entirely, plus stray characters leaking into the prompt.

Worse, the first implementation drove the picker with digit shortcuts, which works until
an option carries a preview. Then the picker renders a preview pane and **ignores digit
keys entirely**. The keys were accepted and discarded, the form never advanced, and
nothing errored.

The working design is submit-once:

- Tapping a control changes *nothing* in the terminal. It records a selection in memory.
  The card is never re-rendered; the platform's interactive controls hold their visual
  state client-side.
- Submit validates that every question has an answer — a gap posts a nudge visible only to the
  person who submitted, and leaves the card up — then converts the whole selection set into one ordered burst of
  cursor keys.
- Only cursor navigation and confirm keys are used. No digits, since they are unreliable.
- Keys are sent with a small gap between them, because the picker has to repaint before
  the next one lands.
- Per-pane operations are serialised through a promise chain, so two rapid taps cannot
  interleave a read-modify-write on the selection state.

Ordering at the end is load-bearing: the keystroke burst must be *awaited* before the card
is marked resolved. Resolve first and the user sees a confirmation for keystrokes that
have not been delivered.

The card also resolves itself if you answer in the terminal instead — the agent's
forward-progress signal is what tells cockpit the question is no longer open. Answering in
either place always closes both.

Both problems have the same shape: cockpit is a second actor operating a system designed
for one. The answers rhyme too — declare your own writes rather than inferring them
downstream, and batch into one atomic interaction rather than streaming increments into a
state machine you cannot see.
