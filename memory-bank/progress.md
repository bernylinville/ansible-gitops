## 2025-11-25 – M1 Step 1

- Created the baseline directories required for the Ansible-first layout: `.github/workflows/`, `inventory/group_vars/vps/`, `inventory/host_vars/`, `playbooks/`, and `roles/`.
- Added `ansible.cfg` with `[defaults] roles_path = roles` so Galaxy roles resolve correctly in later steps.
- Deferred verification of `ansible-config dump | rg roles_path` to the user environment because `ansible-config` is unavailable locally.

Next dev can proceed with M1 Step 2 once any additional validation is complete.
