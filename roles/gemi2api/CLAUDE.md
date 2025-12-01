[根目录](../../CLAUDE.md) > [roles](../) > **gemi2api**

---

# Gemi2api Role - 模块文档

## 模块职责

`gemi2api` 角色负责在目标主机上部署 Gemini to OpenAI API 转换服务容器。该服务将 Google Gemini API 封装为 OpenAI 兼容接口，便于在现有工具链中无缝集成 Gemini 模型。

**核心功能**：
1. 部署 `bernylinville/gemi2api` 容器
2. 配置 Gemini 认证凭据（SECURE_1PSID, SECURE_1PSIDTS）
3. 绑定到 Tailscale IP（仅内网访问）或本地回环
4. 可选启用思维链（Thinking）模式

**使用场景**：
- 在 OpenAI 客户端中使用 Gemini 模型
- 内网 AI 服务（通过 Tailscale 访问）
- 本地开发与测试

---

## 入口与启动

### 入口文件
- `tasks/main.yml` - 主任务流程（24 行）

### 启动流程

1. **部署容器**（当 `gemi2api_enable: true` 时）
   - 拉取镜像（默认 `bernylinville/gemi2api:latest`）
   - 配置环境变量（认证凭据、API 密钥、功能开关）
   - 绑定到指定地址和端口（默认 `127.0.0.1:8000`）
   - 加入共享 Docker 网络（`proxy_net`）

2. **清理容器**（当 `gemi2api_enable: false` 时）
   - 强制停止并删除容器

### 运行命令示例

```bash
# 在 playbooks/site.yml 中条件性引入
- role: gemi2api
  when: gemi2api_enable | default(false)
```

---

## 对外接口

### 关键变量（从 inventory 注入）

#### 必需变量

| 变量名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| `gemi2api_enable` | Boolean | 是否启用服务 | `true` |
| `SECURE_1PSID` | String (Env Var) | Gemini 认证 Cookie | `<cookie_value>` |
| `SECURE_1PSIDTS` | String (Env Var) | Gemini 时间戳 Cookie | `<timestamp>` |
| `API_KEY` | String (Env Var) | API 访问密钥 | `sk-custom-key` |

#### 可选变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `gemi2api_image_tag` | `latest` | 镜像版本标签 | `v1.17.1` |
| `gemi2api_bind_address` | `127.0.0.1` | 绑定地址（可改为 Tailscale IP） | `100.122.29.40` |
| `gemi2api_bind_port` | `8000` | 绑定端口 | `8000` |
| `gemi2api_container_port` | `8000` | 容器内部端口 | `8000` |
| `gemi2api_pull` | `true` | 是否每次拉取最新镜像 | `false` |
| `gemi2api_restart_policy` | `unless-stopped` | 重启策略 | `always` |
| `ENABLE_THINKING` | `"true"` | 是否启用思维链模式 | `"false"` |

### 环境变量注入

```yaml
# inventory/host_vars/<hostname>/secrets.yml (Vault 加密)
gemi2api_env_overrides:
  SECURE_1PSID: "{{ gemini_1psid }}"
  SECURE_1PSIDTS: "{{ gemini_1psidts }}"
  API_KEY: "{{ gemini_api_key }}"
```

### 网络配置

```yaml
# inventory/host_vars/lab-sfo-txy-01/main.yml
gemi2api_enable: true
gemi2api_image_tag: v1.17.1
gemi2api_bind_address: "{{ tailscale_ip }}"  # 绑定到 Tailscale 内网 IP
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

#### defaults/main.yml

```yaml
gemi2api_enable: false
gemi2api_image: bernylinville/gemi2api
gemi2api_image_tag: latest
gemi2api_container_name: gemi2api
gemi2api_bind_address: 127.0.0.1
gemi2api_bind_port: 8000
gemi2api_container_port: 8000
gemi2api_network_name: "{{ docker_shared_network_name }}"  # proxy_net
gemi2api_pull: true
gemi2api_restart_policy: unless-stopped
gemi2api_env_defaults:
  SECURE_1PSID: ""
  SECURE_1PSIDTS: ""
  API_KEY: ""
  ENABLE_THINKING: "true"
gemi2api_env_overrides: {}
gemi2api_labels: {}
gemi2api_state: started
```

---

## 数据模型

### API 请求示例

```bash
# OpenAI 兼容接口
curl http://100.122.29.40:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-custom-key" \
  -d '{
    "model": "gemini-2.0-flash-thinking-exp-1219",
    "messages": [
      {"role": "user", "content": "解释量子纠缠"}
    ],
    "stream": false
  }'
```

### 环境变量配置

```yaml
# 运行时合并逻辑
gemi2api_env_final: "{{ gemi2api_env_defaults | combine(gemi2api_env_overrides, recursive=True) }}"
```

### 支持的模型列表

根据 gemi2api 项目文档（需确认），常见模型包括：
- `gemini-2.0-flash-thinking-exp-1219` (思维链版本)
- `gemini-2.0-flash-exp`
- `gemini-1.5-pro-exp`
- `gemini-1.5-flash-8b`

---

## 测试与质量

### Molecule 测试配置

**当前状态**: 未在 Molecule 测试中启用（需有效 Gemini 凭据）

**潜在测试场景**:
```yaml
# molecule/default/host_vars/debian13/main.yml
gemi2api_enable: true
gemi2api_bind_address: "0.0.0.0"  # 允许测试容器访问

