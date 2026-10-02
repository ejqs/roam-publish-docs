# Trust boundary

Three parties: **Roam** (the user's graph), the **extension** (runs inside Roam, in the user's browser), and the
**server** (roam.pub). This page is about what crosses between them. Each repo's own security notes:
the extension README's *Safety* section, and the server's moderation/auth code.

## What crosses where

| From → To | What | When |
| --- | --- | --- |
| Roam → extension | The selected tree, referenced block text, embedded trees | Only on a publish action |
| Extension → server | `PublishPayload` + hash + author, API key | Only on a user action |
| Server → extension | Status, URL, hash, visibility, publication list, error strings | In response |
| Website → Roam Append API | One block on today's daily note, using a token the user pasted | Once, at graph verification |
| Server → feed readers | Title, link, date, plain-text excerpt and byline of open, listed pages | Graph and collection feeds only when the owner turns them on; Discover's always |
| Extension → anywhere else | Nothing | — |

## Two different credentials, never mixed

- **Roam append-only token** (`roam-graph-token-…`): entered on the **website**, used once to prove control of a
  graph, never stored. Can only add blocks; can't read the graph.
- **Roam Publish API key** (`rp_…`): issued on the website, pasted into the **extension**. Lets the server act for
  that person on that one graph. Gives **no** access to the Roam graph. Stored hashed server-side; shown once.

## Why setup doesn't go through the graph

An earlier design had the server write a claim code to the daily note and the extension read it to fetch a key.
Anyone who could read the graph (collaborators on a shared graph) could claim it first. Now verification finishes on
the website and the user pastes the key, so nothing in the graph is a secret. `POST /api/ext/claim` returns `410`
for old extension builds.

## RSS feeds

Feed readers fetch `feed.xml` without cookies or a session, so a feed can't check a password or membership. Feeds
therefore only ever list pages that are open to everyone and listed (public in the graph, or listed in the
collection), and a graph or collection feed `404`s unless its front page is open too. Unlisted, password-protected,
members-only and removed pages never appear. Once an item is in a feed, readers may keep a copy of its excerpt after
the page is unpublished; the extension's "Make public" is the step that can put a page there.

## Shared graphs

- First account to verify a graph owns it. Others join only by the owner's email invite, then get **their own**
  key. Removing a member revokes their key; their pages stay.
- Open question: whether Roam Depot extension settings (where the key lives) are per-person or shared in a
  multiplayer graph. See [open-questions.md](open-questions.md).

## Deleting graphs and accounts

Deleting can't be a way out of a moderation action:

- **A single graph** can't be deleted while it's suspended or has a removed page, just as a suspended collection
  or a removed page can't be. Deleting a graph deletes all its pages, members' pages included, and every key for it
  (`401 Invalid API key` from then on).
- **An account** can always be deleted. If a moderator had acted on it (ban, suspended graph or collection, removed
  page), a blocklist keeps what a fresh account would need to pick up where it left off:
  - **Graph names.** This is the strong anchor: connecting a graph takes the graph's own Roam token, so a new
    email doesn't get past it. Every graph the account owned is listed, not only the sanctioned one.
  - **The email**, as a sha256 hash of the address lowercased and without its `+tag`. A new address gets past it,
    so it only slows people down.
  - **Usernames**, current and former, so nobody can claim the name and impersonate the account.
  - **Suspended collections' slugs**, which stay reserved under `/c/`.

  Admins lift entries at `/admin/blocked`. Accounts deleted without any moderation action leave nothing behind
  except a log line.

## The server never trusts the payload for identity

The graph, the person and their role all come from the key. The server re-hashes content, validates with zod, caps
size, and enforces ownership, removal and suspension on every call.
