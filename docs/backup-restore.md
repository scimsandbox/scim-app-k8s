# Database Backup to Google Drive

This guide explains how to set up daily backups of the `scim-postgres-server` and `scim-postgres-validator` CloudNativePG databases into Google Drive. Dumps are compressed, encrypted client-side with [rclone crypt](https://rclone.org/crypt/), and named by date. The last 14 days are kept.

## Architecture

A Kubernetes `CronJob` spins up an Alpine container, installs `pg_dump` and `rclone`, dumps each database over the network via its `-rw` service, and streams the result straight to Drive:

```
pg_dump | gzip -c | rclone rcat gdrive-crypt:scim-server-YYYY-MM-DD.sql.gz
```

Nothing is staged on local disk, so the job needs no meaningful ephemeral storage regardless of database size.

Two rclone remotes are involved. `gdrive` talks to Google Drive; `gdrive-crypt` wraps it and encrypts file *contents*. Filename encryption is deliberately off, so backups stay browsable as `scim-server-2026-08-08.sql.gz` in the Drive web UI while their contents remain unreadable to Google or to anyone with the folder link.

The `postgresql18-client` package must track the CloudNativePG major version in `k8s/app/database/` — `pg_dump` refuses to dump a server newer than itself.

## One-time Google setup

rclone's built-in shared OAuth client is being retired during 2026, so a dedicated client ID is required.

1. In the [Google Cloud Console](https://console.cloud.google.com/), create a project and enable the **Google Drive API**.

2. Configure the OAuth consent screen: user type **External**, then **publish it to production**.

   > Leaving the app in *Testing* expires every refresh token after 7 days, and the backup job will break weekly.

3. Under **Credentials**, create an **OAuth client ID** of type **Desktop app**. Note the client ID and secret.

4. Locally, install rclone (`brew install rclone`) and run `rclone config` twice:

   - Remote `gdrive` — type `drive`, using your own `client_id` and `client_secret`, and scope **`drive.file`**. Complete the browser consent step.
   - Remote `gdrive-crypt` — type `crypt`, remote `gdrive:scim-backups`, filename encryption `off`, directory name encryption `false`, and a strong password plus a *different* `password2` salt.

   `drive.file` is chosen on purpose: it is a non-sensitive scope, so publishing the app needs no Google verification review, and it restricts the job to files it created — it cannot read or delete anything else in your Drive.

   > **Store both crypt passwords in your password manager.** They are not recoverable from anywhere else, and without them every backup is permanently unreadable.

5. Edit `~/.config/rclone/rclone.conf` and add `use_trash = false` under `[gdrive]`, so pruned backups free up quota immediately instead of lingering in Drive's trash.

## Configuration

Copy the resulting config into the SOPS-encrypted secret:

```bash
sops k8s/app/backup/backup-secrets.sops.yaml
```

The `rclone.conf` key holds the config file verbatim — paste in the output of `cat ~/.config/rclone/rclone.conf`, replacing the placeholders:

```yaml
stringData:
  rclone.conf: |
    [gdrive]
    type = drive
    client_id = ....apps.googleusercontent.com
    client_secret = ...
    scope = drive.file
    token = {"access_token":"...","token_type":"Bearer","refresh_token":"...","expiry":"..."}
    use_trash = false

    [gdrive-crypt]
    type = crypt
    remote = gdrive:scim-backups
    filename_encryption = off
    directory_name_encryption = false
    suffix = none
    password = <obscured>
    password2 = <obscured>
```

The `password` and `password2` values must be in rclone's obscured form. They already are if the text came from `rclone config`; otherwise generate them with `rclone obscure <password>`.

Then apply the stack as usual — the CronJob is part of `k8s/app` and ships with both overlays:

```bash
kustomize build --enable-alpha-plugins --enable-exec k8s/app-spring | kubectl apply -f -
```

## Managing backups

### Manually trigger a backup

To run the job instantly rather than waiting for 3 AM:

```bash
kubectl create job -n scim --from=cronjob/scim-db-backup manual-backup-1
```

```bash
kubectl logs -n scim -f job/manual-backup-1
```

### Listing backups

From your laptop, with the same rclone config used to create them:

```bash
rclone lsl gdrive-crypt:
```

They are also visible directly in the Drive web UI under `scim-backups/`, though the contents will not open — that is the encryption working as intended.

### Getting an admin shell in the cluster

Useful for restores, since the databases are ClusterIP-only:

```bash
kubectl run -n scim backup-admin --image=alpine:3.23 --restart=Never --rm -it \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "backup-admin",
        "image": "alpine:3.23",
        "command": ["/bin/sh"],
        "stdin": true,
        "tty": true,
        "env": [
          { "name": "SERVER_DB_PASSWORD", "valueFrom": { "secretKeyRef": { "name": "scim-postgres-server-superuser", "key": "password" } } },
          { "name": "VALIDATOR_DB_PASSWORD", "valueFrom": { "secretKeyRef": { "name": "scim-postgres-validator-superuser", "key": "password" } } }
        ],
        "volumeMounts": [{ "name": "backup-secrets", "mountPath": "/etc/backup-secrets", "readOnly": true }]
      }],
      "volumes": [{ "name": "backup-secrets", "secret": { "secretName": "backup-secrets" } }]
    }
  }'
```

Once inside the shell:

```bash
apk add --no-cache postgresql18-client rclone ca-certificates
install -m 600 /etc/backup-secrets/rclone.conf /tmp/rclone.conf
export RCLONE_CONFIG=/tmp/rclone.conf

rclone lsl gdrive-crypt:
```

The config is copied out of the secret mount because rclone refreshes the OAuth token as it runs and needs somewhere writable to store it.

## Restoring from a backup

1. Get an admin shell as described above.

2. List the available backups and pick a date:

   ```bash
   rclone lsl gdrive-crypt:
   ```

3. Stream it straight back into the database:

   ```bash
   rclone cat gdrive-crypt:scim-server-2026-08-08.sql.gz | gunzip \
     | PGPASSWORD=$SERVER_DB_PASSWORD psql -h scim-postgres-server-rw -U postgres scimserver
   ```

   For the validator, substitute `scim-validator-<date>.sql.gz`, `$VALIDATOR_DB_PASSWORD`, `scim-postgres-validator-rw` and `scimvalidator`.

*Note: Since the database is actively writing, dropping existing tables/schema might be necessary prior to restore. You can execute `DROP SCHEMA public CASCADE; CREATE SCHEMA public;` using `psql` before piping the restore if needed.*

### Rehearsing a restore

Restore into a throwaway database rather than over a live one, and compare row counts afterwards:

```bash
PGPASSWORD=$VALIDATOR_DB_PASSWORD psql -h scim-postgres-validator-rw -U postgres -c 'CREATE DATABASE restoretest;'
```

Drop `restoretest` when finished. An untested backup is not a backup.

## How failures surface

The job is written so that a failure is always louder than a silent bad backup:

- `set -euo pipefail` means a `pg_dump` that dies mid-stream fails the whole job, rather than uploading a truncated dump as though it succeeded.
- `rclone rcat` creates the object before a mid-stream failure can surface, so a failed dump has its partial object deleted again — a failed run leaves nothing behind that could be mistaken for a real backup.
- Each upload is size-checked before the job proceeds; anything under 4 KiB is treated as a stub and fails the run.
- Retention runs only after both uploads have succeeded and passed that check, so a bad run can never prune good history.
