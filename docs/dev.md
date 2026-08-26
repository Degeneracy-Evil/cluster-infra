# Cluster Infra v0.1 开发文档

## 1. 项目目标

建立一套面向小规模 HPC / AI 裸金属集群的统一部署与配置方案，典型规模为 3～20 台服务器。

技术栈：

* **Ubuntu 24.04 LTS**
* **MAAS**：负责裸机发现、PXE、系统安装和重装。
* **Ansible**：负责系统配置与长期状态收敛。
* **Git**：保存所有集群配置和自动化代码。

当前阶段重点开发 **Ansible 基础配置能力**。MAAS、NVIDIA GPU、RDMA、Slurm、Kubernetes、Prometheus 后续逐步加入。

开发环境和 Ansible collection 均安装在仓库本地。首次初始化或依赖更新后执行：

```bash
uv sync --group dev
cd ansible
uv run --project .. ansible-galaxy collection install \
  -r collections/requirements.yml -p .local/collections
```

`ansible.cfg` 已将 collection、临时文件和日志指向 `ansible/.local/`，不需要修改
系统级 Ansible 配置。

核心目标：

1. 新 Ubuntu 节点可以通过一条 Ansible 命令完成基础配置。
2. 所有配置尽量使用 Ansible 原生 module，并保证严格幂等。
3. 同一个 playbook 连续运行两次时，第二次应尽可能达到 `changed=0`。
4. 集群配置通过 inventory 管理，可以方便复用于比赛、实验室和公司集群。
5. 关键软件版本有统一基线，同时保留比赛现场人工调整的灵活性。
6. 普通维护操作保持安全，不自动执行 kernel 切换、驱动替换和 reboot。

---

## 2. 节点规范

统一使用：

```text
n01
n02
...
n10
...
n99
```

推荐管理网地址保持直观对应：

```text
n01 → x.x.x.101
n02 → x.x.x.102
...
n10 → x.x.x.110
```

`n01` 默认可以同时承担：

```text
bootstrap
MAAS
Ansible
compute
```

其余节点作为普通计算节点。

---

## 3. 第一阶段目录结构

建立：

```text
cluster-infra/
├── ansible/
│   ├── ansible.cfg
│   │
│   ├── inventories/
│   │   ├── example/
│   │   │   ├── hosts.yml
│   │   │   ├── group_vars/
│   │   │   │   └── all.yml
│   │   │   └── host_vars/
│   │   │
│   │   └── test/
│   │       ├── hosts.yml
│   │       └── group_vars/
│   │           └── all.yml
│   │
│   ├── roles/
│   │   ├── base/
│   │   ├── network/
│   │   ├── gpu/
│   │   ├── rdma/
│   │   └── audit/
│   │
│   └── playbooks/
│       ├── setup.yml
│       ├── converge.yml
│       ├── audit.yml
│       └── apply-versions.yml
│
├── maas/
│   └── README.md
│
├── docs/
│   └── design.md
│
├── .gitignore
└── README.md
```

当前只完整实现：

```text
base
audit
setup.yml
converge.yml
audit.yml
```

`network/gpu/rdma/apply-versions.yml` 先建立目录或占位文件，后续开发。

---

## 4. Inventory 设计

示例：

```yaml
# inventories/example/hosts.yml

all:
  children:
    compute:
      hosts:
        n01:
          ansible_host: 10.10.0.101
        n02:
          ansible_host: 10.10.0.102

    bootstrap:
      hosts:
        n01:
```

集群公共配置：

```yaml
# inventories/example/group_vars/all.yml

cluster_name: example-cluster

os_release: "24.04"
timezone: Asia/Shanghai

ansible_user: clusteradmin

disable_unattended_upgrades: true
kernel_hold: true

base_packages:
  - vim
  - curl
  - wget
  - git
  - htop
  - tmux
  - rsync
  - jq
  - pciutils
  - usbutils
  - lsof
  - tree
```

特殊节点使用：

```text
host_vars/nXX.yml
```

进行覆盖。

优先采用 Ansible 原生的 inventory、`group_vars` 和 `host_vars` 变量机制，不额外设计配置生成器。

---

## 5. `base` Role

第一阶段 `base` 管理以下基础系统状态：

### Hostname

确保节点 hostname 与 inventory 名称一致：

```text
n01
n02
...
```

### APT

完成：

