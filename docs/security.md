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
| Website → Roam Append API | One block on today's daily note, using a token the user pasted | At graph verification, and when a token is added in graph settings |
| Server → Roam Append API | Change log entries (date, event, roam.pub URLs) under a page's shortlink block | After publish/website changes, when the graph has a stored token and the page has a shortlink block |
| Extension → Roam | One shortlink block (`{server}/p/{id} {tag}`) per published page or block, created or edited in place | On publish, unless turned off in settings |
| Server → feed readers | Title, link, date, plain-text excerpt and byline of open, listed pages | Graph and collection feeds only when the owner turns them on; Discover's always |
| Extension → anywhere else | Nothing | — |

## Two different credentials, never mixed

- **Roam append-only token** (`roam-graph-token-…`): entered on the **website**. Proves control of a graph at
  verification, then is kept **AES-256-GCM encrypted** (`graph.append_token_enc`, key `APPEND_TOKEN_KEY`, never
  logged or sent back) for the change log. Can only add blocks; can't read, edit or delete. The owner removes it in
  graph settings or revokes it in Roam; a 401/403 from Roam marks it invalid and stops all writes until replaced.
  - What a leaked token (plus the key) allows: appending blocks to that one graph. Rotating `APPEND_TOKEN_KEY`
    makes stored tokens undecryptable (`decryptToken` returns null and nothing is written), so owners must add
    theirs again.
  - What the server writes: only dated entries under a block uid the extension reported as the page's shortlink
    block. Text comes from server-side events (titles, collection names, URLs, a moderator's removal reason), never
    from readers.
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

## The server never trusts the payload for identity

The graph, the person and their role all come from the key. The server re-hashes content, validates with zod, caps
size, and enforces ownership, removal and suspension on every call.
