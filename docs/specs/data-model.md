# Spec: Data Model

This spec defines the persistent data model owned by the **Engine
service**. It is the contract that the Engine's storage port
(`Storage` interface) and its adapters (SQLite default, Postgres
optional) implement.

The schema is the same on both backends. Type names below are SQLite
flavor; the Postgres adapter maps them to native types.

**Scope boundary** (see ADR 0004): the Engine's database holds
*business* data only — the user as a business entity, lists, items,
labels, ACLs. **Credentials, identities, sessions, and invites are
owned by the Authenticator service in a separate SQLite database**
under the Authenticator's own `StateDirectory=`. The two databases
never share a file. They are linked only by `user_id`. See
`docs/specs/authentication.md` for the Authenticator's schema.

## Tables

### users

This is the **Engine's view** of a user — the business identity that
owns lists and items. The Engine does not store credentials and does
not know how anyone authenticates. See ADR 0004 and
`docs/specs/authentication.md` for the Authenticator's parallel
representation.

```
id            TEXT PRIMARY KEY              -- UUIDv7
handle        TEXT NOT NULL UNIQUE          -- login name, 3-32 chars,
                                            -- [a-z0-9_-], case-folded
display_name  TEXT NOT NULL                 -- shown in UI
created_at    INTEGER NOT NULL              -- unix seconds
disabled_at   INTEGER                       -- nullable; set when admin
                                            -- disables; preserved for
                                            -- audit (do not delete row)
```

- UUIDv7 IDs: time-ordered, good locality for B-tree indexes, no
  central allocator needed.
- `handle` is the login identity, immutable. `display_name` can change.
- **No `password_hash` column.** Credential material lives in the
  Authenticator's database. Adding a credential column here would
  defeat the privilege separation between services — see ADR 0004.
- We never delete users; we disable them. List ownership and item
  authorship references stay valid. Operator can hard-delete via a
  dedicated admin command that walks the cascade (rare, audited).
- **Disable-user coordination** (see ADR 0004 and ADR 0005): when an
  admin disables a user, two writes happen in sequence: Engine sets
  `disabled_at`, then Authenticator deletes all sessions and
  identities for that `user_id`. Either partial failure mode is
  safe: stale session → Engine rejects with `403 disabled`; deleted
  sessions but `disabled_at` unset → login fails because login
  flow asks Engine "is user X enabled?" before issuing a session.

Indexes: PK on `id`, UNIQUE on `handle`.

### lists

```
id             TEXT PRIMARY KEY              -- UUIDv7
owner_user_id  TEXT NOT NULL REFERENCES users(id)
title          TEXT NOT NULL                 -- 1-200 chars
version        INTEGER NOT NULL DEFAULT 1    -- envelope version; see below
created_at     INTEGER NOT NULL
archived_at    INTEGER                       -- nullable; soft-archive
last_purged_at INTEGER                       -- nullable; set when an item
                                             -- in this list is purged
                                             -- (see items / purge)
```

- Owner has implicit full control; no row in `list_members` needed.
- **`version` is the list's envelope version.** It is bumped on:
  title change, archive/unarchive, membership changes
  (add / remove / role change), and label vocabulary changes
  (create / update / delete a label on this list). It is **not**
  bumped by item mutations — items carry their own versions, and
  bumping the list row on every item edit would serialize writes to
  shared lists and destroy the signal's meaning. The envelope version
  serves two purposes:
  1. `If-Match` locking for list-metadata mutations (`PATCH`,
     `DELETE`, archive) — the `ETag` of a list is this version.
  2. A cheap change signal in the list index poll (see
     `sync-and-concurrency.md`): clients re-fetch metadata, members,
     and labels only when it moved.
- Archived lists stay queryable but are hidden from default views.
- A user always has at least one list (created at signup, title
  "Inbox"). Cannot be deleted while it's the only list.

Indexes: PK on `id`, INDEX on `owner_user_id`.

### list_members

```
list_id    TEXT NOT NULL REFERENCES lists(id) ON DELETE CASCADE
user_id    TEXT NOT NULL REFERENCES users(id)
role       TEXT NOT NULL CHECK (role IN ('reader', 'writer'))
added_at   INTEGER NOT NULL
added_by   TEXT NOT NULL REFERENCES users(id)
PRIMARY KEY (list_id, user_id)
```

- Owner is **not** stored here — owner role is implicit via
  `lists.owner_user_id`. Avoids two sources of truth.
- `reader`: can list and read items. `writer`: also create / update /
  delete items. Cannot change ACL — only the owner can.
- `added_by` retained for audit. Even if the adding user is later
  disabled, the membership remains valid.
- **No version column.** Membership rows are pure relations; their
  mutations are owner-only and unconditional (no `If-Match` —
  concurrent ACL edits from two devices of the same owner are rare
  enough that last-write-wins is acceptable). Every membership
  mutation **bumps the parent list's envelope `version`**, which is
  how other clients learn to re-fetch the member set.

