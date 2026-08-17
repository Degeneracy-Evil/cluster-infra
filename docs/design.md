# Cluster Infra v0.3 设计

## 范围

当前阶段提供 Ubuntu 24.04 计算节点的基础配置、独立 Fabric 网络、硬件和 NVIDIA
GPU/Driver/CUDA Toolkit 审计，以及显式 NVIDIA Driver package 安装。MAAS、管理
网络、NAT、RDMA tuning、调度系统、容器平台、监控和 kernel 切换均不在范围内。

## 架构原则

1. Inventory 是集群差异的唯一入口；公共变量放在 `group_vars/all.yml`，单机差异
   放在 `host_vars/nXX.yml`。
2. Role 保持通用，不生成 inventory，也不引入额外配置 DSL。
3. 配置优先使用 Ansible module，并让模板内容稳定排序，以获得可预测的幂等结果。
4. `converge.yml` 只管理可安全重复应用的状态；关键版本切换和 reboot 不属于它。
5. `audit.yml` 只读取和判断状态，不触发 handler，也不执行修复。
6. 扩展以实际需求为准，不提前引入复杂分组和抽象。
7. Management network 由 MAAS 或现有系统管理；Ansible 不修改当前默认路由接口。
8. Fabric network 由 Ansible 管理，但仅限 inventory 中显式声明的
   `fabric_interfaces`。
9. GPU 硬件、NVIDIA Driver 和 CUDA Toolkit 是三个独立状态；硬件检测不依赖 Driver。
10. 关键版本变更只能通过显式入口执行，并且永不自动 reboot。

## 执行入口

| Playbook | 用途 | 行为 |
| --- | --- | --- |
| `setup.yml` | 新装节点首次配置 | 校验 Ubuntu 版本，执行 `base`、`network`，刷新 handler 后执行 `audit` |
| `converge.yml` | 日常维护 | 执行 `base`、`network`，不 reboot、不切换 kernel/驱动 |
| `audit.yml` | 状态检查 | 只读采集系统、硬件、GPU、Driver、CUDA 和 Fabric 状态 |
| `apply-versions.yml` | 显式 Driver 变更 | 串行安装明确声明的 NVIDIA Driver package，不 reboot |

三个有效入口默认面向 `compute` group。`bootstrap` 表示节点职能，不会让同一主机
重复执行 play。

## `base` role

`base` 管理以下状态：

- hostname 等于 `inventory_hostname`；
- APT 索引在缓存过期时更新，基础软件包保持 `present`；
- periodic APT 和 unattended kernel upgrade 策略；
- timezone 与 `systemd-timesyncd`；
- SSH server、独立 drop-in 配置及安全的校验后 reload；
- 可选管理员账号和 authorized keys；
- 可选 sysctl 和 PAM limits 配置。

SSH 默认仅设置连接保活，不修改 `PasswordAuthentication`、`PermitRootLogin` 等认证
选项，避免默认配置造成管理入口锁定。需要加强认证策略时，应在 inventory 中显式
设置 `base_ssh_settings`，并先在隔离 VM 验证。

`kernel_hold: true` 只向 unattended-upgrades 添加 kernel 软件包黑名单。它不会运行
`apt-mark hold`、不会安装或删除内核，也不会改变当前启动内核。设置为 `false` 时只
删除本项目管理的黑名单文件。

## `network` role

`network` 只读取每台主机的 `fabric_interfaces`，每项包含 `name`、`address` 和
`mtu`。它将所有 rail 稳定排序后写入独立的
`/etc/netplan/90-cluster-fabric.yaml`，只配置静态地址与 MTU，不设置 gateway、DNS
或 default route，也不删除或改写其他 Netplan 文件。变量为空时删除本项目管理的
文件，使持久配置和运行状态经 Netplan 收敛；其他 Netplan 文件保持不变。

写入前会验证字段、接口名唯一性和接口是否真实存在；若声明接口等于 facts 中当前
默认路由接口，任务会在写文件前失败，以保护管理网络。配置变化后 handler 先运行
`netplan generate`，成功后才运行 `netplan apply`。模板没有变化时两个 handler 均
不会执行。

## 硬件与 GPU 审计

hardware audit 使用 facts 汇总 CPU model、socket、core、vCPU、内存和根文件系统
容量，并通过 `/lib/modules/<running-kernel>/build` 报告当前 kernel headers 状态。
headers 缺失默认只报告，不触发自动安装或 kernel 变更。

GPU audit 通过 PCI class `0300/0302/0380` 检测 NVIDIA 硬件，因此 Driver 缺失时仍
能报告 GPU。`/sys/module/nvidia` 表示模块状态，`nvidia-smi` 提供 Driver version，
`nvcc` 只用于判断可选 CUDA Toolkit。CPU-only 节点默认合法；需要 GPU 时显式设置
`nvidia_gpu_required: true`。

