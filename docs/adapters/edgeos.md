# Ubiquiti EdgeRouter (EdgeOS) adapter

The built-in DHCP server of Ubiquiti EdgeRouters (EdgeOS 2.x, the Vyatta-derived firmware). Vendor id `edgeos`.
DHCPv4 only (EdgeOS's `dhcpv6-server` is a different tree and is not covered).

> **Status: not validated on real firmware.** EdgeOS has no documented API. Everything below is derived from
> community documentation and client code (sources S1 to S9), and **no fixture in this repository was recorded
> from a device**: they are hand-built from the documented shapes and labelled `SYNTHETIC` (the file's
> `description`, `testdata/README.md`, this page). Section "API facts and their evidence" lists every endpoint,
> field and status code the adapter relies on, which sources agree, and which are inferred. Live validation on
> a non-production EdgeRouter is a tracked follow-up; until it is done, treat every write as
> experimental. A write the adapter cannot confirm is never reported as a success (see "Writes").
>
> **EdgeOS 3.x (3.0.0, 3.0.1) is unvalidated** (tracked as an open validation task). Nothing in the adapter refuses a firmware version; an unknown or newer
> release is a probe warning (see "Firmware versions"). No document found describes a change to the web API or to commit-confirm in 3.x,
> and none of this was checked on hardware (F39).
>
> The **SSH transport** (`transport: ssh`) is also **experimental and not validated on EdgeOS**: its protocol was exercised only
> against a VyOS simulator, which is a sibling system, not EdgeOS (F27 to F38 say which facts were observed there and which are inferred).

## Sources

| Id | Source | What it is |
|---|---|---|
| S1 | [ubntwiki, "EdgeOS API Guide"](https://ubntwiki.com/products/software/edgeos/api) (raw: `?do=export_raw`) | community wiki; login, `get.json`, `data.json`, `batch.json` example, `config/save.json` |
| S2 | [matthew1471/EdgeOS-API, `Documentation/`](https://github.com/Matthew1471/EdgeOS-API/tree/master/Documentation) (`README.adoc`, `REST API/*.adoc`) | the most detailed write-up: login, sessions, CSRF, `batch.json`/`set.json`/`delete.json`, `data.json` shapes |
| S3 | [matthew1471/EdgeOS-API, `EdgeOS.API/WebClient.cs`](https://github.com/Matthew1471/EdgeOS-API/blob/master/EdgeOS.API/WebClient.cs) | C# client that works against devices: login, cookie handling, `X-CSRF-TOKEN` header, batch, logout |
| S4 | [andrewstuart/edgeos-rest](https://github.com/andrewstuart/edgeos-rest) (`pkg/edgeos/csrf.go`, `client.go`, `config.go`) | Go client; `config.go` is a Go type generated from a **real `get.json` of a router with a DHCP server** |
| S5 | [brontide/aioedgeos](https://github.com/brontide/aioedgeos) (`aioedgeos/api.py`) | Python client: login, cookies, `get.json`, `data.json` |
| S6 | [rjhornsby's gist "Show EdgeOS DHCP lease table"](https://gist.github.com/rjhornsby/544cd3d79ca8f813762a246735789362) | shell script: login with `curl -d`, `data=dhcp_leases` |
| S7 | [Logan Marchione, "EdgeRouter Lite dnsmasq setup"](https://loganmarchione.com/2016/08/edgerouter-lite-dnsmasq-setup/) | `use-dnsmasq`: what it ignores, that CLI status is unsupported under it |
| S8 | Ubiquiti UISP help centre, "EdgeRouter - DHCP Server Using Dnsmasq" and "... Static Mapping" | **seen only as search-result summaries** (the site refused the fetch); used for CLI names only |
| S9 | Teldat OSDx documentation, "DHCP failover" (a Vyatta-family system, **not EdgeOS**) | **seen only as a search-result summary**; the only evidence for the `failover` block |

| S10 | VyOS forum, ["How do I cancel a commit-confirm if the commit fails?"](https://forum.vyos.io/t/how-do-i-cancel-a-commit-confirm-if-the-commit-fails/17151) | VyOS (not EdgeOS): quotes the prompt "commit-confirm will automatically reboot in N minutes unless changes are confirmed", the reboot on timeout, and a device that rebooted after a commit that failed. **Seen only as a search-result summary** |
| S11 | [juanico10/Ubiquiti](https://github.com/juanico10/Ubiquiti) (a Spanish-language EdgeRouter guide) | says `commit-confirm` reboots the device after 10 minutes by default unless `confirm` is run. **Seen only as a search-result summary** |
| S12 | [HostiFi, "Ubiquiti finally releases EdgeRouter 3.0 with brand new design"](https://www.hostifi.com/blog/ubiquiti-finally-releases-edgerouter-3-0-with-brand-new-design) | EdgeOS 3.0: redesigned web interface, WireGuard. Names no API change. **Search-result summary only** |
| S13 | [chromic.org, "EdgeRouter ER-X Firmware v3.0.1"](https://chromic.org/blog/erx-firmware-v301/) | dates: v3.0.0 2025-07-29, v3.0.1 2025-10-30 (ER-X, ER-X-SFP, EP-R6, ER-10X); one upgrade report from v2.0.9-hotfix.7, no API or CLI details (fetched) |
| S14 | [beyondcloud.solutions, EdgeOS v3 release](https://beyondcloud.solutions/edgeos-v3-release-wireguard-vpn-new-ui-ubiquitis-bold-new-era) | third-party summary of 3.0: new UI, WireGuard, opt-in analytics. Names no API change. **Search-result summary only** |
| S15 | community gist (dabrovnijk) and the [UISP help article "EdgeRouter - How to Find the Base MAC Address"](https://help.uisp.com/hc/en-us/articles/22591166862743-EdgeRouter-How-to-Find-the-Base-MAC-Address) | example `show version` output: `Version:`, `Build ID:`, `HW model:`, `HW S/N:` lines (v1.10.x and a newer EdgeRouter 4). **Search-result summaries only** |
| S16 | direct observation on the a VyOS simulator (rolling release; **not EdgeOS**), 2026-11-07 | the CLI, the configure session, the commit-confirm prompt and the rollback, driven by the shared SSH package (see "What the VyOS simulator did and did not prove") |

S1 and S2 are not independent (S1 says it took a subset from S2). S3 is S2's author's client. S4, S5 and S6 are
independent of them and of each other.

## Transport and authentication

With the default `web` transport everything goes over HTTPS to the router's web interface (the `ssh` transport is described under "SSH transport"). The adapter accepts an `https://host[:port]` endpoint and
nothing else: a path, a query, credentials in the URL or plain `http` are refused, because the login form carries the
password.

One EdgeOS web session per adapter session:

1. **Login**: `POST /` with the form fields `username` and `password` (`application/x-www-form-urlencoded`).
   Success is `303 See Other` with the session cookies; failure is `200 OK` with the login form again
   (S1, S2; S3 handles exactly these two cases). The SDK's HTTP client does not follow redirects, so the 303 and its
   cookies are seen. A `303` without a session cookie, a `200`, `401` or `403` is `ErrAuthFailed`; anything else is
   `ErrUnreachable`.
2. **Cookies**: the adapter keeps every non-empty cookie of the login answer and sends them back. It does **not**
   assume the session cookie's name: S1 and S2 show `PHPSESSID`, S5 reads `beaker.session.id` (S2 notes the backend
   moved from PHP to a Beaker session in firmware 1.9.7). The `secure` attribute is ignored (the adapter only talks https).
3. **CSRF**: the login also sets a cookie `X-CSRF-TOKEN`; every state-changing request (the batch) sends its value in the
   `X-CSRF-TOKEN` request header (S2: required for anything but GET/HEAD/OPTIONS/TRACE; S3, S4, S5 send the header). S2
   says the token is also accepted in the query string; the adapter never uses that form (a token in a URL ends up in logs
   and error text). `Referer` and `X-Requested-With: XMLHttpRequest` are sent as the web interface does (S4); nothing says they are
   required.
4. **Session end**: a read that gets 401, 403, a redirect or a non-JSON answer logs the old session out (best effort), logs in
   again **once** and repeats the read; a write is never repeated blindly. How an expired session is answered is not documented
   anywhere (see the evidence table); the adapter treats all of those the same. A rejected login **latches**: the rest of the
   session fails fast with `ErrAuthFailed` and presents the credentials no more (a loop of wrong passwords could lock the account).
   The cookies a device may hand the session are bounded (16 cookies of at most 4096 bytes each).
5. **Logout**: `GET /logout` (S2, S3) on `Close`, best effort and bounded to five seconds. The credentials are dropped
   from the session on `Close`. A session also ends by itself after 15 minutes without a request (S1; S2 says 24 minutes, see below).

Credentials live in `adapter.Secrets` and, inside the session, only as the bytes needed to log in again. They are
never logged, never part of an error, never in `Native`. Cookies, the CSRF token and response bodies are never put in an
error or a note either: every message is fixed text.

### Least privilege

EdgeOS has two account levels, `admin` and `operator` (S2, `auth.json`). Changes need `admin` (an `operator` account is
read-only according to the community documentation; **not confirmed on firmware**), and there is no finer role: the
account Lachesis uses can reconfigure, reboot and reset the whole router. Give Lachesis its own account, restrict the web
interface to a management network (`service gui listen-address`), and keep it off networks clients can reach. Lachesis
only sends the requests listed below.

### TLS: the usual self-signed certificate

A new EdgeRouter serves a self-signed certificate. Verification stays on. The server form's endpoint help explains the
two ways to trust it:

- **pinned** (usual): set the TLS mode to *pinned* and paste the certificate's SHA-256 fingerprint:

      openssl s_client -connect router.example.com:443 </dev/null 2>/dev/null | openssl x509 -noout -fingerprint -sha256

  The pin is the trust anchor: the chain and the name are not checked, so it works when the self-signed certificate has the
  wrong name, and it fails closed when the certificate changes (an `ErrUnreachable` from the handshake; the password is
  never sent). Upper case with colons, as `openssl` prints it, is accepted.
- **ca**: install a certificate signed by your own CA on the router and give Lachesis the CA.

Do not use skip-verify: it is a permanent warning badge and an audit event.

Connection settings:

| Field | Default | Meaning |
|---|---|---|
| `max_items` | 100000 | the most leases (or static mappings of one subnet, or subnets) one read accepts; a router that returns more is treated as faulty and the read fails. One answer may not exceed 16 MiB |

## Probe

`Probe` runs `GET /api/edge/data.json?data=sys_info` and `GET /api/edge/get.json` (both read-only) and reports:

- **Detection and version**: `sys_info` answers `{"success": "1", "output": {"sw_ver": "EdgeRouter.ER-e300.v2.0.9-beta.3.5296219.200507.1538", ...}}`
  (S1, S2). The release is read from `v<release>` (`2.0.9-beta.3`), `Details["product"]`/`["model"]` from the first two parts,
  `Details["firmware"]` is the full string. An endpoint whose `sys_info` is not that JSON, or whose version does not start
  with `Edge`, is reported `OK=false` with every capability unavailable and the reason (not an error).
  A 1.x release, a release newer than 2.x (3.0.0, 3.0.1: unvalidated), or a version string of another shape, adds a warning: the shapes are documented for 2.x. A version is never a refusal (see "Firmware versions").
- **The DHCP server**: `Details["dhcp_backend"]` is `isc` or `dnsmasq` (`use-dnsmasq`), `Details["dhcp_server"]`
  `enabled`/`disabled` (`service dhcp-server disabled true`; the scopes are then shown as disabled).
- **Experimental**: until that validation is done, the first warning is `EXPERIMENTAL: ...`, `Details["status"]` is `experimental` and the
  adapter's description says so.
- **Skip-verify**: a warning when the server's TLS settings switch verification off.
- **Failover**: `Details["failover"] = "isc"`, `Details["failover_partner_check"] = "not enforced"`, `ProbeResult.Failover`
  (config sync `none`, lease sync `automatic`) and a prominent `ISC FAILOVER, PARTNER NOT CHECKED` warning when any subnet has a
  `failover` block; see below.
- **No CSRF token** at login: a warning (changes may be refused).
- **No subnet configured**: the write capabilities are unavailable (`create the shared network and subnet on the router first`).

## Capabilities

| Capability | Status | Notes |
|---|---|---|
| scopes.read | yes (mode `config`) | shared network + subnet, address ranges, scope options |
| reservations.read | yes (mode `config`) | static mappings |
| leases.read | yes (mode `data.json`) | the router's lease table; see the quirks |
| reservations.write | yes (mode `batch`) | create, update, delete a static mapping |
| pools.write | yes (mode `batch`) | create, change, delete an address range (`start`/`stop`) |
| options.write | yes (mode `batch`) | scope options 3, 6, 15, 42, 51 |
| leases.write | **no** | the only release operation the API documents (`operation/release-dhcp.json`, S2) releases the router's *own* DHCP client lease on an interface, not a lease it handed out |
| failover | **no** | ISC failover is reported on the scope and in Probe, not managed |

`Preflight` is implemented: EdgeOS has no dry-run endpoint, so it validates the plan against the current configuration (names,
values, collisions) without sending anything.

### Mapping

| Lachesis | EdgeOS tree (`service dhcp-server`) |
|---|---|
| scope | `shared-network-name <N> subnet <CIDR>`; the scope's name is `<N>`; the subnet is the CIDR masked (the node's name may have host bits set, as in S4's capture and S7's example; the node name is kept in `Native["subnet_key"]` and used for writes) |
| scope id | `<N>@<CIDR with / as _>`; characters of `<N>` outside `[A-Za-z0-9._-]` are written `~XX`, so the id is path safe and reversible |
| pool | `start <A> { stop <B> }` (several allowed) |
| pool exclusions | not modelled; `exclude` addresses are listed in `Scope.Native["exclude"]`, and a write with exclusions is refused |
| reservation | `static-mapping <name> { ip-address, mac-address }`; `<name>` is the reservation's host name (required); no description, no client id, no options |
| option 3 / 6 / 15 / 42 / 51 | `default-router` / `dns-server` (multi) / `domain-name` / `ntp-server` (multi) / `lease`; scope level only; `lease` is also `Scope.LeaseTimeSec` |
| lease | `dhcp_leases` output: `expiration`, `mac`, `client-hostname`, placed in the subnet of its shared network that contains the address |

Static mappings the model cannot represent (no address, no hardware address, outside the subnet, the network or broadcast
address) are not listed (a log line counts them). Subnet nodes that are not IPv4 CIDRs, or repeat an earlier
subnet, are skipped (SDK rule: subnets are unique per server) and counted in a Probe warning.

## Writes

Every write goes through the plan engine; the adapter has no other path. For each op:

1. **Read** the configuration (`GET /api/edge/get.json`), keeping only `service.dhcp-server`; the rest (password hashes, VPN keys)
   is dropped as soon as it is parsed. Plan the op against it: validation, collisions, drift against `Before`. When the snapshot
   already shows the wanted state, nothing is sent and the result is **unconfirmed**: "present in the running configuration; the router's
   boot configuration was not checked". An earlier attempt may have left the change live but unsaved, and nothing here can see the boot
   configuration, so a retry never turns into a success by finding its own earlier work.
2. **Send** one `POST /api/edge/batch.json` with `SET` and/or `DELETE` below `service.dhcp-server` (the batch endpoint of S2 and S3).
   The device sets, **commits and saves inside that request** and reports each step: the answer carries `SET`/`DELETE`, `COMMIT`,
   `SAVE` ({`"success": "1"`, `"failure": "0"`}) and `success` (S1 and S2 examples show all of them for a request that asked only
   for a `SET` or a `DELETE`). The adapter therefore sends no separate commit or save request.
3. **Re-read** the configuration and the lease table. The configuration decides: the result is *confirmed* only when the batch,
   the commit and the save all reported success **and** the re-read shows the change. The lease table adds warnings (an address
   still leased to another client, leases outside a shrunk range) and a lease table that cannot be read adds a warning, never a failure.

   **The lease re-read is advisory, deliberately unlike the issue's wording ("verify by re-reading config and leases").** A lease is
   a binding the server made with a client, not configuration: a correct change does not make any lease appear or disappear (a
   reservation takes effect at the client's next renewal), so requiring the lease table to show the change would report correct writes
   as unverified, and under `use-dnsmasq` the lease table may not be readable at all. The configuration is the only thing that can
   confirm a change; the lease table is read to warn about conflicts with current leases.

| What happened | Result |
|---|---|
| 401/403 on the batch (session, CSRF token or account level refused) | `ErrAuthFailed`, nothing was processed |
| `SET`/`DELETE` reported as failed by the device (first batch) | `ErrInvalidInput`: "the router reported the change as failed" |
| `COMMIT` reported as failed, and the re-read does not show the change | `ErrInvalidInput`: "the router reported the commit as failed; the change is not live (a staged change may remain on the router)" |
| no `COMMIT` member at all, and the re-read does not show the change | `ErrUnreachable`: "the router did not report a commit; the change is not live; a staged change may remain on the router" |
| `COMMIT` failed or missing, but the re-read shows the change | unconfirmed (nil error, `Verified=false`, a note) |
| a 200 answer that is not the API's JSON (a web page, `OK`, `[]`) | the outcome is unknown, so the configuration is read back: unconfirmed if the change is visible, else `ErrUnreachable` |
| the change is already in the running configuration (nothing sent) | unconfirmed: the boot configuration was not checked |
| committed, `SAVE` failed or missing | unconfirmed: live, but a reboot may undo it |
| every status says success, the re-read does not show the change (a **discarded write**) | unconfirmed |
| everything succeeded, the re-read failed | unconfirmed |
| the answer was lost or malformed (5xx, timeout) | unconfirmed if the re-read shows the change, else the error |
| a later step of a two-step write failed | unconfirmed; for an option replace the note says the option is now **absent** and must be re-created |
| every status success and the re-read shows the change | confirmed |

"Unconfirmed" is the SDK's way of saying `unverified`: a nil error with `Verified=false` and a note; the plan engine reports the
step `unverified`, never `success` (the product's rule for every write). The adapter does not return `ErrUnverified` itself (the SDK reserves it
for the engine). A failed commit is not rolled back by the adapter: there is no documented discard endpoint, and the web session is
logged out on `Close`.

### The two-step write

Setting a multi-valued node (`dns-server`, `ntp-server`) adds a value; and the order in which a `DELETE` and a `SET` that name the same
node are processed inside one batch is not documented. Replacing the values of such an option is therefore two batches: `DELETE` the
node, then `SET` the list. Everything else uses disjoint keys inside one batch (a rename or a moved range start is a `DELETE` of the old
node and a `SET` of the new one). The SET payload for a multi-valued node (a JSON array) has **not been seen**; it is inferred from the
shape `get.json` reports. If the router answers otherwise, the re-read shows it and the write is reported unconfirmed.

If the second batch fails, the option is **gone** (the first batch is committed): the note says "the option is now ABSENT on the router and
must be re-created", and planning the same update again does that, because an update whose option is absent on the router, while `Before`
shows it existed, is applied as a create (with a warning). Without `Before` (a preflight) it stays a conflict.

### What the adapter checks first (nothing is sent for a failed check)

- Every name that becomes a key in the tree (static mapping name, shared network, subnet) is limited to letters, digits, dot, hyphen and
  underscore, starting and ending with a letter or digit, at most 127 characters, no `..`. A name already in the configuration that does not pass
  (spaces, slashes, quotes, Unicode) is **read and listed, never written**: planning a write into such a scope or deleting such a mapping is
  `ErrInvalidInput`. The scope id given by a caller is never used as a path: the subnet is looked up in the snapshot and its own names are used.
- Values: IPv4 addresses (unicast, not `0.0.0.0`/multicast/broadcast), MAC addresses (6 octets), DNS names (RFC 1123 labels), `lease` a whole
  number of seconds from 1 to 4294967295, at most 8 addresses for `dns-server`/`ntp-server`, no duplicates.
- A reservation inside the subnet (not the network or broadcast address), with a host name, a hardware address and nothing else (a description,
  a client id or options are refused: there is no place for them). Update keeps the address. A rename deletes the old mapping node and creates a new one, so a mapping that holds settings Lachesis does not model (anything but `ip-address` and `mac-address`, for example `static-mapping-parameters`) is **not renamed**: `ErrInvalidInput` names the settings; rename it on the router. Verification of a rename also requires the old node to be gone.
- Collisions are looked for among **every** static-mapping node, including the ones Lachesis cannot represent: the same name (case
  insensitive), the same address, the same MAC in the subnet is `ErrConflict`. A range must lie inside the subnet, may not overlap another or
  start where another starts, and may not have exclusions. A scope option that already has another value is `ErrConflict` on create.
- A `default-router` outside the subnet is allowed with a warning.

The check and the batch are not one transaction: another administrator can change the router between them. The batch names only the keys it
changes, so it never overwrites the rest of the configuration; the re-read tells whether the result is what was planned.

## ISC failover and `use-dnsmasq`

- **ISC failover** (a `failover` block under a subnet): reported on the scope (`Scope.Failover`: relationship name, peer address, config sync
  `none`, lease sync `automatic`) and in Probe (`Details["failover"]`, `ProbeResult.Failover`, a warning). It is a **read-only capability note**:
  the adapter does not implement `FailoverAware`. The router does not replicate configuration to its partner, so the plan engine writes both
  members when the partner is paired in Lachesis, and warns when it is not. The adapter does not check that the partner is serving before a write
  (the rule never to write a peer whose partner is not serving cannot be honoured without a partner-state read; the engine's pairing is the only guard). The block's shape is **inferred
  from S9, a Vyatta-family document**; it has not been seen on EdgeOS.
- **`use-dnsmasq`** (`use-dnsmasq: enable`): the router then ignores ISC-only settings, including failover (S7), so failover is not reported;
  and the lease table (`dhcp_leases`) is not documented for dnsmasq (S7 says the CLI status commands are unsupported under it) and may be empty or
  incomplete. Probe says so (`Details["dhcp_backend"] = "dnsmasq"`, a warning). Configuration reads and writes are the same.

## SSH transport (experimental)

A per-server setting `transport` chooses how Lachesis talks to the router: `web` (the default; everything above) or `ssh`. The server's
endpoint stays the router's `https://host[:port]` (it names the host); with `ssh` the TLS settings are not used and the SSH host key is
pinned instead. The SSH port (default 22) is declared to the egress guard as an extra destination. The connection is opened by
`Deps.Dialer`, so it works directly and through a relay; SSH itself is `golang.org/x/crypto/ssh` run over that connection by a shared SSH package.

Settings (schema-driven connection form, no migration): `transport`, `ssh_host_key`, `ssh_port`, `commit_confirm_minutes` (default 5, 1 to 30).
Credentials: the username plus a password **or** an SSH private key (and its passphrase). The web transport still requires a password.

### Host keys

`ssh_host_key` holds one or more SHA-256 fingerprints (`SHA256:...` as `ssh-keygen -lf` prints, or 64 hex digits). With none set, connecting
fails **before any credential is offered**, and the error says which key type and fingerprint the router presented, so an administrator can
compare it with the router and approve it. There is no trust on first use. A key that differs from the pins is a separate error, never
treated as "unapproved".

### Reads

| Data | Command (run through the router's login shell) |
|---|---|
| firmware | `/opt/vyatta/bin/vyatta-op-cmd-wrapper show version` (`Version:`, `HW model:`) |
| DHCP configuration | `show configuration commands`, filtered **on the router** (`grep`) to `set service dhcp-server ...`, so the rest of the configuration (hashes, keys) never leaves it; the wrapper's exit status is echoed into the stream so an empty answer is told from a failed one |
| leases | `show dhcp leases` (table: address, hardware address, expiration, pool, client name) |

The `set` lines are parsed into the same object tree `get.json` yields, so scopes, reservations, planning and verification are the same code
as the web transport. Probe reports `Details["transport"] = "ssh"` and the capability mode `ssh`.

### Writes: commit-confirm

One configure session per write, driven through a non-interactive `/bin/vbash -s` with a marker after every line (so no prompt has to be
recognised and the router cannot forge the end of an answer):

1. `source /opt/vyatta/etc/functions/script-template`, `configure`
2. the plan's `delete` lines, then its `set` lines (a configure session applies lines in order, so unlike the web batch the order of a delete and a
   set of the same node is defined). Names and values are validated by the planner and checked again against a conservative character set
   before they reach a command line. A `set` or `delete` the router rejects: `discard`, `ErrInvalidInput`, nothing was committed.
3. `commit-confirm <N> <<< y` (the prompt "Proceed ? [Y/n]" reads standard input; the here-string answers it)
4. **read back on a separate channel** and check the plan's expected end state
5. only if it matches: `confirm`, then `save`, then read back again. Confirmed only when all of that succeeded.

| What happened | Result |
|---|---|
| connection lost, or a line rejected, before commit-confirm | error; nothing was committed (the candidate configuration dies with the session) |
| the commit reported failure | `ErrInvalidInput` saying the change was **not confirmed** and the router will undo anything it applied (a commit may apply part of a change before it fails, so the timer is deliberately not cancelled) |
| connection lost during commit-confirm | unconfirmed; the note says whether a read-back shows the change, that nothing was confirmed or saved, and that the router will undo it |
| the read-back fails or does not show the change | unconfirmed, **left pending on purpose**: the router undoes the change when the window ends |
| `confirm` refused or lost after a good read-back | unconfirmed: live until the window ends, not saved |
| confirmed, `save` failed or lost | unconfirmed: live and confirmed, a reboot may undo it |
| confirmed and saved, final read-back shows the change | confirmed |

**What "rolls back" means on EdgeOS (inferred, F33):** `commit-confirm` does not restore the old configuration in place. When the window ends the
router **reboots** into its saved configuration (S10, S11, and observed on the simulator). A write that is left unconfirmed therefore costs a router
reboot, and any running change that was never saved is lost with it. This is why the window is five minutes by default and why nothing here retries
a commit-confirm. The result's `Native` says `commit_confirm: left pending | confirmed`.

### Firmware versions

No code path refuses a firmware version. The web probe refuses only a device that is not EdgeOS at all (no `sys_info`, or a software string that does
not start with `Edge`); the SSH probe the same with `show version`. An unparsable version, a 1.x release, and now any release newer than 2.x
(3.0.0, 3.0.1) are **warnings** and the capabilities are unchanged. Session handling has no version logic either: the login, cookies and CSRF handling are
the documented 2.x shapes, and a 3.x device that answers differently fails the same checks a broken device would (a login that is not a 303 with
cookies is `ErrAuthFailed`/`ErrUnreachable`, a non-JSON answer is "not the EdgeOS API"), with a message, not a version refusal.

**EdgeOS 3.0.1 is unvalidated (tracked as an open validation task).** What the sources say about 3.x (S12 to S14): 3.0.0 is dated 2025-07-29 and 3.0.1 2025-10-30 (ER-X family at least),
with a redesigned web interface, WireGuard and opt-in analytics. **None describes a change to `/api/edge/*`, the login, the CSRF token or `commit-confirm`.**
Absence of a documented change is not evidence of no change: it is unverified on hardware (F39).


## Quirks

- **Lease times**: the lease table has an `expiration` ("2020/06/08 13:07:17", S2) and no start. The zone is not documented; the adapter assumes UTC.
  A lease whose expiration is past is reported `expired`. A network with no leases is the empty string (S1, S2), not an object.
- **Single value or list**: a multi-valued node is an array in `get.json` (S4); the adapter also accepts a plain string.
- **Subnet node names with host bits** are real (S4, S7): the node name is used for writes, the masked prefix is the scope's subnet.
- **Release of a lease** is not offered (see Capabilities).
- **`config/save.json` is not used.** The issue text lists it as the save endpoint, but S1 and S2 document `GET /api/edge/config/save.json` as the
  *download* of the configuration (it writes a temporary file whose path it returns, to be fetched from `/files/config/`). The save the adapter
  relies on is the `SAVE` member of the batch answer.
- **Heartbeat** (`/api/edge/heartbeat.json`, S1, S2) is not used: sessions are short.
- **Data minimisation**: `get.json` returns the whole configuration, secrets included. The adapter reads it, keeps `service.dhcp-server` and
  discards the rest in memory; the recorder hook below does the same before anything is captured. (S2 documents `partial.json?struct=...` for a
  section only; it is described there as used internally and has not been tried.)

## API facts and their evidence

*Multi*: two or more independent sources agree. *Single*: one source (or sources that copy each other). *Inferred*: no source; derived from the shape
of something else. **Nothing here has been seen by the adapter's authors on real firmware.** "Real capture" means a source that shows output of a
real device.

| # | Fact the adapter relies on | Evidence | Status |
|---|---|---|---|
| F1 | `POST /` with form `username`, `password`; `303` + cookies = success; `200` + login form = failure | S1, S2, S3 (code), S4/S5/S6 (post the form to `/`) | Multi |
| F2 | the failure page contains "The username or password you entered is incorrect" (the adapter does **not** depend on the text) | S1, S2, S3 | Multi (unused) |
| F3 | session cookie name | `PHPSESSID` (S1, S2); `beaker.session.id` (S5) | **Conflict**: the adapter takes any non-CSRF cookie |
| F4 | login sets cookie `X-CSRF-TOKEN`; non-GET requests send the same value in header `X-CSRF-TOKEN` | S2 (doc), S3, S4, S5 (code) | Multi |
| F5 | session lifetime 15 minutes, refreshed by every request | 15 min: S1; 1440 s (24 min): S2 | **Conflict** (immaterial: the adapter logs in again on expiry) |
| F6 | how an expired session is answered (401/403/redirect/HTML) | none | **Inferred** (all handled alike) |
| F7 | `GET /logout` ends the session | S2, S3 | Multi |
| F8 | `GET /api/edge/get.json` answers `{"success": true, "GET": {...config...}, "SESSION_ID": "..."}` | S1 (config + SESSION_ID), S4 (struct), S5 (`success`, `GET`) | Multi |
| F9 | `service.dhcp-server`: `disabled`, `hostfile-update`, `use-dnsmasq` (text), `shared-network-name.<N>.authoritative`, `.subnet.<CIDR>.{default-router, dns-server[], domain-name, lease, start.<A>.stop, static-mapping.<name>.{ip-address, mac-address}}` | S4 (real capture); CLI names of `static-mapping`, `ip-address`, `use-dnsmasq enable`, `hostfile-update` in S7 | Single (shape), Multi (names) |
| F10 | subnet node names may have host bits set | S4 (a /22 subnet named by the router's address in a real capture), S7 (the same pattern in CLI examples) | Multi |
| F11 | `ntp-server`, `exclude`, `disable` (flag), `failover` nodes and their JSON shape | the issue text and Vyatta-family knowledge; S9 for `failover` | **Inferred** |
| F12 | `GET /api/edge/data.json?data=dhcp_leases` answers `{"success": "1", "output": {"dhcp-server-leases": {<network>: {<ip>: {expiration, pool, mac, client-hostname}} \| ""}}}` | S1, S2 (shape and example), S3 (typed class), S5, S6 (script reads these fields) | Multi |
| F13 | `expiration` format `YYYY/MM/DD HH:mm:ss`; its time zone | format: S2; zone: none | Single / **Inferred (UTC)** |
| F14 | `data=sys_info` answers `output.sw_ver` like `EdgeRouter.ER-e300.v2.0.9-beta.3.5296219.200507.1538` | S1, S2 (the same example), S5 (calls it) | Single (one example) |
| F15 | `data=dhcp_stats` | S1, S2, S3, S5 | Multi (**not used**) |
| F16 | `POST /api/edge/batch.json` with `SET` / `DELETE` (and `GET`) objects; answer members `SET`/`DELETE`, `COMMIT`, `SAVE`, `SESSION_ID`, `success`; status members `{"failure": "0", "success": "1"}` | S2 (both examples and the field tables), S1 (SET example), S3 (code posts it) | Single (S1 copies S2) |
| F17 | the batch commits **and saves** in the same request (COMMIT and SAVE appear in answers to a request with only SET or DELETE) | S1, S2 examples only; no prose | **Inferred** from the examples |
| F18 | a leaf is deleted with `null` as its value; a flag is set with `null`; a tag node is a nested object | S1/S2 examples (`"disable": null`, `{"custom-attribute": null}`) | Single |
| F19 | SET of a multi-valued leaf takes a JSON array | none (GET reports arrays, S4) | **Inferred** |
| F20 | a DELETE and a SET in one batch both apply; their order | S2 shows `SET`+`GET` and `DELETE` alone, never both write kinds | **Inferred** (the adapter avoids depending on order) |
| F21 | `GET /api/edge/config/save.json` downloads the configuration (temp file path, then `/files/config/`) | S1, S2 | Multi (**not used**; contradicts the issue text) |
| F22 | `operation/release-dhcp.json` takes `interface` and releases the router's own DHCP *client* lease | S2 | Single |
| F23 | account levels `admin`/`operator`; operator is read-only | levels: S2 (`auth.json`); read-only: none | Single / **Inferred** |
| F24 | status code of a CSRF or rights failure on a POST | none | **Inferred** (401/403 handled) |
| F25 | `use-dnsmasq`: ISC-only settings (failover, free-form parameters) are ignored; CLI status is unsupported; lease table content unknown | S7, S8 (summary) | Single / **Inferred** for the lease table |
| F26 | `failover { local-address, peer-address, name, status }` under a subnet | S9 (Vyatta family, summary) | **Inferred** |
| F27 | over SSH, `/opt/vyatta/bin/vyatta-op-cmd-wrapper <op command>` runs an operational command as an exec request | VyOS simulator (S16) only | **Inferred** for EdgeOS |
| F28 | `show configuration commands` prints one `set <path> '<value>'` line per leaf (values single-quoted, names not, flags without a value, multi-valued leaves repeated) | VyOS simulator (S16) only | **Inferred** for EdgeOS |
| F29 | `show version` prints `Version:` and `HW model:` lines | S15 (v1.10.x and a newer EdgeRouter 4; **not v2.x**) | Single / **Inferred** for 2.x and 3.x |
| F30 | `show dhcp leases` prints a table: address, hardware address, `YYYY/MM/DD HH:MM:SS` expiration, pool, client name; the pool column equals the shared-network name | none (the fields are those of F12 plus the pool; the column layout is a guess) | **Inferred** |
| F31 | `/bin/vbash -s` with `source /opt/vyatta/etc/functions/script-template` then `configure` gives a configure session with `set`, `delete`, `commit-confirm`, `confirm`, `save`, `discard` | VyOS simulator (S16) only | **Inferred** for EdgeOS |
| F32 | `commit-confirm <N>` asks "Proceed ? [Y/n]" on standard input, answered by `<<< y` | S10 (prompt text), VyOS simulator (S16) | Single / **Inferred** for EdgeOS |
| F33 | when the window ends without `confirm`, the router **reboots** into its saved configuration (it does not restore in place); default window 10 minutes | S10, S11, VyOS simulator (S16, text "before reboot") | Multi (VyOS, community guide) / **Inferred** for EdgeOS 2.x and 3.x |
| F34 | the commit-confirm timer is system-wide, not tied to the SSH session: closing the channel does not cancel it, and `confirm` works from another session | VyOS simulator (S16) only | **Inferred** for EdgeOS |
| F35 | `commit-confirm` leaves a background process on the channel, so the channel does not reach EOF; output is therefore read up to a marker, not to EOF | VyOS simulator (S16) only | **Inferred** for EdgeOS |
| F36 | a rejected `set`/`delete` or a failed commit has a non-zero exit status and/or the words "failed", "invalid", "not valid" or "error" in its output | none observed (the adapter checks both, and is conservative: either one is a failure) | **Inferred** |
| F37 | a commit that failed may still leave the reboot timer armed, and may have applied part of the change | S10 (one report) | Single / **Inferred** for EdgeOS |
| F38 | the login shell of the admin account is `vbash`, and exec requests are accepted for it | VyOS simulator (`$SHELL` is `/bin/vbash`, S16) | **Inferred** for EdgeOS |
| F39 | EdgeOS 3.0.0 and 3.0.1 keep the 2.x web API, login, CSRF token and commit-confirm unchanged | S12 to S14 mention a new UI and WireGuard and no API change; absence of evidence only | **Inferred**, unverified on hardware |
| F40 | the router's SSH server offers key-exchange, cipher and host-key algorithms the Go library accepts by default (not SHA-1-only `ssh-rsa`) | none | **Inferred** |

