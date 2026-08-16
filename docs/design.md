# Cluster Infra v0.1 设计

## 范围

当前阶段只提供 Ubuntu 24.04 计算节点的 Ansible 基础配置和状态审计。
MAAS、网络拓扑、GPU、RDMA、调度系统、容器平台、监控以及关键版本切换均不在
本阶段实现范围内。

## 架构原则

1. Inventory 是集群差异的唯一入口；公共变量放在 `group_vars/all.yml`，单机差异
   放在 `host_vars/nXX.yml`。
2. Role 保持通用，不生成 inventory，也不引入额外配置 DSL。
3. 配置优先使用 Ansible module，并让模板内容稳定排序，以获得可预测的幂等结果。
4. `converge.yml` 只管理可安全重复应用的状态；关键版本切换和 reboot 不属于它。
5. `audit.yml` 只读取和判断状态，不触发 handler，也不执行修复。
6. 扩展以实际需求为准，不提前引入复杂分组和抽象。

## 执行入口

| Playbook | 用途 | 行为 |
| --- | --- | --- |
| `setup.yml` | 新装节点首次配置 | 校验 Ubuntu 版本，执行 `base`，刷新 handler 后执行 `audit` |
| `converge.yml` | 日常维护 | 仅执行 `base`，不 reboot、不切换 kernel/驱动 |
| `audit.yml` | 状态检查 | 只读采集并汇总，不修改节点 |
| `apply-versions.yml` | 后续版本管理占位 | 当前主动失败且不修改任何状态 |

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
| `audit_expected_kernel` | string | 可选的精确 kernel 基线；空值表示只报告 |
| `audit_fail_on_mismatch` | bool | 审计异常时是否令 play 失败 |
| `audit_require_ntp_synchronized` | bool | 是否把尚未同步 NTP 视为失败 |

管理员账户默认 `append: true`，不会覆盖已有附加组；密码只在创建账户时设置。
`exclusive_authorized_keys: true` 会使声明的 key 列表成为该用户的完整授权集合，使用
前必须确认不会删除仍需保留的 key。

## Audit 语义

审计检查 hostname、Ubuntu release、可选 kernel 基线、timezone、基础包、SSH 配置
及服务、NTP 服务与同步状态、periodic unattended upgrade 策略和 unattended kernel
策略。每台节点先输出结构化摘要，再执行统一断言，便于定位差异。

首次启动后 NTP 可能尚未同步，因此默认只报告 `ntp_synchronized`，不据此失败；需要
严格检查时设置 `audit_require_ntp_synchronized: true`。

## 幂等与验证

- APT index 使用 `cache_valid_time`，短时间内的第二次执行不重复更新。
- 文件由稳定模板管理，只有内容变化才通知 handler。
- SSH 配置变化后先执行 `sshd -t`，成功才 reload。
- audit 中的命令均为只读，并显式设置 `changed_when: false`。
- 项目没有任何 reboot task。

合并前应执行 inventory graph、三个 playbook 的 syntax-check，并在隔离的 Ubuntu
24.04 VM 上连续执行两次 converge。第二次应尽可能达到 `changed=0`；任何合理的
动态变化都必须在测试记录中说明。
