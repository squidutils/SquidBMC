# SquidBMC documentation

Start with [Installation](installation.md), then [Configuration](configuration.md).
If something does not work, go to [Troubleshooting](troubleshooting.md).

## Pages

- [Installation](installation.md) - requirements, Docker Compose, Portainer Git
  stacks, upgrading, backing up and restoring the volumes.
- [Configuration](configuration.md) - every `.env` variable, how the server
  list is seeded and stored, and `bmcfleet.sh`.
- [Using SquidBMC](usage.md) - the dashboard, power actions, adding servers,
  the graphical KVM consoles and the text (serial) consoles.
- [Virtual media](virtual-media.md) - making ISO files available to the
  graphical consoles.
- [Authentication](authentication.md) - local login, GitHub OAuth, OpenID
  Connect, allow-lists, fail-closed behaviour and sessions.
- [Reverse proxy and TLS](reverse-proxy.md) - `TRUST_PROXY`, `WEB_BIND`,
  WebSocket and timeout settings, an example proxy configuration.
- [Architecture](architecture.md) - the two containers, the noVNC and serial
  WebSocket proxies, and why an old Firefox and Java are needed.
- [Development](development.md) - project layout, running the RSpec tests in
  the container, CI.
- [Troubleshooting](troubleshooting.md) - problems and fixes, based on the
  error messages the app and the BMCs actually produce.

## Other documents

- [SECURITY.md](../SECURITY.md) - threat model, security controls, accepted
  risks, hardening steps and how to report a problem.
- [PLAN.md](PLAN.md) - implementation notes and plans. Contains the operator
  recipe for enabling the serial console on the servers themselves (BIOS
  settings and GRUB/getty configuration).
- [ZAP.md](ZAP.md) - plan for automated OWASP ZAP scanning in GitHub Actions
  (not implemented yet).

## Security in brief

The full posture is in [SECURITY.md](../SECURITY.md). In short:

- Every route is behind a login: OpenID Connect single sign-on (any IdP, user or
  group allow-list, fail-closed), GitHub OAuth (allow-list) and/or a local login
  (constant-time compare). The console and serial WebSockets are gated too.
- BMC credentials stay on the server and are never sent to the browser. They
  live in `.env` and in the `servers-data` volume (`servers.json`, plaintext, the
  same trust level as `.env`), both outside git and the image.
- IPMI and console commands are run without a shell (argument arrays, so no
  command injection), page output is auto-escaped (Erubi) against XSS, and
  `SameSite=Lax` cookies are the CSRF mitigation.
- Port `8088` binds to `127.0.0.1` by default. Put a TLS reverse proxy in front
  and set `TRUST_PROXY=1` (see [Reverse proxy and TLS](reverse-proxy.md)).
- Put your BMCs on an isolated management VLAN. IPMI 2.0 on this hardware has
  weaknesses that cannot be fixed; network isolation is the intended mitigation.
- CI runs RSpec and a blocking `bundler-audit` (dependency CVEs) on every push.

## Roadmap

- Vendor-aware sensor naming so HP ambient and fan fields fill in like Dell's.
- More nodes and BMC vendors.
- Graphical KVM consoles for servers added through the dashboard (today only the
  two consoles seeded from `.env` are wired up).
