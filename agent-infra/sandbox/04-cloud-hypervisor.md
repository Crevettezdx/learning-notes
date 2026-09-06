# 沙箱平台如何直接管理 Cloud Hypervisor

[返回目录](./README.md) · [术语速查](./glossary.md)

> 本页问题：API 请求如何变成节点上的一个独立沙箱 VM？

![Cloud Hypervisor 沙箱架构](./assets/cloud-hypervisor-sandbox-architecture.png)

这条路线由沙箱平台选择计算节点，再由节点上的 Cloud Hypervisor 创建或恢复沙箱 VM。云厂商维护资源池，用户通过 API 使用沙箱，无需自行管理节点，因此可以提供 Serverless 形态。

图中采用 **E2B 兼容 API + envd + Cloud Hypervisor / KVM** 的概念组合，不代表 E2B 官方现有部署。Cloud Hypervisor 是面向云工作负载的 VMM，这里展示其用于轻量沙箱的配置。

## 1. 共享资源池与物理硬件

资源池把多台计算节点组织成可调度容量；每个节点的物理硬件是 VM 资源的来源。图顶部展示资源池，节点底部展示硬件，二者是集合与成员的关系。

- **BMS（Bare Metal Server）**：裸金属物理服务器；本图一个计算节点对应一台 BMS。
- **CPU、内存、磁盘和网卡**：提供实际计算、数据保存和通信能力；CPU 的硬件虚拟化功能支撑 Guest 执行。
- **资源池管理**：记录节点健康与可用容量，按需要扩缩容；它是平台控制面的职责。

## 2. Host 内核态

宿主机的 Linux 内核管理物理资源，为节点上的多个 VMM 提供虚拟化、网络和存储能力。

- **KVM**：Linux 内核中的虚拟化模块，借助 CPU 硬件能力执行 Guest。
- **TAP / 网络栈**：连接 VM 虚拟网卡与宿主机网络，配合路由和策略实现通信。
- **cgroup**：对 VMM 等宿主机进程实施 CPU、内存等资源限制。
- **文件系统与块设备**：承载系统盘文件、快照数据和存储后端。

## 3. Host 用户态

这一层运行宿主机上的管理程序。它与第 2 层属于同一个节点、同一个 Host 操作系统。

- **Node Agent**：接收平台任务，准备网络、磁盘和模板，管理节点内多个 VMM 的生命周期。
- **Cloud Hypervisor（VMM）**：配置虚拟 CPU、内存和设备，并调用 KVM 运行 Guest；本方案每个 VMM 进程承载一个沙箱 VM。
- **VMM 管理 API**：Node Agent 控制 Cloud Hypervisor 的接口，可通过本地 Unix socket 调用，用于创建、启动、暂停、快照、恢复等操作。

## 4. 沙箱 VM：Guest 内核与虚拟硬件

每个 VM 提供独立的操作系统执行环境。虚拟硬件位于 Guest 内核下方，为它提供可用的计算与设备接口。

- **Guest Linux Kernel**：管理该 VM 内的进程、内存和设备；不同 VM 有各自的 Guest 内核。
- **vCPU / Guest RAM**：VM 可见的 CPU 与内存，最终由宿主机的物理资源支撑。
- **virtio 设备**：让 Guest 高效访问虚拟磁盘、网卡等设备；需要 Guest 内核具备相应驱动。

## 5. Guest 沙箱环境

这一层位于 VM 的用户态，为执行命令、操作文件和运行应用准备基础环境。

- **envd**：运行在 Guest 内的代理，处理命令、进程和文件请求；不负责创建或调度 VM。
- **init**：Guest 启动后的首个用户态进程，负责启动和管理 envd 等服务。
- **语言与工具环境**：Python、Node.js、Shell、浏览器等，决定沙箱开箱可运行哪些任务。
- **RootFS / Workspace**：分别提供系统根文件系统和用户工作目录；Workspace 是否持久化取决于配置。

## 6. 用户工作负载

这一层运行用户提交的代码、命令和服务，与 envd 同处 Guest 用户态。

- **用户进程**：实际执行 Python、Shell、浏览器或业务服务；一个 Sandbox 可运行多个进程。
- **Guest 系统调用**：用户进程通过 Guest 内核申请内存、读写文件和访问网络；需要磁盘或网络 I/O 时，再经虚拟设备连接宿主机后端。

## 侧边：API、调度与生命周期

控制面管理整个资源池；运行期代理把请求送到已经存在的沙箱。

- **E2B 兼容 API**：向客户端提供创建、查询、续期和销毁等接口；兼容性由平台实现。
- **调度与编排**：根据资源、模板等条件选择节点，并向 Node Agent 下发任务。
- **状态与容量管理**：维护 Sandbox 到节点和地址的映射，处理配额、TTL、回收、扩缩容及模板版本。
- **运行期访问代理**：定位目标 Sandbox，将命令和文件请求转发给其中的 envd。

两条主要请求路径：

- **创建 / 恢复**：客户端 → 平台 API → 调度与编排 → Node Agent → Cloud Hypervisor → KVM / Guest。
- **执行命令 / 文件操作**：客户端 → 运行期代理 → envd → Guest 内的进程或文件。

## 侧边：模板、快照、存储与网络

这些资源由 Host 侧准备、加载或绑定，为 VM 提供启动材料和外部连接。

- **模板制品**：Guest 内核、系统盘及预装环境，定义沙箱初始配置。
- **快照制品**：保存 VM 配置、Guest 内存、CPU 与设备状态；平台还需配套管理一致的磁盘状态。
- **存储后端**：提供系统盘、实例私有写入空间与可选的持久化工作目录。
- **网络资源与策略**：提供 IP、TAP、路由、服务访问和出网控制。

> 替换 VMM 时，Node Agent 管理接口、Guest 模板及设备配置需要适配；envd 可以保留但需验证接入。Firecracker 快照不能直接用于 Cloud Hypervisor，需要重新生成。

数量关系：**1 个资源池 → N 个节点；1 个节点 → N 个 VMM；1 个 VMM → 1 个沙箱 VM；通常 1 个 VM → 1 个 Sandbox → 多个用户进程。** 各处的 N 可以不同，实际数量受资源和策略限制。

参考：[Cloud Hypervisor 官方说明](https://github.com/cloud-hypervisor/cloud-hypervisor#1-what-is-cloud-hypervisor)、[快照恢复文档](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/snapshot_restore.md)。

---

相关内容：[Host、Guest、VMM 与 KVM 的位置](./03-host-guest-vmm.md) · [从零启动一个可用沙箱](./05-cold-start.md)

