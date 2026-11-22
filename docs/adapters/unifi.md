# UniFi Network adapter

The UniFi Network application: a self-hosted Network Server, or the Network application of a UniFi OS console.
Vendor id `unifi`. IPv4 only.

**Validated:** a self-hosted Network Server with a local account (UniFi Network 10.6.109), and the **UniFi OS console mode**
on **UniFi OS Server 5.1.42 with Network 10.5.67**: console login with its anti-forgery token, an API key, the owner
account and a View Only account. **Not validated** (the test console has no adopted device and no gateway): a gateway that serves
DHCP and honours a fixed IP, a hardware console (UDM, UCG, Cloud Key; the software is the same family, the box was not tested),
roles other than the owner and View Only, and other versions. The official integration API is probed but not used.

A UniFi **network** is the scope. Its DHCP range is the pool, its DHCP settings are scope options, the known clients
with a fixed IP are the reservations and the clients the controller currently sees are the leases. One Lachesis
server manages **one site**; add one server per site.

## Which API, and why

The controller offers two API surfaces. The adapter uses the classic one.

| | Classic API (`/api/s/<site>/rest\|stat/...`) | Official integration API (`/integration/v1/...`) |
|---|---|---|
| Authentication | Local account: `POST /api/login`, session cookie (a UniFi OS console: `POST /api/auth/login`, `TOKEN` cookie plus `X-Csrf-Token`). An API key (`X-API-KEY`) also works on a console. | `X-API-KEY` only (a console login session is refused: 401 `api.authentication.missing-credentials`) |
| Can it be used on a self-hosted Network Server? | Yes, tested | Present, but no key can be created there |
| Networks, DHCP range and options | read and write (`rest/networkconf`), tested on both | `GET .../networks[/<id>]` reads range, gateway, DNS, lease time and domain; **writes were refused** on the test console (`api.network.validation.gateway-not-found-on-site`: no gateway adopted) |
| Fixed IPs | read and write (`rest/user`), tested on both | **none** (no reservation endpoint; `404 No endpoint`) |
| Clients and leases | `stat/sta` (connected, synthetic in the fixtures), `rest/user` (known) | read-only list of connected clients; no leases |

Verified on **UniFi Network 10.6.109** (a self-hosted Network Server in a container, no devices adopted):

- `/integration/v1/info` answers 403 `api.forbidden` without a key, and also with a made-up key. The server has the surface
  but **cannot mint a key**: Settings > System has General and Backups only, there is no Control Plane or Integrations
  page, and no cookie-authenticated call creates one. Keys are a UniFi OS console feature.
- The classic API reads and writes everything Lachesis manages, with a local account.

Verified on **UniFi OS Server 5.1.42 / Network 10.5.67** (a console in a VM, no devices, no gateway, local owner account only):

- An API key authenticates the classic paths: `GET /proxy/network/api/s/<site>/rest/networkconf` with `X-API-KEY` answers 200, and
  writes (`PUT rest/networkconf/<id>`, `POST rest/user`, `PUT rest/user/<id>`, the `sitemgr` command) work the same, with **no CSRF token**. The
  unprefixed `/api/s/<site>/...` is a 404 on a console: the `/proxy/network` prefix is required.
- The integration API (`/proxy/network/integration/v1`) answers `GET /info` (`{"applicationVersion":"10.5.67"}`), `/sites`, `/sites/<id>/networks`,
  `/networks/<id>`, `/clients`, `/devices`, `/wifi/broadcasts`, `/dns/policies`. A network carries `ipv4Configuration.dhcpConfiguration`
  (`mode` `SERVER`, `ipAddressRange` start/stop, `gatewayIpAddressOverride`, `dnsServerIpAddressesOverride`, `leaseTimeSeconds`, `domainName`,
  `pingConflictDetectionEnabled`) but no NTP, WINS, TFTP or WPAD option. There is **no fixed-IP or lease endpoint**. `PUT` of a network was refused
  here twice: first `api.network.validation.ipv4.subnet-not-private` ("Auto-scale requires private subnet": a documentation range), then
  `api.network.validation.gateway-not-found-on-site` with auto-scale off; whether it writes with an adopted gateway is **not validated**.
  Without a key or with a wrong key: 401 `{"error":{"code":401,"message":"Unauthorized"}}`. So the integration API covers a subset of
  the DHCP reads (range, gateway, DNS, lease time, domain) and none of the fixed IPs, and Lachesis keeps using the classic paths for everything.

