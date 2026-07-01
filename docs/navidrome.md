# Navidrome (`compose.navidrome.yml`)

[Navidrome](https://www.navidrome.org/) is a self-hosted music streaming server
— a personal Spotify for your own collection, compatible with the Subsonic API
so it works with a wide range of mobile and desktop clients.

## Enabling

Uncomment the `compose.navidrome.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

## Environment variables

Set these in `.env`:

```env
# Host path to your music library (defaults to ./navidrome_music)
NAVIDROME_MUSIC_DIR=/srv/music
```

Navidrome runs as the shared `PUID`/`PGID` from `.env` (defaulting to
`1000:1000`). That user must own `./navidrome_data`:

```bash
mkdir -p navidrome_data
sudo chown -R 1000:1000 navidrome_data
```

Your music folder is mounted read-only, so Navidrome only needs read access to
it. Additional tuning is available through `ND_*` variables — see the
[Navidrome configuration reference](https://www.navidrome.org/docs/usage/configuration-options/).

## Storage

| Path | Contents |
|---|---|
| `./navidrome_data` → `/data` | Database, cache, and settings |
| `${NAVIDROME_MUSIC_DIR}` → `/music` | Your music library, mounted read-only |

## Running behind Nginx Proxy Manager

Navidrome runs on the shared `nginx_proxy_manager_default` network and exposes
port `4533`.

1. Create a proxy host in NPM (e.g. `music.example.com`) forwarding to
   `navidrome:4533` — see [Setting Up DNS and SSL](../README.md#setting-up-dns-and-ssl).

Do **not** add the Authelia snippets — Navidrome has its own login, and Subsonic
client apps authenticate directly against its API.
