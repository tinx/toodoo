# Spec: Service Hardening (systemd Units)

This spec defines the systemd `.service` and `.socket` units for all
four services, the common hardening baseline, per-service refinements,
the socket topology, and the CI gates that keep all of it honest.
Required reading: ADR 0002 (systemd-native isolation), ADR 0005
(request flow, socket trust contract), ADR 0006 (admin/backup groups).

## Static Groups

Created by the package postinst (see `packaging-debian.md`), never
removed on upgrade:

| Group               | Grants access to                  | Members                       |
|---------------------|-----------------------------------|-------------------------------|
| `toodoo-ipc-auth`   | Authenticator main socket         | Listener service only         |
| `toodoo-ipc-engine` | Engine main socket                | Authenticator service only    |
| `toodoo-ipc-signer` | Signer socket                     | Listener service only         |
| `toodoo-admin`      | admin.sock (both services)        | operator-chosen local users   |
| `toodoo-backup`     | backup.sock (both services)       | backup timer + chosen users   |

Static groups are required because `DynamicUser=` UIDs are ephemeral
and cannot appear in ownership rules ahead of time. Service-to-group
membership is granted with `SupplementaryGroups=` in the unit;
human-to-group membership with `usermod -aG` (ADR 0006).

## Socket Topology

All sockets are systemd socket-activated, live under `/run/toodoo/`,
and are owned root with `SocketGroup=`/`SocketMode=0660` so that
exactly one group can connect. Services receive them as inherited
file descriptors, distinguished by `FileDescriptorName=`.

```
toodoo-listener.socket             TCP :443 (host network namespace)
toodoo-signer.socket               /run/toodoo/signer.sock        (toodoo-ipc-signer)
toodoo-authenticator.socket        /run/toodoo/authenticator.sock (toodoo-ipc-auth)
toodoo-authenticator-admin.socket  /run/toodoo/authenticator-admin.sock  (toodoo-admin)
toodoo-authenticator-backup.socket /run/toodoo/authenticator-backup.sock (toodoo-backup)
toodoo-engine.socket               /run/toodoo/engine.sock        (toodoo-ipc-engine)
toodoo-engine-admin.socket         /run/toodoo/engine-admin.sock  (toodoo-admin)
toodoo-engine-backup.socket        /run/toodoo/engine-backup.sock (toodoo-backup)
```

Connection graph (who dials whom):

```
network → toodoo-listener.socket → Listener
Listener  → signer.sock          (TLS private-key ops, gRPC)
Listener  → authenticator.sock   (PROXY v2 header + cleartext HTTP)
Authenticator → engine.sock      (proxied requests + /internal/* RPCs)
admin CLI  → *-admin.sock        (group toodoo-admin)
backup CLI → *-backup.sock       (group toodoo-backup)
```

Example socket unit (pattern is identical for all Unix sockets):

```ini
# toodoo-engine.socket
[Unit]
Description=TooDoo Engine main socket

[Socket]
ListenStream=/run/toodoo/engine.sock
FileDescriptorName=main
SocketUser=root
SocketGroup=toodoo-ipc-engine
SocketMode=0660
DirectoryMode=0755
Service=toodoo-engine.service

[Install]
WantedBy=sockets.target
```

The Listener's TCP socket:

```ini
# toodoo-listener.socket
[Unit]
Description=TooDoo public TLS endpoint

[Socket]
ListenStream=443
BindIPv6Only=both
Backlog=256
FileDescriptorName=public
Service=toodoo-listener.service

[Install]
WantedBy=sockets.target
```

systemd (PID 1) opens `:443`; the Listener itself runs with
`PrivateNetwork=yes` and never creates a network socket. `AF_UNIX`
connections to the signer and authenticator sockets work from inside
a private network namespace because Unix sockets are filesystem
objects, not network objects.

## Common Hardening Baseline

Applied to **every** service. Shipped unit files are generated from a
single template at build time so the baseline cannot drift between
services.

```ini
[Service]
DynamicUser=yes
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
PrivateIPC=yes
PrivateUsers=yes
ProtectProc=invisible
ProcSubset=pid
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectClock=yes
ProtectHostname=yes
ProtectControlGroups=yes
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
RemoveIPC=yes
KeyringMode=private
UMask=0077
CapabilityBoundingSet=
AmbientCapabilities=
SystemCallArchitectures=native
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources @mount @debug @cpu-emulation @obsolete @swap
Restart=on-failure
RestartSec=2
```

