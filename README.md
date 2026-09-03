---
title: README
---

# odoo-bridge

> Setting this up or changing the code? Keep reading. Already running and
> want to know how to actually use it day to day — as an operator or as an
> AI agent with a bridge key? See [MANUAL.md](MANUAL.md).

Lets authorized clients (a Claude session, a script) talk to a self-hosted
Odoo 18.0 Community instance through Odoo's **native** JSON-RPC external
API (`/jsonrpc`) — no third-party MCP module, no per-model code. Whatever
the underlying Odoo user can see and do (CRM, Sales, Projects, Contacts,
Invoicing, anything) is the ceiling of what this tool can do. Nothing is
hardcoded to a specific model.

Three independent layers of control, in order:

1. **Odoo's own ACLs.** Every call ultimately runs as one Odoo user (the
   one whose API key is in `.env`). Odoo enforces that user's groups,
   record rules, and field-level access server-side, regardless of what
   this tool asks for. This is the hard ceiling — nothing below can exceed it.
2. **A per-client permission matrix**, managed through a passphrase-protected
   dashboard. Each caller of the bridge (a Claude session, a script) gets
   its own bridge API key, scoped to exactly the (model, operation) pairs
   an admin granted it. This is how you give different callers different,
   narrower access than the underlying Odoo user has — centrally, without
   touching Odoo's own group config.
3. **Server-wide master switches** (`ODOO_ALLOW_WRITE`, `ODOO_ALLOW_DELETE`),
   off by default. A client can be granted write in its matrix and still get
   refused if the matching switch is off — the "don't let one dashboard
   checkbox alone turn on live writes" backstop.

## Layout

- [`odoo_client.py`](odoo_client.py) — generic Odoo JSON-RPC client (stdlib only).
- [`odoo_server.py`](odoo_server.py) — the bridge: authenticates bridge clients, enforces their matrix, calls Odoo.
- [`db.py`](db.py) — SQLite store for bridge clients and their permission matrix.
- [`dashboard/`](dashboard/app.py) — passphrase-protected Flask UI for managing clients/keys/matrix and viewing the audit log.
- [`audit.py`](audit.py) — append-only JSON-lines audit logger.
- [`dev/mock_odoo.py`](dev/mock_odoo.py) — fake Odoo `/jsonrpc` endpoint for local testing without touching a real instance.
- [`docker-compose.yml`](docker-compose.yml), [`Dockerfile.server`](Dockerfile.server), [`Dockerfile.dashboard`](Dockerfile.dashboard) — deployment.

`odoo_client.py`, `odoo_server.py`, `audit.py`, `db.py` use only the Python
3 standard library. Only the dashboard needs a dependency (Flask).

## Which Odoo user should this run as

**Do not bind this to your own admin login.** Create a dedicated Odoo user,
e.g. `claude-bot@yourdomain.com`, and give it only the access groups that
match the broadest thing you'd ever want any bridge client touching — e.g.
"Sales / User", "CRM / User", "Project / User", "Contacts / User",
"Invoicing / Billing" as needed, and *not* Settings/Administration. The
per-client matrix (below) then narrows *individual* clients further, but it
can never grant more than this user already has.

To mint the API key:

1. Log in to Odoo **as that user**.
2. **Preferences → Account Security → API Keys → New API Key**, description e.g. "bridge", confirm your password.
3. Copy the key immediately — Odoo shows it once. This is your `ODOO_API_KEY`.

Don't know your database name? Odoo exposes it:

```bash
curl -s -X POST https://your-instance/web/database/list \
  -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","method":"call","params":{}}'
```

## Setup

```bash
cp .env.example .env
```

Fill in `ODOO_URL`, `ODOO_DB`, `ODOO_USERNAME`, `ODOO_API_KEY`, and
`DASHBOARD_PASSPHRASE` (a passphrase of your choosing — the dashboard
refuses to start without one) and `DASHBOARD_SECRET_KEY` (generate with
`python3 -c "import secrets; print(secrets.token_hex(32))"`). Leave
`ODOO_ALLOW_WRITE=0` / `ODOO_ALLOW_DELETE=0` until you're ready.

## Running it — Docker (recommended)

```bash
docker compose up -d --build
```

This starts two containers sharing one persisted SQLite volume
(`bridge-data`) that holds the client/permission-matrix database and the
audit log:

- **odoo-server** — the bridge, on `127.0.0.1:8088` (host-mapped, not exposed beyond localhost)
- **odoo-dashboard** — the admin UI, on `127.0.0.1:8090`

A third, optional service (`odoo-client`) is a throwaway container on the
same Docker network for poking at the bridge without exposing it further —
it doesn't start with plain `up`:

```bash
docker compose --profile tools run --rm odoo-client bash
# inside: curl http://odoo-server:8088/models -H "X-Bridge-Key: ..."
```

