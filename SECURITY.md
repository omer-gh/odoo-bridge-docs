---
title: Security
---

# Security

## How access is layered

Three independent controls, each narrower than the last:

1. **Odoo's own access control** — every call ultimately runs as one Odoo
   user. Odoo enforces that user's groups and record rules regardless of
   what the bridge asks for. Nothing below this can exceed it.
2. **A per-client permission matrix** — each bridge API key is scoped to
   exactly the models and operations an admin granted it, set through the
   dashboard.
3. **Server-wide master switches** (`ODOO_ALLOW_WRITE`, `ODOO_ALLOW_DELETE`)
   — off by default. A client can be granted write in its matrix and still
   be refused if the switch is off.

The dashboard itself is gated by a passphrase you choose on first visit
(see the [User Manual](MANUAL.md)), with:

- CSRF protection on every state-changing action
- Rate limiting on login attempts
- A separate, randomly-generated recovery code (shown once) required to
  reset access if the passphrase is lost — not a phrase anyone could guess
  or that's the same across installations
- An audit log of every request, allowed or refused, with automatic
  rotation so it can't grow without bound

## Network exposure

The bridge server and dashboard bind to `127.0.0.1` by default — nothing
is reachable beyond the machine running them unless you deliberately widen
`HOST`/`DASHBOARD_HOST`. **If you do widen either, put a TLS-terminating
reverse proxy in front first** (nginx, Caddy, Traefik — whatever you
already run). Neither service implements TLS itself, so bridge API keys,
the dashboard passphrase, and the Odoo API key would otherwise cross the
network in cleartext.

## Audit log field redaction

By default, the full field values of every create/write are recorded in
the audit log. If a bridge client might ever write something sensitive
you don't want logged in plaintext, set `AUDIT_REDACT_FIELDS` (see the
[User Manual](MANUAL.md)) — matching field values are replaced with
`[REDACTED]` before logging; the field name still appears, so the trail
still shows a write happened, just not the value.

## Security history

A review conducted on 2026-09-04, after an actual incident (a bridge
client's data was deleted via an unauthenticated request), found and fixed
nine issues — from the root cause (an unauthenticated destructive reset
gated only by a phrase identical in every install) through defense-in-depth
hardening (CSRF, rate limiting, non-root containers, bounded request
sizes, log rotation, trimmed error detail, optional field redaction). All
nine are fixed. The layered access model, recovery-code reset, CSRF
protection, and rate limiting described above are the result.

## Reporting a concern

Open an issue on this repo (the public docs repo) describing what you
found — it'll be picked up from there.
