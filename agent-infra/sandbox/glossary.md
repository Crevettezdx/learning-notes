# 核心术语与协作关系

[返回目录](./README.md)

> 本页问题：这些组件分别负责什么，彼此怎样配合？

每条只回答两个问题：**它负责什么？它与谁配合？** 可以先读“虚拟机与沙箱执行”这一组，再按需查阅其他术语。

## 基础：机器、内核与进程

- **BMS（Bare Metal Server，裸金属服务器）**：提供物理 CPU、内存、磁盘和网卡；上面安装 Host 操作系统，再承载容器或 VM。
- **Node（节点）**：平台可调度的一台计算机器，可以是物理机或云 VM；调度器选中它后，由节点上的管理程序落实任务。
- **VM / microVM（虚拟机 / 轻量虚拟机）**：拥有虚拟硬件和独立操作系统的执行环境；microVM 通常精简设备和启动流程，二者都由 VMM 管理。
- **Host / Guest（宿主 / 客体）**：Host 是承载 VM 的机器及其系统，Guest 是 VM 内的系统；VMM 在 Host 侧运行，用户代码可以在 Guest 内运行。
- **Kernel（内核）**：操作系统管理进程、内存和设备的核心；Host Kernel 管宿主机资源，Guest Kernel 管 VM 内的资源。
- **用户态 / 内核态**：程序运行的两种权限层级。应用、VMM、envd 属于相应系统的用户态；Host Linux Kernel 和 KVM 属于 Host 内核态，Guest Kernel 属于 Guest 内核态。
- **进程 / 系统调用（syscall）**：进程是运行中的程序；它通过系统调用请求所属操作系统内核提供文件、网络、内存等服务。

## 虚拟机与沙箱执行：谁管理谁

- **Sandbox（沙箱）**：平台提供的受约束执行环境，通常有实例 ID、资源限制和生命周期；底层可以是容器、VM 或语言执行环境。
- **控制面 / Scheduler（调度器）**：控制面维护沙箱状态与策略，调度器负责选择执行节点；选定后将任务交给 Node Agent 或 Kubernetes 节点组件。
- **Node Agent（节点代理）**：本文指沙箱节点上的守护程序，是通用角色名；接收控制面指令，准备网络和磁盘，再调用 VMM 创建、恢复或销毁 VM。
- **VMM（Virtual Machine Monitor，虚拟机监控器）**：配置并管理 VM 的 vCPU、内存和虚拟设备；在本文 Linux/KVM 路线中运行于 Host 用户态，调用 KVM 执行 Guest。
- **KVM（Kernel-based Virtual Machine）**：Host Linux Kernel 中的虚拟化模块；接收 VMM 的调用，借助 CPU 硬件虚拟化能力运行 Guest，不负责整个沙箱资源池的调度。
- **Cloud Hypervisor / Firecracker / QEMU**：VMM 的具体实现；Node Agent 可以直接调用它们，Kata 也可以选用其中的实现。
- **vCPU / Guest RAM**：Guest 看到的虚拟 CPU 和内存；由 VMM 配置，并由 KVM、Host 内核及物理资源共同支撑。
- **envd（Guest Agent）**：E2B 的环境内代理，负责命令、进程和文件操作；访问代理将请求送给它，它在 Guest 内操作，不创建自己的 VM。
- **init**：Linux 启动后的首个用户态进程，通常负责启动和管理服务；Guest 内的 envd 等程序由它或其启动链拉起。

常见协作：**控制面选节点 → Node Agent 准备资源 → VMM 调用 KVM 运行 Guest → envd 接收操作请求 → 用户进程执行。** 这是职责关系；创建实例与执行命令是两条不同的请求路径。

## Kubernetes 与容器：如何接入执行环境

