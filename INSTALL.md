---
title: Install
---

# Installing Odoo Bridge (prebuilt image)

This page is the fast path: run the published container image directly, no
source checkout required. If you'd rather build from source instead (e.g.
to modify the code), see [MANUAL.md](MANUAL.md)'s Installation section.

Both the bridge server and the admin dashboard ship as **one image**, run
twice with a different `command:` — there's no separate "dashboard image."

## Quick start

```bash
docker pull ghcr.io/omer-gh/odoo-bridge:latest
```

Grab [`docker-compose.yml`](docker-compose.yml) and
[`.env.example`](.env.example) from this repo, in an empty directory:

```bash
curl -O https://raw.githubusercontent.com/omer-gh/odoo-bridge-docs/main/docker-compose.yml
curl -O https://raw.githubusercontent.com/omer-gh/odoo-bridge-docs/main/.env.example
cp .env.example .env
```

Edit `.env` and set `DASHBOARD_SECRET_KEY` — it's the only required value
(generate one with `python3 -c "import secrets; print(secrets.token_hex(32))"`).
Everything else, including your Odoo connection, can be set up later
through the dashboard itself. Then:

```bash
docker compose up -d
```

This starts two containers:

- the bridge server, on `http://127.0.0.1:8088`
- the dashboard, on `http://127.0.0.1:8090`

Open the dashboard, choose a passphrase on the first-run screen, and
continue from **Part A** of the [User Manual](MANUAL.md) — first-time
setup, configuring the Odoo connection, creating bridge clients, and the
full API reference all live there. This page is only about getting the
containers running.

## Supported tags

| Tag | Meaning |
|---|---|
| `latest` | Most recent release |
| `vX.Y.Z` | A specific pinned version, e.g. `v1.4.0` |

Built for `linux/amd64` and `linux/arm64`.

## Environment variables

The variables that matter for getting started are documented in full in
the User Manual's [Installation → Configure](MANUAL.md#2-configure)
section. Summary:

| Variable | Required | Notes |
|---|---|---|
| `DASHBOARD_SECRET_KEY` | Yes | Signs dashboard login sessions |
| `ODOO_URL`, `ODOO_DB`, `ODOO_USERNAME`, `ODOO_API_KEY` | No | Optional pre-seed; otherwise set interactively via the dashboard's "Odoo connection" page after first login |
| `ODOO_ALLOW_WRITE`, `ODOO_ALLOW_DELETE` | No | Leave `0` (default) until you're ready to go beyond read-only — see [MANUAL.md](MANUAL.md#turning-on-writesdeletes) |
| `AUDIT_REDACT_FIELDS` | No | Comma-separated field names to redact in the audit log — see [SECURITY.md](SECURITY.md#audit-log-field-redaction) |

See [`.env.example`](.env.example) for the full, commented list including
the `_FILE` variants below.

## Volumes

Both containers mount a single named volume, `bridge-data`, at `/data`.
It holds:

- the SQLite database — bridge clients/keys, the permission matrix, the
  dashboard passphrase, and the Odoo connection config
- the audit log

This volume must persist across restarts and upgrades — losing it means
recreating every bridge client and reconfiguring the Odoo connection from
scratch. See [MANUAL.md](MANUAL.md#backup--recovery) for backup guidance.

## Docker secrets

If you're using Docker Swarm secrets rather than a plain `.env` file,
each of `DASHBOARD_SECRET_KEY`, `DASHBOARD_PASSPHRASE`, and `ODOO_API_KEY`
also accepts a `_FILE`-suffixed variant pointing at a file containing the
value — e.g. `DASHBOARD_SECRET_KEY_FILE=/run/secrets/dashboard_secret_key`.
If both a plain variable and its `_FILE` variant are set, `_FILE` wins.

```yaml
services:
  odoo-server:
    image: ghcr.io/omer-gh/odoo-bridge:latest
    command: ["server"]
    environment:
      DASHBOARD_SECRET_KEY_FILE: /run/secrets/dashboard_secret_key
    secrets:
      - dashboard_secret_key

secrets:
  dashboard_secret_key:
    external: true
```

This is an alternative to the plain `.env` approach in the quick start
above, not a replacement — use whichever fits how you deploy.

## What next

Once the containers are up, everything about actually *using* the tool —
first-time dashboard setup, configuring the Odoo connection, creating and
scoping bridge clients, turning on writes, reading the audit log, and the
full HTTP API reference for a script or an AI agent — is in the
[User Manual](MANUAL.md).

## License

[Apache License 2.0](LICENSE). Provided "AS IS" — no warranty of any
kind, no liability for the authors. The license text is baked into the
image itself (`/app/LICENSE`, plus an `org.opencontainers.image.licenses`
label), not just linked here.