Indexes: PK on `(list_id, user_id)`, INDEX on `user_id` for
"lists shared with me" queries.

### items

```
id          TEXT PRIMARY KEY                -- UUIDv7
list_id     TEXT NOT NULL REFERENCES lists(id) ON DELETE CASCADE
text        TEXT NOT NULL                   -- 1-65536 chars; bounded
                                            -- because all input must be
                                            -- validated (SECURITY.md),
                                            -- generous because tickets
                                            -- carry real text
done        INTEGER NOT NULL DEFAULT 0      -- 0/1 boolean
due_at      INTEGER                         -- nullable, unix seconds
priority    INTEGER                         -- nullable; 0-3, higher
                                            -- = more urgent
position    REAL NOT NULL                   -- sort key; see below
version     INTEGER NOT NULL DEFAULT 1      -- bumped on every mutation
created_at  INTEGER NOT NULL
updated_at  INTEGER NOT NULL                -- = created_at on insert
deleted_at  INTEGER                         -- nullable; soft delete
created_by  TEXT NOT NULL REFERENCES users(id)
```

- `version` is a monotonic counter bumped by the Engine on every
  mutation. Client must send the version it observed; if it doesn't
  match current, server returns `409 Conflict`. See
  `sync-and-concurrency.md`.
- `deleted_at`: soft delete. **Tombstones are retained indefinitely**
  — there is no background purge job. This gives us three things:
  offline clients can be away arbitrarily long and still receive
  deletion deltas; deleted items form a proper trash with restore
  (`POST /api/items/{id}/restore`, writer-permitted); and the sync
  protocol needs no retention-window reset case.
- **Purge** (`DELETE /api/items/{id}?purge=1`, owner-only) removes a
  row permanently — the escape hatch for "this must actually be
  gone" (privacy: otherwise anything ever deleted in a shared list
  stays recoverable by all members forever). Purging stamps the
  list's `last_purged_at`; sync tokens older than that timestamp get
  a forced re-fetch for the list (see `sync-and-concurrency.md`),
  because a purged item leaves no tombstone for stale clients.
- `position` is a fractional sort key (e.g. 1024.0, 1536.0, 1280.0)
  that lets us insert between two items without renumbering. Reindex
  to integers in a maintenance job when fractions get too tight (rare).
- `priority` semantics: NULL = unset, 0 = low, 1 = normal, 2 = high,
  3 = urgent. UI may render NULL identically to 1 but the distinction
  is preserved.

Indexes: PK on `id`, INDEX on `(list_id, deleted_at, position)` for the
main "items in list, sorted" query, INDEX on `(list_id, updated_at)`
for sync delta queries.

### labels

```
id          TEXT PRIMARY KEY                -- UUIDv7
list_id     TEXT NOT NULL REFERENCES lists(id) ON DELETE CASCADE
text        TEXT NOT NULL                   -- 1-64 chars, case-folded
                                            -- on insert/update
color       TEXT                            -- nullable; "#RRGGBB" hex
description TEXT                            -- nullable; user-visible
                                            -- note, max 280 chars
version     INTEGER NOT NULL DEFAULT 1      -- bumped on every mutation
created_at  INTEGER NOT NULL
created_by  TEXT NOT NULL REFERENCES users(id)
UNIQUE (list_id, text)
```

- **Labels are list-scoped.** The same text (`"job"`, `"optional"`,
  `"public"`) can exist as distinct labels on different lists with
  different colors, descriptions, and — once v2 policy attributes
  arrive — different semantics. This matches how labels are actually
  used: their meaning is contextual to the list they live in.
- A label belongs to its list. When the list is hard-deleted, its
  labels go with it. When an item is moved between lists (not in v1
  but anticipated), labels do not follow — the destination list has
  its own label namespace.
- Case-folded `text` so "Work" and "work" are the same label within a
  list. The `UNIQUE (list_id, text)` constraint enforces this.
- Color and description are present from v1 but nullable; UI can omit
  them. Hex format is enforced at the API boundary.
- **Two-level versioning**: each label carries its own `version` for
  `If-Match` locking on label mutations (two users recoloring
  *different* labels must not conflict with each other). In addition,
  every label create / update / delete bumps the parent list's
  envelope `version`, which is the signal that makes other clients
  re-fetch the label vocabulary (see `sync-and-concurrency.md`).
- This is a real table specifically so that future label-driven
  features (visibility policy like a `"public"` label, AI agent
  routing, automation triggers) can attach attributes here without a
  migration scramble. v1 does not implement those — see Open Items.

Indexes: PK on `id`, UNIQUE on `(list_id, text)` (also serves as the
"list all labels on a list" index).

### item_labels

```
item_id  TEXT NOT NULL REFERENCES items(id) ON DELETE CASCADE
label_id TEXT NOT NULL REFERENCES labels(id) ON DELETE CASCADE
PRIMARY KEY (item_id, label_id)
```

- Pure join table. Cascade on both sides: deleting an item or a label
  drops the association.
