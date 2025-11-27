<!-- Improved compatibility of back to top link: See: https://github.com/othneildrew/Best-README-Template/pull/73 -->

<a id="readme-top"></a>

<!-- PROJECT SHIELDS -->

[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![project_license][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/bernylinville/ansible-gitops">
    <img src="docs/images/ansible-gitops.png" alt="Logo" width="80" height="80">
  </a>

<h3 align="center">Ansible GitOps for VPS & Homelab</h3>

  <p align="center">
    基于 Ansible 的 VPS / Homelab GitOps 基础设施，聚焦"Configuration over Creation"。
    <br />
    <a href="./docs"><strong>浏览架构与设计文档 »</strong></a>
    <br />
    <br />
    <a href="https://github.com/bernylinville/ansible-gitops">查看仓库</a>
    &middot;
    <a href="https://github.com/bernylinville/ansible-gitops/issues/new?labels=bug&template=bug-report---.md">反馈问题</a>
    &middot;
    <a href="https://github.com/bernylinville/ansible-gitops/issues/new?labels=enhancement&template=feature-request---.md">提出改进</a>
  </p>
</div>

<!-- TABLE OF CONTENTS -->
<details>
  <summary>目录</summary>
  <ol>
    <li>
      <a href="#about-the-project">项目概述</a>
      <ul>
        <li><a href="#built-with">技术栈</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">快速上手</a>
      <ul>
        <li><a href="#prerequisites">前置依赖</a></li>
        <li><a href="#installation">安装部署</a></li>
      </ul>
    </li>
    <li><a href="#usage">使用方式</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">贡献指南</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">联系方式</a></li>
    <li><a href="#acknowledgments">致谢</a></li>
  </ol>
</details>

<!-- ABOUT THE PROJECT -->

## About The Project

Ansible GitOps 旨在以声明式、可审计的方式管理个人 VPS 与 Homelab 主机。仓库遵循"Configuration over Creation"原则：优先复用成熟的 Galaxy 角色 (`geerlingguy.*`, `prometheus.prometheus`, `grafana.grafana`)，仅保留两个自定义角色处理共享 Docker 网络与 Cloudflare Tunnel 胶水逻辑。

```text
.
├── inventory/                # hosts、group_vars、host_vars + Vault secrets
├── playbooks/site.yml        # GitOps 入口，编排 geerlingguy + 监控 + Cloudflared
├── roles/
│   ├── cloudflared/          # 渲染 config/credentials 并运行容器
│   └── docker_shared_network # 统一维护 proxy_net 桥接网络
├── collections/requirements.yml
├── requirements.yml          # geerlingguy.* 版本锁
├── molecule/                 # 默认场景校验角色与隧道回归
└── memory-bank/              # PRD 与架构决策记录
```

### 核心能力

- **安全基线**：`geerlingguy.security` + `geerlingguy.firewall` 固化 SSH 加固与最小暴露端口策略。
- **容器平台**：`geerlingguy.docker` 负责 Engine/Compose 安装，`docker_shared_network` 在安装后创建 `proxy_net` 供 Cloudflared 与业务容器共享。
- **Zero-Trust 接入**：`cloudflared` role 渲染 `/opt/cloudflared/{config.yml,credentials.json}` 并将 Ingress 规则配置化。
- **监控栈**：对 `lab-sfo-txy-01` 启用 Prometheus + Alertmanager + Grafana，默认通过 Cloudflared 暴露，避免额外放行端口。
- **GitOps Ready**：`ansible.cfg` 指定 `vault_password_file`，CI 仅需注入 Vault 密码与 SSH Key 即可执行 `playbooks/site.yml`。

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Built With

- [![Ansible][Ansible.com]][Ansible-url]
- [![Molecule][Molecule.dev]][Molecule-url]
- [![Docker][Docker.com]][Docker-url]
- [![Cloudflare][Cloudflare.com]][Cloudflare-url]
- [![Prometheus][Prometheus.io]][Prometheus-url]

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- GETTING STARTED -->

## Getting Started

以下步骤帮助你在本地或 CI 环境快速复现相同的部署流程。

### Prerequisites

- Python 3.11+ 与 `pip`
- `python3-venv`（Linux 下创建虚拟环境）
- 具备访问目标主机的 SSH Key（默认为 `~/.ssh/id_ed25519`，可通过 `ANSIBLE_SSH_KEY_PATH` 覆盖）
- Cloudflare Tunnel 凭据 JSON 与 Vault 密码（用于加密 `inventory/**/secrets.yml`）

### Installation

1. 克隆仓库：
   ```bash
   git clone https://github.com/bernylinville/ansible-gitops.git
   cd ansible-gitops
   ```
2. 创建虚拟环境并安装 Python 依赖：
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install --upgrade pip
   pip install -r requirements.txt
   ```
3. 安装所需角色与集合：
   ```bash
   ansible-galaxy install -r requirements.yml
   ansible-galaxy collection install -r collections/requirements.yml
   ```
4. 准备库存与凭据：
   ```bash
   cp inventory/group_vars/vps/secrets.example.yml inventory/group_vars/vps/secrets.yml
   $EDITOR inventory/group_vars/vps/secrets.yml
   ansible-vault encrypt inventory/group_vars/vps/secrets.yml
   # 针对每台主机复制 host_vars/<hostname>/ 并加密 secrets.yml
   ```
5. 在仓库根创建 `.vault-password`（或在 CI 中通过 `ANSIBLE_VAULT_PASSWORD_FILE` 注入）。

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- USAGE EXAMPLES -->

## Usage

常用工作流如下：

```bash
# 1. 激活虚拟环境
source .venv/bin/activate

# 2. 静态检查
ansible-lint
yamllint .

# 3. Dry Run（针对 vps 组）
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff

# 4. 实际执行
ansible-playbook -i inventory/hosts.yml playbooks/site.yml

# 5. Molecule 回归（验证 Cloudflared + proxy_net）
cd molecule && molecule test
```

- 可通过 `--tags facts,deps,cloudflared` 等方式选择性执行任务。
- `inventory/host_vars/<hostname>/main.yml` 控制各主机的功能开关（如 `prometheus_enabled`）。
- 所有敏感变量必须存放在对应的 `secrets.yml` 中并使用 Vault 加密。
- GitHub Actions 中的 **Molecule CI** workflow 会在特性分支 push 与指向 `main` 的 Pull Request 上自动运行 `molecule test`，确保自定义角色与监控栈在容器内持续回归。

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- ROADMAP -->

## Roadmap

- [x] 搭建 Ansible-first 仓库骨架与版本锁
- [x] 集成 geerlingguy 安全 / 防火墙 / Docker 角色
- [x] 交付共享 Docker 网络与 Cloudflared 自定义角色
- [x] 为 `lab-sfo-txy-01` 启用 Prometheus + Alertmanager + Grafana
- [x] 将 GitHub Actions 工作流纳入 CI（galaxy install + ansible-lint + --check）
- [ ] 使用 Molecule + Testinfra 覆盖更多业务角色
- [ ] 为更多主机/环境抽象 inventory（`inventory/dev`, `inventory/prod`）
- [ ] 自动化发布应用栈（`app_stack_enabled`)

查看 [open issues](https://github.com/bernylinville/ansible-gitops/issues) 了解更多计划与问题列表。

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CONTRIBUTING -->

## Contributing

欢迎提交 Issue 或 PR！请遵循以下约定：

1. Fork 仓库并创建特性分支（`git checkout -b feature/cloudflared-v2`）。
2. 变更前先阅读 `memory-bank/` 中的 PRD 与架构文档，确保设计一致。
3. 保持任务幂等，优先使用 Ansible 模块而非 `command/shell`。
4. 所有变更需通过 `ansible-lint`、`yamllint` 以及相关角色的 `molecule test`。
5. 使用 Conventional Commits（如 `feat: add grafana datasource override`），并在 PR 描述中附上 `--check --diff` 的输出摘要。

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- LICENSE -->

## License

当前尚未选择开源协议；在添加 `LICENSE` 文件前，默认保留全部权利。若计划在外部分发，请先联系维护者。

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CONTACT -->

## Contact

如需讨论新特性或遇到问题，请在 Issue 中发帖或通过 Discussions 与维护者沟通。

Project Link: [https://github.com/bernylinville/ansible-gitops](https://github.com/bernylinville/ansible-gitops)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- ACKNOWLEDGMENTS -->

## Acknowledgments

- [Jeff Geerling 的 Ansible 角色集合](https://www.jeffgeerling.com/projects/ansible-roles)
- [Cloudflare Tunnel 文档](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Prometheus Operator / Collection](https://galaxy.ansible.com/prometheus/prometheus)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->

[contributors-shield]: https://img.shields.io/github/contributors/bernylinville/ansible-gitops.svg?style=for-the-badge
[contributors-url]: https://github.com/bernylinville/ansible-gitops/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/bernylinville/ansible-gitops.svg?style=for-the-badge
[forks-url]: https://github.com/bernylinville/ansible-gitops/network/members
[stars-shield]: https://img.shields.io/github/stars/bernylinville/ansible-gitops.svg?style=for-the-badge
[stars-url]: https://github.com/bernylinville/ansible-gitops/stargazers
[issues-shield]: https://img.shields.io/github/issues/bernylinville/ansible-gitops.svg?style=for-the-badge
[issues-url]: https://github.com/bernylinville/ansible-gitops/issues
[license-shield]: https://img.shields.io/badge/license-TBD-lightgrey?style=for-the-badge
[license-url]: #license
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://www.linkedin.com
[Ansible.com]: https://img.shields.io/badge/ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white
[Ansible-url]: https://www.ansible.com/
[Molecule.dev]: https://img.shields.io/badge/molecule-0A0A0A?style=for-the-badge&logo=pytest&logoColor=white
[Molecule-url]: https://molecule.readthedocs.io/
[Docker.com]: https://img.shields.io/badge/docker-0db7ed?style=for-the-badge&logo=docker&logoColor=white
[Docker-url]: https://www.docker.com/
[Cloudflare.com]: https://img.shields.io/badge/cloudflare-f38020?style=for-the-badge&logo=cloudflare&logoColor=white
[Cloudflare-url]: https://www.cloudflare.com/
[Prometheus.io]: https://img.shields.io/badge/prometheus-e6522c?style=for-the-badge&logo=prometheus&logoColor=white
[Prometheus-url]: https://prometheus.io/
