# cockpit — documentation

*Describes cockpit as of 2026 Q3. Documentation version 1.0.*

**Cockpit** is a desktop application that puts a single pane of glass over a machine's
terminal multiplexer sessions, where those sessions are running coding agents. It lists
every session, window and pane as a tree; streams any pane's live terminal; identifies
which panes are agents and what each one is currently doing; and turns an issue or a
third-party inbox item into a running agent in one click — worktree, branch, terminal
window, and task handover included.

It is a personal tool built by one developer for their own workflow. It is not a product,
has no users other than its author, and is not trying to acquire any.

## This repository is documentation, not code

The application itself is private, and stays private. What is here is a written account of
how it works — the architecture, the mechanisms, the decisions, and the things that turned
out to be wrong.

That split was deliberate. An audit found the codebase clean of credentials — but scanners
only find secrets, and not everything sensitive is a secret. Identifiers are harmless in
isolation and disclosive in combination, and no tool will tell you which combinations
matter. Certainty about what a repository discloses cannot come from tooling; it has to
come from a surface small enough that a person can read all of it. So this is that
surface.

Every word here was written for this repository rather than copied out of the private one.
The code samples are hand-authored illustrations of real mechanisms — accurate about how
something works, not extracts of how it is implemented. Chapters declare their own level
of abstraction and stay at it; where implementation detail would be the only honest
answer, the chapter says so instead of inventing one.

## The chapters

Read in order. Chapters 2–4 are the spine; if you read three, read those.

| # | Chapter | What it answers |
|---|---|---|
| 1 | [Overview](docs/01-overview.md) | What problem this solves, and for whom |
| 2 | [Architecture](docs/02-architecture.md) | Process split, the IPC contract, why there is no backend |
| 3 | [The multiplexer model](docs/03-tmux-model.md) | Why the app owns no terminals, and what follows from that |
| 4 | [Dispatch](docs/04-dispatch.md) | Issue → worktree → terminal → running agent, in one action |
| 5 | [Knowing what an agent is doing](docs/05-agent-detection.md) | Identity by inspection, activity by report |
| 6 | [Agent hooks](docs/06-agent-hooks.md) | Lifecycle events, and the two hardest bugs in the system |
| 7 | [Scheduled agent dispatch](docs/07-scheduled-tasks.md) | Cron that launches agents rather than doing work |
| 8 | [Integrations](docs/08-integrations.md) | Patterns that recur across every external system |
| 9 | [Telemetry from transcripts](docs/09-insights.md) | Measuring your own tooling with code, not vibes |
| 10 | [The design system](docs/10-design-system.md) | Tokens, two themes, sprites as data |
| 11 | [The security model](docs/11-security-model.md) | Invariants, and where the trust actually sits |
| 12 | [What worked, what didn't](docs/12-lessons.md) | The retrospective |

## If you are an agent

Start at [AGENTS.md](AGENTS.md).

## Asking a question

Open an issue using the question template. A human reads every issue before anything
happens to it; questions that get answered are answered by an agent, reviewed before
posting. Expect a delay, and expect some questions to be declined — anything needing the
private source is out of scope by construction.

## Licence

MIT. See [LICENSE](LICENSE). It covers the documentation in this repository; the
application it describes is not licensed to anyone.