* `apt update` 的合理管理。
* 基础软件包安装。
* unattended upgrades 策略配置。
* 软件安装保持幂等。

### Time

配置：

```text
timezone
NTP / systemd-timesyncd
```

### SSH

管理基础 SSH Server 状态和必要配置。

SSH 配置修改通过 handler 重载或重启服务。

### Users

提供基础管理员用户管理能力。

具体用户列表通过变量提供，role 本身保持通用。

### sysctl

使用 Ansible sysctl module 管理系统参数。

后续 HPC、GPU、RDMA 专用参数由相应 role 添加。

### limits

支持通过变量管理 `/etc/security/limits.d/` 配置。

### Kernel Update Policy

当：

```yaml
kernel_hold: true
```

时，配置系统避免 unattended kernel upgrade。

关键 kernel 版本切换留给后续 `apply-versions.yml`。

---

## 6. Playbook 行为

### `setup.yml`

面向刚安装完成的 Ubuntu 节点。

第一阶段执行：

```text
precheck
→ base
→ audit
```

未来会继续加入：

```text
network
gpu
rdma
```

### `converge.yml`

日常维护入口。

目标：

```bash
ansible-playbook \
  -i inventories/<cluster> \
  playbooks/converge.yml
```

可以随时安全执行。

行为要求：

* 幂等。
* 收敛基础系统配置。
* 配置变化时才触发对应 handler。
* 保留现场临时 kernel / driver 调整。
* reboot 作为显式操作。

### `audit.yml`

只检查节点状态。

第一阶段检查：

```text
hostname
Ubuntu release
kernel
timezone
基础 packages
SSH
NTP
unattended upgrade policy
基础系统状态
```

输出应便于快速发现异常节点。

---

## 7. 幂等性开发规范

所有 role 以幂等性为核心质量要求。

优先使用：

```text
apt
package
template
copy
lineinfile
blockinfile
user
authorized_key
service
systemd
sysctl
file
mount
```

等 Ansible module。

使用 `command` 或 `shell` 时，应根据实际情况设置：

```text
changed_when
failed_when
creates
removes
```

配置文件发生变化后通过 handler 更新服务。

稳定环境的软件包默认使用明确状态或版本策略。

---

## 8. 测试要求

当前没有裸金属测试环境，因此第一阶段使用 Ubuntu 24.04 VM。

建议至少准备：

```text
n01
n02
```

两台测试 VM。

每次完成一个 role 后执行：

```bash
ansible-playbook \
  -i inventories/test \
  playbooks/converge.yml
```

连续运行两次。

期望：

```text
第一次：正常产生 changed
第二次：changed=0
```

如存在合理的动态 task，应在代码中清楚说明原因。

同时执行：

```bash
ansible-playbook --syntax-check ...
ansible-inventory --graph ...
```

验证 playbook 和 inventory。

---

## 9. 当前开发范围

本次开发完成以下内容即可：

```text
1. 初始化 Git 项目目录。
2. 建立 example/test inventory。
3. 完成 ansible.cfg。
4. 完成 base role。
5. 完成 setup.yml。
6. 完成 converge.yml。
7. 完成基础 audit role 和 audit.yml。
8. 编写简短 README。
9. 编写 docs/design.md，记录当前架构原则。
10. 在 Ubuntu 24.04 VM 上验证两次执行后的幂等性。
```

当前阶段暂缓实现：

```text
MAAS 自动化
NVIDIA Driver / CUDA
IB / RoCE
Slurm
Kubernetes
Prometheus
复杂网络拓扑
关键版本自动切换
```

后续按照新的开发任务逐项加入。

---

## 10. 开发原则

整个项目优先遵循以下原则：

1. **简单优先**：面向几台到十几台服务器，维持较低使用和维护成本。
2. **幂等优先**：Ansible 多次执行能够稳定收敛到相同状态。
3. **显式配置**：hostname、IP、重要配置尽量可以直接从 inventory 阅读。
4. **安全收敛**：日常 `converge.yml` 适合随时执行。
5. **灵活版本策略**：生产基线明确，比赛现场允许人工快速调整。
6. **逐步扩展**：只有真实需求出现时再增加 group、role 和抽象层。
7. **Git 记录正式状态**：值得长期保留的人工修改最终写回 Ansible 并提交 Git。

本阶段完成后停止继续扩展功能，汇报代码结构、实现内容、幂等测试结果以及发现的问题，等待下一阶段任务。
