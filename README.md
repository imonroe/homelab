# Homelab in a Box

Go from bare metal to a fully operational self-hosted home lab in minutes. This project provides a single `docker-compose.yml` entry point that wires together a reverse proxy, authentication middleware, and a growing library of optional self-hosted applications — all pre-configured to work together.

## What's Included

### Core Stack (always on)

| Service | Purpose | Port |
|---|---|---|
| [Nginx Proxy Manager](https://nginxproxymanager.com/) | Reverse proxy + automatic SSL via Let's Encrypt | 81 (admin UI) |
| [Authelia](https://www.authelia.com/) | Single sign-on and two-factor authentication | — |
| Redis | Session store for Authelia | — |
| [Portainer](https://www.portainer.io/) | Docker management web UI | 9000 |

### Optional Apps

Enable any of these by uncommenting the relevant line in `docker-compose.yml` (see [Adding or Removing Apps](#adding-or-removing-apps)).

| Compose file | Apps | Notes |
|---|---|---|
| `compose.mysql.yml` | MySQL 8 + Adminer | General-purpose database server with a web-based admin UI |
| `compose.vaultwarden.yml` | [Vaultwarden](https://github.com/dani-garcia/vaultwarden) | Bitwarden-compatible password manager |
| `compose.homeassistant.yml` | [Home Assistant](https://www.home-assistant.io/) | Home automation platform for smart-home devices |
| `compose.pihole.yml` | [Pi-hole](https://pi-hole.net/) | Network-wide DNS ad blocker |

---

## Prerequisites

### 1. Install Docker

Docker is the only dependency. Install it for your platform:

- **Linux (Ubuntu/Debian):** [docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- **Linux (Fedora/RHEL):** [docs.docker.com/engine/install/fedora](https://docs.docker.com/engine/install/fedora/)
- **macOS:** [docs.docker.com/desktop/install/mac-install](https://docs.docker.com/desktop/install/mac-install/) (Docker Desktop)
- **Windows:** [docs.docker.com/desktop/install/windows-install](https://docs.docker.com/desktop/install/windows-install/) (Docker Desktop)

After installing, verify Docker is running:

```bash
docker --version
docker compose version
```

You need Docker Compose v2.20 or later (check with `docker compose version`). It ships with Docker Desktop and recent Docker Engine installs.

### 2. Clone this repository

```bash
git clone https://github.com/your-username/homelab.git
cd homelab
```

### 3. Create the shared network

All services communicate over a shared Docker network. Create it once:

```bash
docker network create nginx_proxy_manager_default
```

### 4. Set up your environment file

Copy the example env file and fill in your values:

```bash
cp .env.example .env
```

Open `.env` in a text editor and update any values marked `changeme`.

---

## Getting Started

### Start the core stack

```bash
docker compose up -d
```

This starts Nginx Proxy Manager, Authelia, Redis, and Portainer in the background.

### Access the admin interfaces

| App | URL | Default credentials |
|---|---|---|
| Nginx Proxy Manager | http://your-server-ip:81 | admin@example.com / changeme |
| Portainer | http://your-server-ip:9000 | Set on first visit |

> **Important:** Change the NPM default password immediately after your first login.

---

## Setting Up DNS and SSL

Nginx Proxy Manager acts as the front door for all your apps. Instead of accessing services by IP and port number, you point domain names at your server and NPM routes traffic to the right container.

### How it works

```
Internet → your-domain.com → your server (ports 80/443) → NPM → internal service
```

### Step 1 — Point a domain at your server

For each app you want to expose, create a DNS `A` record pointing to your server's IP address. You do this in your domain registrar or DNS provider (e.g. Cloudflare, Namecheap, Route 53).

```
vault.example.com   →   A   →   203.0.113.10   (your server's public IP)
adminer.example.com →   A   →   203.0.113.10
portainer.example.com → A  →   203.0.113.10
```

If all your subdomains point to the same server, a wildcard record is convenient:

```
*.example.com  →  A  →  203.0.113.10
```

DNS changes can take a few minutes to propagate. You can check with:

```bash
nslookup vault.example.com
```

### Step 2 — Create a proxy host in NPM

1. Open NPM at `http://your-server-ip:81`
2. Go to **Hosts → Proxy Hosts → Add Proxy Host**
3. Fill in:
   - **Domain Names:** your subdomain (e.g. `vault.example.com`)
   - **Scheme:** `http`
   - **Forward Hostname / IP:** the container name (e.g. `vaultwarden`)
   - **Forward Port:** the app's internal port (see [App-Specific Configuration](#app-specific-configuration))
4. On the **SSL** tab, request a **Let's Encrypt** certificate and enable **Force SSL**
5. Save

NPM will automatically obtain and renew the SSL certificate.

### Step 3 — (Optional) Protect the app with Authelia

For apps that don't have their own login (like Adminer), add these lines under the **Advanced** tab in the NPM proxy host:

```nginx
include /config/nginx/snippets/authelia-location.conf;
include /config/nginx/snippets/authelia-authrequest.conf;
```

> **Do not** add the Authelia snippets to Vaultwarden — it manages its own authentication and the native Bitwarden client apps need direct API access.

---

## Adding or Removing Apps

Optional apps are listed in the `include:` block at the bottom of `docker-compose.yml`:

```yaml
include:
  # - compose.mysql.yml        # MySQL 8 + Adminer
  # - compose.vaultwarden.yml  # Vaultwarden (Bitwarden-compatible password manager)
```

**To enable an app:** remove the `#` at the start of the line, then run:

```bash
docker compose up -d
```

**To disable an app:** add `#` back to the start of the line, then run:

```bash
docker compose down
docker compose up -d
```

> Disabling an app stops its containers but does **not** delete its data. Data is stored in local directories (e.g. `./vaultwarden_data/`) and will be there when you re-enable it.

---

## Common Operations

### View running containers

```bash
docker compose ps
```

### Restart a single service

```bash
docker compose restart authelia
docker compose restart nginxproxymanager
docker compose restart portainer
```

For optional services:

```bash
docker compose restart vaultwarden
docker compose restart mysql
docker compose restart adminer
```

### View logs for a service

```bash
docker compose logs -f authelia
docker compose logs -f nginxproxymanager
```

### Stop everything

```bash
docker compose down
```

### Pull the latest images and restart

```bash
docker compose pull
docker compose up -d
```

---

## App-Specific Configuration

Per-app setup, environment variables, and configuration details live in the
[`docs/`](docs/) directory:

| App | Documentation |
|---|---|
| Nginx Proxy Manager | [docs/nginx-proxy-manager.md](docs/nginx-proxy-manager.md) |
| Authelia | [docs/authelia.md](docs/authelia.md) |
| Portainer | [docs/portainer.md](docs/portainer.md) |
| MySQL + Adminer | [docs/mysql.md](docs/mysql.md) |
| Vaultwarden | [docs/vaultwarden.md](docs/vaultwarden.md) |
| Home Assistant | [docs/home-assistant.md](docs/home-assistant.md) |
| Pi-hole | [docs/pi-hole.md](docs/pi-hole.md) |

---

## Troubleshooting

### "network nginx_proxy_manager_default not found"

You need to create the network first:

```bash
docker network create nginx_proxy_manager_default
```

### A service won't start

Check its logs:

```bash
docker compose logs -f <service-name>
```

### SSL certificate fails to issue

- Make sure your DNS `A` record is pointing to the correct server IP and has propagated
- Make sure ports 80 and 443 on your router/firewall are forwarding to your server
- NPM needs to be reachable from the public internet on port 80 for the Let's Encrypt HTTP challenge
