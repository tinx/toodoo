# Spec: Debian Packaging

This spec defines the `.deb` packages, their file layout, maintainer
scripts, and upgrade/downgrade behavior. Required reading: ADR 0003
(distribution decision, Ubuntu 24.04 LTS floor), ADR 0002 (systemd
units instead of containers), ADR 0006 (static groups).

## Packages

| Package         | Contents                                              |
|-----------------|-------------------------------------------------------|
| `toodoo-server` | service binaries, unit files, config, fail2ban filters, LE hook, `toodoo-backup`/`toodoo-restore` CLIs |
| `toodoo-cli`    | the `toodoo` end-user/admin CLI binary only           |

`toodoo-cli` installs independently (a member's laptop needs no
server bits). `toodoo-server` does not depend on `toodoo-cli`, but
`Recommends:` it — admin commands are issued through it.

### Control fields (toodoo-server)

```
Depends:  systemd (>= 255), fail2ban, certbot, adduser
Recommends: toodoo-cli, gnupg
Architecture: amd64 arm64
```

- `adduser` provides `addgroup --system` for postinst.
- `gnupg` backs the encrypted-by-default backups
  (`backup-restore.md`); Recommends rather than Depends so a
  no-backup evaluation install stays minimal.
- No podman, no container runtime, no node.js anywhere (ADR 0002,
  `docs/CODING.md`).

## File Layout

```
/usr/libexec/toodoo/listener            # service binaries, not in PATH
/usr/libexec/toodoo/signer
/usr/libexec/toodoo/authenticator
/usr/libexec/toodoo/engine
/usr/bin/toodoo-backup                  # host CLIs (toodoo-server)
/usr/bin/toodoo-restore
/usr/bin/toodoo                         # from toodoo-cli
/usr/lib/systemd/system/toodoo-*.service
/usr/lib/systemd/system/toodoo-*.socket
/usr/lib/systemd/system/toodoo-backup.{service,timer}   # disabled by default
/etc/toodoo/config.toml                 # conffile
/etc/toodoo/tls/                        # cert deploy target (LE hook)
/etc/toodoo/tls/private/                # key dir, root-owned 0700 — see note
/etc/letsencrypt/renewal-hooks/deploy/toodoo
/etc/fail2ban/filter.d/toodoo.conf      # conffile
/etc/fail2ban/jail.d/toodoo.conf        # conffile
/usr/share/doc/toodoo-server/README.Debian
/var/lib/toodoo-engine/                 # created by systemd StateDirectory=
/var/lib/toodoo-authenticator/          #   not by the package
/var/lib/toodoo-backups/                # created by backup service unit
```

Web assets are **not** shipped as files — they are embedded in the
authenticator binary via `embed.FS` (ADR 0005).

Note on the TLS key directory: `StateDirectory`-style dynamic
ownership doesn't apply to `/etc`. The signer unit reaches the key
via `BindReadOnlyPaths=/etc/toodoo/tls/private`; the directory itself
is root-owned `0700` on disk — the bind into the sandbox is what
grants the signer (and only the signer) read access. No group or ACL
gymnastics needed.

## Maintainer Scripts

### postinst (configure)

Fresh install:

1. Create static groups, idempotently:
   `toodoo-ipc-auth`, `toodoo-ipc-engine`, `toodoo-ipc-signer`,
   `toodoo-admin`, `toodoo-backup` — all via
   `addgroup --system <name>`.
2. `systemctl daemon-reload`.
3. `systemctl enable --now` the four `.socket` units and the four
   `.service` units. (Socket activation means services start lazily;
   enabling them eagerly anyway gives the operator immediate
   feedback that the install works.)
4. Print a post-install hint: run `certbot certonly`, add yourself
   to `toodoo-admin`, create the first invite. (Full text in
   `README.Debian`.)

Upgrade (`postinst configure <old-version>`):

1. Services were stopped by `prerm` (below).
2. `systemctl daemon-reload`; re-enable any new units.
3. Start services in dependency order: signer, engine,
   authenticator, listener. The Engine and Authenticator run their
   own schema migrations at startup (see Migration Gating).
4. If a service fails to start, abort loudly with `journalctl`
   pointers — do not mask a failed migration behind a "successfully
   installed" message.

No user creation anywhere: `DynamicUser=` handles service identities
(ADR 0002); only the five static groups are package-managed.

### prerm (upgrade or remove)

Stop services in reverse dependency order: listener, authenticator,
engine, signer (and their sockets, so nothing re-activates them
mid-upgrade).

### postrm

