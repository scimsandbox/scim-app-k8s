# Database Backup to Hetzner Storage Box

This guide explains how to set up daily backups of the `scim-postgres-server` and `scim-postgres-validator` CloudNativePG databases directly into a Hetzner Storage Box via SSH/SFTP. The backup strategy uses [Restic](https://restic.net/) to provide incremental backups with deduplication and keeps exactly the last 3 backps.

## Architecture

We use a Kubernetes `CronJob` that spins up an Alpine container, installs `pg_dump` and `restic`, dumps the database over network via the `-rw` load balancer, and then uploads it using `restic backup --stdin`.

## Configuration Steps

1. **Update the Secrets file**

   An encrypted template for `backup-secrets.sops.yaml` has been provided in `k8s/app/backup/backup-secrets.sops.yaml`. If you have the `age` key configured, decrypt it to fill in your real Hetzner credentials and SSH keys, then encrypt it back:

   ```bash
   sops k8s/app/backup/backup-secrets.sops.yaml
   ```

   The secret should look like:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: backup-secrets
   type: Opaque
   stringData:
     storage-box-user: "u123456" # Replace with your storage box username
     storage-box-host: "u123456.your-storagebox.de" # Replace with your storage box host
     storage-box-path: "/home/restic-backups/scim" # Ensure the directory exists or restic will create it
     restic-password: "YOUR_STRONG_RESTIC_PASSWORD"
     ssh-password: "YOUR_SSH_PASSWORD"
   ```

2. **Apply the Backup CronJob**

   The backup mechanism is declared in `k8s/app/backup/backup-cronjob.yaml`. Apply it or add it to your `kustomization.yaml`:

   ```bash
   kubectl apply -f k8s/app/backup/backup-cronjob.yaml
   ```

## Managing Backups

### Manually trigger a backup

If you want to run the job instantly rather than waiting for 3 AM:

```bash
kubectl create job --from=cronjob/scim-db-backup manual-backup-1
# View logs:
kubectl logs -f job/manual-backup-1
```

### Checking Backups and Restic Status

To list the current snapshots, run a temporary command pod:

```bash
kubectl run -i --tty restic-admin --image=alpine:3.19 --restart=Never --rm \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "restic-admin",
        "image": "alpine:3.19",
        "command": ["/bin/sh"],
        "stdin": true,
        "tty": true,
        "env": [
          { "name": "STORAGE_BOX_USER", "valueFrom": { "secretKeyRef": { "name": "backup-secrets", "key": "storage-box-user" } } },
          { "name": "STORAGE_BOX_HOST", "valueFrom": { "secretKeyRef": { "name": "backup-secrets", "key": "storage-box-host" } } },
          { "name": "STORAGE_BOX_PATH", "valueFrom": { "secretKeyRef": { "name": "backup-secrets", "key": "storage-box-path" } } },
          { "name": "RESTIC_PASSWORD", "valueFrom": { "secretKeyRef": { "name": "backup-secrets", "key": "restic-password" } } },
          { "name": "SSHPASS", "valueFrom": { "secretKeyRef": { "name": "backup-secrets", "key": "ssh-password" } } }
        ],
        "volumeMounts": [{ "name": "backup-secrets", "mountPath": "/etc/backup-secrets", "readOnly": true }]
      }],
      "volumes": [{ "name": "backup-secrets", "secret": { "secretName": "backup-secrets" } }]
    }
  }'
```

Once inside the shell, type:
```bash
apk add --no-cache restic openssh-client sshpass
mkdir -p ~/.ssh
echo -e "Host *.your-storagebox.de\n\tStrictHostKeyChecking no\n" > ~/.ssh/config
export RESTIC_REPOSITORY="sftp:${STORAGE_BOX_USER}@${STORAGE_BOX_HOST}:${STORAGE_BOX_PATH}"

# List all snapshots:
sshpass -e restic snapshots
```

## Restoring from a backup

### Restore Process Steps

1. Get a restic admin shell as explained in "Checking Backups and Restic Status" above. Ensure you have `postgresql-client` installed:
   ```bash
   apk add --no-cache postgresql-client restic openssh-client curl sshpass
   ```

2. List your snapshots to find the ID you want to restore:
   ```bash
   sshpass -e restic snapshots
   ```

3. Set the DB Variables:
   ```bash
   SERVER_DB_PASSWORD=$(kubectl get secret scim-postgres-server-superuser -o jsonpath='{.data.password}' | base64 -d)
   ```

4. Restore a specific standard input dump directly to your database:
   ```bash
   # Make sure you setup SSH and exports first as demonstrated earlier
   export RESTIC_REPOSITORY="..."
   export RESTIC_PASSWORD="..."

   # Replace 'SNAPSHOT_ID' below with the actual snapshot ID from `restic snapshots`
   # Replace `scim-server.sql` with `scim-validator.sql` and change DB credentials appropriately if restoring the validator

   sshpass -e restic dump SNAPSHOT_ID scim-server.sql | PGPASSWORD=$SERVER_DB_PASSWORD psql -h scim-postgres-server-rw -U postgres scimserver
   ```

*Note: Since the database is actively writing, dropping existing tables/schema might be necessary prior to restore. You can execute `DROP SCHEMA public CASCADE; CREATE SCHEMA public;` using `psql` before piping the restore if needed.*
