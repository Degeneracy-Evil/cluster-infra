# Cluster Infra v0.3 GPU / Version 开发任务

## 1. 开发目标

在当前 `main` 的 v0.2 基础上实现：

1. 修复 Fabric 空配置残留问题。
2. 增强基础硬件审计。
3. 增加 NVIDIA GPU / Driver 审计。
4. 建立 Kernel / NVIDIA Driver 版本基线机制。
5. 实现显式的 NVIDIA Driver `apply-versions.yml`。
6. 保持日常 `converge.yml` 安全、幂等，不自动切换关键版本。

当前阶段仍不实现：

```text
RDMA / RoCE tuning
Slurm
Kubernetes
Prometheus
MAAS
Kernel 自动切换
自动 reboot
```

CUDA Toolkit 本阶段以**检测和报告为主**，不作为所有节点必须安装的组件。

---

## 2. Git 分支

从最新 `main` 创建：

```text
agent/gpu-version
```

当前 v0.2 `main` 已包含 `base + network + audit`。

完成开发和验证后汇报结果，暂不扩展其他功能。

---

## 3. 修复 v0.2 Fabric 残留

当前：

```yaml
fabric_interfaces: []
```

时，应保证本项目管理的：

```text
/etc/netplan/90-cluster-fabric.yaml
```

处于 `absent` 状态。

语义：

```text
fabric_interfaces 非空
→ 90-cluster-fabric.yaml present

fabric_interfaces 为空
→ 90-cluster-fabric.yaml absent
```

删除配置文件后同样：

```text
netplan generate
→ netplan apply
```

仅删除 cluster-infra 自己管理的文件。

继续保持：

```text
持久配置 drift → converge 修复
纯运行时 ip 命令造成的 drift → audit 发现
```

---

## 4. Hardware Audit

扩展现有 `audit` role，使其能够快速了解陌生计算节点。

至少输出：

```text
Node
Hostname
Ubuntu version
Kernel

CPU model
CPU sockets / cores / vCPUs

Memory total

Root filesystem capacity

NVIDIA GPU:
  detected
  count
  model
```

保持适度简单，不实现完整资产管理。

当前不需要采集：

```text
DIMM 明细
SMART
PCIe topology
NUMA topology
BIOS inventory
```

所有 audit task 必须保持：

```text
changed=0
```

---

## 5. NVIDIA GPU Audit

GPU 检测需要区分：

```text
GPU hardware
NVIDIA driver
CUDA Toolkit
```

三种状态互相独立。

### GPU Hardware

检测机器是否存在 NVIDIA GPU。

建议优先从 PCI 信息检测，使 driver 未安装时仍能识别 NVIDIA GPU。

输出例如：

```text
gpu_present: true
gpu_count: 8
gpu_models:
  - NVIDIA H20
```

CPU-only 节点属于合法状态。

---

### NVIDIA Driver

如果 NVIDIA driver 可用，检测：

```text
driver_loaded
driver_version
driver_branch
```

例如：

```text
driver_version: 595.57.03
driver_branch: 595
```

GPU 存在但 driver 不可用时，应明确报告：

```text
GPU hardware: detected
Driver: missing / unavailable
```

避免简单因为 `nvidia-smi` 失败而失去硬件信息。

---

### CUDA Toolkit

检查宿主机是否存在：

```text
nvcc
```

如果存在则报告版本。

如果不存在：

```text
cuda_toolkit_installed: false
```

这属于合法状态，默认不导致 audit 失败。

宿主机 NVIDIA Driver 与 CUDA Toolkit 必须作为两个独立概念处理。

---

## 6. Kernel / Driver Baseline

继续使用简单的 Ansible 变量，不增加额外配置 DSL。

建议支持：

```yaml
audit_expected_kernel: ""

nvidia_driver_expected_branch: ""
nvidia_driver_expected_version: ""
```

### Kernel

如果：

```yaml
audit_expected_kernel: ""
```

仅报告当前 kernel。

如果配置：

```yaml
audit_expected_kernel: "6.8.0-136-generic"
```

