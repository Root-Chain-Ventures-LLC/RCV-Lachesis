# Kea adapter

ISC Kea DHCPv4, versions 2.6 and 3.x. Vendor id `kea`.
DHCPv6 is not covered yet (the model has the family; the adapter addresses the `dhcp4` service only).

## Transport and authentication

Lachesis speaks the Kea control API: one `POST` of `{"command", "service": ["dhcp4"], "arguments"}`
per call, to the endpoint URL given on the server form. The endpoint is either the DHCPv4 daemon's own
HTTP control socket (`"control-sockets": [{"socket-type": "http", ...}]`, Kea 2.4 and later) or a
Control Agent; the same request works against both, so there is nothing to choose. Use `https`:
credentials go out on every request, and Probe warns when basic-auth credentials would travel over plain `http://`.

Credentials are secrets and reach the adapter only through `adapter.Secrets`:

| Secret | Use |
|---|---|
| `username` + `password` | HTTP basic auth (the control socket's `authentication` block) |
| `client_cert` + `client_key` (PEM) | TLS client certificate, for a socket or reverse proxy that requires one |

Either set is enough; both together are sent together. The server certificate is verified by default
(system roots, a custom CA, or a pinned leaf: the per-server TLS settings of the core). The client
certificate is added to a clone of the SDK's HTTP client (`adapter.WithClientCertificate`), so every
connection still goes through the guarded dialer. A rejected credential or certificate is
`ErrAuthFailed` (HTTP 401/403); a handshake the server refuses, a refused connection or a timeout is
`ErrUnreachable`.

Connection settings:

| Field | Default | Meaning |
|---|---|---|
| `write_mode` | `auto` | `auto` detects (below); `host_cmds` or `config` force the reservation write mode |
| `ha_settle_seconds` | 30 | how long a config-mode write waits for an HA pair that is still settling after the last configuration change (below); 0 does not wait; 0 to 60. Used before the write and again after it |
| `page_size` | 1000 | leases or reservations requested per page |
| `max_items` | 5000000 | the most leases (or reservations of one subnet) one read accepts; a server that sends more is treated as faulty and the read fails. Paging also fails if a cursor does not move strictly forward, and one answer may not exceed 16 MiB |

### Least privilege

The Kea control API has no read-only role: whoever can authenticate can run `config-set`, `shutdown`
and `lease4-wipe`. Give Lachesis its own basic-auth client (or its own client certificate), bind the
control socket to a management address, and keep it off networks clients can reach. Lachesis only
sends the commands listed below.

## Probe

`Probe` runs `version-get`, `config-get` and, when an HA relationship is configured, `status-get`
(all read-only), and reports:

- **Version** (`ServerVersion`, for example `3.2.1`).
- **Loaded hooks**, from the running configuration's `hooks-libraries` (`Details["hooks"]`, such as
  `ha,host_cmds,lease_cmds,subnet_cmds`).
- **HA mode and peers**: `Details["ha_mode"]` and `ha_state`, the capability `failover`, the failover
  traits and warnings (state not serving, partner communication interrupted). `Peers` lists the
  members with role, mode and state.
- **Write mode**: `Details["write_mode"]` and the `Mode` of each write capability.

### Write mode detection

Reservations (`reservations.write`):

| Mode | When | How a reservation is written |
|---|---|---|
| `host_cmds` | the `host_cmds` hook is loaded **and** a hosts database of type `mysql` or `postgresql` is configured (`hosts-database` or `hosts-databases`) and is not `"readonly": true` **and**, from Kea 3.0 (where database backends are hooks) or when the version is unknown, the backend hook `libdhcp_mysql` / `libdhcp_pgsql` is loaded | `reservation-add`, `reservation-update`, `reservation-del` |
| `config` | anything else, unless a configuration backend owns the configuration | `config-get`, edit, `config-test`, `config-set`, `config-write` |
| unavailable | a configuration backend (`config-control` with `config-databases`) is in use and there is no `host_cmds` hosts database | `reservations.write` is reported unavailable with the reason |

`write_mode` = `host_cmds` or `config` overrides the detection. Forcing `host_cmds` without the hook
makes the capability unavailable; without a detectable hosts database it is kept and Probe warns that
`reservation-add` will fail.

