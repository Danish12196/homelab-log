---
title: "n8n and an MCP gateway"
date: 2026-11-12
slug: n8n-and-an-mcp-gateway
tags: [homelab, n8n, docker, mcp, metamcp, mcpjungle, openmausbot, nginx-proxy-manager, postgres, tailscale, adguard-home, arcane]
status: draft
---

# n8n and an MCP gateway

**Log #5 · 12 November 2026 (Asia/Dubai)**

Six threads today:

1. Stand up a fresh n8n on the GEEKOM, with its built-in MCP server, so OpenMausBot has tools.
2. Decide a gateway is better than using n8n itself as the universal MCP bridge.
3. Try MCPJungle first, after upstream MetaMCP was archived the same day.
4. Switch to the maintained MetaMCP fork, and work through its setup gotchas.
5. Point MausBot at a MetaMCP endpoint, so later services show up automatically.
6. Add Beszel, Uptime Kuma, Portainer, Arcane, NPM and AdGuard as their own MCP servers.

> Throughout this log, `<tailscale-ip>` is the homelab's Tailscale address (a `100.x.y.z` IP).

---

## Part 1: A fresh n8n for MausBot

Log #4 left OpenMausBot running as a systemd service on the GEEKOM (`homelab`, Ubuntu 26.04). The next obvious step is giving it tools. n8n is the first one: workflows I can build myself, plus a built-in MCP server the bot can call.

I already had an n8n on the Synology NAS. I decided not to migrate it. A clean start on the mini PC is simpler, this instance is just for the lab's bots and automations, and I didn't want to drag NAS-era config into a Docker stack I'm managing in Arcane under `/opt/docker`.

### The stack

Postgres 16 and the official n8n image, published on port `5678`. The important environment:

```yaml
# n8n service, trimmed to the bits that matter
image: docker.n8n.io/n8nio/n8n:latest
ports:
  - "5678:5678"
environment:
  DB_TYPE: postgresdb
  N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
  N8N_HOST: n8n.homelab
  N8N_PROTOCOL: http
  WEBHOOK_URL: http://n8n.homelab/
  N8N_SECURE_COOKIE: "false"
  TZ: Asia/Dubai
volumes:
  - ./data:/home/node/.n8n
```

`N8N_ENCRYPTION_KEY` is generated once and kept in `.env`:

```bash
openssl rand -hex 32
```

`N8N_SECURE_COOKIE=false` is required because this is plain HTTP on `n8n.homelab`, not HTTPS. The data volume is owned by n8n's user; if the container complains about permissions, `chown 1000:1000` on `./data` fixes it.

NPM gets a proxy host `n8n.homelab` → `host.docker.internal:5678`, with Websockets Support on. AdGuard already rewrites `*.homelab` to the Tailscale IP, so the name just works on the tailnet.

### Built-in MCP

n8n has instance-level MCP under **Settings → MCP Access**. That gives you a server URL at `/mcp-server/http` and an access token. Two catches:

- Each workflow has to be individually enabled for MCP, and it has to be active.
- The workflow has to start with a Webhook, Schedule, Chat, or Form trigger. A workflow that starts with something else won't show up as a tool.

I pointed MausBot at that URL directly first, with the token, and it connected. That was enough to prove the path. It is not how I want to run things long-term, which is the rest of this log.

### Why I skipped Composio

I looked at Composio as a hosted MCP layer. I skipped it. It's a cloud service, so it can't reach tailnet-only things (n8n, Arcane, the *arr stack, and so on) unless I expose them to the internet. Tailscale already gives me remote access. I don't want a public n8n just so a SaaS broker can see it.

---

## Part 2: Why a gateway, not n8n as middleware

n8n is great at multi-step automations. It is clunky as a universal bridge. Every action the bot should have — "restart this container", "what's down in Kuma", "add this to Sonarr" — would be a hand-built workflow, with its own trigger and MCP toggle. That does not scale, and it makes n8n a middleman for tools that already speak MCP themselves.

What I actually want is one MCP endpoint that aggregates per-service MCP servers: n8n, Arcane, the *arrs, Uptime Kuma, and whatever comes next. Any bot connects once, and new tools appear when I add the server behind the gateway, not when I rebuild a workflow.

---

## Part 3: MCPJungle first

