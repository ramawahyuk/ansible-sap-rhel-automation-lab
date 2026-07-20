# 🚀 ansible-sap-rhel-automation

> **Ansible automation framework for SAP pre-installation preparation and environment standardization on Red Hat Enterprise Linux (RHEL 8).**  
> This repository documents my hands-on lab work building a repeatable, automated SAP deployment pipeline replacing manual OS preparation steps with idempotent Ansible playbooks following the `community.sap_install` collection conventions used in enterprise SAP projects. Credentials are protected with AES-256 Ansible Vault encryption. The lab originally targeted ECC6 on Oracle 19c and has since moved to the SAP HANA / S/4HANA stack and validated against a two-node lab environment.
> 
[![Ansible](https://img.shields.io/badge/Ansible-2.21.1-red?logo=ansible)](https://www.ansible.com/)
[![RHEL](https://img.shields.io/badge/RHEL-8.x-red?logo=redhat)](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux)
[![SAP](https://img.shields.io/badge/SAP-S%2F4HANA%201709%20%7C%20HANA%20DB-blue?logo=sap)](https://www.sap.com/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![Collection](https://img.shields.io/badge/Collection-community.sap__install%201.9.2-orange)](https://galaxy.ansible.com/ui/repo/published/community/sap_install/)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#lab-architecture)
- [Prerequisites](#prerequisites)
- [Project Directory Structure](#project-directory-structure)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Playbooks](#playbooks)
- [Ansible Vault](#ansible-vault)
- [Installation Sequence](#installation-sequence)
- [Troubleshooting](#troubleshooting)

---

## Overview

This project automates the full lifecycle of SAP software deployment using Ansible from OS baseline configuration to SAP S/4HANA (AS ABAP, OneHost) installation via SWPM against an existing SAP HANA database. It implements security best practices (AES-256 vault encryption), idempotent role execution, and follows the `community.sap_install` collection conventions used in enterprise SAP projects.

**What this project automates:**

| Phase | Role / Playbook | Description |
|-------|----------------|-------------|
| 1 | `sap_general_preconfigure` | OS kernel tuning, SELinux, packages, `/etc/hosts` |
| 2 | `sap_netweaver_preconfigure` | Process limits, ABAP-specific packages |
| 3 | `sap_install_media_detect` | Detect/extract SAP HANA client and installation media staged under `/sapmedia` |
| 4 | `sap_hana_preconfigure` | Covers HANA specific OS Tuning |
| 5 | `swpm_install` | Silent SWPM execution, SAP S/4HANA AS ABAP system creation against an existing HANA DB |
| Util | `os_info.yml` | Pre-install read-only system information gather |
| Util | `vault_test.yml` | Vault decryption verification (no passwords printed) |

> **Note:** This automation connects SWPM to an already-running SAP HANA instance (installed separately, outside this repo's scope) rather than installing HANA itself. HANA client media detection/extraction is handled by `sap_install_media_detect`.

---

## Lab Architecture

<img width="1535" height="1024" alt="SAP_S4HANA DIAG" src="https://github.com/user-attachments/assets/c108a847-3574-41c0-a08a-a7d399b6f528" />



| Node | Hostname | IP Address | Role |
|------|----------|------------|------|
| ansnode | ansnode.local | 192.168.1.18 | Ansible Controller |
| saperp | saperp.local | 192.168.1.17 | SAP Application + DB Host |

**SAP System Details:**
- Application SID: `NW1`
- HANA DB SID: `S4D` (pre-existing instance, instance number `00`)
- Database: SAP HANA
- SAP Product: SAP S/4HANA 1709, AS ABAP OneHost (ASCS + PAS combined, `NW_ABAP_OneHost:S4HANA1709.CORE.HDB.ABAP`)
- ASCS instance: `01` · PAS instance: `02`
- OS: RHEL 8.10

> ⚠️ **TODO:** Confirm and update `saperp`'s IP/hostname resolution below — the host has two NICs (bridged `ens160` and NAT `ens224`), and SAP's registered hostname currently resolves via whichever adapter `/etc/hosts` points to. Verify this matches your Ansible inventory before relying on this table.

---
## Current Status

| Phase | Status | Notes |
|-------|--------|-------|
| OS Preparation | ✅ Tested | Validated on RHEL 8.10 |
| SAP Prerequisites | ✅ Tested | Validated against SAP S/4HANA 1709 requirements |
| HANA Prerequisites | ✅ Tested | sap_hana_preconfigure per SAP Note 2777782 |
| HANA DB Installation | ✅ Tested | Silent HANA DB installation on `saperp` via `hana_install`  |
| Environment Validation | ✅ Tested | Pre-SWPM checklist automated |
| SWPM (S/4HANA) Install | ✅ Tested | Silent SWPM against the HANA DB |
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

SAP server (`saperp`): RHEL 8.x with active subscription · HANA filesystems mounted (`/hana/data`, `/hana/log`, `/hana/shared` on dedicated volumes recommended) · HANA and SWPM media staged at `/hana/software` · all SAP RHEL repositories enabled:

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

## Project Directory Structure

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
│       │   ├── hana.yml                 ← HANA DB configuration (SID, filesystem, directories, media paths)
│       │   └── swpm.yml                 ← SWPM inifile vars (SID, HANA connection, media paths)
│       └── hosts.yml                    ← SAP Ansible Inventory
│
├── playbooks/
│   ├── os_info.yml                      ← Read-only pre-install system report
│   ├── vault_test.yml                   ← Verify vault decryption (no values printed)
│   ├── sap_general_preconfigure.yml     ← Step 1: OS baseline for SAP
│   ├── sap_netweaver_preconfigure.yml   ← Step 2: ABAP-specific tuning
│   ├── sap_hana_preconfigure.yml        ← Step 3: SAP HANA specific OS tuning
│   ├── hana_install.yml                 ← Step 4: SAP HANA DB installation via hdbclm
│   └── swpm_install.yml                 ← Step 5: SAP S/4HANA AS ABAP via SWPM
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


### 1. Install Ansible

```bash
sudo dnf install -y python3.12 python3.12-pip
python3.12 -m pip install ansible
ansible --version | head -2
```

### 2. Configure Hosts

Edit `/etc/hosts` on the controller and SAP server:

```
192.168.1.18  ansnode.tes.com  ansnode
192.168.1.17  saperp.tes.com   saperp
```

### 3. Set Up SSH Key Access

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id root@192.168.1.17
ssh root@192.168.1.17   # verify passwordless access
```
Test ssh access to the sap server, it should not require any password

Assume that the [directory](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/directory_structure.md) structure already made 

### 4.Install Collections

```bash
ansible-galaxy collection install -r requirements.yml -p collections/ --force-with-deps
ansible-galaxy collection list | grep -E "sap_install|linux_system"
```

### 5. Create and Protect the Vault Password File

```bash
echo "YourStrongVaultPassword" > .vault_pass
chmod 600 .vault_pass
```

### 6. Create and Encrypt Vault

```bash
# Edit inventory/group_vars/sap_servers/vault.yml with your passwords first
ansible-vault encrypt inventory/group_vars/sap_servers/vault.yml
```

### 7. Verify Everything

Check the connectivity between Ansible controller & Managed node

```bash
# Test connectivity
cd ~/sap-ansible
ansible -m ping sap_servers
```
From inside ~/sap-ansible, run ping without specifying -i because [ansible.cfg](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/~/sap-ansible/ansible.cfg) now defines the default inventory


### Verify Vault Decryption Works in a Playbook 

```
# Run vault verification
ansible-playbook playbooks/vault_test.yml

```

Create a vault test playbook that confirms decryption works without printing the actual passwords, named as [vault_test.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/playbooks/vault_test.yml):

### Gather SAP System Information

Run [os_info.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/playbooks/os_info.yml) to gather and display SAP-relevant system information from saperp before any installation begins, read-only and makes zero changes to the target system.

```
# Run pre-install system report (read-only)
ansible-playbook playbooks/os_info.yml

```


`ansible.cfg` defines the default inventory and vault password file, so no `-i` or `--vault-password-file` flags are needed when running from the repo root. Expected outputs and screenshots: [docs/EXPECTED_OUTPUT.md](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/docs/EXPECTED_OUTPUT.md).

### Network Configuration (Dual NIC Setup)

This lab uses two network adapters on both the controller and SAP server to separate SAP traffic from internet access (needed for collection installation).


## Playbooks

 Playbook | Purpose |
|---|---|
| `os_info.yml` | Read-only report: hostname/FQDN, OS, CPU/RAM/swap, SAP filesystems, Python, SAP repos, HANA base directory. Makes zero changes. |
| `vault_test.yml` | Confirms every vault variable is defined and non-empty. Prints character counts only, never values. |
| `sap_general_preconfigure.yml` | Step 1 — OS baseline per SAP Notes 2002167, 2772999, 3108302 (kernel params, packages, SELinux, tmpfs, `/etc/hosts`). |
| `sap_netweaver_preconfigure.yml` | Step 2a — process limits and ABAP-specific packages. |
| `hana_preconfigure.yml` | Step 2b — HANA-specific OS tuning on the DB host (SAP Note 2777782 for RHEL 8). |
| `hana_install.yml` | Step 3 — silent SAP HANA installation via `hdblcm` (SID `HDB`, instance 00). |
| `swpm_install.yml` | Step 4 — silent SWPM run installing S/4HANA against the HANA DB (SID `ERP`, SAPHANADB schema). |

Always dry-run a change playbook first:

```bash
ansible-playbook playbooks/sap_general_preconfigure.yml --check
ansible-playbook playbooks/sap_general_preconfigure.yml
```

Or run the whole sequence via tags:

```bash
ansible-playbook site.yml --tags preflight
ansible-playbook site.yml --tags step1,step2
```


Refer to [this](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/tree/main/playbooks) directory for the Ansible playbooks.


## Ansible Vault

All credentials use file-level AES-256 encryption:

```
vault.yml        → encrypted (safe to commit)
.vault_pass      → plain-text password file (chmod 600, NEVER committed)
sap_common.yml   → references secrets via {{ vault_* }} variables
```

| Variable | Purpose | Used by |
|---|---|---|
| `vault_hana_master_password` | HANA SYSTEM / `<sid>adm` master | `sap_hana_install`, `sap_swpm` |
| `vault_sap_master_password` | SAP master (SAP*, DDIC) | `sap_swpm` |
| `vault_sap_db_schema_password` | ABAP schema (SAPHANADB) | `sap_swpm` |
| `vault_sap_diagnostics_agent_password` | Diagnostics agent | `sap_swpm` |

Common operations: `ansible-vault view|edit|rekey inventory/group_vars/sap_servers/vault.yml`

> **Production note:** replace the `.vault_pass` file with HashiCorp Vault, CyberArk, or AAP Credentials Manager. See [VAULT_GUIDE.md](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/VAULT_GUIDE.md) and [SECURITY.md](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/SECURITY.md).


## Installation Sequence

```
Step 1: sap_general_preconfigure                     ← OS tuning for ALL SAP
        - kernel parameters (vm.max_map_count, etc.)
        - package installation
        - SELinux → permissive
        - tmpfs configuration
        - /etc/hosts verification

Step 2: sap_netweaver_preconfigure                   ← Additional ABAP tuning
        - process limits (nproc, nofile)
        - ABAP-specific packages

Step 3: sap_hana_preconfigure                        ← Stage and extract installation media
Step 4: hana_install                                 ← HANA HDB/00 via silent hdblcm 

Step 5: sap_swpm                                     ← S/4HANA AS ABAP installation
        - run SWPM silently against existing HANA DB
        - create SAP system (NW1)
        - create ABAP schema on HANA (SAPHANADB) via hdbuserstore
        - import ABAP content (NW_CreateDBandLoad, 59 packages)
```



## Known Issue

See docs/[troubleshooting.md](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/docs/troubleshooting.md) for the current Python interpreter / libselinux blocker on the SELinux boolean task, including diagnostic commands and candidate fixes.

**Additional issues discovered during the SWPM run against HANA:**

- **HDB client media path not saved automatically.** SWPM has a known glitch where the HANA DB client media location isn't persisted into `inifile.params` after the install dialog. Fix: set it explicitly —
  ```yaml
  sap_swpm_inifile_parameters_dict:
    SAPINST.CD.PACKAGE.RDBMS-HDB-CLIENT: "/sapmedia/sap_hana_client_extracted/SAP_HANA_CLIENT"
  ```
- **`SAPHANADB` / `DBACOCKPIT` username collision** (`CJS-30207`) during `replicateSchemaPasswordsForExistingDatabase`. Fix: explicitly disable DBACOCKPIT user creation if not needed —
  ```yaml
  sap_swpm_inifile_parameters_dict:
    hdb.create.dbacockpit.user: "false"
  ```
- **Instance number collisions.** On a `NW_ABAP_OneHost` topology, ASCS and PAS are still separate instances requiring unique instance numbers on the host — and must also avoid the HANA DB's own instance number. Working config: HANA `00`, ASCS `01`, PAS `02`.
- **No resume/reuseInifile mode in `community.sap_install.sap_swpm`.** Every playbook run regenerates a fresh `inifile.params` and launches a brand-new `sapinst` session rather than resuming a prior one. If a run is interrupted after the kernel/instance-creation phase but before `NW_CreateDBandLoad`, expect `nw.directoryIsNotEmptyUnattended` or instance-already-exists errors on retry — clear `/usr/sap/<SID>/SYS/exe/uc/linuxx86_64/*` before rerunning. The ABAP import itself (`ImportMonitor`) tracks completion per-package independently of the SWPM session, so already-imported packages are correctly skipped on retry even after a full VM crash.
- **`sapinst` runs detached from Ansible** (`async: 86400, poll: 0`). If the Ansible control node (`ansnode`) becomes unreachable mid-install, `sapinst` keeps running unaffected on the SAP host — check `ps -ef | grep '\./sapinst'` there directly rather than assuming the install died. **Never rerun the playbook while a prior `sapinst` process is still alive** — this would launch a second concurrent session against the same HANA schema.
- **Brittle polling condition.** `roles/sap_swpm/tasks/swpm.yml` has an `until` condition (`__sap_swpm_register_pids_sapinst.stdout | length == 0`) that crashes the play if a single poll returns a malformed/empty result (e.g. transient SSH hiccup). Patched locally to:
  ```yaml
  until: "(__sap_swpm_register_pids_sapinst.stdout | default('')) | length == 0"
  ```


## Acknowledgements

- [community.sap_install](https://galaxy.ansible.com/ui/repo/published/community/sap_install/) — SAP automation collection by the Ansible SAP community
- [fedora.linux_system_roles](https://galaxy.ansible.com/ui/repo/published/fedora/linux_system_roles/) — Linux system roles dependency
- SAP Notes: 2002167, 2772999, 3108302 (RHEL 8 for SAP)