Driver baseline 支持 branch 和 exact version。配置 exact version 时优先精确比较；
否则比较 branch；两个值都为空时只报告。role 不根据 GPU 型号猜测目标版本。

## 显式 Driver apply

`apply-versions.yml` 是唯一允许安装 NVIDIA Driver 的入口。它逐台验证 NVIDIA GPU、
当前 kernel module 目录、对应 headers、非空的 `nvidia_driver_packages` 和每个 APT
candidate，然后以 `state: present` 安装明确 package。package 变化或系统已有 reboot
marker 时报告 `reboot_required: true`，但不执行 reboot。

普通 `setup.yml`、`converge.yml` 和 `audit.yml` 不进入 apply task，不安装 Driver、
CUDA Toolkit 或 kernel。

```yaml
fabric_interfaces:
  - name: enp65s0f0
    address: 10.20.0.101/24
    mtu: 9000
  - name: enp65s0f1
    address: 10.21.0.101/24
    mtu: 9000
```

## 变量接口

| 变量 | 类型 | 说明 |
| --- | --- | --- |
| `os_release` | string | setup/audit 期望的 Ubuntu 版本 |
| `timezone` | string | IANA 时区名称 |
| `base_packages` | list[string] | 集群要求的基础软件包 |
| `disable_unattended_upgrades` | bool | 是否关闭周期性 unattended upgrade |
| `kernel_hold` | bool | 是否禁止 unattended kernel upgrade |
| `base_ssh_settings` | mapping | 写入独立 sshd drop-in 的键值 |
| `base_admin_users` | list[mapping] | 管理员账户、组、shell、密码策略和公钥 |
| `base_sysctl` | mapping | sysctl 名称和值 |
| `base_limits` | list[mapping] | `domain/type/item/value` limits 条目 |
| `fabric_interfaces` | list[mapping] | Fabric 接口的显式 `name/address/mtu` 声明；默认空列表 |
| `audit_expected_kernel` | string | 可选的精确 kernel 基线；空值表示只报告 |
| `nvidia_gpu_required` | bool | 节点是否必须存在 NVIDIA GPU；默认 false |
| `nvidia_driver_expected_branch` | string | 可选 Driver branch 基线 |
| `nvidia_driver_expected_version` | string | 可选精确 Driver 版本；优先于 branch |
| `nvidia_driver_packages` | list[string] | 仅供 apply-versions 使用的完整 APT package 名称 |
| `audit_fail_on_mismatch` | bool | 审计异常时是否令 play 失败 |
| `audit_require_ntp_synchronized` | bool | 是否把尚未同步 NTP 视为失败 |

管理员账户默认 `append: true`，不会覆盖已有附加组；密码只在创建账户时设置。
`exclusive_authorized_keys: true` 会使声明的 key 列表成为该用户的完整授权集合，使用
前必须确认不会删除仍需保留的 key。

## Audit 语义

审计检查 hostname、Ubuntu release、可选 kernel 基线、timezone、基础包、SSH 配置
及服务、NTP 服务与同步状态、periodic unattended upgrade 策略和 unattended kernel
策略。摘要还包含 CPU、内存、根文件系统、kernel headers、GPU 硬件、Driver、CUDA
Toolkit。对于每个声明的 Fabric 接口，使用 `ip -j` 检查接口存在、链路 UP、CIDR、
MTU 和无 default route。每台节点先输出结构化摘要，再执行统一断言，便于定位差异。

首次启动后 NTP 可能尚未同步，因此默认只报告 `ntp_synchronized`，不据此失败；需要
严格检查时设置 `audit_require_ntp_synchronized: true`。

## 幂等与验证

- APT index 使用 `cache_valid_time`，短时间内的第二次执行不重复更新。
- 文件由稳定模板管理，只有内容变化才通知 handler。
- SSH 配置变化后先执行 `sshd -t`，成功才 reload。
- Fabric 模板稳定排序；Netplan 变化后先 generate，再 apply。
- Fabric 列表为空时，仅删除本项目的 Netplan 文件并按相同顺序收敛。
- audit 中的命令均为只读，并显式设置 `changed_when: false`。
- 项目没有任何 reboot task。

发布前应执行 inventory graph、四个 playbook 的 syntax-check 和 ansible-lint，并在
隔离的 Ubuntu 24.04 VM 上验证首次配置、硬件/GPU absent 审计、Fabric 配置与删除、
连续两次 converge、只读 audit 以及 apply 前置条件安全失败。稳定状态下 converge 和
audit 均应达到 `changed=0`。真实 GPU 节点先只读 audit；Driver apply 需要用户明确
目标节点和 package 后单独验证。
