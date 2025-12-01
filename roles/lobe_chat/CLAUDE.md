[根目录](../../CLAUDE.md) > [roles](../) > **lobe_chat**

---

# LobeChat Role - 模块文档

## 模块职责

`lobe_chat` 角色负责在目标主机上部署 LobeChat 聊天应用及其依赖的 PostgreSQL 数据库。LobeChat 是一个开源的 AI 聊天界面,支持多种 AI 模型、知识库管理和 Clerk 身份认证。

**核心功能**:
1. 部署 PostgreSQL (pgvector) 数据库容器
2. 部署 LobeChat 主应用容器
3. 配置 S3 对象存储（文件上传）
4. 集成 Clerk 身份认证
5. 通过 Traefik 提供 HTTPS 访问（无需暴露端口）
6. 数据持久化（PostgreSQL 数据卷）

**使用场景**:
- 私有化部署 AI 聊天界面
- 多用户知识库管理
- 企业内部 AI 助手平台

---

## 入口与启动

### 入口文件
- `tasks/main.yml` - 主任务流程

### 启动流程

1. **创建数据目录**
   - `/opt/lobe_chat/` - 根目录
   - `/opt/lobe_chat/data/` - PostgreSQL 数据存储

2. **部署 PostgreSQL 容器**
   - 拉取 `pgvector/pgvector:0.8.1-pg16-trixie` 镜像
   - 配置环境变量（数据库名、用户、密码）
   - 挂载数据卷（持久化存储）
   - 健康检查（pg_isready）

3. **等待数据库就绪**
   - 使用 `docker_container_exec` 模块检查数据库状态
   - 最多重试 10 次，间隔 3 秒

4. **部署 LobeChat 容器**
   - 拉取 `bernylinville/lobe-chat-db-clerk:v1.143.0` 镜像
   - 配置环境变量（数据库连接、S3、Clerk 等）
   - 仅加入 `proxy_net` 网络（不暴露端口）
   - 添加 Traefik 路由标签

### 运行命令示例

```bash
# 在 playbooks/site.yml 中引入
- role: lobe_chat
  when: lobe_chat_enable | default(false)
```

---

## 对外接口

### 关键变量（从 inventory 注入）

#### 必需变量

| 变量名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| `lobe_chat_enable` | Boolean | 是否启用服务 | `true` |
| `lobe_chat_postgres_password` | String | PostgreSQL 密码 | `<random_password>` |
| `lobe_chat_key_vaults_secret` | String | 敏感信息加密密钥 | `openssl rand -base64 32` |
| `lobe_chat_s3_access_key_id` | String | S3 访问密钥 | `<cloudflare_r2_key>` |
| `lobe_chat_s3_secret_access_key` | String | S3 密钥 | `<cloudflare_r2_secret>` |
| `lobe_chat_s3_endpoint` | String | S3 存储桶端点 | `https://xxx.r2.cloudflarestorage.com` |
| `lobe_chat_s3_public_domain` | String | S3 公共访问域名 | `https://s3.example.com` |
| `lobe_chat_clerk_publishable_key` | String | Clerk 公钥 | `pk_live_xxx` |
| `lobe_chat_clerk_secret_key` | String | Clerk 私钥 | `sk_live_xxx` |
| `lobe_chat_clerk_webhook_secret` | String | Clerk Webhook 密钥 | `whsec_xxx` |

#### 可选变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `lobe_chat_image_tag` | `v1.143.0` | LobeChat 镜像版本 |
| `lobe_chat_domain` | `lobe.{{ domain_name }}` | 访问域名 |
| `lobe_chat_postgres_db` | `lobe` | 数据库名 |
| `lobe_chat_postgres_user` | `lobe` | 数据库用户 |
| `lobe_chat_s3_bucket` | `lobechat` | S3 存储桶名称 |
| `lobe_chat_s3_enable_path_style` | `"0"` | S3 路径样式（MinIO 需设为 `"1"`） |
| `lobe_chat_traefik_enabled` | `true` | 是否启用 Traefik 路由 |

### 环境变量注入

```yaml
# inventory/host_vars/lab-lax-rnd-02/secrets.yml (Vault 加密)
lobe_chat_postgres_password: "uWNZugjBqixf8dxC"
lobe_chat_key_vaults_secret: "Kix2wcUONd4CX51E/ZPAd36BqM4wzJgKjPtz2sGztqQ="
lobe_chat_s3_access_key_id: "9998d6757e276cf9f1edbd325b7083a6"
lobe_chat_s3_secret_access_key: "55af75d8eb6b99f189f6a35f855336ea62cd9c4751a5cf4337c53c1d3f497ac2"
lobe_chat_s3_endpoint: "https://0b33a03b5c993fd2f453379dc36558e5.r2.cloudflarestorage.com"
lobe_chat_s3_public_domain: "https://s3-dev.your-domain.com"
lobe_chat_clerk_publishable_key: "pk_live_xxxxxxxxxxx"
lobe_chat_clerk_secret_key: "sk_live_xxxxxxxxxxxxxxxxxxxxxx"
lobe_chat_clerk_webhook_secret: "whsec_xxxxxxxxxxxxxxxxxxxxxx"
```

