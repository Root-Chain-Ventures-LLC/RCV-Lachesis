# Lachesis HTTP API

This guide is for people who write scripts and agents against Lachesis. The machine-readable description is
the OpenAPI 3.1 document at `GET /api/openapi.json`. It is generated from the code: the request and
response schemas are the Go types the handlers use, and a test fails when a route in the router is not
documented or a documented route does not exist. This page explains what the document cannot:
authentication, capabilities, errors, paging, limits, discovery and a few worked examples. The MCP endpoint
has its own guide, `docs/mcp.md`.

(Throughout, "Lachesis" is the product name the API reports in `/api/integrations` and `/llms.txt`.)

## Where things are

| What | Where | Who can read it |
|---|---|---|
| OpenAPI 3.1 description | `GET /api/openapi.json` | any signed-in session or API token (401 otherwise) |
| Discovery summary | `GET /api/integrations` | any signed-in session or API token |
| Plain-text summary for AI agents | `GET /llms.txt` | anyone (no data, no secrets) |
| This guide, the MCP guide | `docs/api.md`, `docs/mcp.md` | |

The document lists only the routes this installation serves right now: the `/mcp` and OAuth routes only
while the "MCP server" setting is on (the document, `/api/integrations` and `/llms.txt` all follow it), and
the plan, audit and network-facts routes when the installation has those parts, which a normal deployment
always does (the network-facts routes answer "not connected" until a source is configured). `servers[0].url` is the
configured `LACHESIS_BASE_URL`.

## Authentication

Create an API token in **My account -> API tokens** (or `POST /api/tokens` from a browser session). The
secret (`lch_<id>_<secret>`) is shown once; only its SHA-256 is stored. Tokens expire (you choose, at most
365 days), can be revoked at any time, and an account may hold up to 50 live ones, so a token can be rotated
without downtime: make the new one, switch the client, revoke the old one.

Send the token in either header; they are the same credential:

```
Authorization: Bearer lch_...
X-Api-Key: lch_...
```

- Sending both with two different Bearer tokens is `401 conflicting_credentials`, with a `WWW-Authenticate`
  challenge. Sending both with the same token is fine. An `Authorization` header that is not Bearer (Basic,
  added by a reverse proxy, say) is ignored when `X-Api-Key` is present, and the key is used.
- A bad, expired or revoked token is always `401 invalid_token` (the reason is never revealed); it never
  falls back to a browser cookie. A request with a token needs no CSRF token and ignores any cookie.
- A token owned by a single-sign-on account stops working when that account is disabled, when the identity
  provider changes, or when the role map changes until its owner signs in again
  (`401 token_owner_must_sign_in`). Local (break-glass) accounts may hold tokens only while single sign-on is
  off.
- Requests that present a token in either header (`Authorization` or `X-Api-Key`) are rate limited per client address (default 600 a minute;
  Settings, `bearer_rate_ip`): over it, `429 rate_limited` with `Retry-After`.

### A token's authority

A token never has more authority than its owner. On **every request** its stored scopes are intersected with
what the owner's **current** role allows (a viewer may hold `read`; operators and administrators `read`,
`plan` and `apply`), so demoting or disabling the owner takes effect on the next call, and tokens minted
earlier keep working with the reach they had. A token may also be limited to some servers; a server outside
its list is `403 server_not_allowed`, never an empty answer.

### Capabilities

Every route that accepts a token declares a **capability**, named `domain.action` for what is allowed. The
OpenAPI document carries it on each operation as `x-capability` (with `x-required-scopes` and `x-min-role`),
the list of all of them as `x-capabilities`, and how each operation authenticates as `x-auth`:
`public`, `session` (a browser session only), `token` (a session or an API token holding the capability) or
`metrics` (the scrape token). A token on a `session` route gets `403 session_required`, whatever its owner's
role and scopes: tokens can never mint tokens, change settings, manage users or approve a plan.

