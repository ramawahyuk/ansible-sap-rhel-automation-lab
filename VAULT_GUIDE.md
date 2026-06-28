# Ansible Vault Guide — SAP Credentials Management

This document covers everything you need to work with the encrypted vault files in this project.

---

## Why Vault?

Storing SAP and Oracle passwords in plain text is a common mistake that exposes credentials in:
- Git history (permanent, even after deletion)
- Shell history (`ansible-playbook site.yml -e "password=Welcome1"`)
- Process lists (`ps aux` shows command-line arguments)

Ansible Vault solves this with AES-256-CBC encryption. The encrypted file is safe to commit to version control. Only someone with the vault password can decrypt it.

There are two Vault patterns

1. Encrypt entire fileused for variables files, where the whole file becomens encrypted ciphertext, and only can be edited with `ansible-vault edit`.
2. Encrypt individual string used in YAML, which only the value is encrypted, and the variable name remains readable.

For this SAP automation we use patterns number one, where we encrypt entire vault file. This is cleaner, easier to manage, and the standard used by community.sap_install documentation.

```
#Encrypt entire vault file
ansible-vault encrypt inventory/group_vars/sap_servers/vault.yml

```

### Variables Encrypted in `vault.yml`

| Variable | Purpose | Used By |
|----------|---------|---------|
| `vault_oracle_sys_password` | Oracle SYS DBA superuser | `sap_anydb_install_oracle` |
| `vault_oracle_system_password` | Oracle SYSTEM DBA | `sap_anydb_install_oracle` |
| `vault_oracle_dbsnmp_password` | Oracle monitoring account | `sap_anydb_install_oracle` |
| `vault_sap_master_password` | SAP master (SAP*, DDIC) | `sap_swpm` |
| `vault_sap_db_schema_password` | SAP DB schema (SAPSR3) | `sap_swpm` |
| `vault_sap_diagnostics_agent_password` | SAP diagnostics agent | `sap_swpm` |

> **Production Note:** Replace `.vault_pass` file with HashiCorp Vault, CyberArk, or AAP Credentials Manager.

---

## How It Works

<img width="1536" height="1024" alt="How Ansible vault works" src="https://github.com/user-attachments/assets/2eae2833-2548-4777-943b-26b1c4ce9380" />


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

## Create vault password file

Create on `controller node`/ `ansnode`, the vault password file stores the password that encrypts/decrypts all vault files. It must be protected and never committed to version control:

```bash
cd ~/sap-ansible

```
Create vault password file

```bash
echo "YourStrongVaultPassword" > .vault_pass

#restrict permissions where only root can read it
chmod 600 .vault_pass

#verify permission
ls -la .vault_pass

```

<img width="419" height="48" alt="image" src="https://github.com/user-attachments/assets/e7822ba2-fe23-4d10-855e-751e4d76d6db" />

Then we need to tell [ansible.cfg](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/ansible.cfg) to use this file automatically, so no need to pass `--vault-password-file` on every command:

Add this file reference into ansible.cfg.

```
# ── Vault configuration ──────────────────────────────────────
# Automatically use this file for vault decryption
vault_password_file = .vault_pass

```

And to prevent accidental commits, we need to add `.vault_pass` to `.gitignore`:

```
echo ".vault_pass" > .gitignore
echo "ansible.log" >> .gitignore
echo "/tmp/ansible_facts_cache" >> .gitignore

cat .gitignore

```

<img width="381" height="75" alt="image" src="https://github.com/user-attachments/assets/db05d597-153b-482d-9319-feb151df3d0f" />

## Create the Encrypted Vault Files

Create SAP server vault file and put it at [~/sap-ansible/inventory/group_vars/sap_servers/vault.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/vault.yml.example) then encrypt it


### Encrypt a file
```bash
ansible-vault encrypt inventory/group_vars/sap_servers/vault.yml
```

### To verify it is actully encrypted check the vault.yml

```
cd ~/sap-ansible
cat inventory/group_vars/sap_servers/vault.yml

```

It is expected that it is already change into ciphertext and not readable:

<img width="657" height="106" alt="image" src="https://github.com/user-attachments/assets/f3cbfd14-40e0-4908-b0fd-6dfa5c766200" />

---

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

## Create password reference to stored the value in encrypted

add this into our [sap_common.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/sap_common.yml) file

```
# ── Password References (values stored encrypted in vault.yml) ──
oracle_sys_password: "{{ vault_oracle_sys_password }}"
oracle_system_password: "{{ vault_oracle_system_password }}"
oracle_dbsnmp_password: "{{ vault_oracle_dbsnmp_password }}"
sap_master_password: "{{ vault_sap_master_password }}"
sap_swpm_db_schema_password: "{{ vault_sap_db_schema_password }}"
sap_swpm_diagnostics_agent_password: "{{ vault_sap_diagnostics_agent_password }}"

```
Then to verify Vault decryption works on our system, we need to create [vault_test](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/vault_test.yml) playbook to confirms it without printing the actual password:

```
cd ~/sap-ansible

#run this to test if its works or not 
ansible-playbook playbooks/vault_test.yml

```

Expected Results:


<img width="722" height="662" alt="image" src="https://github.com/user-attachments/assets/02393d09-2cec-4862-92c5-335b127a5f06" />


The character count confirms the vault decrypted to the correct password length without ever displaying the actual values. This is the correct way to verify vault configuration in a runbook or audit.

---

## Troubleshooting

> This is some of error i founf during the works on this part:

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
