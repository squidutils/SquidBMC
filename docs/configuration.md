# Configuration

All settings are environment variables. For Docker Compose put them in `.env`
in the repository root (copy `.env.example`). For a Portainer Git stack add them
as stack environment variables (see [Installation](installation.md)).

Rules for `.env`:

- One `NAME=value` per line, no quotes. Docker reads the values literally, so
  quotes would become part of the value.
- `.env` is listed in `.gitignore` and `.dockerignore`: it is never committed
  and never copied into an image.
- BMC credentials are only used on the server side and are never sent to the
  browser.
- An unset variable is treated as empty, except where a default is listed below.

## BMCs

- `ILO_IP`, `ILO_USER`, `ILO_PASS` - the HP iLO2's management address and
  login. Seeds the `HP-NODE` entry on the dashboard and is the target of the
  iLO2 graphical console.
- `IDRAC_IP`, `IDRAC_USER`, `IDRAC_PASS` - the Dell iDRAC6's management
  address and login. Seeds the `DELL-NODE` entry and is the target of the
  iDRAC6 graphical console.
- `IPMI_CIPHER_SUITE` - the IPMI 2.0 cipher suite passed to every `ipmitool`
  call as `-C`. Default `3` (SHA1 authentication and integrity, AES-128), which
  works on iLO2 and iDRAC6. Set it to an empty value to let `ipmitool` discover
  the suite itself; old BMCs such as iLO2 do not answer that discovery and
  `ipmitool` prints "Unable to Get Channel Cipher Suites" before falling back.
  In `docker-compose.yml` an unset variable means `3`, and an empty one means
  auto-discovery.
- `FAN_MAX_ILO` (default `18000`) and `FAN_MAX_IDRAC` (default `12000`) - the
  maximum fan speed in RPM for the two seeded servers. Used only by the
  dashboard's fan unit toggle to convert between percent and RPM. These are
  best-guess defaults; tune them to your hardware.

## Login

See [Authentication](authentication.md) for how these work together.

- `SESSION_SECRET` - signs the session cookie. Generate it with
  `openssl rand -hex 32`. If it is unset, a random secret is generated at every
  start, which logs everybody out on each restart.
- `PLATFORM_USER`, `PLATFORM_PASS` - a local username and password. Leave
  `PLATFORM_USER` empty to turn local login off.
- `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` - a GitHub OAuth app. GitHub login
  is shown when `GITHUB_CLIENT_ID` is set.
- `GITHUB_ALLOWED_USERS` - comma-separated GitHub logins allowed in
  (case-insensitive).
- `OIDC_ISSUER` - the OpenID Connect issuer URL. OIDC login is turned on when
  this is set; the app reads `<issuer>/.well-known/openid-configuration`.
- `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` - the confidential client registered at
  your identity provider.
- `OIDC_REDIRECT_URI` - the callback URL, `https://<your-host>/auth/oidc/callback`.
  It must match the URI registered at the identity provider exactly.
- `OIDC_ALLOWED_USERS` - comma-separated usernames or email addresses allowed in
  (case-insensitive).
- `OIDC_ALLOWED_GROUPS` - comma-separated group names allowed in, matched
  exactly against the `groups` claim.
- `OIDC_LABEL` - the button text, "Sign in with <OIDC_LABEL>". Default
  "single sign-on".

If both OIDC allow-lists are empty, nobody can sign in through OIDC.

If no login method at all is configured (no `PLATFORM_USER`, no
`GITHUB_CLIENT_ID`, no `OIDC_ISSUER`), the app runs open with no login.

## Network and reverse proxy

See [Reverse proxy and TLS](reverse-proxy.md).

- `TRUST_PROXY` - set to `1` only when the app is reached solely through a TLS
  reverse proxy that sets `X-Forwarded-Proto`. It makes OAuth redirects use
  `https` and marks the session cookie `Secure`. Any other value means off.
  Default off (`.env.example` sets `0`).
- `WEB_BIND` - the host interface port 8088 is published on. Default
  `127.0.0.1` (this host only). `0.0.0.0` makes it reachable from the network
  directly, which is not advised.

## Set by docker-compose.yml

These are set in the compose file and normally do not need changing:

- `CONSOLE_HOST` - the hostname of the KVM engine on the compose network.
  Default `console`.
- `VNC_PORT` - set to `5901` in the compose file, but not read by the current
  code: the VNC ports are fixed (5901 for iLO2, 5902 for iDRAC6).
- `SERVERS_PATH` - not set in the compose file; the app defaults to
  `/data/servers.json` (on the `servers-data` volume). The tests point it at a
  temporary file.

## Which container gets what

- `web` receives all the variables above.
- `console` (the KVM engine) receives only `ILO_IP`, `IDRAC_IP`, `IDRAC_USER`
  and `IDRAC_PASS` through `environment:`, plus the whole `.env` file when one
  exists (`env_file`). The launch scripts open the BMC's login page; you log in
  to the BMC yourself.

## The server list (`servers.json`)

The dashboard's server list is stored in `/data/servers.json` on the
`servers-data` volume. It is created from `.env` the first time the app needs
it: one `HP-NODE` entry from the `ILO_*` values and one `DELL-NODE` entry from
the `IDRAC_*` values, with the `FAN_MAX_*` values.

After that, `servers.json` is the source of truth for the dashboard, power
actions and text consoles. Changing `ILO_*` or `IDRAC_*` in `.env` later does
**not** update it. To change a server, use **Edit** on the dashboard, or delete
the file to re-seed it from `.env`:

```bash
docker exec bmc-web rm /data/servers.json
```

The graphical consoles are different: the engine reads `ILO_IP` and `IDRAC_IP`
directly from its environment, so after changing those run
`docker compose up -d` to recreate the containers with the new values.

Each entry holds a label, the BMC IP, username, password, vendor (`hp`, `dell`
or `other`) and the fan maximum. Passwords are stored in plaintext; see
[SECURITY.md](../SECURITY.md#credential-handling).

## `bmc.conf.example` and `bmcfleet.sh`

- `bmc.conf.example` shows a simple list format, one BMC per line:
  `label ip user pass`. Copy it to `bmc.conf` (gitignored) if you want such a
  list. Nothing in the web app or `bmcfleet.sh` reads `bmc.conf` at present.
- `bmcfleet.sh` is a small command-line version of the dashboard. It reads the
  `ILO_*` and `IDRAC_*` values from a `.env` file (default `./.env`, or the path
  given as the first argument) and prints one line per node: power state,
  ambient temperature, fan count and power draw. It runs `ipmitool` on the
  machine you run it on, so it needs `ipmitool` installed there. It does not
  pass `-C`, so it uses `ipmitool`'s cipher suite discovery.
