# Shipping changes across both repos

Installed extensions update on Roam Depot's schedule, not ours, and old builds stay in use. The server must keep
working with every extension build still out there.

## Order

1. **Server first.** Accept the new field/endpoint, tolerate its absence, update `docs/api-contract.md`. Deploy.
2. **Then the extension.** Start sending the field. Release via Roam Depot.
3. Never the other way round: the server strips unknown keys before hashing, so a new field from the extension
   fails with `Content hash mismatch` until the server knows it.

## Checklist for a contract change

- [ ] `api-contract.md` updated in `roam-publish-web`.
- [ ] `Node` type + zod schema (server) and `Node` type + serializer (extension) agree.
- [ ] New optional fields are **omitted** at their default value, so existing trees still hash the same and
      nobody's pages show as changed.
- [ ] `stable-stringify.ts` untouched — or changed identically in both repos (this would rehash every page).
- [ ] Old extensions still work: missing field ⇒ previous behaviour (e.g. `author` absent leaves the byline alone).
- [ ] Error strings read well verbatim in a Roam toast.
- [ ] Commit message in the extension notes "requires roam-publish-web change deployed first".

## Retiring an endpoint

Return `410 { error }` with copy that tells the user what to do now (as `/api/ext/claim` does), rather than
removing the route, so old builds show a useful message.

## Local end-to-end

```bash
# roam-publish-web
bun install && cp .env.example .env.local && bun run db:migrate && bun dev   # :3000

# roam-publish
npm install && ROAM_PUBLISH_SERVER=http://localhost:3000 npm run dev
# Roam → Settings → Roam Depot → Developer mode → load the roam-publish folder
```

CORS allows `http://localhost:*` only outside production.
