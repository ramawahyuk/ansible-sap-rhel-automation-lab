# 🚀 ansible-sap-rhel-automation

> **Ansible automation framework for SAP pre-installation preparation and environment standardization on Red Hat Enterprise Linux (RHEL 8).**  
> This repository documents my hands-on lab work building a repeatable, automated SAP deployment pipeline replacing manual OS preparation steps with idempotent Ansible playbooks validated against a two-node lab environment.
> 
[![Ansible](https://img.shields.io/badge/Ansible-2.21.1-red?logo=ansible)](https://www.ansible.com/)
[![RHEL](https://img.shields.io/badge/RHEL-8.x-red?logo=redhat)](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux)
[![SAP](https://img.shields.io/badge/SAP-ABAP%20%7C%20Oracle%2019c-blue?logo=sap)](https://www.sap.com/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![Collection](https://img.shields.io/badge/Collection-community.sap__install%201.9.2-orange)](https://galaxy.ansible.com/ui/repo/published/community/sap_install/)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#lab-architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Playbooks](#playbooks)
- [Ansible Vault](#ansible-vault)
- [Installation Sequence](#installation-sequence)
- [Troubleshooting](#troubleshooting)

---

## Overview

This project automates the full lifecycle of SAP software deployment using Ansible from OS baseline configuration to Oracle DB installation and SAP ABAP stack setup via SWPM. It implements security best practices (AES-256 vault encryption), idempotent role execution, and follows the `community.sap_install` collection conventions used in enterprise SAP projects.

**What this project automates:**

| Phase | Role / Playbook | Description |
|-------|----------------|-------------|
| 1 | `sap_general_preconfigure` | OS kernel tuning, SELinux, packages, `/etc/hosts` |
| 2 | `sap_netweaver_preconfigure` | Process limits, ABAP-specific packages |
| 3 | `sap_anydb_install_oracle` | Oracle DB 19c extraction, install, inventory |
| 4 | `sap_swpm` | Silent SWPM execution, SAP system creation |
| Util | `os_info.yml` | Pre-install read-only system information gather |
| Util | `vault_test.yml` | Vault decryption verification (no passwords printed) |

---

## Lab Architecture

<img width="1536" height="1024" alt="Ansible SAP Automation Architecture" src="https://github.com/user-attachments/assets/2da30101-55f2-407c-9e49-948f84a486ac" />

---

| Node | Hostname | IP Address | Role |
|------|----------|------------|------|
| ansnode | ansnode.local | 192.168.1.18 | Ansible Controller |
| saperp | saperp.local | 192.168.1.17 | SAP Application + DB Host |

**SAP System Details:**
- SID: `ERP`
- Database: Oracle Database 19c
- SAP Product: SAP ECC6 EHP8
- OS: RHEL 8.10

---
## Current Status

| Phase | Status | Notes |
|-------|--------|-------|
| OS Preparation | ✅ Tested | Validated on RHEL 8.10 |
| SAP Prerequisites | ✅ Tested | Validated against SAP ECC6 requirements |
| Oracle Prerequisites | 🔄 In Progress | Validated against Oracle 19c requirements |
| Environment Validation | 🔄 In Progress | Pre-SWPM checklist automated |
| SWPM Automation | 🔄 In Progress | Manual SWPM execution follows automated prep |
| Post-Install Automation | 📋 Planned | Future phase |

---
## Prerequisites

### Controller Node (`ansnode`)

- RHEL 8.x with active subscription
- Python 3.12
- Ansible 2.16 (`python3.12 -m pip install ansible`)
- SSH key-based access to SAP server (passwordless)
- Internet access for collection installation

### SAP Server (`saperp`)

- RHEL 8.x with active subscription
- All 5 SAP RHEL repositories enabled
- `/oracle` filesystem mounted (dedicated volume recommended)
- SAP installation media staged at `/oracle/software`
- Python 3.6

### Required RHEL Repositories

```bash
subscription-manager repos \
  --enable=rhel-8-for-x86_64-baseos-rpms \
  --enable=rhel-8-for-x86_64-appstream-rpms \
  --enable=rhel-8-for-x86_64-sap-solutions-rpms \
  --enable=rhel-8-for-x86_64-sap-netweaver-rpms \
  --enable=rhel-8-for-x86_64-highavailability-rpms
```

---

## Project Structure

```
~/sap-ansible/
│
├── ansible.cfg                          ← Engine config (inventory, vault, SSH tuning)
├── requirements.yml                     ← Collection declarations (pinned versions)
├── .vault_pass                          ← Vault password (chmod 600, never committed)
├── .gitignore                           ← Excludes vault_pass, logs, cache
│
├── inventory/
│   ├── hosts.yml                        ← Host definitions and group membership
│   └── group_vars/
│       ├── all/
│       │   └── common.yml               ← Variables for ALL hosts (timezone, domain)
│       ├── sap_servers/
│       │   ├── sap_common.yml           ← SAP SID, IP, preconfigure vars, pwd refs
│       │   └── vault.yml                ← AES-256 encrypted credentials
│       ├── sap_db/
│       │   └── oracle.yml               ← Oracle 19c role-specific variables
│       └── sap_app/                     ← (future: dedicated app server)
│
├── playbooks/
│   ├── os_info.yml                      ← Read-only pre-install system report
│   ├── vault_test.yml                   ← Verify vault decryption (no values printed)
│   ├── sap_general_preconfigure.yml     ← Step 1: OS baseline for SAP
│   ├── sap_netweaver_preconfigure.yml   ← Step 2: ABAP-specific tuning
│   ├── hana_install.yml                 ← (future: SAP HANA path)
│   └── swpm_install.yml                 ← Step 4: SAP ABAP via SWPM
│
├── roles/                               ← Custom roles (not collection roles)
│
└── collections/                         ← Project-local collection install path
    └── ansible_collections/
        └── community/
            └── sap_install/             ← 12 SAP automation roles live here
```

---

## Quick Start

### 1. Clone and Enter the Project

```bash
git clone https://github.com/ramawahyuk/ansible-sap-rhel-automation.git
cd ansible-sap-rhel-automation
```

### 2. Install Ansible

```bash
sudo dnf install -y python3.12 python3.12-pip
python3.12 -m pip install ansible
ansible --version | head -2
```

### 3. Configure Hosts

Edit `/etc/hosts` on the controller and SAP server:

```
192.168.1.18  ansnode.tes.com  ansnode
192.168.1.17  saperp.tes.com   saperp
```

### 4. Set Up SSH Key Access

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id root@192.168.1.17
ssh root@192.168.1.17   # verify passwordless access
```
Test ssh access to the sap server, it should not require any password

<img width="688" height="113" alt="image" src="https://github.com/user-attachments/assets/f2acdec9-9fb4-4dae-b69b-cc1553170c84" />


Assume that the [directory](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/directory_structure.md) structure already made 

### 5.Install Collections

```bash
ansible-galaxy collection install -r requirements.yml -p collections/ --force-with-deps
ansible-galaxy collection list | grep -E "sap_install|linux_system"
```

### 6. Create and Protect the Vault Password File

```bash
echo "YourStrongVaultPassword" > .vault_pass
chmod 600 .vault_pass
```

### 7. Create and Encrypt Vault

```bash
# Edit inventory/group_vars/sap_servers/vault.yml with your passwords first
ansible-vault encrypt inventory/group_vars/sap_servers/vault.yml
```

### 8. Verify Everything

Check the connectivity between Ansible controller & Managed node

```bash
# Test connectivity
cd ~/sap-ansible
ansible -m ping sap_servers
```
From inside ~/sap-ansible, run ping without specifying -i because [ansible.cfg](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/~/sap-ansible/ansible.cfg) now defines the default inventory

Expected Output:

<img width="461" height="93" alt="image" src="https://github.com/user-attachments/assets/083f3197-e884-48e0-b0d9-bcad122cb1a6" />

---



Check the timezone of Managed node

```
# Verify variables load
ansible -m debug -a "var=vars" saperp 2>/dev/null | grep -E "sap_|oracle_|timezone"

```

Expected Output:

<img width="921" height="183" alt="image" src="https://github.com/user-attachments/assets/dc56f87d-ea81-4272-a909-01b466530732" />

---



Verify Vault Decryption Works in a Playbook 

```
# Run vault verification
ansible-playbook playbooks/vault_test.yml

```

Create a vault test playbook that confirms decryption works without printing the actual passwords, named as [vault_test.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/playbooks/vault_test.yml):

Expected Output:

<img width="1031" height="644" alt="image" src="https://github.com/user-attachments/assets/af5fbff3-303c-498f-bb98-7ee3393c8e4a" />

---



Gather SAP System Information

```
# Run pre-install system report (read-only)
ansible-playbook playbooks/os_info.yml

```
Run [os_info.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/playbooks/os_info.yml) to gather and display SAP-relevant system information from saperp before any installation begins, read-only and makes zero changes to the target system.

Expected Output (partially):

<img width="1026" height="666" alt="image" src="https://github.com/user-attachments/assets/bb7bdb46-1f42-4dc2-9f69-bca248dfc3e0" />
<img width="1028" height="586" alt="image" src="https://github.com/user-attachments/assets/b5e2b793-f6e9-4268-b3fa-7ae163571cf0" />
<img width="1026" height="600" alt="image" src="https://github.com/user-attachments/assets/e34bcf61-459d-47ec-bd82-558fd79cf1a1" />


based on that we can see confirmed functional results such as:

```
Hostname    : saperp          ✅
FQDN        : saperp.tes.com  ✅
SAP SID     : ERP             ✅
SAP IP      : 192.168.1.17    ✅

OS          : RedHat 8.10     ✅
Kernel      : 4.18.0-553      ✅
CPUs        : 8 vCPU          ✅
RAM         : 31.1 GB         ✅

/           : 59.9GB  | 49.2GB free  | 17.8% used  ✅
/sapmnt     : 139.9GB | 138.9GB free | 0.7% used   ✅
/oracle     : 299.8GB | 297.7GB free | 0.7% used   ✅
/boot       : 50.0GB  | 49.3GB free  | 1.3% used   ✅
/usr/sap    : 149.9GB | 148.7GB free | 0.8% used   ✅
/home       : skipped  ← Correctly excluded by 'when' filter ✅

Python      : 3.12.12         ✅
Oracle base : /oracle exists  ✅ Is directory ✅

SAP Repos   : sap-netweaver-rpms ✅
              sap-solutions-rpms ✅

OS Check    : RedHat 8.10 SUPPORTED ✅

```



---




## Configuration

### `ansible.cfg` Key Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| `inventory` | `inventory/hosts.yml` | Default inventory (no `-i` needed) |
| `remote_user` | `root` | SSH user for all managed nodes |
| `host_key_checking` | `False` | Lab convenience (set `True` in production) |
| `forks` | `5` | Parallel host execution (increase for prod) |
| `vault_password_file` | `/root/sap-ansible/.vault_pass` | Auto vault decryption |
| `pipelining` | `True` | SSH performance — critical for SAP installs |
| `fact_caching` | `jsonfile` | Cache facts in `/tmp/ansible_facts_cache` |
| `inject_facts_as_vars` | `True` | Compatibility with `community.sap_install` roles |

### Network Configuration (Dual NIC Setup)

This lab uses two network adapters on both the controller and SAP server to separate SAP traffic from internet access (needed for collection installation).

## Adapter Roles

| Adapter | Type | Purpose | IP Range |
|---------|------|---------|----------|
| `ens160` | Bridge (Host-Only) | SAP internal network — Ansible SSH, SAP ports | 192.168.1.x |
| `ens224` | NAT | Internet access — package/collection downloads | DHCP (NAT) |

---

## Playbooks

### `os_info.yml` — Pre-Install System Report

Run [os_info.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/playbooks/os_info.yml) which is **Read-only. Makes zero changes.**

```bash
ansible-playbook playbooks/os_info.yml
```

Checks: hostname/FQDN, OS version, vCPU, RAM, swap, SAP-critical filesystems (`/`, `/oracle`, `/sapmnt`, `/usr/sap`), Python version, active SAP repositories, Oracle base directory, OS compatibility.

---

### `vault_test.yml` — Vault Verification

Create a [vault test](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/playbooks/vault_test.yml) playbook that confirms decryption works without printing the actual passwords:

```bash
ansible-playbook playbooks/vault_test.yml
```

This confirms all vault variables are defined and non-empty **without printing actual password values**. Shows character counts only.

---

### `sap_general_preconfigure.yml` — OS Baseline (Step 1)

Create [sap_general_configure.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/playbooks/sap_general_preconfigure.yml) to configure OS baseline for SAP software installation to implements SAP Notes: 2002167, 2772999, 3108302 (RHEL8), based on the community.sap_install.sap_general_preconfigure roles.

Do a dry run first, to evaluate every task and reports what it would do without making any actual changes. This is how we validate a playbook before committing to real changes on a SAP server.

```bash
# Dry run first
ansible-playbook playbooks/sap_general_preconfigure.yml --check

# Execute
ansible-playbook playbooks/sap_general_preconfigure.yml
```

---

## Ansible Vault

This project encrypts variable files using **file-level AES-256 encryption** for all credentials with the following pattern:

```
vault.yml           → encrypted (committed to git)
.vault_pass         → plain text password file (NEVER committed)
sap_common.yml      → references vault variables with {{ vault_* }} syntax
```

The encrypted file can be safely committed to version control. Only someone with the vault password can decrypt and read the values.

```
      WITHOUT VAULT                                   WITH VAULT

┌──────────────────────┐                    ┌─────────────────────────────┐
│       oracle.yml     │                    │          vault.yml          │
│                      │                    │                             │
│ oracle_sys_password: │                    │ $ANSIBLE_VAULT;1.1;AES256   │
│ Oracle123            │                    │ 3462396661613538...         │
└──────────┬───────────┘                    └──────────────┬──────────────┘
           │                                               │
           ▼                                               ▼
    👁️ Anyone can read                           🔒 Encrypted Data
           │                                               │
           ▼                                               ▼
    Password Exposed                           Requires Vault Password
                                                           │
                                                           ▼
                                                  🔑 Ansible Vault
                                                           │
                                                           ▼
                                                 Decrypts in Memory Only
                                                           │
                                                           ▼
                                               oracle_sys_password=*****
```

### Common Vault Operations

```bash
# View encrypted file
ansible-vault view inventory/group_vars/sap_servers/vault.yml

# Edit encrypted file
ansible-vault edit inventory/group_vars/sap_servers/vault.yml

# Re-encrypt with new password
ansible-vault rekey inventory/group_vars/sap_servers/vault.yml

# Decrypt temporarily (use with caution)
ansible-vault decrypt inventory/group_vars/sap_servers/vault.yml
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

For further explanation see [Vault Guide](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/VAULT_GUIDE.md) and [Security](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/SECURITY.md).

---

## Installation Sequence

```
Step 1: sap_general_preconfigure     ← OS tuning for ALL SAP
        - kernel parameters (vm.max_map_count, etc.)
        - package installation
        - SELinux → permissive
        - tmpfs configuration
        - /etc/hosts verification

Step 2: sap_netweaver_preconfigure   ← Additional ABAP tuning
        - process limits (nproc, nofile)
        - ABAP-specific packages

Step 3: sap_anydb_install_oracle     ← Oracle DB 19c install
        - extract Oracle media
        - run Oracle installer (silent)
        - apply SAP Oracle patch
        - create Oracle central inventory

Step 4: sap_swpm                     ← ABAP stack installation
        - run SWPM silently
        - create SAP system (ERP/SID)
        - configure Oracle schema (SAPSR3)
```

---

## Troubleshooting

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| `No vault secrets found` | `vault_password_file` outside `[defaults]` section | Move inside `[defaults]`, use absolute path `/root/sap-ansible/.vault_pass` |
| `Vault not found` | UTF-8 corruption in `ansible.cfg` from box-drawing chars | Rewrite with pure ASCII heredoc |
| `Vault not found` | Tilde `~` not expanded in `ansible.cfg` | Use absolute path `/root/sap-ansible/` |
| `YAML parse error` | Stray `"` at end of `vault.yml` from `vi` | Remove via `ansible-vault edit` |
| Ping fails | SSH key not copied | Run `ssh-copy-id root@192.168.1.17` |
| Wrong IP binding | SAP binding to `ens224` instead of `ens160` | Set explicit `sap_ip: "192.168.1.17"` in `sap_common.yml` |
| Collection roles fail | Missing `fedora.linux_system_roles` dependency | Add to `requirements.yml` and reinstall |

---

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

## Acknowledgements

- [community.sap_install](https://galaxy.ansible.com/ui/repo/published/community/sap_install/) — SAP automation collection by the Ansible SAP community
- [fedora.linux_system_roles](https://galaxy.ansible.com/ui/repo/published/fedora/linux_system_roles/) — Linux system roles dependency
- SAP Notes: 2002167, 2772999, 3108302 (RHEL 8 for SAP)

---

