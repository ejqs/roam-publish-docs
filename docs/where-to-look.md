# Where things are documented

Everything not about the interaction lives in its own repo.

## `roam-publish` (extension)

| Topic | Where |
| --- | --- |
| Setup, usage, commands | [README → Setup / Usage](https://github.com/ejqs/roam-publish#setup) |
| What renders, privacy summary | [README → Supported blocks / Privacy and safety](https://github.com/ejqs/roam-publish#privacy-and-safety) |
| What's read, sent and written; Append API | [Data flow and the Append API](data-and-append-api.md) |
| Serialization (refs, embeds, view types) | `src/serialize.ts` |
| Local cache and settings | `src/state.ts`, `src/settings.ts` |
| Shortlink block in Roam | `src/publish.ts` (`ensureShortlinkBlock`), `src/serialize.ts` (`isShortlinkBlock`) |
| Encrypting in Roam, keyed hash (0.2.0, `develop`) | `src/seal.ts`, `sealFor` in `src/publish.ts`, `getHashKey` in `src/state.ts` |
| Collapsed blocks (0.2.0, `develop`) | `foldedUids`, `refold` and the Republish prompts in `src/publish.ts` |
| Version header, update notice | `src/api.ts` (`checkMinVersion`), `CLAUDE.md` → Versions and roam.pub |
| Release notes | `CHANGELOG.md` (shown at roam.pub/updates) |
| Build / Roam Depot | `build.mjs`, `build.sh` |

## `roam-publish-web` (server + site)

| Topic | Where |
| --- | --- |
| **Wire contract** | [`docs/api-contract.md`](https://github.com/ejqs/roam-publish-web/blob/main/docs/api-contract.md) |
| Dev setup, schema changes | [README](https://github.com/ejqs/roam-publish-web#readme) |
| Extension endpoints | `src/app/api/ext/**`, `src/lib/ext-auth.ts` |
| Graph verification | `src/server/actions/onboarding.ts`, `src/lib/roam-append.ts` |
| Shortlinks (`/p/{id}`), change log | `src/lib/shortlinks.ts`, `src/lib/changelog.ts`, `src/app/p/[id]/page.tsx`, contract § *Shortlinks and the change log* |
| Stored append-only token | `src/lib/append-token.ts`, `src/server/actions/change-log.ts`, `src/app/(app)/dashboard/[graph]/settings/change-log-form.tsx` |
| Server actions (dashboard, unlock, tags) | `src/server/actions/` (moved out of `src/app` in October 2026) |
| Keys | `src/lib/keys.ts`, `/dashboard/keys` |
| Access, places, passwords | `src/lib/gates.ts`, `src/lib/places.ts`, contract § *Places and access* |
| Visibility ladder (Discover, Public, Unlisted, Password, Members) | `src/lib/listing.ts` (`extListing` for the extension), `src/lib/control-rules.ts` |
| Encryption (v1 server, v2 sealed publish, reader decryption, unlock) | `src/lib/encryption.ts`, `src/lib/encryption-rules.ts`, `src/lib/e2e-publish.ts`, `src/lib/reader-crypto.ts`, `src/server/actions/unlock.ts`, `src/app/api/ext/publications/[rootUid]/seal/route.ts`, roam.pub `/privacy/encryption` and `/privacy/encryption/versions` |
| Extension versions in use, minimum version, upcoming changes | `src/lib/ext-compat.ts`, `src/lib/ext-version.ts`, `src/lib/upcoming.ts`, `scripts/ext-gate.ts`, `/admin/extension`, roam.pub `/updates/upcoming`, `CLAUDE.md` → Extension compatibility |
| Release notes | `CHANGELOG.md` (shown at roam.pub/updates with the extension's), `src/lib/whats-new.ts` |
| Discover rules | `src/lib/discover-rules.ts`, `src/lib/discover.ts` |
| RSS feeds | `src/lib/feeds.ts`, `src/app/**/feed.xml/route.ts`, contract § *RSS feeds* |
| Tags, search, list filters | `src/lib/tags.ts`, `src/lib/list-params.ts`, `src/lib/list-query.ts`, `src/lib/site-search.ts`, `src/server/actions/tags.ts` (website tag edits), `canSearchSite` in `src/lib/graph-access.ts`, contract § *Tags and search* |
| Members, invites, transfers | `src/lib/invites.ts`, `src/lib/graph-access.ts` |
| Moderation | `src/lib/moderation*.ts`, `src/app/admin/**` |
| Monitoring: latency, errors, health, failure emails | `src/lib/telemetry.ts`, `src/lib/alerts.ts`, `/admin/status`, `GET /api/health`, [README → Monitoring](https://github.com/ejqs/roam-publish-web#monitoring) |
| Rendering Roam markup, `[[links]]` (`PageLinks`), collapsing, zoom, outline, code and Mermaid blocks | `src/components/roam/*`, `src/lib/folds.ts`; links are built in `src/app/[graph]/[uid]/[[...slug]]/page.tsx` and `src/app/c/[id]/[[...slug]]/page.tsx` |
| Front pages, folders, Discover | `src/app/[graph]/page.tsx`, `src/app/c/[id]/`, `src/app/discover/` |
| Data model | `src/db/app-schema.ts`, `drizzle/` |
| Framework notes (Next.js 16) | `AGENTS.md` |
