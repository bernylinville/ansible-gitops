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
- `inventory/host_vars/<hostname>/main.yml` 中按主机粒度声明 Cloudflared 镜像 tag、数据目录与 Ingress 规则；生产主机可关闭 `cloudflared_whoami_enabled`/`cloudflared_verify_whoami`，而 Molecule Host 则开启它们以运行 `traefik/whoami` 验证容器，自带 catch-all `http_status:404` 兜底。
- `cloudflared_tunnel_credentials_json` 中已经包含 Tunnel ID；Role 会在运行时从 JSON 自动解析，无需在公开变量额外声明，且 Vault 仍是唯一的数据源。
- Playbook 在 `cloudflared_enabled: true` 时附加该 Role；生产主机仅运行 Cloudflared 容器并将 Ingress 指向实际服务，Molecule 场景则附带 `cloudflared-whoami` 验证容器，可通过 `curl`/`uri` 调用 `https://whoami.<cloudflared_tunnel_domain_name>` 检查 Tunnel/Ingress。Molecule 通过 symlink 复用生产 Vault secrets，而仓库级 `ansible.cfg` 默认 `vault_password_file = .vault-password`，因此只需在仓库根放置该文件（或覆盖 `--vault-password-file`）即可在本地复用真实凭据。

## Monitoring Stack on lab-sfo-txy-01 (M5 Step 2)

- 监控主机 `lab-sfo-txy-01` 启用 `prometheus.prometheus.prometheus` 与 `prometheus.prometheus.alertmanager` 角色，同时通过 `grafana.grafana.grafana` 角色部署 Grafana Server；Playbook 以 `prometheus_enabled`、`alertmanager_enabled`、`grafana_enabled` 三个布尔值按主机粒度切换。
- Prometheus 的关键变量（来自 `prometheus.yaml`）集中在 `host_vars/lab-sfo-txy-01/main.yml`：显式设置 `prometheus_storage_retention: "31d"`、`prometheus_alertmanager_config` 指向本机 9093、`prometheus_scrape_configs` 覆盖 Prometheus/Node/Alertmanager/Grafana 四个 Job，`prometheus_targets` 生成 file_sd 结构。
- Alertmanager 采用最小可用配置（`alertmanager_route` + `alertmanager_receivers`），后续只需扩充 receivers/notifications 即可；`alertmanager_config_dir` 固定为 `/etc/alertmanager` 便于 Verify 步骤定位。
- Grafana 使用 `grafana_ini` 设置 `server.domain = grafana.<domain_name>` 与 Admin 凭据，并通过 `grafana_datasources` 自动把 `http://localhost:9090` 注册为默认 Prometheus 数据源；`monitoring_grafana_datasources_path` 指向 `/etc/grafana/provisioning/datasources/datasources.yml` 以方便 Molecule 断言。
- 生产环境防火墙仅保留 SSH 端口，监控端口由 Cloudflared 暴露（无须对外开放 9090/9093/3000）；Molecule 场景可放行本地 9093/3000 以便运行 `verify.yml` 中的 systemd/HTTP/datasource 健康检查。
