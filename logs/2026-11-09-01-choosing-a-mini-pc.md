---
title: "Choosing a mini PC for the homelab"
date: 2026-11-09
slug: choosing-a-mini-pc
tags: [homelab, mini-pc, geekom, synology, plex, docker, 11.11]
status: published
---

# Choosing a mini PC for the homelab

**Log #1 · 9 November 2026 (Asia/Dubai)**

My Synology DS923+ (`dna-nas`) has been the heart of the lab for a while — media, backups, and about **23 Docker containers** piled on top of each other. That worked… until it started to feel like I was asking a NAS to be a full always-on compute box. This post is the decision log for why I bought a dedicated mini PC, how I compared the options during the UAE 11.11 sales, and why I landed on the **GEEKOM IT13 Max**.

If you’re in a similar spot (NAS struggling under containers, want proper hardware transcoding, thinking about self-hosting AI agents), this is the “why” behind the buy.

---

## The problem: a NAS doing too many jobs

`dna-nas` is a Synology DS923+ running DSM with Container Manager and Portainer CE. Rough stack on it today:

- **Media:** Plex + Jellyfin
- **\*arr:** Sonarr (+ Sonarr-Anime), Radarr, Lidarr, Seerr, Pulsarr
- **Automation / apps:** n8n (+ Postgres on the NVMe volume), PocketBase, NPM
- **Plus** monitoring and various smaller containers — roughly **23** in total

Storage side is messy too. In the main pool, **sata1** is a failing Seagate ST8000VN004 8TB that needs replacing first (CMR 12TB planned). The NAS still stays as **central media + backups over NFS**. The new box is for **compute**.

What I wanted out of a dedicated always-on mini PC:

1. Take Docker / Portainer load **off** the Synology
2. Better **hardware transcoding** for Plex/Jellyfin (Intel Quick Sync / Arc)
3. Headroom for **20–40 light containers**
4. Future-proof a bit for **self-hosted AI agents** (Hermes, n8n, cloud models; light local AI later — not a priority to run big local LLMs right now)
5. Stay inside roughly **AED 3,000–3,200** during 11.11 on Amazon.ae / Noon (Noon card = 5% cashback)

Rule I kept the whole time: **never buy without an explicit yes from me.**

---

## Timeline (Asia/Dubai)

Dated from Homey / Homey Jr handoff notes, the 11.11 research brief, used-listing search notes, and the shopping chat. Where a clock time isn’t in the record, I mark it approximate.

| When | What happened |
|------|----------------|
| **~5–6 Nov 2026** | Homey and I were already deep in **drive** research (failing 8TB, CMR 12TB lean on Noon Toshiba N300). Mini PC as a *compute replacement* wasn’t Homey’s old plan yet. |
| **6 Nov 2026** | I spun up **Homey Jr** specifically for the home-server upgrade: mini PC + ~12TB drive(s), 11.11 shopping on Amazon.ae/Noon, used market as a side path. Homey handed over the dna-nas context the same day (“as of Nov 6”). |
| **6 Nov 2026 ~02:30** | Homey Jr wrote the full **11.11 buy brief** (`home-server-1111-research.md`): drive first, then mini PC; early lean was **Minisforum NAB9S**. |
| **6 Nov 2026 (day)** | Used mini-PC sweep (Dubizzle / Reddit / etc.). No clean “buy this used NAB9S-class box today” winner. Best *lead* was an old Reddit NUC 13 Pro offer — not verified available. |
| **6 Nov 2026 ~23:40** | Amazon.ae payment for a **Minisforum NAB9S + 8TB WD Red Plus** (~AED 4,548) was **declined** / put on hold (card fraud hold drama — separate story). That attempt did **not** complete. |
| **~7–8 Nov 2026** | Kept comparing configured NAB9S vs DIY barebone+RAM+SSD math, and whether AMD (UM890 Pro) was worth it. |
| **9 Nov 2026** | Found the **GEEKOM IT13 Max** 11.11 listing on Amazon.ae (~AED 2,498.99). Spec / upgradeability / Arc vs Iris Xe comparison vs NAB9S and UM890. |
| **9 Nov 2026** | Rejected a few Reddit distractions (Jetson J10, mini-ITX case build, move-out sale junk) as primary compute. |
| **9 Nov 2026** | Decided **GEEKOM > NAB9S** for this use case. Bought it. **Same-day delivery** in Dubai. |
| **9 Nov 2026 (evening)** | OS choice settled on **Ubuntu Desktop** (not Windows, not bare Debian server, not Mint). Started the install path — see Log #2 (draft). |

---

## Candidates I actually looked at

### Comparison table (prices as seen; AED)

