# SAP S/4HANA 1709 on SAP HANA 2.0 SPS03 — Install Checklist (RHEL 8.10)

Based on your confirmed media set (screenshots) and your existing `ansible-sap-rhel-automation-lab`
repo. This replaces the Oracle 19c path with SAP HANA.

Legend: 🔲 pending · ⚙️ automated by your Ansible roles

---

## 0. Media Inventory (confirmed)

| Material # | Folder | Role | Confirmed |
|---|---|---|---|
| 51053061 | `HANA.2.0_SPS03-REV30` | **HANA DB installer** | ✅ verified path matches SAP docs |
| 51052190 | `S4HANA_SPS01` / `SAP_EXPORT` | S/4HANA ABAP install export | ✅ |
| 51052189 | `SAP_LANG` | Language packs (optional) | ✅ |
| 51052271 | `SAP_SCM` | SCM add-on (optional) | ✅ |
| 51052377 | `SAP_CLIENT` | SAP GUI/frontend | ✅ |
| 51052318 | `SAP_COMP` | S4HOP_NW75_SP08_JAVA — Java stack export (EHS Expert/WWI, ESR content). **Not needed for ABAP-only install** | ✅ confirmed, skip |
| — | `HANA_CLIENT.2.0-SPS03` | HANA client libs | ✅ |
| — | `HANA_LCAPPS.2.0-SPS03-REV30` | HANA Live content (optional) | ✅ |
| — | `KERNEL64_EXTRACT` | SAP kernel | ✅ generic, reusable |
| — | `SWPM10SP23` | Software Provisioning Manager | ✅ generic, reusable |

🔲 Open `MID.XML` inside `51052318` to confirm what it is before relying on it.

---

## 1. Pre-Installation — Host & OS (RHEL 8.10)

- 🔲 Confirm `saperp` host is RHEL 8.10 (not SLES) — matches your existing preconfig roles
- 🔲 Confirm allocated RAM meets HANA minimum. `hdblcm` enforces a hard check
  (lab installs have failed at <1 GB; production minimum guidance is 24 GB+ per SAP Note 1637145 —
  for a sandbox you may need `--ignore=check_min_mem`, but plan real RAM if possible)
- ⚙️ `sap_preconfiguration` role already applied (kernel params, users, SELinux, swap)
- ⚙️ `sap_netweaver_preconfiguration` role already applied
- 🔲 **NEW:** Run `community.sap_install.sap_hana_preconfigure` (from `requirements.yml` we added)
  — covers HANA-specific OS tuning (SAP Note 2235581, 2009879, 2578899) not covered by your
  NetWeaver-only preconfig role

---

## 2. Filesystem Layout for HANA

Replace your Oracle `/oracle/<SID>` mounts with:

- 🔲 `/hana/data/<SID>` — sized per `sap_hana_data_size_gb`
- 🔲 `/hana/log/<SID>` — sized per `sap_hana_log_size_gb`
- 🔲 `/hana/shared/<SID>`
- 🔲 `/usr/sap/<SID>`
- 🔲 Verify disk headroom: install kit (~2.7 GB) is separate from installed footprint,
  which typically needs 15–30+ GB even for a minimal sandbox instance

---

## 3. Media Placement & Permissions

- 🔲 Copy/mount `HANA.2.0_SPS03-REV30` to target host (e.g. `/software/...`)
- 🔲 Verify SAPCAR builds are generic Linux x86_64, not SLES-specific
- 🔲 Set executable permissions inside `HDB_SERVER_LINUX_X86_64`:
  ```bash
  cd .../HANA.2.0_SPS03-REV30/DATA_UNITS/HDB_SERVER_LINUX_X86_64
  chmod 755 hdblcm hdbinst instruntime/sdbrun
  ```
- 🔲 Copy `S4HANA_SPS01`/`SAP_EXPORT` (51052190) to a separate accessible path for SWPM later
- 🔲 Copy `SWPM10SP23` and `KERNEL64_EXTRACT` to target host

---

## 4. HANA Database Installation

- 🔲 Decide SID, instance number, sizing → fill in `group_vars/hana.yml`
- 🔲 Add `vault_sap_hana_sys_password` to `group_vars/vault.yml`
- ⚙️ Run `playbooks/hana_install.yml` (wraps `hdblcm` unattended install via
  `community.sap_install.sap_hana_install`)
  - Manual equivalent if troubleshooting interactively:
    ```bash
    cd .../HDB_SERVER_LINUX_X86_64
    ./hdblcm
    ```
- 🔲 Post-install verification:
  ```bash
  su - <sid>adm
  HDB info        # should show all processes running
  ```
- 🔲 Confirm `sapcontrol -nr <instance_nr> -function GetProcessList` reports `GREEN`

---

## 5. HANA Client (on app server host, if separate from DB host)

- 🔲 Install `HANA_CLIENT.2.0-SPS03` (match DB patch level exactly)
  ```bash
  cd .../HDB_CLIENT_LINUX_X86_64
  ./hdbinst
  ```

---

## 6. Optional HANA Add-ons

- 🔲 `HANA_LCAPPS.2.0-SPS03-REV30` — only if you need HANA Live / LiveCache reporting content
- 🔲 Skip `XSA_*`, `XSAC_*`, `HCO_*`, EPM/RSA/Spark folders inside `DATA_UNITS` unless your
  scope specifically needs XS Advanced apps or SDI connectors

---

## 7. S/4HANA Application Install via SWPM

- 🔲 Extract kernel: `KERNEL64_EXTRACT`
- 🔲 Extract SWPM: `SAPCAR -xvf SWPM10SP23_*.SAR`
- 🔲 Update your S4HANA install playbook/SWPM vars to point DB connection at the new HANA
  instance instead of Oracle:
  - DB host, instance number, `SYSTEM` user credentials
  - Media Browser path → your `SAP_EXPORT`/51052190 folder
- 🔲 Run SWPM (GUI, or `sapinst` with a stack XML from Maintenance Planner if you have one)
- 🔲 At **SAP HANA Multitenant Database Containers** screen: confirm instance number + password
- 🔲 At **Database Schema** screen: set DBACOCKPIT password
- 🔲 At **Declustering/Depooling Option**: recommended to enable for all ABAP tables (HANA-native)
- 🔲 At **SAP System Database Import**: default parallel jobs is commonly set to match CPU
  count (many guides use 48 — adjust to your lab's actual core count)
- 🔲 Complete PAS/ASCS instance configuration screens
- 🔲 Let import run to completion

---

## 8. Optional: Apply Support Package Stack

- 🔲 `S4HANA_SPS02` — only if you want to move beyond the SPS01 base import
- 🔲 `SAP_COMP`/`COMP` — additional component/support packages if in scope

---

## 9. Post-Install Sanity Checks

- 🔲 Log on via SAP GUI / SAP HANA Studio using `SYSTEM` user
- 🔲 Confirm ABAP system starts cleanly (`sapcontrol -nr <PAS_nr> -function GetProcessList`)
- 🔲 Check installation log directories for warnings/errors before declaring done
- 🔲 Document actual SID, instance numbers, hostnames back into your GitHub repo for
  reproducibility

---

## Notes / Open Items to Resolve

1. **51052318** — confirm exact content before depending on it in the install path.
2. **RAM sizing** — confirm before running `hdblcm`; this is the most common early-failure point.
3. **Ansible role variable names** — `community.sap_install.sap_hana_install` variable names can
   shift between collection versions; run `ansible-doc -t role community.sap_install.sap_hana_install`
   after installing the collection and reconcile against `roles/sap_hana_install/tasks/main.yml`.