- `remove`: disable units; leave `/etc/toodoo/`, `/var/lib/toodoo-*`
  and the groups in place — a re-install picks up where it left off.
- `purge`: delete `/var/lib/toodoo-engine/`,
  `/var/lib/toodoo-authenticator/`, `/var/lib/toodoo-backups/` and
  `/etc/toodoo/`. **Purge destroys both databases**; the package
  description and `README.Debian` warn about this prominently.
  Static groups are left in place even on purge (re-creating groups
  risks GID reuse; orphaned system groups are harmless and standard
  practice).

### Conffiles

`/etc/toodoo/config.toml` and the two fail2ban files are dpkg
conffiles: operator edits are preserved across upgrades, conflicts
surface through the standard dpkg prompt. Unit files are **not**
conffiles — operators customize them with systemd drop-ins
(`systemctl edit toodoo-engine`), which survive upgrades natively,
rather than by editing shipped units.

## Migration Gating

- Engine and Authenticator each own their schema (separate
  `schema_version` tables, per ADR 0004) and migrate themselves at
  startup, additive-only by default.
- **Destructive migrations** (drop/rewrite that loses data) refuse to
  run unless `storage.allow_destructive_migration = true` is set in
  `config.toml`. When a release contains one, postinst raises a
  debconf prompt (priority high) explaining what will be lost and
  offering to set the flag; declining leaves services stopped with
  clear instructions. Releases avoid destructive migrations except
  at major versions.
- **Downgrade refusal**: each service compares the on-disk
  `schema_version` with its supported range at startup and refuses
  to start if the disk is newer (the data, not the package, is the
  authority). `apt` will happily install an older version; the
  service is the gate. Recovery from a bad upgrade is
  `toodoo-restore` from the pre-upgrade backup, not a binary
  downgrade.
- postinst suggests (does not run) a `toodoo-backup` invocation
  before major-version upgrades, via debconf note.

## Versioning

- Package: semver `MAJOR.MINOR.PATCH`.
- Schema versions (both DBs) are internal integers, independent of
  the package version; a package supports a contiguous range of
  schema versions, recorded in the binaries and surfaced in
  `toodoo --version` / backup manifests.

## Release Pipeline

- CI builds both packages reproducibly: pinned Go toolchain,
  `CGO_ENABLED=0`, `-trimpath`, fixed `SOURCE_DATE_EPOCH` from the
  git commit.
- The `.deb`s are lintian-clean (CI gate) and signed; they are
  published to an apt repository whose GPG key is documented as the
  trust root. (Repository hosting choice — managed service vs.
  GitHub-Releases-backed static repo — is an implementation decision;
  the spec requirement is: `apt install` + `apt upgrade` must work
  with signature verification, no `curl | sh`.)
- Build tooling: native `debhelper` (`dh`) is the default choice —
  it is the boring, well-trodden path and keeps us lintian-friendly.
  If it proves heavier than the project's appetite, `nfpm` is the
  fallback; the spec constrains the *artifact behavior*, not the
  build tool.

## README.Debian (operator quick start)

Shipped content covers, in order:

1. `sudo apt install toodoo-server toodoo-cli`
2. `sudo certbot certonly --standalone -d todo.example.org`
   (or the operator's preferred ACME flow), and what the deploy hook
   does from then on.
3. `sudo usermod -aG toodoo-admin $USER` (+ re-login) — per ADR
   0006, no root needed afterwards.
4. `toodoo admin invite create --note "me"` → sign up in the web UI.
5. Enabling nightly encrypted backups:
   set `backup.gpg_recipient` in `config.toml`,
   `sudo systemctl enable --now toodoo-backup.timer`
   (see `backup-restore.md`).
6. The purge warning.

## Verification

- Scenario (d)/(f) walkthroughs from the plan: fresh install on a
  clean 24.04 VM; upgrade with an additive migration; both fully
  unattended.
- `piuparts` (install/upgrade/remove/purge cycles) in CI.
- The negative isolation tests and exposure-score ratchet from
  `service-hardening.md` run against the installed package, not
  loose files — packaging bugs (wrong mode, missing group) surface
  as boundary-test failures.

## Open Items

- Whether `toodoo-server` should be split further (`toodoo-engine`,
  `toodoo-listener`, ...) for partial deployments. No identified
  need; revisit only if someone wants an Engine-only install behind
  their own proxy.
- apt repository hosting choice (managed vs. static-on-GH).
- Debian-proper (non-Ubuntu) builds explicitly deferred (ADR 0003);
  the only expected blocker is the systemd version floor.
