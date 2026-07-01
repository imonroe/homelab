# n8n (`compose.n8n.yml`)

[n8n](https://n8n.io/) is a fair-code workflow automation platform with 400+
integrations and native AI nodes — a self-hosted way to wire apps and AI models
together with little or no code.

## Enabling

Uncomment the `compose.n8n.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

## Data directory permissions

n8n runs as the container's `node` user (UID `1000`) and stores its data in
`/home/node/.n8n`. Create the host directory and give it to UID 1000 before the
first start, or n8n won't be able to write its settings:

```bash
mkdir -p n8n_data
sudo chown -R 1000:1000 n8n_data
```

On first start n8n generates an encryption key and saves it to
`./n8n_data/config`. **Back this file up** — it's required to decrypt your saved
credentials. To pin a fixed key instead, set `N8N_ENCRYPTION_KEY` in the
service's environment.

## Environment variables

Set these in `.env` so n8n generates correct links and webhook URLs behind the
proxy:

```env
# The public hostname n8n is served on
N8N_HOST=n8n.example.com
# Full external base URL, with trailing slash
N8N_WEBHOOK_URL=https://n8n.example.com/
# Protocol (defaults to https)
N8N_PROTOCOL=https
# Reverse proxies in front of n8n — NPM counts as 1 (the default)
N8N_PROXY_HOPS=1
```

`TZ`/`GENERIC_TIMEZONE` come from the shared `TZ` value in `.env`.

## Running behind Nginx Proxy Manager

n8n runs on the shared `nginx_proxy_manager_default` network and exposes port
`5678`.

1. Create a proxy host in NPM (e.g. `n8n.example.com`) forwarding to `n8n:5678`
   — see [Setting Up DNS and SSL](../README.md#setting-up-dns-and-ssl).
2. Enable **Websockets Support** on the proxy host — the editor UI relies on it.

You can optionally protect n8n with the
[Authelia snippets](authelia.md#protecting-an-app-with-authelia), but note that
inbound **webhooks** must remain reachable without authentication, so only add
Authelia if you aren't relying on externally-triggered webhooks.
