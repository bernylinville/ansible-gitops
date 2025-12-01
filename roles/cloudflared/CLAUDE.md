[根目录](../../CLAUDE.md) > [roles](../) > **cloudflared**

---

# Cloudflared Role - 模块文档

## 模块职责

`cloudflared` 角色负责在目标主机上部署和配置 Cloudflare Tunnel 容器，实现零信任网络接入。该角色专注于：

1. 渲染 Cloudflared 配置文件（`config.yml`）
2. 安全存储隧道凭据（`credentials.json`）
3. 启动 `cloudflare/cloudflared` 容器并加入共享 Docker 网络
4. （可选）部署 `traefik/whoami` 验证容器用于测试

**关键特性**：
- 从加密的 Vault 变量中自动解析 Tunnel ID
- 支持动态 Ingress 规则配置
- 与 `docker_custom` 角色集成（自动加入 `proxy_net`）
- Molecule 测试友好（可选启用 whoami 验证容器）

---

## 入口与启动

### 入口文件
- `tasks/main.yml` - 主任务流程

### 启动流程

1. **验证配置上下文**
   - 断言 `cloudflared_tunnel_credentials_json` 和 `cloudflared_ingress_rules` 已定义
   - 解析 JSON 凭据并提取 Tunnel ID

2. **准备文件系统**
   - 创建数据目录（默认 `/opt/cloudflared`）
   - 渲染 `config.yml`（使用 `templates/config.yml.j2`）
   - 写入 `credentials.json`（敏感操作，使用 `no_log: true`）

3. **启动容器**
   - 构建容器网络附加列表（`proxy_net` + 额外网络）
   - （可选）启动 whoami 验证容器
   - 启动 cloudflared 容器（配置或凭据变更时自动重建）

### 运行命令示例

```bash
# 在 playbooks/site.yml 中条件性引入
- role: cloudflared
  when: cloudflared_enabled | default(false)
```

---

## 对外接口

### 关键变量（从 inventory 注入）

#### 必需变量

| 变量名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| `cloudflared_tunnel_credentials_json` | String (JSON) | Tunnel 凭据（Vault 加密） | `'{"TunnelID":"abc123",...}'` |
| `cloudflared_ingress_rules` | List[Dict] | Ingress 路由规则 | 见下方示例 |

#### 可选变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `cloudflared_image_tag` | `"2025.11.1"` | Cloudflared 镜像版本 |
| `cloudflared_data_directory` | `/opt/cloudflared` | 配置文件存储路径 |
| `cloudflared_whoami_enabled` | `false` | 是否启动验证容器 |
| `cloudflared_verify_whoami` | `false` | 是否在测试中验证 whoami 端点 |
| `cloudflared_etc_hosts` | `{}` | 容器额外 hosts 映射 |
| `cloudflared_extra_networks` | `[]` | 除 `proxy_net` 外的额外网络 |

### Ingress 规则示例

```yaml
cloudflared_ingress_rules:
  - hostname: "grafana.devopsthink.org"
    service: "http://host.docker.internal:3000"
  - hostname: "whoami.devopsthink.org"
    service: "http://cloudflared-whoami:80"
  - service: "http_status:404"  # 必需的 catch-all 规则
```

### 容器暴露

- **Cloudflared 容器**: 无需暴露端口（出站连接到 Cloudflare Edge）
- **Whoami 容器**: 仅在 `proxy_net` 内暴露 80 端口

---

## 关键依赖与配置

### 角色依赖

- **`docker_custom`**: 必须在此角色之前执行，确保 `proxy_net` 网络存在
- **`geerlingguy.docker`**: 提供 Docker Engine 和 Python SDK

### 集合依赖

```yaml
# collections/requirements.yml
- name: community.docker
  version: 5.0.2
```

### 配置文件

#### templates/config.yml.j2
```jinja2
tunnel: {{ cloudflared_effective_tunnel_id }}
credentials-file: {{ cloudflared_credentials_container_path }}
ingress:
{{ cloudflared_ingress_rules | to_nice_yaml(indent=2) | indent(2, true) }}
```

#### defaults/main.yml（部分）
```yaml
cloudflared_container_name: cloudflared
cloudflared_image: cloudflare/cloudflared
cloudflared_image_tag: "2025.11.1"
cloudflared_restart_policy: unless-stopped
cloudflared_config_path: "{{ cloudflared_data_directory }}/config.yml"
cloudflared_credentials_path: "{{ cloudflared_data_directory }}/credentials.json"
cloudflared_config_container_path: /etc/cloudflared/config.yml
cloudflared_credentials_container_path: /etc/cloudflared/credentials.json
```

---

## 数据模型

### Vault 秘密结构

存储在 `inventory/group_vars/vps/secrets.yml`:

