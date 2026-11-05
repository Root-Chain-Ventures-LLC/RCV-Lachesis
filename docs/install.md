# Installing Lachesis

Lachesis ships as one container image (`ghcr.io/root-chain-ventures-llc/lachesis`, public: pulled without
logging in) plus PostgreSQL, wired together by `compose.yaml`. This page lists what has to exist before
`docker compose up` works, and how to start, upgrade and roll back. Everything here is generic: substitute
your own names and addresses.

## 1. Requirements

- Docker with the compose plugin, and access to `ghcr.io` to pull the image.
- A host that can reach your DHCP servers (the egress guard still limits which addresses adapters
  may dial; see `LACHESIS_EGRESS_ALLOW`).
- For anything beyond local trials, a reverse proxy that terminates TLS (section 5).

Get the files of a release: `git clone https://github.com/Root-Chain-Ventures-LLC/RCV-Lachesis.git`, or
unpack `install-bundle.tar.gz` from the latest release (check it against `SHA256SUMS` and `cosign.pub`, see
the README). They are `compose.yaml`, `.env.example` and these docs.

## 2. `.env`

```
cp .env.example .env
```

Edit `.env`. Compose refuses to start without `POSTGRES_PASSWORD` and `LACHESIS_DB_APP_PASSWORD`
(letters and digits, the second at least 16 characters: `openssl rand -hex 16` for each; see "Database
roles" below). Every other value is documented in `.env.example`; the ones you will
usually change:

| Variable | Meaning |
|---|---|
| `LACHESIS_BASE_URL` | The address people type into the browser, for example `https://lachesis.example.com`. State-changing requests must carry exactly this origin. An `https://` address turns on Secure cookies and HSTS and requires `TRUST_PROXY`. |
| `LACHESIS_PUBLISH_ADDR` | Host address the container port 8080 is published on (section 4). |
| `TRUST_PROXY` | Your reverse proxy (section 5). |
| `LACHESIS_SETUP_TOKEN` | At least 16 characters; the only way to create the first administrator without single sign-on (section 7). |
| `LACHESIS_OIDC_*` | Optional single sign-on (section 7). |
| `LACHESIS_LOCAL_ADMINS_ONLY` | Restrict password sign-in to administrators. Defaults to `true` when single sign-on is configured, `false` otherwise. |
| `LACHESIS_MCP_ENABLED`, `LACHESIS_MCP_RATE_TOKEN` | MCP for AI agents, off by default. `true` serves it at `/mcp` for API tokens with the `read` scope (cookies are not accepted there, and a request that carries an `Origin` must carry this Lachesis's own, so web pages cannot reach it); the rate is messages per minute per token. A token with the `plan` or `apply` scope whose owner is an operator or administrator can also plan and apply changes through the MCP write tools (`plan_reservation`, `plan_pool_change`, `plan_option_change`, `get_plan`, `apply_plan`); they go through the same plan engine, approval rules and limits as the REST API, a token can never approve a plan, and (as over REST) any apply-scoped operator can apply any ready plan on servers in the token's allow-list. A request is capped at 64 KiB and JSON-RPC batches are refused. The 403 (not allowed) and 429 (rate limited) answers of MCP travel inside JSON-RPC errors (`-32030`, `-32029` and, for a busy server, `-32031`, with the HTTP status in the error data) over HTTP 200, so a proxy or monitor sees them as successful requests; only authentication failures (401, 403 `missing_scope`, `bad_origin`) and oversized or batched bodies use HTTP status codes. A local agent that speaks stdio runs `lachesis mcp --stdio --token-file PATH --url https://lachesis.example.com` (the token file must be mode 0600). |
| `LACHESIS_SECRET_REF_DIRS`, `LACHESIS_SECRET_REF_ENV_PREFIX` | Which `file://` and `env://` secret references may resolve (section 8). Unset, file references are off and env references need the `LACHESIS_REF_` prefix. |

`.env` is never baked into the image. The `compose.yaml` of a release runs the image of that release;
`LACHESIS_VERSION` in `.env` selects another release (or `latest`), and `LACHESIS_IMAGE` replaces the image
reference altogether.

## 3. `secrets/master.key`

The master key (KEK) wraps the data keys that protect stored credentials. It never lives in the
database. It is a file containing 32 random bytes, base64-encoded:

```
mkdir -p secrets
( umask 077; openssl rand -base64 32 > secrets/master.key )
sudo chown 65532 secrets/master.key
chmod 600 secrets/master.key
```

- Compose, without swarm, **bind-mounts** a file secret exactly as it is on the host. The `uid`,
  `gid` and `mode` fields of the long secret syntax are ignored, so the file itself must be owned
  by uid 65532 (the container user) and not readable by group or others.
- `docker compose up` fails with a missing-file error if the file does not exist: it is a
  prerequisite, not something compose creates. A different path can be set with
  `LACHESIS_MASTER_KEY_SECRET_FILE`.
- Back the key up separately from the database backups. Without it, stored credentials cannot be
  decrypted; with both together, backups are as sensitive as the credentials.
- `secrets/` is git-ignored.
- **Startup fails closed.** `lachesis serve` refuses to start, with the reason on stderr, when no key
  is configured, the file is missing or unreadable, is not a regular file, is readable by group or
  others, or does not hold exactly 32 base64-encoded bytes. It also refuses to start when stored
  credentials were wrapped by a different key than the one loaded (the wrong key file, or a rotation
  that was not finished). Nothing is migrated or listened on before the key is accepted.
- `LACHESIS_MASTER_KEY` (the key itself in the environment) is accepted for setups that cannot mount
  a file, and logs a warning at start. Environment variables leak through process listings and
  crash dumps; prefer the file.
- **Back up the key separately** from the database backups, in a place that is itself access
  controlled. Anyone holding both can read every stored credential. Lose the key and the stored
  credentials are unrecoverable: delete them (`DELETE FROM secrets`, with Lachesis stopped) and enter
  them again.

### Rotating the master key

Rotation re-wraps each credential's data key under a new master key. The encrypted credentials
themselves are not touched. Do it on a schedule you choose, and at once if the key may have been
exposed (a backup of the key leaked; someone with access to it left).

```
# 1. a new key, same rules as the old one
( umask 077; openssl rand -base64 32 > secrets/master.key.new )
sudo chown 65532 secrets/master.key.new && chmod 600 secrets/master.key.new

# 2. stop the app, then rotate. Both keys are named explicitly:
docker compose stop app
docker compose run --rm --no-deps \
  -v "$PWD/secrets/master.key.new:/run/secrets/new_master_key:ro" \
  app keys rotate --old-key-file /run/secrets/lachesis_master_key --new-key-file /run/secrets/new_master_key

# 3. swap the key file, start, and confirm
mv secrets/master.key secrets/master.key.old && mv secrets/master.key.new secrets/master.key
docker compose up -d app
docker compose exec app /lachesis keys check

# 4. only now destroy master.key.old and every backup of it
```

`keys rotate` runs in one transaction: it decrypts every credential under the old key, re-wraps its
data key under the new one, decrypts it again under the new key and compares, and commits only if
all of them match; any failure leaves everything as it was. Re-running it after an interruption is
safe, and `--old-key-file` lets you repair a rotation whichever key file is currently configured.

A running server holds the master key in memory and would go on writing under the old one, so the
server keeps a shared database lock for as long as it runs and `keys rotate` takes it exclusively:
**it refuses to run while any server is up**, and a server refuses to start while a rotation is
running. `lachesis keys check` loads the configured key and verifies that every stored credential
decrypts; it prints counts and opaque ids, never values. Keep **both** keys until `keys check`
passes with the new one.

If the rotation's database session ends while it runs (the database restarted, the connection dropped),
its lock is gone. `keys rotate` checks the lock before each credential and before it commits; when it is
lost, it rolls back, changes nothing, prints that the lock was lost and exits non-zero. Check that the
database is healthy and run the same `keys rotate` command again. Until it succeeds, every credential is
still wrapped by the old key, so keep the old key file in place: a server started with it works, and a
server started with the new key refuses to start. The rotation keeps the credentials table locked against
reads as well as writes until it ends, so a server that starts after the rotation lost its lock waits
at startup for the rotation to finish: it then starts normally if the rotation rolled back, and refuses to
start (wrong key) if it committed. If the rotation's own database connection fails, it prints "commit
result unknown": run `keys check` once with the old key and once with the new one, then run `keys rotate`
again. The unlock at the end of `keys rotate` and of `serve` gives up after 5 seconds if the database does
not answer.

## 4. Where the port is published

`compose.yaml` publishes `${LACHESIS_PUBLISH_ADDR:-127.0.0.1}:8080:8080`: **loopback only** by
default, so a fresh install is not reachable from the network. Choose one:

- Keep loopback and run a reverse proxy on the same host (recommended).
- Set `LACHESIS_PUBLISH_ADDR` to one private address of the host so that a proxy or a health
  check on another machine can reach it. Do not publish plain HTTP on a public address.

Anything that probes the app from outside the host (a load balancer, an automated deploy that
health-checks `http://<host>:8080/api/health`) needs `LACHESIS_PUBLISH_ADDR` set to an address it
can reach.

## 5. Reverse proxy and `TRUST_PROXY`

Lachesis speaks plain HTTP. Terminate TLS in a reverse proxy that forwards to the published port
and passes the original `Host` and `X-Forwarded-For`. Then:

- set `LACHESIS_BASE_URL` to the public `https://` address, and
- set `TRUST_PROXY` to the proxy's address or a comma-separated list of addresses or CIDR prefixes
  (for example `192.0.2.10` or `198.51.100.0/24`; `1` means one proxy on a private address).

Lachesis then takes each visitor's address from `X-Forwarded-For` for rate limits and the audit
log. **The process refuses to start** if `LACHESIS_BASE_URL` is `https://` and `TRUST_PROXY` is
empty. Without a proxy, leave `TRUST_PROXY` empty: nothing is trusted.

Do not let the proxy add its own `Authorization` header (for example HTTP basic auth in front of
Lachesis): a request that carries one is treated as an API-token request, so every session route
would answer `401 invalid_token`. Authenticate at the proxy by other means (client certificates,
an IP allow-list) or not at all.

## 6. Start, check, upgrade

```
docker compose pull
docker compose up -d
curl -fsS http://127.0.0.1:8080/api/health      # {"status":"ok",...}  (process is up)
curl -fsS http://127.0.0.1:8080/api/ready       # {"status":"ready",...} (database + migrations)
```

Migrations are applied at start. A database newer than the binary (a downgrade) is refused, and
a failed migration stops the process immediately with the error; neither is retried.

To upgrade, take the backup below, get the files of the new release (`git pull` in your clone, or unpack the
new `install-bundle.tar.gz` over it and keep `.env` and `secrets/`; the `compose.yaml` names the new version),
then:

```
docker compose pull && docker compose up -d
curl -fsS http://127.0.0.1:8080/api/ready
```

`migrate` runs the new version's migrations first; `app` is recreated when it has finished. To move to a
version without changing `compose.yaml`, set `LACHESIS_VERSION=<version>` in `.env` before `docker compose pull`.

### Before you upgrade

1. **Back up the database.** Take the dump through the `db` service, so the client is the same PostgreSQL
   version as the server, and keep the file off the host, next to the separate backup of `secrets/master.key`
   (stored credentials cannot be decrypted without it) and your `.env`:

   ```
   docker compose exec -T db pg_dump -U lachesis -d lachesis --format=custom > lachesis-pre-upgrade.dump
   ```

2. **Know what rollback means: restoring that backup.** There are no down migrations, and the binary is
   in lock-step with the schema: it refuses to start on a database that is newer than itself
   (`ErrSchemaAhead`: "the database schema is newer than this version of Lachesis; run the newer release, or
   restore a matching database"), so the previous release cannot simply be started again on an upgraded
   database. To go back, restore the backup and start the previous release:

   ```
   docker compose stop app
   docker compose exec -T db pg_restore -U lachesis -d lachesis --clean --if-exists --single-transaction --no-owner < lachesis-pre-upgrade.dump
   # set LACHESIS_VERSION=<the version you upgraded from> in .env (or restore that release's files), then:
   docker compose pull && docker compose up -d   # migrate re-applies the grants; app starts at the old schema
   ```

   Everything written after the backup is lost with it: audit rows, plans, tokens, sessions, server changes.
   `.env` and `secrets/master.key` are not touched by an upgrade or a restore; keep them as they were.

3. **Migrations expand first and contract a release later.** For the few seconds between `migrate`
   finishing and `app` being recreated, the old binary runs on the new schema. So a migration only
   *expands* in release N: it creates tables, columns (nullable or with a default), indexes and
   functions, and never drops, renames, retypes or tightens anything the previous release still uses. The
   *contract* step (drop the column, add `NOT NULL`, ...) ships in release N+1 or later, once nothing
   reads the old shape.
   The guarantee covers one release at a time: if you skip releases,
   the old binary can meet a contracted schema in that window, which is one more reason to take the backup
   and to upgrade one release at a time when you can.

## 7. Sign-in, roles and API tokens

### 7.1 First administrator

Without single sign-on, set `LACHESIS_SETUP_TOKEN` (`openssl rand -hex 16`), restart, open
`LACHESIS_BASE_URL` and fill in the setup form: the token, a username and a password of at least
12 characters. Setup works only while **no account of any kind exists** (disabled or not, local or
single sign-on, whatever its role) and then never again, even if every administrator is later
demoted or disabled; with no token configured it is unavailable (nothing is generated or logged).
If you configure single sign-on, the first person who signs in creates an account and closes setup
for good, so the token is then useless. Keep the token out of your shell history anyway, and rotate
it by editing `.env` if it was exposed. If you ever lose every administrator, recover from the
database (set `role = 'admin'` on an account you control), not by reopening setup.

### 7.2 Single sign-on (OIDC)

Any OpenID Connect provider works. Register Lachesis as a **confidential client** that uses the
**authorization code flow with PKCE** and give it:

| Setting | Value |
|---|---|
| Redirect URI | `LACHESIS_BASE_URL` + `/auth/oidc/callback`, for example `https://lachesis.example.com/auth/oidc/callback` |
| Scopes | `openid profile email groups` |
| ID token signature | RS256, RS384, RS512, ES256/384/512 or PS256/384/512 (never HMAC) |
| Groups | the user's groups in the **ID token**, as an array of strings (or one string), under the claim named by `LACHESIS_OIDC_GROUPS_CLAIM` (default `groups`) |

Then set, in `.env`:

```
LACHESIS_OIDC_ISSUER=https://idp.example.com/application/o/lachesis/   # must be https
LACHESIS_OIDC_CLIENT_ID=lachesis
LACHESIS_OIDC_CLIENT_SECRET_FILE=/run/secrets/lachesis_oidc_client_secret
LACHESIS_OIDC_ROLE_MAP=admins=admin,netops=operator,readers=viewer
```

- The client secret is read from a **file** that only its owner can read (`chmod 600`; Lachesis
  refuses a group- or world-readable file and an empty one). Mount it into the container, for
  example with a `compose.override.yaml` bind mount of the file at that path for uid 65532.
- `LACHESIS_OIDC_ROLE_MAP` is `group=role` pairs (group names are case-sensitive). A person in
  several mapped groups gets the highest role. **A person in no mapped group is refused** with a
  "No access" page; there is no default role. The role is recomputed at every sign-in, so removing
  someone from a group takes effect at their next sign-in (and ends their current sessions the next
  time they try). Because of that, the role of a single-sign-on account cannot be edited in
  Lachesis (change the group or the map); administrators can still disable the account.
- Accounts are identified by the provider's issuer and `sub` claim, not by email.
- The identity provider being down does not stop Lachesis starting; the local administrator
  accounts below keep working.

### 7.3 Local accounts (break-glass)

Local accounts sign in with a username and password hashed with **argon2id** (RFC 9106, `t=3`,
`m=64 MiB`, `p=2`, 16-byte salt, 32-byte key, about 100 to 250 ms and 64 MiB per sign-in; at most
four hashes run at once). The parameters are stored in each hash, so they can be raised later and
existing hashes are upgraded at the next successful sign-in.

- With single sign-on configured, password sign-in is limited to local **administrators** by
  default (`LACHESIS_LOCAL_ADMINS_ONLY=true`), so ordinary users cannot bypass the provider.
  Administrators create further local accounts under Users (`POST /api/users`).
- Sign-in attempts are limited per client address and username: five attempts in 15 minutes, then
  that address and username are locked out for 15 minutes (429), doubling on repeated lockouts up to
  four hours. A second limit applies per client address alone (a burst of 10, then one attempt per
  6 seconds). Both are keyed on the address, so a stranger cannot lock the real user out: the real
  user signing in from another address is never refused. Across all addresses an account that is
  being guessed is only **slowed** (each attempt waits up to 2 seconds once more than 100 attempts
  have been made recently); it is never refused on that account, because that would let a
  distributed attacker lock the break-glass account out. argon2id and the 12-character password
  minimum bound what such an attacker can do. The counters live in memory (capped in size, stale
  entries swept every minute) and reset when Lachesis restarts. Behind a proxy the client address
  comes from `X-Forwarded-For`, which is why `TRUST_PROXY` must name your proxy.
- Local accounts can create API tokens only while single sign-on is **not** configured
  (`LACHESIS_OIDC_ISSUER` empty). With single sign-on configured they are break-glass
  accounts: they get an administrator back in, not standing machine credentials. The decision
  follows the configuration, never whether the provider answers, and it is applied when a token is
  **used** too: a token minted by a local account stops working as soon as single sign-on is
  configured (and works again if it is removed).

### 7.4 Roles

| Role | May |
|---|---|
| `viewer` | read |
| `operator` | read, plan, apply |
| `admin` | everything operators can, plus servers, credentials, users, tokens (everyone's), settings and approvals |

Anything a role may not do answers **403**, never an empty list; no session answers **401**.

### 7.4a Approvals queue and the approval webhook

Servers that require approval put their plans in the **Approvals** queue (administrators see a count in the
navigation). An administrator other than the author approves; any administrator may reject with a reason, and the
author may reject (withdraw) their own plan. To be told when a plan starts waiting, or is rejected, set:

```
LACHESIS_APPROVAL_WEBHOOK_URL=https://hooks.example.com/lachesis
LACHESIS_APPROVAL_WEBHOOK_SECRET_FILE=/run/secrets/lachesis_approval_webhook_secret   # chmod 600, mounted like the OIDC secret
```

- The message is JSON: `event` (`plan.pending_approval` or `plan.rejected`), `plan_id`, `kind`, `servers` (names),
  `author` (display name), `expires_at` and `link`. It has no reason text and nothing secret.
- Verify it. `X-Lachesis-Signature` is `sha256=` plus the hex HMAC-SHA256, keyed with the secret, of this string
  (the values of three headers, then the raw body, joined by dots):

  ```
  <X-Lachesis-Timestamp>.<X-Lachesis-Delivery>.<X-Lachesis-Event>.<raw body>
  ```

  Recompute it, compare in constant time, reject a timestamp older than a few minutes and remember delivery ids for
  that long, so a replayed delivery is dropped. The delivery and event headers are signed, so a receiver can trust
  them.
- The URL may carry a token: it is never logged. Deliveries go through the egress guard. A receiver on a private
  network (for example `10.x`, `192.168.x`, `172.16-31.x`, a unique-local IPv6 address) works as it is: you chose the
  URL, so private ranges are allowed. Loopback, link-local (cloud metadata included) and the unspecified address are
  refused unless their network is in `LACHESIS_EGRESS_ALLOW`. Failed deliveries are retried three times and then
  audited (`plan.webhook`), repeated failures as one summary row per five minutes.

### 7.5 Sessions

A session is a random 256-bit value in the cookie `__Host-lachesis_session` (`Secure`, `HttpOnly`,
`SameSite=Lax`, `Path=/`); only its SHA-256 is stored. It ends after 8 hours without a request or
24 hours after sign-in, whichever comes first, is replaced at every sign-in, and is deleted at
sign-out or when the account is disabled. Browsers only accept the `__Host-` prefix on `Secure`
cookies, so with a plain `http://` base URL (local trials) the cookie is named `lachesis_session`
without `Secure`; use `https://` for anything else.

State-changing requests made with a session must come from `LACHESIS_BASE_URL` (the `Origin`
header, or `Referer` when there is none) **and** carry the header `X-CSRF-Token` with the value
`GET /api/auth/me` returns. The check uses `LACHESIS_BASE_URL`, never the `Host` header, so the
proxy does not have to preserve `Host`.

### 7.6 API tokens

Create tokens under your account (single-sign-on users; local accounts only when single sign-on is
not configured, see 7.3): a required name, an expiry of 1 to
365 days, scopes `read`, `plan` and/or `apply` (limited by your role: viewers can only hold `read`),
and optionally a list of server ids the token is limited to. The token `lch_<id>_<secret>` is shown
**once**; Lachesis stores only a SHA-256 of the secret. Send it as `Authorization: Bearer <token>`;
requests made with a token are not subject to CSRF checks and ignore cookies. Name your client with
the `X-Client-Name` header (the `User-Agent` is used otherwise) to see it in the token's
"last used" line. A token stops working when it is revoked, expires, its owner is disabled, and
loses scopes the owner's role no longer allows.

## 9. Automated deploys

If a pipeline copies the compose files to the host before running compose, make sure it does **not**
overwrite or delete `.env` and `secrets/`: provision them on the host once, in advance, and
exclude them from the copy. A first deploy on a host without them fails at `docker compose up`.

## 8. Database roles

The server must not be able to rewrite its own audit log, so it does not own the database objects.
Two roles are used:

- the **owner** (`POSTGRES_USER`, `lachesis`) applies the migrations. Only the one-shot `migrate`
  container in `compose.yaml` holds its password;
- the **application role** (`lachesis_app`) is what the server connects as (`DATABASE_URL`). It can
  read and write every table and use the sequences, except that `audit_log` is `SELECT, INSERT` only
  and `schema_migrations` is `SELECT` only. It owns nothing, so it cannot update, delete or truncate
  audit records, drop or disable their trigger, replace the trigger function, or change the schema.

`lachesis migrate` applies pending migrations and then, when `LACHESIS_DB_APP_ROLE` is set, creates the
application role if it does not exist (from `LACHESIS_DB_APP_PASSWORD`, at least 16 characters; an
existing role keeps its password) and (re)applies its grants. The grants are idempotent and cover tables
that later migrations add, so it runs on every start. `docker compose up` does all of this: the `app`
service starts after `migrate` has completed. To upgrade an installation that ran as a single role, set
`LACHESIS_DB_APP_PASSWORD` in `.env` and run `docker compose pull && docker compose up -d`; no manual SQL is needed.

**Recommended: run `lachesis migrate` as its own step** (a one-shot container, as `compose.yaml` does, or
an init step of your service manager) with the owner's connection string in `DATABASE_URL`, and start the
server with only the application role's `DATABASE_URL`. The owner's credential then never exists in the
server's environment.

`lachesis serve` also accepts `MIGRATE_DATABASE_URL` (the owner) alongside `DATABASE_URL` (the application
role): it migrates with the first, connects with the second, and, with `LACHESIS_DB_APP_ROLE` set, grants
the role too. That works, but it leaves `MIGRATE_DATABASE_URL` in the server's environment, where a
compromised server could read it and rewrite the audit log, so `serve` logs a **warning** at every start
when it is set. Use it for convenience on a trial, not in production. With only `DATABASE_URL` set, one
role does everything and the audit log is only as tamper-resistant as that role: acceptable for trials, not
for production.

Superusers and table owners can always disable triggers. Keep the owner's password out of the server's
reach, and ship audit records off the host (a SIEM) if you need protection against the database
administrator.

### Audit retention

The application role can append to `audit_log` but never delete from it, so pruning is the owner's job.
`LACHESIS_AUDIT_RETENTION_DAYS` (default **365**; `0` keeps every row forever; otherwise 7 or more) says how
long to keep rows. What happens, and what to do:

1. **Automatically at every deploy.** The one-shot `migrate` container (the owner) prunes rows older than the
   retention after applying migrations and the grants. A prune failure is logged as a warning and does not stop
   the server from starting. Set `LACHESIS_AUDIT_RETENTION_DAYS` in `.env` (the compose file passes it to
   `migrate` and `audit-prune`).
2. **On a schedule.** A server that is rarely redeployed needs the same run on a timer. Add to cron or a systemd
   timer (daily is plenty):

       docker compose --profile maintenance run --rm audit-prune

   Add `--days 400` to override the setting once, or `--dry-run` to count what would go. Outside compose:
   `lachesis audit prune`, with the owner's connection string in `MIGRATE_DATABASE_URL` (or in `DATABASE_URL`);
   it is refused for the application role.
3. **What it leaves behind.** Every batch (5000 rows) writes an `audit.prune` row (actor `system / retention`)
   with the number of rows pruned, the cutoff, the first and last row id and the oldest and newest time removed,
   in the same transaction as the delete. These rows are never pruned. The append-only triggers stay enabled; the
   prune is the `audit_log_prune` database function, executable only by the owner. It refuses a cutoff
   younger than 7 days.
4. **Keeping older rows.** Pruning deletes. If you need the history, copy it out first as the owner, for example
   `pg_dump -t audit_log` or `COPY (SELECT * FROM audit_log WHERE at < now() - interval '365 days') TO ...`, or ship
   the rows to a SIEM, or set retention to `0` and size the disk for it.
5. **Check it.** `SELECT at, "after" FROM audit_log WHERE action = 'audit.prune' ORDER BY id DESC LIMIT 5;`
   shows the last runs. Large first runs can take a while. Each batch takes a table lock on `audit_log` for a moment. If a
   long transaction that wrote audit rows is open, the prune gives up after one second and tries again (up to six
   times), so an audit insert waits about a second at most, not for the whole prune; a run that still cannot get
   the lock fails with an error and is simply run again later.

Rejected API tokens appear as one `token.rejected` row per token (or `lch` prefix for unknown ones) and reason per
5-minute window, with `count`, `first_at` and `last_at`, about six minutes after the window ends. The per-address
limit on requests with an `Authorization` header is `LACHESIS_BEARER_RATE_IP` (default 600 a minute; an IPv6 client
counts by its /64). **Behind a reverse proxy, set `TRUST_PROXY`**: without it every client arrives from the proxy's
address, shares one budget, and a single stream of junk requests can answer everyone 429. The first rejection of a
token (or of the unknown-token prefix) per reason and window is also written to the server log as a `Warn` line.

If the database session that holds the master-key lock ends (database restart, killed connection), the server
logs `the master-key lock was lost`, drains and exits with status 1, and `restart: unless-stopped` starts it again.

### Secret references

A credential can be stored as a reference instead of a value: `env://NAME` or `file:///path`, resolved
each time an adapter session opens and never stored. References are **default-deny**, because otherwise
an administrator could point one at `/etc/passwd` or `$PATH` and have it sent to a server they control:

- `file://` resolves only under the directories in `LACHESIS_SECRET_REF_DIRS` (colon separated, absolute).
  Unset, file references are disabled. Use a **dedicated directory** that holds only the files meant to
  be referenced (for example `/run/secrets/lachesis-refs`), not a general one such as `/run/secrets`,
  `/etc` or a home directory: every readable file under a listed directory can be turned into a credential. The path is checked when the reference is saved and again, on the
  real path of the opened file, each time it is resolved. The master key file, the OIDC client secret file
  and anything under `/proc`, `/sys` and `/dev` can never be referenced.
- `env://` resolves only for names starting with `LACHESIS_SECRET_REF_ENV_PREFIX` (default
  `LACHESIS_REF_`). Lachesis's own settings (`DATABASE_URL`, `LACHESIS_MASTER_KEY`, ...) can never be
  referenced.

