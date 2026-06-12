# ADR 0005: Request Flow and TLS Termination Boundary

**Status:** Accepted
**Date:** 2026-06-10
**Revised:** 2026-06-12 (security review, pre-implementation): public
static assets are embedded in and served by the Authenticator (not the
Listener, correcting `README.md`); PROXY protocol v2 client-IP
propagation on the Listener→Authenticator socket; normative
strip-before-inject rule for `X-Toodoo-*` headers; admin sockets are
group-guarded (`toodoo-admin` / `toodoo-backup`) instead of
root-only — see ADR 0006.
**Related:** ADR 0002 (service isolation via systemd hardening),
ADR 0004 (Authenticator owns its own datastore)

## Context

toodoo runs as four cooperating services on a single host: Listener,
TLS Signer, Authenticator, Engine. Each runs in its own systemd
sandbox under a distinct dynamic UID (per ADR 0002), and the
Authenticator and Engine each own a private database (per ADR 0004).

This ADR fixes two architectural questions that became load-bearing
once the services started materializing:

1. **Where in the request lifecycle does authentication happen?**
   The two main candidates were:
   - **Flow A (linear)**: client → Listener → Authenticator → Engine.
     The Authenticator is a per-request reverse proxy that validates
     the session inline and forwards authenticated requests to the
     Engine. The Engine never sees unauthenticated traffic.
   - **Flow B (consultation)**: client → Listener → Engine. The
     Engine receives requests directly and consults the Authenticator
     via RPC to validate the bearer token. The Authenticator is a
     side-service, not a proxy.

2. **Where does TLS termination happen, and where does TLS state
   live?** TLS termination has to happen at the network boundary
   (the Listener) because the rest of the system isn't reachable
   from the network. The follow-on question is whether the Listener
   stays in the connection path forever (cleartext relay) or hands
   off the TCP socket to a downstream service via `SCM_RIGHTS`
   fd-passing or kTLS.

Both questions affect the same code paths and the same trust model,
so they are decided together.

## Decision

**Flow A is the request flow. TLS state stays in the Listener for
the entire lifetime of every TCP connection. No fd-passing. No kTLS
in v1.**

### Request flow

```
                                      ┌─────────────┐
client ──TLS──> Listener ──cleartext──> Authenticator ──cleartext──> Engine
                  :443    Unix socket           Unix socket
                            cleartext             cleartext
                            HTTP/1.1               HTTP/1.1
                                                   +X-Toodoo-User-Id
                                                   +X-Toodoo-Session-Id
                                                   (injected by Authenticator)
```

Step by step:

1. **Listener** accepts TCP on `:443`, performs the TLS 1.3 handshake
   (offloading private-key operations to the TLS Signer via gRPC over
   a Unix socket — Signer never sees cleartext, Listener never has the
   private key).
2. Listener opens a Unix socket connection to the **Authenticator**
   for this client connection, writes a **PROXY protocol v2 header**
   carrying the client's IP address and port, then splices cleartext
   bytes both directions between the decrypted TLS socket and the
   Authenticator socket. The Listener stays in the byte-splice loop
   for the connection's lifetime; when the client closes (or the
   Authenticator closes), it tears down both ends.

   The PROXY header is what makes fail2ban work: the Authenticator
   has no network visibility (it sits behind a Unix socket), so the
   client IP it logs for failed authentication attempts comes from
   this header. The Authenticator rejects connections whose first
   bytes are not a valid PROXY v2 header (fail closed — a
   misconfigured direct connection must not be attributed to a wrong
   or empty address).
