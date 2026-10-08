# Open questions

Assumptions that involve both repos and haven't been verified. Single-repo questions belong in that repo.

- **Are Roam extension settings per person in a shared graph?** The API key, Author name and publication cache
  are stored via Roam Depot's extension settings, which live in the graph, and so is the `hash-key` behind encrypted
  pages' keyed hashes (extension 0.2.0). If collaborators see each other's
  values, they'd share one key — defeating one-key-per-person. Fix would be browser `localStorage` keyed by graph.
  *(Extension stores it; server's per-member keys and ownership rules assume it's private.)*
- **Do Roam image URLs stay live?** Media is sent as URLs (usually Firebase Storage with a long-lived token) and the
  server renders them as-is. If they expire or change, published pages break and the extension would need to
  re-host files at publish time, which means a new upload endpoint on the server.
- **Page context-menu payload shape** is undocumented; the extension guesses `page-uid` / `uid` / `title` keys and
  falls back to a title query.
- **Stale local cache.** The extension's cache is refilled only when empty or on "Sync". Changes made on the
  website (unpublish, visibility) aren't reflected until then; the server is always authoritative.
- **Can every Roam publish v2 once the plain path goes?** Extension 0.2.0 needs X25519 in WebCrypto to encrypt in
  Roam, and falls back to sending the plain tree (v1) without it, for example in an older Roam desktop app. Website
  1.0.0, announced on `/updates/upcoming`, will refuse plain trees for encrypted pages, so such a Roam couldn't
  republish them. Either the extension bundles an X25519 fallback (as the design suggested, `@noble/curves`) or the
  server keeps v1 for clients that can't seal. *(Extension `canSeal` in `src/seal.ts`; server
  `src/app/api/ext/publications/route.ts`.)*
- **Is "Password always means encrypted" next?** The Visibility design (2026-10-08) decided Password pages would
  always be end-to-end encrypted, but today encryption is still opt-in per page or through "Encrypt new password
  pages" (website 0.18.0), and a Password page that isn't encrypted is only gated. The docs describe what's built.
