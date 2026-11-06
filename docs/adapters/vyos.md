# VyOS adapter

The DHCP server of VyOS 1.4 and later (VyOS runs Kea underneath). Vendor id `vyos`.
IPv4 only: a subnet with an IPv6 prefix is not listed (Probe says how many were left out).

## Transport and authentication

Lachesis speaks the VyOS HTTPS API. Every call is a `POST` of a form with two fields: `data`, a JSON
document, and `key`, the API key. The endpoint is the base URL of the router (`https://router.example.com`,
no path); the API must be enabled (`set service https api keys id <id> key <key>`).

| Endpoint | Used for |
|---|---|
| `/show` | `show version` (Probe), `show dhcp server leases`, and `show log dhcp server` (the HA state, below) |
| `/retrieve` | `showConfig` of `service dhcp-server` (scopes, pools, options, static mappings, HA) |
| `/configure` | a list of `set` / `delete` operations; the call commits them |
| `/config-file` | `save`, so the committed change survives a reboot |

The key is the adapter's one secret (`api_key`). It exists only inside the session, is never logged, is
not part of any error, and is stripped from any text the router sends back. A rejected key (HTTP 401 or
403) is `ErrAuthFailed`; a refused connection, a timeout or a TLS failure is `ErrUnreachable`. Redirects
are not followed (the key would travel to the redirect target). Answers are limited to 16 MiB (64 MiB for the
router's log, below). The server certificate is verified by default (system roots, a custom CA, or a pinned leaf:
the per-server TLS settings of the core). **Writes over plain `http://` are refused** (the capabilities are
unavailable, with the reason) unless the server sets `allow_plain_http`; reads stay allowed, with a warning.

Connection settings:

| Field | Default | Meaning |
|---|---|---|
| `allow_plain_http` | off | allow writes although the endpoint is plain `http://` (the key then travels in clear text) |
| `allow_unknown_ha_state` | off | **risky**: write although the HA state cannot be read; both routers' DHCP can go down (below) |
| `ha_settle_seconds` | 30 | how long a write waits for the router's HA state machine, before the commit and again after it (below); 0 does not wait; 0 to 120 |
| `max_items` | 100000 | the most leases (or static mappings of one subnet) one read accepts; a router that returns more is treated as faulty and the read fails |

### Least privilege

The API has no read-only role: whoever holds a key can change the whole configuration. Give Lachesis its
own key id (`set service https api keys id lachesis key ...`), keep the API on a management address
(`set service https listen-address`), restrict who can reach it (`set service https allow-client address`)
and use a certificate Lachesis can verify.

## Probe

`Probe` runs `show version`, reads `service dhcp-server` and reports:

- **Version**, for example `1.4.2` or `1.5-rolling-202611010000`. **A version below 1.4 is refused**: VyOS
  1.3 and earlier use the ISC DHCP server with another configuration tree. Probe returns `OK=false`, every
  capability is unavailable with that reason, and no read or write is attempted (the gate holds in a session
  opened later from the stored capabilities). An endpoint that answers but does not look like the VyOS API,
  or a version line that cannot be read, is `OK=false` too.
- A warning for a **rolling image** (the syntax can change between builds), for plain `http`, for a router
  with no DHCP server configured, and for subnets that are not listed (IPv6, no `subnet-id`, a name Lachesis
  will not put into a configuration path).
- **The lease table and the router's time zone.** Probe reads the lease table once: a table it cannot read makes
  `leases.read` unavailable with the reason, and rows it does not understand are a warning with their number.
  Lease times are printed in the router's local time with no zone, so they are read in the zone the router is
  configured with (`system time-zone`, read with `/retrieve`; UTC when unset), loaded by name so daylight saving
  time follows the date. A zone name that cannot be loaded here is a warning in Probe (and in the log at every
  lease read), and the times are then taken as UTC and may be wrong. The router's clock is a separate matter:
  `show date` is compared with this server's clock, and a difference of more than five minutes is a warning; it
  does not move the lease times.
