# Authelia

[Authelia](https://www.authelia.com/) provides single sign-on and two-factor
authentication middleware. Protected proxy hosts in NPM delegate their auth
checks to Authelia via an `auth_request` subrequest.

## Configuration

Authelia's configuration lives in `./nginx_proxy_manager_data/authelia_config/`.
The key files are:

- `configuration.yml` — main Authelia config (JWT secret, session settings,
  LDAP/file provider, MFA)
- `users_database.yml` — local user accounts (when using the file authentication
  provider)

Refer to the [Authelia documentation](https://www.authelia.com/configuration/prologue/introduction/)
for full configuration options.

## Protecting an app with Authelia

For apps that don't have their own login (like Adminer), add these lines under
the **Advanced** tab in the NPM proxy host:

```nginx
include /config/nginx/snippets/authelia-location.conf;
include /config/nginx/snippets/authelia-authrequest.conf;
```

> **Do not** add the Authelia snippets to Vaultwarden — it manages its own
> authentication and the native Bitwarden client apps need direct API access.
