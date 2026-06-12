# ADR 0003: Distribution via Debian Package on Ubuntu LTS

**Status:** Accepted
**Date:** 2026-06-10
**Related:** ADR 0002 (service isolation via systemd hardening) — the
two decisions compose: this ADR picks the package format and target
platform, ADR 0002 picks the isolation mechanism, both assume the
other.

## Context

toodoo is intended to be easily self-hosted by small groups (clubs,
teams, circles of friends, small companies). The deployment-friction
budget is roughly "an operator with basic Linux experience should be
able to stand up an instance in under thirty minutes." This rules out
several otherwise-fine options:

- **Tarball + manual install**: every operator reinvents `useradd`,
  systemd unit installation, cert wiring, fail2ban setup. Too much
  bespoke work, too many ways to get it wrong.
- **Container image + `docker-compose.yaml`**: works but assumes
  container literacy, drags in docker/podman as a runtime dep, and
  the operator has to know how to expose ports, mount volumes, manage
  upgrades.
- **Kubernetes manifests / Helm chart**: overkill for an operator
  who doesn't already have a cluster — installing k3s to run toodoo
  is more setup than installing toodoo. But for groups that *do*
  already run k8s (which is a meaningful slice of the target
  audience: startups, technical clubs, even single-server orgs on
  k3s/kind/minikube), k8s is actually less marginal cost than a
  package install. This is the most likely v2 path; see "Future
  packaging targets" below.
- **Snap**: viable on Ubuntu and gives strong confinement, but the
  snapd daemon is a heavy dependency, the confinement model is
  restrictive in ways that fight our multi-service IPC, and snap is
  poorly received outside Ubuntu — which matters for our audience's
  goodwill toward us.

A Debian package installed via `apt` is the standard "thing that just
works" on Ubuntu and Debian: signed package, dependency resolution,
predictable file layout, conffile semantics, postinst/prerm/postrm
hooks, integration with `apt upgrade` and unattended-upgrades.

We do not target Debian itself in v1, though the package is structured
to make a future Debian package straightforward. Ubuntu LTS is the
v1 floor because (a) it is by far the most common self-host platform,
(b) it has predictable release and support cadences, (c) its systemd
versions are recent enough for the hardening directives we rely on.

## Decision

**toodoo is distributed as a `.deb` package, with Ubuntu 24.04 LTS as
the minimum supported release.** No tarball, no container image, no
snap, no flatpak. One vehicle, well engineered.

### Package split

- `toodoo-server` — service binaries (`/usr/libexec/toodoo/*`),
  `.service` and `.socket` unit files, config (`/etc/toodoo/`),
  fail2ban filters, LE renewal hook, backup/restore CLIs.
- `toodoo-cli` — host-installed CLI binary (`/usr/bin/toodoo`) for
  end-user task management. Installable independently of
  `toodoo-server` so a user can put it on a laptop without dragging
  the server in.

### Why Ubuntu 24.04 LTS as the floor

The rationale shifted after ADR 0002 (we no longer need podman 4.4+
for Quadlets), but the choice itself stands:

1. **Modern systemd hardening directives**: 24.04 ships systemd 255,
   which has the full set of directives we use (`ProcSubset=pid` (247),
   `LoadCredential=` (250), `SocketBindDeny=` (249),
   `RestrictNetworkInterfaces=` (251)). Earlier LTS releases would
   force per-directive fallback logic.
2. **Single supported codepath**: supporting both 22.04 and 24.04 in
   the package's postinst, in the unit files, and in CI roughly
   doubles the maintenance cost for a release that EOLs in April
   2027 anyway.
3. **Standard support window**: 24.04 LTS supports until April 2029
   (5-year standard) or April 2034 (10-year ESM). That comfortably
   exceeds any v1 horizon and gives operators a long runway before
   they need to upgrade the host OS.
4. **Recent enough for current security baselines**: TLS 1.3 only
   (per ADR 0001), modern OpenSSL, recent kernel for cgroups v2 and
   BPF features used by `IPAddressDeny=`.
5. **Audience reality**: groups setting up a new self-hosted service
   in 2026 will overwhelmingly install on a current LTS. The
   "but my server runs 20.04" complaint is a vanishing minority and
   has a clean workaround (do-release-upgrade).

### Dependencies (`Depends:`)

- `systemd (>= 255)`
- `fail2ban`
- `certbot`

That's all. Notably absent: `podman`, `slirp4netns`, `uidmap`,
`docker.io`, anything from the container runtime ecosystem.
`Recommends:` may add `python3-certbot-nginx` or similar if we ship
example certbot integrations, but those are not load-bearing.

### File layout

Per `docs/specs/packaging-debian.md`, but the high level:

```
/usr/libexec/toodoo/{listener,signer,authenticator,engine}
/usr/bin/toodoo                                   # from toodoo-cli
/lib/systemd/system/toodoo-*.{service,socket}
/etc/toodoo/config.toml                           # conffile
/etc/toodoo/tls/                                  # LE deploy target
/etc/letsencrypt/renewal-hooks/deploy/toodoo
/etc/fail2ban/filter.d/toodoo.conf
/etc/fail2ban/jail.d/toodoo.conf
/var/lib/toodoo/                                  # Engine StateDirectory
/var/lib/toodoo/backups/
```

### Install / upgrade / uninstall semantics

- **Install** (`postinst configure`): install unit files,
  `systemctl daemon-reload`, `systemctl enable --now` each service.
  No user creation (DynamicUser handles it). Engine self-migrates the
  schema on first start, creating the initial database.