Notes:

- `MemoryDenyWriteExecute=yes` is safe for pure Go (no JIT, no cgo —
  and `CGO_ENABLED=0` is enforced in CI).
- `DynamicUser=yes` gives each service its own ephemeral UID; the
  per-service `StateDirectory=` is chowned to it automatically, which
  is the mechanical enforcement of ADR 0004's data separation.
- A crash under attack restarts cleanly (`Restart=on-failure`); a
  service that flaps hits systemd's default start-limit and stays
  down loudly rather than looping silently.

## Per-Service Units

Only the deltas from the baseline are shown.

### toodoo-listener.service

The most exposed service (parses TLS from the network) and therefore
the most constrained one. Holds all TLS state (ADR 0005); links
`crypto/tls`; does nothing else.

```ini
[Unit]
Description=TooDoo Listener (TLS termination)
Requires=toodoo-listener.socket toodoo-signer.socket toodoo-authenticator.socket
After=toodoo-signer.socket toodoo-authenticator.socket

[Service]
ExecStart=/usr/libexec/toodoo/listener
Sockets=toodoo-listener.socket
SupplementaryGroups=toodoo-ipc-signer toodoo-ipc-auth
PrivateNetwork=yes
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
BindReadOnlyPaths=/etc/toodoo/tls
TasksMax=64
MemoryMax=128M
CPUQuota=100%
LimitNOFILE=4096
```

Normative runtime limits (configurable, defaults):

- TLS handshake deadline: 10 s (a stalled handshake must not pin a
  goroutine and an fd).
- Per-connection idle timeout: 60 s; absolute connection lifetime:
  1 h (ADR 0005).
- Per-source-IP concurrent connection cap: 32; global cap sized to
  `LimitNOFILE` headroom.
- TLS 1.3 only (ADR 0001). No HTTP parsing of any kind.

On accept: handshake (key ops via signer.sock), then write a PROXY
protocol v2 header to a fresh authenticator.sock connection and
splice bytes until either side closes.

### toodoo-signer.service

Smallest service: holds the TLS private key, answers signing requests
over gRPC. No network, no state directory, key file read-only.

```ini
[Unit]
Description=TooDoo TLS Signer
Requires=toodoo-signer.socket

[Service]
ExecStart=/usr/libexec/toodoo/signer
Sockets=toodoo-signer.socket
PrivateNetwork=yes
RestrictAddressFamilies=AF_UNIX
BindReadOnlyPaths=/etc/toodoo/tls/private
TasksMax=16
MemoryMax=64M
CPUQuota=50%
LimitNOFILE=256
```

- **Signing-oracle visibility** (security review): the Signer counts
  and logs signing operations (structured journald). A configurable
  rate alarm (default: >10 ops/s sustained for 60 s) logs a loud
  warning — a compromised Listener abusing the Signer as an oracle
  becomes observable. The Signer also rate-limits hard at a ceiling
  (default 50 ops/s) since legitimate handshake load for a
  small-group instance is orders of magnitude lower.
- Syscall filter is further narrowed for this service (no file-write
  syscalls beyond logging; measured set recorded during integration
  tests and pinned).

### toodoo-authenticator.service

Parses all HTTP, validates sessions, serves public assets, proxies to
the Engine. Owns the auth database.

```ini
[Unit]
Description=TooDoo Authenticator
Requires=toodoo-authenticator.socket toodoo-authenticator-admin.socket toodoo-authenticator-backup.socket toodoo-engine.socket
After=toodoo-engine.socket

[Service]
ExecStart=/usr/libexec/toodoo/authenticator
Sockets=toodoo-authenticator.socket toodoo-authenticator-admin.socket toodoo-authenticator-backup.socket
SupplementaryGroups=toodoo-ipc-engine
StateDirectory=toodoo-authenticator
StateDirectoryMode=0700
PrivateNetwork=yes
RestrictAddressFamilies=AF_UNIX
TasksMax=128
MemoryMax=512M
CPUQuota=200%
LimitNOFILE=2048
```

