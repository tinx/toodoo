# ADR 0002: Service Isolation via systemd-Native Hardening

**Status:** Accepted
**Date:** 2026-06-10
**Supersedes:** an earlier draft of the design that placed services in
rootless podman containers managed by systemd Quadlets.

## Context

toodoo is a privilege-separated multi-service system (see `README.md`):
Listener, TLS Signer, Authenticator, Engine. Each service exists in its
own process for a reason — to confine the blast radius of a compromise
in any one of them. To make that confinement real (and not just
"different binaries running as the same user"), each service needs
strong runtime isolation: filesystem, capabilities, namespaces,
seccomp, resource limits.

The initial plan was to run each service in a rootless podman
container, managed by a systemd Quadlet, distributed as image tarballs
inside the `.deb` package. That was chosen because podman is familiar,
because the `docs/SECURITY.md` discussion was framed in container
terms, and because the lockdown directives (`--cap-drop=ALL`,
`--security-opt=no-new-privileges`, `--read-only`, …) are easy to
articulate.

On review, this choice was load-bearing without being clearly load-
*justified*. Modern systemd (250+, definitely 255+ which ships in
Ubuntu 24.04 LTS) provides equivalent isolation primitives directly in
the service manager. Container runtimes and systemd both compose the
same kernel mechanisms — namespaces, seccomp BPF, capabilities, cgroups
v2 — into a sandbox. The question is which composition is the right
fit for toodoo's specific deployment shape: one `.deb` package, one
operator, one Ubuntu LTS target.

## Decision

**toodoo services run as native systemd units with full sandboxing
directives. We do not use podman, Quadlets, container images, or any
other container runtime.**

Each service ships as a binary in `/usr/libexec/toodoo/` plus a
`.service` (and where appropriate `.socket`) unit file in
`/lib/systemd/system/`. The unit files apply the systemd hardening
suite to constrain the service.

### Baseline hardening (applied to every service)

```ini
DynamicUser=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
PrivateIPC=yes
PrivateUsers=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectClock=yes
ProtectHostname=yes
ProtectProc=invisible
ProcSubset=pid
ProtectControlGroups=yes
RestrictNamespaces=yes
RestrictSUIDSGID=yes
RestrictRealtime=yes
NoNewPrivileges=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
CapabilityBoundingSet=
AmbientCapabilities=
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources @mount @debug @cpu-emulation @obsolete @swap
SystemCallArchitectures=native
RemoveIPC=yes
KeyringMode=private
UMask=0077
```

Per-service additions narrow further (smaller syscall sets for Signer,
`StateDirectory=` for Engine, socket activation for Listener, etc.).
The per-service unit files live in `docs/specs/service-hardening.md`.

### Network isolation

- **Engine, Signer, Authenticator**: `PrivateNetwork=yes`. They have
  no network namespace beyond loopback. Communication with the
  Listener is via socket-activated Unix sockets in
  `RuntimeDirectory=`.
- **Listener**: also `PrivateNetwork=yes`, but receives its listening
  TCP socket via **systemd socket activation**. systemd opens the
  listening socket in the host network namespace and hands the file
  descriptor to the service. The service itself runs network-less.
- **No outbound network** for any v1 service. (The OIDC adapter in v2
  will need outbound HTTPS in the Authenticator; that's an explicit
  loosening that ADR will be written when needed.)

### Inter-service IPC

`.socket` units define each Unix socket with explicit owner, group,
mode. Services declare `Sockets=` and receive the fd via socket
activation. No shared `RuntimeDirectory` between distinct dynamic
users — systemd handles the permission boundary for us.

### Verification

Each service unit is required to score **exposure ≤ 2.0** as reported
by `systemd-analyze security <unit>.service`. CI runs this on every
PR and fails if any unit regresses. The score is not the goal in
itself — it is a regression detector against accidentally relaxing
hardening.

## Why this over podman/Quadlets

