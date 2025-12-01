[根目录](../../CLAUDE.md) > [roles](../) > **traefik**

---

# Traefik Role - 模块文档

## 模块职责

`traefik` 角色负责在目标主机上部署 Traefik v3 反向代理容器，提供自动化 HTTPS、动态路由和服务发现能力。该角色支持：

1. **自动 HTTPS**: 通过 ACME DNS-01 Challenge（Cloudflare）自动获取和续期 Let's Encrypt 证书
2. **动态路由**: 从 Docker 容器标签自动发现服务并配置路由
3. **文件配置**: 支持通过 File Provider 加载静态中间件和路由
4. **监控集成**: 暴露 Prometheus 指标端点
5. **HTTP 压缩**: 可选的 Gzip/Brotli 压缩中间件

**使用场景**：
- VPS 主机需要对外暴露 HTTPS 服务（通过公网 IP）
- 多容器服务共享 80/443 端口
- 自动化证书管理（避免手动维护）

---

## 入口与启动

### 入口文件
- `tasks/main.yml` - 主任务流程（73 行）

### 启动流程

1. **创建目录结构**
   - `/opt/traefik/` - 数据根目录
   - `/opt/traefik/letsencrypt/` - ACME 证书存储
   - `/opt/traefik/dynamic/` - File Provider 配置目录（可选）

2. **初始化 ACME 存储**
   - 创建 `acme.json` 文件（权限 600）

3. **渲染 File Provider 配置**（可选）
   - 使用 `templates/dynamic.yml.j2` 模板
   - 配置静态中间件或路由

4. **部署 Traefik 容器**
   - 拉取镜像（默认 `traefik:v3.6.2`）
   - 挂载 Docker socket（只读）
   - 挂载 ACME 存储和配置目录
   - 发布 80/443 端口
   - 配置健康检查（ping 端点）

5. **部署 Whoami 测试容器**（可选）
   - 用于验证路由和证书配置

### 运行命令示例

```bash
# 在 playbooks/site.yml 中条件性引入
- role: traefik
  when: traefik_enable | default(false)
```

---

## 对外接口

### 关键变量（从 inventory 注入）

#### 必需变量

| 变量名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| `traefik_enable` | Boolean | 是否启用 Traefik | `true` |
| `traefik_acme_email` | String | ACME 邮箱（Let's Encrypt） | `admin@example.com` |
| `traefik_dns_challenge_provider` | String | DNS Challenge Provider | `cloudflare` |
| `CF_API_TOKEN` | String (Env Var) | Cloudflare API Token（通过 `traefik_env_overrides` 注入） | - |

#### 可选变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `traefik_image_tag` | `v3.6.2` | Traefik 镜像版本 |
| `traefik_bind_http_port` | `80` | HTTP 端口 |
| `traefik_bind_https_port` | `443` | HTTPS 端口 |
| `traefik_log_level` | `INFO` | 日志级别 |
| `traefik_metrics_enabled` | `true` | 是否启用 Prometheus 指标 |
| `traefik_compression_enabled` | `false` | 是否启用 HTTP 压缩 |
| `traefik_file_provider_enabled` | `false` | 是否启用 File Provider |
| `traefik_whoami_enabled` | `false` | 是否部署 Whoami 测试容器 |

### 环境变量注入

```yaml
# inventory/host_vars/<hostname>/secrets.yml
traefik_env_overrides:
  CF_API_TOKEN: "{{ cloudflare_api_token }}"  # 从 Vault 读取
```

### Dashboard 与 Whoami 配置

```yaml
traefik_dashboard_domain: "traefik.{{ domain_name }}"
traefik_whoami_domain: "whoami.{{ domain_name }}"
```

---

## 关键依赖与配置

### 角色依赖

- **`docker_shared_network`**: 提供 `proxy_net` 网络
- **`geerlingguy.docker`**: 提供 Docker Engine

### 集合依赖

```yaml
# collections/requirements.yml
- name: community.docker
  version: 5.0.2
```

### 配置文件

#### defaults/main.yml（部分核心变量）

```yaml
traefik_enable: false
traefik_image: traefik
traefik_image_tag: v3.6.2
traefik_container_name: traefik
traefik_network_name: "{{ docker_shared_network_name }}"  # proxy_net
traefik_acme_resolver_name: myresolver
traefik_dns_challenge_provider: cloudflare
traefik_acme_ca_server: https://acme-v02.api.letsencrypt.org/directory
traefik_log_level: INFO
traefik_metrics_enabled: true
traefik_compression_enabled: false
```

#### 命令行标志（traefik_command_flags_defaults）

```yaml
- "--api=true"
- "--api.dashboard=true"
- "--api.insecure=true"  # Dashboard 仅监听内部 8080 端口
- "--providers.docker=true"
- "--providers.docker.exposedbydefault=false"  # 必须显式标记容器
- "--providers.docker.network={{ traefik_network_name }}"
- "--entrypoints.web.address=:80"
- "--entrypoints.web.http.redirections.entryPoint.to=websecure"  # HTTP 自动跳转 HTTPS
- "--entrypoints.websecure.address=:443"
- "--certificatesresolvers.{{ traefik_acme_resolver_name }}.acme.dnschallenge=true"
- "--certificatesresolvers.{{ traefik_acme_resolver_name }}.acme.dnschallenge.provider={{ traefik_dns_challenge_provider }}"
- "--certificatesresolvers.{{ traefik_acme_resolver_name }}.acme.email={{ traefik_acme_email }}"
- "--certificatesresolvers.{{ traefik_acme_resolver_name }}.acme.storage=/letsencrypt/acme.json"
- "--metrics.prometheus=true"
```

#### 容器标签示例（Dashboard 路由）

