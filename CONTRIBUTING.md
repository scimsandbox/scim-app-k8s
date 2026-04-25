# Contributing

Thanks for contributing to scim-app-k8s.

This repository contains the Kubernetes deployment manifests, Kustomize
overlays, and cluster bootstrap helpers for the SCIM Sandbox project.
Keep changes focused on deployment configuration, secret management,
or documentation that matches the live repository structure.

## Ground Rules

- Keep each change narrow and intentional.
- Do not mix unrelated refactors into manifest, workflow, or
  documentation changes.
- Do not commit decrypted secrets, age private keys, or
  cluster-specific credentials.
- Prefer the existing Kustomize overlay structure over new abstraction
  layers unless there is a clear gain.
- Update docs when deployment behavior or secret management changes.

## Before You Start

1. Check for existing issues or pull requests that already cover the same work.
2. Read [README.md](./README.md) before changing deployment manifests or secret management.
3. If the change touches encrypted secrets, verify that `.sops.yaml` rules still apply correctly.

## Validation

Validate changes before opening a PR.

Common checks:

- run `kustomize build k8s/app-spring` or `kustomize build k8s/app-go` to verify the selected overlay renders cleanly
- verify encrypted files decrypt correctly with the intended age key
- check that no plaintext secrets appear in the diff

## Pull Request Checklist

- explains the deployment or infrastructure change and why it is needed
- updates docs when deployment behavior changes
- keeps secrets, private keys, and cluster-specific values out of the diff
- avoids unrelated cleanup
- passes the relevant validation steps

## Reporting Bugs

When reporting a deployment issue, include:

- the affected manifest or overlay
- the Kubernetes version and cluster setup
- the expected and actual behavior
- relevant logs with secrets removed

## Security Issues

Do not report vulnerabilities through public issues.

Follow [SECURITY.md](./SECURITY.md) instead.
