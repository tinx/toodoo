# Spec: Sync and Concurrency

This spec defines how clients keep state in sync with the server and how
concurrent mutations are reconciled. Required reading first:
`docs/adr/0001-no-e2e-server-authoritative.md` (establishes the
server-authoritative model that makes this design possible) and
`docs/specs/data-model.md` (defines the `version` field this spec uses).

## Goals and Non-Goals

**Goals:**
- Clients (CLI, web, MCP) see consistent state after sync.
- Two users editing different items in the same list succeed
  concurrently with no operator intervention.
- Two users editing the **same** item: one succeeds, the other gets a
  clear conflict response with the current state, and the client
  surfaces a "your version vs theirs" choice to the user.
- Offline CLI can queue mutations and replay on reconnect.

**Non-goals:**
- Real-time collaborative editing of a single item (it's a todo, not a
  document). No CRDTs, no operational transforms.
- Server push / websockets in v1. Clients poll. Push is a v2 option.
- Cross-instance federation. Each instance is independent.

## Concurrency Model: Optimistic, Per-Item Versions

Every `items` row has a `version INTEGER` column that increases by 1
on every successful mutation. The server is the single source of truth
for the version.

### Mutation Request

A client mutating an item sends the version it observed at read time:

```http
PATCH /api/items/{id}
If-Match: "42"
Content-Type: application/json

{ "done": true }
```

We use the standard HTTP `If-Match` header rather than a body field so
the same conditional-request semantics apply across PATCH, PUT, and
DELETE. Quoted ETag form per RFC 7232.

The Engine performs the update in a single SQL statement:

```sql
UPDATE items
   SET done = ?, version = version + 1, updated_at = ?
 WHERE id = ? AND version = ?
```

If exactly one row is updated → success. The response includes the new
version as `ETag`. If zero rows are updated, the Engine reads the
current row and returns `409 Conflict` (see below).

Reads always return the current version in `ETag`:

```http
GET /api/items/{id}
→ 200 OK
  ETag: "42"
  ...
```

### Conflict Response

```http
409 Conflict
Content-Type: application/json
ETag: "47"

{
  "error": "version_conflict",
  "current": { ... full current item representation ... },
  "your_version": 42,
  "current_version": 47
}
```

The client renders a conflict UI: show both versions, let the user pick
one or merge by hand, then re-PATCH with `If-Match: "47"`. The
client-side flow never silently overwrites.

### Why Per-Item, Not Per-List

- Per-list `version` would force every item mutation to bump the list
  and would serialize all writes to the list — bad for shared lists.
- Per-item gives the right granularity: two users editing different
  items in the same list don't conflict at all.
- Per-field would reduce conflicts further but adds substantial
  complexity for a marginal benefit (a todo is small).

### Creates and Deletes

- **Create:** `POST /api/lists/{id}/items` is unconditional. Returns
  the new item with `version: 1` and an `ETag`. Two concurrent creates
  produce two distinct items; this is correct behavior, not a conflict
  — distinct creates were genuinely both intended.
- **Delete:** `DELETE /api/items/{id}` requires `If-Match`. Deletes are
  soft (set `deleted_at`, bump `version`), so a stale concurrent edit
  still sees a `409` with the deletion as the current state.
- **Restore:** `POST /api/items/{id}/restore` requires `If-Match`;
  clears `deleted_at`, bumps `version`. Writer-permitted — deleted
  items are a trash, not an archive of secrets.
- **Purge:** `DELETE /api/items/{id}?purge=1` requires `If-Match`;
  owner-only. Removes the row permanently and stamps the list's
  `last_purged_at`, which forces a sync reset (see "Reset Marker")
  on any client whose token predates the purge — purged items leave
  no tombstone, and the reset is what prevents stale clients from
  resurrecting them.

## Pull-Based Sync Protocol

v1 uses simple long-poll style pulls. No websockets, no server-sent
events.

### Initial Fetch

```http
GET /api/lists/{id}/items
→ 200 OK
  X-Sync-Token: "1719340000:v1"

  [ {item}, {item}, ... ]   # live items only; deleted items are
                            # excluded from the initial fetch
```

