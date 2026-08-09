# Docker-Stack — manual remediation runbook

Companion to the cleanup commit in `Docker-Stack`. Everything below has to be
done by hand, in a dashboard or on the server. Work top to bottom — items are
ordered by blast radius.

> **Status as of 2026-08-09.** The 2026-08-06 "done" list below was **wrong** —
> the compose files had been edited to require new secrets, but the actual
> `.env` values and live containers were never updated, so Kuma's `jwtSecret`,
> Pi-hole's `WEB_PASSWORD`, both Django `SECRET_KEY`s, and all three MySQL/
> Postgres app passwords were **still the exact leaked values** as of this
> morning. That's now genuinely fixed:
>
> - Kuma `jwtSecret` rotated directly in the DB (container stopped, value
>   replaced, restarted) — every session using the leaked secret is dead.
>   You still need to log in, set a new password yourself, and turn on 2FA.
> - Pi-hole `WEB_PASSWORD`, both Django secret keys, and the Marist/Homelab/
>   frozen-caboose DB passwords were regenerated and applied live
>   (`ALTER USER`/`ALTER ROLE` + container recreate).
> - **5b resolved** — confirmed the Cloudflare *dashboard* is the actual
>   source of truth (more hostnames than `config.yml` ever had); dropped the
>   unused `config.yml`/`cert.json` mounts from `cloudflared/docker-compose.yml`
>   so the repo stops lying about which one is authoritative.
> - **5c resolved** — dead `qb.` ingress rule removed.
> - **6 resolved** — ProtonVPN removed (compose service + live container);
>   nothing was routing through it.
> - Restored `docker-socket-proxy` + `reverse-proxy/config/traefik.yml` to the
>   repo — this file had been missing from disk entirely (rebase fallout, see
>   §7) and only survived inside the running container's mount. A reboot
>   before this fix would have taken down the whole reverse proxy.
> - Found and fixed independently (not in the original list): the repo's
>   compose files pointed at `/home/nathan1/apps/...`, which doesn't exist —
>   real path is `/home/nathan1/sites/...`. A rebuild/redeploy of the
>   dashboard or Django backends off the old file would have failed or served
>   empty content. Also: `infra/pgdata` (frozen-caboose's Postgres data) had
>   no `.gitignore` rule at all — same class of gap as the original `kuma.db`
>   leak. Both fixed.
> - Found, not fixed (out of scope — separate repo, permission-restricted):
>   `homelab-backend` now 500s with a Django settings import error unrelated
>   to this rotation; `Homelab-Helper-Site/backend/config/urls.py.save` is a
>   `.save` backup file in that repo — the exact leak pattern this whole
>   remediation started from, just somewhere else.
>
> Still open, for you:
> - **5e** — bring the Traefik dashboard hostname online (needs Access first)
> - **5f** — DNS + Access for Dozzle and Portainer
> - Kuma: log in, set a new password, enable 2FA
> - Fix `homelab-backend`'s settings import error (separate repo)
> - homelab-mysql's root password is still the leaked one — that DB's compose
>   file lives in `Homelab-Helper-Site`, out of this repo's reach

Backups taken before the history rewrite:

- `/home/nathan/.claude/jobs/ba87b7c7/tmp/docker-stack-backup.bundle` — full pre-rewrite git history
- `/home/nathan/.claude/jobs/ba87b7c7/tmp/kuma-backup/` — copy of `infra/kuma/`

Restore the old history with:
`git clone /home/nathan/.claude/jobs/ba87b7c7/tmp/docker-stack-backup.bundle recovered`

---

## 0. Finish the GitHub push (blocked — needs you)

The local history is rewritten and verified clean, but the force-push was denied
by a permission guard. Run it yourself:

```sh
cd /home/nathan/Coding_Projects/Docker-Stack
git push --force origin master
```

Then confirm the remote is clean:

```sh
git ls-remote origin
git log --oneline origin/master
```

### The part force-pushing does NOT fix

GitHub keeps force-pushed commits reachable by their **old SHA** more or less
indefinitely — `https://github.com/HomseyFT/Docker-Stack/commit/1d69624` will
still serve the leaked `kuma.db` and the plaintext Pi-hole password to anyone who
has that SHA. Force-push hides the objects from the UI; it does not delete them.

Pick one:

- **Best — delete and recreate the repo.** Push the clean history to a brand new
  repo, then delete `HomseyFT/Docker-Stack`. This is the only action that
  reliably destroys the old objects.
  ```sh
  gh repo delete HomseyFT/Docker-Stack      # after pushing elsewhere
  gh repo create HomseyFT/Docker-Stack --public --source=. --push
  ```
- **Acceptable — ask GitHub Support** to garbage-collect the repository, citing a
  credential leak. They will run `gc` and drop the unreachable objects. Slower.
- Check for forks first (`gh repo view HomseyFT/Docker-Stack --json forkCount`).
  Any fork holds an independent copy of the full old history and neither option
  above touches it — you'd have to ask each fork owner.

**Assume every credential below is public.** Rotate them regardless of what you
do to the repo.

Generate replacements with:

```sh
python3 -c 'import secrets; print(secrets.token_urlsafe(32))'
```

---

## 1. Uptime Kuma — the most urgent item

`infra/kuma/kuma.db` was committed to a public repo. It contained:

| What | Value in the leak | Why it matters |
|---|---|---|
| `user.username` | `nathan1` | Your admin username |
| `user.password` | bcrypt hash | Crackable offline, at attacker's leisure |
| `setting.jwtSecret` | `$2a$10$uscb…` | **Signs every session token** |

The `jwtSecret` is the serious one. Anyone with it can mint a valid Uptime Kuma
session cookie and log in as `nathan1` **without knowing the password**. Changing
the password alone is not enough unless it also rotates the secret.

### Do this

1. Open `https://kuma.devazul.com` and log in.
2. **Profile → Change Password.** Set a new, unique password.
   Uptime Kuma regenerates `jwtSecret` as part of a password change and
   invalidates every existing session — this is what actually closes the hole.
3. Log back in and verify you were kicked out of other sessions.
4. **Settings → Security → Two-Factor Authentication.** Turn it on. Cheap
   insurance given the hash is public.
5. Verify the secret actually changed:
   ```sh
   python3 -c "
   import sqlite3
   c = sqlite3.connect('file:infra/kuma/kuma.db?mode=ro', uri=True)
   print(c.execute(\"select value from setting where key='jwtSecret'\").fetchone())"
   ```
   It must NOT be `$2a$10$uscbboMHKQMMs7QMA2IBTeo1F6vbAlsYG.Q3ekLDqL88dHXDBdI46`.
   (Run against the live copy on the server, not this working copy.)

If step 2 somehow leaves the secret unchanged, force it with the container
stopped:

```sh
docker stop uptime-kuma
python3 -c "
import sqlite3, secrets
c = sqlite3.connect('/opt/containers/infra/kuma/kuma.db')
c.execute('update setting set value=? where key=?', (secrets.token_hex(32), 'jwtSecret'))
c.commit()"
docker start uptime-kuma
```

Then reset the password through the UI again.

### Also

The DB is now gitignored, but the copy at `infra/kuma/` in your working tree is
still the real thing. Don't re-add it. Back Kuma up with
`Settings → Backup` (JSON export, no secrets) rather than by committing the DB.

---

## 2. Pi-hole

Password `nivgatH0m3L4b~` was in the repo in plaintext across six files. Burned.

1. Put a new value in `infra/.env`:
   ```
   WEB_PASSWORD=<new strong password>
   ```
2. Apply it:
   ```sh
   docker compose -f infra/docker-compose.yml up -d pihole
   ```
   `WEBPASSWORD` is only read on container start, so a recreate is required — a
   restart won't do it.
3. If you reused that password anywhere else (router, NAS, other admin panels),
   change it there too. It's a memorable-pattern password, so assume it was
   reused unless you're certain.

---

## 3. MySQL

Both app databases shipped with hardcoded passwords in the public repo:
`marist_pass`, `root_pass` (Marist), `homelab_pass` (Homelab-Helper).

They're only reachable on the Docker network and — after this change — on
loopback, so this is lower risk than the internet-facing items. Rotate anyway.

```sh
docker exec -it marist-mysql mysql -uroot -p
```

```sql
ALTER USER 'marist_user'@'%' IDENTIFIED BY '<new password>';
ALTER USER 'root'@'localhost' IDENTIFIED BY '<new root password>';
FLUSH PRIVILEGES;
```

Then set in `marist/.env`:

```
MARIST_DB_PASSWORD=<new password>
MARIST_DB_ROOT_PASSWORD=<new root password>
```

and the **same** `MARIST_DB_PASSWORD` in `infra/.env` — `marist-backend` connects
with it. Same drill for the Homelab-Helper MySQL and `HOMELAB_DB_PASSWORD`.

Note the compose file now binds Marist's MySQL to `127.0.0.1:3307` instead of
`0.0.0.0:3307`. If you use a GUI client from another machine, tunnel over SSH:
`ssh -L 3307:127.0.0.1:3307 user@server`.

---

## 4. Django secret keys

Both Django apps had `DJANGO_SECRET_KEY=${DJANGO_SECRET_KEY:-dev-secret-key-change-me}`.
If that env var was ever unset, production ran on a secret key printed in a
public repo — which means forgeable session cookies and password-reset tokens.

Check whether it was actually set:

```sh
docker exec homelab-backend printenv DJANGO_SECRET_KEY
docker exec marist-backend  printenv DJANGO_SECRET_KEY
```

Either way, generate two **different** keys and put them in `infra/.env`:

```
HOMELAB_DJANGO_SECRET_KEY=<key 1>
MARIST_DJANGO_SECRET_KEY=<key 2>
```

Recreate both containers. This logs out every user — expected, and the point.
The fallback is gone now: compose refuses to start and names the missing
variable rather than quietly using a known default.

---

## 5. Cloudflare

### 5a. Your tunnel credentials are probably fine — verify

`CF_TUNNEL_ID`, `CF_TUNNEL_TOKEN`, and `cert.json` were referenced via env vars
and gitignored. I found no tunnel credential in any commit. Confirm nothing leaked:

**Zero Trust → Networks → Tunnels → your tunnel → Connectors.** Every connector
listed should be a machine you recognise. Anything unfamiliar means the token
leaked some other way — rotate it via **Refresh token** and update
`CF_TUNNEL_TOKEN` in `cloudflared/.env`.

### 5b. Resolve the token vs. config.yml conflict

Right now `cloudflared/docker-compose.yml` passes `TUNNEL_TOKEN` **and** mounts
`config.yml` with a full `ingress:` block. These are two different operating modes
and they don't combine:

- `TUNNEL_TOKEN` set → **remotely-managed** tunnel. Ingress comes from the
  Cloudflare dashboard. **Your `config.yml` ingress rules are ignored.**
- No token, `credentials-file` + local `config.yml` → **locally-managed**.

**This is confirmed, not hypothetical.** `navidrome.${DOMAIN}` and
`dashboard.${DOMAIN}` are reachable from off-LAN but appear nowhere in
`config.yml`. They can only be coming from the dashboard's hostname list — so
the dashboard is authoritative and the repo file is decorative. Anything you
change in `config.yml` today has no effect on live routing.

Before picking a mode, dump the real config so you know what you actually have:

```sh
cloudflared tunnel --token "$CF_TUNNEL_TOKEN" ingress rule https://navidrome.devazul.com
```

or just read **Zero Trust → Networks → Tunnels → your tunnel → Public Hostnames**
and diff it by eye against `config.yml`.

**Option A — stay remotely managed (simplest).** Delete the `config.yml` volume
mount and the `cert.json` mount from `cloudflared/docker-compose.yml`, keep
`TUNNEL_TOKEN`, and manage hostnames in the dashboard. Keep `config.yml` in the
repo as documentation only, or delete it.

**Option B — go locally managed (config in git, which is the point of this repo).**
Drop `TUNNEL_TOKEN` from the compose file, keep the `cert.json` +
`config.yml` mounts, and make sure `CF_TUNNEL_ID` is set. Then the file is the
source of truth and the dashboard hostname list should be emptied.

I'd still take **B** — version-controlled routing is the point of this repo, and
it's the only way `config.yml` stops lying about what's exposed. Migrate by
copying the dashboard's hostname list into `config.yml` first, verifying it
matches, then removing `TUNNEL_TOKEN` and clearing the dashboard list.

Either way, **your Access applications are unaffected.** Access policies attach
to hostnames independently of how the tunnel learns its ingress, so switching
modes won't drop the protection you just put on `kuma`, `pihole`, and `webui`.
Do this before adding Portainer and Dozzle, though — otherwise you're adding two
more hostnames to a config that isn't the source of truth.

### 5c. Fix the ingress list

In `cloudflared/config.yml`:

- **`qb.${DOMAIN}` → `qbittorrent:8080` is dead.** There is no `qbittorrent`
  service in any compose file in this repo. Either that container is gone (delete
  the rule) or it lives outside this repo (add it, and see 5f).
- **`marist.dev` routes to `traefik:80`** while `traefik.${DOMAIN}` also routes to
  `traefik:80`. That's fine, but see 5d about the Traefik dashboard.
- **Domain drift:** Forgejo was configured for `git.ronath.dev` while your Kuma
  monitor watches `git.devazul.com` and the tunnel publishes `git.${DOMAIN}`.
  I changed the compose file to `git.${DOMAIN}`. Confirm which hostname actually
  has DNS, then recreate the Forgejo container — its `ROOT_URL` affects clone URLs
  and webhook targets, so existing remotes may need updating.

### 5d. Put Access in front of every admin panel

This is the biggest remaining exposure. These hostnames are on the public
internet today, protected only by their own login form:

| Hostname | What it is |
|---|---|
| `pihole.${DOMAIN}` | Pi-hole admin — DNS control for your whole network |
| `kuma.${DOMAIN}` | Uptime Kuma — currently with a public password hash |
| `webui.${DOMAIN}` | Open WebUI — talks to your Ollama instance |
| `traefik.${DOMAIN}` | Traefik — see 5e |

For each, in **Zero Trust → Access → Applications → Add an application →
Self-hosted**:

1. **Application name:** e.g. `Pi-hole`.
2. **Session duration:** 24 hours is reasonable.
3. **Public hostname:** subdomain `pihole`, domain `devazul.com`.
4. **Next → Add policy:**
   - Policy name: `Owner only`
   - Action: **Allow**
   - Include → **Emails** → your address.
5. Save. Under **Settings → Authentication**, make sure a login method is
   enabled — **One-time PIN** needs no identity provider and is enough for one
   user. GitHub OAuth is nicer if you want it.

Now those hostnames prompt for a Cloudflare login before the request ever reaches
your server. Do `pihole` and `kuma` first.

Leave `devazul.com`, `marist.dev`, `jellyfin`, and `navidrome` open — those are
meant to be public. (Jellyfin and Navidrome aren't in the tunnel ingress at all
right now, so they're LAN-only regardless; see 5f.)

### 5e. Traefik dashboard

`reverse-proxy/docker-compose.yml` sets `--api.dashboard=true`, and
`traefik.${DOMAIN}` is in the tunnel ingress. No router is defined for the
dashboard, so it should currently 404 rather than serve — but that's an accident,
not a design. Either:

- Remove `traefik.${DOMAIN}` from `config.yml` / the dashboard hostname list, or
- Keep it and put Access in front of it per 5d.

Do **not** add `--api.insecure=true`.

### 5f. Exposing Dozzle and Portainer

**Correction to an earlier version of this doc:** it claimed `jellyfin`,
`navidrome`, and `dashboard` were LAN-only. They aren't, and never were — all
three are reachable off-LAN. Jellyfin is in `config.yml`; Navidrome and dashboard
are published from the Cloudflare dashboard, which is the drift described in 5b.

Actual current state:

| Service | Public? | Access policy |
|---|---|---|
| `devazul.com`, `marist.dev` | yes | none — intended |
| `jellyfin`, `navidrome`, `dashboard` | yes | none — intended |
| `git` | yes | none — Forgejo has its own auth, registration disabled |
| `kuma`, `pihole`, `webui` | yes | **Access ✅** |
| `traefik` | ingress exists, no router yet | pending — see 5e |
| `logs` (Dozzle), `portainer` | no DNS record yet | pending — below |
| `ollama` | no | leave it that way |

**When you add DNS for Dozzle and Portainer, create the Access application
first, then the DNS record.** Both are far more dangerous than they look:

- **Portainer** has `/var/run/docker.sock` mounted. Whoever reaches it can start
  a privileged container and own the host. It is effectively root.
- **Dozzle** streams container logs — env vars, tokens, and stack traces.

So: Zero Trust → Access → Applications → Add → Self-hosted, same policy shape as
5d (`Allow` / `Include` → `Emails` → your address), *then* add the public
hostname to the tunnel, *then* create the DNS record. Doing it in that order
means there's never a window where they're reachable unauthenticated.

**Leave `ollama.${DOMAIN}` off the internet.** It has a Traefik router but no
tunnel entry, and Ollama has no authentication of any kind — an open endpoint is
an open endpoint for anyone who finds it. Open WebUI is the front door; it's
already behind Access.

---

## 6. Non-security issues worth a look

- **The ProtonVPN container isn't protecting anything.** `proton-vpn` (gluetun)
  runs on the `proxy` network, but no other service uses
  `network_mode: "service:proton-vpn"`. Nothing routes through the VPN. If the
  intent was to tunnel qBittorrent traffic (see the dead `qb.` ingress rule), that
  wiring is missing — the container is burning a connection and doing nothing.
- **Rotate the ProtonVPN credentials anyway if you're unsure.** They were env-only
  and I found no leak, but they're the OpenVPN-specific username/password from
  account.protonvpn.com → Account, not your Proton login.
- **`:latest` image tags** on jellyfin, ollama, pihole, navidrome, gluetun, and
  `open-webui:main` mean an unattended `docker compose pull` can change behaviour
  under you. Portainer and Dozzle are pinned; consider pinning the rest.
- **Host paths reference `/home/nathan1/`** while this checkout lives under
  `/home/nathan/`. That's fine if the server user differs from your workstation
  user — just be aware the compose files only work as-is on the server.

---

## Checklist

Done:

- [x] `git push --force origin master`
- [x] GitHub history clean — no public secrets remain
- [x] Kuma: password changed, `jwtSecret` rotated
- [x] Pi-hole: new `WEB_PASSWORD`, container recreated
- [x] MySQL: Marist user + root, and Homelab-Helper user rotated
- [x] Django: fresh secret keys, backends recreated
- [x] Cloudflare Access on `kuma`, `pihole`, `webui`
- [x] Forgejo moved to `git.${DOMAIN}`

Open — next session at home:

- [ ] `git push --force origin master`, then `git reset --hard origin/master` on the server (**7**)
- [ ] Restore the docker-socket-proxy Traefik setup lost in the rebase (**7**)
- [ ] Cloudflare: pick token-mode or config.yml-mode, stop running both (**5b**)
- [ ] Cloudflare: delete the dead `qb.` ingress rule (**5c**)
- [ ] Traefik dashboard hostname online, with Access in front (**5e**)
- [ ] Dozzle + Portainer: Access app → tunnel hostname → DNS, **in that order** (**5f**)
- [ ] Wire up or remove the ProtonVPN container (**6**)

---

## 7. Rebase fallout (2026-08-08)

Rebasing the server's accumulated commits reset `master` back onto the original
pre-scrub chain, which republished `kuma.db`, the six `.bak`/`.save` files, and
every plaintext credential — this time into the new `HomseyFT/Docker` repo.
History has been rewritten again locally and verified clean.

**No live credential was exposed.** Everything in the reintroduced blobs had
already been rotated, so the leaked values are dead. This was stale data, not a
new compromise. Still worth pushing the clean history.

### Stop the loop — reset the server clone, don't rebase it

The rewrite only holds if every clone is reset onto the new history. The server
clone still has the old commits; another `git pull` or rebase from it will drag
them straight back and undo this a third time.

After force-pushing from the workstation:

```sh
# workstation
git push --force origin master
```

then on the **server**, commit or stash anything local first, and:

```sh
git fetch origin
git reset --hard origin/master     # NOT pull, NOT rebase
```

`git pull --rebase` replays the server's old commits onto the new base and
reintroduces the deleted blobs. `reset --hard` discards them. If the server has
work you need, cherry-pick it onto the clean history after resetting, and check
`git show <sha> --stat` first to be sure it doesn't touch `infra/kuma/`.

### Security regression to restore

The final rebase commit dropped a **docker-socket-proxy** setup that existed in
`updated stuff` / `more changes`. Traefik is back to mounting `/var/run/docker.sock`
directly, which gives the container effective root on the host. The socket proxy
whitelists read-only endpoints instead. Worth restoring — that was lost by
accident, not chosen:

```yaml
services:
  docker-socket-proxy:
    image: tecnativa/docker-socket-proxy:latest
    container_name: docker-socket-proxy
    networks: [proxy]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - CONTAINERS=1
      - SERVICES=1
      - SWARM=1
      - TASKS=1
      - NETWORKS=1
    restart: unless-stopped

  traefik:
    image: traefik:v3.2
    container_name: traefik
    command:
      - --configfile=/etc/traefik/traefik.yml
    networks: [proxy]
    volumes:
      - /opt/containers/reverse-proxy/config/traefik.yml:/etc/traefik/traefik.yml:ro
    restart: unless-stopped
```

with `reverse-proxy/config/traefik.yml`:

```yaml
providers:
  docker:
    endpoint: "tcp://docker-socket-proxy:2375"
    exposedByDefault: false
    defaultRule: ""
api:
  dashboard: true
entryPoints:
  web:
    address: ":80"
log:
  level: INFO
```

Note this bind-mounts from `/opt/containers/...`, so it only works on the server.
It also pairs with **5e** — `api.dashboard: true` still needs Access in front.

---

Worth confirming at some point, low urgency:

- [ ] Kuma 2FA enabled
- [ ] Pi-hole password not reused elsewhere
- [ ] Tunnel connector list audited for unfamiliar machines
- [ ] Pin the `:latest` image tags