Both stacks use the same kernel features for isolation. The decision
hinges on fit, not raw capability:

### Equivalence in primitives

| Podman                                   | systemd                                                   |
|------------------------------------------|-----------------------------------------------------------|
| `--cap-drop=ALL`                         | `CapabilityBoundingSet=` empty, `AmbientCapabilities=` empty |
| `--security-opt=no-new-privileges`       | `NoNewPrivileges=yes`                                     |
| `--read-only`                            | `ProtectSystem=strict` + `ReadWritePaths=`                |
| Rootless UID mapping (subuid)            | `PrivateUsers=yes` + `DynamicUser=yes`                    |
| Default seccomp profile                  | `SystemCallFilter=@system-service ~@privileged …`        |
| Mount namespace + overlay rootfs         | Implicit via `Protect*`/`Private*`/`BindPaths` combo      |
| PID namespace                            | Implicit per unit                                         |
| Network namespace (slirp4netns)          | `PrivateNetwork=yes` + socket activation                  |
| `--pids-limit`, memory, CPU              | `TasksMax=`, `MemoryMax=`, `CPUQuota=`                   |
| Image-as-artifact (signed OCI)           | `.deb` package signature on the binary                    |

### Where they differ, and how toodoo fares

- **Network namespace shape**: slirp4netns gives a user-space network
  stack; systemd's `PrivateNetwork=yes` gives loopback only. For
  toodoo this is a win — Engine, Signer, Authenticator don't need
  network at all, and the Listener uses socket activation. The result
  is *stricter* than the typical rootless-container default.
- **Rootfs immutability**: containers run from a read-only OCI image.
  systemd hardens an existing on-disk filesystem. The threat model
  this difference matters for ("attacker modifies binary on disk")
  requires the attacker to already have privileged write access to
  the host — at which point both stacks have lost. For defense-in-
  depth against an in-namespace process, the property doesn't apply.
- **Trusted Computing Base**: choosing podman means trusting podman,
  crun, slirp4netns, conmon, the rootless-user-systemd-via-machinectl
  path, and the subuid/subgid configuration. Choosing systemd means
  trusting code already in toodoo's TCB (PID 1). Fewer moving parts
  is fewer CVEs to track.
- **Image distribution**: containers ship as portable OCI images. The
  `.deb` ships binaries. For a Ubuntu-only deployment vehicle, OCI
  portability is unused; the `.deb` signature gives equivalent
  provenance.

### Concrete operational benefits

- **Smaller `.deb`**: no image tarballs. The binaries + unit files are
  ~tens of MB rather than ~hundreds of MB.
- **No additional runtime dependencies**: no `podman`, no
  `slirp4netns`, no `uidmap` config, no `conmon`.
- **No `_toodoo` system user**: `DynamicUser=yes` allocates an ephemeral
  UID per service at start. postinst doesn't `adduser`; postrm
  doesn't `userdel`; no `/etc/subuid` allocation.
- **No `loginctl enable-linger`** machinery: services run under PID
  1's systemd, not a per-user systemd inside a lingered session.
- **Native journald integration**: `journalctl -u toodoo-engine` just
  works. No `podman logs` or container-log-driver mapping.
- **Standard debugging**: `systemctl status`, `systemd-analyze`,
  `systemd-cgls` work directly. No `podman exec` indirection.
- **Faster service start** (no container runtime overhead per start).
- **Auditable hardening**: `systemd-analyze security` quantifies each
  service's exposure on a 0–10 scale. There is no equivalent score
  for a Quadlet (the score systemd produces for a Quadlet measures
  the Quadlet wrapper, not the contained service).
- **Ubuntu-native**: this is how modern Ubuntu daemons
  (`systemd-resolved`, `systemd-timesyncd`, snap-confined services)
  are hardened. We match platform conventions.

## Consequences

### Positive

- Strong isolation with smaller TCB and fewer moving parts.
- Cleaner `.deb` install/upgrade/uninstall story.
- Better operational tooling out of the box (journalctl, systemctl,
  systemd-analyze, systemd-cgls).