- `MemoryMax=512M` is deliberate: argon2id at 64 MiB × the
  concurrency cap of 4 (see `authentication.md`) is 256 MiB of hash
  arena alone; the limit leaves headroom without letting a bug eat
  the host.
- `StateDirectory=toodoo-authenticator` → `/var/lib/toodoo-authenticator/`
  (mode 0700, dynamic UID) holds `auth.sqlite`. No other service can
  read it (ADR 0004).
- v2 note: enabling OIDC requires outbound HTTPS — that relaxation
  (`PrivateNetwork=` removal + `AF_INET`) is gated by its own ADR.

### toodoo-engine.service

Business logic and the main database.

```ini
[Unit]
Description=TooDoo Engine
Requires=toodoo-engine.socket toodoo-engine-admin.socket toodoo-engine-backup.socket

[Service]
ExecStart=/usr/libexec/toodoo/engine
Sockets=toodoo-engine.socket toodoo-engine-admin.socket toodoo-engine-backup.socket
StateDirectory=toodoo-engine
StateDirectoryMode=0700
PrivateNetwork=yes
RestrictAddressFamilies=AF_UNIX
TasksMax=128
MemoryMax=256M
CPUQuota=200%
LimitNOFILE=1024
```

- `StateDirectory=toodoo-engine` → `/var/lib/toodoo-engine/` holds
  `data.sqlite` (+ WAL/SHM files).
- No `SupplementaryGroups=` — the Engine dials no one. It only
  accepts.
- Runs schema migrations at startup; refuses to start if the on-disk
  schema is newer than the binary supports (see
  `packaging-debian.md`, downgrades).

## CI Gates

All three gates run on every PR; a regression fails the build.

### 1. Exposure score

`systemd-analyze security --offline --root=<staging-tree> <unit>`
for each of the four services (offline mode analyzes the unit files
without booting them; available on the 24.04 CI runner). **Target:
exposure ≤ 2.0 per service.** The current score per unit is committed
to a fixtures file; the gate fails if any score rises above its
committed value (ratchet), not just above 2.0 — so accidental
loosening is caught even while comfortably under the ceiling. The
reference systemd version for the scores is recorded next to the
fixtures (scores shift across systemd releases).

### 2. Negative isolation tests

An integration stage boots a disposable Ubuntu 24.04 VM, installs the
`.deb`, then asserts that forbidden things **fail**:

- A probe unit cloned from the Engine's hardening attempts to open
  `/var/lib/toodoo-authenticator/auth.sqlite` → must fail
  (EACCES/ENOENT).
- A probe cloned from the Authenticator's hardening attempts an
  outbound TCP connection → must fail (network unreachable).
- An unprivileged user (no group memberships) attempts `connect()`
  on `engine.sock`, `admin.sock`, and `backup.sock` → must fail
  (EACCES).
- A request to the Engine carrying a forged `X-Toodoo-User-Id`
  duplicate header → must be rejected (ADR 0005 header hygiene).
- A direct connection to `authenticator.sock` without a PROXY v2
  preamble → must be closed.

These tests assert the *boundary*, not the score — the score gate
can't see whether a `SupplementaryGroups=` typo quietly widened
access.

### 3. Supply chain

- `govulncheck ./...` on every PR.
- `CGO_ENABLED=0` enforced (build fails if any package requires cgo).
- `-trimpath`, pinned Go toolchain version (in `go.mod` `toolchain`
  directive), `GOFLAGS=-mod=readonly`.
- The `.deb` is signed in the release pipeline; the apt repository
  key is the operator-facing trust root (see `packaging-debian.md`).
- Dependency updates follow the one-week-delay + local diff review
  policy from `docs/SECURITY.md`.

## Open Items

- Measured-syscall pinning for the Signer (record the empirical
  syscall set under integration load, then tighten
  `SystemCallFilter=` to it). Do after the service exists; gate on
  the negative tests, not on guesswork.
- Whether to add `IPAddressDeny=any` to the non-Listener services as
  belt-and-suspenders on top of `PrivateNetwork=yes`. Cheap; decide
  during implementation when scores are measured.
- Cert delivery via the Signer instead of `BindReadOnlyPaths=` into
  the Listener (would remove the Listener's last filesystem access).
  Considered and deferred — the cert chain is public data; the
  complexity isn't justified in v1 (discussed 2026-06-12).
