# v0.2 验证记录

验证日期：2026-08-16

## 当前环境中的验证

- 所有操作均限制在项目目录内。
- 未安装宿主机系统软件，也未修改宿主机配置。
- 使用 `uv` 在仓库内创建隔离环境：`ansible-core 2.21.3`、
  `ansible-lint 26.8.0`。
- `example` 和 `test` inventory graph 均通过，节点分组和 Fabric 变量符合设计。
- 四个 playbook 的 Ansible syntax-check 全部通过。
- `ansible-lint` 通过：0 failure、0 warning，并达到 production profile。
- 使用 Ruby 标准库解析仓库中的全部 YAML 文件，结果全部通过。
- 对全部项目文件执行尾随空白和 Tab 扫描，未发现问题。
- 安全边界检查确认 playbook 中不存在 reboot、kernel/driver 安装或版本切换任务。
- audit role 中仅使用 facts、读取、只读命令、结果计算、输出和断言任务；Fabric
  状态通过 `ip -j` 的 JSON 输出检查。

## VM 验证

测试使用 Ubuntu 24.04 Minimal cloud image（2026-08-01，kernel
`6.8.0-136-generic`）。镜像经 Ubuntu 官方 `SHA256SUMS` 校验通过。镜像下载使用国内
镜像站，VM 内 APT 使用兰州大学镜像。

宿主机没有可用 KVM，因此使用 QEMU TCG。v0.2 为双向 Fabric 连通测试同时运行两台
VM，并通过低优先级和 CPU 绑定限制影响：

- 1 vCPU，分别固定在单个逻辑 CPU；
- 1536 MiB guest memory；
- `nice 15`；
- management NIC 使用 user-mode network，仅转发
  `127.0.0.1:22221/22222` 到 guest SSH；
- 每台增加一张 `ens4`，通过 QEMU socket backend 组成仅存在于两个 guest 间的隔离
  二层 Fabric，不创建宿主机 bridge/tap、不修改宿主机网络；
- 独立 qcow2 overlay、PID、monitor 和串口日志，全部位于 `ansible/.local/`；
- QEMU 使用 `-no-reboot`。

v0.2 测试使用全新的 qcow2 overlay，结果如下：

| 节点 | Fabric | setup 后复跑 | converge 第一次 | converge 第二次 | 最终 audit |
| --- | --- | --- | --- | --- | --- |
| `n01` | `ens4` = `10.20.0.101/24`, MTU 1500 | `changed=0` | `changed=0` | `changed=0` | `changed=0` |
| `n02` | `ens4` = `10.20.0.102/24`, MTU 1500 | `changed=0` | `changed=0` | `changed=0` | `changed=0` |

两台 Minimal 镜像未预装 `ping`。为避免仅为测试安装额外软件，连通性使用 Python
标准库从本机 Fabric 地址绑定源地址并连接对端 Fabric 地址的 TCP/22；`n01 → n02`
和 `n02 → n01` 均成功。两端审计均确认 `ens4` 为 UP、地址和 MTU 正确，并且没有
default route。

Drift 测试仅修改 `n01` 的受管文件 `90-cluster-fabric.yaml`，将 MTU 从 1500 改为
1400，经 `netplan generate/apply` 确认运行态已漂移。下一次 converge 只修改 `n01`，
恢复模板并依次执行 generate、apply；随后再次 converge 两台均为 `changed=0`。

首次 v0.2 setup 暴露了 Fabric 审计结果列表未初始化的问题。此问题只发生在配置已
成功后的 audit 汇总阶段；使用 `default([])` 初始化后，setup 复跑、独立 audit 和
幂等测试均通过。v0.1 中 `/etc/timezone` 可能陈旧的问题仍通过读取有效
`timedatectl` 状态规避。

测试完成后，两台 guest 均通过 guest 内 `systemctl poweroff` 正常关机。已确认对应
QEMU PID 消失，管理转发端口和 Fabric socket 端口均不再监听；宿主机未执行 reboot
或电源操作。
