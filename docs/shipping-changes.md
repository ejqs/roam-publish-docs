# Shipping changes across both repos

Installed extensions update on Roam Depot's schedule, not ours, and old builds stay in use. The server must keep
working with every extension build still out there.

## Order

1. **Server first.** Accept the new field/endpoint, tolerate its absence, update `docs/api-contract.md`. Deploy to
   `main`.
2. **Then the extension.** Start sending the field. Release via Roam Depot, only once roam.pub's `main` has what it
   needs (the extension's `CLAUDE.md`).
3. Never the other way round: the server strips unknown keys before hashing, so a new field from the extension
   fails with `Content hash mismatch` until the server knows it.

Encryption in Roam went this way: website 0.18.0 added the seal route and the sealed publish beside the plain one;
extension 0.2.0 (on `develop`, not yet in Roam Depot) uses them; the plain path for encrypted pages goes later, as
below.

## Versions

Both repos use [semantic versions](https://semver.org), each with its own `CHANGELOG.md`, and roam.pub shows both
at `/updates` (What's new). The two majors don't have to match.

- **A major bump means something was removed.** The website bumps its major only for a release that removes or
  changes something older extensions rely on (an `/api/ext` field or route), with a `Breaking:` bullet; the
  extension bumps its major only when it removes something people rely on. Everything else is a minor (`New:`) or
  patch (`Improved:`, `Fixed:`). Both repos' tests and Changelog checks enforce the bullets.
- **The extension says its version** in `x-roam-publish-version` (from 0.2.0; older builds send nothing and are
  stored as unknown). The server records it per person and graph (`ext_client`, at most hourly) and shows usage at
  `/admin/extension`; an email goes to admins when every install active in the last 30 days is on the version Roam
  Depot serves.
- **The server names the oldest extension it works with** on every `/api/ext` response
  (`x-roam-publish-min-version`, `EXT_MIN_VERSION` in `src/lib/ext-compat.ts`, an exact version, `0.0.0` today).
  An extension from 0.2.0 that's older asks the person once per session to update in Roam Depot (`checkMinVersion`
  in the extension's `src/api.ts`). A route can also refuse one request with `426` and an update message
  (`updateNeeded`).

## Removing something the extension relies on

`/api/ext` changes are additive; where a shape must change, the route branches on `ctx.extVersion`
(`extAtLeast` in `src/lib/ext-version.ts`). Removing the old shape takes three steps, announced at
`roam.pub/updates/upcoming` (`src/lib/upcoming.ts`) from the first:

1. The website adds the new shape beside the old one (a minor release) and lists the change as upcoming: what
   changes, what to do, the website version that will make it, and the extension version everyone needs. The page
   shows how many active installs are ready and, signed in, which of your graphs need an update.
2. An extension release moves to the new shape.
3. Once `/admin/extension` (or the "everyone is on extension x" email) shows everyone has updated, the website
   drops the old shape as its next **major**, raises `EXT_MIN_VERSION` to that extension version, and moves the
   entry from `upcoming.ts` to a `Breaking:` bullet.

Production enforces the wait: the production pre-deploy step (`bun run db:migrate`) runs `scripts/ext-gate.ts`
first, which fails the deploy while anyone active in the last 30 days is on an extension older than
`EXT_MIN_VERSION`. There's no override.

Upcoming now: **Password pages must be encrypted in Roam** (website 1.0.0, needs extension 0.2.0). Older
extensions still send the plain tree and roam.pub encrypts it (v1); 1.0.0 stops accepting that, so every Password
page is end-to-end encrypted.

## Checklist for a contract change

- [ ] `api-contract.md` updated in `roam-publish-web`.
- [ ] `Node` type + zod schema (server) and `Node` type + serializer (extension) agree.
- [ ] New optional fields are **omitted** at their default value, so existing trees still hash the same and
      nobody's pages show as changed.
- [ ] `stable-stringify.ts` untouched — or changed identically in both repos (this would rehash every page).
- [ ] Old extensions still work: missing field ⇒ previous behaviour (e.g. `author` absent leaves the byline alone).
- [ ] Error strings read well verbatim in a Roam toast.
- [ ] Commit message in the extension notes "requires roam-publish-web change deployed first".
- [ ] A change in the encrypted formats is made in `seal.ts`, `encryption.ts` and `reader-crypto.ts` together
      ([design-constraints.md](design-constraints.md#encryption-in-roam-extension-020-website-0180)).
- [ ] Something older extensions rely on is only removed through the three steps above.
- [ ] Each repo's `CHANGELOG.md` has a bullet for anything users would notice.

## Retiring an endpoint

Return `410 { error }` with copy that tells the user what to do now (as `/api/ext/claim` does), rather than
removing the route, so old builds show a useful message. A route that now needs a newer extension answers `426`
with `updateNeeded`, which every version shows as it is.

## Local end-to-end

```bash
# roam-publish-web
bun install && cp .env.example .env.local && bun run db:migrate && bun dev   # :3000

# roam-publish
npm install && ROAM_PUBLISH_SERVER=http://localhost:3000 npm run dev
# Roam → Settings → Roam Depot → Developer mode → load the roam-publish folder
```

CORS allows `http://localhost:*` only outside production.
