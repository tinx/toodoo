# Spec: Authentication (Authenticator Service)

This spec defines the Authenticator service: its datastore, its
request-handling pipeline, the authentication endpoints it exposes,
and its coordination with the Engine. Required reading first:
ADR 0004 (Authenticator owns its own datastore) and ADR 0005 (request
flow, header hygiene, TLS boundary).

The Authenticator has two jobs, deliberately fused into one process:

1. **Reverse proxy**: every byte from the network (via the Listener)
   passes through it. It parses HTTP, serves public paths itself, and
   forwards authenticated requests to the Engine with a trusted
   identity header.
2. **Credential authority**: it is the only process that stores or
   verifies credentials, issues or validates sessions, and manages
   invites.

It is the highest-value target in the system (parses untrusted HTTP,
holds credential material, sits in the path of everything). The
design response is to keep it small: proxy + token lookup on the hot
path, with the expensive/complex flows (login, signup) on rarely-hit
endpoints.

## Datastore

A SQLite database in the Authenticator's own `StateDirectory=`
(`/var/lib/toodoo-authenticator/auth.sqlite`), never readable by any
other service (per ADR 0004). Same SQLite conventions as the Engine's
spec: `PRAGMA foreign_keys = ON`, unix-seconds timestamps, UTC,
numbered migrations, own `schema_version` table.

```
identities
  id          TEXT PRIMARY KEY              -- UUIDv7, internal
  user_id     TEXT NOT NULL                 -- logical FK to Engine users.id
  kind        TEXT NOT NULL CHECK (kind IN ('local', 'oidc'))
  created_at  INTEGER NOT NULL

local_credentials
  identity_id   TEXT PRIMARY KEY REFERENCES identities(id) ON DELETE CASCADE
  password_hash TEXT NOT NULL               -- argon2id encoded string
                                            -- (params + salt included)
  must_change   INTEGER NOT NULL DEFAULT 0  -- 1 after admin reset
  updated_at    INTEGER NOT NULL

oidc_links                                  -- present in v1, used in v2
  identity_id  TEXT PRIMARY KEY REFERENCES identities(id) ON DELETE CASCADE
  issuer       TEXT NOT NULL
  subject      TEXT NOT NULL                -- opaque per-issuer ID
  email_seen   TEXT                         -- last-seen email; informational
  linked_at    INTEGER NOT NULL
  UNIQUE (issuer, subject)

sessions
  token_hash   TEXT PRIMARY KEY             -- hex SHA-256 of full token
  user_id      TEXT NOT NULL
  identity_id  TEXT NOT NULL REFERENCES identities(id) ON DELETE CASCADE
  client_kind  TEXT NOT NULL CHECK (client_kind IN ('web','cli','mcp'))
  user_agent   TEXT                         -- truncated to 256 chars
  created_at   INTEGER NOT NULL
  last_seen    INTEGER NOT NULL
  expires_at   INTEGER NOT NULL

invites
  token_hash  TEXT PRIMARY KEY              -- hex SHA-256 of invite token
  created_by  TEXT NOT NULL                 -- user_id of the admin
  created_at  INTEGER NOT NULL
  expires_at  INTEGER NOT NULL              -- default now + 7 days
  redeemed_at INTEGER
  redeemed_by TEXT                          -- user_id created on redemption
  note        TEXT                          -- operator note ("for Bob")

schema_version (version, applied_at)
```

Indexes: `sessions(user_id)` for list/bulk-revoke,
`identities(user_id)` for disable-user cascades.

### Tokens

- **Format**: `tdo_<kind>_<base64url>` where `<kind>` is `w` (web),
  `c` (CLI), `m` (MCP), `i` (invite) and `<base64url>` encodes
  32 bytes from a CSPRNG. The fixed `tdo_` prefix makes leaked
  tokens findable by secret scanners; the kind letter prevents
  token-type confusion (see Sessions below).
- **Stored hashed**: only `SHA-256(full token string)` is persisted
  (per ADR 0004, revised 2026-06-12). Plaintext exists in the
  client's possession and transiently in Authenticator memory. An
  unsalted fast hash is sufficient — these are 256-bit random
  values, not low-entropy passwords.
- **Lookup**: hash the presented token, query by `token_hash`. No
  constant-time comparison needed; the DB index lookup on a hash of
  attacker-supplied input leaks nothing useful.

### Token lifetimes

| Kind | Default expiry              | Renewal                              |
|------|-----------------------------|--------------------------------------|
| web  | 30 days, rolling            | `expires_at` pushed on use (throttled) |
| cli  | 90 days, fixed              | `toodoo auth rotate` issues a new token and revokes the old |
| mcp  | 90 days, fixed              | same as cli                          |

