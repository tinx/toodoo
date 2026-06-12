# Spec: Let's Encrypt Integration

How TLS certificates get issued, renewed, and delivered to the
Listener and Signer. Required reading: ADR 0005 (TLS boundary:
Listener holds the chain and TLS state, Signer holds the private
key), `packaging-debian.md` (file layout, certbot dependency).

## Design

certbot owns the ACME lifecycle; toodoo never speaks ACME. The
package ships one deploy hook that copies renewed material into
toodoo's own TLS directory and reloads the two affected services.

### Why copy out of /etc/letsencrypt instead of pointing at it

- `/etc/letsencrypt/live/` is a tree of symlinks into `archive/`,
  root-only, and sits next to ACME account keys and renewal state.
  Bind-mounting any of it into a service sandbox exposes more than
  the service needs.
- Copying into `/etc/toodoo/tls/` gives toodoo a stable, minimal,
  atomically-updatable contract: two files, fixed paths, no
  symlink-chasing inside sandboxes.

### File contract

```
/etc/toodoo/tls/fullchain.pem          # cert chain — read by Listener
                                       #   (BindReadOnlyPaths=/etc/toodoo/tls)
/etc/toodoo/tls/private/privkey.pem    # key — read by Signer ONLY
                                       #   (BindReadOnlyPaths=/etc/toodoo/tls/private)
```

Directory modes per `packaging-debian.md`: `tls/` 0755,
`tls/private/` 0700, both root-owned; files 0644 / 0600. The Listener
sandbox cannot see `private/` (its bind covers `tls/` but
`ProtectSystem=strict` + the Signer-only bind keep `private/`
unreadable: the bind into the Listener is of `tls/` *without*
`private/` — implemented as two explicit binds, `tls/fullchain.pem`
into the Listener, `tls/private` into the Signer, to avoid relying
on mode bits alone).

## Initial Issuance (operator, once)

```
sudo certbot certonly --standalone --preferred-challenges http \
     -d todo.example.org
```

- `--standalone` binds port **80** for the `http-01` challenge.
  toodoo never serves plain HTTP, so port 80 is free — no conflict
  with the Listener on 443, and no need to stop services.
- Operators with DNS automation can use `dns-01` instead (also
  enables wildcard certs); the deploy hook is identical.
- certbot runs deploy hooks on first issuance too, so the initial
  `certonly` already populates `/etc/toodoo/tls/` and reloads the
  services — no extra manual step.

## Deploy Hook

Shipped at `/etc/letsencrypt/renewal-hooks/deploy/toodoo`; certbot
executes it (as root) only after a successful issuance/renewal:

1. Copy `$RENEWED_LINEAGE/fullchain.pem` and `privkey.pem` to
   temporary names inside the target directories, set modes/owner,
   then `rename(2)` over the live names — readers never observe a
   partial file.
2. `systemctl reload toodoo-signer.service` — the Signer re-reads
   the key.
3. `systemctl reload toodoo-listener.service` — the Listener
   re-reads the chain.

Reload order is normative (key before chain): a Listener presenting
a new chain against an old key would fail handshakes; the reverse
gap is harmless (old chain + old key remain a valid pair until the
chain reload lands).

`reload` is graceful in both services: new material is loaded and
swapped atomically; in-flight connections complete on the state they
started with. (Go's `tls.Config.GetCertificate` / the Signer's key
handle make this a pointer swap.)

## Before the First Certificate

Fresh installs start services before any certificate exists. The
Listener and Signer must not crash-loop in this window:

- Both start, find no cert/key, log one actionable line
  (`no certificate at /etc/toodoo/tls/fullchain.pem — run certbot;
  see README.Debian`), and keep running.
- The Listener closes incoming connections without a handshake until
  material arrives via the deploy hook + reload.

This keeps `apt install` → `certbot certonly` working in either
order and keeps the units' restart logic out of the bootstrap story.

## Renewal

certbot's own systemd timer (shipped by the certbot package) handles
renewal cadence; toodoo adds nothing. The deploy hook above is the
entire integration surface. Renewal failures are certbot's to report
(its timer logs); the operator-facing symptom of prolonged failure
is cert expiry warnings from clients — `README.Debian` points at
`certbot renew --dry-run` for diagnosis.

## Verification

- Integration test: issue a self-signed "renewal" by invoking the
  hook with a staged `$RENEWED_LINEAGE`, assert both services reload
  and serve the new chain (openssl s_client fingerprint check).
- Integration test: fresh install without certs — services up, no
  crash-loop, log line present; then run the hook → TLS works.
- The hook is a conffile-adjacent script: lintian + shellcheck in CI.

## Open Items

- Non-certbot ACME clients (acme.sh, lego): the file contract above
  is the integration point; documenting recipes for other clients is
  a docs task, not a design change.
- ACME-inside-Listener (mentioned as an alternative in `README.md`)
  stays rejected: it would require write access from the most
  exposed service (ADR 0005 keeps the Listener filesystem-read-only
  and parser-free).
