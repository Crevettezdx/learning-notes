# 项目引子：OpenSandbox

[返回目录](./README.md) · [术语速查](./glossary.md)

> 本页问题：统一沙箱接口的平台，在整体架构中处于什么位置？

![OpenSandbox 与 CubeSandbox 的架构定位](./assets/opensandbox-vs-cubesandbox.png)

OpenSandbox 代表的是一种偏横向的平台取向：先统一 Sandbox API、生命周期和调用协议，再接入 Docker、Kubernetes 以及不同隔离 Runtime。

它主要回答：

> 如何用一致的方式创建、调用、暂停、恢复和销毁不同类型的沙箱？

OpenSandbox 的平台抽象可以与不同执行后端配合。是否进入 Guest Kernel、VMM 和 KVM 路径，取决于实际接入的运行时与部署配置。

项目：[alibaba/OpenSandbox](https://github.com/alibaba/OpenSandbox)。

---

相关内容：[Agent 沙箱技术全景](./02-sandbox-landscape.md)

