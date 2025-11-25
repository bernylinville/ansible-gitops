# vps group vars

该目录保存 vps 组的公开变量 (`main.yml`) 与需要使用 Ansible Vault 管理的敏感变量 (`secrets.yml`)。

- `main.yml`：存放 geerlingguy 角色所需的非敏感配置，例如 Docker/Firewall/Security 的共享默认值。
- `secrets.yml`：必须使用 `ansible-vault encrypt_string --name cloudflare_tunnel_token '<token>'` 写入真实的 Cloudflare Tunnel Token，本仓库中仅保留占位密文，运行前务必替换。