### 主机配置

```yaml
# inventory/host_vars/lab-lax-rnd-02/main.yml
lobe_chat_enable: true
lobe_chat_domain: "lobe.devopsthink.org"
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
lobe_chat_enable: false
lobe_chat_data_directory: /opt/lobe_chat
lobe_chat_postgres_data_directory: "{{ lobe_chat_data_directory }}/data"
lobe_chat_network_name: "{{ docker_custom_network_name }}"

# PostgreSQL 配置
lobe_chat_postgres_image: pgvector/pgvector
lobe_chat_postgres_image_tag: 0.8.1-pg16-trixie
lobe_chat_postgres_container_name: lobe-postgres
lobe_chat_postgres_db: lobe
lobe_chat_postgres_user: lobe

# LobeChat 配置
lobe_chat_image: bernylinville/lobe-chat-db-clerk
lobe_chat_image_tag: v1.143.0
lobe_chat_container_name: lobe-chat
lobe_chat_container_port: 3210

# Traefik 集成
lobe_chat_domain: "lobe.{{ domain_name }}"
lobe_chat_traefik_enabled: true
```

#### Traefik 标签（自动配置）

```yaml
traefik.enable: "true"
traefik.docker.network: "proxy_net"
traefik.http.routers.lobe-chat.rule: 'Host("lobe.devopsthink.org")'
traefik.http.routers.lobe-chat.entrypoints: "websecure"
traefik.http.routers.lobe-chat.tls.certresolver: "myresolver"
traefik.http.services.lobe-chat.loadbalancer.server.port: "3210"
```

---

## 数据模型

### 数据库连接字符串

```yaml
# 自动构建的 DATABASE_URL
postgresql://<user>:<password>@lobe-postgres:5432/<dbname>
# 示例
postgresql://lobe:uWNZugjBqixf8dxC@lobe-postgres:5432/lobe
```

### 目录结构

```
/opt/lobe_chat/
├── data/                    # PostgreSQL 数据卷
│   ├── base/               # 数据库文件
│   ├── global/
│   ├── pg_wal/             # WAL 日志
│   └── ...
```

### 容器网络架构

```
┌─────────────────────┐
│   Traefik          │ :443 (HTTPS)
│   (反向代理)       │
└──────┬──────────────┘
       │ proxy_net
       │
       ├─────────────────┐
       │                 │
┌──────▼──────┐   ┌──────▼──────────┐
│ LobeChat    │   │  lobe-postgres  │
│ :3210       │◄──┤  :5432          │
│ (Web 界面)  │   │  (数据库)       │
└─────────────┘   └─────────────────┘
```

---

## 测试与质量

### Molecule 测试配置

**当前状态**: 未在 Molecule 测试中启用（需配置 S3 和 Clerk 凭据）

**潜在测试场景**:
```yaml
# molecule/default/host_vars/debian13/main.yml
lobe_chat_enable: true
lobe_chat_domain: "lobe.test.local"

# 使用模拟凭据
lobe_chat_postgres_password: "test_password"
lobe_chat_key_vaults_secret: "test_secret"
lobe_chat_clerk_publishable_key: "pk_test_xxx"
lobe_chat_clerk_secret_key: "sk_test_xxx"
```

### 手动验证

```bash
# 1. 检查容器状态
docker ps -f name=lobe

# 2. 查看日志
docker logs lobe-chat -f
docker logs lobe-postgres -f

# 3. 测试数据库连接
docker exec -it lobe-postgres psql -U lobe -d lobe -c "\dt"

# 4. 测试 Web 界面（通过域名）
curl -I https://lobe.devopsthink.org

# 5. 检查 Traefik 路由
curl http://localhost:8080/api/http/routers | jq '.[] | select(.name=="lobe-chat")'

# 6. 检查数据持久化
ls -lh /opt/lobe_chat/data/
```

---

## 常见问题 (FAQ)

### Q1: 如何初始化管理员账户？

**A**: LobeChat 使用 Clerk 身份认证，需要在 Clerk Dashboard 中配置:

1. 访问 https://clerk.com/ 创建应用
2. 获取 API 密钥（publishable key, secret key）
3. 配置 Webhook URL: `https://lobe.devopsthink.org/api/webhook/clerk`
4. 在 Clerk Dashboard 添加用户

### Q2: 如何配置 S3 存储？

**A**: 推荐使用 Cloudflare R2（兼容 S3 协议）:

1. 创建 R2 存储桶（如 `lobechat`）
2. 创建 API Token（读写权限）
3. 配置公共域名（用于访问上传的文件）