- **HA**: `Details["ha_mode"]`, `ha_status` (`primary` or `secondary`), `ha_state` (this router's own state) and
  `ha_partner_state_last_seen`, and the capability `failover`, when the DHCP high-availability node exists
  (`service dhcp-server high-availability`, or `failover` on earlier 1.4 builds; both are read). Warnings when the
  router is the secondary, when its own state is not a serving one, and when the HA state cannot be read. The
  partner's state is a snapshot from this router's last transition and is only shown as last seen.

## Capabilities

| Capability | Status | Notes |
|---|---|---|
| scopes.read | yes | one scope per subnet; the scope id is the subnet's `subnet-id`, the name the shared network; ranges are the pools; lease time from `lease` |
| leases.read | yes | the text of `show dhcp server leases` (see quirks); the scope is found from the address |
| reservations.read | yes | the subnet's static mappings (a disabled mapping is not a reservation) |
| reservations.write | `commit` | create, update, delete of a static mapping; a repeat finds nothing to change and ends in a save (below) |
| pools.write | `commit` | create, update, delete of a range |
| options.write | `commit` | scope-level options of the supported set (below) |
| leases.write | no | the API has no call to release one lease |
| failover | when HA is configured | `Peers`; `Replicate` is not supported |

### How a write goes

Every write is plan, approve, apply and verify in the core; inside the adapter:

1. `/retrieve` the DHCP configuration and build the change against **that** read. What is already as wanted
   sends no change (a repeat is harmless), but a read shows the **running** configuration only, so such a write
   still ends in a **save** (step 3): if the save fails, the result is unconfirmed ("present in the running
   configuration ... the boot configuration was not checked"), so a repeat of a write whose save failed is never
   reported as done. A conflict (address, host name or MAC held by another mapping, an overlapping range, an
   option set to another value, an object changed since the plan) is `ErrConflict` before anything is sent.
2. With HA configured, the HA gate (below): the commit restarts Kea. Then one `/configure` call with every `set` /
   `delete` of the change: **one commit** (at most 64 operations). A refused commit changes nothing (VyOS
   discards it). An error text that says the configuration is locked or a commit is in progress, or HTTP 503,
   is read as a router busy with another configuration change: `ErrBusy` with a retry hint, for a commit that was
   not applied. Any other wording of a busy router is not recognised and is reported as a refused change.
3. `/config-file` `save`.
4. `/retrieve` again: every `set` must be present and every `delete` gone.
5. With HA configured, the router's log is polled until its state machine has moved and serves again (below).

The change is **live from step 2**, so a later problem is not a failure: a save that failed ("the change will
be lost when the router reboots") or a re-read that does not show the change (the router accepted something it
did not apply) is a nil error with `Verified=false` and a `VerifyNote`, which the plan engine reports as
`unverified`, never `success`. If the connection drops after `/configure` was sent, or the router answers with
something that is not the API's own (a web page from a proxy), the adapter looks at the configuration before
saying anything: a change that is there is carried on and noted, a change that is not is `ErrUnreachable`. After
the commit the save and the re-read do not stop for the caller's time limit ending. A replaced multi-valued
option must show exactly the new values in the re-read.

**A save writes the whole running configuration**, including commits that other operators made and did not save.
That is how the router's save works; the adapter cannot save only its own change.

`Preflight` (plan time) reads the configuration, builds every change as the write would, and reads the HA gate,
without writing: a conflict, the last-range refusal, an invalid name or an HA refusal on any router of a plan is
found before the first router is written. **It models the plan**: each change is judged on the configuration the
earlier changes of the same plan leave behind (their edits are applied to a copy of the tree), so "delete the two
mappings, then the range" is refused at plan time instead of at the third step of the apply. A pool update carries
no Before, so the range it replaces cannot be identified and its edits are not modelled; it still gets the checks
the write makes (the scope, the subnet, the range itself), the HA gate, and counts toward the cap. The HA gate is
read **without waiting**: a router that is settling is refused at plan time at once (busy, retry later) rather
than waited for, so a plan made in the ten seconds after someone's commit is refused. VyOS has no call that
validates a configuration without committing it, so a refusal that only the router's commit makes is still found
at apply time.

**Cost on an HA router.** Each change is its own commit followed by a wait for the pair to settle: about 12 s was
observed, and up to `ha_settle_seconds`. The plan engine's apply has a time limit (5 minutes by default) and a
plan that does not finish within it leaves the routers half changed. That limit covers the whole plan, which on a
pair is both routers: 2 x 8 commits x 15 s = 240 s. So a plan with **more than 8 changes for one router** of a pair
is refused at plan time (a per-router limit sized to the plan's 5-minute apply budget on a pair), and a plan of two
or more carries a warning with the expected time.
Each object's edits (a rename, a replaced option) are one commit; separate objects are separate commits.

### Names and values placed in configuration paths

The path of a `set` is a JSON list, so the JSON cannot be broken out of, but VyOS hands the elements to its own
command layer. Every element is therefore checked in the one place requests are built: ASCII letters, digits and
`_ . : / @ + -` only, at most 128 characters, no leading dash. Beyond that:

- A static-mapping name is the reservation's host name and must be a DNS name (labels of letters, digits and
  hyphens, joined by dots). A reservation without one is refused.
- Addresses, prefixes and MACs are parsed and written back in canonical form.
- Option values are checked by the option's type (IPv4 addresses, a domain name, a file name without `..`).
- Names read from the router (a shared network, a mapping, a range) pass the same check before they are listed
  or used in a path; a subnet inside an unsafe name is left out of the listing, not written to.

### Options

Scope level only, for these codes (the VyOS keyword in brackets): 3 routers (`default-router`), 6 DNS servers
(`name-server`), 15 domain name (`domain-name`), 42 NTP servers (`ntp-server`), 44 NetBIOS name servers
(`wins-server`), 66 TFTP server name (`tftp-server-name`), 67 boot file name (`bootfile-name`), 119 domain
search list (`domain-search`). Others are not written; those VyOS shows that are not on the list are not
returned. Server-level, pool-level, class-scoped and reservation-level options are `ErrInvalidInput` (VyOS has
none of the last two; a static mapping carries no options of its own).

## HA pairs

VyOS runs Kea's HA hook between two routers (`high-availability` or `failover`, with `mode` `active-passive`
(hot-standby) or `active-active`, `status` `primary` or `secondary`, `source-address` and `remote`; the `name` is
the **partner's host name**, which Kea needs to equal the partner's own server name). Kea HA replicates the
leases clients obtain; **VyOS does not copy DHCP configuration to the partner** (verified on the simulator pair).
The adapter therefore reports `FailoverTraits{ConfigSync: none, LeaseSync: automatic}`, the plan engine writes
**each** router and verifies each, and `Replicate` is not supported. Pair the routers explicitly; a plan against
the primary writes the primary first, and Probe warns when the server a plan starts from is the secondary.

### The HA state, and waiting for it

The API has no call for the state of the HA state machine, but the router's DHCP log does carry it
(`/show` `["log","dhcp","server"]`): every transition is a line

    <time> kea-dhcp4[<pid>]: INFO  HA_STATE_TRANSITION <server>: server transitions from READY to HOT-STANDBY state, partner state is HOT-STANDBY

The last such line is the local state, and the partner's state as it was at that transition (a snapshot, which
is stale as soon as the partner moves: it is shown as last seen and never used in a decision). Only lines that
begin like this count: a Kea INFO line from the start of the line, naming this router's own host name (read with
`show host name`; if that cannot be read the match falls back to any server name and Probe says so). Text a client puts into another log line (a host name) is therefore not counted. This is a
filter, not authentication: the log has no integrity of its own. Nothing else of the log is read or kept. The log
has no shorter form than the router's whole journal since boot, so it is read with a 64 MiB limit; a log larger
than that is `ErrConflict` with the command that trims it (`journalctl --vacuum-size`).

