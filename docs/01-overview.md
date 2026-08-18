# 1. Overview

Cockpit is a desktop application that puts a single pane of glass over a machine's
running terminal multiplexer sessions, where those sessions are running coding agents.

That sentence is doing a lot of work, so here is the situation it comes from.

## The problem

Once you are running more than about three coding agents at once, the bottleneck stops
being the agents and starts being you. Each agent lives in its own terminal pane. Each
one is in a different repository, on a different branch, at a different stage — one is
waiting for you to approve a plan, one has finished and is sitting idle, one is halfway
through a refactor, one died forty minutes ago and you haven't noticed.

The terminal is an excellent place to *run* that work and a poor place to *survey* it.
Finding the right pane means remembering which window you put it in. Knowing whether an
agent needs you means switching to it and looking. There is no list. There is no status.
The information you need — who is blocked, who is done, who is burning tokens on
something you'd have stopped — is technically all present on your machine and
practically invisible.

Meanwhile the work that *creates* those agents is its own tax. Starting an agent on an
issue means: read the issue, pick the repo, create a branch, create a worktree so you
don't disturb whatever else is in flight, `cd` into it, start the agent, paste in the
issue text, and tell it what "done" means. Six or seven steps, every time, correctly, or
you get an agent working on the wrong branch in the wrong directory.

## What cockpit does

Cockpit addresses both halves.

**Survey.** The sidebar lists every session, window, and pane on the machine as a tree.
Panes running an agent are identified as such and carry a live status — working, needs
you, done, error. Clicking a pane shows its real terminal, streamed live, fully
interactive: you can type into it, paste into it, and it behaves like the terminal it is.
The point is that finding the right pane becomes a glance instead of a search.

**Dispatch.** Issue trackers, pull requests, and a third-party inbox appear in the same
sidebar. Selecting one and clicking dispatch performs the whole setup sequence: it
creates an isolated worktree, opens a terminal window in it, starts the agent, waits for
the agent's interface to actually finish booting, and hands it the task text. One click
replaces the seven steps, and — more importantly — replaces them *identically every
time*.

Around those two things sit the pieces that make a day's work continuous rather than
punctuated: scheduled tasks that dispatch agents on a recurring basis, a chat integration
that mirrors an agent's questions to your phone so you can unblock it from anywhere, and
a telemetry layer that reads the agents' own transcripts to tell you what your tooling is
actually costing and which parts of it are working.

## Who it is for

One person. This is the single most important thing to know before reading further.

Cockpit is a personal tool built by one developer for their own workflow, and every
design decision in it is downstream of that. There is no multi-user model, no permission
system, no onboarding, no configuration UI, and no attempt to be general. Where a product
would add a setting, cockpit hardcodes a choice. Where a product would add a confirmation
dialog, cockpit assumes you meant it.

That makes it a bad product and an unusually honest piece of documentation. Nothing here
has been generalised to look reusable. What you are reading is what one person actually
built, including the parts that only make sense for them.

## What it is not

**It is not a terminal.** It does not own any pseudo-terminals. The multiplexer already
running on the machine owns them; cockpit attaches as an additional viewer. Chapter 3
explains why this single decision shapes everything else.

**It is not a backend.** There is no server, no database, no HTTP API, no account. The
application reads the state of the machine it is running on and writes to it. Closing the
app does not stop a single agent.

**It is not an agent harness.** Cockpit does not decide what an agent does, how it
reasons, or when it stops. It creates the conditions for an agent to start and gives you
a way to watch it. The agent's own tooling owns everything after that.

The chapter map, and what the code samples in this repository are and are not, live in
[the README](../README.md).
