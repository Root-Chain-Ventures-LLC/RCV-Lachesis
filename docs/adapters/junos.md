# Juniper Junos adapter

The DHCP server of Juniper Junos devices: Junos (MX, EX, QFX, ACX, PTX), Junos Evolved and SRX. Vendor id `junos`, transport
NETCONF over SSH. IPv4 only.

> **Status: experimental, not validated on a real device.** No Junos image or device was available when this adapter was
> written. Every RPC name, XML shape and device behaviour below is taken from Juniper's documentation or inferred from it
> (sources S1 to S13, facts F1 to F24, each marked *documented*, *summary* or *inferred*), and every test fixture is **synthetic**
> (see "Fixtures and tests"). Probe says `EXPERIMENTAL` until the facts have been checked on a non-production
> device ("Validation steps for production"). A write the adapter cannot confirm is never reported as a success.

## What it does

| Capability | Junos source | Notes |
|---|---|---|
| scopes (read) | `access address-assignment pool <name> family inet` in the committed configuration, per routing instance | one scope per IPv4 pool; ranges, lease time and options 3, 6, 15 from `dhcp-attributes` |
| reservations (read) | `host` entries of the pool; legacy `static-binding` entries | |
| leases (read) | the DHCP local server's binding table (`show dhcp server binding detail`), once per routing instance | extended server only |
| reservations (write) | `host` entries, through the candidate configuration | extended server only; see "Writes" |
| pools, options, lease release, failover | not supported | not offered; the capability is absent |

Refused clearly, never guessed at: IPv6 / `inet6` pools and DHCPv6; the legacy DHCP server's leases and writes; a device with
both servers configured (writes); logical systems (their own DHCP configuration is not read); a reservation with a client id,
a description or per-host options (a Junos `host` entry has only a name, a hardware address and an address); changing the
address of an existing reservation (that is a delete and a create); rewriting or deleting a host entry that carries statements
Lachesis does not model. Each refusal says why, and nothing is sent to the device for it.

## Junos, Junos Evolved and SRX

| | Junos | Junos Evolved | SRX |
|---|---|---|---|
| Detected by | any other model | release string ends in `-EVO` (F18) | model starts with `srx` or `vsrx` (F18) |
| DHCP server | extended (jdhcpd) in current releases; legacy `system services dhcp` exists on older and some platforms (F1) | extended DHCP is documented for Junos; **which Evolved platforms and releases support the DHCP local server was not found** | both syntaxes exist; one secondary source says the legacy server is deprecated on SRX from 15.1X49-D60 and 17.3R1 (S13, to be checked against release notes) |
| Pools and server | pools external to the server, in the same logical system and routing instance (F3) | assumed the same | same |
| NETCONF | subsystem `netconf` (F11) | assumed the same | same |
| What the adapter does differently | nothing | probe warning only | probe warning only; a legacy configuration is read-only |

Nothing about Evolved or SRX behaviour was observed. The only difference the code acts on is the platform label and the
warning that goes with it. The device model is shown in the probe details.

## Connection

Endpoint: the management address, `host` or `host:port` (port **830** when absent, the NETCONF-over-SSH port; give `:22` if the
device serves NETCONF on the standard SSH port). The connection is made only through the guarded dialer the core hands to every
adapter, so it works unchanged behind an edge relay. Authentication is a password or an SSH key (with its passphrase when
encrypted). The SSH host key is **pinned** by SHA-256 fingerprint (`ssh_host_key`); with none set, the connection stops after
the key exchange, before any credential is sent, and the error shows the fingerprint to compare with the device. A key that
differs from the pin is a different, never auto-approved error.

NETCONF is implemented in the adapter (hello, framing, requests, `rpc-error`), without a third-party module. It offers base 1.0
and 1.1 and uses chunked framing only when the device offers 1.1 as well (F12). Replies are matched on local element names, so a
device with `rfc-compliant` (prefixed `nc:` tags, no default namespace) works. Every request has a deadline
(`rpc_timeout_seconds`) and every reply a size limit (16 MiB); a request that exceeds either closes the session. A lost SSH
connection is re-established through the same dialer for the next request.

## Probe

`get-software-information` gives the release and model (F9; older releases print the release only in the package comment, which
is also read). The DHCP configuration is read with a subtree filter that names only the DHCP hierarchies (F8), so the rest of the
configuration never leaves the device. Probe reports:

- **Version and platform**: release, `platform` (`junos`, `junos-evolved`, `srx`), `product_model`, NETCONF base (`1.0` or `1.1`).
- **Which server is in use** (`dhcp_backend`): `jdhcpd` (`system services dhcp-local-server`), `legacy` (`system services dhcp`),
  `both` or `none`. Routing instances are read too (F3).
