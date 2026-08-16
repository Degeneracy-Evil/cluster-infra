# v0.1 验证记录

验证日期：2026-08-16

## 当前环境中的验证

- 所有操作均限制在项目目录内。
- 未安装宿主机系统软件，也未修改宿主机配置。
- 使用 `uv` 在仓库内创建隔离环境：`ansible-core 2.21.3`、
  `ansible-lint 26.8.0`。
- `example` 和 `test` inventory graph 均通过，节点分组符合设计。
- 四个 playbook 的 Ansible syntax-check 全部通过。
- `ansible-lint` 通过：0 failure、0 warning，并达到 production profile。
- 使用 Ruby 标准库解析仓库中的全部 YAML 文件，结果全部通过。
- 对全部项目文件执行尾随空白和 Tab 扫描，未发现问题。
- 安全边界检查确认 playbook 中不存在 reboot、kernel/driver 安装或版本切换任务。
- audit role 中仅使用 facts、读取、只读命令、结果计算、输出和断言任务。

## VM 验证

测试使用 Ubuntu 24.04 Minimal cloud image（2026-08-01，kernel
`6.8.0-136-generic`）。镜像经 Ubuntu 官方 `SHA256SUMS` 校验通过。镜像下载使用国内
镜像站，VM 内 APT 使用兰州大学镜像。

宿主机没有可用 KVM，因此两台 VM 按顺序使用 QEMU TCG 运行，每次只运行一台：

- 1 vCPU，固定在单个逻辑 CPU；
- 1536 MiB guest memory；
- `nice 15`；
- user-mode network，仅转发 `127.0.0.1:22221/22222` 到 guest SSH；
- 独立 qcow2 overlay、PID、monitor 和串口日志，全部位于 `ansible/.local/`；
- QEMU 使用 `-no-reboot`。

测试结果：

| 节点 | setup | audit | converge 第一次 | converge 第二次 |
| --- | --- | --- | --- | --- |
| `n01` | 基础配置 `changed=8` | `changed=0`，全部通过 | `changed=0` | `changed=0` |
| `n02` | `changed=8`，内置 audit 通过 | setup 内通过 | `changed=0` | `changed=0` |

`n01` 首次 setup 暴露了一个 audit 缺陷：Ubuntu 24.04 Minimal 的有效 timezone 已由
systemd 修改，但旧 `/etc/timezone` 文本未同步。audit 已改为通过只读
`timedatectl show --property=Timezone` 检查系统有效值；修复后 syntax-check、lint 和
真实 audit 均通过。

测试完成后，两台 guest 均通过 guest 内 `systemctl poweroff` 正常关机。已确认对应
QEMU PID 消失，两个本地转发端口均不再监听；宿主机未执行 reboot 或电源操作。
