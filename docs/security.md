# Trust boundary

Three parties: **Roam** (the user's graph), the **extension** (runs inside Roam, in the user's browser), and the
**server** (roam.pub). This page is about what crosses between them. Each repo's own security notes:
the extension README's *Safety* section, and the server's moderation/auth code.

## What crosses where

| From → To | What | When |
| --- | --- | --- |
| Roam → extension | The selected tree, referenced block text, embedded trees | Only on a publish action |
| Extension → server | `PublishPayload` + hash + author, API key | Only on a user action |
| Server → extension | Status, URL, hash, visibility, publication list, error strings | In response |
| Website → Roam Append API | One block on today's daily note, using a token the user pasted | Once, at graph verification |
| Extension → anywhere else | Nothing | — |

## Two different credentials, never mixed

- **Roam append-only token** (`roam-graph-token-…`): entered on the **website**, used once to prove control of a
  graph, never stored. Can only add blocks; can't read the graph.
- **Roam Publish API key** (`rp_…`): issued on the website, pasted into the **extension**. Lets the server act for
  that person on that one graph. Gives **no** access to the Roam graph. Stored hashed server-side; shown once.

## Why setup doesn't go through the graph

An earlier design had the server write a claim code to the daily note and the extension read it to fetch a key.
Anyone who could read the graph (collaborators on a shared graph) could claim it first. Now verification finishes on
the website and the user pastes the key, so nothing in the graph is a secret. `POST /api/ext/claim` returns `410`
for old extension builds.

## Shared graphs

- First account to verify a graph owns it. Others join only by the owner's email invite, then get **their own**
  key. Removing a member revokes their key; their pages stay.
- Open question: whether Roam Depot extension settings (where the key lives) are per-person or shared in a
  multiplayer graph. See [open-questions.md](open-questions.md).

## The server never trusts the payload for identity

The graph, the person and their role all come from the key. The server re-hashes content, validates with zod, caps
size, and enforces ownership, removal and suspension on every call.
