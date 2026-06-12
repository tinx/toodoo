# ADR 0006: Application Administration via Unix Groups, Not Root

**Status:** Accepted
**Date:** 2026-06-12
**Related:** ADR 0004 (Authenticator owns its own datastore),
ADR 0005 (request flow; admin sockets), `docs/specs/authentication.md`

## Context

The Authenticator and the Engine each expose an admin Unix socket for
out-of-band operations: invite creation, user disable/enable, password
reset, session revocation, list reassignment, hard delete, and backup
snapshot streaming. An earlier draft made these sockets reachable by
`root` only (`SocketMode=0600`, owned by root).

That model has two practical problems:

1. **Role conflation.** It forces the server's root operator and the
   application's administrator to be the same person. In practice
   they rarely are: the person who patches the host is not necessarily
   the person who invites new club members and resets forgotten
   passwords. Root-only admin access either blocks the app admin or
   pressures the operator to hand out root.
2. **Routine root usage.** Day-to-day app administration (an invite a
   week, an occasional password reset) would require a root shell
   every time. Frequent root logins are a risk in themselves — every
   session is an opportunity for a fat-fingered command with
   unlimited blast radius, and broad root usage erodes the value of
   the audit trail.

The system already has an established pattern for access control via
filesystem permissions: the inter-service sockets are guarded by
static Unix groups (`toodoo-ipc-*`) with `SocketGroup=` /
`SocketMode=0660` / `SupplementaryGroups=` (per ADR 0005). Admin
access can use the same mechanism.

## Decision

**Admin access is granted via membership in static Unix groups, not
via root.** The filesystem permission on the socket remains the
security boundary; no application-level admin authentication is
added.

Two groups, two sockets per service, split by power level:

| Group           | Socket                | Operations                                              |
|-----------------|-----------------------|---------------------------------------------------------|
| `toodoo-admin`  | `admin.sock` (per service)  | invites, user disable/enable, password reset, session revocation, list reassignment, hard user delete |
| `toodoo-backup` | `backup.sock` (per service) | consistent snapshot streaming (SQLite online backup) — nothing else |

- Both the Authenticator and the Engine expose both sockets
  (`SocketMode=0660`, `SocketGroup=` as above), defined as `.socket`
  units like all other sockets in the system.
- The groups are static, created by the package postinst alongside
  the `toodoo-ipc-*` groups.
- The operator grants application-admin rights with
  `sudo usermod -aG toodoo-admin <user>` — once, not per session.
  Backup automation (the systemd backup timer's service unit) runs
  with `SupplementaryGroups=toodoo-backup` and nothing more.
- Root retains implicit access: file permissions never bind root.
  Nothing is lost relative to the old model; the new model is a
  strict superset.
- Every admin operation is logged with the caller's UID (and
  resolved username) obtained from `SO_PEERCRED` — the kernel-
  verified peer credential, not anything the client claims.

### Why the admin/backup split

The two power levels are qualitatively different:

- `toodoo-admin` is **day-to-day application administration** —
  managing people. Its most dangerous power is resetting a password
  (account takeover of one user at a time, visibly logged).
- `toodoo-backup` is **bulk data export** — the snapshot contains
  both databases in full: every list, every item, all credential
  hashes, all (hashed) session tokens.

A backup automation account that can also reset passwords is an
unnecessarily attractive target; an app admin who automatically gets
silent bulk export is an unnecessarily broad grant. Separating the
groups keeps each principal at its minimum. An operator who wants one
person to hold both simply adds them to both groups.

## Consequences

### Positive

- App administration without root: the club secretary can run
  `toodoo admin invite create` from their own account.
- Root shells become rare events again, and the audit log
  distinguishes *who* performed each admin action (per-user UIDs
  instead of a shared root).
- Backup automation runs with exactly one capability: read a
  consistent snapshot.
- Same mechanism as everything else in the system — one more
  application of the static-group + socket-permission pattern, no
  new security machinery to reason about.

### Negative (accepted)

- **Group membership is coarse.** Everyone in `toodoo-admin` can do
  everything on the admin socket; there is no per-operation
  granularity within the group. Acceptable at small-group scale; if
  finer roles are ever needed, the admin socket protocol can add
  application-level checks without changing the transport.
- **Membership is persistent** (until revoked), unlike a sudo
  session. This is the intended trade — but it means revoking an
  admin requires remembering to `gpasswd -d`. Documented in the
  operator guide.
- **A compromised admin-group account is an application-level
  compromise** (user management, not host). Strictly better than the
  alternative, where the same compromise would have required — and
  therefore exposed — root.
- Group changes take effect at next login (supplementary groups are
  evaluated at session setup). Operators must re-login after
  `usermod`. Minor, documented.

### Not affected

- The inter-service socket topology and trust contract (ADR 0005).
- The network-facing API: admin sockets remain unreachable from the
  network, unchanged.
- The two-database separation (ADR 0004): the backup socket on each
  service streams only that service's own database; the
  `toodoo-backup` CLI assembles the combined archive client-side.

## References

- ADR 0004 — datastore separation; what a backup snapshot contains
- ADR 0005 — socket topology, static-group access mechanism
- `docs/specs/authentication.md` — Admin Socket section (operations
  per socket)
- `docs/specs/backup-restore.md` (to be written) — backup CLI and
  timer unit using `toodoo-backup`
- `docs/specs/packaging-debian.md` (to be written) — group creation
  in postinst
- Plan: `/root/.claude/plans/i-m-not-sure-e2e-whimsical-rocket.md`
