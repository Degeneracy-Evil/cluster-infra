# v0.3 验证记录

验证日期：2026-08-17

## 静态检查

- 所有开发、缓存、临时文件和日志均限制在项目目录内。
- Python 工具继续由仓库内 `uv` 环境管理：`ansible-core 2.21.3`、
  `ansible-lint 26.8.0`。
- `example` 和 `test` inventory graph 均通过。
- `setup.yml`、`converge.yml`、`audit.yml`、`apply-versions.yml` 的 syntax-check
  全部通过。
- `ansible-lint` 通过：0 failure、0 warning，并达到 production profile。
- 项目版本和 `uv.lock` 已同步更新到 `0.3.0`。
- `setup.yml` 与 `converge.yml` 不引用 Driver apply；项目不存在 reboot task，也不
  实现 kernel 或 CUDA Toolkit 安装。

## QEMU 环境

测试继续使用经官方 SHA256 校验的 Ubuntu 24.04 Minimal cloud image（2026-08-01，
kernel `6.8.0-136-generic`），VM 内 APT 使用兰州大学镜像。v0.3 使用两张全新的
qcow2 overlay：

- 两台 VM 各 1 vCPU、1536 MiB 内存、`nice 15`，分别绑定单个逻辑 CPU；
- management NIC 使用 QEMU user-mode network，只将 SSH 转发到
  `127.0.0.1:22221/22222`；
- Fabric NIC 使用 QEMU socket backend 构成 guest 间隔离二层网络；
- 不创建宿主机 bridge/tap，不修改宿主机接口、路由或配置；
- QEMU 使用 `-no-reboot`，全部 overlay、PID、monitor 和日志位于
  `ansible/.local/`。

## VM 功能与幂等结果

全新 VM 的 `setup.yml` 在两台节点均成功，单节点 recap 为 `changed=10`。内置 audit
和后续独立 audit 得到一致硬件结果：

| 项目 | `n01` | `n02` |
| --- | --- | --- |
| CPU | QEMU TCG，1 socket / 1 core / 1 vCPU | QEMU TCG，1 socket / 1 core / 1 vCPU |
| Memory | 1463 MiB | 1463 MiB |
| Root filesystem | 6.71 GiB | 6.71 GiB |
| Kernel | `6.8.0-136-generic`，baseline OK | `6.8.0-136-generic`，baseline OK |
| Kernel headers | MISSING，仅报告 | MISSING，仅报告 |
| NVIDIA GPU | absent，合法 | absent，合法 |
| NVIDIA Driver | unloaded / unavailable | unloaded / unavailable |
| CUDA Toolkit | absent，合法 | absent，合法 |
| Fabric | `ens4`、CIDR、MTU、无默认路由均 OK | `ens4`、CIDR、MTU、无默认路由均 OK |

setup 后连续两次执行 `converge.yml`，两次均在 `n01` 和 `n02` 达到
`changed=0`。独立 `audit.yml` 两台均为 `changed=0`，CPU-only 状态没有导致失败。

## Fabric 空配置回归

初始 inventory 为两台节点声明 `ens4` Fabric 地址，setup 后确认
`90-cluster-fabric.yaml` 存在且地址生效。随后仅在忽略的 VM inventory 中将两台
`fabric_interfaces` 改为空列表：

- converge 只删除本项目管理的 Netplan 文件；
- `netplan generate` 成功后执行 apply；
- `10.20.0.101/24` 和 `10.20.0.102/24` 均从 `ens4` 撤销；
- 管理网络和其他 Netplan 文件不变；
- 再次 converge 两台均为 `changed=0`。

接口仍可能保留自动生成的 IPv6 link-local 地址，这不属于本项目声明的持久 Fabric
配置。

## Driver apply 安全测试

在 CPU-only 的 `n01` 上以精确 `--limit` 和显式测试 package 调用
`apply-versions.yml`。playbook 正确检测到：

```text
gpu_detection=True
gpu_present=False
kernel_recognized=True
kernel_headers=False
```

任务在安装前置断言处清晰失败，recap 为 `changed=0`；APT candidate 查询和安装任务
均未执行。VM 中没有安装 Driver、CUDA Toolkit 或 kernel，也没有执行 reboot。

## 尚待真实硬件验证

仓库当前只有 `example` 和使用 RFC 5737 地址的 `test` inventory，没有三台真实机器
的连接 inventory，因此尚未执行真实机器 audit。真实 NVIDIA Driver apply 也未获得
明确的目标节点和 package，未执行任何真实 Driver 变更。

取得真实 inventory 后，第一步只运行 `audit.yml`，用于验证异构 CPU、GPU 型号与
数量、Driver version/branch、CUDA Toolkit、kernel headers 和 Fabric 检测。只有用户
另行明确目标节点与 package 后，才考虑单机 Driver apply。

## 清理确认

两台 guest 均通过 guest 内 `systemctl poweroff` 正常关机。对应 QEMU 进程、两个
SSH 转发端口和 Fabric socket 端口均已确认消失；宿主机未执行 reboot 或电源操作。
