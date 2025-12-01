[根目录](../../CLAUDE.md) > [roles](../) > **docker_shared_network**

---

# Docker Shared Network Role - 模块文档

## 模块职责

`docker_shared_network` 角色负责创建和维护所有容器化服务共享的 Docker 桥接网络 `proxy_net`。该角色是 Cloudflared、Traefik、Gemi2api 等自托管服务的网络基础设施。

**核心功能**：
1. 创建名为 `proxy_net` 的自定义桥接网络
2. 配置固定子网（10.203.57.0/24）和网关（10.203.57.1）
3. 启用 `attachable` 属性，允许非 Swarm 容器动态加入
4. 提供统一的 IPAM 配置，便于跨角色引用

**设计理念**：
- **单一职责**：仅负责网络创建，不管理容器
- **幂等性**：重复执行不会导致错误
- **配置化**：所有参数通过变量控制，无硬编码

---

## 入口与启动

### 入口文件
- `tasks/main.yml` - 单一任务文件（11 行）

### 启动流程

```yaml
# playbooks/site.yml 中的执行顺序
- role: geerlingguy.docker        # 1. 安装 Docker Engine
  become: true
- role: docker_shared_network     # 2. 创建共享网络
- role: traefik                   # 3. 部署依赖此网络的服务
- role: cloudflared               # 4. 部署 Tunnel 容器
```

**关键任务**：

```yaml
- name: Ensure shared Docker network is present
  when: docker_shared_network_create | default(true)
  community.docker.docker_network:
    name: "{{ docker_shared_network_name }}"
    driver: "{{ docker_shared_network_driver }}"
    attachable: "{{ docker_shared_network_attachable }}"
    enable_ipv6: "{{ docker_shared_network_enable_ipv6 }}"
    ipam_config: "{{ docker_shared_network_ipam_config }}"
    labels: "{{ docker_shared_network_labels }}"
```

---

## 对外接口

### 变量定义（defaults/main.yml）

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `docker_shared_network_name` | `proxy_net` | 网络名称（被其他角色引用） |
| `docker_shared_network_subnet` | `"10.203.57.0/24"` | 子网 CIDR |
| `docker_shared_network_gateway` | `"10.203.57.1"` | 网关 IP |
| `docker_shared_network_driver` | `bridge` | 网络驱动类型 |
| `docker_shared_network_attachable` | `true` | 允许容器动态加入 |
| `docker_shared_network_enable_ipv6` | `false` | IPv6 支持 |
| `docker_shared_network_labels` | `{}` | 元数据标签 |
| `docker_shared_network_create` | `true` | 是否创建网络（开关） |

### IPAM 配置

```yaml
docker_shared_network_ipam_config:
  - subnet: "{{ docker_shared_network_subnet }}"
    gateway: "{{ docker_shared_network_gateway }}"
```

### 其他角色引用示例

#### Cloudflared 角色
```yaml
# roles/cloudflared/tasks/main.yml
- name: Build Cloudflared network attachment list
  ansible.builtin.set_fact:
    cloudflared_network_attachments: >-
      {{ ([{'name': docker_shared_network_name}] if docker_shared_network_name is defined else []) +
         (cloudflared_extra_networks | default([])) }}

- name: Run Cloudflared tunnel container
  community.docker.docker_container:
    networks: "{{ cloudflared_network_attachments }}"
    # ...
```

#### Traefik 角色
```yaml
# roles/traefik/defaults/main.yml
traefik_network_name: "{{ docker_shared_network_name }}"

# roles/traefik/tasks/main.yml
- name: 部署 Traefik 容器
  community.docker.docker_container:
    networks:
      - name: "{{ traefik_network_name }}"
    # ...
```

---

## 关键依赖与配置

### 前置依赖

- **角色**: `geerlingguy.docker` 必须在此角色之前执行
- **集合**: `community.docker >= 5.0.2`（提供 `docker_network` 模块）

### 系统要求

- Docker Engine 已安装并运行
- 用户在 `docker` 组（或具有 root 权限）
- Python 模块 `docker` 已安装（通过 playbook pre_tasks 确保）

### 配置覆盖

在 `inventory/group_vars/vps/main.yml` 中全局覆盖：

```yaml
docker_shared_network_name: proxy_net
docker_shared_network_subnet: "10.203.57.0/24"
docker_shared_network_gateway: "10.203.57.1"
docker_shared_network_ipam_config:
  - subnet: "{{ docker_shared_network_subnet }}"
    gateway: "{{ docker_shared_network_gateway }}"
```

---

## 数据模型

### 网络拓扑

