# Development

## Project layout

- `docker-compose.yml` - the two services (`console`, `web`) and four named
  volumes.
- `Dockerfile` - the legacy KVM engine image (`console`): Debian stretch,
  OpenJDK 8, IcedTea-Web, Firefox 52 ESR, firefox-esr, Xvnc, matchbox.
- `start.sh` - engine entrypoint: starts the two Xvnc displays and window
  managers, then the launch listener.
- `launchd.py` - engine launch listener on port 9000 (`/launch`, `/kill`,
  `/relaunch`).
- `ilo2`, `idrac6` - engine launch scripts: prepare a Firefox profile and open
  the BMC's web UI.
- `trusted.certs` - pre-seeded IcedTea-Web trust store (JKS) with the BMC applet
  signing certificates.
- `web/Dockerfile` - the `web` image: Ruby, ipmitool, ssh, sshpass, noVNC 1.4,
  xterm.js.
- `web/app.rb` - the Sinatra app: login, dashboard, server store, power actions,
  console pages.
- `web/config.ru` - Rack entrypoint: session and OmniAuth middleware,
  `TrustProxy`, the VNC WebSocket proxy (`/websockify`) and the serial bridge
  (`/solws`).
- `web/views/` - ERB templates.
- `web/spec/` - RSpec tests.
- `bmcfleet.sh` - command-line IPMI summary (see
  [Configuration](configuration.md#bmcconfexample-and-bmcfleetsh)).
- `isos/` - default virtual media folder.

## Making a change

The app code is copied into the images at build time; nothing is bind-mounted.
After editing, rebuild the service you changed:

```bash
docker compose up -d --build web       # app, views, specs, Gemfile
docker compose up -d --build console   # engine image or launch scripts
```

## Running the tests

Build the `web` image and run the specs in a throwaway container, without the
Compose environment:

```bash
docker build -t squidbmc-web-test web
docker run --rm squidbmc-web-test bundle exec rspec
```

This is what CI does in effect (35 examples, 0 failures at the time of
writing). The specs are copied into the image, so rebuild after changing them.

Running the specs inside the live container (`docker exec bmc-web bundle exec
rspec`) or through `docker compose run web` is not reliable. Compose sets every
variable the app reads, to an empty string if it is not in `.env`, and
`spec_helper.rb` only fills in test values for variables that are not set at
all. With empty or real values in the environment, most examples fail.

RSpec + Rack::Test cover:

- Authentication: local login, GitHub allow-list, OIDC users and groups,
  fail-closed behaviour, partial group names, the OIDC callback when OIDC is
  off, failure messages without echoed markup, logout.
- Routing and the landing page, the dashboard, and the power-action allow-list.
- The console launch trigger, reload/exit, and noVNC embedding.
- The serial console page.
- Server CRUD, including keeping the password when the field is left blank.
- The `ipmitool` command line, including the pinned cipher suite and the
  `IPMI_CIPHER_SUITE` override.

IPMI is stubbed, so the tests never touch a real BMC. `spec_helper.rb` sets
`RACK_ENV=test` (which also stops the background IPMI poller), fake login
settings, and a temporary `SERVERS_PATH`; it builds the same session and
OmniAuth middleware stack as `config.ru`.

## CI

`.github/workflows/ci.yml` runs on every push to `main` and every pull request,
on Ruby 3.3 in `web/`:

- **RSpec**.
- **bundler-audit** against `Gemfile.lock`. Blocking: the build fails on a known
  CVE.
- **Brakeman**. Best-effort and non-blocking; it is aimed at Rails and misses
  Sinatra-specific issues.

To run the dependency audit locally without installing anything on the host,
use the full `ruby` image (the slim app image has no `git`, which
`bundle-audit --update` needs to fetch the advisory database):

```bash
docker run --rm -v "$PWD/web":/app:ro -w /app ruby:3.3 \
  sh -c 'gem install bundler-audit && bundle-audit check --update'
```

A plan for automated OWASP ZAP scanning is in [ZAP.md](ZAP.md).

## Adding a BMC's Java signing certificate to the image

The image ships with trust for HPE's iLO2 applet code-signing certificate. To
bake in another one:

1. Open the console, tick "Always trust content from this publisher" in the
   Java dialog once, and run the console.
2. Copy the updated trust store out of the engine:

   ```bash
   docker cp bmc-console:/root/.config/icedtea-web/security/trusted.certs .
   ```

3. List the entries and delete any that is not a vendor code-signing
   certificate. The live store can also pick up your own BMC's self-signed
   HTTPS certificate, whose subject is that BMC's hostname, and that must not
   be committed. It is a JKS keystore with the default password `changeit`:

   ```bash
   docker cp trusted.certs bmc-console:/tmp/trusted.certs
   docker exec bmc-console keytool -list -v -keystore /tmp/trusted.certs -storepass changeit
   docker exec bmc-console keytool -delete -alias <alias> -keystore /tmp/trusted.certs -storepass changeit
   docker cp bmc-console:/tmp/trusted.certs .
   ```

4. Commit the new `trusted.certs` and rebuild the `console` image.

Existing installs keep their own trust store on the `icedtea-trust` volume; the
image copy is used only when that volume is first created.
