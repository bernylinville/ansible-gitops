# host_vars 指南

为每台主机创建一个目录 `<hostname>/`，下含：

- `main.yml`：该主机的功能开关或非敏感覆盖项，可引用 `secrets.yml` 中的变量（例如 `ansible_port: "{{ security_ssh_port }}"`）。
- `secrets.yml`：敏感数据（如 `ansible_host`、`security_ssh_port`、需要隐藏的 `ansible_user`）。请先写入明文占位值，确认无误后运行 `ansible-vault encrypt inventory/host_vars/<hostname>/secrets.yml`，这样 IP/端口等连接信息即被 Vault 保护。

示例 `lab-sfo-txy-01/` 目录展示了约定结构，可复制后替换为实际主机名。
