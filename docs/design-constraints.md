# Shared invariants

Rules that only hold if **both** repos keep them. Breaking one usually breaks installed extensions silently.
Rules internal to one repo (e.g. how access defaults or Discover work) live in that repo's docs.

## Hashing

1. **Both sides compute the hash and it must match byte for byte.**
   `contentHash = hex(sha256(stableStringify({ kind, title, tree })))`. The extension uses it to skip no-op
   publishes; the server recomputes it and rejects mismatches (`400`). The one exception is a page encrypted in
   Roam (extension 0.2.0+): it sends `k1.` + HMAC-SHA256 of that hash under a per-graph key (`hash-key` in the
   extension's settings, shared by every device publishing the graph), which the server stores and compares but
   can't recompute or check against a guess at the text. See invariants 24 to 27.
2. **`stable-stringify.ts` exists as identical copies** —
   `roam-publish/src/stable-stringify.ts` and `roam-publish-web/src/lib/stable-stringify.ts`. Keys sorted
   recursively, no whitespace, `undefined` dropped. Never edit one without the other.
3. **The `Node` type exists twice** — `roam-publish/src/serialize.ts` and `roam-publish-web/src/db/app-schema.ts`
   (plus the zod `NodeSchema` in `src/app/api/ext/publications/route.ts`). The server's type is a superset: it also
   accepts the default values (`"bullet"`, `"left"`) the extension never sends.
4. **Optional fields are omitted at their default** so old trees keep hashing the same: no `heading` for 0, no
   `viewType` for bullets, no `align` for left, no `collapsed` for open blocks (extension 0.2.0+; the server has
   accepted it since website 0.13.0).
5. **`author` is outside the hash.** Changing only the author name updates the byline (`status: "updated"`);
   omitting `author` leaves the stored one alone, so older extensions don't wipe it.
6. **The server strips unknown keys before hashing.** A field the server doesn't know yet makes the hashes differ.
   Hence: server ships first ([shipping-changes.md](shipping-changes.md)).

## Serialization the server relies on

7. Children sorted by `:block/order` ascending.
8. Page root is `{ uid: pageUid, string: "", children }`; block root is the block itself.
9. Block refs are **inlined by the extension** (≤3 deep); the server never sees the graph and can't resolve them.
   Refs inside code, embeds and `[label](((uid)))` aliases stay as written — the renderer handles those.
10. Embed text stays in `string`; the embedded tree goes in `embed` (≤2 deep, cycles skipped). Page embeds carry
    `title`; `embed-children` has `string: ""`.
11. Block titles: the extension sends the first 200 chars of the string; the server strips markup and keeps 80.

## Identity

12. **A publication is `(graph of the key, rootUid)`.** The extension's local cache is keyed by `rootUid`; the
    server's unique index is `(graphId, rootUid)`. The graph comes from the key, never from the payload. The
    extension also names the Roam graph it's in (`x-roam-graph`); the server answers `409` when that isn't the key's
    graph, so a key pasted into another graph can't publish there. Requests without it (older builds) aren't checked.
13. **URLs come from the server.** The extension never builds a URL; it stores whatever `url` the server returns
    (graph URL, or the first collection URL when the page isn't in the graph).

## Behaviour the extension assumes

14. **New publications are Unlisted in their graph** (`listing: "unlisted"`). The "Make listed" / "Make
    discoverable" toast actions depend on `created` + `unlisted`.
15. **Republishing never changes Visibility, access or encryption.** An encrypted page stays encrypted: newer
    extensions seal the new tree, and the server seals a plain one from older extensions.
16. **Listing is the only Visibility concept the extension sees** (`unlisted` | `listed` | `discover`, plus the
    server's `discoverBlocked` reason, `inGraph`, and `encrypted`). The website's five-step Visibility ladder
    (Discover, Public, Unlisted, Password, Members) is built from `access` × `listing` per place and stays
    website-only, so the server can evolve it without an extension release; the Discover rules live on the server
    only. Wire values keep their old names (`listed` is the website's Public) because older extensions rely on
    them; renaming them would be a breaking change ([shipping-changes.md](shipping-changes.md)).
17. **Errors are shown verbatim.** The extension displays `error` (+ ` Reason: {reason}`) as-is, so server error
    strings are user-facing copy. The one exception: `401` with `Invalid API key` is replaced by "get a new key".
18. **Ownership is enforced server-side only.** Members get `403` on others' pages; the extension's `mine` field is
    informational.
19. **A removed page can't be deleted or republished** (`403`), so unpublish-and-republish can't dodge a takedown.
    For the same reason, a graph with a removed page can't be deleted from the website either. Once a graph or account
    is deleted, its keys get `401 Invalid API key`, which the extension shows as "get a new key". See
    [security.md](security.md#deleting-graphs-and-accounts).

## Transport

20. Headers `x-api-key: rp_…` (one key per person per graph), `x-roam-graph: <graph name>`, and from extension
    0.2.0 `x-roam-publish-version: <its package.json version>`. Every response carries
    `x-roam-publish-min-version`, the oldest extension the server works with (`EXT_MIN_VERSION`).
21. CORS: `https://roamresearch.com` (+ `localhost` outside production); allowed headers `content-type, x-api-key,
    x-roam-graph, x-roam-publish-version`; exposed header `x-roam-publish-min-version` (without it the extension
    couldn't read it); methods `GET, POST, PATCH, DELETE, OPTIONS`.
22. Payload ≤1 MB (bytes); block strings ≤100k chars; titles ≤1000; uids ≤64; trees ≤200 levels deep, counting
    children and embeds. The extension doesn't pre-check these.
23. Default server URL is baked in at build time (`ROAM_PUBLISH_SERVER`, default `https://roam.pub`) and
    overridable in settings.

## Encryption in Roam (extension 0.2.0+, website 0.18.0+)

24. **The ciphertext formats exist twice and must match byte for byte**: the extension's `src/seal.ts` and the
    website's `src/lib/encryption.ts` (server, v1) and `src/lib/reader-crypto.ts` (readers' browsers). Tree:
    `v1.{iv}.{tag}.{body}`, AES-256-GCM with additional data `tree:{publicationId}`, base64url without padding.
    Sealed key: `v1.{ephemeralSpki}.{iv}.{tag}.{body}`, X25519 then HKDF-SHA256 (salt: the ephemeral public key,
    info `roam-publish:content-key`), AES-256-GCM with additional data `content-key`. The server's zod schema checks
    those shapes. The `v1` prefix is the byte format, not the encryption version: v1 and v2 pages share it, and
    differ only in who ran it. Change neither copy without the other, and add an encryption version when it changes.
25. **The server decides whether a page is encrypted; the extension only carries it out.** The seal plan is asked
    before every publish and names the page id the cipher is bound to and every place's public key. The server
    re-checks the plan on `POST` and answers `409 { reseal: true }` when it moved; the extension retries once.
26. **Only public keys reach the extension.** It never sees a password, a private key or a reader's key, and
    keeps no content key after the publish.
27. **Falling back is always allowed for now.** No seal route (`404`), no X25519 in Roam, or an extension before
    0.2.0: the plain tree is sent and the server encrypts it (v1). Website 1.0.0 will refuse that for encrypted
    pages once every active extension is 0.2.0+ ([shipping-changes.md](shipping-changes.md)).
