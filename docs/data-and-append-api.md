# Data flow and the Append API

What the extension reads from a graph, what it sends to roam.pub, what it writes back into the graph, and how
roam.pub uses Roam's Append API. Written for anyone checking what Roam Publish does with their data.

The short version is in the extension's README under
[Privacy and safety](https://github.com/ejqs/roam-publish#privacy-and-safety). Code is the source of truth: the
extension's `src/publish.ts`, `src/serialize.ts`, `src/index.ts`, and the server's `src/lib/changelog.ts`,
`src/lib/roam-append.ts` and `src/app/(app)/onboarding/actions.ts`.

## Two credentials

| | roam.pub API key | Roam append-only token |
| --- | --- | --- |
| Made by | roam.pub, at `/dashboard/keys` | Roam, in Settings → Graph → API tokens |
| Entered in | The extension's settings | The roam.pub website only |
| Stored | Roam's extension settings for the graph | roam.pub's database, AES-256-GCM encrypted |
| Lets the holder | Publish, update and unpublish on roam.pub for that graph | Add blocks to that one graph |
| Can't | Read or change anything in Roam | Read, edit, move or delete anything |

The extension never sees the Roam token, and the server never gets access to the graph beyond appending.

## What the extension reads

The extension only reads the graph through `roamAlphaAPI` in the browser, and only when one of these happens:

| When | What it reads |
| --- | --- |
| **Publish** or **status** on a page or block | The page or block and all its children: `uid`, text, heading, text alignment, view type (bullets, numbered, document), page title and order. |
| ...that contains a block reference `((uid))` | The referenced block's text (or page title), up to 3 levels of references deep, wherever it is in the graph. Refs inside code, embeds and `[label](((uid)))` aliases are left as written. |
| ...that contains an embed `{{embed: …}}` | The embedded block or page with its children, up to 2 embeds deep. Cycles stop. Every embed in a block (up to 20) is sent: the first as `embed`, the rest in order as `moreEmbeds`. |
| **Publish** with the Roam Publish block on | The page's direct children and grandchildren, to find an existing status link block. |
| Every 5 minutes while Roam is open | Whether each status link block it knows about still exists (a lookup by `uid`, no text). See [Confirming status link blocks](#confirming-status-link-blocks). |
| **Publish current page** from the palette | The open page or block, and its page, to publish the whole page when you're zoomed in. |

It doesn't read daily notes, other pages, attributes of other blocks, or anything else. Status link blocks and
everything under them are left out of the tree at any depth (`isShortlinkBlock`), so they're never published or
hashed. A block counts as one when its own text starts with one of the graph's status links, or when one of its
children's does and one of its children is a recorded status link (or older "Changelog") block. A status link pasted
under an ordinary block leaves out only the link, not the block it's under. The server applies the same rule
(`withoutShortlinks`).

## What the extension sends

Every request goes to the configured server (default `https://roam.pub`) with the API key in an `x-api-key` header.
The extension refuses a Server URL that isn't `https://` (except `http://localhost` for development), so the key is
never sent unencrypted, and gives up on a request after 60 seconds.
No analytics, no third parties.

| Request | When | Body |
| --- | --- | --- |
| `POST /api/ext/shortlinks` | First publish of a page, with the Roam Publish block on | `rootUid` |
| `POST /api/ext/publications` | **Publish** / **Republish** (the server answers `unchanged` when nothing changed) | `rootUid`, `kind`, `title`, the serialized `tree`, `contentHash` (SHA-256 of `kind`, `title`, `tree`), `author`, `anchorUid` (the status link block's uid, if any), the browser's `timeZone` |
| `PATCH /api/ext/publications/{rootUid}` | **Make public** / **Make unlisted** | `visibility` |
| `DELETE /api/ext/publications/{rootUid}` | **Unpublish**, after you confirm it | none |
| `GET /api/ext/publications` | **Sync**, **status**, or when the local cache is empty | none |
| `POST /api/ext/changelog/confirm` | Every 5 minutes while Roam is open (first run 20 s after load), in batches of 2,000 | `present` and `missing`: lists of `{ rootUid, anchorUid }` |
| `GET /api/ext/changelog` | Instead of the confirm call, while the graph has no token or the change log is paused | none |

The tree contains block text as it appears in Roam, with block references replaced by their text. Images, video,
audio and PDFs are sent as the URLs in the text; the files themselves aren't. The wire format is in
[`api-contract.md`](https://github.com/ejqs/roam-publish-web/blob/main/docs/api-contract.md).

## What the extension writes

| Where | What | When |
| --- | --- | --- |
| The published page, first or last child | The Roam Publish block: `{tag}` with `[{text}]({server}/p/{id})` under it | On publish: for pages when **Add Roam Publish block when publishing pages** is on, for blocks when **Add Roam Publish block when publishing blocks** is on. |
| The same blocks | Updated tag or link text | On the next publish after you change those settings |
| Roam's extension settings for the graph | API key, author name, Roam Publish block settings, and a cache of what's published (hash, URLs, title, visibility, status link uid) | When you change a setting or publish |
| Your clipboard | The published page's URL | On publish |

The extension writes nothing else to the graph. The change log entries under the status link are written by
roam.pub, not the extension.

## How roam.pub uses the Append API

Roam's Append API (`POST https://append-api.roamresearch.com/api/graph/{graph}/append-blocks`) adds blocks as the
last children of a block or page, using a token that can only append. roam.pub uses it for exactly two things.

### 1. Verifying a graph (once)

When someone connects a graph on the website, roam.pub appends one block to that day's daily note:

```
roam.pub connected this graph (safe to delete)
```

Roam only gives a graph's tokens to its admins and rejects a token on any other graph, so a successful write proves
the person controls the graph. The first account to verify a graph owns it on roam.pub. If **Keep a change log in
Roam** is off, the token is used for this write and not stored.

### 2. The change log

If the token is kept, roam.pub appends an entry under a page's status link block whenever something happens to
the page, grouped under one block per day:

```
[[Roam Publish]]
  [Roam Publish Status](https://roam.pub/p/k3Xq9aZt)
    [[October 2nd, 2026]]
      14:03 Published as unlisted: https://roam.pub/…
    [[October 3rd, 2026]]
      09:12 Made public
      09:40 Access in the graph: Password
```

Events: published, republished, byline changed, made public or unlisted, access or password changes, added to or
removed from a collection, listing or Discover changes, tags changed on the website, unpublished (from Roam or the website), and
removed or restored by a moderator. Each entry is the time in the graph's time zone and a short description,
sometimes with a link, under its date (a daily note link). No page content. Entries written before day blocks existed
carry the date on each line, and stay that way.

The owner chooses in the graph's settings, under **Roam change log**:

- **What to log in Roam**: publishing (published, republished, unpublished, byline), who can read (access,
  passwords, encryption), where it's listed (unlisted, public, Discover, shown or hidden in the graph), collections,
  and tags. All are on by default. Moderation is always logged.
- **Merge quick changes** (on by default): within one send, a setting changed several times is written once, with
  its final value, in the entry where it last changed. If that value is what Roam's change log already showed for
  the setting, nothing is written for it. Events such as "Password changed" or "Republished" collapse to the last
  one but are never dropped.
- **Group by day** (on by default): entries go under the day's `[[date]]` block, using the Append API's `nest-under`,
  which reuses the block with exactly that text or creates it. Off puts the date on every line instead.

Every event is also kept as the page's history on its status page (`/p/{id}`), which only the graph's owner and
members can see. That history exists whether or not anything is written to Roam.

#### When an entry goes to Roam

An entry is queued for Roam only if all of these hold when it happens:

- the page has a status link block that roam.pub knows about (`anchorUid`), not reported missing;
- the graph has a stored token that Roam hasn't rejected;
- the owner hasn't paused the change log;
- the owner hasn't left that kind of change out (moderation can't be left out).

Otherwise it's kept as history on the website only and never sent later.

#### Confirming status link blocks

The Append API can't check whether a block exists, and when the target block is missing it writes to the daily note
instead (under "Append API Captures attempted under non-existent blocks"). To avoid that, roam.pub only writes under
blocks the extension has recently confirmed:

1. Every 5 minutes while Roam is open, the extension looks up each status link block it knows about and sends their
   uids as `present` or `missing`. No text is sent.
2. `present` blocks are marked confirmed now. roam.pub writes only to blocks confirmed in the last 10 minutes.
3. `missing` blocks are forgotten, their queued entries dropped, and the page is listed on the dashboard. Publishing
   the page again writes a new status link block.

So the change log is only written while Roam is open somewhere with the extension running. Entries wait while it
isn't, and are dropped after 7 days.

#### Sending

A background worker sends queued entries:

- **Batched per page.** It waits until a page has been quiet for 30 seconds (at most 3 minutes), then sends all of
  its entries oldest first, each dated when the event happened, after merging them (above). That's one call, or one
  per day when grouping by day and the entries span midnight.
- **Rate-limited per graph.** At most one call per graph every 10 seconds. On a `429`, it backs off (honouring
  `Retry-After`, otherwise doubling from 1 minute up to 30).
- **Never twice.** Entries are claimed atomically before sending. If a call fails in a way that may have been applied,
  the entries are marked failed rather than retried. Identical consecutive entries are skipped.
- **Rejected token.** On `401` or `403`, the token is marked invalid, entries stay queued, the dashboard asks the
  owner for a new token, and the extension warns once per session.
- **`400`.** Most likely the status link block was deleted; roam.pub forgets it until the next publish.

### The token on the server

- Sent once from the browser on the website, over HTTPS. Never sent back to the browser or logged.
- Stored encrypted with AES-256-GCM using a server key (`APPEND_TOKEN_KEY`), which can be rotated.
- Decrypted only in the background worker, for the call above.
- Removed when the owner removes it in the graph's settings or deletes the graph. Pausing the change log keeps it.
  Revoking it in Roam makes the next call fail with `401`, which stops the change log.

## What the settings change

| Setting | Effect on reads, sends and writes |
| --- | --- |
| **Add Roam Publish block when publishing pages** off | The extension doesn't write a Roam Publish block on pages and, for pages, doesn't call `/api/ext/shortlinks`, sends no `anchorUid` and leaves them out of the 5-minute confirmation (which doesn't run at all when the blocks setting is off too). roam.pub still gives the page a status link on publish and keeps its history on the website, but writes nothing to Roam for new pages. Pages that already have a status link block from before keep it; their new entries wait for a confirmation that doesn't come, and are dropped after 7 days unless the setting is turned back on. |
| **Add Roam Publish block when publishing blocks** off (default) | Published blocks (as opposed to pages) get no Roam Publish block, so nothing is written under them. The two settings are independent. |
| Change log paused on the website | The extension skips the confirmation and only checks the status. Entries are kept as history only. The token stays stored. |
| No token stored | Same as paused. Verification was the only write. |
| **Server URL** | Every request above goes to this server instead of roam.pub. |
