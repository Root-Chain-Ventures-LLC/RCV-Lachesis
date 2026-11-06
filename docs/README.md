# Lachesis documentation

| Topic | Page | What is in it |
|---|---|---|
| Installing and upgrading | [`install.md`](install.md) | Requirements, `.env`, the master key and its rotation, where the port is published, the reverse proxy and `TRUST_PROXY`, starting and upgrading, backups and rollback, first administrator, single sign-on, local accounts, roles, sessions, API tokens, database roles, audit retention, secret references |
| Kea adapter | [`adapters/kea.md`](adapters/kea.md) | ISC Kea DHCPv4: control API and credentials, what Probe reports, write modes, capabilities, high-availability pairs and vendor quirks |
| Windows adapter | [`adapters/windows.md`](adapters/windows.md) | Windows Server DHCP over WinRM/PSRP: least-privilege account and JEA endpoint, TLS pinning, failover sync modes, capabilities and vendor quirks |
| VyOS adapter | [`adapters/vyos.md`](adapters/vyos.md) | VyOS 1.4 and later over the HTTPS API: API key, TLS pinning, high-availability pairs, commit and save, lease time zones, capabilities and vendor quirks |
| UniFi adapter | [`adapters/unifi.md`](adapters/unifi.md) | UniFi Network application: local account, site, TLS pinning, fixed IPs and DHCP ranges, writes the controller discards, DHCP-off networks; UniFi OS console mode is experimental |
| EdgeRouter adapter (experimental) | [`adapters/edgeos.md`](adapters/edgeos.md) | Ubiquiti EdgeOS: the undocumented web API and the source for every fact used, TLS pinning, static mappings, ranges and options; not yet validated on real firmware |

Lachesis runs as one container image plus PostgreSQL, wired together by the `compose.yaml` in the root of this
repository.
