# ansible-sap-rhel-automation-lab

Ansible automation framework for SAP pre-installation preparation and environment standardization on Red Hat Enterprise Linux (RHEL 8).

This repository documents my hands-on lab work building a repeatable, automated SAP deployment pipeline — replacing manual OS preparation steps with idempotent Ansible playbooks validated against a two-node lab environment.

---

## Lab Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Home Lab                             │
│                                                             │
│   ┌──────────────────┐         ┌──────────────────┐        │
│   │    ansnode        │         │     saperp        │       │
│   │  192.168.1.18    │──SSH───▶│  192.168.1.17    │        │
│   │                  │         │                   │        │
│   │ Ansible Controller│        │  SAP Target Host  │        │
│   │ RHEL 8.10        │         │  RHEL 8.10        │        │
│   │ ansible-core 2.x │         │  SAP ECC6 (ERP)  │        │
│   │                  │         │  Oracle DB 19c    │        │
│   └──────────────────┘         └──────────────────┘        │
│                                                             │
│   Hypervisor: VMware ESXi (vSphere)                        │
└─────────────────────────────────────────────────────────────┘
```

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

## Repository Structure

```
ansible-sap-rhel-automation-lab/
│
├── README.md                        # This file
├── requirements.yml                 # Ansible Galaxy collection dependencies
├── ansible.cfg                      # Ansible configuration
│
├── inventory/
│   ├── hosts.ini                    # Static inventory — lab nodes
│   ├── group_vars/
│   │   ├── all.yml                  # Global variables (non-sensitive)
│   │   └── sap_hosts.yml            # SAP host group variables
│   └── host_vars/
│       └── saperp.yml               # Host-specific variables
│
├── playbooks/
│   ├── 01_validate_connectivity.yml # Verify SSH and sudo access
│   ├── 02_os_preparation.yml        # RHEL OS tuning for SAP
│   ├── 03_sap_prerequisites.yml     # SAP prerequisite packages + kernel params
│   ├── 04_oracle_prereq.yml         # Oracle DB pre-installation requirements
│   ├── 05_validate_environment.yml  # Final pre-installation validation
│   └── site.yml                     # Master playbook — runs full pipeline
│
├── roles/
│   ├── sap_preparation/             # Core SAP OS preparation role
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   ├── vars/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   └── templates/
│   │       └── sap_limits.conf.j2
│   │
│   ├── oracle_prereq/               # Oracle DB prerequisites role
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   └── vars/
│   │       └── main.yml
│   │
│   └── os_hardening/                # Basic OS configuration role
│       ├── tasks/
│       │   └── main.yml
│       └── defaults/
│           └── main.yml
│
├── vars/
│   └── sap_vars.yml                 # SAP-specific variables (non-sensitive)
│
└── docs/
    ├── architecture.md              # Lab architecture details
    ├── prerequisites.md             # Manual steps before running playbooks
    └── troubleshooting.md           # Common issues and resolutions
```

---

## What This Automation Does

### Phase 1 — Connectivity Validation
Verifies SSH connectivity, sudo privilege escalation, and Python interpreter availability on all target hosts before any changes are made.

### Phase 2 — OS Preparation
Applies RHEL-specific tuning required for SAP workloads:
- Hostname and FQDN configuration
- `/etc/hosts` entry validation
- SELinux configuration for SAP compatibility
- Firewall rule adjustment
- NTP/Chrony time synchronization configuration
- RHEL subscription and repository validation

### Phase 3 — SAP Prerequisites
Installs and configures all SAP-required OS packages and kernel parameters:
- SAP-required package groups (compat-libs, uuidd, etc.)
- Kernel parameter tuning via `sysctl`
- Virtual memory and swap configuration
- `ulimit` settings via `/etc/security/limits.conf`
- `uuidd` daemon enablement

### Phase 4 — Oracle DB Prerequisites
Prepares the OS for Oracle Database 19c installation:
- Oracle-required OS packages
- Kernel parameter additions for Oracle (`shmmax`, `shmall`, `semaphores`)
- Oracle user and group creation (`oracle`, `oinstall`, `dba`)
- Required directory creation with correct ownership
- Oracle environment variable configuration

### Phase 5 — Environment Validation
Runs final checks to confirm the environment meets SAP installation standards before SWPM execution:
- Hostname resolution validation
- Required package presence verification
- Kernel parameter value confirmation
- Disk space and swap availability checks
- Service status verification

---

## Prerequisites

Before running any playbooks, complete the steps in [`docs/prerequisites.md`](docs/prerequisites.md).

**Quick summary:**
- RHEL 8.10 installed on `saperp` with valid Red Hat subscription
- SSH key-based authentication from `ansnode` to `saperp` configured
- `ansible-core` installed on `ansnode`
- `community.sap_install` collection installed (see below)
- Ansible Vault password file configured

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/ramawahyukurniawan/ansible-sap-rhel-automation-lab.git
cd ansible-sap-rhel-automation-lab
```