Non-expiring tokens are an explicit operator opt-in
(`auth.allow_nonexpiring_tokens = true` plus `--no-expiry` at
creation), never the default.

`last_seen` and rolling-expiry writes are throttled: updated at most
once per 60 seconds per session, to avoid turning every proxied
request into a database write.

## Request-Handling Pipeline

Per ADR 0005, the Authenticator accepts connections on a Unix socket
from the Listener only (socket group `toodoo-ipc-auth`).

1. **PROXY protocol v2 header** must be the first bytes of every
   connection; it carries the real client IP and port. Connections
   not starting with a valid PROXY v2 header are closed immediately
   (fail closed — never attribute traffic to a missing or default
   address). The parsed client IP is attached to the connection
   context for logging and rate limiting.
2. **HTTP parsing** with hard limits: max header size 16 KiB, max
   request body 1 MiB (auth endpoints: 16 KiB), read/write/idle
   timeouts per ADR 0005's keep-alive policy. Oversized or malformed
   requests get `4xx` and, on repeat, connection closure.
3. **Routing**:
   - **Public paths** (no session required): `/` and `/assets/*`
     (the embedded web UI, served from `embed.FS`), `/openapi.json`,
     `/auth/login`, `/auth/redeem-invite`, `/healthz`.
   - **Auth-session paths**: `/auth/*` other than the public two
     (logout, change-password, sessions management). Handled by the
     Authenticator itself, never forwarded.
   - **API paths**: `/api/*` — validated and proxied to the Engine.
   - **`/internal/*` — always 404 to clients.** This path prefix is
     reserved for requests the Authenticator itself originates
     toward the Engine (see Cross-Service RPCs). Client requests to
     it are never forwarded; the prefix check happens before any
     proxying logic.
   - Everything else: 404.
4. **For `/api/*` requests**: extract credentials, validate
   (below), then **strip every inbound `X-Toodoo-*` header** and
   inject `X-Toodoo-User-Id` and `X-Toodoo-Session-Id`. The
   strip-before-inject order is normative (ADR 0005). The proxied
   request also carries `X-Toodoo-Client-Ip` for the Engine's rate
   limiting and audit logging — same hygiene rules apply.

### Credential extraction and binding

Two presentation channels, strictly bound to client kind:

- **Cookie** `__Host-toodoo_session` — accepted only if the token's
  kind letter is `w`. Bearer headers on the same request are
  rejected (ambiguity = error, not precedence).
- **`Authorization: Bearer tdo_c_...` / `tdo_m_...`** — accepted
  only for CLI/MCP tokens. A web token presented as a bearer header
  is rejected.

This binding means a stolen web cookie can't be replayed through
API tooling unnoticed, and CLI tokens never ride in cookies where
CSRF rules would apply.

### Session validation (hot path)

```
hash = sha256(presented_token)
row  = SELECT ... FROM sessions WHERE token_hash = ?
if row is missing            → 401
if row.expires_at < now      → 401 (and delete row, lazily)
if kind mismatch (see above) → 401
→ valid; attach user_id, session_id; throttled last_seen update
```

No Engine round-trip. The Engine independently rejects disabled
users (`users.disabled_at` check, per ADR 0004/0005), so a freshly
disabled user with a not-yet-deleted session gets `403` from the
Engine, not data access.

## CSRF Policy (web sessions)

