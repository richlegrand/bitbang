# BitBang Cookbook

Things people actually do with `bitbang`, with the commands to do them.

Every recipe here has been tested unless marked **community**. If one doesn't work for you, [open an issue](https://github.com/richlegrand/bitbang-cli/issues) — reproducing specific setups can be challenging, so you may be asked to run a test build.

> **Verified against v0.5.0-dev.** `-L` and `-g` are `connect` flags and behave as described. `-L` on `serve` never existed and the recipes no longer use it -- see [what a forwarding listener actually exposes](#what-a-forwarding-listener-exposes). Expiring access is now a per-link property rather than a `serve` flag.

---

## Why use BitBang

Most of what follows can also be done with a mesh VPN. If you already run one for your own machines, that is a fine answer. Six things here are not available that way:

- **No account, no signup, just a URL.** Nothing to create, nothing to log into, nothing that expires because you stopped paying. This is what the rest of the list rests on.
- **The connecting machine can be a browser.** Nothing installed, no client. A phone, a borrowed laptop, a machine you do not administer.
- **Access can be handed in either direction, between strangers.** You give someone a URL, or they give you one. Neither of you joins the other's network, creates an account, or installs anything.
- **Access can expire.** A link that stops working tomorrow, without you remembering to revoke it.
- **It runs on hardware too small for a VPN client.** Microcontroller support is in progress; that is where this is headed.
- **Nothing to set up, and no lock-in either way.** Everything here works out of the box against `bitba.ng` — no server to run, no infrastructure to stand up. And if you would rather not depend on someone else's, the signaling server is open source: run your own and point the CLI at it.

---

## Contents

**Reach a service at home**
- [Mount your home NAS from anywhere (SMB)](#mount-your-home-nas-from-anywhere-smb)
- [Watch your media library from anywhere (Jellyfin)](#watch-your-media-library-from-anywhere-jellyfin)
- [Use your own LLM from anywhere (Ollama, Open WebUI)](#use-your-own-llm-from-anywhere-ollama-open-webui)
- [Check your security cameras (Frigate)](#check-your-security-cameras-frigate)
- [Reach your home automation without exposing it (Home Assistant)](#reach-your-home-automation-without-exposing-it-home-assistant)
- [Check on a 3D print from work (OctoPrint)](#check-on-a-3d-print-from-work-octoprint)
- [Print to your home printer (IPP, CUPS)](#print-to-your-home-printer-ipp-cups)

**Get on a machine**
- [Get a shell on a machine behind NAT](#get-a-shell-on-a-machine-behind-nat)
- [Get a shell from your phone](#get-a-shell-from-your-phone)
- [Remote desktop into a Windows machine (RDP)](#remote-desktop-into-a-windows-machine-rdp)
- [Reach a Linux or Mac desktop (VNC)](#reach-a-linux-or-mac-desktop-vnc)
- [SSH to a machine with no open port (OpenSSH)](#ssh-to-a-machine-with-no-open-port-openssh)
- [Set up a headless Raspberry Pi](#set-up-a-headless-raspberry-pi)

**Share with someone else**
- [Share files without uploading them anywhere](#share-files-without-uploading-them-anywhere)
- [Show someone your project](#show-someone-your-project)
- [Give someone access that expires](#give-someone-access-that-expires)
- [Check your agent session from your phone (Claude Code, tmux)](#check-your-agent-session-from-your-phone-claude-code-tmux)
- [Fix someone else's router](#fix-someone-elses-router)

**Development and devices**
- [Reach a database from your dev machine (Postgres, MySQL)](#reach-a-database-from-your-dev-machine-postgres-mysql)
- [Sync devices that cannot find each other (Syncthing)](#sync-devices-that-cannot-find-each-other-syncthing)
- [Watch a robot from a browser (ROS, Foxglove)](#watch-a-robot-from-a-browser-ros-foxglove)

**Techniques**
- [What a forwarding listener exposes](#what-a-forwarding-listener-exposes)
- [Let other machines on your LAN use a forward](#let-other-machines-on-your-lan-use-a-forward)
- [Known not to work](#known-not-to-work)
- [Contributing a recipe](#contributing-a-recipe)

---

# Reach a service at home

## Mount your home NAS from anywhere (SMB)

Your NAS speaks SMB on the LAN and nothing else. You want it from a coffee shop.

On any Linux machine on the same LAN as the NAS:

```
bitbang serve shell
```

On the machine you're connecting from:

```
bitbang connect <url> -L 4450:nas.local:445
sudo mount -t cifs //127.0.0.1/share /mnt/nas -o port=4450,username=you
```

Windows can't set a client SMB port before 24H2. On 24H2 or later:

```
New-SmbMapping -LocalPath Z: -RemotePath \\127.0.0.1\share -TcpPort 4450
```

On older Windows, run the forward on a Linux box with `-g` and connect to that machine's standard 445 — see [letting other machines use a forward](#let-other-machines-on-your-lan-use-a-forward). The port restriction never applies, because you're connecting to a different host.

Add `-g` to the `connect` command and the forwarded port binds to your LAN rather than loopback, so any machine on your network can use it with no BitBang install of its own. See [letting other machines use a forward](#let-other-machines-on-your-lan-use-a-forward).

---

## Watch your media library from anywhere (Jellyfin)

Jellyfin has no built-in remote access. The usual answers are a reverse proxy with a domain and certificates, a tunnel service with an account, or a VPN client on every device you watch from.

On the machine running Jellyfin, or any machine on its LAN:

```
bitbang serve proxy jellyfin.local:8096
```

Open the printed URL in any browser. Logins, cookies, and streaming all work.

For 4K direct-play, check you're getting a direct connection rather than a relay — relayed video is slow. Bring your own TURN server if you're consistently relayed.

---

## Use your own LLM from anywhere (Ollama, Open WebUI)

You run Ollama and Open WebUI on a machine with a GPU. You want it from your laptop and your phone without putting a chat interface on the public internet.

```
bitbang serve proxy localhost:8080
```

Open the URL and you have your own models from any browser.

Long generations stream continuously, so this is a case where a relayed connection is noticeably worse than a direct one. Same caution as video.

---

## Check your security cameras (Frigate)

Frigate's web UI is on your LAN. The alternatives are exposing it, running a reverse proxy, or a VPN on every device you check from.

```
bitbang serve proxy frigate.local:5000
```

Live views and recorded clips work in the browser.

Camera streams are the most bandwidth-heavy thing in this cookbook. Verify you're direct rather than relayed before leaving this running.

**community**

---

## Reach your home automation without exposing it (Home Assistant)

Home Assistant's remote options are Nabu Casa's subscription, a reverse proxy you maintain, or a VPN. All three work; all three are more setup than this.

```
bitbang serve proxy homeassistant.local:8123
```

The companion mobile apps expect a URL they can reach, so this works best for browser access. If you want the app working too, you still want one of the conventional approaches.

---

## Check on a 3D print from work (OctoPrint)

Use the [OctoPrint plugin](https://github.com/richlegrand/Octoprint-BitBang) rather than a proxy. It gives you the full OctoPrint UI plus hardware-encoded H.264 video from the printer camera, through one shareable URL.

Install from OctoPrint's Plugin Manager. No account, no port forwarding, nothing installed on whatever you're checking from.

---

## Print to your home printer (IPP, CUPS)

Your printer speaks IPP on port 631. Forward it and it appears as a local printer.

```
bitbang serve shell
bitbang connect <url> -L 6310:printer.local:631
```

Then add a printer at `ipp://127.0.0.1:6310/ipp/print` in CUPS or your OS printer settings.

Expect friction. Driverless printing works best; anything needing a vendor driver or relying on mDNS discovery is harder, and discovery will not traverse the link. Worth it mainly when you genuinely need paper to appear at home while you are elsewhere.

Add `-g` to the `connect` command and the forwarded port binds to your LAN rather than loopback, so any machine on your network can use it with no BitBang install of its own. See [letting other machines use a forward](#let-other-machines-on-your-lan-use-a-forward).

**community**

---

# Get on a machine

## Get a shell on a machine behind NAT

No SSH server, no open port, no router configuration.

```
bitbang serve shell
```

From a browser, open the printed URL and you get a terminal in the page. From another machine:

```
bitbang connect <url>                              # interactive
bitbang connect <url> -- tail -f /var/log/syslog   # one-shot
```

`bitbang` runs as an ordinary user. No root, no daemon, no config file.

---

## Get a shell from your phone

Typing a long URL on a phone is unpleasant. Use the pairing code.

```
bitbang serve shell
```

It prints a URL and a 6-digit pairing code, good for 5 minutes. On the phone, open `bitba.ng` and enter the code. Your phone shows a second 6-digit number — read it back and type it on the machine to approve.

The read-back number is a short authentication string computed independently on both ends. A machine-in-the-middle cannot make the two numbers match.

---

## Remote desktop into a Windows machine (RDP)

RDP is already running on the Windows box. It just isn't reachable.

On a machine on the same LAN:

```
bitbang serve shell
```

From a Linux or Mac client:

```
bitbang connect <url> -L 3389:windows-pc:3389
```

Point your RDP client at `127.0.0.1:3389`.

**If the connecting machine is Windows,** it already has 3389 bound by its own RDP service. Use a different local port:

```
bitbang connect <url> -L 13389:windows-pc:3389
```

and connect to `127.0.0.1:13389`. Or run the forward on a Linux box with `-g` and point Windows at that machine's 3389.

Add `-g` to the `connect` command and the forwarded port binds to your LAN rather than loopback, so any machine on your network can use it with no BitBang install of its own. See [letting other machines use a forward](#let-other-machines-on-your-lan-use-a-forward).

---

## Reach a Linux or Mac desktop (VNC)

Same shape as RDP. Any VNC server — TightVNC, TigerVNC, x11vnc, RealVNC — listens on 5900 with no way to be reached from outside.

```
bitbang serve shell
bitbang connect <url> -L 5900:desktop.local:5900
```

Point your VNC client at `127.0.0.1:5900`.

**Note on VNC security.** RFB's built-in authentication is weak and its base form is unencrypted. The BitBang link encrypts the transport end to end, but anyone holding the URL reaches the VNC server. The URL is now your real access control — consider a [temporary link](#give-someone-access-that-expires).

Add `-g` to the `connect` command and the forwarded port binds to your LAN rather than loopback, so any machine on your network can use it with no BitBang install of its own. See [letting other machines use a forward](#let-other-machines-on-your-lan-use-a-forward).

---

## SSH to a machine with no open port (OpenSSH)

Yes, this is SSH over BitBang, and it isn't circular. SSH needs an inbound path; BitBang provides one without a port forward, a static IP, or a VPN.

```
bitbang serve shell
bitbang connect <url> -L 2222:localhost:22
ssh -p 2222 user@127.0.0.1
```

Useful when SSH is already configured and you simply cannot reach the machine — a box behind CGNAT, or on a network where opening a port is not your call.

Add `-g` to the `connect` command and the forwarded port binds to your LAN rather than loopback, so any machine on your network can use it with no BitBang install of its own. See [letting other machines use a forward](#let-other-machines-on-your-lan-use-a-forward).

---

## Set up a headless Raspberry Pi

Fresh Pi, no monitor, and SSH is disabled by default on Raspberry Pi OS. The usual answer is a flag file on the boot partition plus a `userconf.txt` with a hashed password, or remembering the Imager's customization screen.

If you can get `bitbang` onto the image — Imager's first-run script, or one session with a keyboard — then:

```
bitbang serve shell
```

on boot gives you a terminal from any browser, with no SSH configuration, no key provisioning, and without knowing its IP address.

**community**

---

# Share with someone else

## Share files without uploading them anywhere

Files transfer directly from your machine to the recipient. Nothing touches a third-party service.

```
bitbang serve files ~/share                  # read-only
bitbang serve files ~/share -upload          # allow uploads back
```

Browse, preview, download, and upload in the browser. Or copy from the CLI:

```
bitbang cp <url>:/var/log/app.log ./app.log
bitbang cp ./firmware.bin <url>:/tmp/firmware.bin
```

`-` works for stdin and stdout, so `bitbang cp <url>:/f -` streams to a pipe.

---

## Show someone your project

You built something with a web interface and you want to show it to people who are not in your house.

```
bitbang serve proxy localhost:8080
```

Post the URL. They open it and see the live thing, not a screenshot.

Two cautions. The URL is a bearer credential, so anyone who has it gets whatever you are serving — post a read-only link, not a shell. And it works only while your machine is running and the process is up.

---

## Give someone access that expires

A contractor needs one thing for one day. A friend wants to grab files this weekend.

Expiry is a property of a link, not of the listener, so a listener can hand out several
URLs that grant different things and lapse at different times.

Start the listener:

```
bitbang serve files ~/share
```

Then add a link and reload. `bitbang link edit` opens the table in `$EDITOR`:

```json
[
  {"label": "friend", "scope": ["files"], "expires": "2026-08-21T00:00:00Z"}
]
```

Press Enter at the listener's console (or send it SIGHUP). It mints a code for the new
entry and prints the table:

```
  me      files                  https://bitba.ng/pLC8mt...#XTzRmmZiBgw
  friend  files  expires in 2d   https://bitba.ng/pLC8mt...#uJCeY2hcnIc
```

Send the `friend` URL, not the `me` one. `me` is the identity's own code and grants
everything the listener offers; the link you minted grants only what its `scope` lists,
and stops working at `expires` -- including for whoever is connected at the time, whose
session is closed and told why.

`scope` is drawn from `files`, `shell`, `forward`, and `proxy`, intersected with what the
listener actually offers. Omit it and the link grants everything the listener offers.

Deleting an entry and reloading revokes it the same way: `bitbang link rm friend`, then
Enter at the console.

Expiry retires the code rather than pausing it. Extending a lapsed entry mints a fresh
one, so the URL you already sent stays dead and renewing means sending a new link -- which
is what "expires" ought to mean.

This is the thing the alternatives cannot do without adding someone to your network. A mesh VPN share means they install a client and create an account, and the share persists until you manually revoke it.

---

## Check your agent session from your phone (Claude Code, tmux)

Start a long agent task, walk away, watch it finish from wherever you are.

If you are already inside tmux, suspend with Ctrl-Z, then:

```
bitbang serve shell -shell-cmd "tmux attach"
```

`fg` to resume. The remote browser attaches as a second tmux client on the same session, so you see the same terminal.

For a session you want other people watching rather than just yourself, `bitbang share` is
the purpose-built version: run it from inside tmux and it publishes that session with two
URLs, one that can type and one that can only watch, plus `-max-viewers`, `-read-only`,
and a `-ttl` for the share's lifetime. `bitbang share status|stop|rotate` manages it, and
it returns your prompt immediately -- the worker runs on in a detached tmux session, so
the share outlives the terminal you started it from.

One controller at a time, and the newest wins: opening the control URL again takes the
keyboard, and whoever had it is told so and disconnected. That is what makes the control
URL usable from a second machine without first closing the tab on the first one.

If you are not in tmux there is no way to capture a session already in progress — the running program owns the PTY. Start agent sessions inside `tmux new -A` if you want the option later.

Works for any long-running terminal program, not just agents.

---

## Fix someone else's router

Their router's admin page is on their LAN and nowhere else. You are not on their LAN.

Have them run, on any machine in their house:

```
bitbang serve proxy 192.168.1.1
```

They send you the URL. You open it and you are on their router's admin page.

The hard part of remote family tech support is usually the connection, not the fix. When they close the terminal, the access is gone.

---

# Development and devices

## Reach a database from your dev machine (Postgres, MySQL)

Your Postgres is on a private network. You want psql on your laptop.

```
bitbang serve shell
bitbang connect <url> -L 5432:db.internal:5432
psql -h 127.0.0.1 -p 5432 -U you dbname
```

Works for MySQL, Redis, MongoDB, anything speaking TCP.

Use `-pin` and consider a short-lived link — a database port is not something to leave reachable by anyone holding a URL.

Add `-g` to the `connect` command and the forwarded port binds to your LAN rather than loopback, so any machine on your network can use it with no BitBang install of its own. See [letting other machines use a forward](#let-other-machines-on-your-lan-use-a-forward).

---

## Sync devices that cannot find each other (Syncthing)

Syncthing finds peers by local discovery or through public relays. Devices on different networks often fall back to relaying, which is slow.

```
bitbang serve shell
bitbang connect <url> -L 22000:localhost:22000
```

Then add `127.0.0.1:22000` as the peer address in Syncthing's config. Direct rather than relayed, without a VPN.

Add `-g` to the `connect` command and the forwarded port binds to your LAN rather than loopback, so any machine on your network can use it with no BitBang install of its own. See [letting other machines use a forward](#let-other-machines-on-your-lan-use-a-forward).

**community**

---

## Watch a robot from a browser (ROS, Foxglove)

`foxglove_bridge` exposes a WebSocket that Foxglove connects to for live visualization. On a LAN this is easy. Across the internet it is a VPN or a hosted service.

```
bitbang serve proxy localhost:8765
```

Connect Foxglove's browser version to the resulting URL and you get live topics, camera feeds, and 3D views from a robot on someone else's network.

This covers observation and teleoperation, not ROS-to-ROS communication — DDS discovery is multicast and assumes a LAN, and forwarding it over a WAN link is not something to attempt here.

**community**

---

# Techniques

## What a forwarding listener exposes

Worth reading before handing out a URL for any of the forwarding recipes above, because
the listener side is less specific than it looks.

**The listener never names a target.** Only the connector does, in `connect -L`. So there
is no such thing as a listener that forwards one port -- `bitbang serve shell` offers a
shell and TCP forwarding to any host and port that machine can route to, and whoever holds
the URL chooses. The port in the recipe is the connector's choice, not a restriction on
the listener.

**TCP comes with the shell.** The forwarding handler exists only in shell-bearing modes,
on the reasoning that anyone who can run a shell can already reach anything the machine
can. True, but it means the plain URL grants both.

Two ways to hand out less than that:

- **A scoped link.** `forward` is a scope of its own, so a link scoped to it forwards ports
  and cannot open a shell. See [access that expires](#give-someone-access-that-expires) for
  how to mint one; the same table takes `"scope": ["forward"]` with or without an
  `expires`. Such a link is CLI-only -- there is nothing for a browser to render.
- **A PIN.** `-pin` on the listener adds a second factor to everything it serves.

Restricting which hosts and ports a connector may reach is not a control that exists yet.

---

## Let other machines on your LAN use a forward

`bitbang` does not have to run on the machine doing the connecting. Run it on one always-on box — a Pi, a NAS, an old laptop — with the forwarded port bound to the LAN interface, and other devices on that network reach the service with nothing installed.

```
bitbang connect <url> -L 3389:windows-pc:3389 -g
```

`-g` binds the forwarded port to the LAN rather than loopback. Any machine on your network now points its RDP client at `linux-box:3389`.

This is how you get around Windows' SMB port restriction, and how devices that can never run a tunnel client — a smart TV, a printer, a borrowed laptop — reach a remote service.

Each forward is one service at one address. This is not a route to the whole remote network; it is one port made local.

**Two cautions.** The forwarding machine should be Linux, macOS, or a Pi rather than Windows, since Windows binds 445 for its own SMB server. And `-g` exposes the forwarded service to everyone on that LAN. Fine at home. Not fine on a shared or office network.

---

## Known not to work

**Plex.** Its client apps expect to reach the server through Plex's own account infrastructure, and the web UI makes assumptions that do not survive proxying. Jellyfin works; Plex does not.

**Anything needing a stable public hostname.** BitBang gives you an unguessable URL on `bitba.ng`, not a domain with a certificate. If you need `myapp.example.com`, you want a reverse proxy or a tunnel service.

**Unattended devices with no always-on process.** BitBang runs while `serve` runs. If the process dies, the URL is dead until it restarts. Use a systemd unit for anything you expect to be up.

**Mobile apps that hardcode a server address.** Proxying works for browsers. An app expecting `homeassistant.local:8123` will not find it through a BitBang URL.

*Found something else that does not work? [Open an issue](https://github.com/richlegrand/bitbang-cli/issues) — this section is more useful than the ones above.*

---

## Contributing a recipe

Open a PR adding a section to this file. Keep it to the problem, the commands, and any gotchas — if a recipe needs three screens, it is a tutorial rather than a recipe.

Mark it **community** unless it has been verified on more than one setup. Recipes tested by someone other than their author get the mark removed.
