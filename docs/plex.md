# Plex (`compose.plex.yml`)

[Plex](https://www.plex.tv/) is a media server that organizes your movies, TV,
music, and photos and streams them to Plex apps on any device.

## Enabling

Uncomment the `compose.plex.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

## Environment variables

Set these in `.env` (all optional, but `PLEX_CLAIM` is strongly recommended for
first-time setup):

```env
# One-time claim token from https://www.plex.tv/claim (expires ~4 minutes after
# you generate it — grab it right before starting the container)
PLEX_CLAIM=claim-xxxxxxxxxxxxxxxxxxxx
# URL clients should use to reach the server (needed for bridge networking)
PLEX_ADVERTISE_IP=https://plex.example.com:443
# Host path to your media library (defaults to ./plex_media)
PLEX_MEDIA_DIR=/srv/media
```

`PLEX_UID`/`PLEX_GID` reuse the shared `PUID`/`PGID` from `.env`; leave those
empty to keep the image defaults.

## Storage

| Path | Contents |
|---|---|
| `./plex_data/config` | Plex database, metadata, settings (can grow large) |
| `./plex_data/transcode` | Temporary transcoding scratch space |
| `${PLEX_MEDIA_DIR}` → `/data` | Your media library, mounted read-only |

## Running behind Nginx Proxy Manager

Plex runs on the shared `nginx_proxy_manager_default` network and exposes port
`32400`.

1. Create a proxy host in NPM (e.g. `plex.example.com`) forwarding to
   `plex:32400` — see [Setting Up DNS and SSL](../README.md#setting-up-dns-and-ssl).
2. Enable **Websockets Support** on the proxy host.
3. Set `PLEX_ADVERTISE_IP` in `.env` to the public URL (e.g.
   `https://plex.example.com:443`) so the server advertises the right address.

Do **not** add the Authelia snippets — Plex has its own authentication and the
Plex client apps need direct access.

## Notes

- **Networking:** Plex officially recommends `network_mode: host` for the fewest
  issues with local discovery and direct streaming. This stack uses the proxied
  bridge setup for consistency; if you need host networking, replace the
  `networks`/`expose` block with `network_mode: host` (Plex then listens
  directly on `32400`, bypassing NPM).
- **Hardware transcoding** requires passing through a GPU device (e.g.
  `/dev/dri`), which is beyond the default config — see the
  [Plex Docker docs](https://github.com/plexinc/pms-docker).
