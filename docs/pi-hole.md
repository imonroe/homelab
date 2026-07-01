# Pi-hole (`compose.pihole.yml`)

[Pi-hole](https://pi-hole.net/) is a network-wide DNS sinkhole that blocks ads
and trackers for every device on your network, without client-side software.

## Enabling

Uncomment the `compose.pihole.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

## Required environment variables

Set these in `.env`:

```env
PIHOLE_WEBPASSWORD=a-strong-password
# Optional — upstream resolvers, semicolon-delimited (defaults to Cloudflare)
PIHOLE_DNS_UPSTREAMS=1.1.1.1;1.0.0.1
```

If `PIHOLE_WEBPASSWORD` is left empty, Pi-hole generates a random password on
first start — retrieve it with `docker compose logs pihole`.

## DNS port (host)

Pi-hole publishes port **53** (TCP and UDP) directly on the host so that other
devices can use your server as their DNS resolver.

> **Port 53 conflict:** many Linux hosts run `systemd-resolved`, which already
> binds port 53 and will make the container fail to start. Free the port first:
>
> ```bash
> sudo sed -i 's/#DNSStubListener=yes/DNSStubListener=no/' /etc/systemd/resolved.conf
> sudo systemctl restart systemd-resolved
> ```

## Using Pi-hole as your DNS

Point your devices at the server's IP address to start filtering:

- **Whole network:** set your router's DNS server to the server's LAN IP so
  every device uses Pi-hole automatically.
- **Single device:** set that device's DNS server to the server's LAN IP.

The `FTLCONF_dns_listeningMode: 'ALL'` setting in the compose file is required
because Pi-hole runs on a Docker bridge network — without it, Pi-hole would
ignore queries arriving from outside its own subnet.

## Web admin behind Nginx Proxy Manager

The admin dashboard runs on port `80` inside the container and is reachable at
`/admin`.

1. Create a proxy host in NPM (e.g. `pihole.example.com`) forwarding to
   `pihole:80` — see [Setting Up DNS and SSL](../README.md#setting-up-dns-and-ssl).
2. Browse to `https://pihole.example.com/admin` and log in with
   `PIHOLE_WEBPASSWORD`.

Protecting the dashboard with Authelia is **optional** — Pi-hole already has its
own login. If you do add the [Authelia snippets](authelia.md#protecting-an-app-with-authelia),
apply them only to the web admin host, not to the DNS service.

## Notes

- **DHCP server:** to let Pi-hole also hand out DHCP leases, it needs the
  `NET_ADMIN` capability and direct access to the LAN (`network_mode: host`),
  plus the `67:67/udp` port. This is beyond the default proxied setup — see the
  [Pi-hole Docker docs](https://docs.pi-hole.net/docker/) if you need it.
- Pi-hole's databases and configuration persist in `./pihole_data/`.
