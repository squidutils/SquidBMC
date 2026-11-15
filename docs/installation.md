# Installation

SquidBMC is two Docker containers in one Compose project. Everything it needs
is downloaded while the images build, so it builds straight from a `git clone`
with no files to stage on the host first.

## Requirements

- Docker with Docker Compose v2. The compose file uses the long `env_file`
  syntax (`path:` / `required: false`), which needs Compose 2.24 or newer.
- A 64-bit x86 (amd64) host. The engine image installs the linux-x86_64 build
  of Firefox 52 and amd64 Debian packages.
- Outbound internet access while building. The build downloads from
  `archive.debian.org` (the Debian stretch and jessie archives),
  `archive.mozilla.org` (Firefox 52.9.0 ESR), `github.com` (noVNC),
  `registry.npmjs.org` (xterm.js) and `rubygems.org` (Ruby gems).
- Network access from the host to each BMC:
  - UDP 623 (IPMI) from the `web` container, for the dashboard, power actions
    and IPMI Serial-over-LAN.
  - TCP 22 (SSH) from the `web` container, for the SSH text console.
  - TCP 443 (HTTPS) from the `console` container, for the BMC web interface,
    plus the ports the BMC's Java console and virtual media use. These are set
    in each BMC's network settings; the factory defaults are commonly 23 and
    17988 on iLO2, and 5900 and 3668 on iDRAC6. Check your BMC.
- The browser you use needs to reach `cdn.tailwindcss.com`: the page styling is
  loaded from there at runtime.

## Install with Docker Compose

```bash
git clone https://github.com/squidutils/SquidBMC.git
cd SquidBMC
cp .env.example .env
```

Edit `.env` (never commit it; it is in `.gitignore`):

- `ILO_IP`, `ILO_USER`, `ILO_PASS` and `IDRAC_IP`, `IDRAC_USER`, `IDRAC_PASS`
  for your BMCs.
- `SESSION_SECRET`: generate one with `openssl rand -hex 32`.
- At least one login method: `PLATFORM_USER` and `PLATFORM_PASS`, or GitHub, or
  OpenID Connect. See [Authentication](authentication.md).

Values are read literally, so do not put quotes around them. Every variable is
described in [Configuration](configuration.md).

Build and start:

```bash
docker compose up -d --build
```

Open http://localhost:8088. Port 8088 is bound to `127.0.0.1` only, so it is
not reachable from other machines until you put a reverse proxy in front or
change `WEB_BIND` (see [Reverse proxy and TLS](reverse-proxy.md)).

Check that both containers are running:

```bash
docker compose ps
docker compose logs web
docker compose logs console
```

The containers are named `bmc-web` and `bmc-console`.

## Deploy with Portainer (Git stack)

Because the images are self-contained (Firefox 52 is fetched in a build stage,
not staged on the host), Portainer can build straight from the Git repository:

1. **Stacks -> Add stack -> Repository.**
2. Repository URL `https://github.com/squidutils/SquidBMC`, compose path
   `docker-compose.yml`.
3. **Environment variables:** add your `.env` values (`ILO_*`, `IDRAC_*`,
   `PLATFORM_USER`/`PLATFORM_PASS`, `SESSION_SECRET`, and the `GITHUB_*` trio or
   the `OIDC_*` set if you use single sign-on). Portainer writes these to the
   stack's environment. The compose file reads `.env` only if it exists, so
   nothing breaks when a variable is unset.
4. **GitOps updates:** enable polling (for example every 5 minutes) or add the
   webhook, so a push to `main` re-pulls and rebuilds automatically. This is the
   auto-redeploy path. Watchtower is not needed: Watchtower re-pulls registry
   tags, and here Portainer builds from source.
5. For production put it behind TLS: set `TRUST_PROXY=1`, keep the default
   `WEB_BIND=127.0.0.1`, and front `127.0.0.1:8088` with a reverse proxy that has
   **WebSockets on** and a long read timeout (the KVM and serial streams need
   both). See [Reverse proxy and TLS](reverse-proxy.md) and
   [SECURITY.md](../SECURITY.md#running-behind-tls-reverse-proxy).

Why the compose file maps each variable explicitly: Portainer Git stacks only
use stack variables to fill in `${VAR}` references in the compose file; they
are not passed into the containers otherwise. So every variable the app reads
is listed under `environment:` in `docker-compose.yml`. Both services also set
`pull_policy: build`, so a Portainer redeploy rebuilds the images.

## Upgrading

With Docker Compose:

```bash
cd SquidBMC
git pull
docker compose up -d --build
```

With Portainer, either let GitOps polling or the webhook pick up the change,
or use **Pull and redeploy** on the stack.

What happens during an upgrade:

- Rebuilding recreates `bmc-web`, which drops any open KVM or serial WebSocket.
  Browsers just reconnect (the noVNC view reconnects by itself; use
  **Reconnect** on a text console).
- The named volumes (`servers-data`, `ilo-profile`, `idrac-profile`,
  `icedtea-trust`) persist across rebuilds, so the server list, accepted
  certificate exceptions and Java trust decisions are kept.
- Do not use `docker compose down -v` unless you mean it: `-v` deletes the named
  volumes, including the server list.
- After an upgrade that changes `.env.example`, compare it with your `.env` and
  add any new variables.

## Volumes and what is in them

- `servers-data`, mounted at `/data` in `web`: `servers.json`, the server list
  shown on the dashboard, including BMC passwords **in plaintext**. This is the
  one to back up.
- `ilo-profile` (`/root/.ff52-ilo`) and `idrac-profile` (`/root/.ffesr-idrac`)
  in `console`: the Firefox profiles, holding accepted BMC certificate
  exceptions and the window size. Recreated automatically if lost; you would
  just accept the BMC certificate again.
- `icedtea-trust` (`/root/.config/icedtea-web/security`) in `console`: Java
  ("Always trust this publisher") trust decisions. Seeded from the image's
  `trusted.certs` when the volume is first created.
- `./isos`, bind-mounted read-only at `/isos` in `console`: your ISO files (see
  [Virtual media](virtual-media.md)). This is a normal folder on the host.

Your `.env` file is not in a volume. Back it up separately and keep it private.

## Backing up

These commands use `--volumes-from`, which works whatever your Compose project
is called. Run them from the `SquidBMC` directory; they write to `./backup`.

```bash
mkdir -p backup
docker run --rm --volumes-from bmc-web -v "$PWD/backup":/backup debian:bookworm-slim \
  tar czf /backup/servers-data.tgz -C /data .
docker run --rm --volumes-from bmc-console -v "$PWD/backup":/backup debian:bookworm-slim \
  tar czf /backup/console-state.tgz /root/.ff52-ilo /root/.ffesr-idrac /root/.config/icedtea-web/security
chmod 600 backup/*.tgz
```

`servers-data.tgz` contains BMC passwords in plaintext. Store it with the same
care as `.env`, and do not put it on shared or untrusted storage.

## Restoring

Stop the stack first so nothing writes to the volumes, restore, then start it:

```bash
docker compose stop
docker run --rm --volumes-from bmc-web -v "$PWD/backup":/backup debian:bookworm-slim \
  tar xzf /backup/servers-data.tgz -C /data
docker run --rm --volumes-from bmc-console -v "$PWD/backup":/backup debian:bookworm-slim \
  tar xzf /backup/console-state.tgz -C /
docker compose start
```

To restore onto a fresh host, run `docker compose up -d --build` once so the
containers and volumes exist, then follow the steps above.

## Removing

```bash
docker compose down      # stops and removes the containers, keeps the volumes
docker compose down -v   # also deletes the volumes, including the server list
```
