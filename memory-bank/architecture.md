# Architecture Overview

## Repository Skeleton (M1 Step 1)

- `.github/workflows/`: reserved for GitHub Actions workflows (lint + deploy). Pre-creating the folder lets us drop workflow YAML later without touching other docs.
- `inventory/`: root for inventory data. `group_vars/vps/` prepares shared variables (including the future `main.yml`/`secrets.yml` split) while `host_vars/` will hold per-host overrides that mirror the implementation plan.
- `playbooks/`: home for orchestration entry points such as `playbooks/site.yml`, keeping the repo root uncluttered as scenarios multiply.
- `roles/`: target directory for Galaxy-installed roles and local roles; keeping it empty but present allows idempotent `ansible-galaxy install` runs once dependencies are declared.
- `ansible.cfg`: currently contains `[defaults] roles_path = roles`, ensuring both local runs and CI resolve roles without extra CLI flags; additional defaults can build on this file later.

This minimal scaffold enforces the agreed Ansible-first layout and is the foundation for the remaining M1 tasks (requirements files, inventory population, and playbook scaffolding).