Cookie-authenticated requests are CSRF-relevant; bearer-token
requests are not (the token isn't sent automatically by browsers).
Defense layers, all required:

1. **Cookie attributes**: `__Host-` prefix (forces `Secure`, no
   `Domain`, `Path=/`), `HttpOnly`, `SameSite=Lax`.
2. **No state-changing GETs.** Invariant across the whole API:
   GET/HEAD are side-effect-free. `SameSite=Lax` sends cookies on
   top-level GET navigations only, which are then harmless.
3. **Content-Type gate**: every state-changing request (POST, PATCH,
   PUT, DELETE) on a cookie-authenticated session must carry
   `Content-Type: application/json`, which cannot be produced by
   HTML forms or simple cross-origin requests without a CORS
   preflight — and we never send permissive CORS headers (no
   `Access-Control-Allow-Origin` at all; the SPA is same-origin).

With these three, classic CSRF tokens add no further protection and
are omitted.

## Browser Security Headers

Set by the Authenticator on every response it serves (assets and
API alike; values for the asset paths are normative for the SPA,
see `docs/specs/web-frontend.md`):

```
Content-Security-Policy: default-src 'none'; script-src 'self';
  style-src 'self'; img-src 'self'; connect-src 'self';
  frame-ancestors 'none'; base-uri 'none'; form-action 'none'
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Strict-Transport-Security: max-age=63072000
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-origin
```

`Cache-Control: no-store` on all `/auth/*` and `/api/*` responses.
Static assets get long-lived caching with content-hashed filenames
(see web-frontend spec).

## Endpoints

All bodies JSON. Error format follows the stable shape from
`docs/specs/sync-and-concurrency.md`.

### POST /auth/login  (public)

```json
{ "handle": "alice", "password": "...", "client": "web" }
```

- `client: "web"` → `204` + `Set-Cookie: __Host-toodoo_session=...`
- `client: "cli" | "mcp"` → `200` + `{ "token": "tdo_c_...",
  "expires_at": 1234567890 }` — the only time the plaintext token
  is ever transmitted.
- If the credential row has `must_change = 1`, login succeeds but
  the session is scoped to `/auth/change-password` only; everything
  else returns `403 password_change_required`.

**Anti-abuse pipeline, in order, all before any argon2id work:**

1. Per-IP rate limit (default 10 attempts/min, configurable),
   counted on the PROXY-protocol client IP.
2. Per-handle exponential backoff: after 5 consecutive failures for
   a handle, delay grows (1s, 2s, 4s, ... capped at 60s),
   independent of source IP — blunts distributed guessing.
3. Concurrency cap on argon2id (default 4 simultaneous hashes,
   semaphore); excess requests get `429`. Bounds the CPU/memory DoS
   of the deliberately expensive hash.
4. **Handle resolution + dummy hash**: resolve handle → user_id via
   Engine RPC. If the handle does not exist, verify the password
   against a fixed dummy argon2id hash anyway and return the same
   `401 invalid_credentials` in the same time envelope — login
   timing must not reveal handle existence.
5. On success: confirm the user is enabled (Engine RPC), create the
   session, log `auth_success`. On failure: log `auth_failure` with
   the sanitized handle and client IP (fail2ban contract below).

### POST /auth/redeem-invite  (public)

```json
{ "invite": "tdo_i_...", "handle": "bob", "display_name": "Bob",
  "password": "..." }
```

- Hard per-IP rate limit (default 5/hour) — this endpoint is
  unauthenticated and creates accounts.
- Validates the invite (hash lookup, not expired, not redeemed),
  validates handle format and password policy, then runs the
  signup coordination from ADR 0004: Engine RPC `create user` →
  local `identities` + `local_credentials` insert → mark invite
  redeemed → issue session. Ghost-user reconciliation per ADR 0004.

### POST /auth/logout  (session)

Deletes the session row, clears the cookie (web). `204`.

### POST /auth/change-password  (session, local identities only)

```json
{ "current_password": "...", "new_password": "..." }
```

Requires the current password even with a valid session (limits
damage of a hijacked session). On success: update hash, clear
`must_change`, **revoke all other sessions of this user**, keep the
current one. `204`.

### GET /auth/sessions  (session)

Lists the caller's active sessions: `client_kind`, `user_agent`,
`created_at`, `last_seen`, truncated `token_hash` as display id,
and a marker for "this session".

### DELETE /auth/sessions/{token_hash}  (session)

Revokes one of the caller's own sessions. `204`.

### POST /auth/rotate  (session, cli/mcp only)

Issues a new token of the same kind, atomically revokes the
presented one, returns the new token. Backing for
`toodoo auth rotate`.

## Password Policy

- Minimum 12 characters, maximum 128. No composition rules
  (NIST SP 800-63B direction): length and a denylist beat
  "1 uppercase + 1 symbol" theatrics.
- Unicode allowed; normalized to NFC before hashing so the same
  passphrase typed on different clients hashes identically.
- Checked against an embedded top-10k common-password list
  (offline; no external API).
- **argon2id parameters** (RFC 9106 second recommended option,
  fits small self-hosted boxes): memory 64 MiB, iterations 3,
  parallelism 4, salt 16 bytes, tag 32 bytes. Parameters are
  encoded in the hash string, so they can be raised later;
  hashes are transparently re-hashed at next successful login when
  parameters change.

## Admin and Backup Sockets

Per ADR 0006, application administration does **not** require root.
The Authenticator exposes two additional Unix sockets, distinct from
the Listener-facing socket, guarded by static Unix groups created by
the package (same filesystem-permission boundary as the
`toodoo-ipc-*` sockets):

### admin.sock (`SocketGroup=toodoo-admin`, `SocketMode=0660`)

Consumed by the `toodoo` host CLI; any local user in `toodoo-admin`
can run these (granted once via `usermod -aG toodoo-admin <user>`):

- `invite create [--note ...] [--expires-in 7d]` → prints the
  plaintext invite token once.
- `invite list` / `invite revoke <token-prefix>`
- `user disable <handle>` / `user enable <handle>` — runs the
  two-write coordination from ADR 0004 (Engine first, then session
  purge here; clear error + retry guidance on partial failure).
- `user reset-password <handle>` → generates a strong random
  temporary password, prints it once, sets `must_change = 1`,
  revokes all sessions of that user.
- `sessions revoke --user <handle>` — bulk revoke.
- `reconcile` — lists Engine users without a corresponding identity
  here ("ghost users" from interrupted signups or
  backup-consistency edges, see ADR 0004 and
  `docs/specs/backup-restore.md`) and offers to complete or remove
  them. Reads both sides via this socket and the Engine's admin
  socket.

### backup.sock (`SocketGroup=toodoo-backup`, `SocketMode=0660`)

Exactly one operation: stream a consistent snapshot of `auth.sqlite`
(SQLite online backup API) for the `toodoo-backup` CLI; see
`docs/specs/backup-restore.md`. The split exists so backup automation
(the systemd timer runs with `SupplementaryGroups=toodoo-backup`)
never holds user-management powers, and app admins don't implicitly
get bulk export. See ADR 0006 for the rationale.

Every operation on either socket is logged (structured, see below)
with the caller's UID and resolved username from `SO_PEERCRED` —
kernel-verified, not client-claimed. Root retains implicit access to
both sockets (file permissions never bind root).

## Cross-Service RPCs (Authenticator → Engine)

Carried as HTTP over the existing Authenticator→Engine socket under
the reserved `/internal/*` prefix (which the proxy never forwards
from clients — enforced before proxy logic, returns 404):

- `POST /internal/users` `{handle, display_name}` → `{user_id}` —
  signup step.
- `GET /internal/users/by-handle/{handle}` → `{user_id, disabled}` —
  login handle resolution + enabled check (one call).
- `POST /internal/users/{id}/disabled` `{disabled: true|false}` —
  disable/enable step.

These requests carry no `X-Toodoo-User-Id` (they are not on behalf
of a user); the Engine distinguishes them by path prefix and rejects
`/internal/*` requests that *do* carry user-identity headers, as a
consistency check.

## Logging Contract (fail2ban)

All auth events go to journald with structured fields; the message
line is stable and built only from sanitized values:

```
TOODOO_EVENT=auth_failure | auth_success | invite_redeemed |
             invite_failure | session_revoked | admin_op
TOODOO_CLIENT_IP=<ip from PROXY header>
TOODOO_HANDLE=<sanitized>
```

MESSAGE formats matched by fail2ban (versioned contract, anchored —
see `docs/specs/fail2ban.md`):

```
auth_failure handle=<sanitized> ip=<ip>
invite_failure ip=<ip>
```

Sanitization: a handle is logged verbatim only if it matches the
handle grammar (`[a-z0-9_-]{3,32}`); anything else is logged as the
literal `invalid-handle` (length and charset attacks on the log
stream are thereby impossible). User-Agent values are never part of
fail2ban-matched messages. The fail2ban filter (see
`docs/specs/fail2ban.md`) uses `journalmatch` on `TOODOO_EVENT` plus
a regex over the stable MESSAGE format. This contract is versioned:
changing the MESSAGE format is a breaking change requiring a
fail2ban filter update in the same release.

Never logged, at any level: passwords, tokens (plaintext or hash),
password hashes, request bodies of `/auth/*` endpoints.

## v2 Pathways (design-ready, not implemented)

- **OIDC**: the `oidc_links` table is live from v1. The v2 adapter
  adds `/auth/oidc/start` and `/auth/oidc/callback`, implementing
  Authorization Code flow with **PKCE**, fresh `state` and `nonce`
  per attempt, and a strict `redirect_uri` allowlist pinned to this
  instance's origin. Identity matching is by `(issuer, subject)`
  only — never by email. Linking an OIDC identity to an existing
  account requires an authenticated session (no account takeover by
  email collision). Requires outbound HTTPS from the Authenticator —
  a `PrivateNetwork=` relaxation gated by its own ADR (per ADR 0002).
- **MFA / WebAuthn**: the `identities` table accommodates sibling
  credential tables (`totp_secrets`, `webauthn_credentials`) without
  schema upheaval. WebAuthn/passkeys are the preferred v2 direction
  (phishing-resistant, no SMTP).

## Open Items

- Session storage is per-process SQLite; if the Authenticator is
  ever scaled horizontally (v2 Kubernetes), sessions need a shared
  store or sticky routing. Out of v1 scope; flagged for the k8s ADR.
- Account lockout notifications (tell a user their account is under
  attack) — needs a delivery channel we don't have (no SMTP). Defer;
  the operator sees fail2ban activity.
- Per-IP rate-limit state is in-memory and resets on restart.
  Acceptable: fail2ban provides the durable layer.