Probe reports the surface it found (`Details["api_surface"]`, `Details["auth"]` on a console: `console login` or `API key`, and
`Details["integration_api"]`: `present (needs an API key; not used: no fixed IPs or leases)` or `absent`, found by asking without credentials).
A UniFi OS console is selected with the controller type `UniFi OS` (`platform: unifi_os`).

## Credentials (least privilege)

| Secret | Use |
|---|---|
| `username` + `password` | A local controller account. The adapter logs in when the session starts and logs out when it closes. |
| `api_key` | A UniFi OS console API key (Network > Integrations > Create New API Key), sent as `X-API-KEY`. |

Give Lachesis its own **local** account: no UI/cloud sign-in, MFA off (a cookie login cannot answer a second factor), no
remote access. Role: Administrator on the one site. **Prefer an API key on a UniFi OS console** (below: the console limits logins).

**An API key has the permissions of the account that created it.** On UniFi OS Server there is no read-only key: the limited
administrator cannot create one (`CODE_UNAUTHORIZED`) and the owner cannot create one for it (`CODE_OPERATION_FORBIDDEN`). A key
created by the owner can write everything, so keep it as secret as the owner's password; delete it with the console when it is no longer needed.

**Roles on UniFi OS Server 5.1.42** (Admins & Users > Create New > Admin > untick "Use a Predefined Role"): the Network application's
selector offers Full Management, Hotspot Operator, **View Only**, **Site Admin** and None; the user-management selector Full Management, View Only
and None. The console stores a custom role with `network.management: ["readonly"]` (View Only) or `["s-<site id>-admin"]` (Site Admin).
What the Network application then reports for the account on the site (`GET /proxy/network/api/self/sites`, field `role`):

