# Cluster Infra v0.2 Network 开发任务

## 1. 开发目标

在现有 v0.1 基础上增加 **计算 Fabric 网络配置能力**。

当前阶段只管理计算节点上的独立高速网络接口，例如：

* RoCE Ethernet rail
* InfiniBand IPoIB 接口
* 其他独立计算网络

管理网络继续由现有系统配置，未来主要交给 MAAS。Ansible 的 network role 应避免修改当前 SSH 所使用的管理接口。

本阶段保持现有策略：

* Ubuntu 24.04
* Inventory 为配置入口
* Ansible 严格幂等
* `setup.yml` 用于首次配置
* `converge.yml` 用于安全日常收敛
* `audit.yml` 只读检查
* 暂不实现 MAAS、NAT、GPU、RDMA tuning、Slurm、K8s、Prometheus

---

## 2. Git 分支

当前有效开发基线为：

```text
agent/ansible-base
```

请基于该分支创建：

```text
agent/network
```

如果执行开发时 `agent/ansible-base` 已经正式合入 `main`，则从包含该提交的最新 `main` 创建 `agent/network`。

确认不要从缺少 v0.1 实现的旧 `main` 开发。

---

## 3. Inventory 接口

在 host 变量中显式描述 Fabric 接口。

示例：

```yaml
all:
  children:
    compute:
      hosts:
        n01:
          ansible_host: 192.168.122.101
          fabric_interfaces:
            - name: ens4
              address: 10.20.0.101/24
              mtu: 1500

        n02:
          ansible_host: 192.168.122.102
          fabric_interfaces:
            - name: ens4
              address: 10.20.0.102/24
              mtu: 1500
```

真实集群未来可能使用：

```yaml
fabric_interfaces:
  - name: enp65s0f0
    address: 10.20.0.101/24
    mtu: 9000

  - name: enp65s0f1
    address: 10.21.0.101/24
    mtu: 9000
```

第一版保持字段简单：

```text
name
address
mtu
```

`fabric_interfaces` 未定义或为空时，network role 应安全跳过配置。

IP 和接口名保持显式配置，暂不增加 hostname → IP 自动计算逻辑。

---

## 4. Network Role

实现：

```text
ansible/roles/network/
```

建议包含：

```text
roles/network/
├── defaults/main.yml
├── tasks/main.yml
├── handlers/main.yml
└── templates/
    └── 90-cluster-fabric.yaml.j2
```

### 管理范围

network role 只管理：

```text
fabric_interfaces
```

中声明的接口。

生成独立 Netplan 文件：

```text
/etc/netplan/90-cluster-fabric.yaml
```

保留 MAAS、cloud-init 或人工创建的其他 Netplan 文件。

Fabric 接口默认具有以下属性：

* 静态地址
* 指定 MTU
* 无 gateway
* 无 DNS
* 无 default route

生成结果类似：

```yaml
network:
  version: 2
  ethernets:
    ens4:
      addresses:
        - 10.20.0.101/24
      mtu: 1500
```

多 rail 应在同一项目管理文件中生成。

模板输出必须稳定，确保相同 inventory 多次执行不会产生无意义变化。

---

## 5. 配置安全

写入新的 Netplan 配置后，先执行配置验证。

至少要求：

```bash
netplan generate
```

成功后才能应用网络配置。

只有配置文件实际发生变化时才执行：

```bash
netplan apply
```

使用 handler 实现。

配置错误时 play 应失败，并保留清晰错误信息。

network role 不主动：

* 修改 management NIC
* 修改默认路由
* 修改 DNS
* 删除其他 Netplan 文件
* 配置 NAT
* 配置 PFC / ECN
* 配置 RDMA sysctl
* 配置 NCCL / UCX

---

## 6. Playbook 集成

将 network role 接入现有：

```text
setup.yml
converge.yml
```

顺序建议：

```text
base
→ network
→ audit
```

`converge.yml` 仍然保持可以安全重复执行。

配置没有变化时不得执行 `netplan apply`。

---

## 7. Audit 扩展

