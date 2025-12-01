# 实施计划

以下计划在阅读 PRD、Tech Stack 后统一了关键决策与编号，按 M1–M7 逐步推进；每个里程碑完成后请更新 memory-bank/architecture.md 与 memory-bank/progress.md。

- 关键决策（已定）：
  - Inventory 采用：`inventory/group_vars/vps/` 存放共享配置，`inventory/host_vars/<host>/{main.yml,secrets.yml}` 存放每台主机的个性化与敏感配置（`secrets.yml` 使用 Vault）。
  - Galaxy Roles 统一由根 `requirements.yml` 管理并锁定 geerlingguy 版本；Collections 额外维护在 `collections/requirements.yml`（至少包含 `community.docker`，因为控制端仅安装 `ansible-core`）。
  - Docker 官方源与 keyring 全部交由 `geerlingguy.docker` Role 处理。
  - 连接模型：普通用户 + sudo；固定解释器 `/usr/bin/python3`。
  - 依赖策略：一律 apt 安装（如 `python3-docker`, `python3-requests`）。
  - CI 使用 `ssh-keyscan "$INVENTORY_HOST" >> ~/.ssh/known_hosts` 并启用独立 `ansible-lint` Job + Galaxy 缓存。
  - 每台主机的 SSH 端口在拿到 VPS/Homelab 时已手动改成自定义端口；项目只在 `host_vars/<host>/secrets.yml` 中记录 `security_ssh_port`，并在 `inventory/hosts.yml` 通过 `ansible_port` 对应，Playbook 不做迁移操作。
  - Cloudflared 使用统一的 Tunnel Token（所有主机共享一个 Tunnel）；Token 仅存在 `inventory/group_vars/vps/secrets.yml` 中的 Vault，CI 只注入 Vault 密码。

---

## M1：项目骨架与依赖管理

1. 初始化目录与配置

- 目录：`.github/workflows/`, `inventory/{group_vars/vps,host_vars}`, `playbooks/`, `roles/`, `memory-bank/`。
  - 约定：每台主机一个目录，例如 `inventory/host_vars/vps01/{main.yml,secrets.yml}`。
- `ansible.cfg`：设置 `roles_path = roles`；保持默认 `host_key_checking`（CI 中处理 known_hosts）。
- 验证：`ls -R` 与 `ansible-config dump | rg roles_path`。

2. 定义 Roles 依赖（根 requirements.yml）

```yaml
roles:
  - name: geerlingguy.docker
    version: 7.8.0
  - name: geerlingguy.security
    version: 3.0.0
  - name: geerlingguy.firewall
    version: 2.7.0
```

- 安装与验证：`ansible-galaxy install -r requirements.yml`，确认 `roles/` 下生成目录。

3. 定义 Collections 依赖（`collections/requirements.yml`）

```yaml
collections:
  - name: community.docker
    version: 5.0.2 # 示例版本，实际使用前可按当时稳定版调整，并同步更新 PRD/Tech Stack/本计划
```

- 安装与验证：`ansible-galaxy collection install -r collections/requirements.yml`，并通过 `ansible-doc community.docker.docker_container` 确认模块可用；缓存目录与 Roles 一样放在主机 `~/.ansible/`。

---

## M2：Inventory 与基础连接

1. Inventory 与全局解释器

- `inventory/hosts.yml` 定义组 `vps` 与目标主机，直接写入实际 IP 与 `ansible_port`、`ansible_user`。示例：

```yaml
vps:
  hosts:
    vps01:
      ansible_host: 203.0.113.10
      ansible_port: 52222
      ansible_user: deploy
```

- `inventory/group_vars/all.yml`

```yaml
ansible_python_interpreter: /usr/bin/python3
```

2. 组变量与 Vault

- `inventory/group_vars/vps/main.yml`（非敏感）与 `inventory/group_vars/vps/secrets.yml`（Vault 加密）。
- `inventory/group_vars/vps/main.yml` 还需定义共享 Docker 网络参数：

