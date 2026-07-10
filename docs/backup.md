# Backups

Backups run as a companion container —
[offen/docker-volume-backup](https://github.com/offen/docker-volume-backup) —
defined in `compose.backup.yml`. On a schedule it archives the mounted service
data directories into a single tar.gz, optionally encrypts it, rotates old
archives, and can push a copy off-box to S3, WebDAV, SSH, Azure, Dropbox, or
Google Drive.

A backup you never restore is a rumour. Read the [Restoring](#restoring)
section and do a test restore before you need one for real.

## Enabling backups

1. Uncomment the backup line in the `include:` block of `docker-compose.yml`:

   ```yaml
   include:
     # …
     - compose.backup.yml        # Automated encrypted backups
   ```

2. Set the backup values in your `.env` (all have sensible defaults):

   ```bash
   BACKUP_CRON_EXPRESSION=@daily     # when to run
   BACKUP_RETENTION_DAYS=7           # prune archives older than this
   BACKUP_GPG_PASSPHRASE=            # set this — see Encryption below
   ```

3. Bring it up:

   ```bash
   docker compose up -d
   ```

That's it — the container schedules itself. Archives are written to `./backups/`
on the host and pruned automatically. No host cron required.

## What gets backed up

The backup container archives whatever is mounted under `/backup`. By default
`compose.backup.yml` mounts the **core stack** data read-only:

- `nginx_proxy_manager_data/` (includes Authelia config and Let's Encrypt certs)
- `portainer_data/`
- `redis/`

For **optional apps**, uncomment the matching mount line in
`compose.backup.yml` once you've enabled the app, e.g.:

```yaml
    volumes:
      # …
      - ./vaultwarden_data:/backup/vaultwarden_data:ro
      - ./pihole_data:/backup/pihole_data:ro
```

Media libraries (Plex, Navidrome) are intentionally not mounted — they live
outside the repo (`PLEX_MEDIA_DIR`, `NAVIDROME_MUSIC_DIR`) and are typically far
too large. Back those up with your own media strategy. Only `plex_data/config`
is mounted for Plex, not the transcode scratch.

### Consistency for databases

Copying a database's files while it's mid-write can produce a corrupt archive.
To avoid that, the backup container can stop a service for the (few seconds)
duration of the run and restart it afterward, via a
`docker-volume-backup.stop-during-backup` label. Two services ship with this
label **commented out, ready to enable**:

- **MySQL** (`compose.mysql.yml`)
- **Vaultwarden** (`compose.vaultwarden.yml`)

When you uncomment a service's data mount in `compose.backup.yml`, also uncomment
the stop label in that service's compose file. Do both together: the mount
without the label risks a mid-write copy, and the label without the mount just
stops the service during backups without archiving anything — pure downtime for
no gain. That's why the labels ship commented rather than active by default.

To apply the same treatment to another app, add this label to its service (only
alongside mounting its data in `compose.backup.yml`):

```yaml
    labels:
      - docker-volume-backup.stop-during-backup=true
```

#### Zero-downtime MySQL dump (alternative)

If you'd rather not stop MySQL at all, take a logical dump instead of copying
its files. Leave the stop label commented and instead add `archive-pre` /
`archive-post` labels plus a shared dump volume to the `mysql` service:

```yaml
  mysql:
    volumes:
      - ./mysql_data:/var/lib/mysql
      - ./mysql_dump:/dump
    labels:
      - docker-volume-backup.archive-pre=/bin/sh -c 'mysqldump -uroot -p"$$MYSQL_ROOT_PASSWORD" --all-databases --single-transaction --routines --events > /dump/all-databases.sql'
```

Then mount `./mysql_dump:/backup/mysql_dump:ro` in `compose.backup.yml` and drop
the `./mysql_data` mount. The `$$` escapes the variable so the shell inside the
container expands it at runtime.

## Encryption

**Set `BACKUP_GPG_PASSPHRASE`.** These archives contain your Authelia user
database, your Vaultwarden vault, and Let's Encrypt private keys. With a
passphrase set, every archive is encrypted symmetrically with GPG and written
with a `.gpg` extension. Generate a strong passphrase and store it somewhere
separate from the backups (a real password manager, not the one you're backing
up):

```bash
openssl rand -base64 32
```

Without it, archives are stored in plaintext — acceptable only if `./backups`
never leaves a disk you fully control, and never for an off-box copy.

## Off-box copies

Local archives protect against a fat-fingered config change, but not against the
disk (or the whole box) dying. Ship a copy elsewhere by setting the S3 values in
`.env` and uncommenting the matching `AWS_*` lines in `compose.backup.yml`:

```bash
BACKUP_S3_BUCKET=my-homelab-backups
BACKUP_S3_ENDPOINT=s3.us-west-002.backblazeb2.com   # e.g. Backblaze B2
BACKUP_S3_ACCESS_KEY=…
BACKUP_S3_SECRET_KEY=…
```

Any S3-compatible provider works (AWS S3, Backblaze B2, MinIO, Filebase, …).
offen also supports WebDAV, SSH/SFTP, Azure Blob, Dropbox, and Google Drive —
see the [configuration reference](https://offen.github.io/docker-volume-backup/reference/)
for the relevant environment variables, and add them to the backup service's
`environment:` block. Retention (`BACKUP_RETENTION_DAYS`) applies to remote
copies too.

## Running a backup on demand

The container backs up on its cron schedule, but you can trigger one immediately:

```bash
docker compose exec backup backup
```

Check what it's doing with `docker compose logs -f backup`.

## Restoring

Archives nest everything under a top-level `backup/` directory, so restoring
into the repo root uses `--strip-components=1`.

```bash
cd /path/to/homelab

# 1. Stop the stack (or just the service you're restoring)
docker compose down

# 2. If the archive is encrypted (.tar.gz.gpg), decrypt it first
gpg --output homelab-backup.tar.gz --decrypt backups/homelab-backup-<timestamp>.tar.gz.gpg

# 3. Inspect before overwriting anything (wise)
tar tzf homelab-backup.tar.gz | less

# 4. Extract the data directories back into the repo root.
#    --strip-components=1 drops the leading "backup/" path element.
#    This overwrites current data — make sure you want that.
tar xzf homelab-backup.tar.gz --strip-components=1

# 5. Bring the stack back up
docker compose up -d
```

### Restoring a single service

Pull just the directory you need out of the archive:

```bash
# (decrypt first if the archive is .gpg, as above)
tar xzf homelab-backup.tar.gz --strip-components=1 backup/vaultwarden_data
docker compose restart vaultwarden
```

## Notifications (optional)

To be told when a backup fails (or every run), set a
[shoutrrr](https://containrrr.dev/shoutrrr/) URL. Uncomment `NOTIFICATION_URLS`
in `compose.backup.yml` and set `BACKUP_NOTIFICATION_URLS` in `.env` — e.g. a
Slack, Discord, Telegram, or SMTP URL. Add `NOTIFICATION_LEVEL: info` to the
service to be notified on success too, not just failure.

## Security note

Treat the archives as secret material: they hold password vaults, the Authelia
user database, and TLS private keys. Encrypt them (`BACKUP_GPG_PASSPHRASE`),
store them somewhere access-controlled, and prefer an encrypted off-box
destination. The `backups/` directory is git-ignored so archives are never
committed.

The backup container also mounts the Docker socket (`/var/run/docker.sock`), so
that it can stop and restart labeled services around a run. Access to the Docker
socket — even read-only — is effectively root on the host: anything that can talk
to the daemon can start a privileged container and escape. Treat the backup
container as a privileged component: only enable it if you trust the image
(`offen/docker-volume-backup`), pin it to a specific version tag rather than
tracking `latest` for production, and don't expose the socket more widely.