| Machine | CPU / iGPU | RAM | Storage notes | Networking / extras | Price seen | Store / date | Verdict |
|---------|------------|-----|---------------|---------------------|------------|--------------|---------|
| **GEEKOM IT13 Max** (bought) | Core Ultra 9 **185H**, Arc **8 Xe** (AV1 encode) | **24GB LPDDR5 soldered** | Ships **500GB** SSD; **1× M.2 2280 Gen4** + **1× M.2 2242 SATA** | **2× USB4**, dual **2.5GbE**, Wi‑Fi 7, 3‑yr warranty | **AED 2,498.99** (list ~2,999, 11.11) | Amazon.ae · **9 Nov 2026** | **Winner** |
| **Minisforum NAB9S** 32GB/1TB | i9-**13900HK**, Iris Xe 96EU (Quick Sync, **no AV1 encode**) | **32GB DDR4** (board max 32GB) | 1× M.2 Gen4 + 2.5″ SATA bay in lid (heat concern) | Dual Intel 2.5G; USB‑C 3.2 only (**no USB4**) | **AED 3,099** configured; barebone earlier **~1,699–1,799** | Amazon.ae · configured price in shopping chat; barebone live **6 Nov 2026** | Strong, but ~**AED 600** more for less iGPU / older memory |
| **Minisforum UM890 Pro** | Ryzen 9 class / Radeon 780M | DDR5, configs up to **96GB** | Strong upgrade story; OCuLink interesting | USB4 / OCuLink | Live **~AED 4,665–5,239** for 32GB configs | Amazon.ae / Noon · **6 Nov 2026** | **Skip** — over budget; AMD weaker for Plex HW path vs Intel QS/Arc |
| **GMKtec M2 Pro S** | i7-1185G7, Iris Xe | **16GB** (upgradable) | 512GB class configs | Fine ports for the price | **~AED 1,999–2,199** | Amazon.ae / Noon · **6 Nov 2026** | Budget alt only — tight for 20–40 containers + Hermes |
| **Beelink EQ14 / N150 class** | N150 | 16GB | Fine quiet QS box | — | **~AED 1,449** (11.11) | Amazon.ae · **6 Nov 2026** | **Skip as primary** — underpowered for the consolidated stack |
| **Used market** (NUC 13 Pro lead, old OptiPlex/EliteDesk posts, etc.) | Mixed | Mixed | Mixed | Mixed | e.g. NUC lead **AED 2,000** (stale Dec 2025 comment) | Reddit / Dubizzle sweep · **6 Nov 2026** | No verified “buy today” NAB9S-class unit |

### Specs that mattered for *my* stack

| Need | Why it mattered | Who won |
|------|-----------------|--------|
| Intel Quick Sync / modern iGPU | Multi-user Plex/Jellyfin HW transcode | **GEEKOM Arc** (AV1 encode + stronger than Iris Xe on the NAB9S) over NAB9S Iris Xe over AMD |
| Enough RAM included | UAE RAM kits are pricey; I didn’t want a barebone tax | NAB9S **32GB** beats GEEKOM **24GB** on paper — but 24GB is enough for this plan |
| Upgrade path | Future containers / light AI | NAB9S RAM capped at 32GB; GEEKOM **24GB soldered** on this SKU (tradeoff). GEEKOM still has a second M.2. There is also a SO-DIMM IT13 Max SKU (up to 96GB) I didn’t chase on Amazon.ae. |
| USB4 / I/O | External storage / future eGPU-ish options | **GEEKOM** (2× USB4) |
| Dual LAN | NFS to Synology + lab networking | Both NAB9S and GEEKOM (dual 2.5GbE) |
| Price ≤ ~3.2k | 11.11 reality | **GEEKOM ~2.5k** vs NAB9S ~3.1k |

---

## Rabbit holes I rejected (briefly)

These showed up while I was shopping (mostly Reddit / classifieds). Useful to log so I don’t reopen them later:

- **NVIDIA Jetson J10 / similar:** end-of-life platform vibes, tiny RAM, wrong tool for a general Docker + Plex host. Maybe a side AI toy later — **not** the main compute box.
- **DIY mini-ITX “just buy a case” listing:** turns into a full parts hunt (board, CPU, RAM, PSU, cooler). Against the “ship me a working box in budget” goal.
- **Move-out sale rack/PC bundle:** mostly not a clean always-on mini server. The only vaguely useful bit in that thread was cheap old OptiPlex-as-test-box energy — not primary compute.
- **UM890 Pro:** cool AMD box (DDR5 headroom, OCuLink), but live UAE pricing was way over budget and Plex HW encoding is the wrong fight to pick with Radeon vs Intel QS/Arc for my media stack.

---

## Why GEEKOM won