```yaml
docker_custom_name: proxy_net
docker_custom_subnet: 10.203.57.0/24
docker_custom_gateway: 10.203.57.1
docker_custom_ipam_config:
  - subnet: "{{ docker_custom_subnet }}"
    gateway: "{{ docker_custom_gateway }}"
```

- 在 `inventory/group_vars/vps/secrets.yml` 中放置共享的 Cloudflared Tunnel 凭据 JSON（`cloudflared tunnel create` 生成，内含 Tunnel ID）：

```yaml
cloudflared_tunnel_credentials_json: !vault |
  # vaulted JSON blob
```

- Cloudflared Token 只放在 Vault；CI 不再通过 `--extra-vars` 注入敏感值。

3. 每台主机的 host_vars 与 Vault

- 对每台主机创建 `inventory/host_vars/<host>/main.yml` 与 `inventory/host_vars/<host>/secrets.yml`（使用 Vault 加密）。
- 在 `host_vars/<host>/secrets.yml` 中放置该主机的 SSH 端口：

```yaml
security_ssh_port: 52222 # 示例，需与该主机实际 sshd 端口、inventory 中 ansible_port 保持一致
```

- 在 `host_vars/<host>/main.yml` 中控制该主机是否启用 Cloudflared：

```yaml
cloudflared_enabled: false # 默认关闭，仅对需要隧道的主机显式设为 true
```

4. 主 Playbook 骨架（先保证可连通与 Python 准备）

- `playbooks/site.yml`（示例片段）

```yaml
- hosts: vps
  gather_facts: false
  become: true
  pre_tasks:
    - name: Ensure Python exists (raw)
    - raw: test -x /usr/bin/python3 || (apt-get update && apt-get install -y python3 python3-apt)
    - name: Gather facts
      setup:
    - name: OS deps for Ansible docker modules
      ansible.builtin.apt:
        name: [python3-requests, python3-docker]
        state: present
        update_cache: true
  roles: []
```

- 验证：`ansible -i inventory/hosts.yml vps -m ping`、`--syntax-check`。

---

## M3：系统加固与防火墙（保持既有自定义端口）

1. 变量约定

- `security_ssh_password_authentication: "no"`，`security_ssh_permit_root_login: "no"`，`security_autoupdate_enabled: true`。
- `security_ssh_port` 与每台主机的 `ansible_port` 必须与人工预设的端口一致；不在此项目里修改端口，只做一致性与幂等校验。
- 防火墙仅允许 `security_ssh_port`，其余入站全部拒绝。

2. Role 行为

- 在 `inventory/group_vars/vps/main.yml` 设定 geerlingguy.security/geerlingguy.firewall 需要的共享变量。
- 在 `host_vars/<host>/secrets.yml` 中定义每台主机的 `security_ssh_port`，供 geerlingguy.security/geerlingguy.firewall 引用，例如：

```yaml
firewall_allowed_tcp_ports:
  - "{{ security_ssh_port }}"
```

- `geerlingguy.security` 会重新应用 sshd 配置，但由于端口与服务器现状一致，因此不会触发变更；若检测到差异可视为告警。

3. 将 Roles 加入 `playbooks/site.yml`

```yaml
roles:
  - geerlingguy.security
  - geerlingguy.firewall
```

- 验证：`ansible-playbook ... --check --diff` 应显示最少变更；在外部终端确认只能通过密钥 + 自定义端口登录，密码登录应失败。

---

## M4：容器运行时环境（Docker）

1. 变量与 Role

- 采用 `geerlingguy.docker` 默认源与 keyring；配置必要选项（如 docker_users）。
- 依赖由前置 apt 安装的 `python3-docker`/`python3-requests` 支持 Ansible Docker 模块。

2. 将 Role 加入 `site.yml`

