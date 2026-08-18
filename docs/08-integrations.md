# 8. Integrations

Cockpit reads from several external systems. This chapter is about the patterns that
recur across them rather than any one API.

## Secrets

One plaintext JSON file in a config directory, read fresh on every access, no caching.
Every field is optional.

Three behaviours make it pleasant rather than brittle:

- **Missing file returns an empty object.** A fresh install has no credentials and every
  consumer degrades through its own missing-key path.
- **Malformed content throws.** A typo is loud; an absent file is quiet.
- **Every missing-key error names the fix.** Not "authentication failed" but the key name
  and the file to put it in. These messages are read months apart, by which point the
  shape of the config file is not memorable.

No secret is ever returned over IPC, and none is logged. The renderer cannot obtain one;
error messages name keys, never values.

One integration deliberately has no secret at all: the code-forge integration shells out
to the vendor's own CLI and inherits whatever credential that tool already holds. Not
storing a token is strictly better than storing one well.

## Shelling out to a vendor CLI

Where a good first-party CLI exists, using it beats reimplementing its API client. It
already handles auth, pagination, enterprise hosts and rate limits.

The cost is that every argument reaching it must be validated, because arguments that
begin with a dash are flags:

```ts
const REPO = /^[\w.-]+\/[\w.-]+$/;

function assertRepo(repo: string): void {
  if (!REPO.test(repo)) throw new Error(`Invalid repository: ${repo}`);
}
```

Every public function validates before invoking. Combined with argument-array invocation
(no shell), that closes both injection and flag-smuggling.

Errors are normalised in one place: a missing binary becomes an install instruction, and
anything else propagates the tool's own stderr, which is invariably more useful than a
generic wrapper.

## Client-side unions

A recurring shape: you want "issues assigned to me *or* filed by me", and the search API
ANDs its filters and rejects the disjunction as malformed.

So the union is assembled client-side — two concurrent queries, merged and deduplicated by
a composite key:

```ts
// Numbers are repository-local: repo-a#12 and repo-b#12 are different issues.
const key = (i: Issue) => `${i.repository}#${i.number}`;
```

Two details are load-bearing. Both queries are awaited together, so one failing rejects
the whole fetch — a half-populated list that *looks* complete is worse than the error the
UI already knows how to show. And sort ties break on the key, so ordering never depends on
which request happened to return first.

The same idea appears in the pull-request list, but inverted: one GraphQL request with
three aliased searches, each pulling check status and review-thread counts inline, so the
list never fans out into a per-item fetch.

## Paging, and a union trap

One API caps a nested collection at a fixed page size, and the naive read took the first
page only — so the visible history was silently truncated at an arbitrary point. The fix is
ordinary (page until the cursor is exhausted, with a high backstop to bound a runaway
rather than to trim real data), but two details are worth noting. The first page arrives
with the parent object, so subsequent pages use a narrower query that fetches only the
page. And because these connections return newest-first, everything is sorted once at the
end rather than assumed to arrive in order.

A related trap in the same API: several members of a returned union expose a same-named
field with *different nullability*, so an unaliased shared selection will not compile
server-side. Alias each member's field to a distinct response key and flatten in
application code.

The discipline that goes with it: entries carrying no content map to `null` and are
filtered, while an entry with no text but with an attachment is *kept*. An image with no
caption is still a message.

## Brokering attachments

An inbound file from a chat platform needs a bearer token to download. An agent should
never see that token.

So the main process downloads the bytes, writes them to a local directory, and types the
resulting **path** into the pane. The agent opens a local file with its ordinary file-read
tool. The credential never leaves the main process on that path.

One trap is worth recording because it is invisible: an under-scoped token does not return
401. It returns 200 with an HTML sign-in page. Without a content-type check you write that
HTML to disk as `screenshot.png` and hand the agent a file that is not an image, with no
error anywhere. The check is a one-liner; finding out you needed it is not.

Each file resolves independently, so one failure does not abort the others, and failures
are reported inline in the message the agent receives rather than silently dropped.

## Routing without addressing

Inbound chat messages are routed **by channel, not by mention**. A top-level post in a
mapped channel is the trigger; no addressing required. A message in a thread routes to
whichever pane owns that thread, regardless of channel, so replying is the natural way to
continue a conversation without triggering a new one.

The authorisation model follows directly and should be stated plainly: **anyone who can
post in the channel can start an agent.** Channel membership is the authentication.
Chapter 11 returns to this.

Two mechanics make it survivable. The cheapest possible filter runs before anything else —
messages from the bot itself are discarded before even reading credentials, since a busy
status feed is mostly the bot's own posts. And every dispatch is idempotent on the
triggering message, so a redelivered event maps to the existing pane rather than spawning
a second agent.

## One-directional imports

A small rule with outsized effect: the chat service imports the handlers, and no handler
imports the service. Callers inject the posting function.

Without it, every handler needs the service and the service needs every handler, and the
resulting cycle makes all of it untestable. With it, handlers are pure enough to unit-test
by passing a stub, which is how the ordering guarantees in chapters 4 and 6 are verified
without a live terminal.
