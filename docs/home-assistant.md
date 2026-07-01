# Home Assistant (`compose.homeassistant.yml`)

[Home Assistant](https://www.home-assistant.io/) is an open-source home
automation platform for controlling smart-home devices, all running locally.

## Enabling

Uncomment the `compose.homeassistant.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

On first start, Home Assistant creates its configuration under
`./homeassistant_data/` and serves its onboarding wizard on port `8123`.

## Running behind Nginx Proxy Manager

Home Assistant runs on the shared `nginx_proxy_manager_default` network and is
proxied by NPM like any other service.

1. Create a proxy host in NPM (e.g. `home.example.com`) forwarding to
   `homeassistant:8123` — see
   [Setting Up DNS and SSL](../README.md#setting-up-dns-and-ssl).
2. On the proxy host's **Details** tab, enable **Websockets Support** — Home
   Assistant's frontend relies on a WebSocket connection and will not load
   without it.
3. Tell Home Assistant to trust the reverse proxy. Add the following to
   `./homeassistant_data/configuration.yml`, then restart the container
   (`docker compose restart homeassistant`):

   ```yaml
   http:
     use_x_forwarded_for: true
     trusted_proxies:
       - 172.16.0.0/12   # Docker bridge networks
       - 192.168.0.0/16  # adjust to your LAN if needed
       - 10.0.0.0/8
   ```

   Without `trusted_proxies`, Home Assistant rejects proxied requests with a
   "400: Bad Request" error.

## Notes

- **Do not** add the Authelia snippets to the Home Assistant proxy host. Home
  Assistant manages its own authentication, and the companion mobile apps and
  API/WebSocket clients need direct access.
- The Bluetooth, Thread/Matter, and local device auto-discovery features (mDNS,
  SSDP, etc.) rely on the container sharing the host's network. The proxied
  bridge-network setup above does **not** expose those. If you need them, run
  Home Assistant with `network_mode: host` instead of the `networks`/`expose`
  block — note that this bypasses NPM, so you would reach it directly at
  `http://your-server-ip:8123`.
