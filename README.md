# Gateway

Independent Docker Compose gateway for Caddy HTTPS and domain routing.
This project does not contain app code or a database.

| Domain | Docker network alias | Internal port |
| --- | --- | --- |
| `cm.suzutsuki.com` | `comiket-web` | 80 |
| `timeline.suzutsuki.com` | `timeline-web` | 3000 |

Both apps and Caddy must join the external Docker network named `gateway`.
Only Caddy publishes public ports. Caddy uses the same `caddy:2.9` image
version as the existing deployment.

## Prerequisites

- Docker Engine and Docker Compose plugin on the server.
- Both domains resolve to the server IP (`139.180.203.201`).
- Server and Vultr firewalls allow TCP 80/443; UDP 443 is optional for HTTP/3.
- No other service occupies ports 80/443. The old project's Caddy must stay stopped.
- Both app containers have the network aliases shown above.

Cloudflare DNS-only records work directly with Caddy's publicly trusted
certificates. If proxying is enabled later, use Full (strict), not Flexible.
No Cloudflare token is needed for this configuration.

## Initial server setup

Upload or clone this project to `/opt/gateway`. Commands below run on the
Linux server, not in Windows PowerShell.

Create the shared network once. Inspect first; if it does not exist, create it:

```bash
docker network inspect gateway
docker network create gateway
```

Prepare the local environment file without overwriting an existing one:

```bash
cd /opt/gateway
test -f .env || cp .env.example .env
nano .env
```

Replace the example email with your real certificate contact email.
Do not commit `.env`, private keys, or certificate data.

## Connect the existing Comiket app

In the old project's Compose configuration, keep the web service's existing
settings and add these networks:

```yaml
services:
  web:
    # Keep the existing build and depends_on settings.
    networks:
      default: {}
      gateway:
        aliases:
          - comiket-web

networks:
  default: {}
  gateway:
    external: true
    name: gateway
```

The default network must remain: web uses it to reach api, and api uses it
to reach db. Do not expose db or api on the gateway network.

Remove the old `caddy` service from the old Compose configuration only when
ready to migrate, retaining a backup of that configuration and its certificate
volumes. Otherwise a future full `up` could start the old Caddy again.
Keep the old Compose project name and database volume unchanged.

To resume only the old app services when ready:

```bash
cd /opt/c108/deploy/vultr
docker compose --env-file .env up -d db api web
```

This project does not automatically resume them. The old GitHub deployment
workflow should remain disabled until its deployment configuration has been
updated to avoid restarting the old Caddy.

## Connect the Timeline app

Add the following to Timeline's own Compose configuration:

```yaml
services:
  timeline:
    # Provide the app's own image or Docker build configuration.
    restart: unless-stopped
    networks:
      gateway:
        aliases:
          - timeline-web

networks:
  gateway:
    external: true
    name: gateway
```

This snippet is not a complete app deployment. Timeline still needs its own
Dockerfile/image. Next.js must listen on `0.0.0.0:3000` inside the container;
listening only on `127.0.0.1` prevents Caddy from reaching it.
No public `ports` mapping is necessary for either app.

An app that is absent or stopped returns a gateway error for its domain;
it does not prevent another correctly connected app from working.

## Validate and start

```bash
cd /opt/gateway
docker compose config --quiet
docker compose run --rm --no-deps caddy \
  caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker compose up -d
docker compose ps
docker compose logs --tail=100 caddy
curl -I https://cm.suzutsuki.com
curl -I https://timeline.suzutsuki.com/timeline
```

Only proceed to startup if validation succeeds. Certificate issuance may take
time and requires reachable DNS and public ports. This gateway creates its own
certificate volumes; it does not delete or reuse the old project's volumes.

## Change routing

Edit `Caddyfile`, validate it, and reload only after validation succeeds:

```bash
docker compose exec caddy \
  caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker compose exec caddy \
  caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
```

If an editor replaces the mounted file rather than writing in place and Caddy
still sees the old contents, recreate only the gateway container:

```bash
docker compose up -d --force-recreate caddy
```

## Pause or roll back

Pause the public gateway without deleting containers or certificate volumes:

```bash
cd /opt/gateway
docker compose stop caddy
```

Resume it with `docker compose start caddy`.

To roll back to the old Caddy, stop this gateway first, restore the old Caddy
Compose configuration if removed, and start that service from its original
project directory. Never run both gateways on the same public ports.
The old app services must also be running for the old site to work.

Do not delete certificate volumes or use volume-removal commands.
App/database backups are managed separately by their own projects.

## GitHub automatic deployment

Repository: `https://github.com/Tinellie/site-gateway`.
The workflow in `.github/workflows/deploy.yml` runs on pushes to `main` or
`master`, or manually from Actions. Actions are pinned to verified commit SHAs.
It validates the configuration in CI, connects as `deploy` to
`139.180.203.201:22`, and updates `/opt/site-gateway` to the exact triggering
commit using a fast-forward merge. Local tracked changes or diverged history
stop deployment instead of being overwritten.

Set these repository Actions secrets:

| Secret | Value |
| --- | --- |
| `VULTR_SSH_KEY` | Private key dedicated to Actions, authorized for the server's deploy user |
| `VULTR_SSH_PASSPHRASE` | That key's passphrase, if set |
| `VULTR_HOST_FINGERPRINT` | SHA256 SSH host-key fingerprint verified through a trusted server console |

The server independently needs read access to this GitHub repository, such as
a read-only repository deploy key. Actions' SSH key authenticates to the server;
it does not grant the server access to GitHub.

The server clone must be in `/opt/site-gateway`, use this repository as origin,
and contain an untracked `.env` with a real ACME email. The external `gateway`
network must already exist. Use this directory in place of `/opt/gateway` in
the setup instructions above for the current server.

The workflow recreates only Caddy, preserving certificate volumes and all app
containers. There is a brief public-entry interruption. It checks Caddy's
internal admin endpoint for readiness; successful deployment does not establish
that stopped/missing upstream apps or external DNS/TLS are working.
No GitHub deployment secrets belong in repository files.
