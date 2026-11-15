# Architecture

SquidBMC is a self-hosted control panel for out-of-band management of older
servers whose own tools (ActiveX consoles, NPAPI Java applets, dead TLS
ciphers) no longer run in modern browsers and operating systems. The design
goal is one authenticated entrance: all the legacy software is kept inside a
container, and you get one modern web UI in front of it.

## Why it exists (and why it is hard)

Modern machines cannot drive these BMCs any more:

- iLO2's **ActiveX** console only ever worked in Internet Explorer and is dead.
- The **Java applets** need the **NPAPI** browser plugin, which every current
  browser has removed.
- The BMCs speak **SSLv3 / TLS 1.0 with RC4 / 3DES**, which modern TLS refuses.
- Their **self-signed certificates** expired long ago.

SquidBMC contains all of that in the engine container, so your everyday
machine stays clean and modern.

## Overview

```
                         SquidBMC
  ┌──────────────────────────────────────────────────────────────┐
  browser ──http:8088──▶  web  (Sinatra + Tailwind)
                          │  · IPMI dashboard + power   (ipmitool)
                          │  · serves noVNC 1.4 + xterm.js
                          │  · proxies the VNC + serial WebSockets
                          │
                          ├─ws /websockify/ilo──▶ engine :5901 ─┐  (KVM)
                          ├─ws /websockify/idrac─▶ engine :5902 ─┤
                          │                                      │
                          ├─ws /solws/<node> ─┐  (serial: a PTY  │
                          │                   │   in the web      │
                          │                   ▼   container)      │
                          │        ipmitool SOL / ssh textcons ───┤
        IPMI 2.0 (lanplus) ───────────────────────────────┐      │
                                                           ▼      ▼
                                                    iLO2 / iDRAC6 BMCs
  └──────────────────────────────────────────────────────────────┘
```

Two containers, one Compose project.

### `web` (`bmc-web`, built from `web/Dockerfile`, base `ruby:3-slim`)

The whole UI, and the only thing you open in a browser.

- Sinatra app (`web/app.rb`) under puma, styled with Tailwind (loaded from the
  Tailwind CDN by the browser).
- Runs `ipmitool` for the dashboard and power actions. A background thread polls
  every BMC about every 12 seconds and the dashboard serves that snapshot. Each
  `ipmitool` call uses a 2-second timeout with one retry (`-N 2 -R 1`) and the
  fixed cipher suite from `IPMI_CIPHER_SUITE`.
- Serves modern **noVNC 1.4** (`/novnc`) and **xterm.js 5.5** (`/xterm`),
  downloaded at build time.
- Proxies both the VNC and the serial-console WebSockets in-process
  (faye-websocket, `web/config.ru`), so there is a single origin and a single
  port.
- Text consoles run as a PTY-hosted `ipmitool` or `ssh` process right here in
  `web`; no legacy software is needed for text.
- Stores the server list in `/data/servers.json` on the `servers-data` volume.
- Publishes port 8088 on `${WEB_BIND:-127.0.0.1}`.

### `console` (`bmc-console`, built from `Dockerfile`, base `debian:stretch`)

The legacy KVM engine. Nothing is published to the host; only `web` reaches it,
over the Compose network.

- **One Xvnc display per BMC**: `:1` on port 5901 for iLO2 and `:2` on port 5902
  for iDRAC6, each 1280x1024, so the consoles are fully isolated.
- Each display runs a kiosk window manager (matchbox, no title bars), and the
  browsers hide their toolbars, so the console fills the screen: no desktop, no
  title bars, no terminal.
- **Firefox 52 ESR + IcedTea NPAPI plugin** for iLO2's in-browser applet
  (`ilo2` launch script, profile in `/root/.ff52-ilo`).
- **firefox-esr + javaws** for iDRAC6's virtual console (`idrac6` launch script,
  profile in `/root/.ffesr-idrac`).
- A tiny launch listener (`launchd.py`, Python 2, internal port 9000) that `web`
  calls with `POST /launch/<node>`, `/kill/<node>` or `/relaunch/<node>`. Opening
  a console page triggers `launch`; the **Reload** and **Exit** buttons trigger
  `relaunch` and `kill`. Consoles are only ever started this way, never from a
  shell.
- Xvnc runs without a VNC password (`-SecurityTypes None`). Access control is the
  login in front of `web`'s WebSocket proxy, plus the fact that the VNC ports are
  only on the Compose network.

## Request flows

- **Graphical console:** the browser loads `/console/<key>`. The app tells the
  engine's launcher to start that browser, then embeds noVNC, which opens a
  WebSocket to `/websockify/<key>`. `config.ru` checks the session and bridges
  the WebSocket to `console:5901` or `console:5902` with a raw TCP socket.
- **Text console:** the browser opens `/serial/<id>`; xterm.js opens a WebSocket
  to `/solws/<id>?via=ssh|sol`. `config.ru` checks the session and spawns the
  console process on a PTY, then relays bytes both ways.
- **Dashboard and power:** plain HTTP to the Sinatra app, which runs `ipmitool`
  over IPMI 2.0 (lanplus) to the BMC.

## How it works (the legacy gauntlet)

