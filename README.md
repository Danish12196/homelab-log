# Homelab Log

Dev logs for my Dubai homelab — what I changed, why I chose it, and how I set it up. Written so future-me can rebuild things at 2am, and so the same notes can turn into blog posts or guides later.

**Lab shape (Nov 2026):** Synology DS923+ (`dna-nas`) stays media + backups over NFS. A dedicated mini PC takes Docker / Portainer compute (Plex, Jellyfin, \*arr, n8n, Hermes, etc.).

## Entries

| # | Date | Title |
|---|------|-------|
| 01 | 2026-11-09 | [Choosing a mini PC for the homelab](logs/2026-11-09-01-choosing-a-mini-pc.md) |
| 02 | 2026-11-09–10 | [Setting up the GEEKOM](logs/2026-11-09-02-setting-up-the-geekom.md) |
| 03 | 2026-11-11 | [Always-on desktop and .homelab names](logs/2026-11-11-03-always-on-desktop-and-homelab-names.md) |
| 04 | 2026-11-11–12 | [Ethernet, headless RDP and MausBot](logs/2026-11-12-04-ethernet-headless-rdp-and-mausbot.md) |
| 05 | 2026-11-12 | [n8n and an MCP gateway](logs/2026-11-12-05-n8n-and-an-mcp-gateway.md) |

## Conventions

- First-person notes, casual tone, blog-ready Markdown
- Each entry: what I did, why (including alternatives), gotchas, final config, next steps
- No secrets, no MAC addresses, no public IPs — LAN IPs are fine
- Drafts stay local until I’m ready to publish the next session