# molecule/default/host_vars/debian13/secrets.yml (Vault 加密)
gemi2api_env_overrides:
  SECURE_1PSID: "test_cookie"
  SECURE_1PSIDTS: "test_timestamp"
  API_KEY: "test-api-key"
```

### 手动验证

```bash
# 1. 检查容器状态
docker ps -f name=gemi2api

# 2. 查看日志
docker logs gemi2api -f

# 3. 测试健康检查（如果服务提供）
curl http://100.122.29.40:8000/health

# 4. 测试 API 端点
curl http://100.122.29.40:8000/v1/models \
  -H "Authorization: Bearer sk-custom-key"

# 5. 测试 Chat Completions
curl http://100.122.29.40:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-custom-key" \
  -d '{
    "model": "gemini-2.0-flash-exp",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

---

## 常见问题 (FAQ)

### Q1: 如何获取 SECURE_1PSID 和 SECURE_1PSIDTS？

**A**:
1. 登录 Google Gemini 网页版（https://gemini.google.com/）
2. 打开浏览器开发者工具（F12）-> 应用程序 -> Cookies
3. 查找并复制以下 Cookie 值：
   - `__Secure-1PSID`
   - `__Secure-1PSIDTS`

**注意**: 这些 Cookie 有有效期，过期后需重新获取。

### Q2: 为什么绑定到 Tailscale IP？

**A**:
- **安全性**: 避免在公网暴露 API（无需端口转发）
- **便捷性**: 通过 Tailscale 内网在任何设备访问
- **灵活性**: 可改为 `0.0.0.0` 在本地测试

```yaml
# 仅本地访问
gemi2api_bind_address: "127.0.0.1"

# Tailscale 内网访问
gemi2api_bind_address: "{{ tailscale_ip }}"

# 公网访问（需配置防火墙）
gemi2api_bind_address: "0.0.0.0"
```

### Q3: 如何在 OpenAI 客户端中使用？

**A**:

**Python 示例**:
```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-custom-key",
    base_url="http://100.122.29.40:8000/v1"
)

response = client.chat.completions.create(
    model="gemini-2.0-flash-exp",
    messages=[{"role": "user", "content": "你好"}]
)
print(response.choices[0].message.content)
```

**Cursor IDE 配置**:
```json
{
  "openaiApiKey": "sk-custom-key",
  "openaiBaseUrl": "http://100.122.29.40:8000/v1"
}
```

### Q4: 思维链模式（ENABLE_THINKING）有什么用？

**A**:
- 启用后，模型会输出中间推理步骤
- 适合需要详细解释的复杂问题
- 可能增加响应时间和 Token 消耗

```yaml
# 禁用思维链（更快响应）
gemi2api_env_overrides:
  ENABLE_THINKING: "false"
```

### Q5: 如何更新服务版本？

**A**:
```yaml
# inventory/host_vars/<hostname>/main.yml
gemi2api_image_tag: v1.18.0  # 修改版本号

# 重新运行 playbook
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags gemi2api
```

### Q6: 服务无响应怎么办？

**A**:
1. **检查容器日志**:
   ```bash
   docker logs gemi2api --tail 50
   ```

2. **常见错误**:
   - `401 Unauthorized`: Cookie 过期，需重新获取
   - `Connection refused`: 容器未启动或端口配置错误
   - `Network error`: 容器网络配置问题

3. **重启容器**:
   ```bash
   docker restart gemi2api
   ```

---

## 相关文件清单

```
roles/gemi2api/
├── defaults/
│   └── main.yml              # 默认变量定义（20 行）
└── tasks/
    └── main.yml              # 主任务流程（24 行）

inventory/host_vars/lab-sfo-txy-01/
├── main.yml                  # 启用标志与绑定地址（第 3-5 行）
└── secrets.yml               # Gemini 凭据（Vault 加密）
```

---

## 变更记录 (Changelog)

### 2025-12-01
- **初始化**: 通过 AI 架构师生成完整的角色文档
- **当前状态**: 生产就绪，已在 lab-sfo-txy-01 部署（v1.17.1）
- **测试覆盖**: 无 Molecule 测试（需有效凭据）
- **已知限制**: 依赖 Google 账号 Cookie（可能被 Google 限制）

### 安全建议

1. **凭据管理**:
   - 始终通过 Vault 加密存储 Cookie 和 API Key
   - 定期轮换 API_KEY
   - 监控异常访问

2. **网络安全**:
   - 生产环境建议绑定到 Tailscale IP
   - 避免在公网暴露（除非通过 Traefik/Cloudflared 保护）

3. **日志审计**:
   - 定期检查容器日志
   - 监控 API 调用频率（避免触发 Google 限流）
