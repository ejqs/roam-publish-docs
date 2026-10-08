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

- The extension only knows how to turn a Roam page or block into a tree, hash it, and send it (from 0.2.0,
  encrypted first when the server says the page is a Password page; see [Flow 4](#flow-4-encrypted-pages)). Of who
  can read a page it knows only where it's listed (`unlisted`, `listed`, `discover`) and whether it's encrypted.
- Each page's **Visibility** in each place it's shown (Discover, Public, Unlisted, Password, Members), passwords,
  collections, members, RSS feeds, bylines, moderation and every setting live on the website. The extension API
  exposes none of it, except adding a page to a collection, where the server applies the collection's own defaults.

This keeps the extension small, keeps the attack surface in Roam tiny, and means product changes ship
by deploying the server, not by waiting for users to update a Roam Depot extension.

## Extension 0.1.x today

Roam Depot serves extension 0.1.1 until 0.2.0 is pinned there, and roam.pub 0.18.0 keeps every 0.1.x path working
(`EXT_MIN_VERSION` is `0.0.0`, so nobody is asked to update). With 0.1.x:

- **Publishing** sends the plain tree. A Password page in a place that encrypts is encrypted by roam.pub on arrival
  (v1); nothing is encrypted in Roam, and no seal plan is asked for.
- **Collapsed blocks** aren't sent, so published pages start fully open, and there are no fold prompts.
- **No version header** is sent: `/admin/extension` counts these installs as "older than 0.2.0", and they never see
  the update notice. If a later website ever needs more than 0.1.x, they get an error message instead, which is why
  such a release waits until everyone active has updated ([shipping-changes.md](shipping-changes.md)).
- **Make listed** on a page that's only in collections is refused with a message saying each collection sets its
  own listing (website 0.15.0); 0.1.x still shows the button.
- **Add to collection** works for encrypted pages too, but 0.1.x doesn't republish afterwards, so the page shows
  Needs republish in that collection until it's republished from Roam.
- The toasts say Listed and Discoverable; the website says Public and Discover for the same thing.

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
   refs (3 deep), attaches embeds (2 deep), and builds `{ rootUid, kind, title, tree }`. From 0.2.0 it also marks
   blocks collapsed in Roam (`collapsed: true`), after asking the first time whether to publish them collapsed or
   expanded (see [Collapsed blocks](#collapsed-blocks)).
3. `contentHash = sha256(stableStringify({ kind, title, tree }))`.
4. From 0.2.0 (coming soon), the extension asks `GET /api/ext/publications/:rootUid/seal` whether the page is (or, new, will be)
   encrypted. If so, it encrypts the tree in Roam and sends the cipher instead ([Flow 4](#flow-4-encrypted-pages)).
5. `POST /api/ext/publications` with the payload, the hash and the author name. The extension doesn't skip this
   when its cache already has the hash: the cache can be stale, and the server answers `unchanged` itself.
6. Server validates with zod, **recomputes the hash and rejects mismatches** (it can't for an encrypted payload,
   whose hash is keyed), checks ownership, then creates or updates. New pages start **Unlisted** in the graph,
   with access copied from the graph's current default, and join default collections the publisher belongs to
   (leaving the graph when one of those collections takes its pages out of their graph). A new page is encrypted
   when every place it lands in is Password and its graph or a collection asks to encrypt new Password pages. The
   server also derives the page's tags (`#tag`, `Tags::`), search text and collapsed blocks (`folded`) from the
   tree; the extension sends nothing extra for them, so the hash is unaffected.
7. Server returns `created | updated | unchanged` with the URL and `encrypted`. The extension caches it and copies
   the link. On `created`+`unlisted` the toast offers **Make listed**, and **Make discoverable** when the server's
   `discoverBlocked` is null.

## After publishing

Reading (rendering, access gates, unlocking and decrypting, slug redirects, view counting, RSS feeds, tags and
search) and managing (Visibility, passwords, encryption, collections, members, Discover, feeds, front pages, bulk
changes) are website-only.

### Visibility, and the words each side uses

Since website 0.17.0, every place a page is shown (its graph, each collection) has one **Visibility**, from most
open to most closed: **Discover**, **Public**, **Unlisted**, **Password**, **Members**. Underneath, each place still
stores two columns, `access` (open, password, members) and `listing` (unlisted, listed, discover), and the
extension API speaks those:

| Website (Visibility) | `access` | `listing` | Extension 0.1.x / 0.2.0 calls it |
| --- | --- | --- | --- |
| Discover | open | discover | Discoverable |
| Public (was "Listed") | open | listed | Listed |
| Unlisted | open | unlisted | Unlisted |
| Password | password | `listed` shows its title on the front page, `unlisted` doesn't | no Make buttons; `encrypted` says if it's encrypted |
| Members | members | the same | no Make buttons |

The extension still says Listed and Discoverable; moving its toasts to the ladder's words is a later extension
release. Display settings (bylines, view counts, breadcrumbs, the RSS feed, where new pages go) moved from the
Sharing tab to the graph's Settings tab in 0.17.0, which changes nothing on the wire.

`[[links]]` to other pages are resolved by the server when it renders, never by the extension (which sends them
as written): a link becomes clickable only when the page it names is published in the same graph or collection and
listed there, which means **Public** or **Discover**, or Password and Members pages that show their title on the
front page. Links to **Unlisted** pages show as plain text, since a link would hand their address to every reader
(website 0.16.2; unlisted-to-unlisted links were tried and reverted).

### What the extension can change

Making a page **listed** (or **discoverable**) from the extension also puts it in its graph's RSS feed when the
owner turned that feed on and the page is open to everyone. The extension's only management calls are:

- **Make listed / make discoverable / make unlisted** (`PATCH` with `listing`). Every response carries `listing`
  and `discoverBlocked`, the reason it can't be Discoverable, so the extension never works out Discover rules
  itself. A page shown only in collections has no graph listing to change: the server answers `409` (website
  0.15.0), and extension 0.2.0 stops offering the buttons for it.
- **Add to collection** (`GET`/`POST /api/ext/publications/:rootUid/collections`). The server lists the holder's
  collections with how a page starts out in each, adds it with that collection's defaults, and takes the page out
  of its graph when the collection asks for that or is locked while the graph place isn't, so the graph link can't
  get around the lock. An encrypted page goes in as Password; the list marks collections it can't go in
  (`blocked`: no password, or one too old or short to encrypt), and the response says `needsRepublish`, which
  extension 0.2.0 answers by republishing at once so the page opens there (website 0.16.0).
- **Unpublish** (`DELETE`).

See [where-to-look.md](where-to-look.md) for the server docs on those.

### Collapsed blocks

Published pages fold like Roam (website 0.12.0 to 0.16.1): readers collapse blocks from the caret or the thread
lines, zoom into any block from its bullet or number (the block's uid goes in the address), and get a heading
outline. Code blocks get a language label and copy button, and Mermaid blocks are drawn (0.14.0). All of that is
rendering on the website; the only part that crosses the boundary is which blocks **start** collapsed:

- Extension 0.2.0 (coming soon) sends `collapsed: true` on blocks with children that are collapsed in Roam (omitted otherwise,
  so older trees hash the same). The server stores the uids as `publication.folded` and returns them as `folded`
  in the publication list.
- Because `collapsed` is in the hash, folding a block in Roam makes the page differ. The extension compares the
  cache's `folded` with Roam's: when only folds changed it offers **Sync open/collapsed blocks**, otherwise
  **Republish as is** or **Republish, keep open/collapsed** (re-applies the published folds to the new tree before
  hashing). `folded` comes from the server, so this works from any computer.
- Older extensions never send `collapsed`, so their pages start fully open.

### View counts

Page footers show a view count ("1.4k views", with the top reader countries as flags). It comes from two sources
the server owns. The extension sends nothing for it.

- **Umami**: the operator's Umami Cloud site. Background jobs read it, never the page request:
  - a daily full sweep reads every path's all-time views;
  - an hourly sweep adds recent views to pages being read, with bigger counts updated less often;
  - a budgeted country lookup runs one call per page.

  Results land in `page_views`.
- **Signed-in Roam readers**: the existing first-party `publication_view` rows.

Graphs and collections choose show, managers only, or off, and each page can override that. Unlisted pages show
their count only when the page itself is set to show. Password-protected pages count successful password entries
instead of Umami visits, and warn their managers when the current password has been entered often. Members-only
pages have no count. Job status is at `/admin/jobs`.

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
   moderation), the server queues an entry, after the response (`after()`), never failing the request. The
   background sender appends it under the anchor with the graph's stored append-only token, under the day's
   `[[date]]` block (`nest-under`), after merging quick changes to one setting. The owner picks which kinds of change
   go to Roam, and can turn merging and day blocks off.
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

## Flow 4: Encrypted pages

A page can be encrypted when **every** place it's shown is Password, with a password of at least 10 characters
there. Encryption is turned on per page (Encrypt with password in Manage), for new pages (Encrypt new password
pages on a graph or collection), or for pages already there (Encrypt existing pages). A Password page that isn't
encrypted is still only gated by the password check. The envelope is the same in both versions: the tree is
encrypted with AES-256-GCM under a fresh content key, bound to the page's id (`tree:{publicationId}`), and the
content key is sealed to each place's password with X25519 + HKDF-SHA256. Each password's X25519 private key is
stored wrapped under scrypt(password), so publishing only ever needs public keys.

Who does the encrypting is what the **encryption version** says, shown on each encrypted page's badge and
explained at `/privacy/encryption/versions`:

| | v1 · Encrypted | v2 · End-to-end encrypted |
| --- | --- | --- |
| Since | website 0.3.0 | website 0.18.0 + extension 0.2.0 (**coming soon**: 0.2.0 isn't in Roam Depot yet) |
| Who encrypts | roam.pub, on arrival (`sealNewContent`) | The extension, in Roam (`src/seal.ts`) |
| roam.pub sees the text | When publishing, and when you encrypt or decrypt on the dashboard | Never |
| Stored `contentHash` | `sealHash` of the plain hash, made by the server | `k1.` + HMAC of the plain hash under a key in the graph's extension settings |
| Made by | Extensions before 0.2.0, Roam without X25519, the dashboard's encrypt switches | Publishing or republishing a Password-everywhere page from 0.2.0 |

Either way, **readers' browsers decrypt** (website 0.16.3): the server sends the page still encrypted, unlocking
sends a proof derived from the password instead of the password, and the unwrapped key stays in the browser
(IndexedDB, 30 days). The extension has no part in reading.

The v2 publish, from extension 0.2.0:

1. `GET /api/ext/publications/:rootUid/seal` → `{ encrypt: false }`, or `{ encrypt: true, publicationId, locks }`:
   the page's id (a fresh one for a new page) and every place's password as `{ scope, id, publicKey }`. A server
   without the route answers `404`, and the extension publishes as before.
2. If it says encrypt and Roam's WebCrypto has X25519 (`canSeal`), the extension encrypts the tree and seals the
   content key to every lock with a public key. Otherwise it sends the plain tree and the server makes it v1.
3. `POST /api/ext/publications` with `sealed: { publicationId, cipher, keys }`, `folded`, and a keyed `contentHash`
   instead of `tree` and the plain hash. Shortlink blocks were left out in Roam.
4. The server checks the seal against a fresh plan. If a place or password changed in between, it answers `409`
   with `reseal: true`, and the extension asks for a new plan and tries once more.
5. The page is stored as v2 with `encryptedBy: "extension {version}"`, no search text or tags. A lock without a
   key pair, or one whose password was reset, leaves it `needsRepublish`.

A v1 page becomes v2 the next time it's republished from extension 0.2.0. Website 1.0.0 will stop accepting plain
trees for encrypted pages, announced at `/updates/upcoming` ([shipping-changes.md](shipping-changes.md)).
