[根目录](../../CLAUDE.md) > [roles](../) > **searxng**

---

# SearXNG Role - 模块文档

## 模块职责

`searxng` 角色负责在目标主机上部署 SearXNG 搜索聚合器容器。SearXNG 是一个开源的隐私友好元搜索引擎，聚合来自多个搜索引擎的结果，不追踪用户，不分析用户行为。

**核心功能**：
1. 部署 SearXNG 容器
2. 渲染自定义配置文件（settings.yml）
3. 通过 Traefik 提供 HTTPS 访问（无需暴露端口）
4. 可选启用 Prometheus 指标
5. 配置持久化（仅配置文件）

**使用场景**：
- 私有化搜索引擎部署
- 隐私保护的搜索服务
- 聚合多个搜索引擎结果
- 企业内部搜索工具

---

## 入口与启动

### 入口文件
- `tasks/main.yml` - 主任务流程

### 启动流程

1. **创建配置目录**
   - `/opt/searxng/config/` - 配置文件存储

2. **渲染配置文件**
   - 使用 `templates/settings.yml.j2` 模板
   - 配置实例名称、密钥、搜索引擎等
   - 注册配置变更（触发容器重建）

3. **部署 SearXNG 容器**
   - 拉取 `docker.io/searxng/searxng` 镜像
   - 挂载配置目录（只读）
   - 仅加入 `proxy_net` 网络（不暴露端口）
   - 添加 Traefik 路由标签

4. **Traefik 自动配置**
   - 通过 Docker Provider 自动发现服务
   - 自动申请 Let's Encrypt 证书
   - 配置 HTTPS 路由

### 运行命令示例

```bash
# 在 playbooks/site.yml 中引入
- role: searxng
  when: searxng_enable | default(false)
```

---

## 对外接口

### 关键变量（从 inventory 注入）

#### 必需变量

| 变量名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| `searxng_enable` | Boolean | 是否启用服务 | `true` |
| `searxng_secret_key` | String | 加密密钥（用于会话加密） | `openssl rand -hex 32` |

#### 可选变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `searxng_image_tag` | `latest` | SearXNG 镜像版本 | `2025.8.1-3d96414` |
| `searxng_domain` | `searxng.{{ domain_name }}` | 访问域名 |
| `searxng_instance_name` | `SearXNG` | 实例名称 |
| `searxng_contact_url` | `""` | 联系方式 URL |
| `searxng_enable_metrics` | `true` | 是否启用 Prometheus 指标 |
| `searxng_autocomplete` | `google` | 自动补全引擎 |
| `searxng_traefik_enabled` | `true` | 是否启用 Traefik 路由 |

### 环境变量注入

```yaml
# inventory/host_vars/lab-lax-rnd-02/secrets.yml (Vault 加密)
searxng_secret_key: "<GENERATED_SECRET_KEY>"  # openssl rand -hex 32
```

### 主机配置

```yaml
# inventory/host_vars/lab-lax-rnd-02/main.yml
searxng_enable: true
searxng_image_tag: 2025.8.1-3d96414
searxng_domain: "searxng.{{ domain_name }}"
searxng_instance_name: "DevOpsThink Search"
searxng_contact_url: "mailto:admin@{{ domain_name }}"
```

---

## 关键依赖与配置

### 角色依赖

- **`docker_custom`**: 提供 `proxy_net` 网络
- **`traefik`**: 提供 HTTPS 反向代理
- **`geerlingguy.docker`**: 提供 Docker Engine

### 集合依赖

```yaml
# collections/requirements.yml
- name: community.docker
  version: 5.0.2
```

### 配置文件

#### defaults/main.yml（核心变量）

```yaml
searxng_enable: false
searxng_data_directory: /opt/searxng
searxng_config_directory: "{{ searxng_data_directory }}/config"
searxng_network_name: "{{ docker_custom_network_name }}"

# 容器配置
searxng_image: docker.io/searxng/searxng
searxng_image_tag: latest
searxng_container_name: searxng
searxng_container_port: 8080

# 应用配置
searxng_domain: "searxng.{{ domain_name }}"
searxng_base_url: "https://{{ searxng_domain }}"
searxng_secret_key: ""
searxng_instance_name: "SearXNG"
searxng_enable_metrics: true

# Traefik 集成
searxng_traefik_enabled: true
```

