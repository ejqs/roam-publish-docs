# Where things are documented

Everything not about the interaction lives in its own repo.

## `roam-publish` (extension)

| Topic | Where |
| --- | --- |
| Setup, usage, commands | [README → Setup / Usage](https://github.com/ejqs/roam-publish#setup) |
| What gets published, what's read/sent/stored | [README → What gets published / Safety](https://github.com/ejqs/roam-publish#safety) |
| Serialization (refs, embeds, view types) | `src/serialize.ts` |
| Local cache and settings | `src/state.ts`, `src/settings.ts` |
| Build / Roam Depot | `build.mjs`, `build.sh` |

## `roam-publish-web` (server + site)

| Topic | Where |
| --- | --- |
| **Wire contract** | [`docs/api-contract.md`](https://github.com/ejqs/roam-publish-web/blob/main/docs/api-contract.md) |
| Dev setup, schema changes | [README](https://github.com/ejqs/roam-publish-web#readme) |
| Extension endpoints | `src/app/api/ext/**`, `src/lib/ext-auth.ts` |
| Graph verification | `src/app/(app)/onboarding/actions.ts`, `src/lib/roam-append.ts` |
| Keys | `src/lib/keys.ts`, `/dashboard/keys` |
| Access, places, passwords | `src/lib/gates.ts`, `src/lib/places.ts`, contract § *Places and access* |
| Discover rules | `src/lib/discover-rules.ts`, `src/lib/discover.ts` |
| Members, invites, transfers | `src/lib/invites.ts`, `src/lib/graph-access.ts` |
| Moderation | `src/lib/moderation*.ts`, `src/app/admin/**` |
| Rendering Roam markup | `src/components/roam/*` |
| Data model | `src/db/app-schema.ts`, `drizzle/` |
| Framework notes (Next.js 16) | `AGENTS.md` |
