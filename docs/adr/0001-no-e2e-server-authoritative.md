# ADR 0001: No End-to-End Encryption; Server-Authoritative Model

**Status:** Accepted
**Date:** 2026-06-10
**Supersedes:** the "Encrypted data" bullet in `README.md` and the
"encryption" item in `docs/NOTES.md`, both of which had earlier implied a
client-side encryption posture.

## Context

toodoo is intended to be self-hosted by small groups — clubs, teams,
circles of friends, small companies — where each instance serves one
group and is operated by a member of that group. The original sketch
called for "encrypted data", which was read as end-to-end (client-side)
encryption: the server stores only ciphertext, clients hold the keys.

Pursuing E2E in this architecture has substantial implications:

- All processing (search, sort, filter) must happen client-side.
- Multiple clients editing in parallel produce conflicts the server
  cannot help resolve (it cannot read the data). This pushes the design
  toward CRDTs or a hand-rolled Last-Writer-Wins scheme with vector
  clocks, tombstone garbage collection, and offline-replay semantics.
- Server-side features become impossible without breaking E2E: full-text
  search, MCP/AI integration, server-rendered web UI, email or push
  notifications, sharing-by-link, password reset / account recovery.
- Key management becomes a load-bearing concern: lose the key, lose the
  data. Key rotation, revocation on shared lists, and recovery options
  all require explicit design.
- The web client must ship a non-trivial crypto + sync runtime,
  significantly larger than a thin client over an HTTP API.

The protection E2E buys is real but narrow: it defends against a
compromised or coerced server operator, and against backup theft. It
does **not** defend against the more common attack vector — compromise
of a client endpoint (browser extension, malware, stolen unlocked
laptop). For the chosen audience, the server operator is a known and
trusted member of the group; defending against them is not a goal.

The group cannot reduce trust to zero in any case. End users must trust:

- the operator (chooses the deployment, has root on the host)
- the client devices and browsers they use
- the toodoo project itself (supply chain)

Adding E2E shifts trust from "the operator" to "every client device,
forever" — which is not obviously an improvement, and is a meaningful
loss in places where client endpoints are the weaker link.

## Decision

**toodoo will not implement end-to-end encryption.** The server is
authoritative: it stores plaintext, executes business logic, performs
search, and resolves authorization. Each instance is a single trust
domain, owned by its operator.

Encryption posture instead:

- **In transit:** TLS terminated by the Listener service. **TLS 1.3
  only** — we do not negotiate down to 1.2. Rationale: TLS 1.3 has a
  small, fixed set of safe AEAD cipher suites with no negotiable
  weaknesses, while TLS 1.2 cipher-suite selection is a recurring
  source of vulnerabilities. Modern browsers (2020+) and any current
  Go HTTP client support 1.3; clients that don't are out of scope.
  Private key isolated in the TLS Signer service. Let's Encrypt for
  cert provisioning.
- **At rest (data):** the SQLite database file lives in
  `/var/lib/toodoo/data/` under the `_toodoo` system user. Operators
  who want stronger at-rest protection are expected to use OS-level
  encryption (LUKS, eCryptfs, or full-disk encryption). The application
  does not implement its own at-rest encryption layer.
- **At rest (backups):** `toodoo-backup` supports optional GPG
  encryption to an operator-provided recipient. Backup manifests
  include schema version and container image digests for reproducible
  restore.
- **Secrets:** never stored in git or in the database. Injected at
  runtime via volume mounts (per `docs/SECURITY.md`).

Concurrency, made possible by the server being authoritative, uses
optimistic per-item version numbers (see
`docs/specs/sync-and-concurrency.md`).

## Consequences

### Positive

- Server-side search, filtering, and aggregations are straightforward.
- MCP server and AI integration can operate on plaintext, which is the
  point of those features.
- Web frontend can be a thin client over a REST API; no crypto runtime
  shipped to the browser.
- Sharing is implemented as a row in `list_members` — no key exchange
  ceremony.
- Password reset and account recovery are operator-administered, no
  data loss on forgotten passwords.
- Sync model collapses to versioned reads and `409 Conflict` on stale
  writes; no CRDT/LWW machinery.
- Self-host story stays simple: one trusted operator per deployment,
  matching the same model that Vaultwarden, Gitea, and Nextcloud
  operate under.

### Negative (accepted)

- A compromised or coerced server operator can read all data in the
  deployment. This is mitigated by: rootless containers with capability
  drops (see `docs/specs/container-images.md`), audit logging in the
  Authenticator, and the deployment being one operator per group rather
  than a multi-tenant SaaS where one operator holds many groups' data.
- Backup theft from an unencrypted host filesystem leaks data. Mitigated
  by recommending OS-level disk encryption and supporting GPG-encrypted
  backups.
- Future direction is constrained: if the project ever wants to host
  multiple groups under a single operator (true SaaS), the trust model
  re-opens and E2E may need to be revisited. Out of scope for v1.

### Not affected

- Privilege separation between toodoo services (Listener / TLS Signer /
  Authenticator / Engine) remains a goal and is implemented through
  rootless podman containers with the lockdown posture from
  `docs/SECURITY.md`. That is **server-internal** isolation, orthogonal
  to E2E.
- TLS private key protection via the TLS Signer service remains
  unchanged.

## References

- `README.md` — overall service architecture
- `docs/SECURITY.md` — privilege separation, supply chain, secret
  injection
- `docs/specs/sync-and-concurrency.md` — concurrency model this decision
  enables
- `docs/specs/sharing-and-authz.md` — sharing model this decision enables
- Design conversation: `/root/.claude/plans/i-m-not-sure-e2e-whimsical-rocket.md`
