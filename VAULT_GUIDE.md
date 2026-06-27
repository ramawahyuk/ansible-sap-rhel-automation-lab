# Ansible Vault Guide — SAP Credentials Management

This document covers everything you need to work with the encrypted vault files in this project.

---

## Why Vault?

Storing SAP and Oracle passwords in plain text is a common mistake that exposes credentials in:
- Git history (permanent, even after deletion)
- Shell history (`ansible-playbook site.yml -e "password=Welcome1"`)
- Process lists (`ps aux` shows command-line arguments)

Ansible Vault solves this with AES-256-CBC encryption. The encrypted file is safe to commit to version control. Only someone with the vault password can decrypt it.

---

## How It Works

```
ENCRYPTION (you run once):
  ansible-vault encrypt inventory/group_vars/sap_servers/vault.yml
        |
        v
  Ansible uses AES-256-CBC
  Key derived from your vault password
        |
        v
  Produces: $ANSIBLE_VAULT;1.1;AES256
            346239666161353...

DECRYPTION (Ansible runs automatically at playbook start):
  ansible-playbook playbooks/sap_general_preconfigure.yml
        |
        v
  Ansible reads .vault_pass
  Decrypts vault variables in memory
  Variables available to roles as plain strings
  Never written to disk unencrypted
        |
        v
  Role receives: oracle_sys_password = "your-actual-password"
  (in memory only - never logged, never stored plain)
```

---

## Variable Naming Convention

This project uses a two-layer naming pattern:

| Layer | File | Variable Name | Purpose |
|-------|------|---------------|---------|
| Vault | `vault.yml` | `vault_oracle_sys_password` | Encrypted storage |
| Reference | `sap_common.yml` | `oracle_sys_password` | What roles actually use |

Roles and playbooks only reference the clean names (without `vault_` prefix). The mapping in `sap_common.yml` connects them:

```yaml
oracle_sys_password: "{{ vault_oracle_sys_password }}"
```

---

## Common Operations

### Create vault password file
```bash
echo "YourStrongVaultPassword" > .vault_pass
chmod 600 .vault_pass
```

### Encrypt a file
```bash
ansible-vault encrypt inventory/group_vars/sap_servers/vault.yml
```

### Decrypt temporarily (use with care)
```bash
ansible-vault decrypt inventory/group_vars/sap_servers/vault.yml
# ... make changes ...
ansible-vault encrypt inventory/group_vars/sap_servers/vault.yml
```

### Edit encrypted file (recommended over decrypt/encrypt cycle)
```bash
ansible-vault edit inventory/group_vars/sap_servers/vault.yml
```

### View encrypted file without editing
```bash
ansible-vault view inventory/group_vars/sap_servers/vault.yml
```

### Change vault password
```bash
ansible-vault rekey inventory/group_vars/sap_servers/vault.yml
# (prompts for old password, then new password)
# Also update .vault_pass with the new password
```

---

## Troubleshooting

### "Attempting to decrypt but no vault secrets found"

The most common cause: `vault_password_file` is outside the `[defaults]` section in `ansible.cfg`.

**Wrong:**
```ini
[ssh_connection]
vault_password_file = ~/.vault_pass   # wrong section!
```

**Correct:**
```ini
[defaults]
vault_password_file = /root/sap-ansible/.vault_pass   # absolute path, correct section
```

### "~" tilde not expanded

`ansible.cfg` does not expand `~`. Always use the full absolute path:

```ini
vault_password_file = /root/sap-ansible/.vault_pass   # correct
vault_password_file = ~/.vault_pass                    # does NOT work
```

### YAML parse error in vault.yml

If you edited `vault.yml` with `vi` before encrypting and it left a stray character:

```bash
ansible-vault edit inventory/group_vars/sap_servers/vault.yml
# This opens in your $EDITOR cleanly without the encryption wrapper causing issues
```

---

## Production Recommendations

For production environments, replace `.vault_pass` with:

- **HashiCorp Vault** — `vault_password_file` pointing to a script that retrieves from HCP Vault
- **CyberArk** — Use the CyberArk Ansible module for credential retrieval
- **AAP Credentials** — Ansible Automation Platform handles vault passwords natively
- **AWS Secrets Manager / Azure Key Vault** — via lookup plugins
