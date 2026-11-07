# Network facts from Seshat

Lachesis can read **Seshat's Device Tracker** as an information source. Seshat learns where devices are on the
network (which switch and port, which VLAN, which vendor, which addresses ARP shows, when each was last seen).
Lachesis shows those facts next to the leases and reservations it already lists, checks a new reservation against
them, reports where the network and the DHCP server disagree, and offers two read-only tools to AI agents.

Seshat is **not a DHCP server** and this is not a vendor adapter. It is a small, read-only, optional "enrichment
source": Lachesis works the same without it, and when Seshat is down the facts show as stale or unavailable and
never turn a page into an error. A plan always builds. (Reaching a DHCP server *through* a Seshat edge is a
different feature: see [Reaching a server through a Seshat edge](../adapters/seshat-edge.md).)

## Set-up

**In Seshat** (0.5 or newer): an administrator registers Lachesis as an integration client (Settings, Integrations,
or `POST /api/integrations/clients`) with

- the capability **`tracker.read`** (single lookup, contract `seshat.tracker.lookup` v1) and **`tracker.bulk`**
  (batch lookup and prefix bindings, contracts `seshat.tracker.lookup` v1 and `seshat.tracker.bindings` v1). Without
  `tracker.bulk` Lachesis falls back to single lookups, a few at a time, and cannot build the drift report;
- a **scope** that names the sites (and tenants) Lachesis may see. An empty scope sees nothing;
- one token. Seshat shows the secret once.

**In Lachesis**, as an administrator: Settings, **Network facts**.

| Field | Meaning |
|---|---|
| Use it | Off keeps everything stored and stops Lachesis asking. |
| Address | Seshat's base address, `https://seshat.example.com`. https only. The certificate is verified against the system's trusted authorities (`SSL_CERT_FILE` and `SSL_CERT_DIR` add a private one); there is no switch to skip it. The address goes through the egress guard like every other connection. |
| Integration token | **Write-only.** Stored envelope-encrypted, never returned, logged or audited. Typing a new token and saving replaces it: that is how a token is **rotated** (Seshat lets a client hold two live tokens during a roll-over, so there is no gap). Remove token deletes it. The token is **bound to the host and port it was saved for**: it is used only while the address still names that endpoint. A changed address (another host or port) needs the token typed again in the same save, and clearing the address deletes the stored token, so it can never be sent to an address it was not saved for, whatever order the changes arrive in. |
| Days before a lease counts as unseen | Used by the drift report. Default 14. |
| Which site each server belongs to | The mapping, below. |

Saving checks the connection at once and shows the result. **Test the connection** runs the same check.

### Mapping servers and scopes to sites

A row maps a Lachesis **server** (leave the scope network, `subnet` in the API, empty) or one **scope** of it (give its
network in `subnet`, `192.0.2.0/24`) to the name of a Seshat **site**, and optionally to one **VRF** of that site (`vrf`).
A scope row wins over the row for its whole server.

Seshat's bindings carry a VRF, and two VRFs of one site may use the same addresses. With a VRF on the row, Lachesis asks
Seshat for that VRF's bindings only (`vrf=` on the bindings call) and ignores a device's addresses in other VRFs. With
none, every VRF of the site is read together, which is only right when the site has one VRF or its VRFs do not overlap.

**A scope with no mapping gets no facts, and Lachesis asks Seshat nothing for it.** There is no default site. This is
what keeps overlapping address space apart: two sites that both use `192.0.2.0/24` are two different mappings, bindings
are read per site, and a device that is currently at another site contributes nothing to this site's view (the row says
"Also seen at" the other site instead).

Seshat has no tenant parameter on these calls: tenant limits are part of the token's scope inside Seshat. Lachesis maps
to sites only.

## Contracts: what Lachesis pins

At start-up, on every change of the settings, every five minutes (every minute while degraded) and on **Test**,
Lachesis reads `GET /api/integrations/contracts` and requires:

- `seshat.tracker.lookup` serving **version 1**. Anything else is a **mismatch**: Lachesis shows "Version not
  supported", names the version Seshat serves, uses no facts and sends no lookups until it is fixed. It never guesses
  at another shape.
