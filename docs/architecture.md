# Architecture

```
 Roam Research (browser)                         roam.pub (Railway)                    Readers
┌──────────────────────────┐   x-api-key: rp_…  ┌────────────────────────────┐        ┌──────────┐
│ roam-publish extension   │ ─────────────────▶ │ /api/ext/publications      │        │ browser  │
│  • serialize page/block  │   PublishPayload   │  • verify key → graph+role │        │          │
│  • inline ((refs))       │   + contentHash    │  • re-hash, compare        │        │          │
│  • sha256(stable JSON)   │ ◀───────────────── │  • upsert publication row  │        │          │
│  • cache {uid → hash,url}│   status, url      │                            │        │          │
└──────────────────────────┘                    │ Next.js website            │ ◀───── │ /{graph} │
                                                │  • onboarding (verify)     │        │ /c/{…}   │
 Roam Append API  ◀── verify + change log ───── │  • dashboard, keys, access │ ─────▶ │ /discover│
 (append-api.roamresearch.com)                  │  • public renderer, RSS    │        │ feed.xml │
                                                │                            │        └──────────┘
                                                │ Postgres (Drizzle)         │
                                                └────────────────────────────┘
```

The split is deliberate: **the extension is a thin, dumb sender; the server owns everything else.**

- The extension only knows how to turn a Roam page or block into a tree, hash it, and send it. It knows about
  `public` / `unlisted`, and nothing else about who can read a page.
- Access (open / password / members), collections, members, Discover, RSS feeds, bylines, moderation and every
  setting live on the website. None of it is exposed to the extension API.

This keeps the extension small, keeps the attack surface in Roam tiny, and means product changes ship
by deploying the server, not by waiting for users to update a Roam Depot extension.

## Flow 1: Connect a graph (website only)

1. Sign up on roam.pub (email + password, email must be verified).
2. On `/onboarding`, enter the graph name and a temporary **append-only** Roam API token. The browser sends its
   local date (`MM-DD-YYYY`) so the block lands on the user's today.
3. The server calls Roam's Append API and writes `roam.pub connected this graph (safe to delete)` to that daily
   note. A successful write proves control: Roam only gives tokens to a graph's admins and rejects a token used on
   another graph. The token is then **stored encrypted** for the change log (Flow 3). Owners of graphs verified
   before that can add one in graph settings.
4. First account to verify a graph owns it (`graph.userId`). Anyone else is told to ask the owner for an invite.
5. User goes to `/dashboard/keys`, generates their key (`rp_…`, shown once), and pastes it into the extension
   settings.

There is no step where the extension reads the graph to finish setup. An earlier design wrote a claim code on
the daily note that the extension picked up; it was retired because anyone who can read the graph (collaborators)
could claim the key first. `POST /api/ext/claim` now returns `410`.

## Flow 2: Publish

1. User right-clicks a block bullet or page title (or uses the command palette).
2. Extension pulls the tree with `roamAlphaAPI.data.async.pull`, sorts children by `:block/order`, inlines block
   refs (3 deep), attaches embeds (2 deep), and builds `{ rootUid, kind, title, tree }`.
3. `contentHash = sha256(stableStringify({ kind, title, tree }))`.
4. If the local cache already has that hash (and the same author name), it says "already published" without a
   network call.
5. Otherwise `POST /api/ext/publications` with the payload, the hash and the author name.
6. Server validates with zod, **recomputes the hash and rejects mismatches**, checks ownership, then creates or
   updates. New pages start `unlisted`, with access copied from the graph's current default, and join default
   collections the publisher belongs to. The server also derives the page's tags (`#tag`, `Tags::`) and search
   text from the tree; the extension sends nothing extra for them, so the hash is unaffected.
7. Server returns `created | updated | unchanged` with the URL. The extension caches it and copies the link. On
   `created`+`unlisted` the toast offers **Make public**.

## After publishing

Reading (rendering, access gates, slug redirects, view counting, RSS feeds, tags and search) and managing (listing, access, passwords,
collections, members, Discover, feeds, bulk changes) are website-only. Making a page **public** from the extension
also puts it in its graph's RSS feed when the owner turned that feed on and the page is open to everyone. The extension's only management calls are
**make public / make unlisted** (`PATCH`) and **unpublish** (`DELETE`). See
[where-to-look.md](where-to-look.md) for the server docs on those.

## Flow 3: Shortlinks and the change log

For the full read/send/write picture from a user's point of view, see
[Data flow and the Append API](data-and-append-api.md).

1. Before publishing, the extension asks `POST /api/ext/shortlinks` for the page's permanent `roam.pub/p/{id}` (8
   chars, keyed by graph + `rootUid`, so it survives unpublishing) and writes `{tag}` with two children, `{shortUrl}`
   and `Changelog`, as the first or last child of the page or block. It sends the `Changelog` block's uid as
   `anchorUid` with the publish.
2. Shortlink blocks and everything under them are left out of the tree before hashing, so they never count as
   content changes. The server drops them from the stored tree as well.
3. Whenever something happens to the page (publish, republish, visibility, Discover, collections, access, unpublish,
   moderation), the server appends a dated entry under the anchor with the graph's stored append-only token, after
   the response (`after()`), never failing the request.
4. `/p/{id}` shows the graph's owner and members where the page lives, with links to copy, and its history.
   Signed-out visitors are sent to log in; anyone else signed in gets a 404, never the page itself.
5. The extension's **Check change log** reads `GET /api/ext/changelog` (status and when Roam last accepted the
   token). Publish and sync responses carry the same status, and the extension warns once per session when Roam
   has rejected the token.

Roam's Append API writes to the daily note (under "Append API Captures attempted under non-existent blocks")
when the target block doesn't exist, and an append-only token can't read the graph to check. So the extension
confirms Changelog blocks every 5 minutes while Roam is open (`POST /api/ext/changelog/confirm`), the server only
writes to blocks confirmed in the last 10 minutes, and a block reported missing stops that page's change log and
shows on the dashboard until the page is republished (or the issue is ignored). Entries are queued and sent in the
background, batched per page, at most one Append API call per graph every 10s.

The Append API can only append (always last, no edit, move or delete), which is why the extension places the
anchor block and the server only ever appends under it.
