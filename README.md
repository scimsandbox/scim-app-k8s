# SCIM Sandbox — Kubernetes deployment

This repository contains the Kubernetes deployment manifests, Kustomize
overlays, and cluster bootstrap helpers for the SCIM Sandbox.

What you'll find here:

- `k8s/app/` — shared namespaced application base (namespace, CloudNativePG, shared apps)
- `k8s/app-spring/` — Spring server implementation overlay
- `k8s/app-go/` — Go server implementation overlay
- `k8s/cluster/` — cluster-level support resources (storage, cloudflared, etc.)
- `.sops.yaml` and encrypted secret files used by `ksops`/Kustomize

This README highlights how to apply the Kubernetes manifests and the
prerequisites for using the encrypted secrets. If you need the full
application source or the old multi-module build, see the corresponding
application repository (the former monorepo that contained the Spring Boot
modules).

## Quick start (Kubernetes)

Prerequisites:

- `kubectl` configured for your cluster
- `kustomize` CLI with exec-plugin support for `ksops`
- `sops` and `ksops` available to Kustomize (for encrypted secrets)
- an `age` key configured for decrypting the SOPS files (or appropriate KMS)
- (optional) CloudNativePG support in the cluster if you intend to use the
  provided PostgreSQL manifests

Apply cluster-level resources, then the application overlay:

```bash
export SOPS_AGE_KEY_FILE=~/path/to/age/keys.txt

# cluster resources (storage classes, cloudflared namespace, etc.)
kustomize build --enable-alpha-plugins --enable-exec k8s/cluster | kubectl apply -f -

# application stack with the Spring server implementation
kustomize build --enable-alpha-plugins --enable-exec k8s/app-spring | kubectl apply -f -

# application stack with the Go server implementation instead of Spring
kustomize build --enable-alpha-plugins --enable-exec k8s/app-go | kubectl apply -f -
```

Notes:

- Secrets under `k8s/**/secrets/*.sops.yaml` are encrypted with SOPS; ensure
  your `SOPS_AGE_KEY_FILE` or other KMS config is available to decrypt them at
  `kustomize` build time.
- `k8s/app` is the shared application base without a server implementation.
  Apply either `k8s/app-spring` or `k8s/app-go` so you deploy exactly one
  server implementation. Both overlays reuse the same Deployment and
  in-cluster Service name, `scim-server-impl`, so switching overlays replaces
  the running implementation instead of creating a second server deployment.
- The manifests target an internal (ClusterIP) deployment model; external
  access is commonly provided through the `cloudflared` tunnel included in
  `k8s/cluster` or by adding an ingress of your choice.

## Database Backup & Restore

To ensure data safety, we have set up daily automated database backups using Restic to a Hetzner Storage Box via SFTP. Incremental backups are executed via a CronJob (dumped through network directly as `stdin/stdout`), and exact steps for configuration and restoration are available in [`docs/backup-restore.md`](docs/backup-restore.md).

## SOPS / age key rotation

Rotate the age recipient by updating the recipient in `.sops.yaml` and then
re-encrypting the tracked secret files:

```bash
find k8s -name '*.sops.yaml' -exec sops updatekeys --yes {} \;
```

Make sure the existing SOPS backend is still available while you rotate keys so
SOPS can decrypt the current file contents during the update.

## Repository layout (Kubernetes-focused)

```text
.
├── docs/
│   └── backup-restore.md # documentation for backing up and restoring databases
├── k8s/
│   ├── app/             # shared namespaced application base
│   ├── app-go/          # Go server implementation overlay
│   ├── app-spring/      # Spring server implementation overlay
│   └── cluster/         # cluster-level resources and operators
├── .sops.yaml           # SOPS configuration
└── README.md            # this file
```
