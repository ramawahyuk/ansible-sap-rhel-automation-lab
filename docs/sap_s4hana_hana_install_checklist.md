# SAP S/4HANA 1709 on SAP HANA 2.0 SPS03 — Install Checklist (RHEL 8.10)

Legend: 🔲 pending · ⚙️ automated by Ansible roles

---

## 1. Pre-Installation — Host & OS (RHEL 8.10)

- 🔲 Confirm allocated RAM meets HANA minimum. `hdblcm` enforces a hard check
  (lab installs have failed at <1 GB; production minimum guidance is 24 GB+ per SAP Note 1637145 for a sandbox we may need `--ignore=check_min_mem`, but plan real RAM if possible)
- ⚙️ `sap_preconfiguration` role already applied (kernel params, users, SELinux, swap)
- ⚙️ `sap_netweaver_preconfiguration` role already applied
- 🔲 **NEW:** Run `community.sap_install.sap_hana_preconfigure` (from `requirements.yml` we added) covers HANA-specific OS tuning (SAP Note 2235581, 2009879, 2578899) not covered by previous NetWeaver-only preconfig role

---

## 2. Filesystem Layout for HANA

mounts filesystem:

- 🔲 `/hana/data/<SID>` -> sized per `sap_hana_data_size_gb`
- 🔲 `/hana/log/<SID>` -> sized per `sap_hana_log_size_gb`
- 🔲 `/hana/shared/<SID>`
- 🔲 `/usr/sap/<SID>`

---

## 3. Media Placement & Permissions

- 🔲 Copy/mount HANA DB file to target host (e.g. `/software/...`)
- 🔲 Verify SAPCAR builds are generic Linux x86_64
- 🔲 Set executable permissions inside `HDB_SERVER_LINUX_X86_64`:
  ```bash
  #example
  cd .../HANA.2.0/DATA_UNITS/HDB_SERVER_LINUX_X86_64
  #set permission
  chmod 755 hdblcm hdbinst instruntime/sdbrun
  ```
---

## 4. HANA Database Installation

- 🔲 Decide SID, instance number, sizing → fill in [group_vars/hana.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/inventory/group_vars/sap_db/hana.yml)
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

<img width="714" height="208" alt="image" src="https://github.com/user-attachments/assets/3cd9c3b1-322a-4ffd-ac04-21008f1f1cae" />


---

## 5. HANA Client (on app server host, if separate from DB host)

- 🔲 Install `HANA_CLIENT` (match DB patch level exactly)
  ```bash
  cd .../HDB_CLIENT_LINUX_X86_64
  ./hdbinst
  ```

---

## 6. Optional HANA Add-ons

- 🔲 Ensure `HANA_LCAPPS` ara avaialable only if we need HANA Live / LiveCache reporting content
- 🔲 Skip `XSA_*`, `XSAC_*`, `HCO_*`, EPM/RSA/Spark folders inside `DATA_UNITS` unless your
  scope specifically needs XS Advanced apps or SDI connectors

---

## 7. S/4HANA Application Install via SWPM

- 🔲 Extract kernel: `KERNEL_FILE`
- 🔲 Extract SWPM: `SAPCAR -xvf SWPMFILE_*.SAR`
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
1. **RAM sizing** — confirm before running `hdblcm`; this is the most common early-failure point.
2. **Ansible role variable names** — `community.sap_install.sap_hana_install` variable names can
   shift between collection versions; run `ansible-doc -t role community.sap_install.sap_hana_install`
   after installing the collection and reconcile against `roles/sap_hana_install/tasks/main.yml`.