#### Traefik 标签（自动配置）

```yaml
traefik.enable: "true"
traefik.docker.network: "proxy_net"
traefik.http.routers.searxng.rule: 'Host("searxng.devopsthink.org")'
traefik.http.routers.searxng.entrypoints: "websecure"
traefik.http.routers.searxng.tls.certresolver: "myresolver"
traefik.http.services.searxng.loadbalancer.server.port: "8080"
```

---

## 数据模型

### 配置文件结构

```yaml
# /opt/searxng/config/settings.yml
general:
  instance_name: "DevOpsThink Search"
  contact_url: "mailto:admin@devopsthink.org"
  enable_metrics: true

server:
  secret_key: "<SECRET_KEY>"
  base_url: "https://searxng.devopsthink.org"
  bind_address: "0.0.0.0"
  port: 8080

search:
  autocomplete: "google"
  default_lang: "auto"

ui:
  default_theme: simple
  theme_args:
    simple_style: auto

engines:
  - name: google
    disabled: false
  - name: bing
    disabled: false
  - name: duckduckgo
    disabled: false
  - name: wikipedia
    disabled: false
  - name: github
    disabled: false
```

### 目录结构

```
/opt/searxng/
└── config/
    └── settings.yml        # 配置文件（只读挂载）
```

### 容器网络架构

```
┌─────────────────────┐
│   Traefik          │ :443 (HTTPS)
│   (反向代理)       │
└──────┬──────────────┘
       │ proxy_net
       │
┌──────▼──────┐
│  SearXNG    │
│  :8080      │
│  (搜索引擎) │
└─────────────┘
```

---

## 测试与质量

### Molecule 测试配置

**当前状态**: 未在 Molecule 测试中启用

**潜在测试场景**:
```yaml
# molecule/default/host_vars/debian13/main.yml
searxng_enable: true
searxng_domain: "searxng.test.local"
searxng_secret_key: "test_secret_key_32_characters_long_minimum"
```

### 手动验证

```bash
# 1. 检查容器状态
docker ps -f name=searxng

# 2. 查看日志
docker logs searxng -f

# 3. 测试 Web 界面（通过域名）
curl -I https://searxng.devopsthink.org

# 4. 测试搜索功能
curl "https://searxng.devopsthink.org/search?q=ansible&format=json"

# 5. 检查 Traefik 路由
curl http://localhost:8080/api/http/routers | jq '.[] | select(.name=="searxng@docker")'

# 6. 测试 Prometheus 指标（如果启用）
curl https://searxng.devopsthink.org/stats

# 7. 检查配置文件
cat /opt/searxng/config/settings.yml

# 8. 检查网络
docker network inspect proxy_net | grep -A 3 searxng
```

---

## 常见问题 (FAQ)

### Q1: 如何生成 secret_key？

**A**: 使用 openssl 生成随机密钥：

```bash
openssl rand -hex 32
# 输出示例: 3c0f7e95d8a9b6e4f2a1c8d7e9b4f6a3e2c1d9b8a7f6e5d4c3b2a1f0e9d8c7b6
```

然后在 secrets.yml 中配置：
```bash
ansible-vault edit inventory/host_vars/lab-lax-rnd-02/secrets.yml
```

添加：
```yaml
searxng_secret_key: "3c0f7e95d8a9b6e4f2a1c8d7e9b4f6a3e2c1d9b8a7f6e5d4c3b2a1f0e9d8c7b6"
```

### Q2: 如何添加自定义搜索引擎？

**A**: 编辑配置模板 `roles/searxng/templates/settings.yml.j2`：

```yaml
engines:
  - name: google
    disabled: false
  - name: bing
    disabled: false
  - name: stackoverflow
    disabled: false  # 新增
  - name: reddit
    disabled: false  # 新增
```