Lachesis's stored token scopes are `read`, `plan` and `apply`. They did not change: no token was re-issued
and no migration ran. The mapping from scopes to capability names is a fixed table in the server,
shown here, in `GET /api/integrations` (`capabilities`, `scopes`), in the OpenAPI document (`x-capabilities`,
`x-scopes`) and for the caller in `GET /api/auth/me` (`capabilities`: what this credential holds right now,
after the owner's role is applied). A capability is held when the credential has **all** the scopes in its
row and its owner's role is at least the role in its row.

| Capability | Scopes | Owner's lowest role | What it opens |
|---|---|---|---|
| `account.read` | `read` | any | `GET /api/auth/me`, `GET /api/openapi.json`, `GET /api/integrations` |
| `inventory.read` | `read` | any | adapters, servers, scopes, leases, reservations, CSV exports, search, nearly-full, network facts and status |
| `drift.read` | `read` | any | `GET /api/drift`, `GET /api/drift/{id}/{peer}`, the network drift reports |
| `servers.probe` | `read` | operator | `POST /api/servers/{id}/probe` |
| `servers.write` | `apply` | administrator | add, change and remove servers; set and remove credentials and the relay token |
| `credentials.read` | `read` | administrator | `GET /api/servers/{id}/credentials` (names and times, never values) |
| `plans.read` | `read` | any | `GET /api/plans`, `GET /api/plans/{id}` |
| `plans.create` | `plan` | operator | `POST /api/plans` |
| `plans.create_from_drift` | `read` and `plan` | operator | `POST /api/drift/{id}/{peer}/plan` |
| `plans.apply` | `apply` | operator | `POST /api/plans/{id}/apply` |
| `approvals.read` | `read` | administrator | `GET /api/approvals` |
| `audit.read` | `read` | administrator | `GET /api/audit` |
| `settings.read` | `read` | administrator | `GET /api/settings`, `GET /api/enrichment/config` |
| `mcp.invoke` | `read` | any | `POST /mcp` (each tool then needs its own capability, see `docs/mcp.md`) |

Scope by scope: `read` opens every capability above that lists only `read` (and `plans.create_from_drift`
together with `plan`), `plan` opens `plans.create`, `apply` opens `plans.apply` and `servers.write`. The
scopes are independent: `apply` does not imply `plan`, and `plan` does not imply `read`.

(Not to be confused with the DHCP server capabilities such as `scopes.read` and `leases.read` that
`/api/servers` reports: those say what a server's adapter can do.)

Routes that need a person and never a token (`x-auth: session`): sign-out, password change, users, tokens,
the OAuth consent steps and client management, plan approval, rejection and abandonment, changing settings
and the network-facts configuration, and the single-sign-on test.

## Errors

Every error from `/api`, `/mcp` and the web app's own paths is JSON in one shape:

```json
{ "error": { "code": "snake_case_code", "message": "Human text", "problems": [ { "field": "name", "message": "Name is required." } ] } }
```

`problems` is present only for a validation failure; each entry names the field (the form field or a dotted
path) and says what is wrong. A client branches on `code`, never on `message`.

| Status | Codes |
|---|---|
| 400 | `bad_request`, `unknown_field` (a field the endpoint does not know), `invalid_field`, `batch_not_supported` |
| 401 | `unauthenticated` (no credential), `invalid_token`, `conflicting_credentials`, `token_owner_must_sign_in`, `bearer_required` (MCP), `unauthorized` (`/metrics`) |
| 403 | `forbidden` (the role is too low), `missing_scope`, `session_required`, `no_capability` (a route with no declared guard is closed), `server_not_allowed`, `bad_origin`, `bad_csrf_token`, `step_up_required`, `self_approval` |
| 404 | `not_found` (no such address, server, scope, plan) |
| 405 | `method_not_allowed` (with `Allow` on `/mcp`) |
| 409 | `conflict`, `name_taken`, `stale`, `server_busy` (with `Retry-After`), `server_applying`, `setting_locked`, `bad_status`, `pending_approval`, `last_admin` |
| 410 | `expired` |
| 413 | `body_too_large`, `request_too_large` |
| 415 | `unsupported_media_type` (bodies are `application/json`) |
| 422 | `invalid_input`, `invalid_credentials`, `invalid_settings`, `invalid_enrichment`, `not_supported` (always with `problems` where a field is at fault) |
| 429 | `rate_limited`, always with `Retry-After` (seconds) |
| 500, 502, 503, 504 | `internal`, `apply_failed` (502; the plan says which steps ran), `unreachable`, `auth_failed`, `unavailable`, `audit_unavailable`, `timeout` |

`GET /api/ready` answers 503 with its own small status body while the database or a migration is not ready.

**Routes that keep another protocol's error body** (exempt from the shared one, because standard clients
parse them): the OAuth endpoints `/oauth/token`, `/oauth/revoke`, `/oauth/register` and `/oauth/authorize`
answer RFC 6749 and RFC 7591 errors (`{"error":"invalid_request","error_description":"..."}`), and
`/auth/oidc/callback` answers with redirects. The OpenAPI document marks them with the `OAuthError` schema.
`GET /metrics` answers the shared body for its refusals and Prometheus text for success.

### `problems` and `fields`

Validation detail is `problems: [{field, message}]` in Lachesis. It was already the shape here before the
shared contract, and the web app reads it, so it was kept; the name `fields` that other apps use would be a
breaking change for the browser UI and for every existing client of the 422 bodies.

## Paging and sizes

List routes take `limit` and `cursor` and answer `next_cursor` when there is more (an opaque string; on the
plans route the timestamp of the last plan). Leases and reservations answer a page of at most 1000 rows
(`limit`, default 100); the CSV exports one page of 1000 rows per request with the same filters; plans at
most 200; the audit log at most 500; MCP lists are bounded and say when they were cut. Request bodies are JSON,
at most 64 KiB.

Reads are served from a cache that every answer describes in `meta` (age, source); `refresh=1` forces a new
read for operators and administrators and is ignored for viewers. Every read is rate limited per caller
(`429` with `Retry-After`).

## The plan, approve, apply path

Nothing writes to a DHCP server outside this path. `POST /api/plans` (or the plan tools over MCP) makes a plan
and changes nothing: it returns the diff, warnings and an expiry. A server may require **approval**: an
administrator other than the plan's author approves in the web app (no token can approve). `POST
/api/plans/{id}/apply` applies a ready plan and verifies it; the outcome is `success` (200), `unverified`
(202: the writes finished but their effect could not be confirmed, which is never reported as success) or
`failed` (502, `apply_failed`).

## Untrusted text

Host names, descriptions, option values, vendor strings and scope and server names in responses were written
by other people and devices. They are data to display, not instructions. In MCP results the fields that hold
such text are listed under `untrusted.paths`.

## Worked examples

Find out who you are and what the token may do:

```
curl -s https://lachesis.example.com/api/auth/me -H "Authorization: Bearer $LACHESIS_TOKEN"
```

List the servers, then the leases of one scope (a CSV export uses the same path with `.csv`):

```
curl -s https://lachesis.example.com/api/servers -H "X-Api-Key: $LACHESIS_TOKEN"
curl -s "https://lachesis.example.com/api/servers/$SERVER/scopes/192.0.2.0%2F24/leases?limit=50" -H "X-Api-Key: $LACHESIS_TOKEN"
```

Plan a reservation, then apply it in a separate call (needs `plan` and `apply` scopes and an operator or
administrator owner):

```
curl -s -X POST https://lachesis.example.com/api/plans -H "Authorization: Bearer $LACHESIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"kind":"reservation.create","server_id":"'$SERVER'","reservation":{"scope_id":"192.0.2.0/24","ip":"192.0.2.50","hw_addr":"02:00:5e:00:53:10","hostname":"printer1"},"reason":"new printer"}'
curl -s -X POST https://lachesis.example.com/api/plans/$PLAN/apply -H "Authorization: Bearer $LACHESIS_TOKEN"
```

## Metrics

`GET /metrics` is off by default (404). When it is on it answers Prometheus text to the **metrics bearer
token** only (a write-only setting or `LACHESIS_METRICS_TOKEN_FILE`). An API token cannot read it and the
metrics token cannot call the API. Failed requests are rate limited per address.

## OAuth for MCP clients

MCP clients that implement the MCP authorization flow use the OAuth endpoints (`/.well-known/...`,
`/oauth/authorize`, `/oauth/token`, `/oauth/revoke`, `/oauth/register`) and the consent page. The access token
they receive is used like an API token (`Authorization: Bearer lco_...`) and carries the same scopes, capped
by the person's role. See `docs/mcp.md`.

## Upgrading from 0.6.0

Existing tokens, data and the web app keep working: stored scopes are unchanged, route paths are unchanged
and the error body was already the shared shape. What a client may notice is listed in the changelog:
`X-Api-Key` is accepted; `GET /api/auth/me` has a new `capabilities` list; the router's own 404, 405 and 500
answers are now JSON (they were plain text); `GET`/`DELETE /mcp` answer 405 and MCP is stateless
(`docs/mcp.md`); tool results carry a top-level `untrusted` key.