3. **Authenticator** parses HTTP requests off the cleartext stream.

   **Public paths** (the embedded web UI assets, the OpenAPI spec,
   and the login/invite-redemption endpoints) are served directly by
   the Authenticator without session validation — they are public by
   definition. All assets are compiled into the Authenticator binary
   via `embed.FS`; the Listener never parses HTTP and serves nothing.
   (This corrects the earlier `README.md` sketch, which had the
   Listener serving public assets — that would have put an HTTP
   parser into the unauthenticated network-facing service, exactly
   the surface this ADR minimizes.)

   For each request to a protected path:
   - Extracts the session token from `Authorization: Bearer …` or
     the session cookie.
   - Validates the token against its own SQLite database (per
     ADR 0004) — fast local lookup, no RPC.
   - **If invalid → returns `401 Unauthorized` directly to the
     client.** Engine never sees the request.
   - **If valid → strips all client-supplied `X-Toodoo-*` headers,
     then injects `X-Toodoo-User-Id` and `X-Toodoo-Session-Id`**,
     opens (or reuses) a Unix socket connection to the Engine,
     proxies the request, receives the response, returns it to the
     client. Keep-alive is preserved end-to-end; each subsequent
     request is re-validated. The strip-before-inject order is
     normative: a client must never be able to smuggle its own
     `X-Toodoo-*` values past the proxy. As defense-in-depth, the
     Engine rejects (fails closed on) any request carrying duplicate
     `X-Toodoo-*` headers.
4. **Engine** receives only cleartext HTTP from the Authenticator's
   Unix socket. It trusts the injected headers because:
   - Its socket is filesystem-permissioned to be reachable only by
     the Authenticator's dynamic UID. (Admin operations use the
     separate group-guarded admin sockets — see ADR 0006 — never
     this socket.)
   - The systemd hardening profile (per ADR 0002) blocks any other
     process from interposing.
   - The Authenticator is the *only* code path that writes to that
     socket.
   The Engine additionally performs a defense-in-depth check on
   every request: look up `users.disabled_at` for the
   `X-Toodoo-User-Id` value; if set, return `403 disabled`. This
   catches the rare partial-failure window of the
   disable-user coordination (see ADR 0004).

### TLS termination boundary

- **Only the Listener links against `crypto/tls`.** Authenticator
  and Engine have no TLS code, no TLS-related syscalls in their
  `SystemCallFilter=`, and no filesystem access to certificate
  material.
- **Only the Listener holds TLS state.** Per connection, the Listener
  owns the traffic secrets, sequence numbers, AEAD context, key
  schedule, and post-handshake machinery (KeyUpdate, close_notify,
  NewSessionTicket).
- **TLS 1.3 only** (per ADR 0001). The handshake configuration is
  set in the Listener and verified in startup tests.

### Why Flow A over Flow B

- **Engine never sees unauthenticated bytes.** Bugs in Engine's
  request parsing are only reachable from an authenticated session.
  That is the entire reason the Authenticator exists as a separate
  service.
- **Failed auth never burns Engine resources.** Brute-force,
  credential-stuffing, and replay attacks consume Authenticator
  budget only. fail2ban hooks live at the layer that observes the
  failures.
- **Engine can be ignorant of auth.** It does not parse cookies, does
  not validate bearer tokens, does not know about session lifetimes.
  Its only source of "who is the caller" is the injected
  `X-Toodoo-User-Id` header.
