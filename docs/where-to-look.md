# Where things are documented

Everything not about the interaction lives in its own repo.

## `roam-publish` (extension)

| Topic | Where |
| --- | --- |
| Setup, usage, commands | [README → Setup / Usage](https://github.com/ejqs/roam-publish#setup) |
| What gets published, what's read/sent/stored | [README → What gets published / Safety](https://github.com/ejqs/roam-publish#safety) |
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
| Members, invites, transfers | `src/lib/invites.ts`, `src/lib/graph-access.ts` |
| Moderation | `src/lib/moderation*.ts`, `src/app/admin/**` |
| Rendering Roam markup | `src/components/roam/*` |
| Data model | `src/db/app-schema.ts`, `drizzle/` |
| Framework notes (Next.js 16) | `AGENTS.md` |
