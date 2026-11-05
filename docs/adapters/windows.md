# Windows Server DHCP adapter

Windows Server DHCP (2012 R2 and later: any server with the `DhcpServer` PowerShell module). Vendor id `windows`.
IPv4 only (DHCPv6 scopes are not read or written).

## Transport and authentication

Lachesis connects to **WinRM over HTTPS** (default port 5986) and speaks the **PowerShell Remoting Protocol** (PSRP)
inside WS-Management SOAP, with **NTLM** authentication (Kerberos is a later addition). The endpoint on the server
form is the host name, with the port when it is not 5986: `dhcp.example.com` or `dhcp.example.com:5986`. Plain
`http://` (5985) is refused: the credentials never travel unprotected.

- Every connection goes through the egress guard of the core; the adapter has no dialer or HTTP client of its own.
  The server certificate is verified by default with the per-server TLS settings of the core: system roots, a
  custom CA, or a pinned leaf SHA-256 (the usual choice for a self-signed WinRM certificate).
- NTLM sends a challenge response, never the password, and carries the TLS channel binding. A server with
  `skip_verify` set (the audited, permanently badged override) can be impersonated by whoever sits on the path;
  keep verification on. No service principal name is sent.
- A request that may have reached the server is never sent twice: WS-Man Create and Command are not idempotent.
  A kept-alive connection the server closed while idle is detected before anything is written and replaced; if
  the connection drops after a request was sent the call fails, and the session is deleted over a new connection
  when its id is known. A Create whose reply was lost leaves a session the client cannot name; the server's idle
  timeout reclaims it.
- A rejected account is reported as an authentication failure; a refused connection, a failed handshake or a
  timeout as unreachable.

WinRM over HTTPS needs a listener with a certificate whose name matches the endpoint
(`winrm quickconfig -transport:https`, or a group policy).

### What runs on the server

Only **native `DhcpServer` cmdlets with typed parameters**. There is no script, no custom function and no module to
install on the host:

- The adapter runs a fixed list of cmdlets, each with one set of named parameters; the transport builds the
  pipeline `<cmdlet> <parameters> | ConvertTo-Json -Depth 1 -Compress` and sends every value as a typed CLIXML value
  (string, integer, boolean, list of strings). Nothing a caller supplies is ever parsed as PowerShell. Every call
  carries `-ErrorAction Stop`, so a failure is a failed pipeline.
- Each value is validated into its narrowest type before it is sent: an IPv4 address in canonical text, a client id
  as 1 to 64 hex octets, an RFC 1123 host name, option ids 1 to 254, bounded text without control, format or
  bidirectional characters and without characters outside the Basic Multilingual Plane.
- Output is decoded tolerantly: the module serializes the same logical value in more than one shape.
- Vendor text (host names, option names, error text) is cleaned (control and format characters replaced, secrets
  masked, capped) before it is stored, shown or returned.

| Cmdlet | Used for |
|---|---|
| `Get-DhcpServerVersion`, `Get-DhcpServerSetting` | Probe |
| `Get-DhcpServerv4Scope`, `Get-DhcpServerv4ExclusionRange`, `Get-DhcpServerv4OptionValue` | scopes, exclusions, scope / reservation / server options |
| `Get-DhcpServerv4Reservation` | reservations |
| `Get-DhcpServerv4Lease` | leases (a scope with `-AllLeases`, or one `-IPAddress`) |
| `Get-DhcpServerv4Failover` | relationships: mode, partner, state, scopes |
| `Add-` / `Set-` / `Remove-DhcpServerv4Reservation` | reservation writes |
| `Set-` / `Remove-DhcpServerv4OptionValue` | scope and reservation option writes |
| `Set-DhcpServerv4Scope`, `Add-` / `Remove-DhcpServerv4ExclusionRange` | scope range and exclusions |
| `Invoke-DhcpServerv4FailoverReplication` | push a scope to the failover partner |

## Credentials and least privilege

A secret `username` and `password` (both required): **a dedicated account for Lachesis**. Never a domain
administrator, never an account that anyone logs in with. Use `DOMAIN\name`, `name@domain` or a local name (a
workgroup server takes the bare local name).