- **Upgrade** (`postinst configure <old-version>`): stop services in
  reverse dependency order, install new binaries + unit files,
  `daemon-reload`, start in dependency order. Engine self-migrates;
  refuses to start if the on-disk schema is newer than the binary
  supports. Conffile changes prompt via standard dpkg conffile
  handling.
- **Remove** (`prerm remove`): stop services, disable units. Leave
  `/etc/toodoo/`, `/var/lib/toodoo/` in place so a re-install picks
  back up.
- **Purge** (`postrm purge`): remove configs and state. We warn
  prominently in the package description that purge destroys the
  database.
- **Downgrade**: refused by the Engine at startup if it would require
  rolling back a schema migration. We do not implement reverse
  migrations.

### Versioning

- Semantic version on the package (`MAJOR.MINOR.PATCH`).
- Schema version is an internal integer maintained by the Engine,
  independent of package version. A given package version supports a
  range of schema versions. Backup manifests record schema version so
  `toodoo-restore` can detect mismatches.

### Future packaging targets (not in v1)

The most likely v2 deployment target is **Kubernetes**, not another
distro's package format. Rationale:

- **Audience fit**: startups, technical clubs, and small companies
  often already run a cluster (often k3s, kind, or minikube on a
  single server, sometimes a proper managed cluster). For those
  operators, "apply this Helm chart" is less work than "set up an
  Ubuntu VM and `apt install`."
- **OS independence**: k8s sidesteps the systemd-only-Linux
  constraint ADR 0002 accepted. The cluster's host OS no longer
  matters; the container runtime provides the isolation primitives.
- **Architectural fit**: the privilege-separated multi-service design
  maps cleanly onto pods. The systemd hardening directives have
  direct k8s equivalents (`securityContext.capabilities.drop: ALL`,
  `runAsNonRoot`, `readOnlyRootFilesystem`,
  `seccompProfile.type: RuntimeDefault`, NetworkPolicy for inter-pod
  isolation, PodSecurityAdmission `restricted` baseline). Same
  posture, different wrapper.
- **Backend choice**: the Postgres adapter (already a design
  requirement, see `docs/specs/data-model.md`) becomes the obvious
  data store for k8s deployments. SQLite in a StatefulSet with a
  ReadWriteOnce PVC is viable for the simplest case but Postgres is
  more conventional for k8s.
- **Cert and TLS**: cert-manager replaces certbot. The Listener sits
  behind an ingress (or as a Service of type `LoadBalancer`) for TLS
  termination. The TLS Signer separation is preserved as a separate
  pod with the key mounted only there.
- **Artifacts needed**: per-service OCI images built in CI, published
  to a registry (GHCR by default). A Helm chart with values for the
  common knobs (DB backend, TLS source, ingress class, resource
  limits). A separate spec when this is committed to.

Explicitly **not** future targets:

- Native Debian (non-Ubuntu) or RPM (Fedora/RHEL) packages. If
  someone wants toodoo on non-Ubuntu Linux, they should either build
  from source or use the k8s path.
- A Quadlet bundle. The remaining use case ("rootless containers on
  a single host without k8s") is too thin a slice once both the
  `.deb` path and the k8s path exist.
- snap, flatpak, Windows packages, macOS bundles. Out of scope.

## Why not snap?

Briefly, since this comes up:

- **Pros**: native to Ubuntu, AppArmor confinement out of the box,
  delta updates, channel-based release management.
- **Cons that decided it**: snapd is a heavy dependency; the
  confinement model uses interfaces that are awkward to compose for a
  multi-service architecture with Unix-socket IPC between
  differently-confined processes; snap is unpopular with the
  self-host community we're targeting (real and arguably unreasonable,
  but a cost we'd pay); upgrades happen on snap's schedule, not the
  operator's, which surprises people.

snap's strengths (confinement, transactional updates) don't
substantially exceed what we already get from systemd hardening +
apt, and its costs land directly on the operators we're trying to
make happy.

## Why not Flatpak?

Flatpak is a desktop application format. Not applicable.

## Consequences

### Positive

- One supported install path. `sudo apt install toodoo-server`. Done.
- Predictable upgrade story via `apt upgrade` and
  `unattended-upgrades`.
- Familiar conventions for operators (conffiles, postinst hooks,
  signed package, dependency resolution).
- Compact `.deb`: tens of MB rather than hundreds.
- Standard signing and provenance via the apt repository's GPG key.

### Negative (accepted)

- Non-Ubuntu Linux users (Fedora, Rocky, Arch, NixOS) cannot use the
  v1 package. They can build from source, but there is no first-class
  installer for them. Acceptable for v1.
- Non-systemd distributions are out of scope. Documented; not a goal.
- Hosting the apt repository (GPG key management, package upload
  workflow) becomes operational work for the project. Mitigated by
  using a managed-repo service or GitHub Releases with a simple
  `apt-get`-able layout.

### Not affected

- The internal service architecture (Listener / Signer / Authenticator
  / Engine), the data model, the sync model, the sharing model — all
  agnostic to packaging.

## References

- `docs/adr/0001-no-e2e-server-authoritative.md`
- `docs/adr/0002-service-isolation-via-systemd-hardening.md`
- `docs/specs/packaging-debian.md` — detailed packaging mechanics
- `docs/specs/letsencrypt.md` — LE integration shipped in the package
- `docs/specs/fail2ban.md` — fail2ban integration shipped in the
  package
- Plan: `/root/.claude/plans/i-m-not-sure-e2e-whimsical-rocket.md`
