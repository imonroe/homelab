# OliveTin (`compose.olivetin.yml`)

[OliveTin](https://www.olivetin.app/) gives you a clean web interface for safely
running a predefined set of shell commands — handy for homelab operations like
restarting a service, triggering a backup, or running a script, without handing
out a terminal.

## Enabling

Uncomment the `compose.olivetin.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

## Configuration

OliveTin is driven by a `config.yaml` file mounted at `/config`. Create one at
`./olivetin_data/config.yaml` before starting the container — a minimal example:

```yaml
logLevel: "INFO"
actions:
  - title: Ping Google
    shell: ping -c 1 8.8.8.8
```

Full reference: [OliveTin configuration docs](https://docs.olivetin.app/).

There are no required environment variables. Actions and their commands all live
in `config.yaml`, which persists in `./olivetin_data/`.

## Running behind Nginx Proxy Manager

OliveTin runs on the shared `nginx_proxy_manager_default` network and exposes
port `1337`.

1. Create a proxy host in NPM (e.g. `ops.example.com`) forwarding to
   `olivetin:1337` — see [Setting Up DNS and SSL](../README.md#setting-up-dns-and-ssl).
2. Because OliveTin can run arbitrary commands, protect it with Authelia by
   adding the [auth snippets](authelia.md#protecting-an-app-with-authelia) to the
   proxy host.

## Controlling Docker from OliveTin

To let OliveTin actions manage other containers, it needs access to the Docker
socket. Uncomment the socket mount in `compose.olivetin.yml`:

```yaml
- /var/run/docker.sock:/var/run/docker.sock
```

> Mounting the raw Docker socket grants root-equivalent control of the host. For
> anything exposed beyond your LAN, put a
> [socket-proxy](https://docs.olivetin.app/action_examples/docker-proxy) in front
> of it and restrict the allowed operations.
