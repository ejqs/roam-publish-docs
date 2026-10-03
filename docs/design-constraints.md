# Shared invariants

Rules that only hold if **both** repos keep them. Breaking one usually breaks installed extensions silently.
Rules internal to one repo (e.g. how access defaults or Discover work) live in that repo's docs.

## Hashing

1. **Both sides compute the hash and it must match byte for byte.**
   `contentHash = hex(sha256(stableStringify({ kind, title, tree })))`. The extension uses it to skip no-op
   publishes; the server recomputes it and rejects mismatches (`400`).
2. **`stable-stringify.ts` exists as identical copies** —
   `roam-publish/src/stable-stringify.ts` and `roam-publish-web/src/lib/stable-stringify.ts`. Keys sorted
   recursively, no whitespace, `undefined` dropped. Never edit one without the other.
3. **The `Node` type exists twice** — `roam-publish/src/serialize.ts` and `roam-publish-web/src/db/app-schema.ts`
   (plus the zod `NodeSchema` in `src/app/api/ext/publications/route.ts`). The server's type is a superset: it also
   accepts the default values (`"bullet"`, `"left"`) the extension never sends.
4. **Optional fields are omitted at their default** so old trees keep hashing the same: no `heading` for 0, no
   `viewType` for bullets, no `align` for left.
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
    server's unique index is `(graphId, rootUid)`. The graph comes from the key, never from the payload.
13. **URLs come from the server.** The extension never builds a URL; it stores whatever `url` the server returns
    (graph URL, or the first collection URL when the page isn't in the graph).

## Behaviour the extension assumes

14. **New publications are `unlisted`.** The "Make public" toast action depends on `created` + `unlisted`.
15. **Republishing never changes visibility or access.**
16. **Visibility is the only access concept the extension sees** (`public` | `unlisted`). Everything else is
    website-only by design, so the server can evolve it without an extension release.
17. **Errors are shown verbatim.** The extension displays `error` (+ ` Reason: {reason}`) as-is, so server error
    strings are user-facing copy. The one exception: `401` with `Invalid API key` is replaced by "get a new key".
18. **Ownership is enforced server-side only.** Members get `403` on others' pages; the extension's `mine` field is
    informational.
19. **A removed page can't be deleted or republished** (`403`), so unpublish-and-republish can't dodge a takedown.
    For the same reason, a graph with a removed page can't be deleted from the website either. Once a graph or account
    is deleted, its keys get `401 Invalid API key`, which the extension shows as "get a new key". See
    [security.md](security.md#deleting-graphs-and-accounts).

## Transport

20. Header `x-api-key: rp_…`; one key per person per graph.
21. CORS: `https://roamresearch.com` (+ `localhost` outside production); headers `content-type, x-api-key`;
    methods `GET, POST, PATCH, DELETE, OPTIONS`.
22. Payload ≤1 MB (bytes); block strings ≤100k chars; titles ≤1000; uids ≤64; trees ≤200 levels deep, counting
    children and embeds. The extension doesn't pre-check these.
23. Default server URL is baked in at build time (`ROAM_PUBLISH_SERVER`, default `https://roam.pub`) and
    overridable in settings.