A commit makes VyOS reconfigure Kea, whose HA state machine then walks waiting, syncing and ready to its
serving state. On VyOS 2026.11.05 the walk shows in the log **about ten seconds after the commit** and takes a
second or two; the API answers the commit within a second. The write therefore:

- **before the commit** reads this router's own state: serving (`hot-standby`, `load-balancing`,
  `passive-backup`, `backup`) goes ahead; settling (`waiting`, `syncing`, `ready`) is waited for, polling every two
  seconds for up to `ha_settle_seconds` (the log is read whole each time, so a large one is polled less often);
  if it is still settling at the end, the write is `ErrBusy` with a retry hint and nothing written; any state that
  waiting does not cure is `ErrConflict`, with the remedy in the message: `partner-down` and
  `communication-recovery` (bring the partner back or repair the link, wait for both routers to show
  hot-standby, plan again), `terminated`, `unavailable`, `in-maintenance` ... **A log with no
  transition at all is an unknown state and is refused** (`ErrBusy`, saying so): on a long-running pair the last
  transition ages out of the router's journal, and a commit would restart a pair whose condition nobody knows.
  `allow_unknown_ha_state` on the server lets such a write go ahead, with a warning. **Risky**: with it on, a
  commit can restart the HA state machines of both routers at once, and DHCP can go down on both.