- CI-enforced exposure score creates an explicit regression gate.

### Negative (accepted)

- We are committing to systemd-based Linux distributions for
  deployment. Non-systemd distros (Alpine with OpenRC, Devuan, Void)
  are out of scope. Acceptable given the `.deb`-on-Ubuntu target
  (see ADR 0003).
- We give up container-image portability (running the same artifact
  on Fedora/RHEL/etc. without rebuild). Not in v1 scope.
- Operators who think in containers must learn the systemd directives.
  The mapping is 1:1, but it's a learning curve.
- We give up image immutability via OCI digest. Replaced with `.deb`
  signature on the binary, which is the same chain-of-trust property
  in a different shape.

### Not affected

- The privilege-separated multi-service architecture
  (Listener / Signer / Authenticator / Engine) is unchanged. We're
  changing the *mechanism* of isolation, not the *decomposition*.
- The lockdown posture from `docs/SECURITY.md` (cap drops, no-new-privs,
  read-only filesystems, seccomp filtering) is preserved in full.
- The CLI client and web frontend specs are unaffected.

## Related Invariant: TLS State Stays in the Listener

The Listener service is the **only** process linked against
`crypto/tls` and the only one holding TLS connection state (traffic
secrets, sequence numbers, AEAD context, key schedule). This is not a
performance optimization — it is the architectural reason the Listener
service exists at all. ADR 0005 documents the request-flow
implications and the explicit rejection of `SCM_RIGHTS` fd-passing
and kTLS for v1. Restating here because the hardening posture depends
on it:

- The Listener's hardening profile is the tightest of the four
  services because attacker-controlled bytes only ever reach a TLS
  endpoint, and that endpoint is the Listener.
- The Authenticator and Engine receive only cleartext over Unix
  sockets and therefore do not need any TLS-related syscalls, library
  code, or filesystem access to certificate material.
- Future "optimizations" that try to push TLS state into other
  services (fd-passing the TCP socket; using kTLS to install keys
  into the kernel for downstream read; passing the master secret over
  the IPC channel) all multiply the TLS code surface across processes
  and are explicitly out of scope. See ADR 0005 for the full
  argument.

## Forward-Looking Notes

- **v2 consideration — Kubernetes as the second deployment target**:
  if/when a second deployment path is added, the most likely target
  is Kubernetes (see ADR 0003), not Quadlets-on-other-distros nor
  per-distro packages. The kernel isolation primitives we rely on
  (capabilities, seccomp, namespaces) all have direct equivalents in
  pod `securityContext` + NetworkPolicy + PodSecurityAdmission. The
  service binaries remain the same artifact in both shapes — only
  the surrounding wrapper changes (`.service` unit on Ubuntu, OCI
  image + Helm chart on k8s). Quadlets specifically have a thin
  remaining use case (rootless-containers-on-a-single-host without
  k8s) that does not justify carrying a third deployment path.
- **OIDC in v2** will require outbound HTTPS in the Authenticator;
  that's a `PrivateNetwork=` relaxation gated by an explicit ADR at
  the time.
- The CI exposure-score gate will need maintenance as new systemd
  directives appear in future releases (a service that scored 1.8 on
  systemd 255 may score differently on 260+). Document the
  reference systemd version in the gate.

## References

- `README.md` — service architecture
- `docs/SECURITY.md` — security posture (privilege separation, supply
  chain, secret injection)
- `docs/adr/0003-distribution-debian-package-on-ubuntu-lts.md` — the
  deployment-vehicle decision this composes with
- `docs/specs/service-hardening.md` — per-service unit files and
  detailed hardening
- `docs/specs/packaging-debian.md` — packaging mechanics
- `systemd.exec(5)`, `systemd-analyze(1)` — primary references for
  the directives used
- Plan: `/root/.claude/plans/i-m-not-sure-e2e-whimsical-rocket.md`
