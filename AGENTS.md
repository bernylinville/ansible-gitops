# Repository Guidelines

## Project Structure & Module Organization

- Ansible-first layout:
  - `playbooks/` (entry points, e.g., `site.yml`)
  - `roles/<role>/{tasks,handlers,templates,files,defaults,vars}`
  - `inventory/{dev,stage,prod}/` with `group_vars/` and `host_vars/`
  - `collections/requirements.yml`, `ansible.cfg`, `docs/`
- Keep env‑specific vars in `inventory/<env>/group_vars/` and defaults in `roles/<role>/defaults/main.yml`.

## Build, Test, and Development Commands

- Bootstrap (local venv):
  ```bash
  python3 -m venv .venv && source .venv/bin/activate
  pip install ansible ansible-lint yamllint molecule[docker] pre-commit
  ansible-galaxy collection install -r collections/requirements.yml
  ```
- Lint fast: `ansible-lint` and `yamllint .`
- Dry run (dev): `ansible-playbook -i inventory/dev site.yml --check --diff`
- Role tests: `cd roles/<role> && molecule test`

## Coding Style & Naming Conventions

- YAML: 2‑space indent; UTF‑8; wrap at ~120 cols; no tabs.
- Names: variables `snake_case`; files/dirs `kebab-case`; role names `namespace.role` if published.
- Tasks must be idempotent and named; prefer modules over `command/shell`.
- Tag important paths (e.g., `tags: [deploy, rollback]`). Secrets go in Ansible Vault only.

## Testing Guidelines

- Use Molecule per role (`roles/<role>/molecule/<scenario>`). Include converge + idempotence.
- For inventory/playbook checks, run `--check --diff` and capture output in PRs.
- Optional verify with Testinfra/PyTest in Molecule `verify.yml` or `tests/`.

## Commit & Pull Request Guidelines

- Conventional Commits: `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`. Subject ≤72 chars; body explains why + context.
- PRs: clear description, linked issue, what changed/why, lint/results snippet, and any screenshots/logs.

## Security & Configuration Tips

- Never commit secrets or vault keys. Use `ansible-vault encrypt` and keep passwords outside the repo.
- Pin collection/role versions in `collections/requirements.yml`; run `ansible-config dump` to debug config.

## IMPORTANT:

- Always read memory-bank/architecture.md before writing any code. Include entire database schema.
- Always read memory-bank/product-requirements-document.md before writing any code.
- After adding a major feature or completing a milestone, update memory-bank/architecture.md.