The initial fetch excludes soft-deleted items: a client starting from
nothing has nothing to remove, and tombstones are retained forever
(see data-model.md), so including them all would bloat the first
fetch without bound. A trash view fetches them explicitly with
`?include_deleted=true`. *Incremental* responses do include
tombstones — that is how an existing client learns about deletions.

The `X-Sync-Token` opaquely encodes "what the client has seen up to."
Format is server-internal; clients treat it as a black box. v1
implementation: `unix_seconds:schema_version`.

### Incremental Sync

```http
GET /api/lists/{id}/items?since=1719340000:v1
→ 200 OK
  X-Sync-Token: "1719340900:v1"

  [ {item with updated_at >= 1719340000} ]
```

Server-side this is `WHERE list_id = ? AND updated_at >= ?` ordered by
`updated_at`. The index on `(list_id, updated_at)` makes this cheap.

Notes:
- `updated_at` is server-set; clients never write it. This avoids
  clock-skew issues.
- Soft-deleted items are included in incremental responses so clients
  learn about deletions. Tombstones are retained indefinitely (see
  data-model.md), so there is no retention window to outlast — a
  client can be offline arbitrarily long and still sync deltas.
- The only events that invalidate a sync token are a **schema change**
  and a **purge** (see "Reset Marker" below).
- The schema version in the token allows the server to force a full
  re-fetch after a schema change.

### Reset Marker

Two events make an incremental sync unservable: a **schema change**
(the token's embedded schema version no longer matches) and a
**purge** (the token predates the list's `last_purged_at` — purged
items leave no tombstone, so serving a delta would let the stale
client resurrect them). In either case the server responds
`409 Conflict`:

```http
GET /api/lists/{id}/items?since=1700000000:v0
→ 409 Conflict
  X-Sync-Reset: "1"

  { "error": "sync_reset_required" }
```

Client discards local state for the list and re-fetches from scratch.

### List Index Poll (discovery)

The per-list delta sync above only covers *items in lists the client
already knows about*. Everything else — new lists, lists shared with
you, lost access, deleted lists, title changes, membership changes,
label-vocabulary changes — is discovered through the **index poll**:

```http
GET /api/lists
→ 200 OK
[
  { "id": "...", "title": "Groceries", "version": 7,
    "owner": { "id": "...", "handle": "alice" },
    "my_role": "writer", "archived_at": null },
  ...
]
```

- Returns every list the caller owns or is a member of, with each
  list's envelope `version` (see data-model.md: bumped on title,
  archive state, membership, and label changes — never on item
  mutations).
- Clients poll this at the normal cadence and **diff against local
  state**:
  - **New id** → run the initial item fetch, fetch labels and
    members.
  - **Missing id** → the list was deleted or the caller was removed
    from it; drop all local data for that list. (This is also how a
    client learns it has been unshared.)
  - **`version` changed** → re-fetch the envelope: list metadata,
    members, labels. Items are unaffected — the per-list delta
    covers them.
  - **`version` unchanged** → nothing to do for this list.
- This is a deliberate full fetch on every poll, not a delta: the
  index is small (a user has dozens of lists, not thousands), and a
  single cheap query beats delta bookkeeping at this tier.
- Archived lists are included (`archived_at` set) so clients can
  offer an archive view; hiding them is a client-side filter.

Together the two tiers fully define how clients converge: the index
poll discovers and invalidates lists; the per-list delta sync
converges items within each known list.

### Polling Cadence

- Web client: poll every 15s when foregrounded, back off when tab is
  hidden.
- CLI when running a `--watch` command: 5s.
- CLI for one-shot commands: single sync at start, no polling.
- The server caps polling rate per session at 1 req/s and returns 429
  on abuse — this is a rate-limit defense, not a correctness one.

## Offline Client Behavior (CLI)

