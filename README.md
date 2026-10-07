# Traefik

A hardened, single-host [Traefik v3](https://doc.traefik.io/traefik/) reverse proxy for Docker Compose. It gives you automatic HTTPS for every service on the host using one Let's Encrypt wildcard certificate, issued through the Cloudflare DNS challenge.

## Features

- **Wildcard TLS by default.** One `*.DOMAIN` certificate is the default for every router on `websecure`, so services don't need their own TLS or cert-resolver labels.
- **HTTP → HTTPS redirect** on port 80.
- **No raw Docker socket in Traefik.** Traefik reads the Docker API through [docker-socket-proxy](https://github.com/Tecnativa/docker-socket-proxy), which allows only container listing, inspection and events.
- **Host networking.** Real client IPs (including IPv6) are preserved, and Traefik can reach services bound to `127.0.0.1` on the host, such as Portainer.
- **Hardened container.** All capabilities dropped except `NET_BIND_SERVICE`, `no-new-privileges`, a tmpfs `/tmp`, and CPU, memory and PID limits.
- **Request hardening.** Alias headers (e.g. `X_Auth_User`) are deleted, encoded special characters in paths are rejected, and strict TLS options are enforced.
- **Dashboard** at `traefik.DOMAIN`, protected by basic auth. The API and ping endpoint listen only on `127.0.0.1:1080`.
- **JSON access logs**, with rotated, compressed container logs.

## Requirements

- Linux host with Docker Engine and Docker Compose v2. Host networking needs Linux.
- A domain whose DNS is managed by Cloudflare.
- A Cloudflare API token with **Zone → DNS → Edit** on that zone. Use a scoped token, not the Global API Key.
- Ports `80`, `443` and `50000` free on the host. Port `2375` must be free on `127.0.0.1`.
- `htpasswd` (from `apache2-utils` / `httpd-tools`) to generate the dashboard password.

## Quick start

```sh
cp .env.example .env
chmod 600 .env
```

Edit `.env` and set at least these values:

| Variable                 | Description                                                    |
|--------------------------|----------------------------------------------------------------|
| `DOMAIN`                 | Base domain, e.g. `example.com`. Certificate covers `DOMAIN` and `*.DOMAIN`. |
| `ACME_EMAIL`             | Email address for your Let's Encrypt account.                  |
| `CF_DNS_API_TOKEN`       | Cloudflare API token (passed to Traefik as a Docker secret).   |
| `TRAEFIK_DASHBOARD_AUTH` | Basic-auth `user:hash` for the dashboard. See below.           |

Generate the dashboard credentials:

```sh
htpasswd -nbB admin 'your-password'
```

Paste the result in **single quotes**, so Compose doesn't try to expand the `$` characters in the hash:

```sh
TRAEFIK_DASHBOARD_AUTH='admin:$2y$05$...'
```

Point DNS at the host (an `A`/`AAAA` record for `DOMAIN` and `*.DOMAIN`), then start the stack:

```sh
docker compose up -d
docker compose logs -f traefik
```

The first certificate can take a minute or two while the DNS challenge propagates. Then open `https://traefik.<DOMAIN>`.

## Configuration

Optional overrides in `.env`:

| Variable                  | Default                          | Description                                    |
|---------------------------|----------------------------------|------------------------------------------------|
| `TRAEFIK_TAG`             | `v3.7.13`                        | Traefik image tag.                             |
| `SOCKET_PROXY_REPOSITORY` | `tecnativa/docker-socket-proxy`  | Socket-proxy image repository.                 |
| `SOCKET_PROXY_TAG`        | `v0.5.0`                         | Socket-proxy image tag.                        |
| `TZ`                      | `Europe/Athens`                  | Container time zone.                           |
| `TRAEFIK_LOG_LEVEL`       | `INFO`                           | `DEBUG`, `INFO`, `WARN`, `ERROR`.              |
| `TRAEFIK_READ_TIMEOUT`    | `600` (seconds)                  | `websecure` read timeout. `0` = unlimited.     |
| `TRAEFIK_WRITE_TIMEOUT`   | `600` (seconds)                  | `websecure` write timeout. `0` = unlimited.    |
| `TRAEFIK_CPU_LIMIT`       | `1`                              | CPU limit.                                     |
| `TRAEFIK_MEM_LIMIT`       | `128M`                           | Memory limit.                                  |
| `TRAEFIK_MEM_RESERVATION` | `64M`                            | Memory reservation.                            |
| `TRAEFIK_PIDS_LIMIT`      | `200`                            | PID limit.                                     |
| `TRAEFIK_NETWORK_SUBNET_V4` | `10.100.0.0/24`                  | IPv4 subnet of the `traefik` network.          |
| `TRAEFIK_NETWORK_SUBNET_V6` | `fd00:100::/64`                  | IPv6 subnet of the `traefik` network.          |

> **Network:** The `traefik` Docker network is dual-stack (IPv4 + IPv6). Pick subnets that don't overlap your LAN or other Docker networks. Docker can't change an existing network's subnets, so after changing them (or when upgrading from an IPv4-only network), run `docker compose down` and then `docker compose up -d` to recreate it. Disconnect containers from other stacks first.

> **Timeouts:** A request (read) or response (write) taking longer than 10 minutes is cut off. Raise the value if you serve very large uploads or long-lived streams. Avoid `0` (unlimited): it leaves the proxy open to slow-client (Slowloris) attacks.

### Alternative socket-proxy image

`.env.fork.example` is the same as `.env.example`, except it uses the `ghcr.io/kostelidisdev/docker-socket-proxy` image instead of the upstream Tecnativa one. To use it, copy that file to `.env` instead.

## Entrypoints

| Name        | Address          | Purpose                                         |
|-------------|------------------|-------------------------------------------------|
| `web`       | `:80`            | Redirects everything to `websecure`.            |
| `websecure` | `:443`           | HTTPS, with the wildcard certificate as default. |
| `tcp50000`  | `:50000`         | Extra entrypoint for services that need port 50000. |
| `traefik`   | `127.0.0.1:1080` | Internal API and health check (`/ping`).        |

## Exposing a service

Containers are **not** exposed by default. Opt in with labels. You don't need TLS labels, because `websecure` already applies the wildcard certificate:

```yaml
services:
  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.services.whoami.loadbalancer.server.port=80"
```

## Security notes

- **Encoded path characters are rejected** with `400 Bad Request` (`%2F`, `%5C`, `%3B`, `%25`, `%3F`, `%23`, `%00`). Some backends need them, such as container registries, WebDAV, Git APIs and file managers. If one breaks, re-allow only the character it needs, preferably per router with the [`EncodedCharacters`](https://doc.traefik.io/traefik/middlewares/http/encodedcharacters/) middleware rather than on the whole entrypoint.
- **The socket proxy listens on `127.0.0.1:2375`** of the host. It is read-only (containers only), but any local process can query it. Firewall it accordingly on shared hosts.
- **Never commit `.env`.** It holds your Cloudflare token and dashboard hash.

## Operations

```sh
docker compose ps                     # health status
docker compose logs -f traefik        # follow logs (access log is JSON)
docker compose pull && docker compose up -d   # upgrade after bumping tags in .env
```

Certificates are stored in the `traefik_letsencrypt` volume (`/letsencrypt/acme.json`). Keep it across upgrades to avoid hitting Let's Encrypt rate limits.
