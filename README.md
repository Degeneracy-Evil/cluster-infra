# Cluster Infra

面向 3～20 台 Ubuntu 24.04 HPC / AI 节点的基础设施配置仓库。当前 v0.1
只实现 Ansible 基础配置和审计；MAAS、GPU、RDMA、Slurm、Kubernetes 与监控
尚未实现。

## 安全边界

- `converge.yml` 不切换 kernel 或驱动，也不执行 reboot。
- `audit.yml` 只读取状态，不修复配置。
- `apply-versions.yml` 当前是安全失败的占位 playbook，不进行任何修改。
- `inventories/test` 默认使用 RFC 5737 文档地址，必须替换后才能连接测试 VM。
- 执行前应检查 inventory 和 `--limit`，先使用 `--check --diff` 预览。

## 准备

Python 工具统一由 `uv` 管理。先在仓库根目录同步隔离环境，再进入 `ansible/`
安装 collection。项目将本地临时文件、日志、缓存与 collection 放在忽略目录中。

```bash
uv sync --dev
cd ansible
uv run --project .. ansible-galaxy collection install \
  -r requirements.yml -p .local/collections
```

复制或修改 inventory，至少确认：

- `ansible_host` 指向隔离测试节点或正确的管理网地址；
- `ansible_user` 已存在且拥有 sudo 权限；
- 管理员公钥不是示例值；
- 时区、软件包和安全策略符合目标集群要求。

## 验证和执行

```bash
uv run --project .. ansible-inventory -i inventories/test/hosts.yml --graph
uv run --project .. ansible-playbook \
  -i inventories/test/hosts.yml playbooks/converge.yml --syntax-check
uv run --project .. ansible-playbook \
  -i inventories/test/hosts.yml playbooks/audit.yml --syntax-check

# 首次应用前预览；建议同时使用精确的 --limit。
uv run --project .. ansible-playbook \
  -i inventories/test/hosts.yml playbooks/setup.yml \
  --check --diff --limit n01

# 日常安全收敛。
uv run --project .. ansible-playbook \
  -i inventories/test/hosts.yml playbooks/converge.yml

# 只读审计。
uv run --project .. ansible-playbook \
  -i inventories/test/hosts.yml playbooks/audit.yml
```

幂等测试需在 Ubuntu 24.04 VM 上连续执行两次 `converge.yml`，第二次目标为
`changed=0`。详细设计和变量说明见 [docs/design.md](docs/design.md)，原始开发范围见
[docs/dev.md](docs/dev.md)。