```
宿主机
├── docker0 (默认桥接网络, 172.17.0.0/16)
└── proxy_net (自定义桥接网络, 10.203.57.0/24)
    ├── 网关: 10.203.57.1
    ├── cloudflared 容器 (动态 IP)
    ├── traefik 容器 (动态 IP)
    ├── gemi2api 容器 (动态 IP)
    └── cloudflared-whoami 容器 (仅测试环境, 动态 IP)
```

### 网络属性

- **驱动**: bridge（Linux 内核虚拟网桥）
- **隔离性**: 与 `docker0` 网络完全隔离
- **可达性**:
  - 容器间通过容器名互相访问（Docker 内置 DNS）
  - 容器可访问宿主机网关（10.203.57.1）
  - 宿主机可访问容器 IP（通过 iptables NAT）

### 特殊用途：`host.docker.internal`

Cloudflared 通过以下配置访问宿主机服务：

```yaml
# inventory/host_vars/lab-sfo-txy-01/main.yml
cloudflared_etc_hosts:
  host.docker.internal: "{{ docker_shared_network_gateway }}"  # 10.203.57.1

cloudflared_ingress_rules:
  - hostname: "grafana.devopsthink.org"
    service: "http://host.docker.internal:3000"  # 宿主机 Grafana
```

---

## 测试与质量

### Molecule 验证

**场景**: `molecule/default/verify.yml`

```yaml
- name: Inspect shared proxy network
  community.docker.docker_network_info:
    name: "{{ docker_shared_network_name }}"
  register: proxy_network_info

- name: Assert proxy network exists
  ansible.builtin.assert:
    that:
      - proxy_network_info.exists
    fail_msg: "Shared Docker network {{ docker_shared_network_name }} is missing"
```

### 手动验证命令

```bash
# 检查网络是否存在
docker network ls | grep proxy_net

# 查看网络详细信息
docker network inspect proxy_net

# 查看连接的容器
docker network inspect proxy_net --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'

# 测试容器间连通性
docker run --rm --network proxy_net alpine ping -c 3 cloudflared
```

---

## 常见问题 (FAQ)

### Q1: 为什么选择 10.203.57.0/24 子网？

**A**:
- 避免与常见私有网络冲突（如 192.168.0.0/16, 10.0.0.0/8 的前段）
- `/24` 子网提供 254 个可用 IP，足够小型部署
- 可通过 `inventory/group_vars/vps/main.yml` 自定义

### Q2: 如何更改网络名称或子网？

**A**:
1. 编辑 `inventory/group_vars/vps/main.yml`:
   ```yaml
   docker_shared_network_name: my_custom_net
   docker_shared_network_subnet: "10.100.0.0/24"
   docker_shared_network_gateway: "10.100.0.1"
   ```
2. 重新运行 playbook（网络名称变更会创建新网络，旧网络需手动删除）

### Q3: 如何删除网络？

**A**:
```bash
# 1. 停止所有连接的容器
docker stop $(docker ps -q --filter network=proxy_net)

# 2. 删除网络
docker network rm proxy_net

# 3. 重新运行 playbook 重建
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags docker_shared_network
```

### Q4: 网络与防火墙规则冲突怎么办？

**A**: Docker 会自动在 iptables 中添加规则，可能与 UFW 冲突。解决方案：

```yaml
# inventory/host_vars/<hostname>/main.yml
firewall_forward_policy: ACCEPT  # 允许 Docker 网络转发

firewall_additional_rules:
  - "iptables -A INPUT -p tcp -s 10.203.57.0/24 --dport 3000 -j ACCEPT"
```

### Q5: 容器无法通过容器名互访？

**A**: 确认：
1. 容器都连接到 `proxy_net` 网络
2. 使用容器名而非 IP 地址
3. Docker 内置 DNS 正常工作：
   ```bash
   docker exec <container> nslookup <target-container>
   ```

---

## 相关文件清单

```
roles/docker_shared_network/
├── defaults/
│   └── main.yml              # 网络配置变量（13 行）
└── tasks/
    └── main.yml              # 网络创建任务（11 行）

inventory/group_vars/vps/
└── main.yml                  # 全局网络配置覆盖（第 54-60 行）

molecule/default/verify.yml   # 网络存在性断言（第 92-101 行）
```

---

## 变更记录 (Changelog)

### 2025-12-01
- **初始化**: 通过 AI 架构师生成角色文档
- **当前状态**: 生产就绪，已在所有 VPS 主机部署
- **测试覆盖**: Molecule 验证网络创建成功
- **已知限制**: 不支持 IPv6（`enable_ipv6: false`）
