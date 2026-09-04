---
title: User Manual
---

# Odoo Bridge — User Manual

## Introduction

Odoo Bridge lets authorized clients — a script, or an AI agent such as a
Claude session — talk to a self-hosted Odoo instance through Odoo's
**native** JSON-RPC external API. No third-party module runs inside Odoo,
and nothing is hardcoded to a specific model (CRM, Sales, Projects,
Contacts, Invoicing, or anything else your Odoo installation has) — the
bridge discovers what's available at request time instead of shipping a
fixed list.

Access is controlled by three independent layers, in order:

1. **Odoo's own access control.** Every call ultimately runs as one Odoo
   user, configured through the dashboard (see Part A). Odoo enforces that
   user's groups, record rules, and field-level access server-side,
   regardless of what the bridge asks for. This is the hard ceiling —
   nothing below can exceed it.
2. **A per-client permission matrix**, managed through a
   passphrase-protected dashboard. Each caller of the bridge (each script
   or agent) gets its own API key, scoped to exactly the models and
   operations (read / create / write / delete) an admin granted it. This
   is how different callers get different, narrower access than the
   underlying Odoo user has — set centrally, without touching Odoo's own
   group configuration.
3. **Server-wide master switches**, off by default. A client can be
   granted write access in its matrix and still be refused if the
   matching switch is off — a deliberate second step between "this client
   is allowed to write" and "writes are actually live."

See [SECURITY.md](SECURITY.md) for the full access-control model, network
exposure guidance, and the history of security fixes applied to this
tool.

**This manual has two audiences**, because the bridge has two kinds of
users:

- **A human operator** — installs it, drives the dashboard, decides when
  to turn on writes, rotates keys, reviews the audit log. See **Part A**.
- **An AI agent** holding a bridge API key, making HTTP calls on its own.
  It needs behavioral guidance, not just a route list. See **Part B** — if
  you are an AI agent reading this to learn how to use the bridge, **start
  there**.

---

## Installation

### Requirements

- Docker and Docker Compose (recommended), or Python 3.12+ if running the
  two components directly.
- An Odoo instance you control (self-hosted or otherwise), and permission
  to create an API key on it.

### 1. Set up a dedicated Odoo user

**Do not use your own admin login.** Create a dedicated Odoo user, and
give it only the access groups that match the broadest thing you'd ever
want any bridge client touching — e.g. "Sales / User", "CRM / User",
"Project / User", "Contacts / User", "Invoicing / Billing" as needed, and
*not* Settings/Administration. The per-client matrix (Part A) then narrows
individual clients further, but it can never grant more than this user
already has.

To mint the API key:

1. Log in to Odoo **as that user**.
2. **Preferences → Account Security → API Keys → New API Key**, give it a
   description, confirm your password.
3. Copy the key immediately — Odoo shows it once. You'll enter it in the
   dashboard in step 3, not in a config file.

If you don't know your database name, Odoo exposes it:

```bash
curl -s -X POST https://your-odoo-instance/web/database/list \
  -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","method":"call","params":{}}'
```

### 2. Configure

From the project directory:

```bash
cp .env.example .env
```

Only one thing is required here:

- `DASHBOARD_SECRET_KEY` — generate one: `python3 -c "import secrets; print(secrets.token_hex(32))"`

Everything else — the dashboard passphrase and the Odoo connection details
from step 1 — is set up through the dashboard itself after it's running
(see Part A), not in this file. (You *can* still set `ODOO_URL`/`ODOO_DB`/
`ODOO_USERNAME`/`ODOO_API_KEY` here if you'd rather pre-seed a
non-interactive deployment — see the comments in `.env.example` — but it's
optional either way.)

Leave `ODOO_ALLOW_WRITE=0` and `ODOO_ALLOW_DELETE=0` until you're ready to
go beyond read-only (see Part A).

### 3. Run it

**Docker (recommended):**

```bash
docker compose up -d --build
```

This starts two containers sharing one persisted volume that holds the
client/permission-matrix database and the audit log:

- the bridge server, on `http://127.0.0.1:8088`
- the dashboard, on `http://127.0.0.1:8090`