```yaml
# ansible-vault encrypt 加密后的内容
cloudflared_tunnel_credentials_json: |
  {
    "AccountTag": "abc123...",
    "TunnelSecret": "def456...",
    "TunnelID": "ghi789...",
    "TunnelName": "my-tunnel"
  }
```

### 运行时变量解析

```yaml
# 任务流程中自动解析
cloudflared_credentials_struct: "{{ cloudflared_tunnel_credentials_json | from_json }}"
cloudflared_effective_tunnel_id: "{{ cloudflared_credentials_struct.TunnelID }}"
```

### 网络附加逻辑

```yaml
# 动态构建网络列表
cloudflared_network_attachments: >-
  {{ ([{'name': docker_custom_name}] if docker_custom_name is defined else []) +
     (cloudflared_extra_networks | default([])) }}
```

---

## 测试与质量

### Molecule 测试配置

**场景**: `molecule/default/`

#### 测试主机变量覆盖

```yaml
# molecule/default/host_vars/debian13/main.yml
cloudflared_enabled: true
cloudflared_whoami_enabled: true  # 启用验证容器
cloudflared_verify_whoami: false  # CI 中无法访问公网，禁用 HTTPS 验证
cloudflared_ingress_rules:
  - hostname: "whoami.devopsthink.org"
    service: "http://{{ cloudflared_whoami_container_name }}:80"
  - service: http_status:404
```

#### 验证断言（molecule/default/verify.yml）

```yaml
# （当前已注释，因生产场景不启用 Cloudflared）
# - name: Inspect cloudflared container
#   community.docker.docker_container_info:
#     name: "{{ cloudflared_container_name }}"
#   register: cloudflared_container_info

# - name: Assert cloudflared containers exist
#   ansible.builtin.assert:
#     that:
#       - cloudflared_container_info.exists
#       - docker_custom_name in (cloudflared_container_info.container.NetworkSettings.Networks.keys())
```

### 质量检查

```bash
# 角色级 ansible-lint
ansible-lint roles/cloudflared/

# 容器状态检查
docker ps -f name=cloudflared
docker logs cloudflared
```

---

## 常见问题 (FAQ)

### Q1: 如何获取 Tunnel 凭据 JSON？

**A**: 使用 Cloudflare Tunnel CLI 创建隧道：

```bash
# 创建隧道
cloudflared tunnel create my-tunnel

# 凭据文件位置（复制 JSON 内容到 Vault）
cat ~/.cloudflared/<tunnel-id>.json
```

### Q2: 为什么容器使用 `host.docker.internal`？

**A**: Cloudflared 需要访问宿主机上的服务（如 Grafana 3000 端口）。通过 `cloudflared_etc_hosts` 将 `host.docker.internal` 映射到 `docker_custom_gateway`（10.203.57.1），实现容器到主机的反向访问。

```yaml
cloudflared_etc_hosts:
  host.docker.internal: "{{ docker_custom_gateway }}"
```

### Q3: 如何添加新的 Ingress 规则？

**A**: 编辑主机的 `inventory/host_vars/<hostname>/main.yml`：

```yaml
cloudflared_ingress_rules:
  - hostname: "new-service.devopsthink.org"
    service: "http://service-container:8080"
  - service: "http_status:404"  # 保持 catch-all 规则在最后
```

重新运行 playbook，角色会自动重建容器。

### Q4: 为什么 Molecule 测试中禁用 `cloudflared_verify_whoami`？

**A**: CI 环境中的容器无法访问公网 DNS 和 Cloudflare Edge，HTTPS 验证会失败。生产环境可启用此功能验证隧道端到端可达性。

### Q5: 如何调试 Tunnel 连接问题？

**A**:
```bash
# 查看容器日志
docker logs cloudflared -f

# 检查 Ingress 配置
docker exec cloudflared cat /etc/cloudflared/config.yml

# 测试容器网络连通性
docker exec cloudflared ping <service-container-name>
```

---

## 相关文件清单

```
roles/cloudflared/
├── defaults/
│   └── main.yml              # 默认变量定义
├── tasks/
│   └── main.yml              # 主任务流程（95 行）
└── templates/
    └── config.yml.j2         # Cloudflared 配置模板（5 行）

inventory/group_vars/vps/
└── secrets.yml               # Vault 加密的 Tunnel 凭据

inventory/host_vars/lab-sfo-txy-01/
└── main.yml                  # 主机级 Ingress 规则配置

molecule/default/
├── group_vars/all/
│   └── secrets.yml           # 测试用凭据（symlink 到生产 Vault）
└── host_vars/debian13/
    └── main.yml              # 测试场景变量覆盖
```

---

## 变更记录 (Changelog)

### 2025-12-01
- **初始化**: 通过 AI 架构师生成完整的角色文档
- **当前状态**: 生产就绪，已在 lab-sfo-txy-01 部署 Grafana Ingress
- **测试覆盖**: Molecule 验证容器启动与网络附加