By the evening of **9 Nov**, the comparison had simplified:

> For this use case, NAB9S mainly wins on **32GB vs 24GB**. That wasn’t worth ~**AED 600** when the GEEKOM brought **better graphics (Arc + AV1 encode)**, **faster LPDDR5**, **USB4**, dual 2.5GbE, Wi‑Fi 7, and a 3-year warranty — for **less money**.

Other reasons that sealed it:

- **Brand reputation:** GEEKOM is one of the more established mini-PC brands (not perfect — I saw a UAE review mentioning Wi‑Fi drops — but fine for a known quantity).
- **Same-day delivery** on Amazon.ae in Dubai. I ordered it and it was on the desk the same day. That alone felt unreal.
- **Soldered 24GB tradeoff accepted:** I’d *prefer* upgradeable RAM, but with 24GB already on board and the Synology keeping media/databases on NFS/volume roles, I’m not pretending this is a 64GB LLM workstation.

### What I actually bought

- **GEEKOM IT13 Max** — Amazon.ae ASIN `B0FWHKYC7P` (sold/fulfilled via GEEKOM official store / Amazon)
- Config as listed: **24GB / 500GB**, Core Ultra 9 185H
- Paid **AED 2,498.99** on the 11.11 deal (list was higher)

Still on the todo list after the box is online: the **12TB CMR** drive to replace the failing 8TB in the Synology. Mini PC first for momentum; drive still matters for pool health.

---

## OS choice (same day as unboxing)

I almost went Windows out of comfort. That would have been a mistake for this box.

### Windows — rejected

- Docker Desktop on Windows sits on **WSL2**, and that path is painful for **Intel Quick Sync / `/dev/dri`**-style GPU access in containers
- Forced reboots / update culture on an always-on host
- Extra RAM tax for the desktop OS I wouldn’t really use day to day

### Debian / Ubuntu Server / Mint — considered

- **Debian 13 / Ubuntu Server:** great for headless + Cockpit + Portainer. Lean. I like the idea.
- **Linux Mint:** closest to a Windows-ish desktop if I want a GUI safety blanket

### What I chose: **Ubuntu Desktop**

I’m not super comfortable living in SSH-only land yet. I like UIs. So the plan is:

- **Ubuntu Desktop** (I’m installing **26.04.1**, amd64/Intel ISO) for a familiar GUI when I need it
- **Remote Desktop** so I can unplug the monitor
- **Cockpit** in the browser for updates / disks / logs
- **Docker + Portainer** for containers (clicky deploys, paste a compose file)

Day to day I expect to live in Portainer and the app web UIs. The desktop is a comfort net, not the product.

Full install write-up is **Log #2** (in progress — not published yet).

---

## Lessons learned (so far)

1. **Separate compute from storage early.** A NAS can run containers; that doesn’t mean it should run *all* of them forever.
2. **Price the whole kit.** NAB9S barebone looked cheap until RAM + SSD in the UAE pushed DIY to the same stretch budget as a configured box — or higher.
3. **iGPU matters more than CPU flex for Plex.** Arc with AV1 encode beat “more RAM on Iris Xe” for my media priorities.
4. **Used market needs verification.** Stale Reddit comments aren’t inventory.
5. **Card holds can nuke a 11.11 plan.** The declined NAB9S order on Nov 6 was a reminder to have payment sorted before flash sales.
6. **Write the decision down.** Future me at 2am will not remember why 24GB soldered was acceptable.

---

## Final config (decision state on 9 Nov 2026)

| Item | Choice |
|------|--------|
| Compute | **GEEKOM IT13 Max** (185H / Arc / 24GB / 500GB) |
| Storage host | **Synology DS923+** (`dna-nas`) over **NFS** |
| Planned LAN IPs | NAS `192.168.1.18` · GEEKOM `192.168.1.20` (static DHCP on the Etisalat HomeBox) |
| OS direction | **Ubuntu Desktop 26.04.1** + Cockpit + Docker + Portainer |
| Budget spent (mini PC) | **AED 2,498.99** |
| Still open | 12TB CMR for failing 8TB · migrate containers · NFS mounts · Tailscale on the new box |

---

## Next steps

1. Finish Ubuntu install + first-boot hardening (Log #2)
2. Static DHCP reservation → Cockpit / Portainer
3. Confirm `/dev/dri` (`renderD128`) for HW transcode
4. Mount Synology shares over NFS
5. Move stacks off the NAS one app at a time (Plex/Jellyfin last / carefully)
6. Buy the **12TB CMR** drive when I’m ready (still needs an explicit yes at purchase time)

---

*Written as a homelab dev log so I can reuse this as a blog/guide later. No secrets, no MACs, no public IPs on purpose.*
