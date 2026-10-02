# Open questions

Assumptions that involve both repos and haven't been verified. Single-repo questions belong in that repo.

- **Are Roam extension settings per person in a shared graph?** The API key, Author name and publication cache
  are stored via Roam Depot's extension settings, which live in the graph. If collaborators see each other's
  values, they'd share one key — defeating one-key-per-person. Fix would be browser `localStorage` keyed by graph.
  *(Extension stores it; server's per-member keys and ownership rules assume it's private.)*
- **Do Roam image URLs stay live?** Media is sent as URLs (usually Firebase Storage with a long-lived token) and the
  server renders them as-is. If they expire or change, published pages break and the extension would need to
  re-host files at publish time, which means a new upload endpoint on the server.
- **Page context-menu payload shape** is undocumented; the extension guesses `page-uid` / `uid` / `title` keys and
  falls back to a title query.
- **Stale local cache.** The extension's cache is refilled only when empty or on "Sync". Changes made on the
  website (unpublish, visibility) aren't reflected until then; the server is always authoritative.