Pools and options (`pools.write`, `options.write`) are **always config mode**. `subnet4-add` and
`subnet4-update` (subnet_cmds) change only the running configuration: they are not written to the
configuration file and not replicated to an HA partner, so they cannot make a durable write.

Reads:

- `scopes.read`: always, from `config-get` (the running configuration includes subnets added at run
  time).
- `reservations.read`: with the `host_cmds` hook, `reservation-get-page` (it covers every host source,
  the hosts database and the configuration file); without it, the configuration file's reservations.
  The `Mode` is `host_cmds` or `config` accordingly.
- `leases.read`: needs the `lease_cmds` hook, otherwise unavailable with that reason.

### What Kea actually does (verified on Kea 3.2.1, 2026-11-02)

Against a pair **without** a hosts database, with reservations in the configuration file:
`reservation-get-page` returns them; `reservation-add` answers result 1 "Host database not available,
cannot add host."; `reservation-update` and `reservation-del` fail the same way. So `host_cmds` cannot
write config-file reservations, which is why config mode exists. Against a pair **with** a shared
PostgreSQL hosts database and no config-file reservations: `reservation-add`, `-update`, `-del` work
on either peer and the result is visible on both.

## Capabilities

| Capability | Status | Notes |
|---|---|---|
| scopes.read | yes | subnets (also inside shared networks), pools, scope- and pool-level options, lease time; scope id is the Kea subnet id |
| leases.read | yes, needs `lease_cmds` | paged with `lease4-get-page` (resumes after the last address of a page); one address, hardware address or client id use the direct lookups; state 0 past its end is reported expired |
| reservations.read | yes | hosts with an address and a hardware address or client id; see quirks |
| reservations.write | `host_cmds` or `config` | create, update, delete; idempotent (below) |
| pools.write | config mode | create, update (range), delete; pool-level options are kept |
| options.write | config mode | scope and pool level, by code and option space |
| leases.write | no | `lease4-del` on one peer is not replicated, so a release can be neither durable nor verified on a pair |
| failover | when an HA relationship is configured | `Peers`; `Replicate` is not supported (Kea has no command to push configuration) |

`Preflight` applies a plan's config-mode changes to a copy of the running configuration and runs
`config-test` on it, so an invalid plan fails at plan time.

### Config mode, step by step

1. `config-get` (the snapshot and its hash). The plan engine opens a session from stored capabilities
   without probing, so the snapshot is checked again here: a configuration backend that has appeared
   makes the write `ErrNotSupported`. A server that returns **no hash** is refused (`ErrConflict`): the
   lost-update guard below cannot work without it.
2. Edit a copy of the `Dhcp4` section; every member Lachesis does not own goes back unchanged.
3. `config-test` with the exact configuration; a refusal is `ErrInvalidInput` carrying Kea's reason.
   Nothing is changed.
