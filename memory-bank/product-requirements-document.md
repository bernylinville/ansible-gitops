# 产品需求文档 (PRD): Ansible GitOps

## 1. 项目概述

**项目名称**: Ansible GitOps for VPS & Homelab
**核心策略**: **Configuration over Creation**。优先集成成熟的开源 Ansible Roles（以 Geerlingguy 为主），仅针对 Cloudflare Tunnel 或特定业务逻辑编写少量自定义 Glue Code（胶水代码）。
**目标系统**: Debian 13 (Trixie)

## 2. 核心架构

- **基础设施即代码 (IaC)**: 全量配置代码化。
- **依赖管理**: 使用 `ansible-galaxy` + `requirements.yml` 管理第三方 Roles。
- **网络接入**: Cloudflare Tunnel (无需入站端口)。
- **容器化**: 所有业务服务 Docker 化。

## 3. 组件选型 (Role Strategy)

我们将优先使用以下成熟 Roles，通过 `inventory` 变量进行配置，而非修改代码：

| 功能模块     | 推荐 Role (Galaxy)         | 核心用途                                | 备注                        |
| :----------- | :------------------------- | :-------------------------------------- | :-------------------------- |
| **Docker**   | `geerlingguy.docker`       | 安装 Docker Engine, Compose, Python SDK | 业内标准，稳定              |
| **基础安全** | `geerlingguy.security`     | SSH 加固, 禁用 Root, 自动更新配置       | 开箱即用的安全基线          |
| **防火墙**   | `geerlingguy.firewall`     | 管理 UFW/Iptables                       | 实现"默认拒绝，只允许 SSH"  |
| **系统基础** | 自定义 / `geerlingguy.git` | 安装基础包 (curl, wget, git)            | 简单任务可直接写在 Playbook |
| **内网穿透** | 自定义 (Local Role)        | 部署 `cloudflared` 容器                 | 需处理 Token 注入逻辑       |

## 4. 功能需求 (Functional Requirements)

### 4.1 依赖管理 (P0)

- 维护 `requirements.yml`，锁定 Role 版本号，确保构建的可重复性。
- CI/CD 流程必须包含 `ansible-galaxy install` 步骤。

### 4.2 VPS 初始化与加固 (P0)

- **调用 `geerlingguy.security`**:
  - 配置 `security_ssh_allowed_users`: 仅允许运维用户。
  - 配置 `security_ssh_password_authentication`: "no"。
- **调用 `geerlingguy.firewall`**:
  - 配置 `firewall_allowed_tcp_ports`: `[22]` (仅保留 SSH 管理通道)。
  - 确保默认入站策略为 REJECT/DROP。
- **调用 `geerlingguy.docker`**:
  - 配置 `docker_users`: 将运维用户加入 docker 组。
  - 配置 `docker_compose_package`: 确保安装 Compose 插件。

### 4.3 网络与服务 (P0 - P1)

- **Cloudflare Tunnel (自定义 Role)**:
  - 拉取 `cloudflare/cloudflared` 镜像。
  - 使用 `CF_TUNNEL_TOKEN` 启动 Tunnel。
  - 加入 `proxy_net` Docker 网络。
- **应用部署**:
  - 使用 Ansible 的 `community.docker.docker_container` 模块或 `docker_compose` 模块部署业务容器。
  - 业务容器不暴露端口到宿主机，仅暴露给 `proxy_net` 供 Tunnel 访问。

## 5. 目录结构规范

引入社区 Role 后，目录结构更加简洁：

```text
.
├── .github
│   └── workflows
│       └── gitops.yml      # 增加 galaxy install 步骤
├── inventory
│   ├── hosts.yml
│   └── group_vars
│       ├── all.yml         # 全局变量 (Python路径等)
│       └── vps.yml         # 核心配置：定义 docker_users, firewall_rules 等
├── roles
│   └── cloudflared         # 自定义的小型 Role，仅处理 Tunnel
├── playbooks
│   └── site.yml            # 编排 Community Roles 的执行顺序
├── ansible.cfg             # 配置 roles_path
└── requirements.yml        # 核心：列出 geerlingguy.docker 等
```

## 6. GitOps 工作流 (Updated)

1.  **CI 环境准备**: GitHub Actions 启动 Runner。
2.  **依赖安装**: 执行 `ansible-galaxy install -r requirements.yml` 下载社区 Roles。
3.  **凭证注入**: 解密 Vault 密码，加载 SSH Key，注入 Cloudflare Token。
4.  **Playbook 执行**:
    - Step 1: `geerlingguy.security` (系统加固)
    - Step 2: `geerlingguy.firewall` (防火墙收口)
    - Step 3: `geerlingguy.docker` (安装环境)
    - Step 4: `local.cloudflared` (建立隧道)
    - Step 5: 部署业务容器

## 7. 关键配置文件示例

### 7.1 requirements.yml

```yaml
roles:
  - name: geerlingguy.docker
    version: 7.8.0
  - name: geerlingguy.security
    version: 3.0.0
  - name: geerlingguy.firewall
    version: 2.7.0
```

### 7.2 inventory/group_vars/vps.yml (配置化核心)

```yaml
# Docker 配置
docker_edition: "ce"
docker_packages:
  - "docker-{{ docker_edition }}"
  - "docker-{{ docker_edition }}-cli"
  - "containerd.io"
  - "docker-buildx-plugin"
docker_packages_state: present
docker_users: ["myadmin"]
# Docker Compose Plugin options.
docker_install_compose_plugin: true
docker_compose_package: docker-compose-plugin
docker_compose_package_state: present
# Docker daemon options.
docker_daemon_options:
  storage-driver: "overlay2"
  log-opts:
    max-size: "100m"

# 防火墙配置 (仅 SSH)
firewall_allowed_tcp_ports:
  - "22"
firewall_state: started
firewall_enabled_at_boot: true

# 安全配置
security_ssh_port: 22
security_ssh_password_authentication: "no"
security_ssh_permit_root_login: "no"
security_autoupdate_enabled: true
```

### 7.3 GitHub Action Step (Diff)

```yaml
- name: Install Ansible Roles
  run: ansible-galaxy install -r requirements.yml
```

## 8. 实施路线图

1.  **M1 - 骨架搭建**: 创建 Git 仓库，编写 `requirements.yml`，配置 GitHub Actions。
2.  **M2 - 参数调试**: 在本地 Vagrant 或测试机上调试 `group_vars`，确保 Geerlingguy 的 Roles 能在 Debian 13 上正确运行（可能需要设置 `ansible_python_interpreter`）。
3.  **M3 - VPS 接入**: 跑通 CI，完成 VPS 的 Docker 化和 SSH 加固。
4.  **M4 - 隧道打通**: 编写简单的 Cloudflared Task，实现内网穿透。
5.  **M5 - 业务上线**: 定义第一个业务容器的 Ansible Task。