- **Matches the existing README** ("the connection is passed on to
  the Task Service, including the information about the user that
  was successfully identified"). No rework of the published
  architecture sketch.
- **Authenticator-as-reverse-proxy is a well-understood pattern.**
  Go's `net/http/httputil.ReverseProxy` implements it in
  ~hundreds of lines of code. nginx's `auth_request` does the same
  thing externally. The pattern is not novel.

### Why TLS state stays in the Listener (no fd-passing, no kTLS)

The hard technical reason: TLS is stateful. After the handshake the
Listener holds, per connection:

- Traffic secrets (TLS 1.3
  `client_application_traffic_secret_0` / `server_application_traffic_secret_0`)
  and the AEAD keys derived from them.
- Per-direction record sequence numbers (used as AEAD nonces).
- The AEAD cipher state (ChaCha20-Poly1305 or AES-GCM context, plus
  any partial record buffered because TCP arrived mid-record).
- The key schedule needed to handle post-handshake `KeyUpdate` —
  mandatory in TLS 1.3 after enough bytes have been transferred.
- Logic for `NewSessionTicket`, alerts, and `close_notify`.

Handing off the bare TCP file descriptor without this state gives
the receiving process an opaque encrypted byte stream it cannot
read. To actually transfer the connection, *all of the above* would
have to be serialized and reconstituted in the receiver — and the
receiver would need the entire TLS library to do anything with it.

Doing that anyway buys us:

- **TLS code in three processes instead of one**, multiplying the
  attack surface for the broad class of TLS-implementation CVEs
  (Heartbleed, ROBOT, LogJam, BEAST, CRIME, FREAK, dozens of
  padding-oracle variants).
- **Traffic secrets in three address spaces simultaneously**,
  defeating the principle of minimal key-material exposure that
  motivates the TLS Signer in the first place.
- **A custom sub-protocol for TLS state synchronization** between
  cooperating processes (whose turn is it to issue a `KeyUpdate`?
  who saw the next sequence number first?). Any bug in that
  sub-protocol is a session-secret leak. TLS 1.3 was designed
  assuming one stateful endpoint per side; we would be reinventing
  multi-endpoint TLS without the benefit of having designed it.

**kTLS** (Linux kernel TLS, installed via `setsockopt(TLS_TX/TLS_RX)`)
is the only mechanism that makes "downstream process reads cleartext
from the original fd" technically work. But it doesn't help here:

- The handshake-doing process (Listener) still installs the keys.
  TLS state has moved to the kernel rather than to another userspace
  process, but the same state-confinement question reappears at the
  kernel boundary instead of the process boundary.
- The receiving process still needs to know the socket is TLS to
  handle `KeyUpdate` and `close_notify`. Some kTLS implementations
  punt these back to userspace.
- Distro/kernel-version dependencies, partial AEAD support, no help
  for non-Linux.
- For toodoo's throughput, the architectural cost outweighs the
  benefit. We don't need zero-copy splice.

**mTLS / service-mesh** (each service does its own TLS handshake on
an internal Unix-socket-or-TCP connection) is a different model. It
doesn't pass state — it double-terminates. The point is to
authenticate inter-service traffic over an *untrusted network*
(cross-zone, cross-host, cross-cluster). For toodoo with all
services on the same host talking over filesystem-permissioned Unix
sockets, the network is already trusted; mTLS would be ceremony
without security gain. (The v2 Kubernetes deployment target in
ADR 0003 may revisit this, since pods can land on different nodes.)

### Service-to-service trust contract

The whole architecture rests on filesystem-permissioned Unix sockets
between services:

Access mechanism (uniform for all three service sockets): each
`.socket` unit sets `SocketMode=0660` and `SocketGroup=` to a static
group created by the package (`toodoo-ipc-auth`, `toodoo-ipc-engine`,
`toodoo-ipc-signer`), and the one service allowed to connect joins
that group via `SupplementaryGroups=`. Static groups are required
because `DynamicUser=` UIDs are ephemeral and cannot be referenced in
ownership rules ahead of time. Details in
`docs/specs/service-hardening.md`.

- **Listener → Authenticator socket**: created by Authenticator's
  `.socket` unit; `SocketGroup=toodoo-ipc-auth`; only the Listener
  has `SupplementaryGroups=toodoo-ipc-auth`.
- **Authenticator → Engine socket**: created by Engine's `.socket`
  unit; `SocketGroup=toodoo-ipc-engine`; only the Authenticator is
  in that group.
- **Listener → TLS Signer socket**: created by Signer's `.socket`
  unit; `SocketGroup=toodoo-ipc-signer`; only the Listener is in
  that group.
- **Admin sockets** (see ADR 0006): each of Authenticator and Engine
  exposes two additional sockets, guarded by static Unix groups
  rather than root — `admin.sock` (`SocketGroup=toodoo-admin`,
  `SocketMode=0660`) for user/invite/session management, and
  `backup.sock` (`SocketGroup=toodoo-backup`) for snapshot streaming
  only. Used by the `toodoo` host CLI; callers are identified and
  logged via `SO_PEERCRED`. Never reachable from the network.

The Engine's trust of the `X-Toodoo-User-Id` header rests entirely
on the fact that only the Authenticator can connect to its main
socket. Compromising that trust requires (a) breaking systemd's
filesystem isolation, (b) cracking root, or (c) compromising the
Authenticator itself — at which point the attacker is already past
the auth perimeter regardless.

## Consequences

### Positive

- **Single TLS endpoint** in the trust boundary, with the smallest
  possible attack surface.
- **Engine receives only authenticated traffic**, narrowing the
  attack surface for any future request-parsing bug.
- **Authentication is a perimeter concern**, not a per-handler
  concern in business logic. The `authorize(user, list_id, action)`
  function (per `docs/specs/sharing-and-authz.md`) trusts the user
  identity it receives because the Authenticator already vouched.
- **Operational simplicity**: standard reverse-proxy pattern, no
  novel fd-passing dance, no kernel-version dependencies.

### Negative (accepted)

- **Listener stays in the byte-splice loop for the entire connection
  lifetime.** One extra `read()/write()` pair per direction. At
  toodoo's scale this is a rounding error; at any scale it is the
  cost of keeping TLS state confined.
- **Authenticator is in the critical path of every authenticated
  request.** A bug or hang in the Authenticator stalls user traffic.
  Mitigated by: Authenticator is small (proxy + token lookup), its
  hot path is a single SQLite read per request, and its hardening
  profile makes most classes of compromise produce a fail-fast crash
  the systemd unit restarts.
- **Engine cannot independently reject a token.** It trusts what the
  Authenticator sends. If a future requirement needs token-level
  introspection in the Engine (e.g., "does this token have scope
  `read:lists`?"), the Authenticator either injects more headers or
  the model becomes hybrid. Acceptable trade-off for v1; flagged for
  future review.

### Not affected

- The Engine's API surface and storage model.
- The Authenticator's database schema (defined by ADR 0004).
- The hexagonal port abstractions.
- The sync, sharing, and authorization specs.
- The systemd hardening directives in ADR 0002 (this ADR builds on
  them by specifying the socket topology).

## Forward-Looking Notes

- **v2 Kubernetes deployment** (per ADR 0003) inverts some of these
  assumptions: pods may be on different nodes, so service-to-service
  traffic crosses an untrusted network. The natural answer is
  mTLS via cert-manager or a service mesh, with each pod doing its
  own short-lived TLS termination on internal traffic. The Listener
  becomes "an ingress controller fronting the Authenticator pod",
  but the trust model otherwise carries: Engine still trusts the
  Authenticator's injected `user_id` header, just over an mTLS
  connection rather than a Unix socket. This ADR remains correct
  for v1 single-host; a follow-up ADR will address the multi-host
  case at the time.
- **OIDC redirects in v2** require the client to be redirected to
  the IdP's authorization endpoint and back. The Authenticator
  becomes the OIDC client (initiates the flow, validates the ID
  token, links the identity), but this stays within the existing
  request-flow shape — the redirect handling is at the Authenticator
  layer, the Engine remains oblivious.
- **Keep-alive policy**: v1 uses standard HTTP/1.1 keep-alive with
  conservative idle timeouts (~60s) and a hard cap on connection
  lifetime (e.g. 1h) to bound the impact of a leaked session token.
  HTTP/2 and HTTP/3 are not in scope for v1 — the additional
  complexity in the Listener and Authenticator does not pay back at
  toodoo's load.

## References

- ADR 0001 — server-authoritative model
- ADR 0002 — service isolation via systemd hardening
- ADR 0003 — Debian package distribution; v2 Kubernetes pathway
- ADR 0004 — Authenticator owns its own datastore
- `README.md` — service architecture
- `docs/SECURITY.md` — privilege separation policy
- `docs/specs/data-model.md` — Engine's schema
- `docs/specs/sharing-and-authz.md` — `authorize()` model that
  consumes the injected user identity
- `docs/specs/service-hardening.md` — exact systemd unit definitions
  and socket topology (to be written as task #4)
- `docs/specs/authentication.md` — Authenticator's schema, RPCs, and
  signup/login flows (to be written as task #3)
- Plan: `/root/.claude/plans/i-m-not-sure-e2e-whimsical-rocket.md`
