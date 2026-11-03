# Lachesis documentation

| Topic | Page | What is in it |
|---|---|---|
| Installing and upgrading | [`install.md`](install.md) | Requirements, `.env`, the master key and its rotation, where the port is published, the reverse proxy and `TRUST_PROXY`, starting and upgrading, backups and rollback, first administrator, single sign-on, local accounts, roles, sessions, API tokens, database roles, audit retention, secret references |
| Kea adapter | [`adapters/kea.md`](adapters/kea.md) | ISC Kea DHCPv4: control API and credentials, what Probe reports, write modes, capabilities, high-availability pairs and vendor quirks |

Lachesis runs as one container image plus PostgreSQL, wired together by the `compose.yaml` in the root of this
repository.
