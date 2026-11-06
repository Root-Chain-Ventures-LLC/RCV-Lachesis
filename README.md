# RCV Lachesis

**Lachesis** is a self-hosted, white-label DHCP manager for networks that run more than one kind of DHCP
server. It gives you one place to see and manage leases, reservations, scopes and pools, and options across
vendors. This release ships adapters for ISC Kea, Windows Server DHCP, VyOS and UniFi Network, plus an experimental
Ubiquiti EdgeRouter adapter; every vendor is a separate adapter, and the rest of the product never depends on which
vendor a server is.

- **Plan, then apply.** Every change is previewed as a plan, can require approval, is applied, and is read back
  from the server to verify it. A write that cannot be verified is reported as unverified, never as a success.
- **Audit log.** Every change is recorded in an append-only log that the application itself cannot rewrite.
- **Single sign-on.** Any OpenID Connect provider, with local break-glass accounts and roles (viewer, operator,
  admin). API tokens are scoped and expire.
- **MCP server (off by default).** A read-only Model Context Protocol endpoint for AI agents, behind API tokens.
- **Credentials stay sealed.** Server credentials are encrypted under a master key you hold and are never
  shown again after you save them.

Lachesis runs as one container image plus PostgreSQL.

## License

**RCV Community License 1.1**: free for personal and home-lab use and for any organization's own internal
operations. A paid commercial license is required only to offer the software to third parties as a hosted,
managed, or SaaS service. See [`LICENSE`](LICENSE). Commercial inquiries: **legal@rootchainventures.com**.

Lachesis is built from a private source repository; the source is not published. See
[`CONTRIBUTING.md`](CONTRIBUTING.md).

## What you download

Each release carries the install artifacts and nothing else (the release notes are empty on purpose):

| Artifact | |
|---|---|
| `install-bundle.tar.gz` | the compose file, `.env.example`, the documentation, the license and the signing key |
| `SHA256SUMS`, `SHA256SUMS.sig`, `cosign.pub` | checksums, their signature and the public key |

The container image (linux/amd64), pulled without logging in:

- `ghcr.io/root-chain-ventures-llc/lachesis`

Only the newest stable release is kept here; older releases and tags are removed when a new one is published.

## Requirements

- A Linux host with Docker Engine and the Docker Compose v2 plugin.
- A host that can reach your DHCP servers.
- A DNS name and a TLS certificate for anything beyond a local trial. Lachesis serves plain HTTP and expects a
  **reverse proxy to terminate TLS** in front of it ([`docs/install.md`](docs/install.md), section 5).

## Quick start

```sh
git clone https://github.com/Root-Chain-Ventures-LLC/RCV-Lachesis.git
cd RCV-Lachesis
cp .env.example .env
sed -i "s|^POSTGRES_PASSWORD=.*|POSTGRES_PASSWORD=$(openssl rand -hex 16)|" .env
sed -i "s|^LACHESIS_DB_APP_PASSWORD=.*|LACHESIS_DB_APP_PASSWORD=$(openssl rand -hex 16)|" .env
sed -i "s|^LACHESIS_SETUP_TOKEN=.*|LACHESIS_SETUP_TOKEN=$(openssl rand -hex 16)|" .env   # for the first administrator
mkdir -p secrets && ( umask 077; openssl rand -base64 32 > secrets/master.key )           # the master key
sudo chown 65532 secrets/master.key && chmod 600 secrets/master.key
docker compose pull && docker compose up -d
curl -fsS http://127.0.0.1:8080/api/ready
```

Open `http://127.0.0.1:8080` and create the first administrator with the setup token from `.env`. The compose
file publishes the web port on the loopback interface only, because Lachesis speaks plain HTTP: put a TLS
proxy in front of it before exposing it, and set `LACHESIS_BASE_URL` and `TRUST_PROXY` in `.env`.

**Back up `secrets/master.key` separately from the database.** It wraps the keys that protect every stored
credential; without it they cannot be read, and stored next to a database backup it protects nothing.

The compose file runs the image of the release you cloned. `LACHESIS_VERSION` in `.env` selects another
release (or `latest`), and `LACHESIS_IMAGE` overrides the image completely. The containers run as a non-root
user with a read-only root file system, no capabilities and `no-new-privileges`.

## Verify what you downloaded

Every release is signed with a key pair; the public key is [`cosign.pub`](cosign.pub) in this repository. Take it
from here (or from a release you verified earlier), not from the download being checked.

```sh
cosign verify-blob --key cosign.pub --signature SHA256SUMS.sig --insecure-ignore-tlog=true SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS
cosign verify --key cosign.pub --insecure-ignore-tlog=true ghcr.io/root-chain-ventures-llc/lachesis:<version>
```

`--insecure-ignore-tlog=true` is expected: signatures are made offline with a key pair and are not uploaded to a
public log. A release without `SHA256SUMS.sig` is not a Lachesis release: do not install it.

## Upgrade

1. **Back up the database** and keep `.env` and `secrets/master.key` as they are
   ([`docs/install.md`](docs/install.md), "Before you upgrade"). Upgrades cannot be undone by starting the
   previous version: going back means restoring that backup.
2. Get the files of the new release: `git pull` in your clone (or unpack the new `install-bundle.tar.gz` over it,
   keeping `.env` and `secrets/`). The compose file names the new version.
3. `docker compose pull && docker compose up -d`, then check `curl -fsS http://127.0.0.1:8080/api/ready`.
   Migrations run before the new version starts.

Upgrade one release at a time when you can.

## Documentation

[`docs/README.md`](docs/README.md) indexes the operator documentation: [installing and upgrading](docs/install.md)
(master key, reverse proxy, sign-in, roles, API tokens, database roles, audit retention) and one guide per adapter:
[Kea](docs/adapters/kea.md), [Windows](docs/adapters/windows.md), [VyOS](docs/adapters/vyos.md),
[UniFi](docs/adapters/unifi.md) and [EdgeRouter](docs/adapters/edgeos.md) (experimental).

## Reporting problems

- **Security issues:** please report them privately, see [`SECURITY.md`](SECURITY.md). Do not open a public issue.
- **Bugs and feature requests:** GitHub Issues on this repository. Use synthetic data (no real server names,
  addresses or credentials).
- **Commercial licensing:** legal@rootchainventures.com.

A Root Chain Ventures App