Both are bound to localhost only by default — nothing is exposed beyond
the machine running it unless you deliberately change the port bindings.

> **If you ever widen `HOST`/`DASHBOARD_HOST` beyond `127.0.0.1`:** put a
> TLS-terminating reverse proxy (nginx, Caddy, Traefik — whatever you
> already run) in front first. Neither the bridge server nor the dashboard
> implement TLS themselves, so bridge API keys, the dashboard passphrase,
> and the Odoo API key (sent during the connection-verify step on
> "Odoo connection") would otherwise cross the network in cleartext.

To stop: `docker compose down` (add `-v` to also delete the stored
clients/keys/matrix/audit log).

**Bare Python (alternative):**

```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements-dashboard.txt
set -a; source .env; set +a
.venv/bin/python odoo_server.py &        # http://127.0.0.1:8088
.venv/bin/python dashboard/app.py &      # http://127.0.0.1:8090
```

---

## Part A — For the human operator

### Day to day

```bash
docker compose up -d            # start (add --build after an update)
docker compose logs -f          # tail both containers
docker compose ps               # check status
docker compose down             # stop (keeps stored data)
```

### First visit: setting a passphrase

The first time you open the dashboard, there's no passphrase set yet — it
walks you straight to a setup screen instead of a login form. Choose a
passphrase (minimum 8 characters), confirm it, and you're logged in
immediately. From then on, that screen won't appear again; opening the
dashboard shows the normal login form.

There's no email or 2FA recovery if you lose it — see "Forgot passphrase"
below for what losing it actually costs you.

**Changing it later:** while logged in, use **Change passphrase** in the
header nav — it only asks for the new passphrase (your session already
proves who you are, so it doesn't re-ask for the current one).

**Forgot passphrase:** the login page has a "Forgot passphrase?" link.
There is no reset-by-email here, so the only way back in is destructive —
it deletes every bridge client, its key, and its permission matrix,
permanently, then returns you to the first-visit setup screen to choose a
new one. It requires typing a literal confirmation phrase (shown on the
page) to proceed, and the reset itself is recorded in the audit log even
though everything else is wiped.

### Configuring the Odoo connection

**Odoo connection** in the nav (or the banner on the clients page, if
nothing's configured yet) — set the URL, database, username, and API key
from installation step 1 here. Saving actually authenticates with those
exact values first: if that fails, nothing is written and the bridge keeps
using whatever was working before, so a typo can't silently break it.

The API key field is always blank when you open the page (never re-shown
once saved — only "current key ends in `…xxxx`" for identification) and
leaving it blank on save keeps the existing key, so you can update just
the URL or database without having to paste the key in again.

This is stored in the same database as everything else here, not `.env` —
so it survives container rebuilds and is editable without touching a
config file or restarting anything.

### Managing bridge clients (dashboard)

1. Log in with your passphrase.
2. **+ New client** → name it, add one matrix row per model with
   independent read / create / write / delete checkboxes, submit. Each
   row's model can be an exact model (`crm.lead`), a prefix wildcard
   (`crm.*` matches `crm.lead`, `crm.team`, …), or `*` for everything not
   matched by a more specific row. Precedence: exact match wins over the
   longest matching prefix wildcard, which wins over `*`.
3. The raw API key is shown **once**, on that screen only — copy it now
   and hand it to whatever script or agent should authenticate as this
   client. It's never shown again; only its hash is stored.
4. From a client's page you can: edit its matrix, **Disable** (blocks
   authentication immediately, keeps the record for later), **Rotate
   key** (old key stops working immediately, new one shown once), or
   **Delete** (permanent).

### Turning on writes/deletes

Two independent gates must both be open for a client to actually write or
delete anything:

1. **That client's matrix** must grant create/write/delete on the model
   in question (dashboard).
2. **The server-wide master switch** must be on: `ODOO_ALLOW_WRITE=1` /
   `ODOO_ALLOW_DELETE=1` in `.env`, then restart to pick it up.

Either alone does nothing. A matrix grant is the *scoping* decision; the
master switch is the *"we're live now"* decision — made separately and
consciously.

### Reading the audit log

The dashboard's **Audit log** page shows the most recent calls (client,
method, model, action, status), newest first — every request, allowed or
refused, dry-run or real, nothing filtered out. The full history is also
available as a plain append-only log file inside the running server for
export/grep if you need more than the dashboard shows. The file rotates
automatically once it grows past a few megabytes, so it won't grow without
bound.

