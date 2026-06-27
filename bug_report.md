---
name: Bug Report
about: Something is broken or behaving unexpectedly
title: "[BUG] "
labels: bug
assignees: ''
---

## Describe the Bug
A clear description of what is broken and what you expected to happen.

## Environment

| Item | Value |
|------|-------|
| RHEL version | `cat /etc/redhat-release` |
| Ansible version | `ansible --version` |
| community.sap_install version | `ansible-galaxy collection list \| grep sap_install` |
| Playbook run | e.g. `sap_general_preconfigure.yml` |

## Steps to Reproduce
1. Run `ansible-playbook playbooks/...`
2. Observe error at task `...`

## Error Output
```
Paste the full error output here.
Redact any passwords or sensitive values before pasting.
```

## Additional Context
Any other context — network topology, custom variables, workarounds tried.
