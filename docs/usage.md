# Using SquidBMC

After signing in you land on the home page, with four entry points: the iLO2
console, the iDRAC6 console, the text (serial) console and the IPMI dashboard.
The same links are in the header, next to your username and **Log out**.

## Dashboard (`/dashboard`)

Live readings and power controls for every server in the list. The page
reloads itself every 20 seconds. Readings come from a background poll of every
BMC over IPMI roughly every 12 seconds, so the page loads from the last
snapshot instead of waiting on the BMCs.

For each server it shows:

- Power state (`on`, `off`, or `unreachable` when the BMC did not answer).
- Power draw, ambient temperature, number of fans reporting OK, and the number
  of temperature and voltage sensors.
- Tables of temperatures and fans, and the last four entries of the system
  event log (SEL) under **Recent SEL events**.

Fan readings can be shown as reported (**Native**), as **%** or as **RPM**.
Converted values are prefixed with `~` and use the server's "fan max RPM". Your
choice is remembered in the browser.

HP iLO2 and Dell iDRAC6 name their sensors differently, so some fields (for
example ambient temperature and fans on the HP) may be missing or show the
first reading instead. Vendor-aware sensor naming is on the roadmap; see
[PLAN.md](PLAN.md).

### Power actions

Buttons under each server: **Power On**, **Soft Shutdown**, **Power Cycle**,
**Reset**, **Force Off**, **Identify LED** (lights the chassis identify LED for
15 seconds) and **Clear SEL**. Power cycle, reset, force off and clear SEL ask
for confirmation first. The result from the BMC is shown in a pop-up.

### Adding, editing and removing servers

- **+ Add server** opens a form: label, BMC IP, vendor (HP, Dell or Other),
  username, password and fan max RPM (default 12000).
- **Edit** opens the same form filled in. The password field is left empty;
  leave it empty to keep the stored password.
- **Delete** removes the server after a confirmation.

Servers you add get the dashboard, power actions and the text console. The
graphical KVM console is only wired up for the two servers seeded from `.env`
(see [Configuration](configuration.md#the-server-list-serversjson)).

## Graphical consoles (`/console/ilo`, `/console/idrac`)

Click a console card and the console launches by itself and is shown inline in
the page through noVNC. Each BMC has its own page and its own virtual display
inside the engine container, so the two consoles are completely separate.

What you see is a browser inside the engine container, already pointed at the
BMC's web interface:

- **iLO2:** log in, then open the **Remote Console** tab. The Java applet runs
  inside the page.
- **iDRAC6:** log in, then **Console/Media -> Launch Virtual Console** (plug-in
  type Java). The console opens in its own window on the same display.

The Java security prompts (unverified signature, HTTPS certificate, "uses
resources from remote locations") are suppressed in advance in the engine
image, so the console starts without clicking through dialogs.

The browser's own "Your connection is not secure" page is different: the first
time you open a console, accept your BMC's self-signed certificate once
(**Advanced** -> **Add Exception**). The exception is kept in the console's
Firefox profile volume, so you are not asked again after a restart or rebuild.
See [Troubleshooting](troubleshooting.md#your-connection-is-not-secure-in-the-console-browser).

Buttons under the console:

- **Reload** - closes and restarts the browser on the engine, then reconnects.
- **Exit** - closes the browser on the engine and returns to the home page.

To attach an ISO, see [Virtual media](virtual-media.md).

## Text (serial) consoles (`/serial`, `/serial/<server>`)

A text terminal in the browser connected to the server's serial console, from
POST all the way to a login shell, with no Java. Pick a server on the left; the
coloured dot shows its power state (green on, red off, grey unreachable). Each
server with a BMC IP also has a **Serial** link on the dashboard.

HP and Dell servers have an **SSH / SOL** switch:

- **SSH** (the default for HP and Dell) - the BMC's own text console over its
  SSH command line.
  - iLO2: `textcons`. It mirrors the server's VGA text screen and repaints when
    you connect. Graphical screens show a "not in text mode" notice instead.
  - iDRAC6: `console com2`. The COM2 serial stream, with the serial history
    buffer replayed when you connect, so you see recent output rather than a
    blank screen.
- **SOL** - raw IPMI Serial-over-LAN (`ipmitool sol activate`). Servers with
  vendor "Other" always use SOL.

Other controls: **Reconnect**, **Exit**, and **Power On**, **Soft Shutdown**
and **Force Off** (with a confirmation), so you can restart a server without
leaving the console.

Only one text session per server is allowed: opening a new one takes over from
the previous one, which shows `[console session closed]`.

### Getting a login prompt on the text console

A blank screen is normal until the server sends something. To get more than
POST and firmware screens, the server itself needs a serial console:

- In the BIOS (Dell) or RBSU (HP), turn on serial redirection on **COM2** at
  115200 baud.
- In Linux, run a getty on `ttyS1`. On systemd, adding
  `console=ttyS1,115200n8` to the kernel command line starts
  `serial-getty@ttyS1` automatically.
- Press Enter to wake an already-running getty.

The exact BIOS menus and a full GRUB example are in the serial-console notes in
[PLAN.md](PLAN.md).
