# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

"Homelab in a Box" — a single `docker-compose.yml` entry point for a self-hosted home lab. The goal is to go from bare metal to a full home lab quickly.

## Common Commands

```bash
# Create the required external network (one-time setup)
docker network create nginx_proxy_manager_default

# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs for a specific service
docker compose logs -f authelia
docker compose logs -f nginxproxymanager

# Restart a single service
docker compose restart authelia
```

## Architecture

### Services

| Service | Image | Ports | Purpose |
|---|---|---|---|
| `nginxproxymanager` | `jc21/nginx-proxy-manager` | 8100 (HTTP), 81 (admin), 8143 (HTTPS) | Reverse proxy + SSL termination |
| `authelia` | `authelia/authelia` | 9091 | SSO / authentication middleware |
| `redis` | `redis:alpine` | 6379 (internal) | Session/state store for Authelia |
| `portainer` | `portainer/portainer-ce` | 9000 | Docker management UI |

### Network

All services share an **external** Docker network named `nginx_proxy_manager_default`. This network must be created manually before `docker compose up`. It being external allows other compose stacks to join the same network and be proxied by NPM.

### Auth Flow (Authelia + Nginx Proxy Manager)

Protected proxy hosts in NPM use two nginx snippets together:
1. `authelia-authrequest.conf` — adds an `auth_request /authelia` subrequest to every request; Authelia returns 200 (pass) or 401 (redirect to login).
2. `authelia-location.conf` — defines the internal `/authelia` location that forwards to Authelia's `POST /api/authz/auth-request` endpoint.

The Authelia portal itself is served via `auth.conf` at the `auth.*` subdomain through NPM.

`proxy.conf` sets standard forwarding headers and handles real-IP extraction for RFC 1918 ranges (10/8, 172.16/12, 192.168/16).

### Data Persistence

| Path | Contents |
|---|---|
| `./nginx_proxy_manager_data/` | NPM data, Let's Encrypt certs, nginx snippets/site-confs |
| `./nginx_proxy_manager_data/authelia_config/` | Authelia `configuration.yml` and user database |
| `./portainer_data/` | Portainer state |
| `./redis/` | Redis AOF/RDB data |

### Adding a New Protected Service

1. Add the service to `docker-compose.yml` on the `nginx_proxy_manager_default` network.
2. In NPM admin (`:81`), create a proxy host pointing to the container name + port.
3. In the proxy host's "Advanced" nginx config, include the auth snippets:
   ```nginx
   include /config/nginx/snippets/authelia-location.conf;
   include /config/nginx/snippets/authelia-authrequest.conf;
   ```

### Adding a New Public (No-Auth) Service

Same as above, but omit the Authelia snippet includes in the proxy host config.
