---
title: "Setting up the GEEKOM"
date: 2026-11-09
slug: setting-up-the-geekom
tags: [homelab, geekom, ubuntu, docker, portainer, cockpit, arcane, komodo, beszel, uptime-kuma, cloudflare, tunnel, access]
status: published
---

# Setting up the GEEKOM

**Log #2 · 9–10 November 2026 (Asia/Dubai)**

The [GEEKOM IT13 Max](https://www.amazon.ae/dp/B0FWHKYC7P) showed up the **same day** I ordered it. This log is the install path: from sealed box to a Ubuntu Desktop host ready for Docker, then the first services I stood up on it — Beszel, Uptime Kuma, a family status page, and a Cloudflare Tunnel with Access.

Log #1 covers *why* this machine. This one is *how* I stood it up.

---

## Where I am right now (9 Nov 2026, evening)

### Done

1. **Unboxed the GEEKOM IT13 Max** (24GB LPDDR5 / 500GB SSD / Core Ultra 9 185H).
2. **Downloaded Ubuntu 26.04.1 Desktop** — the **Intel / amd64** ISO (not an ARM build).
3. **Made a bootable USB with [Rufus](https://rufus.ie)** from that ISO.

That’s the complete “done” list for this entry so far. Everything below is the planned path I’ll fill in as I actually do it.

---

## In progress / planned next steps

I’ll tick these off and expand each section with what actually happened (including mistakes).

### 1. BIOS check ✅ (10 Nov 2026, ~12:10 AM)

Before wiping anything I went into the BIOS (AMI Aptio). Tapping `F7` at boot brings up the boot menu; **Enter Setup** from there opens the BIOS.

**What the Main page showed:**
- BIOS version: `IT13.Max 0.28` (build date 13 May 2026), EC 1.15
- CPU: Intel Core Ultra 9 185H
- Memory: **24576 MB (LPDDR5)** = the full 24GB ✅
- Boot menu listed the stock SSD as **Kingston SNV3S500G** (500GB NV3) with Windows Boot Manager on it

**What I changed (Advanced → Power Management Configuration):**

| Setting | Default | Changed to | Why |
|---|---|---|---|
| Power-On after Power-Fail | Power Off | **Power On** | Box comes back by itself after a power cut. Essential for an always-on server |
| Wake Up by LAN | Disabled | **Enabled** | Lets me wake it remotely if it's ever shut down (Ethernet only, not Wi-Fi) |
| PowerOn by RTC Alarm | Disabled | Left disabled | Scheduled power-on is pointless for a 24/7 box |
| EuP Function | Disabled | Left disabled | EuP's deep power-saving mode can kill Wake-on-LAN |

Saved with `F4` (Save & Exit).

**Gotcha:** the BIOS clock was way off (showed 13:07 when it was past midnight in Dubai). I ignored it, since Ubuntu syncs time over the network.

**Decision: wiping Windows.** It came preinstalled on the Kingston SSD. That's safe: the Windows license is tied to the board, so a clean reinstall would auto-activate, and drivers are on GEEKOM's site. The only thing lost for good is GEEKOM's factory recovery image, which I'm fine with on a homelab box.

### 2. Install Ubuntu Desktop 26.04.1 ✅ (10 Nov 2026, started ~12:15 AM, finished overnight)

Booted the Rufus USB from the `F7` boot menu (**UEFI: USB, Partition 1**) and picked "Try or Install Ubuntu".

**Installer choices and why:**

| Step | What I picked | Why |
|---|---|---|
| Installation type | **Interactive** | "Automated with Landscape" is for mass rollouts from a config file. Not needed for one box |
| Proprietary software | **Ticked both** (third-party graphics/Wi-Fi drivers + media codecs) | Extra firmware and codecs, no downside on a desktop install |
| Disk | **Erase disk and install Ubuntu**, no encryption | No dual-boot hassle on a server; no encryption so it can reboot unattended after a power cut |
| Name / hostname / user | Danish / `homelab` / `danish` | Simple, and `homelab` becomes the Tailscale name too |
| Password | Strong password, login required | Only guards the desktop login. Docker, Tailscale and Cockpit start at boot before anyone logs in |

**Gotcha: the install crawled over Wi-Fi.** Because I was on Wi-Fi (no Ethernet cable long enough) with the third-party/codec boxes ticked, the installer started pulling updates, including the new HWE kernel 7.0.0-43 (221MB total), from the UAE mirror `ae.archive.ubuntu.com`. Around 12:20 AM it slowed to a crawl: a 2.4MB package took over a minute, and the log sat on a single 170MB `linux-modules` download for 15+ minutes. It wasn't frozen, just very slow.

**Lesson:** for a quick install, either use Ethernet or choose **"Do not connect to the internet"** and leave the driver/codec boxes unticked. The install then runs entirely off the USB in about 10 minutes, and updates can be done afterwards (switching to a faster mirror if needed).

**What I did:** it was ~12:40 AM, so I left it downloading overnight and will finish in the morning.

### Plan for tomorrow morning (~15–20 min before work)

1. Click **Restart now**, pull out the USB, log in
2. Install Tailscale: `curl -fsSL https://tailscale.com/install.sh | sh` then `sudo tailscale up --ssh`, and **disable key expiry** for `homelab` in the Tailscale admin console
3. Settings → Power: **Automatic Suspend off**, Screen Blank never (Ubuntu Desktop sleeps on idle by default, which would kill remote access)
4. Settings → System → Remote Desktop → **Remote Login** on, with its own RDP username/password
5. Test from my phone on mobile data, then do the rest (full upgrade, Cockpit, Docker, Portainer) remotely from the office

### Morning session: remote access ✅ (10 Nov 2026, ~6:25–6:51 AM)

The install had finished overnight. I clicked **Restart now**, pulled the USB, and logged in.

**Order I did things in:** I went with remote access first and deferred the full `apt full-upgrade`. The UAE mirror had been slow the night before, and I didn't want to burn the morning on updates when I could run them from the office.

**Tailscale**

- **Gotcha:** `curl` isn't installed on a fresh Ubuntu 26.04 Desktop. I fixed it with `sudo apt update && sudo apt install -y curl`.
- **Gotcha:** "curl: could not resolve host" on the first try. It was a typo or Wi-Fi not being up yet. Retyping the URL fixed it.
- Then `curl -fsSL https://tailscale.com/install.sh | sh` and `sudo tailscale up --ssh`, signed in through the link, and disabled key expiry for `homelab` in the admin console.
- `tailscale ssh danish@homelab` is now a backup way in if remote desktop ever breaks.

**Power:** Settings → Power, **Automatic Suspend off**, Screen Blank **Never**.

**Remote desktop (RDP)**

- Settings → System → Remote Desktop → **Remote Login** tab (not Desktop Sharing). Remote Login runs its own login session, so it works after a reboot with nobody logged in. Desktop Sharing only mirrors a session that's already open.
- **Gotcha: "This computer can't connect to the remote computer."** Ping over Tailscale worked, so the network was fine. `sudo systemctl status gnome-remote-desktop` showed:
  - `Init TPM credentials failed because Failed to initialize transmission interface`
  - `[RDP] Credentials are not set, denying client`
- Cause: GNOME Remote Desktop stores the RDP password in the TPM. Its service user wasn't allowed to talk to the TPM, so the credentials from the Settings app never got saved.
- I checked the BIOS (Advanced → TPM Configuration). **Security Device Support was already Enabled**, so the BIOS wasn't the problem.
- **Fix:**

```bash
sudo usermod -aG tss gnome-remote-desktop
sudo systemctl restart gnome-remote-desktop
sudo grdctl --system rdp set-credentials danish '<rdp password>'
```

- After that, connecting from my phone over Tailscale worked ✅
- Fallback if this ever breaks again: `xrdp` (`sudo apt install -y xrdp`) with Remote Login turned off.

**Before leaving the house:** do a reboot test, then connect from the phone on mobile data without touching the box.

**Lesson:** on Ubuntu 26.04, if RDP refuses connections and the logs say "Credentials are not set", check the TPM error first. Adding the service user to the `tss` group and setting the credentials with `grdctl` fixes it.

### Office session: full upgrade + first remote reboot (10 Nov 2026, ~8:20–8:41 AM)

From the office over RDP:

1. Software & Updates → "Download from" → **Main server** (the UAE mirror was slow the night before)
2. `sudo apt update && sudo apt full-upgrade -y`, then `sudo reboot`
   - I used `sudo -s` to stop retyping the password. That's fine, but `exit` before any step that uses `$USER` (like `usermod -aG docker $USER`), or it targets root instead of `danish`.
3. Before rebooting, `systemctl is-enabled tailscaled gnome-remote-desktop` should print **enabled** for both, so they come back on their own.

**Gotcha: RDP broke after the reboot even though Tailscale showed the box online.** The status log was full of:

```
[RDP] Couldn't retrieve RDP credentials: Esys_ContextL…failed
```

GNOME Remote Desktop couldn't read the saved password back out of the TPM after the reboot.

### 🔧 How to fix RDP after a reboot (my reference)

If remote desktop won't connect after a restart:

1. Get in over SSH instead. From Windows Terminal or PowerShell on any machine on my tailnet:
   ```
   ssh danish@homelab
   ```
   Tailscale SSH may ask me to approve a login link the first time. Note: once I'm in, the prompt is `danish@homelab:~$`. Don't run `ssh` again from there, because that connects to itself and fails with "Connection refused".
2. Confirm the cause:
   ```bash
   sudo systemctl status gnome-remote-desktop --no-pager
   ```
   Lines saying "Couldn't retrieve RDP credentials" mean it's this issue.
3. Re-set the credentials and restart:
   ```bash
   sudo systemctl daemon-reload
   id gnome-remote-desktop            # should list tss
   sudo grdctl --system rdp set-credentials danish '<rdp password>'
   sudo systemctl restart gnome-remote-desktop
   ```
4. Connect with RDP again. ✅ Worked at 8:41 AM.

**Why it's worth knowing:** Tailscale SSH is the safety net. As long as Tailscale is online I can always get a terminal and fix the GUI side. **Possible permanent fix later:** switch to `xrdp`, which doesn't use the TPM, if this keeps happening on every reboot.

### Cockpit, Docker and three container managers ✅ (10 Nov 2026, ~8:42–9:11 AM)

**Cockpit** (web dashboard for the OS): `sudo apt install -y cockpit`, then open `http://homelab:9090` and log in as `danish`. Click **Turn on administrative access** to manage services and updates. It showed the GEEKOM IT13 Max with 22 CPU threads, 22 GiB usable RAM, and "System is up to date".

**Docker:**
```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker danish   # then log out/in
docker run hello-world
```

**Why three managers?** I wanted to move toward fully open source (FOSS) and wasn't sure Portainer was the best fit, so I'm running them side by side for a while and keeping the one I like:

| Tool | Port | Why it's in the running |
|---|---|---|
| Portainer CE | `https://homelab:9443` | Most polished GUI, the one everyone knows; core is open source but some features are paid (Business Edition) |
| Arcane | `http://homelab:3552` | Newer, fully FOSS, very clean modern UI, single container with built-in SQLite |
| Komodo | `http://homelab:9120` | Fully FOSS, built for many servers, git-backed stacks, needs a database (Mongo) |
| Dockge (skipped) | n/a | Simplest compose UI, but weak for multiple nodes and I'll have 4 |

Everything lives under `/opt/docker` (one folder per stack), which is tidy and easy to back up.

**Portainer:**
```bash
docker volume create portainer_data
docker run -d -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data portainer/portainer-ce:latest
```
Gotcha: you have to create the admin account within about 5 minutes of starting, or it locks itself and needs a container restart.

**Arcane:** `/opt/docker/arcane/compose.yaml` with the official image `ghcr.io/getarcaneapp/manager:latest` on port 3552, the Docker socket mounted, `PROJECTS_DIRECTORY=/opt/docker`, and an `ENCRYPTION_KEY` from `openssl rand -hex 32`. Then `docker compose up -d`. First login is `arcane` / `arcane-admin`, and it forces a password change.

**Komodo:** downloaded the official `mongo.compose.yaml` and `compose.env` into `/opt/docker/komodo`, then edited `compose.env`:

| Variable | Changed to | Why |
|---|---|---|
| `KOMODO_DATABASE_PASSWORD` | random | Default is `admin` |
| `KOMODO_INIT_ADMIN_PASSWORD` | strong password | Default is `changeme` |
| `KOMODO_WEBHOOK_SECRET` | random (`openssl rand -hex 24`) | Default placeholder |
| `KOMODO_JWT_SECRET` | random | Signs login sessions |
| `KOMODO_HOST` | `http://homelab:9120` | Used for webhook/login links (in nano: Ctrl+W to find it) |
| `TZ` | `Asia/Dubai` | Default is UTC |
| `KOMODO_DISABLE_USER_REGISTRATION` | `true` | Nobody else can sign up |

```bash
docker compose -p komodo -f komodo/mongo.compose.yaml --env-file komodo/compose.env up -d
```
Log in with `admin` (from `KOMODO_INIT_ADMIN_USERNAME`) and the init password.

**Result (9:11 AM):** all three running, and I'm signed in to each. Next I pick a favourite after using them for a bit.

### Quick Sync check ✅ (10 Nov 2026, 9:12 AM)

`ls /dev/dri` showed `by-path  card1  renderD128`. `renderD128` is the Intel Arc render node, so Plex/Jellyfin can hardware-transcode once I pass `/dev/dri` into their containers.

### NAS shares over NFS ✅ (10 Nov 2026, ~9:26–9:36 AM)

**Decision:** I'm not moving Plex/Jellyfin yet; I'm starting simple. I'm only connecting the shares now, and no apps use them yet.

**Which shares:** only **Media** (Volume 1) and **Media - Expansion** (Volume 3). I skipped the `docker`/`docker-ssd` shares because the GEEKOM keeps its own container data on its own SSD. I skipped `Torrents` because it's empty on the NAS and the real torrents will live on the homelab laptop, which is better anyway since downloads hammer the disk.

**On the Synology (DSM):**
1. Control Panel → File Services → NFS → **Enable NFS service**, max protocol **NFSv4.1**
2. Control Panel → Shared Folder → (share) → Edit → NFS Permissions → Create:
   - Hostname/IP: `192.168.1.0/24`
   - Privilege: Read/Write
   - Squash: Map all users to admin
   - ✅ Allow connections from non-privileged ports
   - ✅ Allow users to access mounted subfolders

**On the GEEKOM:**
```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/nas/media /mnt/nas/media-expansion
sudo tee -a /etc/fstab <<'EOF2'
192.168.1.18:/volume1/Media /mnt/nas/media nfs4 defaults,_netdev,nofail,x-systemd.automount 0 0
192.168.1.18:/volume3/Media\040-\040Expansion /mnt/nas/media-expansion nfs4 defaults,_netdev,nofail,x-systemd.automount 0 0
EOF2
sudo systemctl daemon-reload
sudo systemctl restart remote-fs.target
sudo mount -a
```

**What the fstab options do:**
- `_netdev`: wait for the network before mounting
- `nofail`: don't hang the boot if the NAS is off
- `x-systemd.automount`: connect on first access

**Gotchas:**
- Paths are **case-sensitive**: it's `/volume1/Media`, not `media`.
- In fstab, a space is written `\040` ("Media - Expansion").
- I organised everything under `/mnt/nas/<share>` so it's obvious what's remote.
- Right after `daemon-reload` the folders looked **empty**. It wasn't broken: the automount wasn't active yet. `systemctl restart remote-fs.target` plus `mount -a` fixed it.

**Result:** `/mnt/nas/media` shows Bazarr, Jellyfin, Manual Downloads, Media Server Poster and TdarrCache. `/mnt/nas/media-expansion` shows Emulation, Homelab-Laptop, Immich, Jellyfin, SSD Backup, TdarrCache and Torrents ✅

The numbered plan below is the original evening checklist. The morning sessions above already covered remote desktop, Cockpit, Docker, Portainer/Arcane/Komodo, the Quick Sync check, and NFS. I'm leaving the plan here for the record. Ethernet + the DHCP reservation are still waiting on a longer cable.

### 3. Static DHCP on the Etisalat (e&) HomeBox

> **Change of plan (9 Nov, ~11:58 PM):** I didn't have a long enough Ethernet cable, so I'm setting up over **Wi-Fi** for now (the Intel BE200 Wi-Fi 7 card works out of the box) and using **Tailscale** for a stable address/name instead of a static DHCP lease. I'll move to Ethernet + the reservation below once I get a longer cable, since NFS and Plex streaming will be much faster wired.


Router UI path we already found:

> **Settings → LAN → Static DHCP - Home Network**

- Add the GEEKOM (Ethernet MAC, not Wi‑Fi)
- Target IP: **`192.168.1.20`** (NAS is already reserved at **`192.168.1.18`**)
- Apply, reboot the mini PC

Also: the HomeBox **firewall** toggle was off in one screenshot — worth turning back on; the mini PC doesn’t need it disabled.

### 4. Remote desktop

- Enable Ubuntu’s built-in **Remote Desktop** (Settings → System → Remote Desktop)
- Unplug the monitor and connect from the Windows PC

### 5. Cockpit, Docker, Portainer

Planned commands (will confirm versions / any 26.04 quirks when I run them):

```bash
sudo apt install -y cockpit
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
sudo docker volume create portainer_data
sudo docker run -d -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data portainer/portainer-ce:latest
```

Then:

- Cockpit → `http://192.168.1.20:9090`
- Portainer → `https://192.168.1.20:9443` (browser cert warning is normal on first visit)

### 6. Confirm Intel Arc / Quick Sync device nodes

```bash
ls /dev/dri
```

Looking for **`renderD128`**. That’s the green light for passing `/dev/dri` into Plex/Jellyfin containers later.

---

## Why this order

| Step | Why |
|------|-----|
| BIOS first | Catch a bad RAM/SSD report before investing install time |
| Ubuntu Desktop (not Server / not Windows) | GUI comfort + remote desktop; Docker can use `/dev/dri` properly on Linux |
| Static DHCP early | Cockpit/Portainer URLs shouldn’t move |
| Cockpit + Portainer before migrating apps | Get a clickable control plane before touching the Synology stacks |
| `/dev/dri` check before Plex move | No point migrating media apps if HW transcode isn’t visible |

---

## First services, a status page, and a tunnel (10 Nov 2026, morning–afternoon)

With the box online, NFS mounted, and three container managers to try, I spent the rest of the morning standing up monitoring and a public status page, then put a Cloudflare Tunnel in front of it. Times below are Asia/Dubai.

### Beszel via Arcane ✅ (10 Nov 2026, ~9:40–10:00 AM)

This was the first stack I created in **Arcane**, partly to learn the UI.

**What I deployed:** a Beszel hub + agent compose. The agent talks to the hub over the unix socket `/beszel_socket/beszel.sock`. Data lives in `./beszel_data`. The hub is on port **8090**. The `KEY` and `TOKEN` come from the hub's **Add System** dialog (they stay in the local `.env`, not in git).

**Gotcha:** Arcane's sample `.env` included an `NGINX_HOST_PORT` leftover that I had to clear before the stack would come up cleanly.

**Decision: start fresh**, rather than port the old Beszel hub from the NAS (that one is on port 8095). The new hub is only monitoring **homelab** for now. The NAS and laptop agents stay on the old hub until those machines migrate. Long-term data / backups for Beszel are deferred.

I later published the hub at `beszel.danishandarwa.com` (see the tunnel and Access sections below).

### Uptime Kuma via Komodo ✅ (10 Nov 2026, late morning)

Image: `louislam/uptime-kuma:2`, on port **3001**.

The laptop already had a Kuma instance, but it only had three monitors, so I treated it as a duplicate and **recreated the monitors** on the new one rather than migrating that config.

**Monitor groups:**

| Group | Monitors |
|---|---|
| Homelab | Homelab Ping TS |
| Laptop | Ping TS, Ping LAN |
| NAS | NAS Ping TS, Jellyfin, Plex LAN, NAS Ping LAN |
| Network | Internet, Router |

I split **Tailscale vs LAN pings** on purpose, so I can tell whether Tailscale is the problem or the LAN itself is down.

**Gotcha:** some LAN monitors showed faded / paused, with **Resume** greyed out. Likely a UI glitch — a refresh, or edit-and-save, brought them back.

**Gotcha:** reaching `100.x` Tailscale IPs from inside a container may need `network_mode: host`.

**Status page:** I created one called **Home Media** (slug `family`) with Plex, Jellyfin and Internet, so Arwa can see whether the media stack is up. **Create incident** is just a manual banner on that page. Maintenance windows exist too (including recurring ones, e.g. Sunday 3–4 AM for updates) — noted for later. Notifications (Telegram / Discord / etc.) are also deferred.

### Proxmox: considered, then rejected

I thought about wiping the Ubuntu install and running Proxmox, with separate VMs for media, management, services and experimental stacks. I decided against it:

- It would **wipe the Ubuntu install** I just finished
- **iGPU passthrough** for Quick Sync is harder than just passing `/dev/dri` into a container
- VMs **reserve RAM** even when idle, and I've only got 24GB

**Instead:** use Docker resource limits (`cpus:`, `mem_limit:`) so something like Jellyfin can't starve n8n. If I ever need a real VM, I can add virt-manager or Cockpit Machines later for occasional ones, without turning the whole box into a hypervisor.

### Cloudflare Tunnel ✅ (10 Nov 2026, ~11:30 AM–12:15 PM)

I wanted a public URL for the family status page (and later for admin tools) without opening ports on the Etisalat router.

**Why a tunnel instead of orange-cloud DNS / port forwarding:**

| Upside | Downside |
|---|---|
| No open ports on the router | Everything depends on the `cloudflared` container staying up |
| The home IP stays hidden | Same Cloudflare streaming ToS grey area as orange-cloud DNS (relevant later for Plex / Jellyfin) |
| No dynamic DNS, and no breakage when the IP changes | `*.home` internal names still need a local proxy |
| Works behind CGNAT | |
| Easy to put Cloudflare Access in front | |

**Steps:**

1. Zero Trust → Networks → Tunnels → Create. Type **Cloudflared**, name **`homelab`**, install method **Docker**. Copy the token only (it lives in a local `.env`, never in this log or in git).
2. A `cloudflared` stack in **Komodo**:

```yaml
# shape of the compose — token stays in .env
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    environment:
      TUNNEL_TOKEN: ${TUNNEL_TOKEN}
    network_mode: host
    restart: unless-stopped
```

The tunnel showed **Healthy**.

**Gotcha:** the new Cloudflare UI no longer says "public hostnames". They're under **Routes** → **Add route** → **Published application**.

**Gotcha:** the service URL **can't contain a path**. `http://localhost:3001/status/...` was rejected. The fix: point the hostname at `http://localhost:3001`, then in Kuma set `status.danishandarwa.com` under the status page's **Domain Names**, so the root of that hostname opens the status page.

**Result:** `status.danishandarwa.com` is live. I also added `beszel.danishandarwa.com` → `http://localhost:8090`.

### Cloudflare Access ✅ (10 Nov 2026, early afternoon)

The status page should stay public for the family. The Kuma admin UI, and all of Beszel, should not.

**Status page app:** Access → Applications → Add → Self-hosted → Public DNS. Destinations on `status.danishandarwa.com` with paths `dashboard*`, `settings*`, `manage-status-page*`, `add*`, `edit*`, `maintenance*` — so the public status page itself stays open. Policy **Only Me**: Allow, include Emails = my email, session **1 week**. Login is the default one-time PIN emailed by Cloudflare.

**Gotcha:** Kuma's login runs over a **websocket**, so path-based protection isn't airtight. I'm keeping Kuma's own password as a second lock.

**Beszel:** a separate Access app on `beszel.danishandarwa.com` with **no path** (the whole domain is locked), using the same **Only Me** policy.

I looked at Cloudflare's built-in MFA (security key / authenticator app, enrolled via the App Launcher) and at adding Google, Microsoft or GitHub as identity providers so I could skip the email PIN. That's a lot of extra verification for one status page, so I left it at the PIN with a 1-week session. Worth revisiting when Portainer, DSM and the other admin UIs go behind Access — one login would then cover everything.

### Plans / next up

**What's where today**

On the **NAS:** Plex, Jellyfin, the \*arr stack (Radarr, Radarr-4k, Sonarr-Anime, Sonarr-Shows, Prowlarr, Lidarr, Bazarr, Cleanuparr, Seerr, pulsarr), n8n + Postgres, Home Assistant, esphome, matter-server, PocketBase, Actual Budget (+ http-api), Yamtrack + Redis, Tautulli, NPM, the old Beszel hub + agent, Portainer, and a Discord bot. Exited / stale: watchstate, navet, mealie.

On the **laptop:** qBittorrent, slskd, soularr, lazylibrarian, the hermes stack, streamystats, meilisearch, aura, NPM, the old uptime-kuma, and a beszel-agent.

**Migration order:** Kuma + Beszel (done, at least the hubs) → Actual Budget, PocketBase (easy next) → n8n → Home Assistant → \*arr → media last. Torrents / downloaders stay on the laptop.

**Nginx Proxy Manager:** combine the two NPMs (external `*.danishandarwa.com` + internal `*.home`) into one on the GEEKOM, with a home-only access list (`192.168.1.0/24` + `100.64.0.0/10`). Admin services go behind the tunnel + Access; eventually I can close the router port forwards.

**AdGuard Home / Pi-hole** for local DNS, once the homelab is on Ethernet. Watch for `systemd-resolved` holding port 53, and give the box a DHCP reservation first so the DNS address can't drift.

**Git-backed stacks:** Arcane can pull a compose file from a repo and keep the local `.env` (secrets stay off GitHub). Idea: a private `homelab-stacks` repo, one folder per stack.

**Container manager:** still undecided between Portainer, Arcane and Komodo. Leaning FOSS, but I want to live with them a bit longer before picking a winner.

---

## Gotchas so far

- **Same-day delivery** is amazing and also means you start installing the same evening whether you’re ready or not.
- Etisalat HomeBox DHCP reservations are under **LAN → Static DHCP**, not under the Internet page.
- **RDP loses its password on reboot (TPM issue).** See "How to fix RDP after a reboot" above. Tailscale SSH is the way back in.
- Fresh Ubuntu 26.04 Desktop has no `curl`.
- I briefly considered Windows for “transcoding will be easier.” It wouldn’t — Docker-on-WSL2 is the wrong shape for Quick Sync in containers.
- Arcane's sample `.env` had a leftover `NGINX_HOST_PORT` that had to be cleared before Beszel would start cleanly.
- Some Uptime Kuma LAN monitors showed faded / paused with **Resume** greyed out. A refresh or edit-and-save fixed it — likely a UI glitch.
- Reaching `100.x` Tailscale IPs from a container may need `network_mode: host`.
- The new Cloudflare Tunnel UI calls public hostnames **Routes** → **Add route** → **Published application**.
- A tunnel service URL can't contain a path (`http://localhost:3001/status/...` is rejected). Point the hostname at the app root and set the domain on the Kuma status page instead.
- Kuma's login runs over a websocket, so Cloudflare Access path rules aren't airtight. Keep Kuma's own password.
- When I get to AdGuard: Ubuntu's `systemd-resolved` already holds port 53, and the GEEKOM will need a DHCP reservation so the DNS address can't drift. Deferred until the box is on Ethernet.

---

## Final config (as of 10 Nov 2026, afternoon)

| Item | Value |
|------|--------|
| Hostname | `homelab` |
| Remote access | Tailscale (`--ssh`, key expiry off) + GNOME Remote Login (RDP :3389) |
| LAN IP | DHCP on Wi-Fi for now; planned static `192.168.1.20` once on Ethernet |
| OS | Ubuntu Desktop **26.04.1** (amd64) |
| Cockpit | `http://homelab:9090` |
| Portainer | `https://homelab:9443` |
| Arcane | `http://homelab:3552` |
| Komodo | `http://homelab:9120` |
| Stacks folder | `/opt/docker` |
| HW transcode node | `/dev/dri/renderD128` ✅ |
| NAS mounts | `/mnt/nas/media` (volume1/Media), `/mnt/nas/media-expansion` (volume3/Media - Expansion), NFSv4 |
| Beszel | Hub + agent on homelab, `:8090`, data in `./beszel_data`, agent via `/beszel_socket/beszel.sock`. Public URL `beszel.danishandarwa.com` (Access-protected). NAS/laptop agents still on the old NAS hub (`:8095`) |
| Uptime Kuma | `louislam/uptime-kuma:2` on `:3001`. Status page **Home Media** (slug `family`) at `status.danishandarwa.com` |
| Cloudflare Tunnel | Named `homelab`, `cloudflare/cloudflared:latest`, `network_mode: host`, token in local `.env` |
| Cloudflare Access | Policy **Only Me** (my email, 1-week session, one-time PIN). `status.danishandarwa.com` locks admin paths; `beszel.danishandarwa.com` is locked for the whole domain. Public status page stays open |
| Hypervisor | No Proxmox. Docker `cpus:` / `mem_limit:` instead; virt-manager / Cockpit Machines later if needed |

---

## Next after this log

- Easy container moves first: Actual Budget, then PocketBase
- Combine the two Nginx Proxy Managers onto the GEEKOM (external + `*.home`, with a LAN/Tailscale access list)
- Put more admin UIs behind the tunnel + Access; close router port forwards when that's done
- Point the NAS and laptop Beszel agents at the new hub when those machines migrate
- AdGuard Home / Pi-hole, once the box is on Ethernet and has a DHCP reservation
- Private `homelab-stacks` repo for Git-synced compose files (local `.env` for secrets)
- Pick a container-manager winner (Portainer vs Arcane vs Komodo; leaning FOSS)
- 12TB CMR drive purchase (still needs an explicit yes)
- Media apps (Plex / Jellyfin / \*arr) last; torrents stay on the laptop

---

*Written as a homelab dev log so I can reuse this as a blog/guide later. No secrets, no MACs, no public IPs, no tokens on purpose.*