- **No version column — assignment changes ride the item's version.**
  Labels are attached/detached via `PATCH /api/items/{id}` (the
  `labels` array), which is an item mutation: it requires `If-Match`,
  bumps the item's `version`, and updates `updated_at`. Assignment
  changes therefore appear in the normal per-list item delta sync;
  clients never poll for them separately.
- An item can only carry labels from its own list (enforced by the
  application layer at attach time; the FK does not catch this on its
  own). Validation: `labels.list_id == items.list_id` must hold.

Indexes: PK on `(item_id, label_id)`, INDEX on `label_id` for
"items carrying label X" queries.

### sessions, invites, credentials, identities

**Not in the Engine's database.** These all live in the Authenticator's
separate SQLite file, per ADR 0004. The Engine does not store, query,
or know how to validate session tokens, passwords, OIDC linkages, or
invite tokens. The Engine learns "who is the caller" from the trusted
`X-Toodoo-User-Id` header injected by the Authenticator as part of the
request flow (see ADR 0005). Authoritative spec:
`docs/specs/authentication.md`.

### schema_version

```
version    INTEGER NOT NULL PRIMARY KEY
applied_at INTEGER NOT NULL
```

- Single-row pattern: Engine writes the new version after a successful
  migration. Used by `toodoo-restore` to refuse restoring a backup
  whose schema is newer than the installed binary supports, and by
  the Engine on startup to refuse running against a too-new DB.

## ID Generation

UUIDv7 everywhere in the Engine's schema. Random opaque tokens (for
sessions and invites) live in the Authenticator's schema, not here.
Rationale for UUIDv7:

- Time-ordered: new rows cluster at the end of the B-tree.
- 128-bit globally unique: clients can mint IDs offline without
  collision risk. The Engine still validates format and rejects
  invalid IDs.
- No autoincrement: simplifies replication, sharding, and offline-first
  client behavior.

## Timestamps

All timestamp columns: `INTEGER`, unix seconds (not milliseconds — we
have no use case requiring sub-second precision and integer seconds
read cleanly in SQLite). Always UTC. Display localization is the
client's job.

## Foreign Keys and Cascade Behavior

- `lists.owner_user_id` → users(id): no cascade. We never delete users.
- `list_members.list_id` → lists(id) `ON DELETE CASCADE`: when a list
  is hard-deleted, its ACL goes with it.
- `list_members.user_id` → users(id): no cascade.
- `items.list_id` → lists(id) `ON DELETE CASCADE`.
- `item_labels.item_id` → items(id) `ON DELETE CASCADE`.
- `item_labels.label_id` → labels(id) `ON DELETE CASCADE`.
- `labels.list_id` → lists(id) `ON DELETE CASCADE`.
- `labels.created_by` → users(id): no cascade.
- `items.created_by` → users(id): no cascade.

The Authenticator's database has its own FK graph linking back to
`users.id` as a logical (cross-database) reference; see
`docs/specs/authentication.md`. The Engine does not enforce or even
know about those references.

SQLite requires `PRAGMA foreign_keys = ON` per connection — the
adapter must set this on every open.

## Validation Rules

Per `docs/SECURITY.md` ("trust nothing, verify all input") the Engine
re-validates everything it reads from the database, not just user input:

- All TEXT fields: length bounds enforced at the boundary.
- All ID fields: UUIDv7 format validation.
- `role`: only `'reader'` or `'writer'`.
- `priority`: NULL or 0..3.
- Timestamps: positive integers, sanity-checked against a far-future
  bound (rejects nonsense like year 9999).

Bad data is logged and the relevant request fails closed with a 500;
we do not silently coerce.

## Migrations

- Migrations live in `internal/engine/storage/migrations/` as numbered
  `.sql` files (`001_initial.sql`, `002_add_x.sql`, ...).
- Engine runs pending migrations at startup, in a single transaction
  per file.
- Additive-only by default. Destructive migrations (drop column, drop
  table, change constraint that loses data) require setting an explicit
  config flag (`storage.allow_destructive_migration = true`) which the
  Debian postinst sets after a debconf warning.
- Each migration bumps `schema_version`.

## Open Items

- Whether to keep item history (audit log of mutations) for v1, or
  defer to v2. Cost: doubles write volume. Benefit: undo, blame.
  Tentative: defer.
- Attachments (BLOBs) mentioned in `docs/NOTES.md`. Tentative: defer to
  v2 with a separate `attachments` table and content-addressed
  filesystem storage.
- Comments / notes on items, mentioned in `docs/NOTES.md`. Tentative:
  defer to v2.
- Label policy attributes (v2): the `labels` table is intended to
  carry future columns such as `visibility` (e.g. a `"public"` label
  that exposes items more broadly than the list ACL), `ai_route`
  (which AI agent handles items with this label), or `policy_json`
  (extensible bag). v1 ships only `color` and `description`; later
  additions are pure schema migrations on the existing table.
