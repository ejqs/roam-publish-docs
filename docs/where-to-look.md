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
| Build / Roam Depot | `build.mjs`, `build.sh` |

## `roam-publish-web` (server + site)

| Topic | Where |
| --- | --- |
| **Wire contract** | [`docs/api-contract.md`](https://github.com/ejqs/roam-publish-web/blob/main/docs/api-contract.md) |
| Dev setup, schema changes | [README](https://github.com/ejqs/roam-publish-web#readme) |
| Extension endpoints | `src/app/api/ext/**`, `src/lib/ext-auth.ts` |
| Graph verification | `src/app/(app)/onboarding/actions.ts`, `src/lib/roam-append.ts` |
| Shortlinks (`/p/{id}`), change log | `src/lib/shortlinks.ts`, `src/lib/changelog.ts`, `src/app/p/[id]/page.tsx`, contract § *Shortlinks and the change log* |
| Stored append-only token | `src/lib/append-token.ts`, `src/app/(app)/dashboard/[graph]/settings/change-log-*` |
| Keys | `src/lib/keys.ts`, `/dashboard/keys` |
| Access, places, passwords | `src/lib/gates.ts`, `src/lib/places.ts`, contract § *Places and access* |
| Discover rules | `src/lib/discover-rules.ts`, `src/lib/discover.ts` |
| RSS feeds | `src/lib/feeds.ts`, `src/app/**/feed.xml/route.ts`, contract § *RSS feeds* |
| Tags, search, list filters | `src/lib/tags.ts`, `src/lib/list-params.ts`, `src/lib/list-query.ts`, `src/lib/site-search.ts`, `src/app/(app)/dashboard/tag-actions.ts` (website tag edits), `canSearchSite` in `src/lib/graph-access.ts`, contract § *Tags and search* |
| Members, invites, transfers | `src/lib/invites.ts`, `src/lib/graph-access.ts` |
| Moderation | `src/lib/moderation*.ts`, `src/app/admin/**` |
| Monitoring: latency, errors, health, failure emails | `src/lib/telemetry.ts`, `src/lib/alerts.ts`, `/admin/status`, `GET /api/health`, [README → Monitoring](https://github.com/ejqs/roam-publish-web#monitoring) |
| Rendering Roam markup | `src/components/roam/*` |
| Data model | `src/db/app-schema.ts`, `drizzle/` |
| Framework notes (Next.js 16) | `AGENTS.md` |