My first choice was [MetaMCP](https://github.com/metatool-ai/metamcp). The same day I went to use it, the upstream `metatool-ai/metamcp` repo was archived (read-only). I didn't want to build a gateway on an unmaintained project, so I went with [MCPJungle](https://github.com/mcpjungle/MCPJungle) instead: a Go gateway, one endpoint at `/mcp`, with tool groups.

I stood it up in Arcane, registered n8n, and it worked. Getting there was not free of gotchas, and the feature split is why I didn't stay.

### Enterprise mode first, then a wipe

I started it in enterprise mode. Later I wanted the dashboard, which meant development mode, and it refused:

```
server is already initialized in enterprise mode, cannot start in development mode
```

The fix was wiping the `./postgres` folder and starting clean in the mode I actually wanted. Not a big deal on a brand-new stack, but easy to forget if you flip the mode after the database has been initialized.

### Bind the ports tightly

The other must-do: bind the published ports to `127.0.0.1` and `<tailscale-ip>` only. A bare `8085:8080` would have exposed an unauthenticated gateway to the whole LAN and Wi-Fi. Same lesson as AdGuard and NPM in the earlier logs — if a service shouldn't be on the house network, don't publish it there.

### The reason I left

Then I sat with the product split. The dashboard (still beta) only works in development mode, and development mode has no auth. Access control — per-client tokens with `--allow` lists — only exists in enterprise mode, and enterprise mode has no dashboard. I wanted both: a UI I can look at, and real access control so each bot gets its own credentials. MCPJungle doesn't offer that yet. So I started looking for a maintained MetaMCP again.

---

## Part 4: The maintained MetaMCP fork

That landed on [Umbrella-IT-Group/metamcp](https://github.com/Umbrella-IT-Group/metamcp), image `ghcr.io/umbrella-it-group/metamcp:latest`, port `12008`. It's a maintained fork of the archived upstream, with a full web UI: logins, namespaces that feed endpoints that issue API keys, per-namespace tool allowlists, RBAC, live logs, and a tool-call audit. That's the combination I was missing.

I stopped and removed the MCPJungle project in Arcane, then created a new project called `metamcp`.

### MCPJungle vs the MetaMCP fork

| | MCPJungle | MetaMCP (Umbrella fork) |
|---|---|---|
| Dashboard | Only in development mode, and that mode has no auth | Full web UI with logins |
| Per-bot access | Enterprise mode only (tokens with `--allow` lists), and that mode has no dashboard | Namespaces → endpoints → API keys |
| Per-tool control | Tool groups | Per-namespace allowlists, plus RBAC |
| Logs | No comparable audit UI once you want auth as well | Live logs and tool-call audit |
| Maintenance | Actively developed, but the dashboard / auth split is still a gap | Maintained fork; upstream `metatool-ai/metamcp` is archived |
| Weight | Light (Go, one endpoint at `/mcp`) | Heavier (Next.js app + Postgres, port `12008`) |

MCPJungle is the leaner gateway, and it did work. I switched because I wanted a dashboard *and* access control, not because it failed.

### Compose and env

App container plus `postgres:16-alpine`. Ports are bound the same way as MCPJungle, so the UI isn't on the whole LAN:

```yaml
services:
  app:
    image: ghcr.io/umbrella-it-group/metamcp:latest
    pull_policy: always
    restart: unless-stopped
    env_file: .env
    ports:
      - "127.0.0.1:12008:12008"
      - "<tailscale-ip>:12008:12008"
    environment:
      DATABASE_URL: postgresql://metamcp_user:${POSTGRES_PASSWORD}@postgres:5432/metamcp_db
      NEXT_PUBLIC_APP_URL: ${APP_URL}
    extra_hosts:
      - "host.docker.internal:host-gateway"
    dns:
      - "<tailscale-ip>"
    volumes:
      - ./uv_cache:/home/nextjs/.cache/uv
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: metamcp_db
      POSTGRES_USER: metamcp_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U metamcp_user -d metamcp_db"]
      interval: 10s
      retries: 5
      start_period: 30s
```

The `dns` line came later (gotcha (c) below). It's AdGuard on the Tailscale IP, so the container can resolve `*.homelab` names.

`.env`, with secrets as placeholders:

```
NODE_ENV=production
POSTGRES_PASSWORD=<generated>
BETTER_AUTH_SECRET=<generated>
APP_URL=http://mcp.homelab
TRANSFORM_LOCALHOST_TO_DOCKER_INTERNAL=true

BOOTSTRAP_USER_EMAIL=<email>
BOOTSTRAP_USER_NAME=Danish
BOOTSTRAP_USER_PASSWORD=<generated>
BOOTSTRAP_ONLY_FIRST_RUN=true
BOOTSTRAP_NAMESPACES=[{"name":"Default","description":"Default namespace","is_public":true,"update":true}]
BOOTSTRAP_ENDPOINTS=[{"name":"Default","description":"Default endpoint","enable_auth":true,"is_public":true,"update":true}]
METAMCP_ADMIN_EMAIL=<email>

M365_TOKEN_KEK=<generated>
MCP_PROXY_URL_ALLOWED_HOSTS=host.docker.internal,n8n.homelab,arcane.homelab,beszel.homelab,kuma.homelab
```

`POSTGRES_PASSWORD` and `BETTER_AUTH_SECRET` can be `openssl rand -hex` values; those two have no special length rule. The bootstrap password does: at least 14 characters. `M365_TOKEN_KEK` is pickier, and it's gotcha (b).

### What "bootstrap" means here

Bootstrap is just the first-run setup MetaMCP does from `.env`. In this fork, sign-up in the UI is turned off, so `BOOTSTRAP_USER_*` is how the admin account gets created. It also creates a starting namespace and endpoint; I set `enable_auth` on that endpoint so it isn't left open. `BOOTSTRAP_ONLY_FIRST_RUN=true` means this only happens on the first start — later redeploys won't overwrite what I've changed in the UI.

After the first login, those lines can be ignored.

### (a) Domain mismatch

I put it behind NPM as `mcp.homelab` (proxy host → `host.docker.internal:12008`, websockets on), then opened that URL while `APP_URL` was still `http://<tailscale-ip>:12008`. The UI warned:

```
Domain Mismatch Warning
You're accessing this app from http://mcp.homelab but it's configured for http://<tailscale-ip>:12008.
```

The login only works from the exact URL in `APP_URL`. The fix is to set `APP_URL=http://mcp.homelab` and **recreate** the container — Arcane's Redeploy, or `docker compose up -d --force-recreate`. A plain restart is not enough; the value is read at start and a restart won't pick up a changed `.env`. After that, always use `http://mcp.homelab`. The IP will show the same warning.

### (b) `M365_TOKEN_KEK`

Creating the n8n MCP server failed until this was set. The first error was blunt:

```
Cannot store an MCP server bearer token: M365_TOKEN_KEK is not configured, and a gateway credential must not be written in plaintext.
```

The name says M365, but the fork uses this key to encrypt *every* stored server bearer token, not just Microsoft ones. It has to be base64 for exactly 32 bytes:

```bash
openssl rand -base64 32
```

That value is 44 characters and ends in `=`. The `=` is padding, and it is part of the key — don't trim it. The line looks like `M365_TOKEN_KEK=abc...xyz=`: the first `=` is the assignment, the last one belongs to the key.

My first paste was wrong. The next error was:

```
M365_TOKEN_KEK must be base64 for exactly 32 bytes (AES-256); got 21 bytes
```

Usually that means the value was typed by hand, cut off, or wrapped in quotes. I replaced the line and confirmed it reached the container:

```bash
cd /opt/docker/metamcp
sed -i '/^M365_TOKEN_KEK=/d' .env
echo "M365_TOKEN_KEK=$(openssl rand -base64 32)" >> .env
# then recreate, and:
docker exec $(docker ps -qf name=metamcp-app) sh -c 'echo -n "$M365_TOKEN_KEK" | base64 -d | wc -c'
# should print 32
```

Back that key up somewhere safe (Bitwarden, in my case). If it is lost, every stored token becomes unreadable and has to be re-entered.

### (c) "Not a permitted destination"

Once the token could be stored, the gateway still wouldn't connect to n8n:

```
The MCP server URL, or a URL it redirected to, is not a permitted destination:
its host does not resolve, or resolves to an address range this gateway does not connect to.
```

That's an SSRF guard. It blocks private address ranges by default, and `host.docker.internal` counts. Tailscale addresses in `100.64.0.0/10` are always refused, even if you try to allow them by IP, so don't point MCP server URLs at the Tailscale address.

Allowed hosts are trusted by **exact name** — no wildcards, so `*.homelab` does nothing. For a name on the list, the address check is skipped:

```
MCP_PROXY_URL_ALLOWED_HOSTS=host.docker.internal,n8n.homelab,arcane.homelab,beszel.homelab,kuma.homelab
```

The container also can't resolve `.homelab` names by itself. Rather than adding an `extra_hosts` line for every service, I pointed the app's DNS at AdGuard:

```yaml
    dns:
      - "<tailscale-ip>"
```

Adding a service later means adding its exact name to `MCP_PROXY_URL_ALLOWED_HOSTS` and recreating. Having to list them one by one is the point of the check.

### (d) 401s that weren't the gateway

With the allow list in place, n8n started answering, but the logs filled up with:

```
Unauthorized: Authorization header not sent
```

and later, after I regenerated the n8n token to be safe, a plain `401 Unauthorized` until the new token was saved. The status in the UI still flipped to **Connected**, which was confusing.

The "header not sent" lines are from MetaMCP's built-in **Inspector** — the test panel on the server page. The Inspector does not send the saved Bearer Token field at all; it only sends Custom Headers. The real gateway path, which is what a client like MausBot uses, *does* send the token. That's why the server can show **Connected** while the Inspector keeps 401ing.

Two ways to test:

- To make the Inspector work, add a custom header `Authorization` with the value `Bearer <n8n token>`. Don't put `Bearer ` in the Bearer Token field itself; MetaMCP adds that prefix there.
- The real test is through a client, which is Part 5.

If a server was first created while the KEK was broken, it's safer to delete it and add it again than to keep editing the old row.

### Public vs private ownership

When you add an MCP server, "Public" only means every MetaMCP account can see it. It does not mean the internet. I'm the only user anyway, and a public namespace may not accept a private server. Who can actually call n8n is still the endpoint's API key.

---

## Part 5: Wiring MausBot

n8n is connected through MetaMCP. The last step is pointing MausBot at an endpoint for it, instead of at n8n directly.

1. **Namespaces:** create one called `mausbot` and add the n8n server to it.
2. **Endpoints:** create one called `mausbot`, pick that namespace, and turn on API key auth.
3. **API Keys:** create a key, name it `mausbot`, and copy it.
4. **In MausBot:** remove the old direct n8n MCP entry and add a new MCP server:
   - URL: `http://localhost:12008/metamcp/mausbot/mcp`
   - Header: `X-API-Key` with the raw key (`sk_mt_...`) as the value

MausBot runs on the same box, so `localhost` skips NPM entirely.

### Which header?

I first tried a header called `Authorization`, and then one called `Bearer`, and neither worked. MetaMCP's endpoint auth accepts either of these:

- `X-API-Key: sk_mt_...`, with just the key as the value. This is what worked for me.
- `Authorization: Bearer sk_mt_...`, where the value is the word `Bearer`, a space, then the key.

There's also a `?api_key=` query parameter, but it only works if query-param auth is switched on for the endpoint. For the endpoint I left OAuth off and "automatically create an MCP server" off, and turned on API key auth only.

With `X-API-Key` set, MausBot listed the n8n tools and called them through the gateway. ✅

Later, whenever I add Kuma or Arcane (or anything else) to the `mausbot` namespace, MausBot should pick them up automatically, with no changes on its side.

---

## Part 6: Six more MCP servers

With n8n working, I wanted the rest of the homelab in there too: Beszel, Uptime Kuma, Portainer, Arcane, Nginx Proxy Manager and AdGuard Home. None of them has a built-in MCP server, so each one gets a community server running as its own container.

| Service | MCP server | Auth to MetaMCP |
|---|---|---|
| Beszel | [maxbth/beszel-mcp](https://github.com/maxbth/beszel-mcp) (read-only by design) | Bearer gate token |
| Uptime Kuma (v2) | [DavidFuchs/mcp-uptime-kuma](https://github.com/DavidFuchs/mcp-uptime-kuma) | Bearer gate token |
| Portainer CE | [portainer/portainer-mcp](https://github.com/portainer/portainer-mcp) (official, deprecated; tag matches Portainer minor) | Bearer gate token (hex) + `X-Portainer-API-Key: ptr_...` |
| Arcane | [RandomSynergy17/Arcane-MCP-Server](https://github.com/RandomSynergy17/Arcane-MCP-Server) (no image, built locally, patched) | none |
| NPM | [b3nw/nginx-proxy-manager-mcp](https://github.com/b3nw/nginx-proxy-manager-mcp) | Bearer gate token |
| AdGuard Home | [Samik081/mcp-adguard-home](https://github.com/Samik081/mcp-adguard-home) (MCP at `/`, not `/mcp`) | none |

### No published ports, one shared network

All six live in one `/opt/docker/mcp-servers` stack, managed in Arcane, with **no published ports**. They join a Docker network called `mcp`, and MetaMCP's app container joins it too (`networks: [default, mcp]` plus `mcp: external: true`). MetaMCP then reaches them by container name, e.g. `http://beszel-mcp:3000/mcp`, and those names go into `MCP_PROXY_URL_ALLOWED_HOSTS`.

I first planned to publish them on `127.0.0.1` and use `host.docker.internal`, but that can't work: `host.docker.internal` is the Docker bridge gateway, not the host's loopback, so ports bound to `127.0.0.1` aren't reachable from another container.

A shared `x-common` YAML anchor (`<<: *common`) gives every service the same restart policy, network, DNS (AdGuard, for `.homelab` names) and `extra_hosts`.

### Credentials

- **Gate tokens:** for the servers that support MCP-level auth, this is a password you invent (I used Bitwarden's generator, 40+ characters, letters and numbers only so `.env` and headers don't choke). Portainer's must be hex, so that one is `openssl rand -hex 32`.
- **Beszel:** a new user created in the PocketBase admin (`/_/` → `users` collection) with role `readonly`, plus `SHARE_ALL_SYSTEMS=true` on the hub so it sees every system. Beszel's "users" settings just redirect to PocketBase.
- **Uptime Kuma:** admin username and password (2FA off; a JWT from local storage is the alternative if you keep 2FA).
- **Portainer:** an access token from My account → Access tokens, sent as a header from MetaMCP, not in `.env`.
- **Arcane:** an API key with hand-picked permissions: read across the board, plus create/start/stop/restart/redeploy/edit for containers, create/update/deploy/restart for projects, pull/build for images, and environment status. No delete, prune, exec, users, settings or secrets.
- **NPM:** admin email and password (NPM has no API keys). Destructive tools enabled.
- **AdGuard:** admin username and password; `ADGUARD_URL` must include the admin port (`:3000` here), or it fails with "fetch failed".

### Side quest: Uptime Kuma moves from Komodo to Arcane

I'm not loving Komodo, so Kuma moved to Arcane on the way. Its data was a bind mount at `/etc/komodo/stacks/uptime-kuma/data`: stop the stack in Komodo, `cp -a` the folder to `/opt/docker/uptime-kuma/`, create the project in Arcane with the same compose (`louislam/uptime-kuma:2`, pinned to the major so a v3 can't surprise me), deploy.

Arcane failed to create the project with `failed to write project file .env: permission denied` because the folder was made with `sudo`. `chown -R danish:danish` on it fixed that.

### The gotchas

**(e) MetaMCP not on the network.** Beszel failed with the same "not a permitted destination" error as before. `docker network inspect mcp` showed every sidecar but not MetaMCP: the network change hadn't been applied. A full redeploy (not a restart) fixed it.

**(f) AdGuard "fetch failed".** The URL was missing `:3000`.

**(g) Arcane: "Parse error: Invalid JSON".** Every connect attempt returned 400 with a JSON-RPC parse error. The server runs `express.json()`, which consumes the request body, then calls `transport.handleRequest(req, res)` without passing the parsed body, so the MCP SDK tries to read an already-empty stream. The fix is passing `req.body` as the third argument. I patch it at build time:

```dockerfile
FROM node:22-alpine
RUN npm install -g @randomsynergy/arcane-mcp-server@2.1.0 && \
    sed -i 's/await transport.handleRequest(req, res);/await transport.handleRequest(req, res, req.body);/' \
    /usr/local/lib/node_modules/@randomsynergy/arcane-mcp-server/dist/tcp-server.js
ENV ARCANE_HTTP_HOST=0.0.0.0 ARCANE_HTTP_PORT=3000
EXPOSE 3000
CMD ["arcane-mcp-server", "--tcp"]
```

After a rebuild, all six showed **Connected** in MetaMCP. 🎉

### First real run through MausBot

With all seven servers (n8n plus the six above) in the `mausbot` namespace, I asked MausBot for a homelab status report. It pulled Uptime Kuma (15 monitors up, lowest 30-day uptime was Jellyfin), AdGuard (protection on, version, ports), NPM (10 proxy hosts, 0 redirection hosts, which is correct because I've never made one) and Arcane (one environment, 23 running containers).

Two lessons:

- **Too many tools.** Seven servers is a few hundred tools, mostly Arcane's 180 and Portainer's OpenAPI-generated set, and MausBot struggled. Fixes: `ARCANE_TOOL_PRESET=commonly-used`, keep Portainer out of the bot's namespace (Arcane already covers Docker), toggle off unused tools in MetaMCP, or split into a daily namespace and an admin one.
- **Beszel returned 0 systems.** The MCP user could log in but wasn't a member of any system. Either `SHARE_ALL_SYSTEMS=true` on the hub, or add the user to each record in PocketBase's `systems` collection.

Also: Beszel's memory chart looked maxed out, but the big shaded area is Linux page cache. Real usage was about 5.6 GB of 22 GB. The biggest container was an old openbot project, which I stopped in Arcane (not deleted) for now.

Not fixed yet: `portainer.homelab` returns a 401 through NPM while the IP works. Parked for later.

### Write test: "deploy Dozzle"

To see if MausBot could actually *do* things through Arcane, I gave it as little as possible: just "deploy Dozzle", with no link, image, port or compose file. It worked out the rest itself and got Dozzle (a live Docker log viewer) running as a new Arcane project. That's the create, pull and deploy path through the API key confirmed working, with no delete permission needed.

Why Dozzle as the test: it's tiny, stateless (it just reads Docker's own logs through the socket), and something I wanted anyway. If I ever want logs kept long-term, that's a Loki + Grafana or VictoriaLogs job, not Dozzle's.

### Auto-updates in Arcane

Arcane has updates built in, so no Watchtower. Under Environments, pick the environment, then the **Automations** tab:

- **Image Update Watcher** on, with a schedule (needed before Auto Update can be turned on).
- **Auto Update** on.
- **Update Delay** of 48 to 72 hours, so a bad release has time to get pulled or patched before it lands here.
- **Excluded Containers** for anything that would hurt if an update broke it: databases (n8n's and MetaMCP's Postgres, Komodo's Mongo), Arcane itself, NPM, AdGuard, cloudflared, MetaMCP, and the self-built `arcane-mcp` image.

Everything else (Dozzle, Beszel, Uptime Kuma on `:2`, n8n, the other MCP sidecars) can update itself. You can also opt one container out with the label `com.getarcaneapp.arcane.updater=false`, which beats the UI setting.

### Why are my containers called `metamcp-app-1`?

That's Compose's default `<project>-<service>-<number>` name, and the number is there in case you scale a service to several copies. Setting `container_name: metamcp` on a service gives it a clean name. Rename *before* building the auto-update exclude list, because that list matches on container names.

---

## Config summary

| Thing | Where |
|---|---|
| n8n | `http://n8n.homelab` (NPM → `host.docker.internal:5678`, websockets on); container port `:5678`, Postgres 16, data in `./data` |
| n8n MCP | `/mcp-server/http`, with an access token; workflows must be active, MCP-enabled, and started by Webhook / Schedule / Chat / Form |
| MetaMCP | `http://mcp.homelab` (NPM → `host.docker.internal:12008`, websockets on); `:12008` bound to `127.0.0.1` and `<tailscale-ip>` |
| MetaMCP image | `ghcr.io/umbrella-it-group/metamcp:latest` |
| `APP_URL` | `http://mcp.homelab` (must match the URL you log in from; recreate after changing it) |
| Allowed hosts | `MCP_PROXY_URL_ALLOWED_HOSTS` — exact names only, e.g. `host.docker.internal,n8n.homelab,arcane.homelab,...`; app `dns:` is `<tailscale-ip>` (AdGuard) |
| Token encryption | `M365_TOKEN_KEK` — `openssl rand -base64 32` (44 chars, ending `=` is part of the key); backed up in Bitwarden |
| MCP sidecars | `/opt/docker/mcp-servers`, network `mcp`, no published ports; MetaMCP reaches them by container name |
| Auto-updates | Arcane, Environments, then Automations: watcher plus Auto Update, 48 to 72 h delay, DBs and network-critical containers excluded |
| MausBot endpoint | `http://localhost:12008/metamcp/mausbot/mcp`, header `X-API-Key: <key>` (working) |

## Next

- Slim MausBot's tool list and fix Beszel system sharing.
- Give containers clean `container_name`s, then set up the Arcane auto-update exclude list.
- Add the *arr stack.
- Fix the `portainer.homelab` 401.
- Consider `ARCANE_TOOL_PRESET=commonly-used`; 180 tools is a lot for one bot.
- Build real n8n workflows now that the MCP path is in place, not just the empty server.
- Rotate tokens once the setup settles, then keep the new values in Bitwarden.
- Maybe split things into tighter per-bot namespaces later, if one `mausbot` bucket gets too crowded.
- Open PRs #3 and #4, then this entry as PR #5, when I'm ready to publish.