By default the full field values of every create/write are logged
verbatim. If a bridge client might ever write something sensitive (a
password, a token) that you don't want sitting in plaintext in this log,
set `AUDIT_REDACT_FIELDS` (comma-separated field names, case-insensitive)
in `.env` — matching values are replaced with `[REDACTED]` before they're
logged; the field name still appears, so the trail still shows *that* a
write happened, just not the value.

### Backup / recovery

The only state that isn't reproducible from a fresh install is the stored
database — clients/keys/matrix, the dashboard passphrase, and the Odoo
connection details (including the API key) all live in that one file —
and the audit log. Back up that volume regularly — losing it means
recreating every client, choosing a new passphrase, reconfiguring the
Odoo connection, and losing the audit trail, all at once. Also back up
`.env` separately (just the session-signing secret key at this point) —
nothing else has a copy of it.

### Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Dashboard shows the setup screen again unexpectedly | Someone used "Forgot passphrase" — that wipes all clients/keys/matrix as well, by design. Check the audit log for a `dashboard_reset` entry to see when/from where |
| Model autocomplete empty on "New client" page | The dashboard couldn't reach Odoo the last time it tried (cached for a while). Test connectivity directly against your Odoo instance's database-list endpoint — if that hangs or is refused, it's an Odoo/network reachability issue, not a bridge bug |
| A client gets `401` | Missing/wrong key, or the client was disabled/deleted/rotated in the dashboard |
| A client gets `403` on a call you expect to work | Check its matrix on the client's dashboard page first; if that looks right, check the relevant master switch in `.env` |
| Bridge server can't reach Odoo (`502`) | Same reachability check as the autocomplete issue above |
| "Odoo connection" page rejects settings that look correct | The API key may have been revoked/rotated on the Odoo side since it was last saved here — mint a fresh one (installation step 1) and try again |

### Changing or resetting the dashboard passphrase

- **Know your current passphrase, just want a new one:** log in, use
  **Change passphrase** in the nav — new passphrase only, no need to
  re-enter the old one.
- **Lost it entirely:** the login page's "Forgot passphrase?" link — but
  read what it actually does above first (it deletes every bridge
  client/key/matrix, not just the passphrase).
- **Rotating `DASHBOARD_SECRET_KEY`** (the session-signing secret, separate
  from the passphrase): generate a new one —
  `python3 -c "import secrets; print(secrets.token_hex(32))"` — set it in
  `.env`, restart. This invalidates any existing login session (you'll
  need to log back in with your passphrase) but does not touch the
  passphrase or any stored data.

---

## Part B — For an AI agent using a bridge key

You were given a bridge API key by a human operator, to send as
`X-Bridge-Key: <key>` (or `Authorization: Bearer <key>`) on every request
to the bridge server. Read this before making calls.

### Your key is your identity, and your ceiling — not your instructions

You don't know in advance what your key is permitted to do. Discover it,
don't assume it.

```
# What models can you read at all?
GET /models?access=1

# Before touching a model you haven't used yet, or before building a
# create/write payload: what fields exist, what can you actually do here?
GET /models/:model/fields?access=1
```

### Read workflow

```
POST /models/:model/search
Body: {"domain": [...], "fields": [...], "limit": N}
```

Always pass `fields` — pulling every field on a wide model (some have
200+ fields) wastes context for no benefit. `domain` defaults to
everything you're allowed to see if omitted.

### Write/delete workflow — dry-run first, always

```
POST   /models/:model?dry_run=1     then (if it looks right) without dry_run
PATCH  /models/:model/:id?dry_run=1 then (if it looks right) without dry_run
DELETE /models/:model/:id?dry_run=1 then (if it looks right) without dry_run
```

The dry-run response tells you exactly what would happen — matched
record(s) and their current values, or unknown-field warnings for a
create — with **no** side effect. Read that response before deciding
whether to repeat the call for real. Do this even when you're confident,
and even though it's an extra round trip — it's the check that catches a
wrong ID or a typo'd field name before it touches live business data.