### 2. Install required collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 3. Configure Ansible Vault

Create your vault password file (do not commit this):

```bash
echo "your_vault_password" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

Encrypt sensitive variables:

```bash
ansible-vault encrypt_string 'your_password' --name 'ansible_become_password'
```

### 4. Review and update inventory

Edit `inventory/hosts.ini` and `inventory/group_vars/all.yml` to match your environment.

---

## Running the Playbooks

### Full pipeline (recommended)

```bash
ansible-playbook playbooks/site.yml \
  -i inventory/hosts.ini \
  --vault-password-file ~/.vault_pass
```

### Individual phases

```bash
# Phase 1 — Connectivity check only
ansible-playbook playbooks/01_validate_connectivity.yml \
  -i inventory/hosts.ini

# Phase 2 — OS preparation
ansible-playbook playbooks/02_os_preparation.yml \
  -i inventory/hosts.ini \
  --vault-password-file ~/.vault_pass

# Phase 3 — SAP prerequisites
ansible-playbook playbooks/03_sap_prerequisites.yml \
  -i inventory/hosts.ini \
  --vault-password-file ~/.vault_pass

# Phase 5 — Validation only (dry run check)
ansible-playbook playbooks/05_validate_environment.yml \
  -i inventory/hosts.ini \
  --check
```

### Check mode (dry run — no changes applied)

```bash
ansible-playbook playbooks/site.yml \
  -i inventory/hosts.ini \
  --vault-password-file ~/.vault_pass \
  --check --diff
```

---

## Collections Used

| Collection | Version | Purpose |
|------------|---------|---------|
| `community.sap_install` | 1.9.2 | SAP installation roles and OS preparation |
| `fedora.linux_system_roles` | latest | RHEL system configuration roles |

See [`requirements.yml`](requirements.yml) for full dependency list.

---

## Security Notes

- Sensitive credentials (passwords, vault keys) are **never stored in this repository**
- All secrets are managed via **Ansible Vault**
- The `.gitignore` excludes vault password files, SSH keys, and any `*.secret` files
- Inventory variables containing credentials use Vault-encrypted strings

---

## Current Status

| Phase | Status | Notes |
|-------|--------|-------|
| OS Preparation | ✅ Tested | Validated on RHEL 8.10 |
| SAP Prerequisites | ✅ Tested | Validated against SAP ECC6 requirements |
| Oracle Prerequisites | ✅ Tested | Validated against Oracle 19c requirements |
| Environment Validation | ✅ Tested | Pre-SWPM checklist automated |
| SWPM Automation | 🔄 In Progress | Manual SWPM execution follows automated prep |
| Post-Install Automation | 📋 Planned | Future phase |

---

## Related Projects

This automation supports a broader SAP home lab environment:
- **SAP System:** ECC6 EHP8 (SID: ERP) on Oracle 19c / RHEL 8.10
- **SAP S/4HANA 1709** lab running in parallel for HANA administration practice
- **VMware Infrastructure:** ESXi / vCenter / vSAN / SRM — separate DR lab

---

## Author

**Rama Wahyu Kurniawan**
- MSc Computer Science — Universiti Putra Malaysia (GPA 3.77)
- System Engineer | VMware Infrastructure | SAP Basis (Learning) | Ansible Automation
- LinkedIn: [linkedin.com/in/ramawahyukurniawan](https://linkedin.com/in/ramawahyukurniawan)

---

## License

MIT License — feel free to use and adapt for your own SAP lab environments.