The awkward details, written down so a rebuild never turns into archaeology:

- **Engine base is `debian:stretch`** (via `archive.debian.org`, with the
  `Valid-Until` check skipped): the last Debian carrying `openjdk-8` +
  `icedtea-netx` + `firefox-esr` together. Needs `bzip2` installed with apt or
  the Firefox 52 `tar xjf` fails.
- **Firefox 52 ESR is fetched in a `debian:bookworm-slim` build stage** and
  copied into the stretch image with `COPY --from`. Stretch's own wget/curl
  cannot negotiate archive.mozilla.org's TLS, so a modern stage does the
  download. This keeps the repository self-contained (no host-side tarball to
  stage before building). Firefox 52.9 ESR is the last release with NPAPI, which
  iLO2 needs because its console is an in-browser `<applet>`, not a JNLP file
  that javaws could start.
- **iLO2 NPAPI plugin** is grafted from jessie's icedtea-web **1.5.3**: the
  native `IcedTeaPlugin.so` **and both** `plugin.jar` + `netx.jar`. All three
  must be the same version or the applet throws `NoSuchMethodError` on
  `NetxPanel.<init>`. The `.so` hardcodes a java-7 path, so a symlink points
  `/usr/lib/jvm/java-7-openjdk-amd64` at java-8's JRE. Without it the plugin
  loads but never starts the JVM (a blank applet).
- **`java.security`** has its `jdk.tls`, `jdk.certpath` and `jdk.jar`
  `disabledAlgorithms` lists blanked to re-enable the dead crypto the BMCs
  speak.
- **Firefox TLS preferences** re-enable TLS 1.0, 3DES and RC4 for both
  browsers. Both also disable the DHE ciphers to force RSA key exchange; for
  the iDRAC6 the reason given is that its DHE handshake takes about 4 seconds
  against about 1.5 seconds with RSA.
- **Certificate exceptions are pre-seeded**: each launch script writes a
  `cert_override.txt` into its Firefox profile if none exists, for a known BMC
  certificate. The profiles are on volumes, so a certificate exception you
  accept once is kept across rebuilds. If a BMC's certificate changes, delete
  that file and accept the certificate once to regenerate it.
- **Java security prompts are pre-cleared** in the engine image so consoles
  launch without dialogs: `deployment.properties` sets
  `security.level=ALLOW_UNSIGNED`, `manifest.attributes.check=NONE` (the "uses
  resources from remote locations" check), `security.mixcode=DISABLE`, and
  `itw.ignorecertissues=true` (the HTTPS "certificate cannot be verified"
  dialog), and turns off CRL/OCSP checks and the expired and not-in-CA warnings.
  A pre-seeded IcedTea trust store (`trusted.certs`, JKS, captured after
  ticking "Always trust this publisher" for the HPE iLO2 and Dell iDRAC6 signing
  certificates) is baked in, and the runtime `security/` directory is on the
  `icedtea-trust` volume so new trust decisions persist across rebuilds.
- **noVNC 1.4** is served by Sinatra. The WebSocket proxy uses
  **faye-websocket** in `rack.hijack` mode under **puma** with a raw `TCPSocket`
  bridge (no EventMachine reactor), and the app runs in **production** mode so
  `Rack::Lint` does not return a 500 on the WebSocket upgrade.
- **Serial consoles** bridge a browser xterm.js to a PTY-hosted console process
  in the `web` container over `ws /solws/<node>` (login required). SSH mode logs
  into the BMC CLI and types its console command once the prompt appears
  (`textcons` at `hpiLO->`, `console -h com2` at `/admin1->`), with the BMCs'
  legacy SSH crypto re-enabled through `ssh -o` options; the BMC host key is not
  checked. SOL mode runs `ipmitool sol activate`, with `sol deactivate` before
  connecting and after closing so stale sessions do not block the next one. One
  session per node; a new connection takes over. The terminal is advertised as
  `xterm` at 80x25, and the browser drops cursor-position replies so they do not
  reach the login prompt as stray characters.
- **One Xvnc display per BMC** gives each console total isolation.

The engine image also contains Debian's `novnc` and `websockify` packages and a
copy of noVNC 1.1.0 in `/opt/novnc`. `start.sh` does not start them; the
browser-facing noVNC is the 1.4 copy in the `web` container.

## Ports

- `8088/tcp` on `web` - the UI, published on `WEB_BIND` (default `127.0.0.1`).
- `5901/tcp`, `5902/tcp` on `console` - Xvnc for iLO2 and iDRAC6, Compose
  network only.
- `9000/tcp` on `console` - the launch listener, Compose network only.

Outbound from the containers to the BMCs: IPMI (UDP 623) and SSH (TCP 22) from
`web`; HTTPS and the BMC's console and virtual media ports from `console`.

## Limitations

- The graphical consoles are wired to exactly one iLO2 (`ILO_IP`) and one iDRAC6
  (`IDRAC_IP`). Servers added through the dashboard get the dashboard, power
  actions and the text console only.
- Sensor parsing is not vendor-aware yet, so some HP readings are missing.

See the roadmap in the [documentation index](README.md#roadmap).
