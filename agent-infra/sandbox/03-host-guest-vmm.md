# Host、Guest、VMM 与 KVM 的位置

[返回目录](./README.md) · [术语速查](./glossary.md)

> 本页问题：用户进程、两套内核和虚拟化组件分别在哪里运行？

![microVM 与三种启动路径](./assets/sandbox-startup-acceleration.png)

本页只看图的上半部分：microVM 的空间关系。下半部分的启动方式见[预热池](./06-warm-pool.md)和[快照恢复](./09-snapshot-restore.md)。

- **Guest 用户态**：Agent、Python、Shell、Browser 等工作负载。
- **Guest Kernel**：microVM 内部自己的 Linux 内核，管理 Guest 的进程、内存、网络和文件系统。
- **VMM**：宿主机用户态中的虚拟机管理程序，负责创建 vCPU、映射内存并提供虚拟设备。
- **KVM**：Host Linux Kernel 中的虚拟化模块，借助 CPU 虚拟化能力执行 Guest 指令。
- **物理硬件**：CPU、内存、网卡和磁盘。

普通容器没有独立 Guest Kernel，容器中的系统调用直接进入 Host Kernel；microVM 多了一层独立内核和虚拟硬件边界，隔离更强，也多出 VM 启动与恢复问题。

> 核心认识：VMM 负责“管理虚拟机”，KVM 与 CPU 负责“执行虚拟机”。

---

相关内容：[沙箱平台如何直接管理 Cloud Hypervisor](./04-cloud-hypervisor.md) · [云上 VM 如何承载 Kubernetes 容器](./01-kubernetes-on-vm.md)

