# Changelog

All notable changes to this project are documented here.  
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.0.0] - 2026-06-25

### Added
- Initial project structure: `inventory/`, `playbooks/`, `roles/`, `collections/`
- `ansible.cfg` with full SAP-tuned configuration (pipelining, fact caching, vault path, timer callbacks)
- `inventory/hosts.yml` — SAP server group with explicit Python 3.12 interpreter
- `inventory/group_vars/all/common.yml` — timezone and DNS domain for all hosts
- `inventory/group_vars/sap_servers/sap_common.yml` — SAP SID, IP binding, `sap_general_preconfigure` variables, vault password references
- `inventory/group_vars/sap_servers/vault.yml` — AES-256 encrypted SAP and Oracle credentials (6 variables)
- `inventory/group_vars/sap_db/oracle.yml` — `sap_anydb_install_oracle_*` role variable mapping for Oracle 19c
- `requirements.yml` — `community.sap_install >=1.9.0` and `fedora.linux_system_roles >=1.0.0`
- `playbooks/os_info.yml` — read-only 8-section pre-install system report
- `playbooks/vault_test.yml` — vault decryption verification (no password values printed)
- `playbooks/sap_general_preconfigure.yml` — Step 1 OS baseline with pre/post task verification
- `playbooks/swpm_install.yml` — SAP ABAP SWPM silent installation skeleton
- `.gitignore` — excludes `.vault_pass`, `ansible.log`, fact cache, keys
- `README.md` — full documentation with architecture diagram, quick start, and troubleshooting table
- `CONTRIBUTING.md`, `SECURITY.md`, `CHANGELOG.md`, `LICENSE`
- GitHub issue templates for bug reports and feature requests

### Fixed
- Vault `not found` error: `vault_password_file` must be inside `[defaults]` section with absolute path
- Tilde `~` not expanded in `ansible.cfg` — use `/root/sap-ansible/.vault_pass`
- UTF-8 corruption in `ansible.cfg` from terminal box-drawing characters — rewrote as pure ASCII
- YAML parse error in `vault.yml` from stray `"` added by `vi` — corrected via `ansible-vault edit`

### Security
- All credentials encrypted with AES-256 via Ansible Vault
- `.vault_pass` added to `.gitignore` to prevent accidental commits
