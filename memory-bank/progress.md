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
