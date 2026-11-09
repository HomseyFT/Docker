# Docker-Stack

Compose files for a self-hosted homelab, fronted by Traefik and published to the
internet through a Cloudflare Tunnel. Public services live under `*.devazul.com`.

## Layout

| Directory       | What it runs |
|-----------------|--------------|
| `reverse-proxy/`| Traefik v3 — routes by `Host()` label, listens on `:80` only (Cloudflare terminates TLS) |
| `cloudflared/`  | Cloudflare Tunnel connector — the only ingress path from the internet |
| `infra/`        | Portainer, Dozzle, Uptime Kuma, Jellyfin, Navidrome, Ollama + Open WebUI, Pi-hole, the portfolio site, and the Homelab-Helper + Marist + Finance app containers |
| `forgejo/`      | Forgejo git hosting (`git.devazul.com`, SSH on `:2222`) |
| `marist/`       | MySQL backing the Marist app |
| `homelab-helper-site/` | MySQL backing the Homelab-Helper-Site app |
| `chess-coach/`  | Chess.com coaching MCP server for Claude (`chess.devazul.com/mcp/<token>`), built from `~/sites/chess-coach-mcp` |
| `torrent/`      | qBittorrent behind Gluetun (ProtonVPN WireGuard, kill switch, port forwarding). Web UI on the tailnet only |
| `socks/`        | SockOffice, a static sock-themed office app (`socks.devazul.com`), built from `Socks-Knocked-Off` |

All stacks attach to a shared external Docker network named `proxy`.

## Setup

The `proxy` network is external and must exist before any stack starts:

```sh
docker network create proxy
```

Every stack reads its config from a `.env` next to its compose file. Compose does
**not** walk up directories for `.env`, so each stack needs its own copy:

```sh
cp .env.example infra/.env
cp .env.example cloudflared/.env
cp .env.example marist/.env
cp .env.example homelab-helper-site/.env
cp .env.example torrent/.env
cp .env.example forgejo/.env
cp chess-coach/.env.example chess-coach/.env   # has its own template
cp .env.example socks/.env
```

Fill in the blanks in each, then bring the stacks up — reverse proxy first, then
the services, then the tunnel:

```sh
docker compose -f reverse-proxy/docker-compose.yml up -d
docker compose -f infra/docker-compose.yml up -d
docker compose -f marist/docker-compose.yml up -d
docker compose -f homelab-helper-site/docker-compose.yml up -d
docker compose -f torrent/docker-compose.yml up -d
docker compose -f forgejo/docker-compose.yml up -d
docker compose -f chess-coach/docker-compose.yml up -d
docker compose -f socks/docker-compose.yml up -d --build
docker compose -f cloudflared/docker-compose.yml up -d
```

Compose fails fast with a named error if a required secret is missing, rather
than silently starting a service with a default password.

## Torrenting

Never run a torrent client directly on the host. This machine has a public IPv6
address and no VPN on it, so a bare client announces the real address to every
peer and to the ISP. The `torrent/` stack exists so that can't happen: qBittorrent
lives inside Gluetun's network namespace and Gluetun's firewall only permits the
WireGuard tunnel, so a VPN drop stops traffic instead of leaking it.

The web UI is published on `127.0.0.1:8081` only. It is exposed to the tailnet
(and nothing else) with:

```sh
sudo tailscale serve --bg --http 8081 8081
```

That is persistent across reboots. Open `http://nathanwebserver:8081` from any
device on the tailnet. First start: `docker logs qbittorrent` prints a temporary
admin password; set a real one under Settings -> Web UI. Downloads land in
`/mnt/media/downloads`.

To confirm the kill switch and the exit IP:

```sh
docker exec qbittorrent wget -qO- https://ipinfo.io/ip   # must NOT be the home IP
docker stop gluetun && docker exec qbittorrent wget -qO- https://ipinfo.io/ip   # must fail
```
