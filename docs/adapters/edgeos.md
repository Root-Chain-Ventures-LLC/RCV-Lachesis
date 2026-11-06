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

S1 and S2 are not independent (S1 says it took a subset from S2). S3 is S2's author's client. S4, S5 and S6 are
independent of them and of each other.

## Transport and authentication

Everything goes over HTTPS to the router's web interface. The adapter accepts an `https://host[:port]` endpoint and
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
  A 1.x release, or a version string of another shape, adds a warning: the shapes are documented for 2.x.
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

