# UniFi Network adapter

The UniFi Network application: a self-hosted Network Server, or the Network application of a UniFi OS console.
Vendor id `unifi`. IPv4 only.

**Validated:** a self-hosted Network Server with a local account (UniFi Network 10.6.109). **Experimental, not validated on a real
console:** the UniFi OS console mode and API keys, and any role name other than `admin`. Validating the UniFi OS console mode is a
tracked follow-up; the official integration API is not used.

A UniFi **network** is the scope. Its DHCP range is the pool, its DHCP settings are scope options, the known clients
with a fixed IP are the reservations and the clients the controller currently sees are the leases. One Lachesis
server manages **one site**; add one server per site.

## Which API, and why

The controller offers two API surfaces. The adapter uses the classic one.

| | Classic API (`/api/s/<site>/rest\|stat/...`) | Official integration API (`/integration/v1/...`) |
|---|---|---|
| Authentication | Local account: `POST /api/login`, session cookie (a UniFi OS console: `POST /api/auth/login`, cookie plus `X-CSRF-Token`). An API key (`X-API-KEY`) is also sent on a UniFi OS console. | `X-API-KEY` only |
| Can it be used on a self-hosted Network Server? | Yes, tested | Present, but no key can be created: not tested |
| Networks, DHCP range and options | read and write (`rest/networkconf`), tested | not exercised |
| Fixed IPs | read and write (`rest/user`), tested | not exercised |
| Clients | `stat/sta` (connected, synthetic in the fixtures), `rest/user` (known) | not exercised |

Verified on UniFi Network **10.6.109** (a self-hosted Network Server in a container, no devices adopted):

- `/integration/v1/info` answers 403 `api.forbidden` without a key, and also with a made-up key. The server has the surface
  but **cannot mint a key**: Settings > System has General and Backups only, there is no Control Plane or Integrations
  page, and no cookie-authenticated call creates one. Keys are a UniFi OS console feature, so the integration API could
  not be exercised at all, and whether it covers fixed IPs and DHCP settings is not known from this work.
- The classic API reads and writes everything Lachesis manages, with a local account.

Probe reports the surface it found (`Details["api_surface"]`, and `Details["integration_api"]`: `present (needs an API
key; not used)` or `absent`, found by asking without credentials). **A UniFi OS console with an API key** is supported
through the classic paths under `/proxy/network` (set the controller type to UniFi OS and give an `api_key`); that mode is **experimental**; it,
and the console login with its CSRF token, follow the documented behaviour of those consoles and are covered by tests
against a stand-in, **not against a real console** (see "What is not validated").

## Credentials (least privilege)

| Secret | Use |
|---|---|
| `username` + `password` | A local controller account. The adapter logs in when the session starts and logs out when it closes. |
| `api_key` | A UniFi OS console API key (Settings > Control Plane > Integrations), sent as `X-API-KEY`. |

Give Lachesis its own **local** account: no UI/cloud sign-in, MFA off (a cookie login cannot answer a second factor), no
remote access. Role: Administrator on the one site (or a Site Administrator limited to it). A read-only role can read; Probe
sees the role the site reports and marks every write capability unavailable with the reason (Probe also warns whenever the UniFi OS or API-key mode is in use; the role names `readonly`,
`nonadmin` and `hotspot` are taken from the controller's documented roles and are **unvalidated**; only `admin` was seen).

Sessions and logins:

- A wrong password is `ErrAuthFailed` (a Network Server answers it with HTTP 400, a console with 401), too many logins is `ErrBusy`
  with the controller's `Retry-After` (seconds; default 60; clamped to 300). **The first `AuthFailed` or `Busy` login is latched
  for the rest of the session**: every later call returns it without contacting the controller, so a wrong password costs one failed
  login, not one per call.
- A session logs in at most **twice** (the first login and one renewal after the controller drops the session; a 401 means the
  request was not processed, so it is retried once). After that the session is `AuthFailed`.
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
| `platform` | `controller` | `controller`: a Network Server, `/api/login`, paths at the root. `unifi_os`: a console, `/api/auth/login` or an API key, paths under `/proxy/network` (**experimental, not validated on a real console**). |
| `max_items` | 200000 | The most clients or networks one read accepts; one answer may not exceed 32 MiB. |

## Probe

Probe asks `/integration/v1/info` and `/status` without credentials (surface and version), logs in, resolves the site, and
reads `stat/device-basic`. It reports the version (`10.6.109`), `Details`: `platform`, `site` (the internal name), `api_surface`,
`integration_api`, `gateways`; and warns when **no UniFi gateway (USG, UDM, UXG) is adopted** on the site: the controller stores
DHCP settings but a third-party router then serves the addresses, so changes made here may not reach any client. A site that
does not exist, or that two sites claim, is a probe that answers but is unusable (`OK=false`, every capability unavailable with
the reason).

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

- The official integration API (`/integration/v1`): the test controller cannot mint a key; not used.
- A UniFi OS console, the API-key mode and the console login/CSRF flow: **experimental**, implemented from the documented behaviour,
  tested only against a stand-in. Role names other than `admin` are likewise unvalidated. Validating them on a real console is a tracked
  follow-up; the integration API is not used.
- Behaviour with an adopted gateway: the test controller has no devices, so DHCP is not actually served and the connected
  client list is synthetic.
- Other Network versions: only 10.6.109 was exercised. Field names are the classic API's long-standing ones; numbers the
  controller sends as strings (a VLAN id) are read both ways.
