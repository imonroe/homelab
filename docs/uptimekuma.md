# Uptime Kuma (`compose.uptimekuma.yml`)

[Uptime Kuma](https://github.com/louislam/uptime-kuma) is a self-hosted
monitoring tool — a slick dashboard that pings your hosts, HTTP endpoints, DNS,
and more, with notifications when something goes down.

## Enabling

Uncomment the `compose.uptimekuma.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

On first visit you create the admin account directly in the web UI. There are no
required environment variables; all monitors and settings persist in
`./uptimekuma_data/`.

## Running behind Nginx Proxy Manager

Uptime Kuma runs on the shared `nginx_proxy_manager_default` network and exposes
port `3001`.

1. Create a proxy host in NPM (e.g. `status.example.com`) forwarding to
   `uptimekuma:3001` — see [Setting Up DNS and SSL](../README.md#setting-up-dns-and-ssl).
2. Enable **Websockets Support** on the proxy host — Uptime Kuma's UI uses a
   WebSocket connection and won't load without it.

Protecting the dashboard with Authelia is **optional** — Uptime Kuma has its own
login. If you publish a status page for others to see, leave it unprotected (or
scope Authelia to the admin routes only).

## Notes

- Monitors can reach other containers on the `nginx_proxy_manager_default`
  network by their service name (e.g. monitor `http://portainer:9000`).