扩展现有 audit role，对每个 `fabric_interfaces` 条目进行只读检查。

至少检查：

1. 接口是否存在。
2. 接口是否处于可用状态。
3. 声明的 IP/CIDR 是否存在。
4. MTU 是否与 inventory 一致。
5. Fabric 接口没有产生 default route。

审计只读取状态：

```text
changed=0
```

建议优先使用：

```text
ip -j ...
```

等结构化输出，减少对人类可读文本格式的依赖。

最终 audit 摘要中应能够快速看到类似：

```text
n01
  ens4:
    address: OK
    mtu: OK
    default_route: OK

n02
  ens4:
    address: OK
    mtu: OK
    default_route: OK
```

现有 base audit 行为保持不变。

---

## 8. QEMU 测试环境

继续复用现有 Ubuntu 24.04 Minimal QEMU 测试体系。

为 `n01` 和 `n02` 各增加一张额外虚拟 NIC：

```text
n01
├── management NIC → SSH
└── fabric NIC

n02
├── management NIC → SSH
└── fabric NIC
```

管理 NIC 保持现有 SSH 连接方式。

Fabric NIC 放入同一个隔离二层网络，使：

```text
n01 fabric: 10.20.0.101/24
n02 fabric: 10.20.0.102/24
```

两台 VM 可以通过 Fabric 地址互相通信。

测试环境仍需满足：

* 不修改宿主机网络配置。
* 不影响宿主机默认路由。
* 测试相关文件保存在 Git 忽略的 `.local/`。
* 测试结束后正常关闭 VM 并确认相关 QEMU 进程消失。

---

## 9. 必须完成的测试

### 基础检查

执行：

```text
inventory graph
syntax-check
ansible-lint
```

要求全部通过。

### 首次配置

在干净 VM 上执行：

```bash
ansible-playbook \
  -i inventories/test \
  playbooks/setup.yml
```

确认 Fabric IP 和 MTU 正确。

### Fabric 连通性

验证：

```text
n01 → 10.20.0.102
n02 → 10.20.0.101
```

能够正常通信。

### 幂等性

连续执行两次：

```bash
ansible-playbook \
  -i inventories/test \
  playbooks/converge.yml
```

第二次要求：

```text
changed=0
```

### Drift 恢复

人工在 VM 内修改至少一种受管理状态，例如：

```text
错误 MTU
```

或移除声明的 Fabric 地址。

重新执行：

```bash
ansible-playbook \
  -i inventories/test \
  playbooks/converge.yml
```

确认配置恢复到 inventory 声明状态。

之后再次执行 converge，要求：

```text
changed=0
```

### Audit

执行：

```bash
ansible-playbook \
  -i inventories/test \
  playbooks/audit.yml
```

要求：

```text
changed=0
```

所有 Fabric 检查通过。

---

## 10. 文档更新

更新：

```text
docs/design.md
docs/testing.md
README.md
```

只记录本阶段实际已经实现和验证的行为。

`docs/design.md` 增加以下架构原则：

```text
Management network:
    MAAS / existing system owns configuration
    Ansible audits where appropriate

Fabric network:
    Ansible owns explicitly declared fabric_interfaces
```

记录 Ansible 使用独立：

```text
90-cluster-fabric.yaml
```

管理 Fabric 网络。

---

## 11. 当前阶段完成标准

本阶段完成后应具备：

```text
base       ✓
network    ✓
audit      ✓
```

能够通过 inventory 将两台 Ubuntu 24.04 VM 配置为：

```text
management network
+
独立 fabric network
```

同时满足：

```text
setup 成功
fabric 双向连通
drift 可以收敛
第二次 converge changed=0
audit changed=0
ansible-lint 0 failure / 0 warning
```

完成这些工作后停止扩展功能。

暂缓：

```text
NVIDIA / CUDA
kernel 版本切换
RDMA 软件栈
RoCE PFC / ECN
NAT
MAAS
Slurm
Kubernetes
Prometheus
```

提交开发结果、测试记录和发现的问题，等待下一阶段任务。