```yaml
# inventory/host_vars/<hostname>/secrets.yml
lobe_chat_s3_access_key_id: "<R2_ACCESS_KEY>"
lobe_chat_s3_secret_access_key: "<R2_SECRET_KEY>"
lobe_chat_s3_endpoint: "https://<account_id>.r2.cloudflarestorage.com"
lobe_chat_s3_public_domain: "https://s3.example.com"
```

### Q3: 如何迁移现有数据？

**A**:

1. **备份旧数据库**:
   ```bash
   docker exec lobe-postgres pg_dump -U lobe -d lobe > backup.sql
   ```

2. **停止容器**:
   ```bash
   ansible-playbook -i inventory/hosts.yml playbooks/site.yml -e "lobe_chat_enable=false"
   ```

3. **恢复数据**:
   ```bash
   docker exec -i lobe-postgres psql -U lobe -d lobe < backup.sql
   ```

4. **重启服务**:
   ```bash
   ansible-playbook -i inventory/hosts.yml playbooks/site.yml
   ```

### Q4: 数据库密码忘记了怎么办？

**A**:

```bash
# 1. 查看加密的密码（需要 Vault 密码）
ansible-vault view inventory/host_vars/lab-lax-rnd-02/secrets.yml

# 2. 或者重置数据库密码（会导致 LobeChat 连接失败）
docker exec -it lobe-postgres psql -U postgres -c "ALTER USER lobe WITH PASSWORD 'new_password';"

# 3. 更新 secrets.yml 并重新部署
ansible-vault edit inventory/host_vars/lab-lax-rnd-02/secrets.yml
```

### Q5: 如何升级 LobeChat 版本？

**A**:

```yaml
# inventory/host_vars/lab-lax-rnd-02/main.yml
lobe_chat_image_tag: v1.150.0  # 修改为新版本

# 重新部署
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags lobe_chat
```

### Q6: 为什么不直接暴露端口？

**A**:
- **安全性**: 通过 Traefik 统一管理 HTTPS 和认证
- **证书管理**: Traefik 自动申请和续期 Let's Encrypt 证书
- **灵活性**: 可在不修改容器的情况下调整路由规则

如果需要临时测试,可以添加端口映射:
```yaml
# inventory/host_vars/lab-lax-rnd-02/main.yml
lobe_chat_env_overrides:
  published_ports:
    - "127.0.0.1:3210:3210"  # 仅本地访问
```

### Q7: 如何备份数据？

**A**:

```bash
# 方法 1: 数据库逻辑备份
docker exec lobe-postgres pg_dump -U lobe -d lobe -F c -f /tmp/lobe_backup.dump
docker cp lobe-postgres:/tmp/lobe_backup.dump ./lobe_backup_$(date +%Y%m%d).dump

# 方法 2: 数据卷物理备份（需停止容器）
tar -czf lobe_data_$(date +%Y%m%d).tar.gz -C /opt/lobe_chat data/

# 方法 3: 使用 Ansible 自动化备份
- name: 备份 LobeChat 数据库
  community.docker.docker_container_exec:
    container: lobe-postgres
    command: pg_dump -U lobe -d lobe -F c -f /tmp/backup.dump
  register: backup_result
```

---

## 相关文件清单

```
roles/lobe_chat/
├── defaults/
│   └── main.yml              # 默认变量定义
├── tasks/
│   └── main.yml              # 主任务流程
└── CLAUDE.md                 # 本文档

inventory/host_vars/lab-lax-rnd-02/
├── main.yml                  # 启用标志与域名配置
└── secrets.yml               # 敏感凭据（Vault 加密）
```

---

## 变更记录 (Changelog)

### 2025-12-01
- **初始化**: 创建 lobe_chat 角色
- **版本**: v1.143.0
- **特性**:
  - PostgreSQL (pgvector) 数据库
  - S3 对象存储集成
  - Clerk 身份认证
  - Traefik HTTPS 路由
  - 数据持久化
- **测试覆盖**: 未集成到 Molecule 测试

---

## 安全建议

1. **敏感数据管理**:
   - 所有密钥必须通过 Vault 加密
   - 定期轮换数据库密码和 API 密钥
   - 使用强随机密码（`openssl rand -base64 32`）

2. **网络安全**:
   - 不要暴露数据库端口到宿主机
   - 仅通过 Traefik HTTPS 访问
   - 配置 Clerk 限制访问（域名白名单）

3. **备份策略**:
   - 每日自动备份数据库
   - 备份文件异地存储（S3/R2）
   - 定期测试恢复流程

4. **监控**:
   - 监控容器健康状态
   - 监控数据库连接数
   - 监控磁盘使用量（`/opt/lobe_chat/data`）

---

## 参考资源

- [LobeChat 官方文档](https://lobehub.com/docs)
- [LobeChat GitHub](https://github.com/lobehub/lobe-chat)
- [pgvector 文档](https://github.com/pgvector/pgvector)
- [Clerk 文档](https://clerk.com/docs)
- [Cloudflare R2 文档](https://developers.cloudflare.com/r2/)