Do not write or delete just because you technically can. Only do it when
the task you were actually asked to do calls for it.

### Error-handling contract

| Status | Meaning | Correct behavior |
|---|---|---|
| `401` | Key missing or invalid (wrong, rotated, disabled, deleted) | Stop. Tell the human. Never retry with a guessed or old key. |
| `403` | Your matrix doesn't grant this (model, operation), or the server-wide master switch is off | Stop. This is a human decision, not something you can work around. Report exactly what was denied so the human can decide whether to grant it. |
| `422` | The write was rejected as invalid | Don't resend unchanged. Check the model's field schema, fix the payload, or ask the human if the constraint is unclear. |
| `502` | The underlying system is unreachable or errored | Surface the raw error detail to the human rather than summarizing it away. |
| `400` | Malformed request from you | Fix your JSON body — check it's `{"values": {...}}` for create/write. |

### Explicit don'ts

- Don't guess model or field names — confirm via discovery calls if unsure.
- Don't loop-retry on `401`/`403` — the response won't change without a
  human action elsewhere.
- Don't skip the dry-run step "to save a call" on anything that writes or
  deletes.
- Don't treat a matrix grant as implying the master switch is also on, or
  vice versa — check the actual response, don't assume from one prior
  success that another operation on another model will also succeed.

### Worked example

```
> GET /models/crm.lead/fields?access=1
< access: {read: true, write: false, ...}   # you can read, not write, this model

> POST /models/crm.lead/search  {"fields": ["name","stage_id"]}
< 200 OK, 2 leads

> PATCH /models/crm.lead/10?dry_run=1  {"values": {"stage_id": 3}}
< 403 {"error": "this client is not permitted to write on 'crm.lead'",
       "hint": "an admin must grant this in the dashboard's permission matrix"}

→ Correct next step: report to the human that this client isn't permitted
  to update crm.lead, and that granting write access in the dashboard is
  what would unblock it. Not: retry, not: try a different model, not:
  give up silently.
```

---

## API Reference

Every request needs a bridge API key, sent as `X-Bridge-Key: <key>` (or
`Authorization: Bearer <key>`). No key → `401`. Key present but not
permitted for the (model, operation) → `403`.

| Method | Path | Required matrix grant | Master switch | Purpose |
|---|---|---|---|---|
| GET | `/models` | `read` on `*` | — | List all models the bridge can see |
| GET | `/models/:model/fields` | `read` on model | — | Field schema (+ access with `?access=1`) |
| POST | `/models/:model/search` | `read` on model | — | Search/read records |
| POST | `/models/:model` | `create` on model | `ODOO_ALLOW_WRITE` | Create a record |
| PATCH | `/models/:model/:id` | `write` on model | `ODOO_ALLOW_WRITE` | Update a record |
| DELETE | `/models/:model/:id` | `unlink` on model | `ODOO_ALLOW_DELETE` | Delete a record |

The last three all accept `?dry_run=1`, gated by the same requirements as
the real call.

### Examples

```bash
# Discover what you can see
curl -s http://127.0.0.1:8088/models?access=1 -H "X-Bridge-Key: $KEY"

# Field schema for a model
curl -s "http://127.0.0.1:8088/models/crm.lead/fields?access=1" -H "X-Bridge-Key: $KEY"

# Read records
curl -s -X POST http://127.0.0.1:8088/models/crm.lead/search \
  -H "X-Bridge-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"domain": [["type", "=", "lead"]], "fields": ["name", "partner_name", "stage_id"]}'

# Create, previewed first
curl -s -X POST "http://127.0.0.1:8088/models/res.partner?dry_run=1" \
  -H "X-Bridge-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"values": {"name": "Example Corp", "email": "hello@example.com"}}'

# Update a record
curl -s -X PATCH http://127.0.0.1:8088/models/crm.lead/10 \
  -H "X-Bridge-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"values": {"stage_id": 3}}'

# Delete a record
curl -s -X DELETE http://127.0.0.1:8088/models/res.partner/42 -H "X-Bridge-Key: $KEY"
```