```yaml
roles:
  - geerlingguy.security
  - geerlingguy.firewall
  - geerlingguy.docker
  - docker_custom
```

- 验证：使用 Molecule (Vagrant Driver) 进行本地测试，避免 Docker-in-Docker 问题。
- 生产验证：SSH 到服务器执行 `docker --version`、`docker compose version`；非 root 用户 `docker ps` 可用。
- 网络：同阶段确认 `docker network inspect proxy_net` 成功（共享 `docker_custom_name` 供所有容器加入）。

---

## M5：Cloudflare Tunnel（容器化）

1. 自定义轻量 Role `roles/cloudflared`（依赖 `community.docker` Collection，前置共享网络已由 `docker_custom` Role 创建）：

```yaml
- name: Render /opt/cloudflared/config.yml
  ansible.builtin.template:
    src: config.yml.j2
    dest: /opt/cloudflared/config.yml

- name: Copy credentials.json (Vaulted)
  ansible.builtin.copy:
    content: "{{ cloudflared_tunnel_credentials_json }}"
    dest: /opt/cloudflared/credentials.json

- name: Run cloudflared tunnel container (per-host opt-in)
  community.docker.docker_container:
    name: cloudflared
    image: cloudflare/cloudflared:2025.11.1 # 示例 tag，实际使用前可按当时稳定版调整并同步文档
    restart_policy: unless-stopped
    networks:
      - name: "{{ docker_custom_name }}"
    command:
      - "tunnel"
      - "--config"
      - "/etc/cloudflared/config.yml"
      - "run"
      - "{{ cloudflared_tunnel_id }}"
    volumes:
      - "/opt/cloudflared/config.yml:/etc/cloudflared/config.yml"
      - "/opt/cloudflared/credentials.json:/etc/cloudflared/credentials.json"
  when: cloudflared_enabled | default(false)
```

- `cloudflared_tunnel_credentials_json` 由 Vault 解密提供，无需 CLI `--extra-vars`，Role 会从 JSON 自动解析 `TunnelID`，只有在需要覆盖默认值时才显式设置 `cloudflared_tunnel_id`。
- 验证：`docker ps` 容器 Up；`docker logs cloudflared` 含 "Registered tunnel connection"；Cloudflare 控制台显示 Healthy；部署 `whoami` 容器并通过 `curl https://whoami.<domain>`/Ansible `uri` 模块（受 `cloudflared_verify_whoami` 控制）验证 Ingress。Molecule 场景可通过设定 `MOLECULE_CLOUDFLARE_VERIFY=true` 重用生产 Vault secrets 触发相同测试。
- 验证容器：仅在 Molecule 或调试场景下开启 `cloudflared_whoami_enabled`，域名 `whoami.<cloudflared_tunnel_domain_name>` 用于快速 `curl` 健康检查；生产主机默认关闭此容器，仅保留业务 Ingress 与 `http_status:404` 兜底。

### M5 Step 2：监控栈（lab-sfo-txy-01）

- 对监控主机开启 `prometheus_enabled`、`alertmanager_enabled`、`grafana_enabled`，Playbook 末尾串联 `prometheus.prometheus.prometheus` → `prometheus.prometheus.alertmanager` → `grafana.grafana.grafana`，其余主机默认不执行。
- Prometheus：
  - `prometheus_storage_retention: "31d"` 显式控制保留期；`prometheus_alertmanager_config` 指向本机 9093；`prometheus_targets` 生成 node file_sd。
  - `prometheus_scrape_configs` 扩展默认 job（Prometheus + Node）并追加 Alertmanager/Grafana job，`prometheus.yaml` 中的结构作为参考模板。