4. With HA configured, `status-get`: `config-set` restarts this server's HA state machine, so the write only goes
   ahead when this server and its partner are in a serving state (`hot-standby`, `load-balancing`,
   `passive-backup`, `backup`), in touch and not interrupted. A pair that is **still settling** (this server or its
   partner `waiting`, `syncing` or `ready`, the states a restarted state machine passes through) is waited for:
   `status-get` is repeated every second for up to `ha_settle_seconds` (default 30, see
   [Waiting for the HA pair to settle](#waiting-for-the-ha-pair-to-settle)). Nothing is sent to `config-set`
   until the pair serves. Anything waiting cannot cure is refused at once with `ErrConflict`; a pair that is still
   settling at the end of the wait is `ErrBusy` ("the HA pair is still settling ... nothing was written"). There is no override.
5. `config-hash-get`; if the hash differs from the snapshot the configuration changed in between and
   the write fails with `ErrConflict` (Kea has no compare-and-set; this narrows the window).
6. `config-set`, then `config-write` (to the file Kea was started with).
7. `config-get` again. From step 6 on the change is live, so nothing after it is an error: the result is
   `Verified` only if the re-read shows the change **and** `config-write` succeeded; otherwise it is
   returned unconfirmed with a note ("config-write failed ... lost when Kea restarts", "could not be
   read back", "the re-read does not show the change").
8. With HA configured, the **post-write settle**: `status-get` on this same server, repeated every second
   until the pair is serving again (bounded by `ha_settle_seconds`), and only then the result is returned. The
   plan engine writes the peers one after the other; the partner's own `status-get` can still show this server
   as healthy for a few seconds after its restart began, so the check before a write cannot stop the second peer
   being written while the first restarts and both ending up `waiting` together. Waiting here makes the next
   step start on a serving pair. The change is live, so this never fails the call: if the server has not
   returned to a serving state within the bound (or the pair is not serving for a reason waiting cannot cure, or
   `status-get` fails, or the context ends) the result carries the warning "applied; the server had not
   returned to a serving HA state within N s - check Probe before the next change" and is never `ErrBusy`.

Every operation is idempotent: a create that finds an equivalent entry, or a delete that finds the entry
already gone, sends nothing and is `Verified` with a warning. That is what makes writing both HA peers
safe when they share a backend.

### host_cmds mode

`config-get` (the hook and a writable hosts database are checked again; if the database went away or
turned read-only the write is `ErrNotSupported`), then `reservation-get-by-address` (existing entry: equivalent is a no-op, different is `ErrConflict`),
then `reservation-add` / `reservation-update` / `reservation-del`, then `reservation-get-by-address`
to verify; a read-back that fails after a successful write is returned unconfirmed. `reservation-update` replaces the whole host, so the adapter sends the stored host with only
the fields it owns changed. A change of identifier (hardware address to client id) is a delete plus an
add; if the add fails the previous reservation is restored.

### Waiting for the HA pair to settle

Every `config-set` restarts that server's HA state machine. Measured on the Kea 3.2.1 simulators (2026-11-02):
the restarted server reports `waiting` (with no partner state and not in touch) for about 10 seconds and then
returns to `hot-standby`; its partner shows the restarted server as `waiting` for about 10 seconds too, from
about three seconds after the write. A second change applied inside that window used to be refused, so a
create followed by a delete failed for no reason. Now:

| Pair state when a config-mode write starts | Result |
|---|---|
| healthy | write at once |
| this server or its partner `waiting`, `syncing` or `ready` | poll `status-get` every second; write when both serve again |
| still settling after `ha_settle_seconds` | **`ErrBusy`** (`adapter.Busy`, retry hint 10 s); nothing was written |
| partner `unavailable` or `partner-down`, this server `partner-down`, `terminated` or any other state, partner not in touch, communication interrupted | **`ErrConflict` at once**, no wait |
| `status-get` fails, or reports no relationship | `ErrConflict` at once (or the connection error) |

A local server that is `waiting` is still checked for what its partner reports: a partner `unavailable`,
`partner-down` or `terminated`, or an interrupted communication, is `ErrConflict` at once. Only "not in touch" and
an empty partner state are ignored while the local server settles, because that is what a restart looks like.

`ha_settle_seconds` 0 disables both waits: a settling pair is then `ErrBusy` at once before a write, and nothing
is waited for after one. The wait stops with the caller's context; if the plan engine's time limit ends during the
wait before a write the result is still `ErrBusy`, because `config-set` is only sent after the wait.

**Sequence of a two-peer plan** (A, then B), timing from the simulators: A is checked (serving) and written;
A reports `waiting` at once and A's call keeps polling until it is serving again (about 10 s); B is then checked
(B's view of A can still read `waiting` for about 3 s more, which the pre-write wait absorbs) and written; B's
call polls until B serves.

**How it meets the plan engine's apply timeout.** The engine runs every step of one apply inside one timeout
(`plan.DefaultApplyTimeout`, 5 minutes, from the claim to the verified end), and that deadline, not the settle
settings, is what protects correctness: when it ends, `config-set` has either not been sent (the step is refused,
`ErrBusy` on the first step) or the step has already returned. The settle bound counts the one-second pauses; the
time the `status-get` calls themselves take is on top of it, and the deadline caps that too.
Each config-mode step can wait up to `ha_settle_seconds` before its write and again after it, one step after the
other. In the normal case a step waits about ten seconds after its write, and the next one a few seconds before its own. The worst case
for a pair (two steps) is about 2 x (pre-wait + post-wait) = 4 x `ha_settle_seconds`: 120 s with the default 30 s
(inside the 5 minutes), 240 s at the field's maximum of 60 s. A plan with more config-mode steps on a pair
can exceed the 5 minutes if the pair never settles; then the deadline ends the apply and the steps not yet
written are reported as failed (or `ErrBusy` with the plan still `ready` when it was the first). Keep
`ha_settle_seconds` x 2 x (the plan's config-mode steps) well under 300 s. The apply request is held open while it waits (the API's
response deadline is the apply timeout plus two minutes), so a reverse proxy in front of Lachesis needs a read
timeout above the longest wait.

**What the API and UI do with `ErrBusy`.** If the first step of an apply is refused as busy, nothing has been
written: the plan engine returns the busy error, puts the plan back to `ready` and the API answers **409
`server_busy`** with a `Retry-After`; the plan screen says "The server is settling after the last change; try again
in about N seconds" (announced once; the ticking countdown is outside the live region) and offers **Apply
again**. `Retry-After` is kept within 1 to 300 seconds (10 when the adapter gave no hint). A busy refusal at a
later step is an ordinary failed apply (the earlier steps did write), with the step code `server_busy` and a
step message that says the earlier steps were applied.

## HA pairs

Kea HA replicates leases that clients obtain. It does not replicate the configuration, reservations
in the configuration file, or anything made with `lease4-add` / `subnet4-add` (verified on the Kea 3.2.1 simulator: `lease4-add` on one peer, then `lease4-get` on the other answered "Lease not found"; `subnet4-add` does not survive a restart). The adapter therefore
reports `FailoverTraits{ConfigSync: none, LeaseSync: automatic}` (or `none` when `send-lease-updates`
is off), and the plan engine writes **each** peer and verifies each. Because the writes are idempotent,
a pair that **shares a hosts database** is also safe: the first peer's `reservation-add` writes it, the
second finds it and reports "already present". Recorded evidence: `testdata/hostcmds-peer.json` (a
write on the primary, then the standby already has the reservation and lists it).
Pools and options are always per-peer (config mode).

Peers are suggestions (name, role, mode, state, the IP of the HA URL); the administrator pairs servers
explicitly.

## Vendor quirks

- **`config-set` restarts the HA state machine** on that server (observed: `hot-standby` becomes
  `waiting` and returns after about 10 seconds). Every config-mode write carries a warning saying so. The
  adapter waits for a settling pair (up to `ha_settle_seconds`) and refuses a config-mode write while the
  pair is not healthy (step 4 above), so the plan engine can write the peers one after the other.
- **`config-write` needs a writable configuration file.** If Kea cannot write it, the change is live
  and reported unconfirmed (recorded as a negative case, see Fixtures).
- **Kea's text is cleaned before it is shown**: the secret values in the configuration (hosts database and control-socket passwords) and the session's own password are removed from it, `password ...` patterns are masked, and it is cut to 300 bytes on a character boundary.
- **Result codes on writes**: "Host database not available" and an unsupported command are `ErrNotSupported`, a refused write is `ErrInvalidInput`, a duplicate is `ErrConflict`, so none surface as a server error.
- **A configuration backend** (`config-control` with databases) overrides `config-set`; config mode is
  then unavailable (reason in Probe).
- **Reservations that cannot be represented** (identified by `duid`, `circuit-id` or `flex-id`, or
  without an address) are not listed; Probe warns with the count. They are left untouched by writes.
- **One identifier per reservation**: a hardware address, or a client id (a client id that is `01` +
  MAC counts as that hardware address). Two different identifiers are `ErrInvalidInput`.
- **No pool exclusions**: Kea pools have none; a pool with exclusions is `ErrInvalidInput`.
- **Options**: scope and pool level only (server-level options are not writable in SDK v1, class
  options are not supported); identified by option space and code, the value in Kea's CSV text form.
- **Descriptions** are stored as `user-context.comment`; other `user-context` members are preserved.
- **Duplicates in a hosts database** are reported by Kea as "Database duplicate entry error" (result
  1, not 4); the adapter maps that text to `ErrConflict`.

