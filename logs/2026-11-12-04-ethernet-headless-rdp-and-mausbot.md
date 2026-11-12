---
title: "Ethernet, headless RDP and MausBot"
date: 2026-11-12
slug: ethernet-headless-rdp-and-mausbot
tags: [homelab, ethernet, ubuntu, gnome, rdp, grub, adguard-home, openmausbot, tailscale, speedtest, docker, systemd]
status: draft
---

# Ethernet, headless RDP and MausBot

**Log #4 · 11–12 November 2026 (Asia/Dubai)**

Four threads today:

1. Get the GEEKOM on Ethernet, with stable LAN addresses for the boxes that matter.
2. Make Desktop Sharing work with no monitor plugged in, so I can put the mini PC wherever I want.
3. Reach the OpenMausBot instance on the homelab from my Windows PC, and keep it running as a systemd service.
4. Fix `*.homelab` coming back with no ports after a reboot, because Docker was starting before Tailscale had its IP.

> Throughout this log, `<tailscale-ip>` is the homelab's Tailscale address (a `100.x.y.z` IP).

---

## Part 1: The mini PC is on Ethernet

The [GEEKOM IT13 Max](https://www.amazon.ae/dp/B0FWHKYC7P) (`homelab`, Ubuntu 26.04) is finally wired. I'd been putting this off because the PC already used the only convenient Ethernet drop, and a passive cable splitter is not a solution — those don't work at gigabit.

The longer-term plan is a small 5-port unmanaged gigabit switch next to the desk, something like a TP-Link TL-SG105 or LS1005G, so the PC, the mini PC, and whatever else shows up can all share one run. For now I have a cable on the homelab, so I used it.

### What actually came up

```bash
ip -br a
```

- Wired NIC **`enp89s0` = `192.168.1.14`**. That's the one in use.
- Second NIC **`enp88s0`** is unused.
- Wi-Fi **`wlp90s0f0`** is still up on **`192.168.1.78`**. I'll disable that later so the box has one address and stops roaming.

### Static DHCP on the Etisalat router

I reserved the addresses I care about, so they survive a reboot or a power cut:

| Host | Address | Notes |
|---|---|---|
| `homelab` | `192.168.1.14` | The wired NIC |
| `dna-nas` | `192.168.1.18` | Main NAS NIC; the second one is `.25` |
| My PC | `192.168.1.29` | |

---

## Part 2: Speed tests that lied

I wanted a quick sense of the wired link, so I ran `speedtest-cli`. It picked a server in Rostock, Germany (~117 ms away) and reported nonsense: downloads of 2 Mbit/s, then 32 Mbit/s, with upload around 95 Mbit/s. That's not my line, that's a bad server choice plus an ageing CLI.

**Lesson:** the old `speedtest-cli` is unreliable. Pick a local server, or use the official Ookla CLI.

I also looked at running something continuous later:

- **[Speedtest Tracker](https://docs.speedtest-tracker.dev/)** (linuxserver image): a compose stack with an `APP_KEY`, SQLite, a test every 6 hours, UI on port `8765`.
- **[LibreSpeed](https://github.com/librespeed/speedtest)**: local per-device tests, with a telemetry list. Grafana can chart it later if I want history.

Parked for now. I'd rather finish the network shape before I start graphing it.

---

## Part 3: AdGuard on the LAN, but not the whole house

Log #3 bound AdGuard only to the Tailscale IP, which was enough for `.homelab` names on my devices. Now that the box has a stable LAN address, I added the same ports on `192.168.1.14`, still binding to specific IPs because `systemd-resolved` owns `:53` on loopback:

```yaml
ports:
  - "<tailscale-ip>:53:53/tcp"
  - "<tailscale-ip>:53:53/udp"
  - "<tailscale-ip>:3000:3000/tcp"
  - "192.168.1.14:53:53/tcp"
  - "192.168.1.14:53:53/udp"
  - "192.168.1.14:3000:3000/tcp"
```

From Windows, a direct query worked:

```
Resolve-DnsName google.com -Server 192.168.1.14
```

### Why I didn't point the whole house at it

The obvious next step is "make every device use AdGuard." The Etisalat router's DHCP page has **no DNS server field**, so I can't just set AdGuard as the DNS the router hands out.

The alternative is letting **AdGuard itself be the DHCP server** (it needs host networking for that). I skipped it. If the homelab is off, existing leases last until they expire (24 hours), but new or reconnecting devices don't get an IP, and with no DNS nothing loads by name anyway. The house effectively loses internet.

I'd only do that with a second DNS already in place — for example AdGuard on the NAS — so the router (or clients) can fall back. Until then, AdGuard stays available over Tailscale and for any device I point at `192.168.1.14` by hand. Whole-house DNS is deferred.

---

## Part 4: Headless RDP without a dummy plug

I unplugged the monitor and tried to RDP in. Desktop Sharing accepted the login, then hung up. Windows said: **"Your Remote Desktop Services session has ended."**

### First, use the right port

Connecting to `<tailscale-ip>` with no port hits **3389**, which is Remote Login. That still won't serve a session while I'm auto-logged in on the console, which is the whole point of the setup in log #3. The session I want is Desktop Sharing on **`:3390`**.

Checks over SSH:

```bash
ss -ltnp | grep 33
loginctl
grdctl status
```

Both 3389 and 3390 were listening, 3390 belonged to `gnome-remote-desktop`, my user had a `seat0` session, and Desktop Sharing was enabled. From the PC:

```
Test-NetConnection <tailscale-ip> -Port 3390
# TcpTestSucceeded : True
```

So the network path was fine. The hang-up was happening after login.

### The real error

```bash
journalctl --user -u gnome-remote-desktop --since "5 min ago" --no-pager | tail -30
```

The useful line:

```
Failed to record monitor: GDBus.Error:org.freedesktop.DBus.Error.Failed: Unknown monitor
```

Login worked. GNOME just had no screen to share.

### The fix: pretend HDMI is always connected

A cheap HDMI dummy plug (~AED 20) would also fix this, and it's still the fallback. I tried the software version first: tell the kernel the HDMI port is always on at 1080p.

Connectors on this board:

```bash
ls /sys/class/drm/ | grep -i -E "hdmi|dp"
```

```
card1-DP-1
card1-DP-2
card1-HDMI-A-1
card1-HDMI-A-2
card1-HDMI-A-3
card1-HDMI-A-4
card1-HDMI-A-5
```

I used `HDMI-A-1`. In `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash video=HDMI-A-1:1920x1080@60e"
```

The `e` at the end forces the port on, so GNOME thinks a display is there even when nothing is plugged in.

```bash
sudo update-grub && sudo reboot
```

After the reboot, `cat /proc/cmdline` should show the `video=` argument. It did, and **`<tailscale-ip>:3390` worked with no monitor**. No dummy plug needed.

### Saving the RDP login

I don't want to type the Desktop Sharing password every time. Options that store it:

- **MobaXterm**, which I already use for SSH: Session → RDP, host `<tailscale-ip>`, port `3390`, tick "Specify username". It keeps the password in its encrypted store, next to the SSH sessions.
- **mRemoteNG** (free), **Royal TS** (free tier is 10 connections), or **Devolutions Remote Desktop Manager** (also has a free tier).
- Or the plain Windows Remote Desktop app: tick "Allow me to save credentials" and Save As a `.rdp` shortcut.

---

## Part 5: MausBot from the Windows PC

I want to use the OpenMausBot that's already running on the mini PC, from my desk, not a second local install with its own bots and keys.

### What didn't work

The desktop app's **Phones & devices** flow only mints **6-digit codes**, and those are for pairing a phone. One desktop app cannot connect to another that way. A server pairing code is **12 characters** (`XXXX-XXXX-XXXX`) and only exists when OpenMausBot is in dedicated server mode.

Trying the CLI against the already-running desktop app failed in three different ways:

```bash
npx openmausbot serve --tailscale
# something already answers on http://127.0.0.1:8799;
# use npx openmausbot pair against it, or --port for a second server

npx openmausbot pair
# server refused to mint a pairing code:
# forbidden: this change must come from the desktop app or a paired device
```

And the npm package wants **Node 24+**; the homelab is on **Node 22**, so that path is a bad idea anyway.

The desktop app *is* the thing holding port 8799. Pairing codes for a second desktop have to come from server mode, not from `npx` talking to the app.

### What worked

Close the OpenMausBot desktop app on the homelab, then start the **bundled** server with Electron running as Node (so we don't need a newer system Node):

```bash
ELECTRON_RUN_AS_NODE=1 /opt/OpenMausBot/openmausbot /opt/OpenMausBot/resources/server/openmausbot.js serve --lan
```

That prints a 12-character pairing code and a connection link. On the Windows PC, in MausBot: **＋ → Connect to a server**, and paste the link.

That connected. The agents and chats stay on the homelab; the PC is just a client.

There's also a browser option at `http://192.168.1.14:8799` (or `http://<tailscale-ip>:8799`).

> ⚠️ Gotcha: if you start the server in a terminal, closing that terminal stops the server. I found that out the hard way, mid-setup, before I'd even turned on Tailscale access.

### Making it a systemd service

First I stopped the desktop app from coming back and grabbing port 8799: removed `~/.config/autostart/com.openmausbot.app.desktop`, then `pkill`'d the running app. Ubuntu showed a "closed unexpectedly" popup; that's harmless, I killed it on purpose.

Then I installed the bundled server as a service (still with Electron-as-node, same binary as `serve`):

```bash
ELECTRON_RUN_AS_NODE=1 /opt/OpenMausBot/openmausbot /opt/OpenMausBot/resources/server/openmausbot.js service install --tailscale --lan
```

That wrote `~/.openmausbot/openmausbot.service` and printed the `sudo` commands to copy the unit into place, `daemon-reload`, and `enable --now`.

**Gotcha 1: the unit forgot `ELECTRON_RUN_AS_NODE`.** The installed service ran the Electron binary as a normal GUI app, so it crash-looped with `Missing X server or $DISPLAY`, core-dumped, and restarted 40+ times. The fix is a drop-in, not a hand-edit of the generated unit:

```bash
sudo systemctl edit openmausbot
```

```ini
[Service]
Environment=ELECTRON_RUN_AS_NODE=1
```

**Gotcha 2: `--tailscale` was refused** until Tailscale would take commands from my user:

```bash
sudo tailscale set --operator=danish
```

After that, Tailscale Serve published it at `https://homelab.<tailnet>.ts.net`.

**Gotcha 3: Tailscale Serve stole port 443.** Serve bound 443 on the Tailscale IP, which is exactly where Nginx Proxy Manager already lives. The two cannot share that socket.

The stable setup is to drop Serve and just listen on 8799, which is reachable over both the LAN and the tailnet because the server binds all interfaces:

```bash
# reinstall the unit with --lan only, then:
sudo tailscale serve reset
```

The server now listens on `0.0.0.0:8799`. To mint a new pairing code later:

```bash
ELECTRON_RUN_AS_NODE=1 /opt/OpenMausBot/openmausbot /opt/OpenMausBot/resources/server/openmausbot.js pair
# add --phone for the iOS companion
```

Optional nicety: an NPM proxy host `mausbot.homelab` → `host.docker.internal:8799`, with Websockets Support on, so I can use the name instead of the port.

The browser engine is not installed yet (`browser install`). I'll do that when I actually need it.

---

## Part 6: `*.homelab` went dark after a reboot

Log #3 already flagged this as a maybe: if AdGuard or NPM is bound to the Tailscale IP, and Docker starts before `tailscale0` has that IP, the containers come up without their ports. I hadn't seen it yet. Then I rebooted (to test headless RDP, among other things) and every `*.homelab` name broke.

`docker ps` showed `adguard` and `npm` running, but with **no ports**. Inspect agreed, even though compose still had `host_ip` set to `<tailscale-ip>`:

```bash
docker inspect npm --format '{{json .NetworkSettings.Ports}}'
# {}
```

`docker restart` does **not** re-apply those host bindings. The containers have to be recreated: Arcane → Redeploy, or:

```bash
docker compose up -d --force-recreate
```

That brought the names back, but it wouldn't survive the next reboot on its own. Two changes make the bind succeed even if the race is close, and keep Docker behind Tailscale:

**1. A systemd drop-in so Docker waits:**

```bash
sudo systemctl edit docker
```

```ini
[Unit]
After=tailscaled.service network-online.target
Wants=tailscaled.service network-online.target
```

**2. Allow binding an address that isn't on the interface yet:**

`/etc/sysctl.d/99-nonlocal-bind.conf`:

```
net.ipv4.ip_nonlocal_bind=1
```

After that, I recreated the containers one last time, then checked:

```bash
sudo ss -ltnup | grep -E ":(53|80|443|81|8799) "
```

A test reboot later, the ports came back on their own. Useful checks after a boot:

```bash
systemctl is-enabled docker
systemctl status docker
docker ps
```

---

## Config summary

| Thing | Where |
|---|---|
| Homelab wired IP | `192.168.1.14` (`enp89s0`) |
| Homelab Wi-Fi IP (still up) | `192.168.1.78` (`wlp90s0f0`) |
| NAS | `192.168.1.18` (second NIC `.25`) |
| My PC | `192.168.1.29` |
| Desktop Sharing RDP | `<tailscale-ip>:3390` (fake 1080p display via `video=HDMI-A-1:1920x1080@60e`) |
| AdGuard DNS | `<tailscale-ip>:53` and `192.168.1.14:53` |
| AdGuard UI | `http://adguard.homelab` / `:3000` |
| OpenMausBot server | systemd unit `openmausbot`, `0.0.0.0:8799` (`--lan` only; Tailscale Serve left off so it doesn't steal `:443` from NPM) |
| OpenMausBot in the browser | `http://192.168.1.14:8799` / `http://<tailscale-ip>:8799` / `http://mausbot.homelab` (NPM → `host.docker.internal:8799`, websockets on) |
| Docker vs Tailscale boot order | docker drop-in: `After=` / `Wants=` `tailscaled.service` and `network-online.target`; `net.ipv4.ip_nonlocal_bind=1` in `/etc/sysctl.d/99-nonlocal-bind.conf` |

## Next

- Install MausBot's browser engine (`browser install`) when I need it.
- Buy / hook up the 5-port gigabit switch.
- Disable Wi-Fi on the homelab once I'm happy the wired link is boringly reliable.
- Whole-house AdGuard, but only with a second DNS (NAS) so the house isn't one box away from going dark.
- Speedtest Tracker or LibreSpeed, later, when I actually want charts.
- Open PR #3, then this entry as PR #4, when I'm ready to publish.