- Alertmanager：`alertmanager_route` + `alertmanager_receivers` 组合最精简的 default route；`alertmanager_config_dir=/etc/alertmanager` 统一 Verify 路径。
- Grafana：通过 `grafana_ini` 设置域名 `grafana.<domain_name>`，Pin 稳定版 `grafana_version: 12.3.0`，管理员固定 `kchou`，`grafana_admin_password` 仅从 `inventory/host_vars/<host>/secrets.yml` 读取；`grafana_datasources` 自动注册 `http://localhost:9090` DataSource 并写入 `/etc/grafana/provisioning/datasources/datasources.yml`。
- 防火墙：生产主机仅开放 SSH（其余端口由 Cloudflared 暴露在 overlay 网络中）；Molecule 场景可按需放行本地 9093/3000 以通过验证。
- Molecule：`host_vars/debian13` 复制同样的监控变量，`verify.yml` 新增 systemd/URI/datasource 检查与 Prometheus retention flag 断言，确保 CI 中能验证监控栈。

---

## M6：CI/CD（GitHub Actions）

1. Secrets（最终集）：

- `VAULT_PASSWORD`、`SSH_PRIVATE_KEY`、`INVENTORY_HOST`（Cloudflared Token 不再以 GitHub Secret 形式提供）。

2. Workflow 结构（建议 `.github/workflows/gitops.yml`）

- `lint` Job：安装 Ansible、运行 `ansible-lint`。
- `deploy` Job（依赖 lint）：
  - Checkout；安装 Ansible。
  - 写入 SSH 私钥并 `chmod 600`；`ssh-keyscan "$INVENTORY_HOST" >> ~/.ssh/known_hosts`。
  - `ansible-galaxy install -r requirements.yml` 与 `ansible-galaxy collection install -r collections/requirements.yml`。
  - 将 `VAULT_PASSWORD` 写入 `vault_pass` 文件，仅供 `--vault-password-file` 使用。
  - 执行 `ansible-playbook -i inventory/hosts.yml playbooks/site.yml --vault-password-file ./vault_pass`；Cloudflared Token 会在运行时解密。
- 缓存：使用 `actions/cache` 分别缓存 `~/.ansible/roles` 与 `~/.ansible/collections`，Key 由对应 requirements 文件的 hash 计算。

---

## M7：业务容器（待定）

1. 目标

- 为将来选择的开源 self-hosted 服务预留实施空间，不在当前阶段锁定具体服务列表，仅约定技术路径与抽象接口。

2. 约定与约束

- 所有业务容器：
  - 运行在 Docker 上，并加入 `proxy_net` 网络，通过 Cloudflared 暴露。
  - 不直接在宿主机暴露公网端口（如无特殊说明）。
  - 镜像 tag 避免使用 `latest`，尽量固定到具体版本。

3. 建议实现路径（占位，待具体服务确定后细化）

- 新增 `roles/app_stack`（或按服务粒度拆分多个小 Role），内部使用 `community.docker.docker_container`/`docker_compose_v2`。
- 在 `inventory/host_vars/<host>/main.yml` 中，为每台需要部署业务的主机配置：

```yaml
app_stack_enabled: false # 默认关闭，后续按主机/服务打开
```

- 在未来选定第一个服务时：
  - 更新 PRD 与 Tech Stack，写清服务列表与端口/存储需求。
  - 将本节扩展为具体的 M7.x 子任务（部署第一个应用、卷与备份策略等）。

---

## 验收清单（阶段性）

- M3：仅密钥登录；自定义 SSH 端口与 `ansible_port`/Vault 记录一致，且仅该端口对外开放。
- M4：Docker/Compose 可用；非 root 用户可运行 docker；Molecule 测试通过（Vagrant）。
- M5：Cloudflared 容器 Healthy；业务网络 `proxy_net` 存在；按 host_vars 中 `cloudflared_enabled` 控制启用范围。
- M6：Actions 绿灯；lint 独立；Galaxy 角色/collections 可缓存；`known_hosts` 由 `ssh-keyscan` 生成。
- M7：预留 app stack 结构存在但默认关闭；未来选定具体 self-hosted 服务后可在不破坏已有结构的前提下增量接入。
