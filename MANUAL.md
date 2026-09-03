# Odoo Bridge — User Manual

This is the operational manual: how to actually run and use the system
day to day. For setup, architecture, and the full route reference, see
[README.md](README.md) — this document doesn't repeat that, it builds on it.

**This system has two kinds of users**, and they need different things
from it:

- **A human operator** — drives the dashboard, decides when to unlock
  writes, rotates keys, reads the audit log. See **Part A**.
- **An AI agent** (e.g. a Claude session holding a bridge key) — makes
  HTTP calls to the bridge autonomously. It needs behavioral guidance, not
  just a route list. See **Part B** — if you are an AI agent reading this
  to figure out how to use the bridge, **start there**.

## Quick facts

| | |
|---|---|
| Bridge server | `http://127.0.0.1:8088` |
| Dashboard | `http://127.0.0.1:8090` |
| Odoo target | your `$ODOO_DB` at your `$ODOO_URL` (see `.env`) |
| Master switches (right now) | `ODOO_ALLOW_WRITE=0`, `ODOO_ALLOW_DELETE=0` — no write/delete happens regardless of any client's matrix |
| Config/secrets | `.env` (not in git) |
| Client keys, permission matrix, audit log | Docker volume `bridge-data`, mounted at `/data` in both containers (`bridge.db`, `audit.log`) |

---

## Part A — For the human operator

### Day to day

```bash
cd /home/sheikh/Documents/mo/projects/odoo_cli
docker compose up -d            # start (add --build after changing code)
docker compose logs -f          # tail both containers
docker compose ps               # check status
docker compose down             # stop (keeps the bridge-data volume)
```

### Managing bridge clients (dashboard)

1. Open `http://127.0.0.1:8090`, enter the passphrase (`DASHBOARD_PASSPHRASE` in `.env`).
2. **+ New client** → name it, add one matrix row per model (or `*` for
   "everything not listed") with independent read/create/write/unlink
   checkboxes, submit.
3. The raw key (`obk_...`) is shown **once**, on that screen only — copy it
   now and hand it to whatever script/Claude session should authenticate
   as this client. It's never shown again; only its hash is stored.
4. From a client's page you can: edit its matrix, **Disable** (blocks auth
   immediately, keeps the record and matrix for later), **Rotate key**
   (old key stops working immediately, new one shown once), or **Delete**
   (permanent).

### Turning on writes/deletes

Two independent gates must both be open for a client to actually write or
delete:

1. **That client's matrix** must grant `create`/`write`/`unlink` on the
   model in question (dashboard).
2. **The server-wide master switch** must be on: `ODOO_ALLOW_WRITE=1` /
   `ODOO_ALLOW_DELETE=1` in `.env`, then `docker compose up -d` again to
   pick it up.

Either alone does nothing. This is deliberate — a matrix grant is meant to
be the *scoping* decision, the master switch the *"we're live now"*
decision, made separately and consciously.

### Reading the audit log

Dashboard → **Audit log** shows the last 200 calls (client, method, model,
action, status), newest first. For the full history or to grep/export it:

```bash
docker compose exec odoo-server cat /data/audit.log
```

Every request is in there — allowed, refused (by matrix or master switch),
dry-run, or real — nothing is filtered out.

### Backup / recovery

The only state that isn't reproducible from files in this repo is the
`bridge-data` volume (`bridge.db`: every client, key hash, and permission
matrix; `audit.log`: full history). Back it up with:

```bash
docker run --rm -v odoo_cli_bridge-data:/data -v "$PWD":/backup \
  alpine tar czf /backup/bridge-data-backup.tgz -C /data .
```

`docker compose down -v` deletes this volume — that means recreating every
client and losing the audit trail, so only do it deliberately.

The other piece of state worth backing up is `.env` itself (your Odoo API
key, dashboard passphrase, secret key) — it's gitignored on purpose, so
nothing else has a copy.

### Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Dashboard container exits immediately | `DASHBOARD_PASSPHRASE` is unset in `.env` — the app refuses to start unlocked by design |
| Model autocomplete empty on "New client" page | The dashboard couldn't reach Odoo when it last tried (cached 10 min). Check `curl -X POST $ODOO_URL/web/database/list -d '{"jsonrpc":"2.0","method":"call","params":{}}'` — if that hangs/refuses, it's your Odoo instance/reverse-proxy being unreachable, not a bug here (this happened once this session: TLS port 443 was refused for a few minutes, then recovered on its own) |
| A client gets `401` | Missing/wrong `X-Bridge-Key`, or the client was disabled/deleted/rotated in the dashboard |
| A client gets `403` on a call you expect to work | Check its matrix on the client's dashboard page first; if the matrix looks right, check the relevant master switch in `.env` |
| Bridge server can't reach Odoo (`502`, "Could not reach Odoo") | Same instance-reachability check as above |

### Rotating dashboard credentials