则进行精确检查。

本阶段不自动安装、升级、降级或切换 Kernel。

---

### NVIDIA Driver Branch

通常使用：

```yaml
nvidia_driver_expected_branch: "595"
```

例如：

```text
595.x → OK
600.x → DRIFT
```

---

### NVIDIA Driver Exact Version

可选：

```yaml
nvidia_driver_expected_version: "595.57.03"
```

如果配置 exact version，则精确版本检查优先于 branch。

这样既支持长期集群按 branch 管理，也支持比赛现场锁定某个精确版本。

---

## 7. 异构节点与 Override

继续使用 Ansible 原生：

```text
group_vars
host_vars
```

例如集群基线：

```yaml
# group_vars/all.yml

nvidia_driver_expected_branch: "595"
```

特殊节点：

```yaml
# host_vars/n03.yml

nvidia_driver_expected_branch: "600"
override_reason: "Temporary hardware compatibility workaround"
```

无需设计额外 override 数据结构。

Role 不根据 H20、A100、V100 等 GPU 型号自动猜测目标 driver。

硬件负责检测，安装策略由 inventory 明确指定。

---

## 8. NVIDIA Driver Apply

正式实现现有：

```text
playbooks/apply-versions.yml
```

本阶段只负责 **NVIDIA Driver 显式变更**。

建议使用 inventory 明确声明目标 package，例如：

```yaml
nvidia_driver_packages:
  - nvidia-driver-595-server-open
```

Role 不自行拼接或猜测 package 名称。

### 执行前检查

至少检查：

1. 当前节点存在 NVIDIA GPU。
2. 当前运行 kernel 可正常识别。
3. 对应 kernel headers 已安装。
4. `nvidia_driver_packages` 已明确配置。
5. 目标 package 在 APT 中可用。

任一关键前置条件不满足时清晰失败。

---

### Apply 行为

只有显式执行：

```bash
ansible-playbook \
  -i inventories/<cluster> \
  playbooks/apply-versions.yml \
  --limit nXX
```

才允许安装目标 NVIDIA Driver package。

普通：

```text
setup.yml
converge.yml
audit.yml
```

均不得自动切换 NVIDIA Driver。

Driver 发生变化后：

```text
报告 reboot_required
```

但本阶段不自动 reboot。

---

## 9. Kernel Headers

Audit 增加当前 kernel headers 检查。

例如当前：

```text
uname -r
→ 6.8.0-136-generic
```

应检查对应 headers 是否存在。

输出：

```text
kernel_headers: OK / MISSING
```

Driver apply 前要求当前 kernel headers 可用。

---

## 10. Role 结构

保持项目简单。

建议新增：

```text
ansible/roles/gpu/
├── defaults/main.yml
├── tasks/
│   ├── main.yml
│   ├── audit.yml        # 如果实际结构需要
│   └── apply.yml
└── README.md
```

也可以继续让硬件/GPU 的只读检测主要位于现有 `audit` role。

优先选择代码最清楚、重复最少的结构，不为了目录形式额外拆分。

日常入口保持：

```text
setup.yml
converge.yml
audit.yml
apply-versions.yml
```

不要新增大量用户入口。

---

## 11. `converge.yml` 行为

继续保持：

```text
base
+
network
```

GPU Driver 不进入普通 converge 的强制版本切换流程。

如果未来需要管理一些完全安全的 GPU 配置，可以后续再加入。

本阶段重点确保：

```text
converge
→ 不安装 Driver
→ 不切换 Kernel
→ 不安装 CUDA Toolkit
→ 不 reboot
```

---

## 12. `audit.yml` 行为

执行：

```bash
ansible-playbook \
  -i inventories/<cluster> \
  playbooks/audit.yml
```

能够得到类似：

```text
n01

OS:
  Ubuntu 24.04

Kernel:
  6.8.0-136-generic
  baseline: OK
  headers: OK

CPU:
  model: ...
  sockets: 2
  vcpus: 128

Memory:
  ...

GPU:
  present: true
  count: 8
  model: NVIDIA H20

NVIDIA Driver:
  loaded: true
  version: 595.57.03
  branch: 595
  baseline: OK

CUDA Toolkit:
  installed: false

Fabric:
  ens4: OK
```

