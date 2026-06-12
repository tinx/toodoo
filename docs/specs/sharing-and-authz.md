# Spec: Sharing and Authorization

This spec defines how lists are shared between users and how the Engine
authorizes every read and write. Required reading first:
`docs/specs/data-model.md` (defines `lists`, `list_members`, `items`).

## Model

The unit of sharing is the **list**. Items inherit their access from
the list they belong to. There is no per-item ACL.

Each list has exactly one **owner** (the user in `lists.owner_user_id`).
The owner has full control. Other users gain access through rows in
`list_members`:

- `role = 'reader'`: can read everything in the list, cannot mutate.
- `role = 'writer'`: read + create / update / delete items.

No other roles in v1. Specifically, no "admin-but-not-owner" role —
delegating ACL management would require a `manager` role and we don't
need that complexity for the target audience.

Ownership is **not transferable in v1**. If the owner leaves the group,
the operator runs `toodoo admin list reassign --list <id> --to <user>`.

## Authorization Rules

Every endpoint that touches list-scoped data must answer
"can `requesting_user` perform `action` on `list_id`?" The check is
implemented once in a single `authorize(user, list_id, action)`
function used by every handler — no scattered ad-hoc checks.

| Action                            | Owner | Writer | Reader | Other |
|-----------------------------------|-------|--------|--------|-------|
| Read list metadata + items        | yes   | yes    | yes    | no    |
| Create / update / delete items    | yes   | yes    | no     | no    |
| Restore a deleted item            | yes   | yes    | no     | no    |
| Purge an item permanently         | yes   | no     | no     | no    |
| Read labels on the list           | yes   | yes    | yes    | no    |
| Create / update / delete labels   | yes   | yes    | no     | no    |
| Attach / detach labels on items   | yes   | yes    | no     | no    |
| Update list metadata (title)      | yes   | no     | no     | no    |
| Archive / unarchive list          | yes   | no     | no     | no    |
| Add / remove members              | yes   | no     | no     | no    |
| Change member roles               | yes   | no     | no     | no    |
| Delete list                       | yes   | no     | no     | no    |

Note on labels: in v1 they follow the same role check as items. When
label policy attributes arrive in v2 (e.g. a `visibility` field that
makes labels themselves consequential to access control), this table
will need a separate row for "modify label policy" gated to the owner
only — writers can manage label *vocabulary* but should not be able to
change label *semantics*. Out of scope until then.

"Other" includes disabled users (who additionally fail the session
check before reaching authz).

### Implementation Note

```go
// pseudocode
func authorize(user UserID, listID ListID, action Action) error {
    list, err := storage.GetList(listID)
    if err != nil { return err }
    if user == list.OwnerUserID {
        return nil  // owner can do anything
    }
    member, err := storage.GetListMember(listID, user)
    if errors.Is(err, ErrNotFound) {
        return ErrForbidden  // not a member
    }
    if err != nil { return err }
    return checkRole(member.Role, action)
}
```

The check happens **after** the request is parsed and validated but
**before** any state mutation. `ErrForbidden` maps to HTTP 403;
`ErrNotFound` from `GetList` maps to 404 only if the caller has no
visibility into the list's existence at all (i.e., they're also not
the owner). This prevents probing for list IDs.

## API Surface for Sharing

Membership mutations are **unconditional** (no `If-Match`): they are
owner-only, so the only possible conflict is the same owner acting
from two devices, where last-write-wins is acceptable. Every
membership mutation bumps the list's envelope `version` (see
`data-model.md`), which is how other clients learn to re-fetch the
member set via the index poll.

### List members of a list

```http
GET /api/lists/{id}/members
→ 200 OK
[
  { "user": { "id": "...", "handle": "alice", "display_name": "Alice" },
    "role": "owner", "added_at": null, "added_by": null },
  { "user": { ... }, "role": "writer", "added_at": 1719..., "added_by": "..." }
]
```

Owner is included in the response with `role: "owner"` for UI
convenience, even though the storage representation differs.

Allowed for: owner, writers, readers (anyone who can see the list can
see who else can see it).

### Add a member

```http
POST /api/lists/{id}/members
{ "handle": "bob", "role": "writer" }
→ 201 Created
```

Allowed for: owner only.

- The request uses `handle` (not user ID) so the owner shares by name
  the way they think of people. The Engine resolves it.
- If the handle doesn't exist: `404` with `error: "user_not_found"`. We
  do **not** disclose user existence to anyone else (see "Information
  Disclosure" below); this is allowed only when the requester is the
  list owner, which is a deliberate disclosure boundary.
- If the user is already a member: `409` with `error:
  "already_member"`.
- The owner cannot add themselves (they're already implicit owner).

### Change a member's role

```http
PATCH /api/lists/{id}/members/{user_id}
{ "role": "reader" }
→ 200 OK
```

Allowed for: owner only.

### Remove a member

```http
DELETE /api/lists/{id}/members/{user_id}
→ 204 No Content
```

Allowed for: owner only, **or** the member themselves (self-removal:
"leave this list").

## Information Disclosure Rules

Per `docs/SECURITY.md` ("trust nothing, verify all input") we also
take care not to leak information through error codes or response
shapes.

- **User existence:** only disclosed to list owners, via the
  "add member" endpoint when they specify a handle. The handle-search
  endpoint (autocomplete) is owner-of-some-list only and rate-limited.
  An unauthenticated user cannot probe for handles.
- **List existence:** disclosed only to members. A non-member GETting
  a list ID gets 404, not 403, regardless of whether the list exists.
- **Membership lists:** visible to all members (so collaborators know
  who they're collaborating with). Not visible to anyone else.
- **Audit fields** (`added_by`, `added_at`): visible to all members.

## Disabled Users and ACLs

When a user is disabled (`users.disabled_at` set):

- Their sessions are killed immediately.
- They cannot authenticate.
- Their `list_members` rows are **preserved** (so re-enabling restores
  access cleanly).
- Lists they own are **not** reassigned automatically. An operator
  must run the admin reassign command, which fails loudly if there are
  owned lists that need attention. Rationale: we'd rather block on
  human review than silently transfer ownership.

## Hard Delete (Admin)

`toodoo admin user delete --handle <name>` is the only path to fully
remove a user. It refuses unless:

1. The user is disabled.
2. They own no lists (or `--force-reassign-owned-lists-to <handle>` is
   passed).
3. The operator types the handle twice for confirmation (CLI).

On execution:

- Reassigns any remaining owned lists if forced.
- Deletes `list_members` rows.
- Deletes the `users` row.

Items they created remain (the `items.created_by` FK is `NO ACTION`),
authored attribution becomes "unknown user." This is a deliberate
choice: deleting a member should not delete the team's work.

## Open Items

- Sharing a list with "all members of this instance" as a single
  pseudo-member. Useful for clubs where everyone should see the
  shared list by default. Defer; can be implemented as a "share with
  all current users" admin action when needed.
- Per-item assignee field (separate from creator) — defer to v2.
- Public read-only links (shareable URL anyone can view without
  signing in). Not in scope for v1 — would punch a hole in the
  "must be authenticated" model that's worth designing carefully.
