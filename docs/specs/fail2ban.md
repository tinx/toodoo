# Spec: Fail2Ban Integration

How repeated authentication failures turn into firewall bans.
Required reading: `authentication.md` (logging contract, PROXY v2
client IP), ADR 0005 (why the Authenticator knows the client IP at
all), `packaging-debian.md` (shipped conffiles).

## How the pieces connect

1. The Listener prefixes every proxied connection with a PROXY
   protocol v2 header carrying the real client IP (ADR 0005).
2. The Authenticator logs every auth-relevant event to journald with
   structured fields and a stable, sanitized MESSAGE line
   (`authentication.md`, Logging Contract).
3. fail2ban (package dependency, jail shipped enabled) tails the
   journal, matches failures, and bans the source IP at the host
   firewall — which blocks the TCP connection to the Listener
   itself. Bans act at the outermost layer even though detection
   happens two services deep.

## Shipped Configuration

Both files are dpkg conffiles (operator edits survive upgrades).

### /etc/fail2ban/filter.d/toodoo.conf

```ini
[Definition]
# Match only journal entries from the Authenticator that carry our
# structured event field — never free-text scraping.
journalmatch = _SYSTEMD_UNIT=toodoo-authenticator.service + TOODOO_EVENT=auth_failure
               _SYSTEMD_UNIT=toodoo-authenticator.service + TOODOO_EVENT=invite_failure

failregex = ^auth_failure handle=\S{1,32} ip=<HOST>$
            ^invite_failure ip=<HOST>$

ignoreregex =
```

- `journalmatch` pre-filters on the structured `TOODOO_EVENT` field,
  so the regexes only ever see toodoo's own event lines.
- The regexes are anchored (`^...$`) against the MESSAGE format that
  `authentication.md` defines as a versioned contract. The handle
  token can only be a grammar-valid handle or the literal
  `invalid-handle` (sanitization happens before logging), so
  log-injection cannot fabricate a line that frames another IP.
- `invite_failure` counts too: invite redemption is unauthenticated
  and account-creating, the most abusable public endpoint.

### /etc/fail2ban/jail.d/toodoo.conf

```ini
[toodoo]
enabled  = true
backend  = systemd
filter   = toodoo
maxretry = 5
findtime = 10m
bantime  = 1h
bantime.increment = true
bantime.maxtime   = 48h
```

- Enabled by default: `toodoo-server` Depends on fail2ban, so the
  integration should work the moment both are installed — an
  integration the operator must remember to switch on is one that
  is off.
- `backend = systemd`: journal tailing, no log files involved
  (`authentication.md` logs to journald only).
- `bantime.increment` escalates repeat offenders geometrically up to
  48 h — long enough to blunt slow brute force, short enough that a
  household NAT mistake self-heals.
- No `banaction` override: fail2ban's distro default (nftables on
  Ubuntu 24.04) is correct. IPv6 is handled by fail2ban natively.
- Operators add their LAN to `ignoreip` via a drop-in if they want
  local exemption; `README.Debian` shows the one-liner.

## Division of Labor (vs. in-app limits)

fail2ban is the **durable, network-layer** response; it is reactive
and IP-based. It does not replace the Authenticator's own in-memory
defenses (`authentication.md`): per-IP rate limit before hashing,
per-handle exponential backoff (IP-independent — catches distributed
guessing that fail2ban structurally cannot), and the argon2id
concurrency cap. The two layers fail independently: if fail2ban is
broken or the attacker rotates IPs, the in-app limits still hold; if
the Authenticator restarts (losing in-memory counters), fail2ban's
state persists.

## Contract and Compatibility

- The MESSAGE formats matched above are part of the versioned
  logging contract in `authentication.md`. **Any change to those
  lines is a breaking change** and must ship in the same release as
  the updated filter. Because the filter is a conffile, operators
  who edited it get a dpkg prompt on such upgrades — the changelog
  must call this out loudly.
- New ban-worthy event types (e.g. a future `oidc_failure`) extend
  `journalmatch` + `failregex` additively; additions are not
  breaking.

## Verification

- CI runs `fail2ban-regex` against a golden corpus: known-good
  failure lines (must match, correct IP extracted — IPv4 and IPv6),
  success lines (must not match), and adversarial lines simulating
  injection attempts via handle/User-Agent (must not match or
  mis-extract). The corpus lives next to the filter and grows with
  every contract change.
- Integration test (same 24.04 VM as the isolation tests): 6 failed
  logins from a test IP → assert the IP lands in the fail2ban ban
  list and a TCP connect to :443 from it is refused; assert a
  successful login from another IP is unaffected.