- `seshat.tracker.bindings` serving version 1. If not, facts per device still work and only the prefix reads (ARP
  state in views, the drift report) are off, and the status says so.

Contracts and keys Lachesis does not know are ignored (additive changes keep the version). Every response also carries
its own contract stamp, which is checked again. A Seshat older than 0.5 has no contracts list and is reported as a
mismatch too.

## What you see

### Lease and reservation views

Scope screen, Leases and Reservations tabs. Two columns are added when a source is connected and the scope is mapped:

- **On the network**: the switch, port and VLAN the device is attached to, and the vendor. Each fact shows its age and
  how it was learned (`2 min ago via trap`), and a `stale` mark when Seshat itself says the fact is too old or when the
  value comes from Lachesis's cache after Seshat could not be asked. A device that Seshat only knows at another site says
  so. A device Seshat does not know says so.
- **ARP and last seen**: whether the address is in ARP with this hardware address (`In ARP`), with a different one
  (`Another device`, and the address), or not in ARP, and when the device was last seen.

The columns are loaded after the list, in batches of 100, so a slow or absent Seshat never delays the list. A sentence
above the table says when the facts are stale, partial or unavailable, and why. Only addresses the DHCP server itself
lists in the scope are looked up, so the facts route cannot be used to have Lachesis look arbitrary addresses up.

### Plan validation

When a **reservation** plan is built (not when it is applied), Lachesis adds to the plan's list of warnings:

- **Warning**: the target address was seen at this site in the last 24 hours with a different hardware address, saying
  whether that device holds an active lease on it or has no lease or reservation (a static or unknown host).
- **Info** (listed with the prefix `Info: `): the reservation's hardware address is also seen at another site. It says that, never where: a plan author could otherwise ask, with any hardware address, at which sites Seshat has seen it.
- **Info**: the check was skipped because Seshat could not be asked, and the plan was built without it.

Advice is capped at five lines and added **after** the engine's own warnings (and after their cut at twenty), so it can only add to a plan. These findings **never block a plan, never change its state hash and never need a reply**: a plan builds identically
with no Seshat, a slow one or a failing one (the advisor has a four second budget and cannot fail the build). Applying
a plan asks Seshat nothing.

### Drift report

Drift page, section **What the network sees**. Each mapped scope has a **Check** button (a report is made on request,
because every lookup is audited by Seshat). The report lists

1. addresses the network sees that have **no lease and no reservation** (static or unknown hosts);
2. addresses seen with a **different hardware address** than the lease or reservation there;
3. active leases whose hardware address has **not been seen at this site for N days** (the setting), or never.

It says what it could not compare (a server that does not report leases, a list cut at its size limit, more than 1000
active leases) and, when Seshat is down, builds from the last read marked stale.

### MCP tools

Both are read-only, need only the token's `read` scope, and are listed only while a source is connected. Hostnames and
the names of switches and vendors are untrusted data and the tool descriptions say so.

| Tool | Needs | What it does |
|---|---|---|
| `lookup_device` | `scopes.read` (`leases.read`, `reservations.read` optional) | Given `server_id`, `scope_id` and a MAC or IP: the facts above for each matching lease or reservation, each with age and source. A MAC that is not in this scope is asked on its own **only if** a lease or reservation of a server the token may use lists it at the same site; otherwise it is refused (`not_found`), so the token's server list cannot be bypassed. A MAC asked on its own has no ARP state (there is no address to compare). |
| `scope_arp_conflicts` | `scopes.read`, `reservations.read` (`leases.read` optional) | The three drift lists for one scope, with totals and a `truncated` flag. Shares the analysis slots with the other heavy tools. |

Both answer with a `state` (`ok`, `stale`, `partial`, `unavailable`, `unmapped`, `off`) and a `message`; a down Seshat is a
state, not a failure.

## When Seshat is not fine