### Recommended: a JEA endpoint

[Just Enough Administration](https://learn.microsoft.com/powershell/scripting/learn/remoting/jea/overview) exposes
only the cmdlets above to the account, as a virtual account that is a member of DHCP Administrators and nothing
else, in NoLanguage mode (no script blocks, no .NET). The account is mapped to the role in the endpoint's role
definitions and must be allowed to connect over WinRM (for example through Remote Management Users); it is **not** a
local administrator. Set the connection setting **PowerShell endpoint** (`configuration_name`) to the session
configuration name. Tested on Windows Server 2019 with an account that is not a local administrator: the WinRS
command shell is refused for such an account (HTTP 500), PSRP to the JEA endpoint works.

Role capability (`...\Modules\LachesisJea\RoleCapabilities\LachesisDhcp.psrc`):

    @{
        GUID = '5f7c7f2a-2d0e-4a52-9c3e-0d3f6f6c0b11'
        Author = 'Lachesis'
        ModulesToImport = @('DhcpServer')
        # Exactly the cmdlets the adapter runs, and ConvertTo-Json (every result is piped into it).
        VisibleCmdlets = @(
            'Get-DhcpServerVersion', 'Get-DhcpServerSetting', 'Get-DhcpServerv4Scope',
            'Get-DhcpServerv4ExclusionRange', 'Get-DhcpServerv4OptionValue', 'Get-DhcpServerv4Reservation',
            'Get-DhcpServerv4Lease', 'Get-DhcpServerv4Failover',
            'Add-DhcpServerv4Reservation', 'Set-DhcpServerv4Reservation', 'Remove-DhcpServerv4Reservation',
            'Set-DhcpServerv4OptionValue', 'Remove-DhcpServerv4OptionValue', 'Set-DhcpServerv4Scope',
            'Add-DhcpServerv4ExclusionRange', 'Remove-DhcpServerv4ExclusionRange',
            'Invoke-DhcpServerv4FailoverReplication',
            'ConvertTo-Json'
        )
        VisibleFunctions = @()
    }

Session configuration and registration:

    New-PSSessionConfigurationFile -Path .\Lachesis.DHCP.pssc -SessionType RestrictedRemoteServer `
        -LanguageMode NoLanguage -RunAsVirtualAccount -RunAsVirtualAccountGroups 'DHCP Administrators' `
        -TranscriptDirectory 'C:\JeaTranscripts' `
        -RoleDefinitions @{ 'DOMAIN\lachesis-svc' = @{ RoleCapabilities = 'LachesisDhcp' } }
    Register-PSSessionConfiguration -Name Lachesis.DHCP -Path .\Lachesis.DHCP.pssc -Force

Then set **PowerShell endpoint** to `Lachesis.DHCP`. Test from an admin session first:
`Enter-PSSession -ComputerName dhcp.example.com -ConfigurationName Lachesis.DHCP -Credential ...`, then
`Get-Command` lists exactly the cmdlets above.

Expose no more than this list on a production endpoint. Extra cmdlets are not needed by Lachesis.

### Alternative: the default endpoint

With `configuration_name` empty Lachesis uses the host's default endpoint (`Microsoft.PowerShell`): the account is a
member of DHCP Administrators and is allowed to connect (Remote Management Users). The same cmdlets run, with the
account's own rights, in a full-language session, so the account can do anything DHCP Administrators and Remote
Management Users can; prefer the JEA endpoint, and remove the account's access to the default endpoint once JEA
works. There is no WinRS command-shell mode: it needs a local administrator, it ran a script the adapter sent, and
a non-administrator account is refused by the host anyway, so it was removed.

## Connection settings

| Field | Default | Meaning |
|---|---|---|
| `configuration_name` | empty | JEA endpoint name (above); empty uses the default PowerShell endpoint |
| `failover_sync` | `replicate` | `replicate` pushes a written scope to the partner with `Invoke-DhcpServerv4FailoverReplication`; `write_both` makes the plan write each peer instead (below) |
| `failover_settle_seconds` | 30 | how long a write to a failover scope, and the check after a replication, waits for the relationship to be Normal; 0 does not wait; 0 to 120 |
| `command_timeout_seconds` | 120 | how long one remote call may run; 10 to 900 |
| `max_response_mb` | 64 | the largest answer accepted from one cmdlet; 1 to 256 |
| `max_items` | 1000000 | the most leases (or reservations of one scope) one read accepts; more is treated as a fault |
| `reservation_options` | true | also read the options of each reservation (one query per reservation); without them a reservation's options cannot be verified |

## Probe

`Probe` runs `Get-DhcpServerVersion` and `Get-DhcpServerSetting`, reads the failover relationships and the
server-level options (all read-only), and reports the DHCP module version (`ServerVersion`, `10.0` on Server
2016 and later), whether the server is authorized in Active Directory (a warning when it is not), whether it is
domain joined, the endpoint kind (`jea` or `powershell`, also the `Mode` of every capability) and, with a
relationship, the capability `failover`, `Details["ha_state"]` and `ha_mode`, and the traits `config_sync`
(`on_demand` for `replicate`, `none` for `write_both`) and `lease_sync = automatic`. A missing `DhcpServer` module
or a JEA endpoint that does not expose a cmdlet is `OK` false with a warning.

## Capabilities

| Capability | Status | Notes |
|---|---|---|
| scopes.read | yes | scope id is the network address (`192.0.2.0`); one `Pool` per scope (start, end, exclusions); scope-level options; lease time; `Disabled` when the state is not Active |
| leases.read | yes | per scope, sorted by address; an address filter is a direct lookup; `Ends` is the lease expiry; `Starts` is not available |
| reservations.read | yes | with their options |
| reservations.write | yes | create, update, delete; idempotent (below) |
| pools.write | update only | the range and the exclusions of a scope; a scope's range is created and removed with the scope, so create and delete are refused |
| options.write | scope level | create, update, delete of scope-level options |
| leases.write | no | releasing one lease on one partner of a pair has no verified equivalent in the module |
| failover | with a relationship | `Peers`, `Replicate` |

The range change is several cmdlets (remove the exclusions that go, move the range, add the new ones). If a step
fails, the steps already done are undone in reverse order and the error says whether the undo completed. Text
fields refuse characters outside the Basic Multilingual Plane in both directions. A scope option that appeared
after planning is a conflict on create, never overwritten. Not writable by this adapter, so not offered:
server-level options (there is no read path to verify them; the session has `ServerOptions` for the read side),
scope creation and removal, lease time, scope name and state, policies and classes.

### Lease reads: cost and consistency

`Get-DhcpServerv4Lease` has no paging: a scope is read whole in one call (about 0.7 KiB of JSON per lease, so
`max_response_mb` 64 holds roughly 90,000 leases) and sorted here. Leases change while a read is in progress and a
scope is read in one call, but several scopes are read one after the other, so a lease can move between scopes of
one listing; treat a lease listing as a point-in-time approximation (the plan engine does not use leases for its
state hash or its verify step). An address filter is a single direct lookup.

### Mapping

- A reservation is keyed on its client id. A 6-octet client id is reported as the hardware address (`HWAddr`); a
  longer one (`01` + MAC, a DUID) stays a client id. Writing: the client id when given, else the hardware address;
  both must name the same client.
- Windows allows one reservation per client per scope: an address held by another client and a client that already
  has a reservation at another address are both `ErrConflict`, detected by reading before writing.
- Options: `Code` is the option id; `Space` carries the vendor class and `Class` the user class; a multi-value
  option is the comma-joined text. Reservation options in a vendor or user class are not supported. Option values
  come back as a list on a reservation and as one space-joined string on a scope; the adapter splits the string for
  every type but `String`.
- Lease states: `Active`, `Expired`, `Declined` (and abandoned), reservation states (`ActiveReservation`,
  `InactiveReservation`) as `reserved`. A lease in a state the adapter does not know is skipped, never guessed.
- An exclusion outside the scope's range (Windows keeps those when the range is shrunk) is not reported.

## Failover

Windows failover replicates leases by itself but **not scope configuration** (range, exclusions, options,
reservations). Each scope in a relationship carries `ScopeFailover{relationship, peer}`.

- **Plan, `failover_sync` = `replicate`.** The plan engine writes the server it was asked to write, adds a
  `Replicate` step for the scope (`Invoke-DhcpServerv4FailoverReplication -ScopeId ... -Force`), then re-reads both
  servers and compares every entry of both against the expected end state: a difference is `unverified`, never
  `success`. `Replicate` reports itself confirmed only when the cmdlet completed and the relationship is Normal again
  afterwards. The cmdlet makes the written server contact its partner **with the calling account's credentials**:
  that works in a domain where the account (or the JEA virtual account's machine identity) may manage the partner,
  and fails on workgroup servers and for a JEA virtual account (below).
- **Plan, `failover_sync` = `write_both`.** Probe reports `config_sync = none`; the plan writes the scope on each
  peer (each write guarded and verified on its own server) and the engine compares both. `Replicate` answers not
  supported. Use this where replication cannot authenticate to the partner.
- **Guard.** Before any write to a scope in a relationship the adapter reads the relationship
  state. `Normal` proceeds. A transitional state (`Startup`, `RecoverWait`, `RecoverDone`, `Recover`,
  `PotentialConflict`, `ConflictDone`) is waited out for up to `failover_settle_seconds`, then the write is refused
  as **busy** (409 `server_busy`, nothing written, retry shortly). Any other state (`CommunicationInterrupted`,
  `PartnerDown`, `ResolutionInterrupted`, `Invalid`, unknown) is refused as a **conflict**: writing now could leave
  the partners different. A relationship whose state cannot be read is a conflict too (fail closed); reads carry
  on. A scope outside any relationship is not affected by the state of a pair.
- **Replicate never answers busy.** It is the step after the primary was written; if the relationship is settling
  then, the result is unconfirmed with a note ("replication was not run ... replicate the scope again once the
  relationship is Normal"), so busy is only ever returned before the first write of a plan.

## Duplicate reservations and unverified writes

The module's errors for a duplicate address or client id are opaque: it says `Failed to reserve IP address ...` for
a taken address and for a client that already has a reservation alike. The adapter never trusts them: it reads the
scope's reservations before writing (conflict and idempotence are decided there), and when a write fails it reads
again and decides by what it sees: the wanted reservation is there (success), the address or client now belongs to
someone else (`ErrConflict`), a create that half-applied (reservation present, options missing) is reported as
such, anything else is the host's cleaned message. After every successful write it reads again and compares with
the same equivalence the plan engine uses; a write it cannot observe is returned as an unconfirmed result with a
note, which the plan engine reports as `unverified`.


## Windows Server 2019 behaviour

Observed on Windows Server 2019 as an account that is **not** a local administrator (member of DHCP Administrators
and Remote Management Users), through a JEA endpoint unless noted.

**Transport**

- NTLM with TLS channel binding was accepted, and a pinned leaf certificate works. Typed parameters bind as expected:
  a list of strings to `string[]` and `IPAddress[]` parameters, an integer to `OptionId`.
- The WinRS command shell is refused (HTTP 500) for this account; only PSRP works, to the JEA endpoint or to the
  default `Microsoft.PowerShell` endpoint.
- A runspace pool takes about 3 s to open; one reused for every call serves a cmdlet in 10 to 60 ms. A cmdlet the
  JEA role does not expose, or any other command, is refused with a failed pipeline and the session stays usable.
- A cmdlet error is non-terminating by default: the pipeline completes with empty output and the error only in the
  error stream. The adapter therefore sends `-ErrorAction Stop` with every cmdlet (accepted in JEA).

**Output shapes** (`ConvertTo-Json -Depth 1` on Windows PowerShell 5.1): an IPAddress is a plain string in some
properties and `{"value","IPAddressToString"}` in others; a multi-valued option is a list on a reservation and one
space-joined string on a scope; a DateTime is `"/Date(ms)/"`; a TimeSpan is `"d.hh:mm:ss"`; a single result is not
wrapped in an array and no result is no output at all. Depth 1 keeps the output small (a lease is about 0.7 KiB);
deeper output carries the CIM class description of every object. The adapter decodes all of these.

**Cmdlet behaviour**

| Question | Answer |
|---|---|
| Empty description or name on `Set-DhcpServerv4Reservation` | accepted; both can be emptied (so a reservation name can be cleared) |
| Change the client id of an existing reservation | `Set-DhcpServerv4Reservation -IPAddress -ClientId` changes it |
| `Get-DhcpServerv4OptionValue -ScopeId` | returns the options set at that scope only (it includes option 51, the lease time); `-All` returned the same set; with `-IPAddress` it returns the reservation's own options; with no parameter the server-level options (none unless set) |
| Values containing a comma or a space | a `String` option keeps them as one value (`a.test,b.test`, `has space.test`). The canonical text of a multi-valued option is comma separated, so a `String` option value that itself contains a comma reads back ambiguous; IPv4 and numeric options are unaffected |
| Duplicate reservation | one message for both cases, `Failed to reserve IP address ... for ClientID/ClientDUID ... in scope ...`: a taken address and a client that already has a reservation cannot be told apart, hence the re-list |
| Lookup of one object by address that finds none | the cmdlet **fails** instead of returning nothing: `Get-DhcpServerv4Lease -IPAddress` at a free address (or one in no scope), `Get-DhcpServerv4OptionValue -IPAddress` for an address with no reservation, `Get-DhcpServerv4Reservation -IPAddress`. Error id `DHCP 20016,<cmdlet>`, message `Failed to get lease information for IPAddress x from DHCP server <name>.` The adapter matches the **error id** (every error id of the call must be exactly that cmdlet's 20016), reads it as "nothing there" and answers with an empty list; the same words with no id, another id, a mix, or a transport error stay errors. This is what the plan engine asks before it reserves or releases an address |
| A scope that does not exist | `Failed to get scope information ...` / `Failed to enumerate ...`, error id `DHCP 20005,<cmdlet>` (category ObjectNotFound): an error for every read of that scope, never an empty list |
| An option that is not set | `Get-DhcpServerv4OptionValue -OptionId` fails with `DHCP 20010`; the adapter lists options instead of asking for one |
| Error text and the server's name | the message ends with the server it ran on (`... on DHCP server <name>.`); the adapter replaces that by "the DHCP server" in everything it shows or stores |
| `ServerRole` / `ServerType` in a hot-standby relationship | `ServerRole` is the role of the member you asked (`Active` on A, `Standby` on B), `ServerType` is `PrimaryServer` / `SecondaryServer`; `PrimaryServerName` is the partner's IP address on a workgroup server |
| Failover `State` | only `Normal` was observable; the transitional and degraded names the guard uses are from the module's documentation |
| Shrinking a scope's range with exclusions, or adding an exclusion outside the range | accepted, nothing is revalidated: exclusions outside the range stay (they exclude nothing) and are not reported by the adapter; exclusions may be added or removed in any order |
| Active leases | `AddressState` `Active` with a `/Date(ms)/` expiry; reservations appear as `InactiveReservation` leases with no expiry |

**Replication**: `Invoke-DhcpServerv4FailoverReplication -ScopeId ... -Force`, run through the JEA endpoint as the
virtual account, fails with `Failed to get superscope information on DHCP server <partner>`: a workgroup server
(and a JEA virtual account, which has no network credentials) cannot authenticate to its partner. Nothing is
replicated, the partner is unchanged. On a workgroup pair the working mode is `failover_sync` = `write_both`: a
reservation, an option and a range change written to both peers read back identical entry by entry. Running the
replication as the local Administrator inside a scheduled task works but does not overwrite a changed client id of an
existing address. **Not settled:** whether replication succeeds for a domain-joined pair, and the exact text of
`Get-DhcpServerv4Failover` on a server with no relationship at all (the adapter treats
`Failed to get failover relationship ...` as "none", and any other failure as unreadable, which refuses writes).
