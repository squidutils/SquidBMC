# Troubleshooting

Start with the container status and logs:

```bash
docker compose ps
docker compose logs --since 10m web
docker compose logs --since 10m console
```

Logs can contain addresses and usernames from your setup. Check them before
pasting them anywhere public.

## Dashboard and IPMI

### A server shows `unreachable`

The BMC did not answer `chassis power status` within the timeout (2 seconds,
one retry). `ipmitool` errors are not shown in the UI, so test from inside the
`web` container. This reads the password from the container's environment
instead of putting it on the command line:

```bash
docker exec -it bmc-web sh -c 'IPMI_PASSWORD="$ILO_PASS" ipmitool -I lanplus -H "$ILO_IP" -U "$ILO_USER" -E -C 3 chassis power status'
```

(Use `IDRAC_IP`, `IDRAC_USER` and `IDRAC_PASS` for the iDRAC.) Common causes:

- Wrong IP, username or password. Note that the dashboard uses `servers.json`,
  not `.env`; see the next section.
- IPMI over LAN is turned off on the BMC. Turn it on in the BMC's network or
  IPMI settings.
- UDP 623 is blocked between the Docker host and the management network.

### "Unable to Get Channel Cipher Suites"

`ipmitool` prints this when it tries to discover the cipher suite and the BMC
(HP iLO2 in particular) does not answer. SquidBMC avoids it by passing `-C 3`
on every call. If you see it:

- Check `IPMI_CIPHER_SUITE` is `3` or unset, not empty. An empty value turns on
  auto-discovery.
- Run `docker compose up -d` after changing `.env` so the container picks up
  the change.
- `bmcfleet.sh` does not pass `-C`, so it can still show this message.

The same message used to appear in the text console when using SOL on an iLO2;
the fixed cipher suite applies there too.

### I changed `.env` but the dashboard still uses the old IP or password

The dashboard's server list is created from `.env` only once, the first time
the app runs, and is then kept in `/data/servers.json`. Either use **Edit** on
the dashboard, or delete the file so it is created again from `.env`:

```bash
docker exec bmc-web rm /data/servers.json
```

Any servers added through the dashboard are lost when you delete the file.

### Fans or ambient temperature missing on the HP

HP iLO2 and Dell iDRAC6 name their sensors differently, and some iLO2 firmware
does not expose fan speed or power over IPMI at all. Vendor-aware parsing is on
the roadmap; see [PLAN.md](PLAN.md).

### Fan percentages look wrong

Converted fan values (shown with `~`) are calculated from the server's "fan max
RPM" (`FAN_MAX_ILO`, `FAN_MAX_IDRAC`, or the value in the server's form). Set
it to your fans' real maximum. The `FAN_MAX_*` variables only affect the
initial server list; change existing servers with **Edit**.

## Graphical (KVM) consoles

### The console page stays black or shows an empty desktop

- Check the engine is running: `docker compose ps` should show `bmc-console` as
  running.
- Check `ILO_IP` or `IDRAC_IP` is set in the engine's environment. The launcher
  refuses to start a console with no IP, and the page still shows the empty
  display. After changing them, run `docker compose up -d`.
- The engine may still be starting; click **Reload**, which restarts the browser
  on the engine and reconnects.

### The console connects, then drops (behind a proxy)

The reverse proxy is not passing WebSockets. Turn WebSocket support on. If it
drops after about a minute of no activity, raise the proxy's read timeout (for
example `proxy_read_timeout 86400s;`). See
[Reverse proxy and TLS](reverse-proxy.md).

### "Your connection is not secure" in the console browser

Expected the first time you open an iLO2 or iDRAC6 console: the image does not
ship a certificate exception for any particular BMC, because each one has its
own self-signed certificate. Accept it once with **Advanced** -> **Add
Exception** (leave "Permanently store this exception" ticked). Firefox saves it
in the console's profile, on the `ilo-profile` (iLO2) or `idrac-profile`
(iDRAC6) volume, so it is kept across restarts and rebuilds.

If the BMC's certificate changes later and the old exception gets in the way,
delete the stored overrides and accept the new certificate once:

```bash
docker exec bmc-console rm /root/.ff52-ilo/cert_override.txt        # iLO2
docker exec bmc-console rm /root/.ffesr-idrac/cert_override.txt     # iDRAC6
```

Then click **Reload** on the console page.

### Java asks "The digital signature cannot be verified" or about running the application