具体显示格式可以根据现有 audit 风格实现，无需制作复杂报告系统。

---

## 13. VM 测试

继续运行现有 Ubuntu 24.04 QEMU 测试。

VM 没有 NVIDIA GPU属于合法测试环境。

需要验证：

```text
Hardware audit 正常
GPU absent 被正确识别
CUDA absent 被正确识别
audit 不因 GPU 缺失异常
```

如果 inventory 没有声明该节点必须拥有 NVIDIA GPU，则 CPU-only 节点应通过 audit。

同时重新执行：

```text
inventory graph
syntax-check
ansible-lint
```

以及：

```text
setup
converge
converge
audit
```

要求第二次：

```text
converge changed=0
audit changed=0
```

---

## 14. Fabric Bug 回归测试

增加测试：

1. 配置 `fabric_interfaces`。
2. 执行 converge，确认 `90-cluster-fabric.yaml` 存在。
3. 修改 inventory：

```yaml
fabric_interfaces: []
```

4. 再次 converge。
5. 确认项目 Netplan 文件被删除。
6. 确认 Fabric 配置撤销。
7. 再次 converge：

```text
changed=0
```

---

## 15. 三台真实机器验证

代码在 VM/static test 完成后，可使用现有三台异构机器进行第一轮真实硬件测试。

### 第一阶段只读测试

首先只执行：

```bash
ansible-playbook \
  -i inventories/<test-cluster> \
  playbooks/audit.yml
```

此阶段禁止安装/卸载驱动、切换 kernel 或 reboot。

确认三台机器能够正确识别：

```text
CPU
Memory
Kernel
Kernel headers
GPU presence
GPU count/model
NVIDIA Driver version
Driver branch
CUDA Toolkit
Fabric
```

重点验证不同硬件环境不会导致 audit 本身异常。

---

### Driver Apply 测试

只有在用户明确指定：

```text
目标节点
目标 driver package
```

后才执行真实：

```text
apply-versions.yml
```

优先：

```text
--limit 单台测试节点
```

验证完成后再考虑扩大范围。

真实机器验证中不自动 reboot。

---

## 16. 安全要求

整个开发和默认测试过程中：

* VM 环境允许正常修改 guest。
* 真实机器默认只运行 audit。
* 未明确指定时不修改真实机器 driver/kernel。
* 不自动 reboot / shutdown 真实服务器。
* 不执行 kernel upgrade/downgrade。
* 不安装 CUDA Toolkit。
* 不修改宿主开发机系统配置。

---

## 17. 文档更新

更新：

```text
README.md
docs/design.md
docs/testing.md
```

项目版本更新至：

```text
0.3.0
```

记录：

* Hardware/GPU audit。
* Driver baseline 语义。
* branch / exact version 优先级。
* `apply-versions.yml` 的显式危险操作属性。
* CUDA Toolkit 为可选宿主组件。
* Kernel 当前只审计、不自动切换。
* Fabric 空配置残留 bug 的修复。

---

## 18. 完成标准

v0.3 完成时应满足：

```text
Base                       ✓
Fabric Network             ✓
Fabric removal convergence ✓
Hardware Audit             ✓
NVIDIA GPU Audit           ✓
Driver Version Baseline    ✓
Explicit Driver Apply      ✓
Kernel Audit               ✓
```

并通过：

```text
inventory graph
syntax-check
ansible-lint: 0 failure / 0 warning
VM setup
VM converge ×2 → 第二次 changed=0
VM audit → changed=0
Fabric removal regression
三台异构真机只读 audit
```

如果真实 NVIDIA Driver apply 尚未获得明确测试条件，可以保留为经过静态/VM逻辑验证的功能，并在测试文档中明确标记尚未进行真实驱动变更测试。

完成上述范围后停止扩展，汇报实现、验证结果、三台真实机器发现的硬件差异以及需要下一阶段处理的问题。