- **after the commit** (and the save and re-read) polls the log until it holds **more transitions than before the
  commit** (from none, if there were none) and the state is serving again, for at most `ha_settle_seconds`, so the
  plan engine's next step (the partner's commit) does not start while this router is restarting. A pair that has
  not come back, or a log that shows no change in time, is a warning on a result that stays confirmed: the change
  is live.
- **at plan time** (`Preflight`) the same gate is read without waiting, so a refusal is found before the first
  router of the plan is written.

Because the walk appears about ten seconds late, a commit made by someone else in the last ten seconds is not yet
visible to the check before a write.

## Vendor quirks

- **Static mappings are keyed on a MAC and named by the host name.** A client id alone is refused; a client id
  that differs from the MAC is refused. A description and per-reservation options are refused (VyOS has
  neither), so an update that would drop them is visible in the plan, not silently lost.
- **One range name per pool.** A new range gets the smallest unused decimal name (`0`, `1`, ...), as the VyOS
  documentation does; an existing range keeps its name.
- **No pool exclusions**: define separate ranges around the excluded addresses. A pool with exclusions is
  `ErrInvalidInput`. A subnet-level `exclude` on the router is not shown.
- **The lease table has no JSON form** through the API. `show dhcp server leases` is parsed by column (the
  line of dashes under the header gives the columns, so an empty cell is not a problem; without it, columns are
  separated by two or more spaces). A row that does not hold an IPv4 address and a MAC, or whose state is not
  known, is skipped, counted in the log and, at probe time, in a warning, never guessed. **A table that cannot be
  read is an error, not an empty list**: output with no header, or rows none of which could be read. Times are
  corrected for the router's zone (Probe).
  A lease host name that is not a plain DNS name is dropped (it is client data). The Pool column is the shared
  network name; the Origin column is `local` on both peers, also for replicated leases. A lease past its
  expiration that the table still calls `active` is reported expired.
- **VyOS refuses a subnet with neither a range nor an active static mapping** at commit. A change that would
  leave a subnet with neither (deleting its last range or last active mapping) is refused at once with that
  sentence (`ErrInvalidInput`).
- **Names**: static mappings may be named with dots (`host50.example.com`); a shared network has no `subnet-name`
  in this version, so the scope name is the shared network's.
- **A router with no DHCP server** configured (or not running) returns no scopes and no leases; it is not an
  error.
- **Router text is cleaned before it is shown**: the session's key is removed, control and format characters
  are replaced, `password ...` patterns are masked, and the text is cut to 300 bytes on a character boundary.