The image trusts the HPE iLO2 applet signing certificate it was built with. If your firmware's applet is signed with a different certificate,
tick "Always trust content from this publisher" and run it once. The decision
is saved on the `icedtea-trust` volume and kept across rebuilds. To build it
into the image, see [Development](development.md#adding-a-bmcs-java-signing-certificate-to-the-image).

### iLO2 applet is blank, or "no JVM detected"

The iLO2 console must run in the engine's Firefox 52 with the IcedTea plugin;
modern Firefox has no NPAPI. If you changed the `Dockerfile`, check the rules in
[Architecture](architecture.md#how-it-works-the-legacy-gauntlet): the plugin's
`.so`, `plugin.jar` and `netx.jar` must all be version 1.5.3 (otherwise
`NoSuchMethodError` on `NetxPanel.<init>`), and the java-7 path symlink must be
in place (otherwise the plugin loads but never starts the JVM). To check the
plugin is installed:

```bash
docker exec bmc-console ls -l /opt/firefox52/browser/plugins/ /usr/lib/jvm/java-7-openjdk-amd64/bin/java
```

### The iDRAC6 virtual console window does not appear

The iDRAC6 console opens as a separate Java Web Start window on the same
display. Launch it from **Console/Media -> Launch Virtual Console** with the
plug-in type set to Java, and give it a few seconds. If it still does not
appear, click **Reload** and try again.

### A server I added on the dashboard has no graphical console

Expected. Only the two servers from `.env` (`ILO_IP`, `IDRAC_IP`) have
graphical consoles. Added servers get the dashboard, power actions and the
text console.

## Text (serial) consoles

### The text console is blank

Normal until the server sends something. Press Enter to wake an existing
getty. For BIOS output you need serial redirection on COM2 in the BIOS/RBSU;
for a Linux login you need a getty on `ttyS1` (`console=ttyS1,115200n8` on the
kernel command line). See [Using SquidBMC](usage.md#getting-a-login-prompt-on-the-text-console)
and the serial-console notes in [PLAN.md](PLAN.md).

### iLO2 SSH console says the server is not in text mode

`textcons` only mirrors VGA text mode. Once Linux switches to a graphical
framebuffer it stops. Booting with `nomodeset` and `GRUB_GFXPAYLOAD_LINUX=text`
keeps the console in text mode (see [PLAN.md](PLAN.md)), or switch to **SOL**.

### `[console session closed]`

The session ended. Either someone opened a text console for the same server
(only one session per server is allowed; the newest one wins), the BMC closed
the connection, or the login to the BMC failed. Click **Reconnect**.

### SSH mode never gets past login

SSH mode logs in to the BMC with the stored username and password and waits
for the BMC prompt (`hpiLO->` on iLO2, `/admin1->` on iDRAC6) before starting
the console. Check the credentials with **Edit** on the dashboard, check SSH is
turned on in the BMC, and that TCP 22 is reachable. Or use **SOL**.

## Login and single sign-on

### Everyone is logged out after every restart

`SESSION_SECRET` is not set, so a new random one is generated at each start.
Set it in `.env` (`openssl rand -hex 32`) and run `docker compose up -d`.

### Login does nothing and I end up on the login page again

You have `TRUST_PROXY=1` but are browsing over plain HTTP. The session cookie
is marked `Secure`, so the browser never sends it back. Either browse through
your HTTPS proxy, or set `TRUST_PROXY=0` for direct HTTP access.

### "Invalid credentials"

Wrong local username or password. Values in `.env` are taken literally, so
quotes or trailing spaces become part of the password.

### "<name> is not authorised"

You signed in at GitHub or your identity provider, but that identity is not on
the allow-list:

- GitHub: add the login to `GITHUB_ALLOWED_USERS`.
- OIDC: add the username or email to `OIDC_ALLOWED_USERS`, or one of the user's
  groups to `OIDC_ALLOWED_GROUPS`. Group names must match exactly, including
  case, and the provider must actually send a `groups` claim. If both lists are
  empty, nobody can sign in through OIDC.

### The identity provider says the redirect URI does not match

The provider compares the redirect URI exactly. `OIDC_REDIRECT_URI` must be
identical to the URI registered with the provider, including `https`, the host
name, any port and the path `/auth/oidc/callback`, and it must be the address
your browser actually uses.

For GitHub, the callback URL registered on the OAuth app must match the
address you browse from (`<base URL>/auth/github/callback`). Behind an HTTPS
proxy set `TRUST_PROXY=1`, otherwise the app sends an `http://` callback URL.

### "Single sign-on failed: ..." or "GitHub sign-in failed: ..."

The provider or the OAuth library returned an error; the text after the colon
is its message. Common causes: a wrong client ID or secret, the `web` container
cannot reach the provider (it fetches
`<OIDC_ISSUER>/.well-known/openid-configuration`), or `OIDC_ISSUER` is not
exactly the issuer the provider publishes.

### There is no login page at all

No login method is configured, so the app runs open. Set `PLATFORM_USER`, or
`GITHUB_CLIENT_ID`, or `OIDC_ISSUER`. See [Authentication](authentication.md).

## Network and access

### I cannot reach port 8088 from another machine

By design: `WEB_BIND` defaults to `127.0.0.1`. Use a reverse proxy on the same
host, or set `WEB_BIND=0.0.0.0` (not advised). See
[Reverse proxy and TLS](reverse-proxy.md).

### My containerised reverse proxy cannot reach the app

`127.0.0.1` inside the proxy's container is the proxy itself. See
[When the proxy runs in a container](reverse-proxy.md#when-the-proxy-runs-in-a-container).

### The pages have no styling

The browser loads the styling from `cdn.tailwindcss.com`. If the browser has no
internet access, the pages work but look unstyled.

## Building

### The build fails while downloading

The images download from `archive.debian.org`, `archive.mozilla.org`,
`github.com`, `registry.npmjs.org` and `rubygems.org`. Check the build host can
reach them; retry later if one of the archives is temporarily unavailable.

### The build fails on an ARM machine

The engine image needs an amd64 (x86-64) host: it installs the x86-64 build of
Firefox 52 and amd64 Debian packages.

### My code change does not show up

The code is copied into the images, not mounted. Rebuild:
`docker compose up -d --build web` (or `console`).