- **Capabilities, honestly**: reads from the configuration; leases only with a `dhcp-local-server` group that has an interface
  (F5); writes only with the extended server alone, at least one IPv4 pool and the device advertising the NETCONF `candidate` and
  `confirmed-commit` capabilities. Every unavailable capability carries a reason.
- Warnings: experimental; pools not served (disabled); inet6 items not covered; entries not representable; logical systems not
  read; host entries with unmodelled statements; and that Lachesis does not detect or manage Junos redundancy (chassis cluster,
  dual Routing Engines, virtual chassis). Failover traits are not reported and the `failover` capability is absent: two separate
  devices serving the same clients are paired by the administrator in Lachesis, and a plan then writes both.

## Reads

Scope ids are `<routing instance>/<pool name>` (`master/LAN1`, `guest/GUEST`; `master` is the default instance) and
`legacy/<prefix>` for the legacy server. A pool whose routing instance has no DHCP local server group is shown as **disabled**.
Two pools with the same subnet cannot both be scopes (the SDK needs unique subnets per server): the first by scope id is kept and
the rest are counted in a probe warning.

Leases: one `get-dhcp-server-binding-information` request with `detail` per routing instance that has a DHCP local server group
(F15, F16). `BOUND`, `RENEWING` and `FORCERENEW` are active; `RELEASE` is released; `INIT`, `SELECTING` and `REQUESTING` are not
leases yet and are skipped. A binding is placed in the pool it names (`Client Pool Name`), else in the pool of the same routing
instance whose network holds the address; one that fits no pool is dropped. The table has no host name, so leases have none.
Times are read only when the device prints them in UTC (F17); otherwise the end is left unset and the raw text is kept in `Native`.

Reservations: a host entry named like a person's host name reads back as `Hostname`. A host entry written by Lachesis without a
host name is named `lch-<mac without colons>` and reads back with no host name.

## Writes

Reservation create, update and delete, one operation per call, on a NETCONF session of its own so the lock and the rollback timer
never outlive the call:

1. `lock` the candidate. A lock held by another session, or uncommitted edits of another session, is **Busy** (nothing written;
   retry later) (F13).
2. Read the committed configuration under the lock and plan the change: unknown pool, address outside the network or equal to its
   network or broadcast address, a collision with another host entry in the pool (address, hardware address or name) and a
   configuration that no longer matches what the plan was computed against are refused, with nothing loaded. A write that is
   already in the configuration is not written again.
3. `load-configuration` (merge, XML) of the host entries only: `replace` for a create or update, `delete` for a delete or a rename (F19).
4. `commit-configuration` with `check` (**commit-check**) (F20).
5. `commit` with `confirmed` and `confirm-timeout` of `commit_confirm_minutes` (default 5), sent in seconds as NETCONF requires (F21).
6. Read the committed configuration back and compare it with the expected end state using the SDK's reservation equality
   (same pool and address, same client, same host name, same description, same options).
7. Only if the read-back matches: the confirming `commit`, then a last read-back.
8. `discard-changes`, `unlock`, `close-session`.

Failure handling:

- **Before the commit confirmed** (lock refused, load error, commit-check failure, a refused commit, a session lost): nothing is
  live. The candidate is discarded and unlocked (a lost session drops it, F14) and the error says nothing was committed.
- **From the commit confirmed on**: the adapter never discards or retries. If the read-back fails or does not show the change, or
  the confirming commit is refused or its answer lost, the result is `Verified=false` with a note that the change was **not
  confirmed and the device rolls it back by itself when the window ends** (F21, F23), or, if the confirm was applied but the last
  read-back failed, that the device should be checked. A lost session during the commit confirmed is observed over a new
  session for the note only; the adapter does not know whether the device applied it and says so.
- A commit that fails with an `rpc-error` is treated as not applied (Junos commits atomically, F22), and the candidate is discarded.

