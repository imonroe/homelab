# Homepage (`compose.homepage.yml`)

[Homepage](https://gethomepage.dev/) is a fast, static, self-hosted dashboard
for your services. This stack is wired up so it **automatically discovers** the
other containers in your homelab via Docker labels — enable it and your services
show up on the dashboard with no manual list to maintain.

## Enabling

Uncomment the `compose.homepage.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

## Environment variables

Set these in `.env`:

```env
# Hosts allowed to serve the dashboard (comma-separated, no spaces, include the
# port if used). localhost:3000 and 127.0.0.1:3000 are always allowed.
HOMEPAGE_ALLOWED_HOSTS=home.example.com
# Base domain used to build the service links on the dashboard
HOMELAB_DOMAIN=example.com
```

`HOMEPAGE_ALLOWED_HOSTS` is **required** to reach Homepage on any hostname other
than localhost — if you proxy it at `home.example.com`, that value must appear
here or Homepage rejects the request. Setting it to `*` disables the check
(not recommended).

## How automatic service discovery works

Three pieces make discovery work, and they're all set up for you:

1. **The Docker socket** is mounted read-only into the container
   (`/var/run/docker.sock:/var/run/docker.sock:ro`).
2. **`homepage_data/docker.yaml`** points Homepage at that socket:

   ```yaml
   my-docker:
     socket: /var/run/docker.sock
   ```

3. **`homepage.*` labels** on every web-facing service in this repo (in
   `docker-compose.yml` and each `compose.*.yml`) tell Homepage how to display
   them. For example, Plex carries:

   ```yaml
   labels:
     - homepage.group=Media
     - homepage.name=Plex
     - homepage.icon=plex.png
     - homepage.href=https://plex.${HOMELAB_DOMAIN:-localhost}
     - homepage.description=Media server
   ```

When Homepage starts, it reads those labels from the running containers and adds
each service to the dashboard automatically — grouped by `homepage.group`, with
the right icon and a link. Only containers that are **running** are discovered,
so a service appears once you enable its compose file and bring it up.

### Service links (`HOMELAB_DOMAIN`)

Each service's `homepage.href` is built from `HOMELAB_DOMAIN` plus a conventional
subdomain (e.g. `plex.`, `vault.`, `status.`). Set `HOMELAB_DOMAIN` to your real
domain and the links resolve to your proxied hosts. If your NPM subdomains differ
from the defaults, edit the `homepage.href` label on that service.

## Configuration files

The default config lives in `./homepage_data/` and is checked into the repo so
discovery works out of the box:

| File | Purpose |
|---|---|
| `docker.yaml` | Connects Homepage to the Docker socket (discovery) |
| `settings.yaml` | Dashboard title and global options |
| `services.yaml` | Manual services (empty — discovery handles the rest) |
| `widgets.yaml` | Header widgets (resource stats + search) |
| `bookmarks.yaml` | Bookmark links |

Homepage's runtime logs are written to `homepage_data/logs/`, which is
git-ignored. See the [Homepage docs](https://gethomepage.dev/configs/) for the
full configuration reference.

## Running behind Nginx Proxy Manager

Homepage runs on the shared `nginx_proxy_manager_default` network and exposes
port `3000`.

1. Create a proxy host in NPM (e.g. `home.example.com`) forwarding to
   `homepage:3000` — see [Setting Up DNS and SSL](../README.md#setting-up-dns-and-ssl).
2. Add that hostname to `HOMEPAGE_ALLOWED_HOSTS` in `.env`.
3. Because the dashboard links to everything in your lab, protecting it with the
   [Authelia snippets](authelia.md#protecting-an-app-with-authelia) is a good idea.

## Notes

- **Docker socket access** grants Homepage read access to the Docker API. It's
  mounted read-only; for a hardened setup, put a
  [socket-proxy](https://gethomepage.dev/configs/docker/#using-a-docker-socket-proxy)
  in front of it instead.
- To add per-service **widgets** (live stats pulled from a service's API), add
  `homepage.widget.*` labels — most require an API key, so they're left out of
  the defaults. See the [Homepage service docs](https://gethomepage.dev/widgets/).
