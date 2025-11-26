# vps group vars

该目录保存 vps 组的公开变量 (`main.yml`) 与需要使用 Ansible Vault 管理的敏感变量 (`secrets.yml`)。

- `main.yml`：存放 geerlingguy 角色所需的非敏感配置、共享 Docker 网络 (`docker_shared_network_*`) 以及 Cloudflared Ingress 默认值。
- `secrets.yml`：写入 `cloudflared_tunnel_credentials_json`（`cloudflared tunnel create` 生成的 JSON，包含 TunnelID/Secret）；Role 会自动从该 JSON 解析 Tunnel ID，如需手动覆盖可设置 `cloudflared_tunnel_id`。仓库中仅提供 `secrets.example.yml` 作为结构样例，请复制后运行 `ansible-vault encrypt inventory/group_vars/vps/secrets.yml`。
