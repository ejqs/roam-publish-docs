# Trust boundary

Three parties: **Roam** (the user's graph), the **extension** (runs inside Roam, in the user's browser), and the
**server** (roam.pub). This page is about what crosses between them. Each repo's own security notes:
the extension README's *Safety* section, and the server's moderation/auth code.

## What crosses where

| From → To | What | When |
| --- | --- | --- |
| Roam → extension | The selected tree, referenced block text, embedded trees, which blocks are collapsed | Only on a publish action |
| Extension → server | `PublishPayload` + hash + author, API key, the extension's version | Only on a user action |
| Server → extension | A seal plan: whether the page is encrypted, its id, and each place's password **public** key | Before each publish, from extension 0.2.0 |
| Extension → server | For an encrypted page, instead of the tree: the cipher, the content key sealed to each public key, the uids of collapsed blocks and a keyed hash | On publish, from extension 0.2.0 (coming soon) |
| Server → extension | Status, URL, hash, listing, `encrypted`, collapsed block uids, publication list, error strings, the oldest extension version it works with | In response |
| Website → Roam Append API | One block on today's daily note, using a token the user pasted | At graph verification, and when a token is added in graph settings |
| Server → Roam Append API | Change log entries (date, event, roam.pub URLs) under a page's shortlink block | After publish/website changes, when the graph has a stored token and the page has a shortlink block |
| Extension → server | Uids of the Changelog blocks it can and can't find (no text) | Every 5 minutes while Roam is open, when shortlink blocks are on |
| Extension → Roam | One shortlink block (`{server}/p/{id} {tag}`) per published page or block, created or edited in place | On publish, unless turned off in settings |
| Server → feed readers | Title, link, date, plain-text excerpt and byline of open, listed pages | Graph and collection feeds only when the owner turns them on; Discover's always |
| Extension → anywhere else | Nothing | — |

## Two different credentials, never mixed

- **Roam append-only token** (`roam-graph-token-…`): entered on the **website**. Proves control of a graph at
  verification, then is kept **AES-256-GCM encrypted** (`graph.append_token_enc`, key `APPEND_TOKEN_KEY`, never
  logged or sent back) for the change log. Can only add blocks; can't read, edit or delete. The owner removes it in
  graph settings or revokes it in Roam; a 401/403 from Roam marks it invalid and stops all writes until replaced.
  - What a leaked token (plus the key) allows: appending blocks to that one graph.
  - **Key leaked, database not:** the key alone decrypts nothing. Rotate it: set `APPEND_TOKEN_KEY_PREVIOUS` to the
    old key and `APPEND_TOKEN_KEY` to a new one, deploy, run `railway run bun run tokens:rotate`, then remove
    `APPEND_TOKEN_KEY_PREVIOUS` and deploy.
  - **Key and database leaked:** treat the tokens as exposed. Set a new key, deploy, run
    `railway run bun run tokens:revoke-all` (owners get a banner asking for a new token), and tell owners to revoke
    the old token in Roam.
  - A token the server can't decrypt (e.g. the key changed without `tokens:rotate`) marks the graph invalid, so its
    owner is asked for a new one rather than the log silently stopping.
  - What the server writes: only dated entries under a block uid the extension reported as the page's shortlink
    block. Text comes from server-side events (titles, collection names, URLs, a moderator's removal reason), never
    from readers.
- **Roam Publish API key** (`rp_…`): issued on the website, pasted into the **extension**. Lets the server act for
  that person on that one graph. Gives **no** access to the Roam graph. Stored hashed server-side; shown once.

## Encrypted pages

Encryption keeps a page's text from anyone who gets a copy of the database, and, when the page is encrypted in Roam
(v2), from roam.pub itself. The full rules and limits are on roam.pub at `/privacy/encryption` and
`/privacy/encryption/versions`; this is what each side holds.

| | Extension | Server | Reader's browser |
| --- | --- | --- | --- |
| Page text | Always (it's the graph) | v1 only, while encrypting it (publish, dashboard encrypt or decrypt) | After unlocking |
| Password | Never | Typed on the dashboard (set, change, encrypt, decrypt, add somewhere new) | Typed to unlock; only a proof derived from it is sent |
| Password's private key | Never | Only wrapped under scrypt(password) | Unwrapped after unlocking, kept non-extractable in IndexedDB for 30 days |
| Public keys | Per publish, from the seal plan | Stored | Not needed |
| Content key | Made fresh per publish, sealed, then dropped | Sealed copies only (v1: briefly in the clear while encrypting) | Unsealed to read |
| Hash key (`hash-key`) | In the graph's extension settings | Never; it stores only the keyed hash | Never |

- **v2 leaves the server out of the text entirely**, but readers still decrypt with JavaScript served by roam.pub,
  so whoever controls the running server could change that code to capture passwords as they're typed. Publishing
  from the extension doesn't have that weakness.
- **Titles stay readable**, and so do which blocks start collapsed (`folded` is plain uids), view counts and
  unlock counts.
- **The keyed hash** lets the extension tell whether an encrypted page changed without the stored hash confirming a
  guess at its text. The key lives in the graph's extension settings, so it shares the open question about whether
  collaborators can read those ([open-questions.md](open-questions.md)).
- **A forgotten password** can't be recovered by anyone; pages it opened show Needs republish until they're
  republished from Roam, which still has the text.

## Why setup doesn't go through the graph

An earlier design had the server write a claim code to the daily note and the extension read it to fetch a key.
Anyone who could read the graph (collaborators on a shared graph) could claim it first. Now verification finishes on
the website and the user pastes the key, so nothing in the graph is a secret. `POST /api/ext/claim` returns `410`
for old extension builds.

## RSS feeds

Feed readers fetch `feed.xml` without cookies or a session, so a feed can't check a password or membership. Feeds
therefore only ever list pages whose Visibility is Public or Discover (open to everyone and listed, in the graph
or in the collection), and a graph or collection feed `404`s unless its front page is open too. Unlisted, password-protected,
members-only and removed pages never appear. Once an item is in a feed, readers may keep a copy of its excerpt after
the page is unpublished; the extension's "Make listed" or "Make discoverable" is the step that can put a page there.

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
