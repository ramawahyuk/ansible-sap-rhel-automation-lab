# Installation sequence

```
Step 1: sap_general_preconfigure      ← OS tuning for ALL SAP
        - kernel parameters (vm.max_map_count, etc.)
        - package installation
        - SELinux → permissive
        - tmpfs configuration
        - /etc/hosts verification

Step 2: sap_netweaver_preconfigure     ← Additional ABAP tuning
        - process limits (nproc, nofile)
        - ABAP-specific packages

Step 3: hana_install                   ← SAP HANA DB Installation
        - extract HANA media
        - run HANA installer (silent)
     
Step 4: swpm_install                   ← ABAP stack installation
        - run SWPM silently
        - create SAP system (ERP/SID)
        - configure Oracle schema (SAPSR3)
```


## 1. `sap_general_preconfigure` ✅

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

## 2. `sap_netweaver_preconfigure`✅

Additional ABAP/NetWeaver-specific OS tuning beyond the general
preconfigure role: extra packages, process limits specific to NetWeaver
work processes.

```
# Run the Dry run and ensure its works succesfully
ansible-playbook playbooks/sap_netweaver_preconfigure.yml --check

# Execute the real one after Dry-run succed
ansible-playbook playbooks/sap_netweaver_preconfigure.yml 

```

<img width="725" height="871" alt="image" src="https://github.com/user-attachments/assets/06ceb2d6-e3d2-4b99-aa20-c41bc885ffd2" />

<img width="724" height="856" alt="image" src="https://github.com/user-attachments/assets/d3474515-8140-435a-8af1-53681fe38b93" />

Key things from the output above:

1. SAP NetWeaver required packages were installed successfully.
2. SAP Note 2526952 (RHEL for SAP packages) was applied.
3. SAP Note 3119751 (SAP Kernel requirements) was applied.
4. The tuned profile was switched from `virtual-guest` to `sap-netweaver`.

<img width="673" height="33" alt="image" src="https://github.com/user-attachments/assets/e5c40b15-9f80-4dfb-ab87-16842abe2667" />

5. Compatibility libraries for newer SAP kernels were configured (`compat-sap-c++-11.so`).

<img width="635" height="58" alt="image" src="https://github.com/user-attachments/assets/c3bc71c9-15fb-452c-8e28-d97484523557" />


6. /usr/sap/lib and the required `libstdc++.so.6` symlink were created.

<img width="723" height="62" alt="image" src="https://github.com/user-attachments/assets/83039165-63ea-4c31-884a-574d4d66766a" />


7. Final result: `failed=0`, `unreachable=0`.


## 3. Stage SAP HANA DB Installation

A. Pre-installation

- ✅ Confirm allocated RAM meets HANA minimum. `hdblcm` enforces a hard check
  (lab installs have failed at <1 GB; production minimum guidance is 24 GB+ per SAP Note 1637145 —
  for a sandbox you may need `--ignore=check_min_mem`, but plan real RAM if possible)
- ⚙️ `sap_preconfiguration` role already applied (kernel params, users, SELinux, swap)
- ⚙️ `sap_netweaver_preconfiguration` role already applied
- ✅ **NEW:** Run `community.sap_install.sap_hana_preconfigure` (from `requirements.yml` we added)
  covers HANA-specific OS tuning (SAP Note 2235581, 2009879, 2578899) not covered by your
  NetWeaver-only preconfig role

```
# Run 
ansible-doc -t role community.sap_install.sap_hana_install

# Execute the real one after Dry-run succed
ansible-playbook playbooks/sap_netweaver_preconfigure.yml 

```






## 5. `sap_swpm` (not yet implemented)

Installs the SAP ABAP stack via SWPM, using the Oracle instance from step
4 as the database backend.

---

Each future step should follow the same pattern as
`playbooks/sap_general_preconfigure.yml`: a `pre_tasks` hostname/OS
assertion, the role itself, and `post_tasks` that verify the expected
end-state before moving to the next playbook.