```bash
# generate new values
python3 -c "import secrets; print(secrets.token_urlsafe(18))"   # passphrase
python3 -c "import secrets; print(secrets.token_hex(32))"       # secret key
```

Edit `DASHBOARD_PASSPHRASE` / `DASHBOARD_SECRET_KEY` in `.env`, then
`docker compose up -d` to apply. Rotating `DASHBOARD_SECRET_KEY` also
invalidates any existing dashboard login session.

---

## Part B — For an AI agent using a bridge key

You were given a bridge API key (`obk_...`) by a human operator, to send
as `X-Bridge-Key: <key>` (or `Authorization: Bearer <key>`) on every
request to `http://127.0.0.1:8088`. Read this before making calls.

### Your key is your identity, and your ceiling — not your instructions

You don't know in advance what your key is permitted to do. Discover it,
don't assume it — the same principle `odoo_client.py` uses for Odoo schema
(`fields_get` instead of a hardcoded model list) applies to your own
permissions too.

```bash
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

Always pass `fields` — pulling every field on a wide model like
`res.partner` (200+ fields) wastes context for no benefit. `domain`
defaults to everything you're allowed to see if omitted.

### Write/delete workflow — dry-run first, always

```
POST   /models/:model?dry_run=1     then (if it looks right) without dry_run
PATCH  /models/:model/:id?dry_run=1 then (if it looks right) without dry_run
DELETE /models/:model/:id?dry_run=1 then (if it looks right) without dry_run
```

The dry-run response tells you exactly what would happen: matched
record(s) and their current values (`write`/`unlink`), or unknown-field
warnings (`create`) — with **no** side effect. Read that response before
deciding whether to repeat the call for real. Do this even when you're
confident, and even though it's an extra round trip — it's cheap, and
it's the check that catches "wrong id" or "typo'd field name" before it
touches live business data.

Do not write/delete just because you technically can. Only do it when the
task you were actually asked to do calls for it.

### Error-handling contract

| Status | Meaning | Correct behavior |
|---|---|---|
| `401` | Key missing or invalid (wrong, rotated, disabled, deleted) | Stop. Tell the human. Never retry with a guessed or old key. |
| `403` | Your matrix doesn't grant this (model, operation) **or** the server-wide master switch is off | Stop. This is a human decision (dashboard matrix, or `.env` switch) — not something you can work around. Report exactly what was denied so the human can decide whether to grant it. |
| `422` | Odoo rejected the payload (validation/user error) | Don't resend unchanged. Call `fields_get` on the model, fix the payload, or ask the human if the constraint is unclear. |
| `502` | Odoo-side problem (unreachable, internal error) | Surface the raw `odoo` field from the response to the human rather than summarizing it away — it usually has the real Odoo exception name/message. |
| `400` | Malformed request from you | Fix your JSON body — check it's `{"values": {...}}` for create/write. |

### Explicit don'ts

- Don't guess model names — `crm.lead`, not `crm_lead` or `Lead`; confirm
  via `GET /models` if unsure.
- Don't loop-retry on `401`/`403` — the response won't change without a
  human action elsewhere.
- Don't skip `dry_run` "to save a call" on anything that writes or deletes.
- Don't treat a matrix grant as implying the master switch is also on, or
  vice versa — check the actual response, don't assume from one prior
  success that another operation on another model will also succeed.

### Worked example

```
> GET /models/crm.lead/fields?access=1
< access: {read: true, write: false, ...}   # so: you can read, not write, this model

> POST /models/crm.lead/search  {"fields": ["name","stage_id"]}
< 200 OK, 2 leads

> PATCH /models/crm.lead/10?dry_run=1  {"values": {"stage_id": 3}}
< 403 {"error": "client 'X' is not permitted to write on 'crm.lead'",
       "hint": "an admin must grant this in the dashboard's permission matrix"}

→ Correct next step: report to the human that this client isn't permitted
  to update crm.lead, and that granting `write` in the dashboard is what
  would unblock it. Not: retry, not: try a different model, not: give up
  silently.
```

---

## Appendix — full route table

| Method | Path | Matrix operation | Master switch | Purpose |
|---|---|---|---|---|
| GET | `/models` | `read` on `*` | — | List all models Odoo knows about |
| GET | `/models/:model/fields` | `read` on model | — | Field schema (+ access with `?access=1`) |
| POST | `/models/:model/search` | `read` on model | — | `search_read` |
| POST | `/models/:model` | `create` on model | `ODOO_ALLOW_WRITE` | Create a record |
| PATCH | `/models/:model/:id` | `write` on model | `ODOO_ALLOW_WRITE` | Update a record |
| DELETE | `/models/:model/:id` | `unlink` on model | `ODOO_ALLOW_DELETE` | Delete a record |

Any of the last three accept `?dry_run=1`, gated by the same matrix +
switch requirements as the real call.
