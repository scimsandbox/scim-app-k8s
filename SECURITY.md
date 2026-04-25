# Security Policy

## Supported Versions

Only the latest version on the `main` branch receives security updates.

## Reporting a Vulnerability

Do not report security vulnerabilities through public issues.

Use GitHub's private vulnerability reporting:
**Security → Report a vulnerability** on this repository.

Include:

- a description of the vulnerability
- the affected file or manifest
- reproduction steps or a proof of concept
- any suggested mitigation

You will receive a response within a reasonable timeframe.

## Scope of Security Review

Areas that are security-sensitive in this repository:

- **Encrypted secrets**: SOPS-encrypted YAML files must never be committed
  in decrypted form. All secrets are encrypted at rest using age keys.
- **Age key management**: The age private key used for decryption must
  never appear in version control.
- **Kubernetes RBAC**: Service account and role bindings should follow the
  principle of least privilege.
- **Network policies**: Cluster-level network configuration should restrict
  traffic to only the required paths.

## Secrets Handling

- `.sops.yaml` defines encryption rules — all matching files are
  automatically encrypted.
- Decrypted secrets are listed in `.gitignore` to prevent accidental commits.
- Never commit `*.agekey`, `*.decrypted.yaml`, or `*.dec.yaml` files.

## Security Testing Expectations

- Verify that `git diff` does not contain plaintext secrets before pushing.
- Verify that SOPS encryption rules cover all sensitive files.
- Review RBAC changes for privilege escalation.