To stop: `docker compose down` (add `-v` to also drop the bridge database —
you'll lose every client/key/matrix you configured).

## Running it — bare Python (alternative)

```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements-dashboard.txt
set -a; source .env; set +a
.venv/bin/python odoo_server.py &        # 127.0.0.1:8088
.venv/bin/python dashboard/app.py &      # 127.0.0.1:8090
```

## Using the dashboard

Open `http://127.0.0.1:8090`, enter your passphrase.

- **New client** → name it (e.g. `claude-session`, `nightly-report-script`),
  build its permission matrix: one row per model (`res.partner`, `crm.lead`,
  …) or `*` for "everything not listed individually", with independent
  read/create/write/unlink checkboxes. An exact-model row always wins over
  a `*` row for that model. Submitting shows the raw bridge API key **once**
  — copy it now, only its hash is stored.
- **Client page** — edit its matrix, disable it (refuses auth immediately,
  keeps the record), rotate its key (old one stops working immediately), or
  delete it outright.
- **Audit log** — every request any client made, allowed or refused,
  dry-run or real, newest first.

Server-wide `ODOO_ALLOW_WRITE`/`ODOO_ALLOW_DELETE` are shown for reference
but are env-configured, not editable from the UI — that's deliberate,
matching the "one more deliberate step than a checkbox" design.

## Calling the bridge

Every request needs a bridge API key from the dashboard, sent as
`X-Bridge-Key: <key>` (or `Authorization: Bearer <key>`). No key → `401`.
Key present but the matrix doesn't grant that (model, operation) → `403`.

### `GET /models`

List models Odoo knows about. Requires a `read` grant on `*` specifically
(it spans every model). Add `?access=1` to also run `check_access_rights`
per model for the underlying Odoo user (slower), or `?limit=N`.

```bash
curl -s http://127.0.0.1:8088/models -H "X-Bridge-Key: $KEY"
```

### `GET /models/:model/fields`

Field discovery: names, types, required/readonly flags, relations. Add
`?access=1` for a `read`/`write`/`create`/`unlink` summary for the
underlying Odoo user on that model. Requires a `read` grant on that model
(or `*`).

```bash
curl -s "http://127.0.0.1:8088/models/crm.lead/fields?access=1" -H "X-Bridge-Key: $KEY"
```

### `POST /models/:model/search`

`search_read`. Requires `read`. Body: `{"domain": [...], "fields": [...], "limit": N, "offset": N, "order": "field desc"}`, all keys optional.

```bash
curl -s -X POST http://127.0.0.1:8088/models/crm.lead/search \
  -H "X-Bridge-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"domain": [["type", "=", "lead"]], "fields": ["name", "partner_name", "stage_id"]}'
```

### `POST /models/:model` — create

Requires `create` in the matrix **and** `ODOO_ALLOW_WRITE=1` server-wide.

```bash
curl -s -X POST "http://127.0.0.1:8088/models/res.partner?dry_run=1" \
  -H "X-Bridge-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"values": {"name": "Acme Corp", "email": "hello@acme.example"}}'
```

### `PATCH /models/:model/:id` — write

Requires `write` in the matrix **and** `ODOO_ALLOW_WRITE=1`.

```bash
curl -s -X PATCH http://127.0.0.1:8088/models/crm.lead/10 \
  -H "X-Bridge-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"values": {"stage_id": 3}}'
```

### `DELETE /models/:model/:id` — unlink

Requires `unlink` in the matrix **and** `ODOO_ALLOW_DELETE=1`.

```bash
curl -s -X DELETE http://127.0.0.1:8088/models/res.partner/42 -H "X-Bridge-Key: $KEY"
```

Every create/write/delete route accepts `?dry_run=1`: resolves what the
operation would touch (matched record(s), current vs. proposed values, or
unknown-field warnings for a create) without calling Odoo's
create/write/unlink. Dry-run needs the same matrix grant + master switch as
the real operation, so what you preview is exactly what would run.

## Error handling

Odoo's real error payload is surfaced, not swallowed:

```json
{
  "error": "You are not allowed to modify this document.",
  "odoo": {"name": "odoo.exceptions.AccessError", "message": "...", "debug": "..."}
}
```

HTTP status: `401` no/bad bridge key, `403` matrix/master-switch refusal or
Odoo `AccessError`, `422` Odoo `ValidationError`/`UserError`, `502` other
Odoo-side errors, `400` malformed body, `404` unknown route.

## Testing without touching your real instance

`dev/mock_odoo.py` is a tiny fake `/jsonrpc` endpoint (in-memory
`res.partner`/`crm.lead`) used to verify the whole stack — bridge-key auth,
matrix enforcement independent of the master switches, dry-run, the
dashboard's client creation/matrix editing/audit view, and the Docker
Compose deployment itself — before ever pointing at a real instance:

```bash
python3 dev/mock_odoo.py &   # 127.0.0.1:8069
export ODOO_URL=http://127.0.0.1:8069 ODOO_DB=mock \
       ODOO_USERNAME=claude.bot ODOO_API_KEY=mock-api-key \
       ODOO_ALLOW_WRITE=1 ODOO_ALLOW_DELETE=1
python3 odoo_server.py &     # 127.0.0.1:8088
```

## Worked examples (against a real instance)

```bash
# All CRM leads (client needs read on crm.lead or *)
curl -s -X POST http://127.0.0.1:8088/models/crm.lead/search \
  -H "X-Bridge-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"fields": ["name", "partner_name", "email_from", "stage_id", "user_id"]}'

# A project's tasks
curl -s -X POST http://127.0.0.1:8088/models/project.task/search \
  -H "X-Bridge-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"domain": [["project_id.name", "=", "My Project"]], "fields": ["name", "stage_id", "date_deadline"]}'

# Create a contact (client needs create on res.partner or *, plus ODOO_ALLOW_WRITE=1)
curl -s -X POST http://127.0.0.1:8088/models/res.partner \
  -H "X-Bridge-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"values": {"name": "Jane Doe", "email": "jane@example.com"}}'
```
