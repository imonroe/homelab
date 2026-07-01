# MySQL + Adminer (`compose.mysql.yml`)

A general-purpose MySQL 8 database server with [Adminer](https://www.adminer.org/),
a web-based database admin UI.

## Enabling

Uncomment the `compose.mysql.yml` line in the `include:` block of
`docker-compose.yml`, then run `docker compose up -d`. See
[Adding or Removing Apps](../README.md#adding-or-removing-apps) in the main README.

## Required environment variables

Set these in `.env`:

```env
MYSQL_ROOT_PASSWORD=a-strong-password
MYSQL_DATABASE=homelab       # optional, defaults to "homelab"
```

## Notes

- MySQL is not exposed on a host port — connect to it from other containers on
  the `mysql_internal` network using the hostname `mysql` and port `3306`
- Adminer (web-based database UI) should be proxied through NPM and protected by
  Authelia. Forward to `adminer:8080`
