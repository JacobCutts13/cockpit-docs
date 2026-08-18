# 11. The security model

This chapter describes the model and its invariants rather than enumerating specific
privileged routes, writable paths, or handler names. A public map of the attack surface
of a personal application is not a useful thing to publish.

What follows is the honest shape of the trust model, including the parts that are
uncomfortable.

## What cockpit is not doing

**Cockpit does not sandbox the agents it runs.** It cannot. A coding agent's entire
purpose is to read and write files, run commands, and push branches. An agent confined
enough to be safe would be an agent that cannot work.

Stating that plainly matters, because the alternative framing — "we run agents securely" —
would be false, and would encourage exactly the wrong assumptions.

## The four invariants

**1. The renderer is inert.** It loads no remote content. Navigation away from the local
document is intercepted and refused. Script execution is restricted to same-origin by
content policy, with no inline or evaluated script permitted. Context isolation is on and
Node integration is off, so the interface has no ambient access to the system — everything
it can do is an explicitly declared channel.

Images are the deliberate exception: they may load over HTTPS from anywhere. Issue
and pull-request bodies render from raw markdown, and their embedded image URLs are
whatever the author wrote. The residual is a privacy leak — a tracking pixel in a pull
request reveals that you viewed it — accepted knowingly rather than overlooked.

**2. Untrusted markup is sanitised, and the pipeline is pinned by test.** Content authored
by arbitrary third parties is rendered through raw-HTML parsing followed by sanitisation,
in that order. There is exactly one place this pipeline is configured, extracted into its
own module specifically so that a component and its test cannot drift apart, and a test
reconstructs the renderer's internal plugin ordering and asserts that scripts, event
handler attributes, script-scheme links and frames are all removed.

**3. One gate reaches the operating system.** Anything that opens a URL externally passes
through a single function with a protocol allowlist — web and mail schemes only. This
matters because terminal output is untrusted: an agent can print anything, terminals
linkify what they print, and without the gate a printed link could invoke an arbitrary
scheme handler. The check runs on the *parsed* protocol and forwards the *normalised* URL,
so the validator and the sink cannot disagree about what the string means.

**4. Secrets stay in the main process.** The renderer cannot obtain a credential — no
channel returns one. Chapter 8 has the handling rules.

## Where the trust actually sits

Three grants of authority hold the system up. Each is deliberate.

**Cockpit trusts the repositories on your machine.** Worktree creation will run a setup
script from a repository if one is present. This is the same trust model as any
direnv-style or devcontainer hook, and it means cloning a hostile repository and then
creating a worktree for it executes its code as you. That is a developer-tool affordance,
not a defect — but it should be a decision you have made rather than one you discover.

**Cockpit trusts the agents it spawns with the credentials it hands them.** An agent that
needs to upload a file to a chat platform needs a token to do it. Containment is the scope
configuration on the credential itself, not anything cockpit enforces.

**Cockpit trusts whoever can reach its trigger surfaces.** Chat routing is by channel:
anyone who can post in a mapped channel can start an agent. Channel membership *is* the
authentication, so trigger channels are treated as a privileged surface and kept to ones
with closed membership.

## Untrusted text reaching an agent

This is the sharpest edge, so it gets stated directly.

Text from a chat message or an issue becomes the task an agent works on. That text is not
validated, not filtered, and not length-capped, because it is a task description and there
is no meaningful schema for one. An agent with filesystem and version-control access then
acts on it. For scheduled runs, that agent has permission prompts disabled.

Four things bound the damage, and they are the actual security model:

- **The reachable set is small.** Triggers are channels you control membership of, and
  issue dispatch is never automatic — a human reads and dispatches every one. There is no
  path from a stranger's text to a running agent that does not pass through a person.
- **Every run is isolated in a fresh worktree**, never a live checkout. This matters most
  where edits would otherwise propagate somewhere before review.
- **A draft pull request is the gate.** Nothing an agent produces reaches a main branch
  without a human merging it. The review is the safety mechanism; the skipped prompts are
  not.
- **Runs are time-bounded**, so a stuck or looping agent stops.

What is *not* bounded, and should be understood: a worktree isolates git state, not the
filesystem. An agent running without permission prompts can write outside its worktree.
The isolation is about not corrupting your working state, not about confinement.

## Development builds are not hardened

Unpackaged builds open a debugging port bound to the loopback interface. That port is full
control of the interface process, which means the entire declared channel surface. This is
intentional — it is how the application is driven for testing — and it is gated so that
packaged builds never do it.

The honest statement: **in a development build there is no trust boundary between cockpit
and any other process running as you.** That is fine on a personal machine and would not
be fine anywhere else.

## What secret scanning cannot do

Deciding what could be published produced one lesson worth closing on. What makes a
personal repository unsafe to publish is rarely credential-shaped. It is identifiers —
harmless alone, identifying in combination, sitting in ordinary prose. No scanner flags
those, because they are not secrets.

So tooling cannot certify a repository as safe to publish. It clears the shapes regexes
catch, so human review can spend its attention on the shapes they do not.
