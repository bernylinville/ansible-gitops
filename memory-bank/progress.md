## 2025-11-25 – M1 Step 1

- Created the baseline directories required for the Ansible-first layout: `.github/workflows/`, `inventory/group_vars/vps/`, `inventory/host_vars/`, `playbooks/`, and `roles/`.
- Added `ansible.cfg` with `[defaults] roles_path = roles` so Galaxy roles resolve correctly in later steps.
- Deferred verification of `ansible-config dump | rg roles_path` to the user environment because `ansible-config` is unavailable locally.

Next dev can proceed with M1 Step 2 once any additional validation is complete.

## 2025-11-25 – M1 Step 1 Verification

- Created a repo-local `.ansible/tmp/` so `ansible-config` can run inside the sandbox without touching `$HOME`.
- Ran `ANSIBLE_LOCAL_TEMP=$PWD/.ansible/tmp ~/.local/share/mise/shims/ansible-config dump | rg -i roles_path` to capture `DEFAULT_ROLES_PATH(/Users/kchou/Code/ansible-gitops/ansible.cfg) = ['/Users/kchou/Code/ansible-gitops/roles']`, confirming the config behaves as intended.
- Ready to begin M1 Step 2 after the user confirms this validation.

## 2025-11-25 – M1 Step 2 (roles requirements)

- Added root-level `requirements.yml` pinning `geerlingguy.docker` 7.8.0, `geerlingguy.security` 3.0.0, and `geerlingguy.firewall` 2.7.0 as required by the implementation plan.
- Verified the file by running `ANSIBLE_LOCAL_TEMP=$PWD/.ansible/tmp ~/.local/share/mise/shims/ansible-galaxy install -r requirements.yml -p roles`, which successfully downloaded all three roles.
- Removed the generated `roles/geerlingguy.*` directories afterward and introduced `.gitignore` entries (plus `.ansible/`) so the repo stays clean while still allowing future local roles to be tracked.

## 2025-11-25 – M1 Step 3 (collections requirements)

- Created `collections/requirements.yml` pointing to `community.docker` 5.0.2 per the implementation plan.
- Validated the manifest with `ANSIBLE_LOCAL_TEMP=$PWD/.ansible/tmp ~/.local/share/mise/shims/ansible-galaxy collection install -r collections/requirements.yml`; Galaxy pulled the tarball and installed it under `/Users/kchou/.ansible/collections/...`.
- Ready to proceed to inventory work (M2) once this step is reviewed.

## 2025-11-25 – M2 Step 2 (group vars & vault shell)

- Bootstrapped `inventory/group_vars/vps/main.yml` so the shared Docker, security, and firewall defaults live in source control and align with the PRD/implementation plan (e.g., Compose plugin, log rotation, SSH hardening flags).
- Added the `inventory/group_vars/vps/secrets.yml` placeholder that documents the required `ansible-vault encrypt_string` workflow and reserves the `cloudflare_tunnel_token` key; replace the dummy ciphertext with the real vaulted token before running playbooks.
- Avoided touching host-specific vars (Plan Step 3) so the user can validate this group-level step first.

## 2025-11-25 – M2 Step 3 (per-host vars & vault shells)

- Created `inventory/host_vars/README.md` to document the expected `main.yml` (feature toggles) vs `secrets.yml` (vaulted data) split for every host.
- Added the first concrete host directory `inventory/host_vars/lab-sfo-txy-01/` where `main.yml` keeps `cloudflared_enabled`/`app_stack_enabled` defaults false until explicitly enabled, demonstrating the per-host override surface.
- Seeded `inventory/host_vars/lab-sfo-txy-01/secrets.yml` with the sensitive tuple (`ansible_host`, `security_ssh_port`, `ansible_user`) and re-encrypted it so the SSH identity stays consistent after rotating credentials.

## 2025-11-25 – M2 Step 1 (inventory & interpreter)

- Added `inventory/hosts.yml` to enumerate `lab-sfo-txy-01` with its public `ansible_user`, leaving `ansible_host`/`security_ssh_port` references to the per-host Vault files for confidentiality.
- Introduced `inventory/group_vars/all.yml` to pin `ansible_python_interpreter: /usr/bin/python3` and added `ansible_ssh_private_key_file` that reads `ANSIBLE_SSH_KEY_PATH` (for GitHub Actions secrets) with a `~/.ssh/id_ed25519` fallback for local runs.

## 2025-11-25 – M2 Step 4 (playbook bootstrap)

- Created `playbooks/site.yml` with the agreed bootstrap flow: ensure Python 3 exists via `raw`, gather facts once Python is present, and install `python3-requests`/`python3-docker` through `ansible.builtin.apt`.
- Left the `roles` list empty (yet ordered) so subsequent milestones can append Geerlingguy roles without rewriting the core pre_tasks block.

## 2025-11-25 – M3 Step 1 (security + firewall integration)

- Extended `inventory/group_vars/vps/main.yml` with `security_ssh_allowed_users: ["{{ ansible_user }}"]` so Geerlingguy security locks SSH access down to the per-host vaulted account while reusing the existing `security_ssh_port`/firewall defaults.
- Appended `geerlingguy.security` and `geerlingguy.firewall` to `playbooks/site.yml` so that once roles are installed (via `ansible-galaxy install -r requirements.yml`), the hardening order matches the PRD (security first, firewall second).
- Galaxy downloads could not be verified inside the sandbox because outbound DNS lookups fail (`Temporary failure in name resolution`); rerun `ansible-galaxy install -r requirements.yml` from an environment with network access (or in CI) before executing the updated playbook.

## 2025-11-25 – M3 Step 2 (fail2ban + unattended upgrades policy)

- Declared all fail2ban-related knobs (`security_fail2ban_enabled`, `security_fail2ban_custom_configuration_template`) in `inventory/group_vars/vps/main.yml` so every VPS inherits a consistent SSH jail while still allowing per-host overrides when needed.
- Locked down unattended-upgrades by explicitly defining blacklist/additional-origin lists (currently empty), disabling automatic reboot, and pinning the maintenance window + mail flags; this makes the role behavior auditable instead of relying on upstream defaults.
- Playbook execution with these new variables still requires sudo + Vault credentials; sandbox runs continue to hit `/dev/shm` restrictions, so rerun `ansible-playbook ... --check --diff` on a real machine or GitHub Actions runner to confirm idempotence.

## 2025-11-25 – Molecule Scenario (Debian 13)

- Added a project-level Molecule scenario under `molecule/default/` that reuses `playbooks/site.yml` against the `bernylinville/docker-debian13-ansible:latest` image, mirroring the production inventory structure without leaking secrets.
- Brought over representative `group_vars`/`host_vars` for the virtual `vps` group (Docker, security, firewall knobs) and lightweight `verify.yml` assertions to ensure fail2ban + UFW are active inside the container.
- Developers can now validate changes locally with `molecule test` (Docker required) before touching real hosts/KVM VMs, giving the “即开即毁” workflow requested.
