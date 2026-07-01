# Nginx Proxy Manager

[Nginx Proxy Manager](https://nginxproxymanager.com/) (NPM) is the front door for
all your apps. It handles reverse proxying and automatic SSL via Let's Encrypt.

## Ports

- Admin UI runs on port **81**
- HTTP traffic enters on port **8100** (mapped from 80 inside the container)
- HTTPS traffic enters on port **8143** (mapped from 443)
- On a public server, make sure ports **80** and **443** on your router/firewall
  forward to ports **8100** and **8143** on your server

## Default credentials

| URL | Default credentials |
|---|---|
| http://your-server-ip:81 | admin@example.com / changeme |

> **Important:** Change the NPM default password immediately after your first login.

See [Setting Up DNS and SSL](../README.md#setting-up-dns-and-ssl) in the main
README for a walkthrough of creating proxy hosts and issuing certificates.
