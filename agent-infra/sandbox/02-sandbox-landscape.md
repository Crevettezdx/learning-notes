# Agent 沙箱技术全景

[返回目录](./README.md) · [术语速查](./glossary.md)

> 本页问题：如何区分编排管理、运行时接入和执行隔离？

![Agent 沙箱技术路径](./assets/agent-sandbox-technology-path.png)

这张图按职责建立五层骨架：用户如何调用、平台如何管理、节点如何执行、代码如何隔离，以及资源来自哪里。同一层列出的是典型实现，可以按场景选择或组合；纵向箭头表示职责与依赖，不是每个请求都依次经过所有组件。

## 1. 使用与 API

这一层向用户和 Agent 提供统一的沙箱操作，隐藏底层节点与运行时差异。

- **Sandbox API / SDK**：提供创建、连接、执行、文件操作和销毁等能力，例如 E2B 兼容 API。
- **接入与访问代理**：接收 HTTP / RPC 请求，定位目标实例；MCP 或工具调用也可以在这一层封装沙箱能力。

## 2. 编排与生命周期

这一层决定实例放在哪里、分配多少资源、保留多久，并维护实际状态与期望状态的一致性。

- **调度与状态管理**：选择节点，记录实例状态、地址和租约，处理回收、容量与故障恢复。
- **Kubernetes 方案**：CRD 扩展资源类型，Controller 管理生命周期，Kubernetes 调度 Pod；OpenKruise Agents 是这一层的典型实现。
- **平台自有控制面**：使用自己的 API、状态存储和调度服务管理执行节点，不要求采用 Kubernetes CRD。

## 3. 运行时接入与执行管理

这一层将管理请求落实为节点上的执行环境。容器管理器、底层运行时和 VMM 分工不同。

- **containerd / CRI-O**：容器生命周期管理组件；在 Kubernetes 中接收 kubelet 经 CRI 发来的请求。
- **runc / runsc / Kata**：分别代表普通容器执行、gVisor 接入和 VM 隔离容器运行时方案。
- **Kata → VMM**：Kata 对接容器生态，可调用 Cloud Hypervisor、Firecracker、QEMU 等；Guest 内的 kata-agent 负责创建和管理容器。
- **Node Agent → VMM**：平台直接管理 VM 的路线；Node Agent 准备资源，通过 VMM 管理接口创建或恢复实例。
- **环境内执行代理**：例如 envd，负责已经启动的沙箱中的命令、进程与文件操作；与管理 VM 的 Node Agent 分工不同。
- **语言执行引擎**：如 V8 Isolate、Wasmtime，在宿主服务中执行受约束的脚本或 WebAssembly 模块。

> OpenKruise 管理沙箱，Kata 提供容器运行时集成，Cloud Hypervisor 等 VMM 承载虚拟机。三者可以组合，但不等同。Kubernetes 的 RuntimeClass 用于声明运行时选择，本身不创建 VM。

## 4. 执行环境与隔离机制

这一层关注代码与外部环境之间的边界，不能仅用项目名称分类。

典型机制：

- **普通容器**：使用 namespace、seccomp、权限和安全策略等约束进程，共享承载它的 Linux 内核。
- **用户态内核**：如 gVisor，其 Sentry 在用户态处理大量 Linux 系统调用，减少应用直接接触的 Host 内核接口。
- **虚拟机隔离**：每个 VM 有独立 Guest Kernel 与虚拟硬件；Kata 接入和直接管理 VM 都可以采用。microVM 是精简的 VM 形态，不是与 VM 完全独立的一种隔离原理。
- **语言 / 模块隔离**：由 V8 或 WebAssembly 引擎约束内存访问和宿主能力，通常不能直接运行任意 Linux 程序。

这些机制可以叠加，例如在 VM 中运行容器。隔离能力、兼容性和性能需要结合配置与工作负载判断。

## 5. 宿主基础设施与资源池

这一层提供实际计算、内存、网络与存储，多台执行节点组成平台可调度的资源池。

- **Host Linux Kernel**：管理宿主进程、内存、网络与文件系统；cgroup 主要用于资源限制。
- **KVM 与 CPU 虚拟化能力**：在 Linux/KVM 的 VM 路线中协同执行 Guest；VMM 位于 Host 用户态，KVM 位于 Host 内核态。
- **物理硬件与计算节点**：最终资源来自 CPU、内存、磁盘和网卡；节点可以是 BMS，也可以是具备相应能力的云 VM。云 VM 内再运行 VM 时，需要嵌套虚拟化等支持。

## 跨层能力：资源、安全与启动加速

这些能力与上述各层配合，不构成额外的顺序执行层。

- **网络与存储**：提供镜像、模板、系统盘、工作目录、IP、路由和访问代理；Kubernetes 路线中常通过 CNI / CSI 对接网络和存储。
- **安全与运维**：身份与租户授权、凭据管理、网络和文件策略、资源限制、超时回收、监控与审计。
- **预热池**：提前准备可领取的实例，由控制面维护数量和分配。
- **快照恢复**：由支持该能力的运行时及存储保存和恢复运行状态；可与预热池组合，不能视为任意运行时都自带的能力。
- **资源预置与缓存**：提前准备节点、镜像、模板和网络资源，减少请求到来后的准备工作。

选型时分别判断：管理方式、隔离边界、启动加速机制。

参考：[OpenKruise Agents](https://github.com/openkruise/agents)、[Kata 组件](https://github.com/kata-containers/kata-containers#main-components)、[Kata 支持的 VMM](https://github.com/kata-containers/kata-containers/blob/main/docs/hypervisors.md)。

---

相关内容：[Host、Guest、VMM 与 KVM 的位置](./03-host-guest-vmm.md) · [沙箱平台如何直接管理 Cloud Hypervisor](./04-cloud-hypervisor.md)

