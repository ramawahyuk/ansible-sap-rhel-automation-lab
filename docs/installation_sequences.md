# Installation sequence

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


## 1. `sap_general_preconfigure` (in progress)

```bash
# Do a dry run — check mode shows what WOULD change without changing anything
ansible-playbook playbooks/sap_general_preconfigure.yml --check

ansible-playbook playbooks/sap_general_preconfigure.yml
```

OS-level baseline tuning for any SAP installation: kernel parameters
(SAP Note 2002167), SELinux → permissive, package groups, `/etc/hosts`
entries, tmpfs sizing (SAP Note 941735), locale, process resource limits.



<img width="730" height="855" alt="image" src="https://github.com/user-attachments/assets/8ef81b6d-2d57-4534-bcb9-f6b3907aae59" />
<img width="730" height="861" alt="image" src="https://github.com/user-attachments/assets/82e19e0d-a206-4060-bc17-5bc526e11ba9" />

>
>**Status:** Step 1 of the installation sequence is complete ✅.
>



---

## 2. `sap_netweaver_preconfigure` (not yet implemented)

Additional ABAP/NetWeaver-specific OS tuning beyond the general
preconfigure role: extra packages, process limits specific to NetWeaver
work processes.

## 3. Stage Oracle 19c media

Oracle installation media must be copied to `saperp:/oracle/software`
(matches `sap_anydb_install_oracle_extract_path` in
`inventory/group_vars/sap_db/oracle.yml`). This repo does not include the
media itself — it's licensed software and must be sourced from SAP/Oracle
directly.

**Note:** `ansnode`'s root disk is only 47 GB — never stage SAP media on
the controller. It must land directly on `saperp`.

## 4. `sap_anydb_install_oracle` (not yet implemented)

Installs Oracle DB 19c using the `community.sap_install.sap_anydb_install_oracle`
role, method `minimal`. Two mandatory variables with no role defaults are
already set in `inventory/group_vars/sap_db/oracle.yml`:

- `sap_anydb_install_oracle_system_password`
- `sap_anydb_install_oracle_extract_path`

## 5. `sap_swpm` (not yet implemented)

Installs the SAP ABAP stack via SWPM, using the Oracle instance from step
4 as the database backend.

---

Each future step should follow the same pattern as
`playbooks/sap_general_preconfigure.yml`: a `pre_tasks` hostname/OS
assertion, the role itself, and `post_tasks` that verify the expected
end-state before moving to the next playbook.