```yaml
traefik.enable: "true"
traefik.http.routers.traefik-dashboard-secure.rule: 'Host("traefik.devopsthink.org")'
traefik.http.routers.traefik-dashboard-secure.entrypoints: "websecure"
traefik.http.routers.traefik-dashboard-secure.tls.certresolver: "myresolver"
traefik.http.routers.traefik-dashboard-secure.service: "dashboard@internal"
```

---

## 数据模型

### 证书存储结构

```json
// /opt/traefik/letsencrypt/acme.json
{
  "myresolver": {
    "Account": {...},
    "Certificates": [
      {
        "domain": {
          "main": "traefik.devopsthink.org"
        },
        "certificate": "-----BEGIN CERTIFICATE-----\n...",
        "key": "-----BEGIN PRIVATE KEY-----\n...",
        "Store": "default"
      }
    ]
  }
}
```

### File Provider 配置（可选）

```yaml
# templates/dynamic.yml.j2
http:
  middlewares:
    http-compression:
      compress: {}  # Gzip/Brotli 压缩

  routers:
    custom-router:
      rule: "Host(`example.com`)"
      service: custom-service
      middlewares:
        - http-compression
      tls:
        certResolver: myresolver

  services:
    custom-service:
      loadBalancer:
        servers:
          - url: "http://backend:8080"
```

---

## 测试与质量

### Molecule 测试配置

**当前状态**: Traefik 在 Molecule 测试中未启用（需公网 IP 和 DNS 解析）

**潜在测试场景**:
```yaml
# molecule/default/host_vars/debian13/main.yml
traefik_enable: true
traefik_whoami_enabled: true
traefik_acme_ca_server: "https://acme-staging-v02.api.letsencrypt.org/directory"  # 使用 Staging 环境
```

### 手动验证

```bash
# 1. 检查容器状态
docker ps -f name=traefik

# 2. 查看日志
docker logs traefik -f

# 3. 测试 Dashboard（内部端口）
curl -I http://localhost:8080/dashboard/

# 4. 测试路由（需 DNS 解析到宿主机 IP）
curl -I https://traefik.devopsthink.org

# 5. 查看证书
openssl s_client -connect traefik.devopsthink.org:443 -servername traefik.devopsthink.org < /dev/null 2>&1 | openssl x509 -noout -text

# 6. 检查 Prometheus 指标
curl http://localhost:8080/metrics
```

---

## 常见问题 (FAQ)

### Q1: 如何为新服务添加 HTTPS 路由？

**A**: 在服务容器中添加 Docker 标签：

```yaml
# 示例：部署 Nginx 容器
- name: Deploy Nginx with Traefik labels
  community.docker.docker_container:
    name: my-nginx
    image: nginx:alpine
    networks:
      - name: proxy_net
    labels:
      traefik.enable: "true"
      traefik.http.routers.nginx.rule: "Host(`example.com`)"
      traefik.http.routers.nginx.entrypoints: "websecure"
      traefik.http.routers.nginx.tls.certresolver: "myresolver"
      traefik.http.services.nginx.loadbalancer.server.port: "80"
```

### Q2: DNS-01 Challenge 失败怎么办？

**A**: 常见原因：
1. **Cloudflare API Token 权限不足**:
   - 确保 Token 具有 `Zone:DNS:Edit` 权限
   - 验证 Token: `curl -X GET "https://api.cloudflare.com/client/v4/user/tokens/verify" -H "Authorization: Bearer <token>"`

2. **DNS 记录未指向宿主机**:
   - 确认 A/AAAA 记录已创建并生效
   - 测试解析: `dig traefik.devopsthink.org`

3. **网络问题**:
   - 容器无法访问 Cloudflare API（检查防火墙规则）

### Q3: 如何启用 HTTP 压缩？

**A**:
```yaml
# inventory/host_vars/<hostname>/main.yml
traefik_compression_enabled: true
```

Traefik 会自动为所有路由添加 `http-compression` 中间件。

### Q4: 如何查看当前路由和证书？

**A**:
```bash
# 通过 API 查看路由
curl http://localhost:8080/api/http/routers | jq

# 通过 API 查看证书
curl http://localhost:8080/api/http/routers/<router-name> | jq '.tls.certificates'

# 或访问 Dashboard（需通过域名访问）
https://traefik.devopsthink.org/dashboard/
```

### Q5: 如何切换到 Let's Encrypt Staging 环境？

**A**:
```yaml
# inventory/host_vars/<hostname>/main.yml
traefik_acme_ca_server: "https://acme-staging-v02.api.letsencrypt.org/directory"
```

Staging 环境不受速率限制，适合测试。生产环境切换回默认值即可。

### Q6: 为什么 Dashboard 使用 `--api.insecure=true`？

**A**:
- `insecure` 模式允许 Dashboard 监听内部 8080 端口（不需要 TLS）
- 外部访问通过容器标签定义的路由（走 443 端口，自动 HTTPS）
- 8080 端口未发布到宿主机，仅容器内部可访问

---

## 相关文件清单

```
roles/traefik/
├── defaults/
│   └── main.yml              # 默认变量定义（136 行）
├── tasks/
│   └── main.yml              # 主任务流程（73 行）
└── templates/
    └── dynamic.yml.j2        # File Provider 配置模板（8 行）

inventory/host_vars/<hostname>/
├── main.yml                  # 启用标志与域名配置
└── secrets.yml               # Cloudflare API Token（Vault 加密）
```

---

## 变更记录 (Changelog)

### 2025-12-01
- **初始化**: 通过 AI 架构师生成完整的角色文档
- **当前状态**: 已实现但未在生产启用（优先使用 Cloudflare Tunnel）
- **测试覆盖**: 无 Molecule 测试（需公网环境）
- **已知限制**: 需要公网 IP 和 DNS 解析，不适用于 NAT 后的 VPS
