# Contributing to ansible-sap-rhel-automation

Thank you for considering a contribution to this project. This guide explains conventions, branch strategy, and what we look for in pull requests.

---

## Ground Rules

- **Never commit `.vault_pass`** or any file containing plain-text credentials.
- All passwords must live in `vault.yml` (encrypted) and be referenced via `{{ vault_* }}` variables.
- Playbooks must be idempotent — running them twice must not create errors or unintended changes.
- Test with `--check` (dry run) before submitting any playbook change.
- Variable names that are role-specific must match the upstream collection convention exactly (e.g., `sap_anydb_install_oracle_sid`).

---

## Branch Naming

| Type | Pattern | Example |
|------|---------|---------|
| Feature | `feature/<short-description>` | `feature/add-hana-playbook` |
| Bug fix | `fix/<short-description>` | `fix/vault-path-resolution` |
| Documentation | `docs/<short-description>` | `docs/update-troubleshooting` |
| Refactor | `refactor/<short-description>` | `refactor/group-vars-cleanup` |

---

## Pull Request Checklist

Before opening a PR, verify:

- [ ] `ansible-playbook <playbook>.yml --check` passes without errors
- [ ] `ansible-lint <playbook>.yml` passes (or violations are justified in the PR)
- [ ] No plain-text secrets in any committed file
- [ ] New variables added to the correct `group_vars` file with inline comments
- [ ] `README.md` updated if new playbooks or variables are introduced
- [ ] Commit messages are descriptive (`Add:`, `Fix:`, `Update:`, `Remove:`)

---

## Commit Message Convention

```
<type>: <short description>

[optional body explaining WHY, not WHAT]
```

Types: `Add`, `Fix`, `Update`, `Remove`, `Refactor`, `Docs`, `Security`

Examples:
```
Add: oracle.yml role variables with sap_anydb_install_oracle_ prefix
Fix: vault_password_file absolute path in ansible.cfg defaults section
Docs: add troubleshooting table for vault not found errors
```

---

## Reporting Issues

Use the GitHub Issues tab. Select the appropriate template:

- **Bug Report** — something is broken or behaving unexpectedly
- **Feature Request** — propose a new playbook, role, or variable pattern
- **Question** — clarification on configuration or expected output

Please include:
- OS version (`cat /etc/redhat-release`)
- Ansible version (`ansible --version`)
- Collection version (`ansible-galaxy collection list | grep sap_install`)
- The full error output (redact any passwords)
