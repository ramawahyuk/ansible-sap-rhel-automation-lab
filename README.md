# 🚀 ansible-sap-rhel-automation

> **Ansible automation framework for SAP pre-installation preparation and environment standardization on Red Hat Enterprise Linux (RHEL 8).**  
> This repository documents my hands-on lab work building a repeatable, automated SAP deployment pipeline replacing manual OS preparation steps with idempotent Ansible playbooks validated against a two-node lab environment.
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


---

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
| HANA Client / Media Detection | ✅ Tested | HANA client media location must be pinned explicitly in `inifile.params` — see [Known Issues](#known-issue) |
| HANA DB Installation | ✅ Tested | HANA DB installation on `saperp`  |
| Environment Validation | ✅ Tested | Pre-SWPM checklist automated |
| SWPM Automation | 🔄 In Progress | ABAP import phase (`NW_CreateDBandLoad`) completes in stages; role has no native resume/reuseInifile mode, see Known Issues |
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
- A pre-existing, running SAP HANA instance reachable from `saperp` (this repo does not install HANA itself)
- SAP installation media (SWPM SAR, NetWeaver kernel, HANA client, export/DATA_UNITS) staged under `/sapmedia`
- HANA client media **must be extracted** (not left as a raw `.SAR`) with `LABEL.ASC`/`LABELIDX.ASC` present at the top level — SWPM cannot resolve a `.SAR` archive on its own in unattended mode
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
│       │   └── swpm.yml                 ← SWPM inifile vars (SID, HANA connection, media paths)
│       └── sap_app/                     ← (future: dedicated app server)
│
├── playbooks/
│   ├── os_info.yml                      ← Read-only pre-install system report
│   ├── vault_test.yml                   ← Verify vault decryption (no values printed)
│   ├── sap_general_preconfigure.yml     ← Step 1: OS baseline for SAP
│   ├── sap_netweaver_preconfigure.yml   ← Step 2: ABAP-specific tuning
│   ├── sap_hana_preconfigure.yml        ← Step 3: SAP HANA specific OS tuning
│   └── swpm_install.yml                 ← Step 4: SAP S/4HANA AS ABAP via SWPM
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

<img width="688" height="113" alt="image" src="https://github.com/user-attachments/assets/f2acdec9-9fb4-4dae-b69b-cc1553170c84" />


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

Expected Output:

<img width="461" height="93" alt="image" src="https://github.com/user-attachments/assets/083f3197-e884-48e0-b0d9-bcad122cb1a6" />

---

Verify Vault Decryption Works in a Playbook 

```
# Run vault verification
ansible-playbook playbooks/vault_test.yml

```

Create a vault test playbook that confirms decryption works without printing the actual passwords, named as [vault_test.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/playbooks/vault_test.yml):

Expected Output:

<img width="1450" height="1085" alt="vault" src="https://github.com/user-attachments/assets/8c131b6b-b9a2-4453-81cd-0cb57af9dd1a" />



---

Gather SAP System Information

```
# Run pre-install system report (read-only)
ansible-playbook playbooks/os_info.yml

```
Run [os_info.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/playbooks/os_info.yml) to gather and display SAP-relevant system information from saperp before any installation begins, read-only and makes zero changes to the target system.

Expected Output (partially):
<img width="1026" height="600" alt="image" src="https://github.com/user-attachments/assets/e34bcf61-459d-47ec-bd82-558fd79cf1a1" />


based on that we can see confirmed functional results such as:

```
Hostname    : saperp          ✅
FQDN        : saperp.tes.com  ✅
SAP SID     : NW1             ✅
SAP IP      : 192.168.1.17    ✅   (confirm against active NIC — see network note above)

OS          : RedHat 8.10     ✅
Kernel      : 4.18.0-553      ✅
CPUs        : 6 vCPU          ✅   (reduced from 8 — see Known Issues, host CPU oversubscription)
RAM         : 31.1 GB         ✅

/           : 59.9GB  | 22GB free   | ~63% used  ✅
/sapmnt     : 139.9GB | 138.9GB free | 0.7% used   ✅
/hana/data  : 100GB   | ~80GB free  | ~21% used   ✅
/hana/log   : 50GB    | ~42GB free  | ~18% used   ✅
/hana/shared: 100GB   | ~94GB free  | ~7% used    ✅
/boot       : 50.0GB  | 49.3GB free  | 1.3% used   ✅
/usr/sap    : 149.9GB | 148.7GB free | 0.8% used   ✅
/sapmedia   : (staging area for SWPM SAR, HANA client, DATA_UNITS)
/home       : skipped  ← Correctly excluded by 'when' filter ✅

Python      : 3.12.12         ✅
HANA DB     : S4D, instance 00 ✅

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

Refer to [this](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/tree/main/playbooks) directory for the Ansible playbooks.

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

Step 3: sap_install_media_detect     ← Stage and extract installation media
        - extract SWPM SAR
        - extract HANA client SAR (must produce LABEL.ASC/LABELIDX.ASC)
        - organize export/DATA_UNITS media

Step 4: sap_swpm                     ← S/4HANA AS ABAP installation
        - run SWPM silently against existing HANA DB
        - create SAP system (NW1)
        - create ABAP schema on HANA (SAPHANADB) via hdbuserstore
        - import ABAP content (NW_CreateDBandLoad, 59 packages)
```

---

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
---

## Acknowledgements

- [community.sap_install](https://galaxy.ansible.com/ui/repo/published/community/sap_install/) — SAP automation collection by the Ansible SAP community
- [fedora.linux_system_roles](https://galaxy.ansible.com/ui/repo/published/fedora/linux_system_roles/) — Linux system roles dependency
- SAP Notes: 2002167, 2772999, 3108302 (RHEL 8 for SAP)

---