The CLI maintains a local SQLite cache mirroring the data model
(read-only from the user's perspective) plus a `pending_mutations`
table:

```
pending_mutations
  id           INTEGER PK AUTOINCREMENT
  item_id      TEXT
  method       TEXT          -- 'PATCH', 'POST', 'DELETE'
  if_match     TEXT          -- the version the user based the change on
  body         BLOB          -- JSON
  queued_at    INTEGER
  last_attempt INTEGER
  last_error   TEXT
```

### Replay

On reconnect, the CLI replays pending mutations in FIFO order:

- **Success:** remove the row, update local cache from the response.
- **409 Conflict:** stop replay, surface the conflict to the user. Once
  resolved (user picks a side), the resolution is re-queued and replay
  resumes.
- **Network error / 5xx:** exponential backoff, retry. Don't drop the
  mutation.
- **4xx (other than 409):** the mutation is invalid — record the error
  in `last_error`, surface to user, do not retry automatically.

### What Cannot Be Done Offline

- Creating users, invites, sessions, ACL changes — these touch shared
  state and must hit the server.
- Reading data the local cache doesn't have. We sync the lists the
  user has membership in; anything else requires a server round-trip.

### Long Absences

There is no retention window to outlast — tombstones persist forever,
so a CLI that has been offline for a year still syncs deltas
normally. The only forced re-fetch cases are a schema change or a
purge in the meantime (see "Reset Marker"). On a forced re-fetch, the
CLI confirms with the user before discarding local pending mutations
against items that no longer exist server-side (a pending mutation
against a purged item gets a `404` on replay and is surfaced, not
silently dropped).

## Web Client Behavior

The web client does **not** maintain a long-lived offline cache. State
lives in memory while the tab is open. Rationale: it's an SPA but not a
PWA, and storing user data in IndexedDB without strong reason
complicates the threat model (XSS that gets a foothold persists across
sessions). When the tab is closed, state is gone; on next open, full
fetch.

If a request is made with a stale `If-Match` because another tab
mutated the item, the client refreshes from the conflict response and
prompts the user.

## API Surface Summary

| Method | Path                            | Notes                          |
|--------|----------------------------------|--------------------------------|
| GET    | /api/lists                       | lists I own or am member of    |
| POST   | /api/lists                       | create list, returns ETag      |
| GET    | /api/lists/{id}                  | metadata, ETag                 |
| PATCH  | /api/lists/{id}                  | requires If-Match              |
| DELETE | /api/lists/{id}                  | requires If-Match; owner only  |
| GET    | /api/lists/{id}/items            | initial fetch                  |
| GET    | /api/lists/{id}/items?since=...  | incremental sync               |
| POST   | /api/lists/{id}/items            | create, unconditional          |
| GET    | /api/items/{id}                  | single item, ETag              |
| PATCH  | /api/items/{id}                  | requires If-Match              |
| DELETE | /api/items/{id}                  | requires If-Match; soft delete |
| POST   | /api/items/{id}/restore          | requires If-Match; writer      |
| DELETE | /api/items/{id}?purge=1          | requires If-Match; owner only  |
| GET    | /api/lists/{id}/labels           | labels defined on this list    |
| POST   | /api/lists/{id}/labels           | create label, returns ETag     |
| PATCH  | /api/labels/{id}                 | requires If-Match              |
| DELETE | /api/labels/{id}                 | requires If-Match              |

Item ↔ label associations are managed inline via the item's `labels`
array: `PATCH /api/items/{id}` with `{ "labels": ["<label_id>", ...] }`
replaces the full set in one round trip. The Engine validates that
every referenced label belongs to the item's list. There are
intentionally no separate `/api/items/{id}/labels` endpoints — fewer
moving parts, and label sets are small.

Authorization checks on every request: see
`docs/specs/sharing-and-authz.md`.

## Error Format

All 4xx and 5xx responses use a stable JSON shape:

```json
{
  "error": "stable_machine_readable_code",
  "message": "human-readable description",
  "details": { ... }  // optional, error-specific
}
```

Stable error codes used by sync:

- `version_conflict` (409) — `If-Match` mismatch on a mutation.
- `sync_reset_required` (409) — incremental sync token unusable.
- `not_found` (404)
- `forbidden` (403)
- `unauthorized` (401)
- `validation_failed` (400) — with `details.field`

## Open Items

- v2: push via SSE for "list page is open, item changed" UI nudges.
  Designed to be additive — clients still pull, push is just a hint
  that "you should pull now."
- Whether to support partial PATCH semantics for lists (currently
  PATCH replaces the whole `title`). Defer until there's a second
  mutable list field.