- owner (and an owner's API key): `admin`, everything works;
- **View Only: `readonly`**, reads work (`rest/networkconf`, `rest/user`, `stat/*`) and every write is refused by the console itself with
  **HTTP 403** `{"meta":{"rc":"error","msg":"api.err.NoPermission"},"data":[]}`. Probe sees `readonly` and marks `reservations.write`, `pools.write`
  and `options.write` unavailable with the reason; a write sent anyway (a stale stored capability) is `ErrAuthFailed` and changes nothing. Both tested.
- Site Admin and Hotspot Operator could not be checked: **a console login only worked for one of the four limited accounts created**. The account
  with Network View Only and user management **View Only** signed in; accounts with user management None, Full Management or a Site Admin
  network role were refused at login with `403 AUTHENTICATION_FAILED_ACCOUNT_NOT_EXIST_LOCALLY` ("You do not have the permission to access this
  device"), although they were created as local accounts and listed as active. The reason is not known. The adapter reports that answer as
  `ErrAuthFailed` with a hint (use a local account that may sign in, or an API key). The role names `readonly` (seen), `nonadmin` and `hotspot`
  (documented, not seen) are mapped; only `admin` and `readonly` were observed.

Sessions and logins:

- A wrong password is `ErrAuthFailed` (a Network Server answers HTTP 400 `api.err.Invalid`; **a UniFi OS console answers HTTP 403
  `{"code":"AUTHENTICATION_FAILED_INVALID_CREDENTIALS","message":"Invalid username or password","level":"debug"}`**). Too many logins is
  `ErrBusy` with the controller's `Retry-After` (seconds; default 60; clamped to 300). **A UniFi OS console limits logins**: on UniFi OS Server 5.1.42
  four logins in quick succession were accepted and the fifth was answered **429 `AUTHENTICATION_FAILED_LIMIT_REACHED`** ("You've reached the login
  attempt limit") with **no `Retry-After`**, and the limit also counts logins that succeeded; it cleared after roughly a minute without attempts
  (polling every 10 seconds kept it shut). Lachesis opens a session, and so logs in, for each operation (a probe, a read, each step of a plan), so a busy
  console login runs into it: use an API key. The first `AuthFailed` or `Busy` login is latched
  for the rest of the session: every later call returns it without contacting the controller, so a wrong password costs one failed
  login, not one per call.
- A session logs in at most **twice** (the first login and one renewal after the controller drops the session; a 401 means the
  request was not processed, so it is retried once). After that the session is `AuthFailed`.
- **Console login and CSRF (observed):** `POST /api/auth/login` with `{"username","password","remember"}` (the console's own UI sends
  `{"username","password","token":"","rememberMe":false}`; both work) answers 200 with the user record, a `TOKEN` cookie (a JWT, `Secure; HttpOnly;
  SameSite=None; Partitioned`, **two hours**; the header `x-token-expire-time` is the expiry in milliseconds) and the anti-forgery token in the
  **response header `X-Csrf-Token`** (there is no `csrf_token` cookie). `X-Updated-Csrf-Token` was not seen. Writes with the cookie go with
  `X-Csrf-Token: <token>`; a **wrong** token is the console's own `HTTP 403 {"error":{"code":403,"message":"Forbidden"}}` (no Network envelope);
  a write with **no** token header was accepted by this build (it reached validation). The adapter always sends the token, and treats that 403
  like a 401: one renewal login, then `ErrAuthFailed`. Reads need no token.
- **Session expiry and errors:** a missing, bogus, logged-out or expired session, and a missing or wrong API key, all answer
  `HTTP 401 {"error":{"code":401,"message":"Unauthorized"}}` (not the Network `api.err.LoginRequired` envelope of a Network Server); after
  `POST /api/auth/logout` (200 `{"success":true}`) the same cookie is a 401. A site the Network application does not know is
  **401** `api.err.NoSiteContext` (with a Network envelope), which the adapter reports as `ErrInvalidInput`. Validation errors are the Network
  Server's: 400 `api.err.InvalidPayload` with the failing field.
- Only the cookies `unifises`, `TOKEN` and `csrf_token` are kept, at most 4, each at most 4 KiB; the CSRF token (header or cookie) is kept only
  if it is printable ASCII of at most 512 characters. Cookies and the key go only to the configured base URL; redirects are not followed.
- A refusal to a logged-in account (no permission) is `ErrAuthFailed`.

Credentials, cookies, CSRF tokens and API keys live inside the session and are dropped on `Close`; they never reach a log,
an error, `Native` or the audit trail. Text from the controller is never echoed: only its short error code
(`api.err.InvalidDHCPRange`) is shown.

## TLS

The endpoint must be `https://`; credentials go out on every request. A UniFi controller presents a **self-signed
certificate**. Verification stays on (system roots by default). Choose one of:

- **Pinned certificate (recommended)**: server TLS mode `pinned`, with the leaf certificate's SHA-256, for example from
  `openssl s_client -connect unifi.example.com:8443 </dev/null 2>/dev/null | openssl x509 -noout -fingerprint -sha256`
  (64 hex digits, colons optional). Renewing the certificate changes the pin.
- **Custom CA**: TLS mode `ca` with the PEM of the CA that issued a certificate you replaced the controller's with.

Skipping verification is the per-server, audited, permanently badged override of certificate checking; do not use it for a controller.

## Connection settings

| Field | Default | Meaning |
|---|---|---|
| `site` | `default` | The site's name (`default`, or the short id the controller gave a site added later) **or** its description as the UI shows it. A description that two sites share is refused. |
| `platform` | `controller` | `controller`: a Network Server, `/api/login`, paths at the root. `unifi_os`: a console, `/api/auth/login` or an API key, paths under `/proxy/network` (validated on UniFi OS Server 5.1.42; a gateway and a hardware console are not). |
| `max_items` | 200000 | The most clients or networks one read accepts; one answer may not exceed 32 MiB. |

## Probe

Probe asks `/integration/v1/info` and `/status` without credentials (surface and version; on a console both answer 401, so the
version comes from the site's `stat/sysinfo` after the login), logs in, resolves the site, and
reads `stat/device-basic`. It reports the version (`10.6.109`, `10.5.67`), `Details`: `platform`, `site` (the internal name), `api_surface`,
`auth` (a console: `console login` or `API key`), `integration_api`, `gateways`; and warns when **no UniFi gateway (USG, UDM, UXG) is adopted** on the site: the controller stores
DHCP settings but a third-party router then serves the addresses, so changes made here may not reach any client. A site that
does not exist, or that two sites claim, is a probe that answers but is unusable (`OK=false`, every capability unavailable with
the reason).

Warnings per mode (what a probe says so the operator is not misled):

| Mode | Warnings |
|---|---|
| Network Server, local account | the gateway warning, when it applies |
| UniFi OS, API key or console login | "validated on UniFi OS Server 5.1.42 (Network 10.5.67) ...; a UniFi gateway or hardware console, other roles (for example Site Admin) and other versions are not validated", and the gateway warning |
| UniFi OS, console login only | also: "a UniFi OS console limits logins (about four a minute; the fifth is answered 429) ... use an API key" |
| a read-only account (any mode) | the write capabilities are unavailable, reason "the account's role on this site (readonly) cannot change settings" |

Capabilities do not differ between the modes: all of `scopes.read`, `leases.read`, `reservations.read`, `reservations.write`, `pools.write`
and `options.write` are `classic` for a writable account, and only the account's site role (`readonly`, `nonadmin`, `hotspot`) takes the
writes away. They are **not** switched off for lack of a gateway: the console stores and verifies the settings either way, and the warning
says whether anything serves them.

## Capabilities

| Capability | Status | Notes |
|---|---|---|
| scopes.read | yes | Networks with an IPv4 subnet and purpose `corporate` or `guest`. WAN, VPN and VLAN-only networks are not scopes. The scope id is the controller's network id. `Disabled` is DHCP off. |
| leases.read | yes | `stat/sta`: connected clients with an address. The controller reports no lease times, so `Starts` and `Ends` are zero and the state is `active`. A client is placed on the network it names, else by subnet; one outside every scope is dropped. |
| reservations.read | yes | Known clients (`rest/user`) with `use_fixedip`. Host name is the client's alias (`name`), description its note. A fixed address outside its network, or the network or broadcast address, is not listed (DHCP cannot serve it). A record whose `_id` is not a 24-hex object id is never used (below). |
| reservations.write | yes | Create, update, delete (below). |
| pools.write | yes | Create (a network with no range) and update. A network has exactly one range: a second create is `ErrConflict`, a delete is `ErrInvalidInput` (turn DHCP off in the controller instead). No excluded ranges. A range that would include an address fixed for a client (that the current range does not already cover) is refused. |
| options.write | yes | Scope-level options 3 (router), 6 (DNS, up to 4), 15 (domain name), 42 (NTP, IPv4 only, up to 2), 44 (WINS, 1), 51 (lease time, 60 to 31536000 s; cannot be removed), 66 (TFTP server), 252 (WPAD URL). Router must be an address inside the network (not its network or broadcast address). Router, DNS, NTP and WINS addresses must be ordinary unicast: never the limited broadcast, link-local (including the 169.254.169.254 metadata address), loopback, multicast, "this network" (0/8), class E, or 100.100.100.200. WPAD URLs use an RFC 3986 allow-list and may not contain a double quote, apostrophe, backslash, semicolon, backtick, dollar sign, curly brace, angle bracket or whitespace. Any other code is `ErrInvalidInput`; the controller silently drops keys it does not know. |
| leases.write | no | The controller cannot end a client's lease. |
| failover | no | A site has one DHCP server (its gateway). |

### Reservations

The controller keys a fixed IP on the MAC (a client id that embeds one, `01:` + MAC, works) and keeps it on the known-client
record, so the three kinds work on that record:

- **create**: a client the controller has never seen gets a new record; a known client without a fixed address gets its record
  updated (a second record for the same MAC is refused by the controller); a client that already has a fixed address, or an
  address another client holds, is `ErrConflict`.
- **update** keeps the address and the network. Changing the client frees the address first, then fixes it for the new one.
- **delete** sets `use_fixedip` off and clears the address; the record and its alias stay (`forget-sta` would erase the client).

The controller **does not validate** a fixed IP: it accepts an address of another subnet, the gateway's own address, an address
two clients share. The adapter checks all of that (inside the network's subnet, not the network or broadcast address, not the
controller's address on the network or the router it hands out, not held by another client, one address per client) before
anything is sent, and Preflight reports it at plan time. A **new** fixed address inside the network's DHCP range is refused (the controller
could lease it to another client first; an existing reservation that is already inside the range can still be edited). No reservation
options: UniFi has none per client.

**Names and notes.** The alias (host name) and note stay free text (spaces, accents and ordinary punctuation are fine) but may not
contain control, format (bidi, zero-width) or invisible characters, nor a double quote, apostrophe, backslash, semicolon, backtick, dollar sign, curly brace or angle bracket (they break out of quoting where
names are later shown or exported), and are capped at 128 and 256 characters. They must also have no leading, trailing or repeated
spaces, and no space other than a plain one: reading collapses runs of spaces, so such a value could never be seen again exactly as
written and its verification would stay unconfirmed forever. This is refused rather than silently normalised, because the plan
engine compares what it asked for with what the re-read shows. A value that the record already holds is not judged again, so a legacy
alias with an apostrophe survives an edit of the note.

**Change of client.** Changing the client of an address is two writes (free the old client, fix the new one). The new client's side is validated first, and a legacy alias or note is carried only by the client that holds it. If the second write fails, the adapter re-reads and puts the old binding back only if nobody holds the address now and the old client's record is exactly as the adapter left it (freed, same name, note and network). Otherwise it restores nothing, because the snapshot would overwrite somebody else's change or create a second holder, and the result is unconfirmed with a note that the address may be free.

### DHCP off

A network with DHCP disabled **takes writes and stores them** (HTTP 200) but serves nothing. The adapter never switches DHCP on
by itself. Every plan touching such a network gets the warning `DHCP is disabled on network "..."` (from Preflight, once per
network, and again in the apply result), and the result of the write is **unconfirmed** with the note `stored but not in effect`: the
SDK's `Verified` means the change was observed to take effect, and here the controller holds it while nothing acts on it. The plan engine
shows such a write as unverified.

## Verified writes

The controller answers **200 for writes it discards** (an unknown key, a field it does not keep). After every write the adapter
re-reads the networks and clients and compares **exactly the fields it wrote**: for a fixed IP the client record found by MAC must hold
the written `use_fixedip`, `fixed_ip`, `network_id`, name and note (for a delete: `use_fixedip` off); for a range or an option every
written field of the network, the cleared ones included (a removed option must show its fields emptied, not only the `*_enabled` flag).
`Verified` only when the re-read shows all of it. Otherwise the adapter returns a nil error with `Verified=false` and a note, which the plan
engine reports as **unverified** (the SDK reserves `ErrUnverified` for the engine; the conformance suite refuses an adapter that returns it).

After a write has been sent, only a **clear refusal** is an error: HTTP 400 or 404 with a controller `api.err.*` code, or 401 or 403. Any other
outcome (a transport error, a 5xx, an answer that is not a controller reply, `rc` other than `ok` with HTTP 200, a 400 without a code) leaves
the result unknown, so the adapter goes straight to the re-read and answers from what it shows: confirmed if the change is there, unconfirmed if not. A write
that was already in place sends nothing and is confirmed; one that conflicts with the current state is `ErrConflict`; a change the controller
refuses is `ErrInvalidInput` with its error code. What is verified is the **controller's configuration**: the controller pushes it to the
gateway, and the adapter cannot see what the gateway then serves.

## Records the adapter cannot read

Networks, clients and connected clients are decoded one record at a time. A malformed record (a field of the wrong type) is skipped, and so is a
known client or network whose `_id` is not a 24-character lower-case hex object id (those ids go into request paths, where a crafted one such as
`../../rest/admin/...` could climb out of the collection behind a normalising proxy; every id is also path-escaped). Skipped records are counted
in the log (counts only) and Preflight adds a warning with the number. One bad record never fails a list.


## What is not validated

- **A gateway that serves DHCP** (a USG, UDM, UXG or Cloud Gateway): the test consoles have no devices, so whether the gateway applies a
  range, an option or a fixed IP inside the range, and what its leases look like, is not known. The connected client list is synthetic in the
  fixtures (the console has no clients). Probe warns when no gateway is adopted.
- A **hardware UniFi OS console** (UDM, UCG, Cloud Key Gen2+): the UniFi OS **Server** software was tested (5.1.42, Network 10.5.67);
  a hardware console runs the same console software but was not tested, and the Network version differs from the self-hosted 10.6.109.
- **Roles** other than the owner (`admin`) and View Only (`readonly`): Site Admin and Hotspot Operator accounts could not sign in on the test console
  (above), so what they report and may do is unknown.
- The **integration API**: reads of networks are known (above), but its writes were refused without a gateway and it has no fixed IPs, so it is not used.
- A **read-only API key** cannot be created on UniFi OS Server, so the key mode was exercised with an owner key only.
- Other Network versions: 10.6.109 (Network Server) and 10.5.67 (UniFi OS Server) were exercised. Field names are the classic API's
  long-standing ones; numbers the controller sends as strings (a VLAN id) are read both ways.
