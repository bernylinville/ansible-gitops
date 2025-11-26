# Architecture Overview

## Repository Skeleton (M1 Step 1)

- `.github/workflows/`: reserved for GitHub Actions workflows (lint + deploy). Pre-creating the folder lets us drop workflow YAML later without touching other docs.
- `inventory/`: root for inventory data. `group_vars/vps/` prepares shared variables (including the future `main.yml`/`secrets.yml` split) while `host_vars/` will hold per-host overrides that mirror the implementation plan.
- `playbooks/`: home for orchestration entry points such as `playbooks/site.yml`, keeping the repo root uncluttered as scenarios multiply.
- `roles/`: target directory for Galaxy-installed roles and local roles; keeping it empty but present allows idempotent `ansible-galaxy install` runs once dependencies are declared.
- `ansible.cfg`: currently contains `[defaults] roles_path = roles`, ensuring both local runs and CI resolve roles without extra CLI flags; additional defaults can build on this file later.

This minimal scaffold enforces the agreed Ansible-first layout and is the foundation for the remaining M1 tasks (requirements files, inventory population, and playbook scaffolding).

## Shared Docker Network (M4 extension)

- M4 增补 `roles/docker_shared_network`，在 `geerlingguy.docker` 完成安装后立即创建/维护所有 self-hosted 服务共用的 `proxy_net` 桥接网络（默认 10.203.57.0/24，可通过 `docker_shared_network_*` 变量覆盖）。
- 任意容器（Cloudflared、业务服务等）都必须加入该网络，确保后续反向代理/零信任隧道的连通性；网络属性（subnet/gateway/attachable）集中定义在 `inventory/group_vars/vps/main.yml`，便于统一管理。

## Cloudflared + Proxy Network (M5 Step 1)

- `roles/cloudflared` 专注于渲染 `/opt/cloudflared/{config.yml,credentials.json}` 并运行 `cloudflare/cloudflared:<tag>` + `traefik/whoami` 验证容器，网络依赖交由前述 `docker_shared_network` role 提供。
- 凭据改为 `cloudflared_tunnel_credentials_json`（`cloudflared tunnel create` 输出的 JSON），通过 Vault 在 `inventory/group_vars/vps/secrets.yml` 注入，模板渲染时直接写入 `credentials.json`。
- `inventory/group_vars/vps/main.yml` 中集中声明 Cloudflared 镜像 tag、数据目录、Ingress 规则和 `whoami` 验证容器（`traefik/whoami`）的主机名；自带 catch-all `http_status:404` 兜底，并通过 `cloudflared_verify_whoami` 控制是否在部署后执行公开域名的 HTTP 健康检查。
- `cloudflared_tunnel_credentials_json` 中已经包含 Tunnel ID；Role 会在运行时从 JSON 自动解析，无需在公开变量额外声明，且 Vault 仍是唯一的数据源。
- Playbook 在 `cloudflared_enabled: true` 时附加该 Role，并自动拉起验证容器 `cloudflared-whoami`，可通过 `curl`/`uri` 调用 `https://whoami.<cloudflared_tunnel_domain_name>` 检查 Tunnel/Ingress；Molecule 通过 symlink 复用生产 Vault secrets，而仓库级 `ansible.cfg` 默认 `vault_password_file = .vault-password`，因此只需在仓库根放置该文件（或覆盖 `--vault-password-file`）即可在本地复用真实凭据。
