# Troubleshooting

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


## RESOLVED — `sap_general_preconfigure` failed due to mismatched python interpreters

```

TASK [community.sap_install.sap_general_preconfigure : Gather package facts] ************
Tuesday 30 June 2026  11:35:15 +0700 (0:00:00.095)       0:00:05.116 **********
An exception occurred during task execution. To see the full traceback, use -vvv. The error was: SyntaxError: future feature annotations is not defined

```
This is the same root issue resurfacing somewhere a Python <3.7 interpreter (most likely RHEL 8's /usr/libexec/platform-python, which is 3.6) 
is being used to execute the module, even though ad-hoc python3 --version showed 3.12.13. That mismatch is the key clue: ad-hoc command execution and module execution don't necessarily use the same interpreter unless ansible_python_interpreter is actually being honored

First confirm what interpreter Ansible is being used and match it with [hosts.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/inventory/hosts.yml):

```
cd ~/sap-ansible
ansible saperp -m setup -a "filter=ansible_python*"

```
```
# Existing host.yml before issue resolved

 vars:
    # Ansible connection defaults
    ansible_user: root
    ansible_python_interpreter: /usr/bin/python3.12
  children:

```

What's actually happening:

`ansible.builtin.package_facts` needs the rpm Python C-extension bindings to query installed packages. Those bindings are only built against `/usr/libexec/platform-python3.6` (RHEL 8's system Python), there's no rpm module compiled for `/usr/bin/python3.12`.
Which means it is interpreter incompatibility. To confirm the binding gap run this following command.

```
ansible saperp -m shell -a "rpm -qa | grep -iE 'rpm-python|python3.*-rpm|python3.*-libdnf'" \
  -e "ansible_python_interpreter=/usr/bin/python3.12"

ansible saperp -m shell -a "python3.12 -c 'import rpm' 2>&1" \
  -e "ansible_python_interpreter=/usr/bin/python3.12"

ansible saperp -m shell -a "dnf list available | grep -iE '^python3\.12.*(rpm|dnf)'" \
  -e "ansible_python_interpreter=/usr/bin/python3.12"

```
Based on that confirmation, the real fix: is to downgrade ansible-core on the controller to a version whose managed-node floor still includes Python 3.6.

```
# Downgrade to a version that still supports 3.6 managed nodes
# (ansible-core 2.15.x is the last branch with broad 3.6 support)
pip3.12 install --user 'ansible-core==2.15.13'

# Reinstall the full ansible meta-package pinned to a matching version
pip3.12 install --user 'ansible==9.13.0'

ansible --version

```

After we downgrade, re-run the same playbook `package_facts`

```
ansible-playbook playbooks/sap_general_preconfigure.yml --check

```

Its still got error 
`TASK [fedora.linux_system_roles.selinux : Set SELinux booleans] *************************
Tuesday 30 June 2026  12:19:03 +0700 (0:00:00.099)       0:01:36.436 **********
An exception occurred during task execution. To see the full traceback, use -vvv. The error was: ModuleNotFoundError: No module named 'semanage'
failed: [saperp] (item={'name': 'selinuxuser_execmod', 'state': 'on', 'persistent'`

Since ansible-core is now downgraded to 2.16.19, it's fully compatible with Python 3.6 as a managed-node interpreter — so the cleanest fix is to stop using 3.12 on saperp entirely and point it at the system Python that already has every binding these system-tuning roles need.

```
#verify the system Python has 
ansible saperp -m shell -a "rpm -qa | grep -iE 'libselinux-python|libsemanage-python|python3.*-policycoreutils'" \
  -e "ansible_python_interpreter=/usr/bin/python3.12"

ansible saperp -m shell -a "/usr/libexec/platform-python3.6 -c 'import semanage, selinux, rpm; print(\"OK\")'" \
  -e "ansible_python_interpreter=/usr/bin/python3.12"

```

If the results come out `OK` then we switch the interpreter on [hosts.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/inventory/hosts.yml)


<img width="525" height="147" alt="image" src="https://github.com/user-attachments/assets/1d0d4961-5f41-4117-8ede-b3092eb096ea" />

<img width="491" height="96" alt="image" src="https://github.com/user-attachments/assets/5188bea0-d374-4188-ab75-8432027d55d1" />

Then Re-ren the dry run again, if its work with `failed=0` means the we can clear for the real run.

```
# Dry-run
ansible-playbook playbooks/sap_general_preconfigure.yml --check

# must be confirm that it has zero failed score 
ansible-playbook playbooks/sap_general_preconfigure.yml

```
---

## RESOLVED — UTF-8 corruption in `ansible.cfg`

Heredoc-generated `ansible.cfg` picked up box-drawing characters from a
copy-paste source, which Ansible's config parser choked on. Fixed by
rewriting the file as pure ASCII.

## RESOLVED — `vault_password_file` silently ignored

`vault_password_file` was placed outside the `[defaults]` section in
`ansible.cfg`. Ansible does not error on this — it just silently ignores
the setting. Moved it inside `[defaults]`, with an absolute path (`~`
is not expanded by Ansible's config parser).

## RESOLVED — RPM/pip `ansible-core` dual-install conflict

The system RPM `ansible-core` (2.16.3) and a pip-installed `ansible-core`
(2.21.1) were both present, with the RPM version pulling in its own
Python dependency tree. Removed the RPM package and reinstalled the full
`ansible` pip package (which also restored the `timer`/`profile_tasks`
callback plugins that the RPM-only install didn't include).

