# Molecule Test Progress

## Overview
This document tracks the progress of Molecule testing for the Ansible GitOps project.

## Test Scenarios

### Default Scenario (`molecule/default`)
- **Platform**: Debian 13 (Bookworm)
- **Driver**: Docker
- **Provisioner**: Ansible
- **Verifier**: Ansible (`verify.yml`)

## Test Log

### 2025-11-25
- **Status**: Configured, Ready for Test
- **Verification**:
    - `molecule.yml` configured correctly.
    - `verify.yml` includes checks for Fail2ban, Firewall, and SSH port.
    - `molecule list` shows instance state as `created: false`, `converged: false`.
- **Next Steps**:
    - Run `molecule test` to validate the full cycle (create, converge, verify, destroy).
    - Address any issues with Docker connectivity or role execution if they arise.

### 2025-11-26
- **Status**: Extended coverage
- **Verification**:
    - Scenario now enables `cloudflared` with a dummy sleep command so containers remain healthy without real tunnel credentials while still allowing real Cloudflare credentials via the shared `inventory/group_vars/vps/secrets.yml` (symlinked into Molecule).
    - `verify.yml` asserts the shared `proxy_net` network exists, `/opt/cloudflared/*` artifacts are created, both `cloudflared` + `cloudflared-whoami` containers attach to the shared network, and (when `MOLECULE_CLOUDFLARE_VERIFY=true`) performs an HTTPS GET against `https://whoami.<domain>` through the tunnel.
    - 仓库级 `ansible.cfg` 指定 `vault_password_file = .vault-password`，并通过 `molecule/default/molecule.yml` 的 `ANSIBLE_CONFIG=${MOLECULE_PROJECT_DIRECTORY}/ansible.cfg` 环境变量强制加载，因此在仓库根放置该文件（或用 `--vault-password-file` 覆盖）即可让 Molecule 自动解密 Vault；`group_vars/vps/secrets.yml` 也被 symlink 进场景，可直接使用生产 Cloudflare 凭据。
    - `cloudflared_verify_whoami` 在 Molecule 默认开启（除非显式设置 `MOLECULE_CLOUDFLARE_VERIFY=false`），运行过程中会多次尝试 `https://whoami.<domain>`，帮助发现 Cloudflare 隧道/证书传播造成的抖动。
- **Next Steps**:
    - Export `MOLECULE_CLOUDFLARE_VERIFY=true` and run `molecule test` after providing valid Cloudflare credentials to validate end-to-end tunnel reachability.

### 2025-11-27
- **Status**: Monitoring stack coverage
- **Verification**:
    - Host vars enable Prometheus/Alertmanager/Grafana inside the scenario with identical settings to production (storage retention 31d, Alertmanager route, Grafana datasource).
    - `verify.yml` now gathers service facts, checks Prometheus retention flags via `systemctl cat`, ensures `file_sd/node.yml` exists, and exercises HTTP readiness endpoints for Prometheus/Alertmanager plus Grafana's login path.
    - Datasource provisioning file (`/etc/grafana/provisioning/datasources/datasources.yml`) is slurped and asserted to point at `http://localhost:9090`.
- **Next Steps**:
    - 在 `molecule/default/host_vars/debian13/secrets.yml`（可使用 Vault）写入 `grafana_admin_password` 后运行 `molecule converge && molecule verify`，验证监控栈在全新环境中的可重复性。
