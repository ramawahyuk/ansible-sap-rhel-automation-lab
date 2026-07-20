> Ansible automation for SAP S/4HANA deployment on SAP HANA and Red Hat Enterprise Linux 8 — from OS baseline to HANA installation and silent SWPM execution, built and validated in a two-node hands-on lab.

[![Ansible](https://img.shields.io/badge/Ansible-2.16-red?logo=ansible)](https://www.ansible.com/)
[![RHEL](https://img.shields.io/badge/RHEL-8.10-red?logo=redhat)](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux)
[![SAP](https://img.shields.io/badge/SAP-S%2F4HANA%20%7C%20SAP%20HANA-blue?logo=sap)](https://www.sap.com/)
[![Collection](https://img.shields.io/badge/Collection-community.sap__install%201.9.2-orange)](https://galaxy.ansible.com/ui/repo/published/community/sap_install/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

This repository replaces manual SAP OS preparation with idempotent Ansible playbooks following the `community.sap_install` collection conventions used in enterprise SAP projects. Credentials are protected with AES-256 Ansible Vault encryption. The lab originally targeted ECC6 on Oracle 19c and has since moved to the SAP HANA / S/4HANA stack.

## Table of Contents

- [Lab Architecture](#lab-architecture)
- [Current Status](#current-status)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Playbooks](#playbooks)
- [Ansible Vault](#ansible-vault)
- [Installation Sequence](#installation-sequence)
- [Roadmap](#roadmap)
- [Troubleshooting](#troubleshooting)

## Lab Architecture

![Lab architecture](docs/img/lab-architecture.png)

| Node    | Hostname          | IP           | Role                      |
|---------|-------------------|--------------|---------------------------|
| ansnode | ansnode.tes.com   | 192.168.1.18 | Ansible Controller        |
| saperp  | saperp.tes.com    | 192.168.1.17 | SAP Application + DB Host |

**SAP System:** SAP SID `ERP` · HANA SID `HDB` (instance 00) · SAP S/4HANA · RHEL 8.10

Both nodes use a dual-NIC setup: a host-only adapter (`ens160`, 192.168.1.x) for SAP/Ansible traffic and a NAT adapter (`ens224`) for internet access. Details: [docs/NETWORK_SETUP.md](docs/NETWORK_SETUP.md).

## Current Status

| Phase                   | Status         | Notes                                        |
|-------------------------|----------------|----------------------------------------------|
| OS Preparation          | ✅ Tested      | Validated on RHEL 8.10                       |
| SAP Prerequisites       | ✅ Tested      | General + NetWeaver preconfigure roles       |
| HANA Prerequisites      | 🔄 In Progress | sap_hana_preconfigure per SAP Note 2777782   |
| HANA Installation       | 🔄 In Progress | Silent hdblcm via sap_hana_install           |
| Environment Validation  | 🔄 In Progress | Assert-based preflight_check.yml             |
| SWPM (S/4HANA) Install  | 🔄 In Progress | Silent SWPM against the HANA DB              |
| Post-Install Automation | 📋 Planned     | Future phase                                 |

## Prerequisites

**Controller (`ansnode`):** RHEL 8.x with active subscription · Python 3.12 · Ansible 2.16 · passwordless SSH to the SAP server · internet access for collection installs.

**SAP server (`saperp`):** RHEL 8.x with active subscription · HANA filesystems mounted (`/hana/data`, `/hana/log`, `/hana/shared` on dedicated volumes recommended) · HANA and SWPM media staged at `/hana/software` · all SAP RHEL repositories enabled:

```bash
subscription-manager repos \
  --enable=rhel-8-for-x86_64-baseos-rpms \
  --enable=rhel-8-for-x86_64-appstream-rpms \
  --enable=rhel-8-for-x86_64-sap-solutions-rpms \
  --enable=rhel-8-for-x86_64-sap-netweaver-rpms \
  --enable=rhel-8-for-x86_64-highavailability-rpms
```

## Project Structure

```
.
├── ansible.cfg                          # Engine config (inventory, vault, SSH tuning)
├── requirements.yml                     # Pinned collection versions
├── site.yml                             # Full installation sequence (tagged)
├── .vault_pass                          # Vault password (chmod 600, NEVER committed)
│
├── inventory/
│   ├── hosts.yml                        # Host definitions and groups
│   └── group_vars/
│       ├── all/common.yml               # Timezone, domain (all hosts)
│       ├── sap_servers/sap_common.yml   # SID, IPs, preconfigure vars, vault refs
│       ├── sap_servers/vault.yml        # AES-256 encrypted credentials
│       ├── sap_db/hana.yml              # SAP HANA role variables
│       └── sap_app/                     # (future: dedicated app server)
│
├── playbooks/
│   ├── os_info.yml                      # Read-only pre-install system report
│   ├── vault_test.yml                   # Vault check (never prints secrets)
│   ├── preflight_check.yml              # Assert-based pre-SWPM validation
│   ├── sap_general_preconfigure.yml     # Step 1: OS baseline
│   ├── sap_netweaver_preconfigure.yml   # Step 2a: ABAP tuning
│   ├── hana_preconfigure.yml            # Step 2b: HANA OS tuning (DB host)
│   ├── hana_install.yml                 # Step 3: SAP HANA via hdblcm
│   └── swpm_install.yml                 # Step 4: S/4HANA via SWPM
│
├── roles/                               # Custom roles (collection roles live in collections/)
└── docs/                                # Network setup, vault guide, troubleshooting
```

## Quick Start

```bash
# 1. Clone
git clone https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab.git
cd ansible-sap-rhel-automation-lab

# 2. Install Ansible on the controller
sudo dnf install -y python3.12 python3.12-pip
python3.12 -m pip install ansible

# 3. Name resolution on both nodes (/etc/hosts)
#    192.168.1.18  ansnode.tes.com  ansnode
#    192.168.1.17  saperp.tes.com   saperp

# 4. Passwordless SSH from controller to SAP server
ssh-keygen -t rsa -b 4096
ssh-copy-id root@192.168.1.17

# 5. Install collections (project-local)
ansible-galaxy collection install -r requirements.yml -p collections/ --force-with-deps

# 6. Create the vault password file (avoids shell history)
(umask 077 && openssl rand -base64 24 > .vault_pass)

# 7. Create and encrypt the vault
cp inventory/group_vars/sap_servers/vault.yml.example \
   inventory/group_vars/sap_servers/vault.yml
vi inventory/group_vars/sap_servers/vault.yml   # fill in real passwords
ansible-vault encrypt inventory/group_vars/sap_servers/vault.yml

# 8. Verify
ansible -m ping sap_servers                     # connectivity
ansible-playbook playbooks/vault_test.yml       # vault decryption (no secrets printed)
ansible-playbook playbooks/os_info.yml          # read-only system report
```

`ansible.cfg` defines the default inventory and vault password file, so no `-i` or `--vault-password-file` flags are needed when running from the repo root. Expected outputs and screenshots: [docs/EXPECTED_OUTPUT.md](docs/EXPECTED_OUTPUT.md).

## Playbooks

| Playbook | Purpose |
|---|---|
| `os_info.yml` | Read-only report: hostname/FQDN, OS, CPU/RAM/swap, SAP filesystems, Python, SAP repos, HANA base directory. Makes zero changes. |
| `vault_test.yml` | Confirms every vault variable is defined and non-empty. Prints character counts only, never values. |
| `preflight_check.yml` | Fails fast if the host isn't SAP-ready: RAM, vCPU, filesystem free space, required repos. |
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

> **Production note:** replace the `.vault_pass` file with HashiCorp Vault, CyberArk, or AAP Credentials Manager. See [docs/VAULT_GUIDE.md](docs/VAULT_GUIDE.md) and [SECURITY.md](SECURITY.md).

## Installation Sequence

```
preflight_check              ← fail fast if host not ready
   │
Step 1: sap_general_preconfigure     kernel params, packages, SELinux, /etc/hosts
   │
Step 2a: sap_netweaver_preconfigure  process limits, ABAP packages
Step 2b: sap_hana_preconfigure       HANA OS tuning on the DB host
   │
Step 3: sap_hana_install             silent hdblcm → HANA HDB/00
   │
Step 4: sap_swpm                     SWPM silent run → S/4HANA system ERP
```

## Roadmap

- Complete HANA and SWPM automation phases (in progress)
- Post-install automation (kernel updates, SAP profile handling, HANA backup config)
- Dedicated application server host (`sap_app` group)
- HANA System Replication (HSR) + pacemaker HA as an advanced phase

## Troubleshooting

Current known issue: Python interpreter / libselinux conflict on the SELinux boolean task. Diagnostics and candidate fixes: [docs/troubleshooting.md](docs/troubleshooting.md).

## License

MIT — see [LICENSE](LICENSE).

## Acknowledgements

- [community.sap_install](https://galaxy.ansible.com/ui/repo/published/community/sap_install/) — SAP automation collection
- [fedora.linux_system_roles](https://galaxy.ansible.com/ui/repo/published/fedora/linux_system_roles/)
- SAP Notes 2002167, 2772999, 3108302 (RHEL 8 for SAP)
