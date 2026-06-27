# Security Policy

## Credential Handling

This project uses Ansible Vault (AES-256) to protect all SAP and Oracle passwords. The following rules are enforced:

- `vault.yml` is always committed **encrypted**. Never decrypt and commit.
- `.vault_pass` is **never committed** — it is listed in `.gitignore`.
- Passwords are never hardcoded in playbooks, roles, or inventory files.
- All password variables use the `vault_` prefix in `vault.yml` and are aliased to clean names in `sap_common.yml`.

## Reporting a Vulnerability

If you discover a security issue in this project (e.g., credentials accidentally committed, insecure default configuration), please **do not open a public issue**.

Instead, contact the maintainer directly via GitHub private message or email (listed in the repository profile).

Please include:
- Description of the vulnerability
- Steps to reproduce or locate the issue
- Suggested remediation if known

We will respond within 48 hours and work to address confirmed issues promptly.

## Production Recommendations

| Lab Setup | Production Replacement |
|-----------|------------------------|
| `.vault_pass` file | HashiCorp Vault, CyberArk, or AAP Credentials Manager |
| `host_key_checking = False` | `host_key_checking = True` with known_hosts management |
| `become = False` (running as root) | Dedicated service account with `become = True` |
| Self-signed or no TLS | Proper certificate management for SAP components |
