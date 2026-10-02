# Roam Publish — docs

How the two Roam Publish repos **work together**. Each repo documents its own internals; this repo covers only
what spans both — the boundary, the shared invariants, and how to change them safely — and points to the rest.

| Repo | Role | Its own docs |
| --- | --- | --- |
| [`ejqs/roam-publish`](https://github.com/ejqs/roam-publish) | Roam Depot extension. Reads a page or block, serializes, hashes, sends. | [README](https://github.com/ejqs/roam-publish#readme) (setup, usage, what gets published, safety) |
| [`ejqs/roam-publish-web`](https://github.com/ejqs/roam-publish-web) | Server + website at [roam.pub](https://roam.pub). Verifies graphs, issues keys, stores, renders, owns all access settings. | [README](https://github.com/ejqs/roam-publish-web#readme), [`docs/api-contract.md`](https://github.com/ejqs/roam-publish-web/blob/main/docs/api-contract.md) |

## Pages

1. [Architecture](docs/architecture.md) — who does what, and the flows that cross the boundary.
2. [Shared invariants](docs/design-constraints.md) — rules both repos must keep in lockstep.
3. [Trust boundary](docs/security.md) — what crosses between Roam, the extension and the server.
4. [Shipping changes](docs/shipping-changes.md) — changing the contract without breaking installed extensions.
5. [Open questions](docs/open-questions.md) — unverified assumptions that involve both sides.
6. [Where things are documented](docs/where-to-look.md) — pointers into each repo for everything else.

`inbox/` holds raw notes not yet folded in.

## Source of truth

The wire contract is [`roam-publish-web/docs/api-contract.md`](https://github.com/ejqs/roam-publish-web/blob/main/docs/api-contract.md).
These pages explain *why* and *how the pieces fit*; they don't restate the contract. When code, contract and these
pages disagree, code and contract win — fix the page.

---

Roam Publish is a third-party service made by [@ejqs](https://ejqs.net). Not affiliated with Roam Research.