| State | Cause | What Lachesis does |
|---|---|---|
| `unreachable` | Seshat timed out, refused the connection or answered 5xx | Shows the cached facts marked stale ("the facts shown are the last ones Lachesis read") for up to 30 minutes; with nothing cached it says only that Seshat cannot be reached. Checks again every minute. |
| `token_refused` | 401, or the client is switched off or expired (`client_inactive`, `token_not_live`) | Says the token was refused and to replace it; stops sending it until the next check. |
| `capability_missing` | 403 `capability_missing` on **every** call (or a bare 403 from a legacy token) | Names `tracker.read` and `tracker.bulk`. |
| bindings refused | 403 `capability_missing` on bindings (and on the batch) from a token with only `tracker.read` | Turns off those two features only, for ten minutes at a time: device facts keep working one lookup at a time, ARP state comes from each device's own address list, the connection stays "Connected", and the facts say "partial" with the reason. The drift report cannot list addresses. |
| out of scope | 403 or 404 `out_of_scope` for the mapped site | Says the site is outside the client's scope; the connection itself is shown as fine. The drift report says it could not check and lists no lease as unseen: a site the client may not read is not a site where nothing was seen. |
| `contract_mismatch` | `contract_version_unsupported`, or another contract version | See above: nothing is used. |
| `rate_limited` | 429 `limit_exceeded` | Honours `Retry-After` (a second to ten minutes, thirty seconds when missing): no request is sent until then, cached facts cover the wait. |

Messages are fixed wording. **Seshat's own message text is never repeated** to a person, in a log or in an audit row, and
neither is the token.

## Cost and rate limits

Every Seshat lookup is audited on Seshat's side, so Lachesis keeps them few:

- Facts are cached **60 seconds**; after that they are read again, and for up to **30 minutes** they can still be served,
  marked stale, if Seshat cannot be asked. The cache is keyed by site, so two sites never share an entry.
- Lookups are **batched**: one `tracker.bulk` batch call carries up to 200 hardware addresses (Seshat allows 500), one
  binding read covers a whole scope (paged, at most 10 pages of 2000).
- Calls are **paced**: at most 30 a minute with a burst of 8, half of Seshat's default for `tracker.bulk`. Pacing charges
  the **real HTTP requests** a call made (every binding page, every single lookup), not one per call, and may go into debt
  that later calls wait out. A paced answer says so and the rest is read on the next refresh.
- Each **account or token** has its own budget on top (30 requests a minute, burst 15; a drift report or an MCP tool costs
  5), for the facts route, the reports and the MCP tools, so one viewer or token cannot spend everyone's share of Seshat
  (429 with `Retry-After`).
- What is kept of an answer is bounded: at most 50 addresses and 200 history intervals per device, 2000 bindings a page,
  8 MiB a response, and a cache bounded by an estimate of its memory (64 MiB) as well as its entries.
- A report is made on request and looks up at most 1000 leases.
- A 429 stops all calls until `Retry-After` has passed.

Events (`events.subscribe`) are **not used**: polling with this cache is enough for what Lachesis shows.

## API

Reads need the `read` scope; changes are administrators in a browser session only (a token, and so MCP, gets 403).

| Route | Who | |
|---|---|---|
| `GET /api/enrichment` | any role | The connection state and sentence. |
| `GET /api/enrichment/config` | administrator | The stored settings, mappings and whether a token is set. Never the token. |
| `PUT /api/enrichment/config` | administrator, session | `enabled`, `base_url`, `unseen_days`, `token` (write-only), `mappings`. A field left out is unchanged. 422 with one problem per field; nothing is written on a problem. |
| `DELETE /api/enrichment/config/token` | administrator, session | Removes the stored token. |
| `POST /api/enrichment/test` | administrator, session | Checks the connection and contracts now (rate limited). |
| `GET /api/servers/{id}/scopes/{scope}/facts?ip=...` | read | Facts for up to 500 addresses of the scope. |
| `GET /api/network-drift` | read | The scopes that have a site mapping. |
| `GET /api/network-drift/{id}/{scope}` | read | One scope's drift report. |

A token limited to some servers is refused (403) on the others, as everywhere. Text from Seshat (names, sources) is cleaned of
control and format characters (text-direction overrides included) and cut, on every path. Every change is audited
(`enrichment.update`, `enrichment.map` with what was added and removed, `secret.set`, `secret.delete`; and `enrichment.test` for each connection test) in one transaction with the change, with the
state of the token in words and never a value.