One plan with several reservations is several commits confirmed in a row (Preflight says so). The adapter does not implement the
SDK's `Committer`: batching several operations into one commit would need the candidate to stay open between calls, which would
hold the lock and the rollback window across calls. Preflight (the SDK's `Preflighter`) runs all the checks of step 2 at plan time
against the configuration as it is then, without locking or writing.

## Minimum device configuration

NETCONF over SSH, a DHCP local server and a pool (documentation addresses):

    set system services netconf ssh
    set system services dhcp-local-server group lan interface ge-0/0/1.0
    set access address-assignment pool LAN1 family inet network 192.0.2.0/24
    set access address-assignment pool LAN1 family inet range r1 low 192.0.2.100 high 192.0.2.199
    set access address-assignment pool LAN1 family inet dhcp-attributes router 192.0.2.1
    set access address-assignment pool LAN1 family inet dhcp-attributes maximum-lease-time 86400

In a routing instance the server and the pool both go inside it (F3): `set routing-instances guest system services dhcp-local-server
group ...` and `set routing-instances guest access address-assignment pool ...`. Static host entries look like
`set access address-assignment pool LAN1 family inet host host10 hardware-address 02:00:5e:00:00:10 ip-address 192.0.2.10` (F6).

### Least-privilege login class

A dedicated user in a class that can use NETCONF, read the configuration and change only the DHCP hierarchies. Reading needs
`view` and `view-configuration`; writes add `configuration` and `commit`:

    set system login class lachesis permissions [ view view-configuration configuration commit ]
    set system login class lachesis allow-configuration "(system services dhcp-local-server|system services dhcp|access address-assignment|routing-instances)"
    set system login class lachesis deny-configuration "(system login|security|system services ssh)"
    set system login user lachesis class lachesis authentication ssh-ed25519 "ssh-ed25519 AAAA...public-key..."

`allow-configuration` and `deny-configuration` are regular expressions over configuration paths; the exact expressions that give a
reading and writing account on your release, and whether a restricted class sees only what it is allowed to, were **not** checked
here. Test them as in the validation steps. Without the `secret` permission a device shows secrets as masked text, and the adapter's
filter never asks for them. A read-only deployment can drop `configuration` and `commit`: probe then still works, and writes fail at
the lock or load with a clear permission error.

## Validation steps for production

Run these on a **non-production** device (or a lab pool) first. Until they pass, treat every write as experimental.

1. **RPC names.** For each command, run it with `| display xml rpc` and compare with the adapter's requests:
   `show version`, `show dhcp server binding detail`, `show dhcp server binding detail routing-instance <name>`. Expected:
   `get-software-information`, `get-dhcp-server-binding-information` (with `detail` and `routing-instance`) (F9, F15).
2. **Reply shapes.** Capture `show dhcp server binding detail | display xml` and
   `show configuration access address-assignment | display xml`; check the tags the adapter reads (`dhcp-binding`, `ip-address`,
   `hardware-address`, `state`, `lease-start`, `lease-expires`, `client-pool-name`; `pool/name`, `family/name`, `network`,
   `range/low/high`, `host/name/hardware-address/ip-address`) and the time format (F4, F17). Scrub the capture and replace the
   synthetic fixtures with it.
3. **NETCONF by hand.** `ssh -s -p 830 lachesis@<device> netconf`, read the hello: base 1.0 / 1.1, `candidate`, `confirmed-commit`
   (F11, F12). If `rfc-compliant` is set, confirm replies carry `nc:` prefixes and the adapter still reads them.
4. **Probe.** Register the device with its host key fingerprint; Probe must report the right version, platform, `dhcp_backend` and
   capabilities, and the warnings you expect.
5. **A write, end to end.** Create a reservation in a test pool. On the device, `show system commit` should show a confirmed commit
   followed by a normal one, `show configuration access address-assignment ...` the host entry, and `show system commit` / the commit
   log no leftover rollback. Update and delete it.
6. **The rollback.** With a 1 minute window, kill the SSH connection between the commit confirmed and the confirming commit (for
   example by blocking the adapter's address for a moment) and check what the device does when the window ends (F23: the documentation
   covers the deadline and `kill-session`, not other drops).
7. **Lock.** Hold `configure exclusive` from a second session and attempt a write: expect Busy and nothing changed. Make an
   uncommitted edit in a shared `configure` session and attempt a write: expect the same (F13).
8. **Login class.** Log in as the restricted user and confirm it can read the DHCP hierarchies and commit, and that it cannot
   change anything else.
9. **Evolved and SRX.** If you run them, repeat 1 to 5 on each platform and record which DHCP server they use (F18).

## Sources and facts

| Id | Source | Used for |
|---|---|---|
| S1 | Juniper, [DHCP legacy and extended](https://www.juniper.net/documentation/us/en/software/junos/dhcp/topics/topic-map/dhcp-legacy-and-extended.html) | the two syntaxes, their hierarchies, external pools, same routing instance (**summary**) |
| S2 | Juniper, [IP Address Assignment Pool](https://www.juniper.net/documentation/us/en/software/junos/dhcp/topics/topic-map/dhcp-ip-address-assignments.html) and [DHCP Server Options](https://www.juniper.net/documentation/us/en/software/junos/dhcp/topics/topic-map/dhcp-serever-options.html) | pools, static bindings, precedence (**summary**) |
| S3 | Juniper, [host (Address-Assignment Pools)](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/host-edit-access.html) | the `host` statement: `hardware-address`, `ip-address`, IPv4 only (**summary**) |
| S4 | Juniper, [address-assignment (Access)](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/access-edit-address-assignment.html) | where pools live (**summary**) |
| S5 | Juniper, [show dhcp server binding](https://www.juniper.net/documentation/us/en/software/junos/agf-user-guide/subscriber-mgmt-sessions/topics/ref/command/show-dhcp-server-binding-command.html) | options (`routing-instance`, `logical-system`, `detail`), output fields, states (**fetched**; its Syntax and Sample Output sections were empty) |
| S6 | Juniper, [show system services dhcp binding](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-services-dhcp-binding.html) | the legacy command exists (**title only**) |
| S7 | Juniper, [Commit the Candidate Configuration Only After Confirmation Using NETCONF](https://www.juniper.net/documentation/us/en/software/junos/netconf/topics/task/netconf-configuration-committing-with-confirmation.html) | `<commit><confirmed/><confirm-timeout>`, seconds, default 600, confirm by `<commit/>`, rollback, `kill-session` (**fetched**) |
| S8 | Juniper, [NETCONF Sessions: locking and unlocking](https://www.juniper.net/documentation/us/en/software/junos/netconf/topics/task/netconf-configuration-locking-unlocking.html) | `<lock>` on `<candidate/>`, the error names the holder, fails with uncommitted changes, unlock or session end discards (**fetched**; the XML samples were stripped) |
| S9 | Juniper, [Configure RFC-Compliant NETCONF Sessions](https://www.juniper.net/documentation/us/en/software/junos/netconf/topics/concept/netconf-session-rfc-compliant.html) | `]]>]]>` by default; chunked framing with `rfc-compliant` when both offer base 1.1; `nc:` prefixes (**summary**) |
| S10 | Juniper, [Supported Network Management Standards](https://www.juniper.net/documentation/us/en/software/junos/standards/topics/concept/network-management.html) | RFC 6241 supported; `confirmed-commit:1.0` advertised, 1.1 not; does not advertise NETCONF 1.1 at session start (**summary**; S9 and S10 differ on 1.1, so the client handles both) |
| S11 | Juniper, [`<commit-configuration>`](https://www.juniper.net/documentation/en_US/junos/topics/reference/tag-summary/junos-xml-protocol-commit-configuration.html) | the `check` form (**summary**) |
| S12 | Juniper, [DHCP Local Server](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/dhcp-local-server-for-routing-devices.html) | the extended server (**title only**) |
| S13 | [naoria45.github.io, SRX DHCP](https://naoria45.github.io/pages/srx/dhcp) | SRX legacy deprecation from 15.1X49-D60 / 17.3R1 (**secondary source, summary**) |

Evidence: **fetched** = the page text was read; **summary** = only a search-result summary of the page was seen; **title only** = the
page exists but its content was not read; **inferred** = not in any source seen, derived from Junos conventions or the facts around
it. Nothing here was observed on a device.

| Id | Fact | Evidence |
|---|---|---|
| F1 | Two DHCP servers: legacy (`system services dhcp`) and extended / jdhcpd (`system services dhcp-local-server`); new features only in the extended one | S1, summary |
| F2 | Legacy pools sit inside the server; extended pools are external, under `access address-assignment` | S1, summary |
| F3 | The extended server and the pools it uses must be in the same logical system and routing instance; both can be inside a routing instance | S1, S2, summary |
| F4 | The XML rendering of the configuration is a container per hierarchy level, a `<name>` child for a named statement, one element per statement or leaf | **Inferred** from Junos XML convention |
| F5 | A DHCP local server serves clients only through a `group` with at least one `interface` | **Inferred** |
| F6 | `host <name> { hardware-address <mac>; ip-address <ip>; }` under `pool <p> family inet`; IPv4 only; the address leaves the dynamic range; a static binding wins over a dynamic one | S2, S3, summary |
| F7 | Legacy: `system services dhcp pool <prefix>` with `static-binding <mac> { fixed-address <ip>; host-name }`; the XML of `address-range` (`low`, `high` as leaves) | static-binding: S1 summary; XML of address-range: **inferred** |
| F8 | `get-config` with `<source><running/></source>` returns the committed configuration; a subtree filter rooted at `<configuration>` selects hierarchies; the filter below asks for DHCP hierarchies, routing-instance names and logical-system names only | RFC 6241 (S10, summary); Junos filter form **inferred** |
| F9 | `get-software-information` answers `software-information` with `junos-version` and `product-model`; old releases carry `JUNOS Software Release [x]` in `package-information/comment` | **Inferred** |
| F10 | on a device with several Routing Engines or an SRX cluster the software information is wrapped per member (`multi-routing-engine-results`); the adapter reads the first version and model it finds | **Inferred** |
| F11 | NETCONF runs as the SSH subsystem `netconf` (`system services netconf ssh`), on port 830 by default | **Inferred** (well known, not in a fetched source) |
| F12 | Base 1.0 `]]>]]>` framing by default; chunked framing only with `rfc-compliant` when both sides offer base 1.1; with `rfc-compliant` replies use `nc:` prefixes and no default namespace | S9, summary; S10 differs on 1.1 |
| F13 | `<lock><target><candidate/></target></lock>` fails with an `rpc-error` that names the holding session, and also when the candidate has uncommitted changes | S8, fetched |
| F14 | Unlocking, or the session ending, before a commit discards the uncommitted changes; the candidate and committed configurations stay as they were | S8, fetched |
| F15 | `show dhcp server binding` takes `routing-instance`, `logical-system` and `detail`; its RPC is `get-dhcp-server-binding-information` | options: S5, fetched; RPC name: **inferred** (naming convention, to check with `\| display xml rpc`) |
| F16 | Without `routing-instance` the request covers the default instance only | **Inferred** |
| F17 | Detail output has Lease Start, Lease Expires, State (`BOUND`, `RENEWING`, `FORCERENEW`, `RELEASE`, `INIT`, `SELECTING`, `REQUESTING`) and Client Pool Name; the XML tags `dhcp-binding`, `ip-address`, `hardware-address`, `state`, `lease-start`, `lease-expires`, `client-pool-name` and the time format `YYYY-MM-DD HH:MM:SS UTC` | fields and states: S5, fetched; **tags and time format inferred** |
| F18 | Junos Evolved: release suffix `-EVO`; SRX: model `srx*` / `vsrx*`; DHCP local server support on Evolved platforms is not established | classification **inferred**; S1 |
| F19 | `load-configuration action="merge" format="xml"` with the `replace="replace"` and `delete="delete"` attributes on an element changes that element; a load error arrives as an `rpc-error` inside `load-configuration-results` and success as `load-success` | **Inferred** from Junos XML protocol convention |
| F20 | `<commit-configuration><check/></commit-configuration>` is the commit check; success is `commit-check-success` | S11, summary; the success marker **inferred** |
| F21 | `<commit><confirmed/><confirm-timeout>N</confirm-timeout></commit>` (N in seconds, 1 to 4294967295, default 600); an empty `<commit/>` before the deadline confirms; otherwise the device loads and commits the previous configuration; a failure is an `rpc-error`. (The CLI and the Junos XML protocol use minutes.) | S7, fetched |
| F22 | A commit is atomic: when it fails, nothing of it is live | **Inferred**; the adapter discards the candidate in that case anyway |
| F23 | The documentation says a pending confirmed commit rolls back at the deadline and when another application ends the session with `kill-session`; it says nothing about other session drops | S7, fetched |
| F24 | `<discard-changes/>` (RFC 6241) discards the candidate | S10, summary; **inferred** for Junos |

The subtree filter the adapter sends:

    <get-config><source><running/></source><filter type="subtree"><configuration>
      <system><services><dhcp-local-server/><dhcp/></services></system>
      <access><address-assignment/></access>
      <routing-instances><instance><name/><system><services><dhcp-local-server/></services></system><access><address-assignment/></access></instance></routing-instances>
      <logical-systems><name/></logical-systems>
    </configuration></filter></get-config>

## Fixtures and tests

Tests use a fake NETCONF device behind the shared fake SSH server (a real SSH handshake over loopback), driven by **synthetic** XML
(`testdata/`, each file labelled `SYNTHETIC`; documentation addresses and `02:00:5e` MACs only). The fake covers the hello with both
framings and `nc:` prefixes, RPC errors, a lock held by another session, a failed commit check, commit confirmed then confirm, a
failed read-back (left to the timer), a lost session at each step, a hung device, an oversized reply, and the SDK conformance suite.
They say nothing about what a device really prints: replace them with scrubbed recordings after step 2 of the validation steps.