重新部署：
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --limit lab-lax-rnd-02
```

### Q3: 为什么不暴露端口？

**A**:
- **安全性**: 通过 Traefik 统一管理 HTTPS 和认证
- **证书管理**: Traefik 自动申请和续期 Let's Encrypt 证书
- **灵活性**: 可在不修改容器的情况下调整路由规则

如果需要临时测试，可以添加端口映射：
```yaml
# inventory/host_vars/lab-lax-rnd-02/main.yml
searxng_env_overrides:
  published_ports:
    - "127.0.0.1:8080:8080"  # 仅本地访问
```

### Q4: 如何查看搜索统计？

**A**:

访问 `/stats` 端点（需启用 `searxng_enable_metrics: true`）：
```bash
curl https://searxng.devopsthink.org/stats
```

或在 Web 界面中访问：
```
https://searxng.devopsthink.org/stats
```

### Q5: 如何升级 SearXNG 版本？

**A**:

```yaml
# inventory/host_vars/lab-lax-rnd-02/main.yml
searxng_image_tag: 2025.9.1-abcd1234  # 修改为新版本

# 重新部署
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --limit lab-lax-rnd-02
```

### Q6: 为什么不持久化缓存数据？

**A**:
- 遵循用户要求（仅持久化配置）
- 缓存数据可重新生成，不影响功能
- 减少磁盘使用和备份负担
- 容器重启后自动重建缓存

如果需要持久化缓存，可以修改 `tasks/main.yml` 添加卷挂载：
```yaml
volumes:
  - "{{ searxng_config_directory }}:/etc/searxng:ro"
  - "{{ searxng_data_directory }}/cache:/var/cache/searxng:rw"  # 新增
```

### Q7: 如何禁用特定搜索引擎？

**A**: 编辑配置模板：

```yaml
engines:
  - name: google
    disabled: true  # 禁用 Google
  - name: bing
    disabled: false
```

或者完全移除：
```yaml
engines:
  - name: bing
    disabled: false
  - name: duckduckgo
    disabled: false
  # 不包含 google
```

### Q8: 如何更改 UI 主题？

**A**: 当前使用 `simple` 主题，可在配置模板中更改：

```yaml
ui:
  default_theme: oscar  # 可选: simple, oscar, pix-art 等
  theme_args:
    oscar_style: logicodev  # Oscar 主题样式
```

完整主题列表参考 [SearXNG 文档](https://docs.searxng.org/admin/settings/settings_ui.html)

---

## 相关文件清单

```
roles/searxng/
├── defaults/
│   └── main.yml              # 默认变量定义
├── tasks/
│   └── main.yml              # 主任务流程
├── templates/
│   └── settings.yml.j2       # 配置模板
└── CLAUDE.md                 # 本文档

inventory/host_vars/lab-lax-rnd-02/
├── main.yml                  # 启用标志与域名配置
└── secrets.yml               # 密钥（Vault 加密）
```

---

## 变更记录 (Changelog)

### 2025-12-02
- **初始化**: 创建 searxng 角色
- **版本**: 2025.8.1-3d96414
- **特性**:
  - 自定义配置模板（Jinja2）
  - Traefik HTTPS 路由（Docker Provider）
  - 配置持久化（不持久化缓存）
  - Prometheus 指标支持
- **测试覆盖**: 未集成到 Molecule 测试
- **部署主机**: lab-lax-rnd-02

---

## 安全建议

1. **密钥管理**:
   - 必须通过 Vault 加密存储 `searxng_secret_key`
   - 使用强随机密钥（至少 32 字节）
   - 定期轮换密钥

2. **网络安全**:
   - 不暴露容器端口到宿主机
   - 仅通过 Traefik HTTPS 访问
   - 配置文件只读挂载（`:ro`）

3. **隐私保护**:
   - SearXNG 默认不记录搜索日志
   - 不追踪用户 IP 和搜索历史
   - 适合作为隐私友好的搜索工具

4. **监控**:
   - 监控容器健康状态
   - 启用 Prometheus 指标
   - 监控磁盘使用量

---

## 参考资源

- [SearXNG 官方文档](https://docs.searxng.org/)
- [SearXNG GitHub](https://github.com/searxng/searxng)
- [Docker 安装文档](https://docs.searxng.org/admin/installation-docker.html)
- [配置参考](https://docs.searxng.org/admin/settings/index.html)
- [支持的搜索引擎列表](https://docs.searxng.org/admin/engines/index.html)
