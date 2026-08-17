# Cluster Infra

面向 3～20 台 Ubuntu 24.04 HPC / AI 节点的基础设施配置仓库。当前 v0.3
实现 Ansible 基础配置、显式声明的 Fabric 网络、硬件与 NVIDIA GPU/Driver/CUDA
Toolkit 只读审计，以及显式的 NVIDIA Driver package 安装入口。MAAS、RDMA
tuning、Slurm、Kubernetes 与监控尚未实现。

## 安全边界

- `converge.yml` 不切换 kernel 或驱动，也不执行 reboot。
- `audit.yml` 只读取状态，不修复配置。
- `network` role 只管理 `fabric_interfaces` 声明的接口和独立的
  `/etc/netplan/90-cluster-fabric.yaml`，不管理默认路由、DNS 或管理网卡。
- Fabric 变更先通过 `netplan generate`，成功后才执行 `netplan apply`；无文件变化
  时不会 apply。
- `fabric_interfaces: []` 会且只会删除本项目管理的 Fabric Netplan 文件。
- `apply-versions.yml` 是显式危险操作入口，只安装 inventory 明确列出的 NVIDIA
  Driver package；它不切换 kernel、不安装 CUDA Toolkit，也不 reboot。
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
- 每台机器的 `fabric_interfaces` 仅包含独立计算网络接口，绝不能填写 SSH 管理接口。

Fabric 接口在 host 变量中显式声明；不需要 Fabric 配置的节点可以省略该变量：

```yaml
fabric_interfaces:
  - name: enp65s0f0
    address: 10.20.0.101/24
    mtu: 9000
```

GPU、Driver 和 CUDA Toolkit 是三个独立审计状态。CPU-only 节点默认合法；Driver
基线可按 branch 或精确版本声明，精确版本优先：

```yaml
nvidia_gpu_required: false
nvidia_driver_expected_branch: "595"
nvidia_driver_expected_version: ""
```

若尚未安装 `pciutils`，audit 会明确报告 `GPU detection unavailable`。这不会使默认的
CPU-only/非必需 GPU 节点失败；设置 `nvidia_gpu_required: true` 时仍会严格失败。

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

只有确认目标节点、package 和维护窗口后才执行 Driver apply。务必使用精确
`--limit`；playbook 会检查 GPU、当前 kernel、headers 和 APT candidate，并逐台执行：

```bash
uv run --project .. ansible-playbook \
  -i inventories/<cluster>/hosts.yml playbooks/apply-versions.yml \
  --limit nXX \
  -e '{"nvidia_driver_packages":["nvidia-driver-595-server-open"]}'
```

Driver package 发生变化时结果会报告 `reboot_required`，但不会自动 reboot。

幂等测试需在 Ubuntu 24.04 VM 上连续执行两次 `converge.yml`，第二次目标为
`changed=0`。详细设计和变量说明见 [docs/design.md](docs/design.md)；阶段需求见
[docs/dev.md](docs/dev.md)、[docs/dev-v0.2.md](docs/dev-v0.2.md) 和
[docs/dev-v0.3.md](docs/dev-v0.3.md)。