- **容器 / Pod**：容器是隔离运行的进程环境，Pod 是 Kubernetes 的最小调度单元，可包含多个容器；普通容器共享所在系统的内核，Kata 可将 Pod 的容器放进 VM。
- **kubelet**：每个 Kubernetes Worker Node 上的节点管理程序；接收已分配到本节点的 Pod，调用容器管理器创建并维护它。
- **CRI（Container Runtime Interface）**：kubelet 调用容器管理器的标准接口；常见实现方是 containerd 和 CRI-O。
- **containerd / CRI-O**：管理镜像和容器生命周期；在 Kubernetes 中接收 CRI 请求，再交给配置好的底层运行时。
- **runc / runsc**：具体执行运行时；runc 创建普通 Linux 容器，runsc 将容器接入 gVisor。
- **gVisor / Sentry**：gVisor 提供沙箱运行时，Sentry 是其中处理大量系统调用的用户态内核；它在应用与 Host Kernel 之间增加一道边界。
- **Kata Containers / kata-agent**：Kata 将容器请求落实为 VM 内的容器；Host 侧运行时管理 VMM，Guest 内的 kata-agent 创建、管理容器。它与处理用户命令的 envd 职责不同。
- **CRD / Controller**：CRD 为 Kubernetes 增加资源类型，Controller 持续把资源的期望状态变为实际状态；沙箱平台可以用它们管理实例和预热池。
- **RuntimeClass**：Kubernetes 中声明 Pod 所用运行时配置的资源；配合节点运行时配置选择 runc、Kata 等，不亲自启动容器或 VM。
- **OpenKruise Agents / SIG Agent-Sandbox**：两个不同的 Kubernetes 沙箱管理项目；它们属于编排管理层，与 Kata、VMM 可以分层组合。
- **namespace / cgroup / seccomp**：分别隔离进程看到的资源视图、限制资源用量、过滤系统调用；容器运行时将它们与权限等机制配合使用。

## 文件、网络与环境

- **镜像 / 模板**：创建环境时使用的预制材料；OCI 镜像通常提供容器程序与依赖，VM 模板还需配套 Guest 内核和虚拟设备配置。
- **RootFS / Workspace**：RootFS 是环境看到的根文件系统，Workspace 是用户工作目录；由存储后端提供，并被容器或 Guest 中的程序访问。
- **virtio**：Guest 与虚拟设备交互的一组标准接口；Guest 驱动通过它访问 VMM 或其他后端提供的磁盘、网络等能力。
- **TAP / vsock**：TAP 在 Host 侧连接虚拟网卡与宿主网络；vsock 提供 Host 与 Guest 之间的通信通道，可用于代理交互，是否采用取决于实现。
- **CNI / CSI**：Kubernetes 生态对接网络和存储的接口体系；相应插件为 Pod 配置网络、供应或挂载存储。
- **V8 Isolate / WebAssembly / Wasmtime**：V8 Isolate 提供引擎内的执行隔离；WebAssembly 是受约束的模块格式，Wasmtime 是执行它的运行时，均需由宿主服务限制可调用的外部能力。

## 启动加速与生命周期

- **WarmPool（预热池）**：提前保留可领取的实例；控制面负责领取和补池，底层仍由运行时完成实例启动。
- **Snapshot / Restore（快照 / 恢复）**：保存并恢复运行状态；VM 快照涉及内存、CPU 与设备状态，磁盘一致性还需存储和平台配合，只有磁盘快照不能直接恢复进程执行点。
- **CoW / reflink（写时复制 / 文件克隆）**：CoW 先共享数据，写入时再生成私有副本；支持 reflink 的文件系统可按此方式克隆文件，用于减少模板或磁盘复制成本。
- **Probe / Ready（探测 / 就绪）**：Probe 检查服务能否正常工作，Ready 表示满足平台的可用条件；进程已启动不一定代表沙箱可以交付。
- **TTL（Time To Live，存活时限）**：实例或租约允许保留的时间；由控制面续期或到期回收，与运行时协作停止并清理环境。

---

相关内容：[Host、Guest、VMM 与 KVM 的位置](./03-host-guest-vmm.md) · [Agent 沙箱技术全景](./02-sandbox-landscape.md)

