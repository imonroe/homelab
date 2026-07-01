# Vaultwarden (`compose.vaultwarden.yml`)

[Vaultwarden](https://github.com/dani-garcia/vaultwarden) is a lightweight,
Bitwarden-compatible password manager.

## Enabling

Uncomment the `compose.vaultwarden.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

## Required environment variables

Set these in `.env`:

```env
VAULTWARDEN_DOMAIN=https://vault.example.com
VAULTWARDEN_SIGNUPS_ALLOWED=false
VAULTWARDEN_ADMIN_TOKEN=your-secret-token
```

Generate a secure admin token:

```bash
openssl rand -base64 48
```

## Notes

- In NPM, proxy `vault.example.com` to `vaultwarden:80` — **do not** add the
  Authelia snippets
- Set `VAULTWARDEN_SIGNUPS_ALLOWED=true` temporarily to register your account,
  then set it back to `false` and restart: `docker compose restart vaultwarden`
- The admin panel is available at `https://vault.example.com/admin` — protect
  this URL with a strong `ADMIN_TOKEN`
- Vaultwarden **requires HTTPS** to function; always proxy it through NPM with
  SSL enabled
