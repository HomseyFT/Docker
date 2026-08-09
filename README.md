# Docker-Stack

Compose files for a self-hosted homelab, fronted by Traefik and published to the
internet through a Cloudflare Tunnel. Public services live under `*.devazul.com`.

## Layout

| Directory       | What it runs |
|-----------------|--------------|
| `reverse-proxy/`| Traefik v3 — routes by `Host()` label, listens on `:80` only (Cloudflare terminates TLS) |
| `cloudflared/`  | Cloudflare Tunnel connector — the only ingress path from the internet |
| `infra/`        | Portainer, Dozzle, Uptime Kuma, Jellyfin, Navidrome, Ollama + Open WebUI, Pi-hole, Gluetun/ProtonVPN, the portfolio site, and the Homelab-Helper + Marist app containers |
| `forgejo/`      | Forgejo git hosting (`git.devazul.com`, SSH on `:2222`) |
| `marist/`       | MySQL backing the Marist app |

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
cp .env.example forgejo/.env
```

Fill in the blanks in each, then bring the stacks up — reverse proxy first, then
the services, then the tunnel:

```sh
docker compose -f reverse-proxy/docker-compose.yml up -d
docker compose -f infra/docker-compose.yml up -d
docker compose -f marist/docker-compose.yml up -d
docker compose -f forgejo/docker-compose.yml up -d
docker compose -f cloudflared/docker-compose.yml up -d
```

Compose fails fast with a named error if a required secret is missing, rather
than silently starting a service with a default password.
