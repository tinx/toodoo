# Spec: Backup and Restore

This spec defines the `toodoo-backup` and `toodoo-restore` host CLIs,
the archive format, the scheduled-backup units, and the safety rules.
Required reading: ADR 0004 (two databases), ADR 0006 (the
`toodoo-backup` group and `backup.sock`).

## Principles

- **Services are the gatekeepers.** Backup never reads database files
  from disk — it asks each service for a consistent snapshot over its
  `backup.sock` (SQLite online backup API serialized over the
  socket). The services' `StateDirectory=` stays private to their
  dynamic UIDs; no permission carve-outs for backup tooling.
- **Backup is routine and unprivileged** — any member of the
  `toodoo-backup` group (ADR 0006). **Restore is rare and
  privileged** — it must stop services and write into
  `StateDirectory=` trees owned by dynamic UIDs, so it requires
  root. This asymmetry is intentional: the operation you run nightly
  needs no root; the disaster-recovery operation you run twice a
  year may reasonably demand it.
- **Encrypted by default.** Backups contain everything: both
  databases in full (all lists and items, argon2id password hashes,
  hashed session tokens) plus the config. `toodoo-backup` refuses to
  produce plaintext output unless `--plaintext` is passed explicitly.

## toodoo-backup

```
toodoo-backup [--output DIR] [--recipient KEYID] [--plaintext]
              [--verify FILE] [--prune KEEP]
```

Default behavior (no flags): create
`toodoo-backup-<UTC timestamp>.tar.gpg` in the current directory,
encrypted to the GPG recipient configured as `backup.gpg_recipient`
in `/etc/toodoo/config.toml`. If neither config nor `--recipient`
provides a key and `--plaintext` is absent: **error out** with a
message explaining both options.

### Archive contents

```
manifest.json
engine.sqlite          # snapshot streamed from engine-backup.sock
auth.sqlite            # snapshot streamed from authenticator-backup.sock
config.toml            # /etc/toodoo/config.toml as installed
```

`config.toml` is included because restore should reproduce a working
instance, not just data — but note it may contain operator-sensitive
values (one more reason encryption is the default). TLS keys and
certs are **not** included: Let's Encrypt re-issues them on the new
host, and private keys should not travel in backups.

### manifest.json

```json
{
  "format_version": 1,
  "created_at": 1781280000,
  "toodoo_version": "1.4.2",
  "engine_schema": 7,
  "auth_schema": 3,
  "files": [
    { "name": "engine.sqlite", "sha256": "...", "bytes": 1234567 },
    { "name": "auth.sqlite",   "sha256": "...", "bytes": 234567 },
    { "name": "config.toml",   "sha256": "...", "bytes": 2345 }
  ]
}
```

Schema versions come from each service's snapshot response (the
service reports its own `schema_version` alongside the stream), not
from inspecting the files — the services are the authority.

### Cross-database consistency

The two snapshots are taken sequentially, not atomically. The only
state that spans both databases is the signup/disable coordination
from ADR 0004, which was designed with safe partial states — so a
backup can, at worst, capture an Engine user whose Authenticator
identity was created milliseconds later (a "ghost user"). This is
the same state as a crashed signup and is handled the same way: the
signup retry path or the admin reconcile command
(`toodoo admin reconcile`, which lists Engine users without
identities and offers to complete or remove them). The backup order
is fixed — **Engine first, then Authenticator** — so the ghost-user
direction is the recoverable one (business row without credentials,
never credentials pointing at a missing user).

### --verify

Decrypts (if applicable), checks every file against the manifest
hashes, validates the manifest schema, prints versions. Does not
touch the running services. Run it from the backup timer after each
backup, and run it before every restore.

### --prune KEEP

Deletes the oldest backups in `--output DIR` beyond the newest KEEP
(default 14 when invoked by the timer). Refuses to delete files that
don't match the `toodoo-backup-*.tar(.gpg)` pattern.

## Scheduled Backups

Shipped disabled; the operator opts in
(`systemctl enable --now toodoo-backup.timer`).

```ini
# toodoo-backup.timer
[Timer]
OnCalendar=*-*-* 03:14:00
RandomizedDelaySec=30m
Persistent=true

# toodoo-backup.service (oneshot)
[Service]
Type=oneshot
ExecStart=/usr/bin/toodoo-backup --output /var/lib/toodoo-backups --prune 14
DynamicUser=yes
SupplementaryGroups=toodoo-backup
StateDirectory=toodoo-backups
# + the full hardening baseline from service-hardening.md
```

The timer's service runs with exactly one power: the
`toodoo-backup` group (snapshot streaming). It cannot reset
passwords, cannot read the live database files, cannot reach the
network. Off-site transport of the (encrypted) archives is the
operator's choice and out of scope — `rsync`, object storage, a USB
stick; the GPG layer is what makes all of those acceptable.

## toodoo-restore

```
sudo toodoo-restore <archive> [--force] [--skip-config]
```

Steps, in order; any failure aborts before changes are made
(verification is complete before the first write):

1. Verify the archive (`--verify` logic; for encrypted archives the
   invoking user's GPG keyring must hold the secret key).
2. **Refuse if the backup's schema versions are newer** than the
   installed binaries support (restoring data from the future cannot
   work; upgrade the package first). Older-schema backups are fine —
   services migrate forward on next start.
3. Refuse if existing databases are present unless `--force` is
   given (protects against fat-fingering a restore over a live
   instance; an empty fresh install proceeds without `--force`).
4. Stop all toodoo services and sockets (reverse dependency order).
5. Write `engine.sqlite` and `auth.sqlite` into the respective
   `StateDirectory=` trees. Ownership note: with `DynamicUser=`,
   systemd re-chowns `StateDirectory=` contents to the service's
   UID at next start, so root-written files are adopted correctly.
6. Restore `config.toml` unless `--skip-config` (a host migration
   often wants the data but a freshly edited config).
7. Start services in dependency order; each runs its forward
   migrations if the backup was older than the binaries.
8. Print a post-restore checklist: re-run `certbot` if this is a new
   host, verify login, verify fail2ban is active.

### What restore explicitly does not do

- Does not restore TLS keys/certs (not in the archive).
- Does not downgrade packages or schemas (step 2).
- Does not merge: restore replaces the instance state wholesale.
  Partial/selective restore (one list, one user) is a v2 question —
  it is an application-level import feature, not a backup feature.

## Failure Modes

| Failure                                  | Behavior                                        |
|------------------------------------------|--------------------------------------------------|
| A service is down during backup          | Backup fails loudly (snapshot socket unreachable); timer reports failure; no partial archive is left behind (output is written to a temp name and renamed only on success). |
| GPG recipient key missing/expired        | Backup errors before contacting any service.     |
| Archive corrupted                        | `--verify` / restore step 1 catches it via manifest hashes. |
| Backup newer than installed binaries     | Restore refuses at step 2 with the exact versions and the required package version. |
| Power loss mid-restore                   | Services were stopped first; re-running the restore from step 1 is always safe (writes are idempotent file replacements). |
| Ghost user captured in backup            | `toodoo admin reconcile` after restore (see Cross-database consistency). |

## Verification

- Plan scenario (e): backup on host A, fresh `.deb` install on host
  B, restore, assert identical state (item counts, a sample of
  ETags, login works) and that the manifest versions were enforced.
- Integration test: corrupt one byte of an archive → `--verify`
  fails; truncate the auth snapshot → restore aborts before stopping
  services.
- Integration test: restore a backup taken on schema N into binaries
  at N+1 → services migrate forward; restore N+1 into N → refused.
- The timer's service unit passes the same exposure-score gate as
  the four main services.
