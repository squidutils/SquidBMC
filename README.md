# SquidBMC

Use your old server's remote console and power controls from a modern browser,
without installing ancient Java or an old browser on your own computer.

Older servers have a small management computer built in (a BMC: HP iLO2,
Dell iDRAC6). Its remote console needs Java browser plugins and old encryption
that current browsers and operating systems no longer support. SquidBMC runs
that old software inside a Docker container and shows the result in a normal
browser tab.

## What you get

- A dashboard for all your servers: power state, power draw, temperatures,
  fans, voltages and recent hardware event log entries.
- Power buttons: on, off, power cycle, reset, soft shutdown, identify LED and
  clear the event log.
- The graphical remote console (keyboard, video, mouse) of each server in a
  browser tab.
- A text console in the browser, from the BIOS screen through to a Linux login
  prompt, with no Java at all.
- Virtual media: attach an ISO file to a server from a folder on the host.
- A login page: local username and password, GitHub, or any OpenID Connect
  single sign-on provider.

## What it works with

- HP iLO2 (tested on an HP ProLiant DL320 G6)
- Dell iDRAC6 (tested on a Dell PowerEdge R510)

The dashboard, power controls and text console use standard IPMI, so other BMCs
can be added to the dashboard too. The graphical console is wired up for one
iLO2 and one iDRAC6.

## Quick start

You need Docker with Docker Compose on a 64-bit x86 machine that can reach your
servers' management ports.

```bash
git clone https://github.com/squidutils/SquidBMC.git
cd SquidBMC
cp .env.example .env
# edit .env: your BMC addresses and logins, and a login for SquidBMC itself
docker compose up -d --build
```

Then open http://localhost:8088 in your browser. The first build takes a while
because it downloads the old Firefox and Java pieces.

Set a login before you let anyone else reach it: with no login configured the
app is open to anyone who can load the page.

## Documentation

- [Documentation index](docs/README.md) - where to start
- [Installation](docs/installation.md) - Docker Compose, Portainer, upgrades and backups
- [Configuration](docs/configuration.md) - every setting in `.env`
- [Using SquidBMC](docs/usage.md) - dashboard, graphical consoles, text consoles
- [Virtual media](docs/virtual-media.md) - attaching ISO files to a server
- [Authentication](docs/authentication.md) - local login, GitHub, OpenID Connect
- [Reverse proxy and TLS](docs/reverse-proxy.md) - putting it behind HTTPS
- [Architecture](docs/architecture.md) - how it works and why the old software is needed
- [Development](docs/development.md) - running the tests, CI, project layout
- [Troubleshooting](docs/troubleshooting.md) - common problems and fixes

## Security

Keep your BMCs on a separate management network, and put SquidBMC behind a login
and HTTPS. The details, including the risks that are accepted on purpose, are in
[SECURITY.md](SECURITY.md).

## Licence

MIT, see [LICENSE](LICENSE).
