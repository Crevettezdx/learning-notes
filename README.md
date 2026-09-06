# Learning Notes

用于持续沉淀个人技术学习笔记。按技术领域组织，每个学习单元聚焦一个问题，以结构图配合精简说明。

## Agent Infra

- [Agent 沙箱学习笔记](./agent-infra/sandbox/README.md)：12 个独立学习单元，覆盖基础架构、冷启动、预热池、快照与组合策略，另附术语速查和项目引子。

## 目录约定

```text
learning-notes/
├── README.md
└── agent-infra/
    └── sandbox/
        ├── README.md                 # 主题目录与阅读路径
        ├── glossary.md               # 术语速查
        ├── 01-kubernetes-on-vm.md     # 独立学习单元
        ├── …                         # 其余学习单元
        ├── appendix-opensandbox.md   # 项目附录
        └── assets/                   # 共用图片
```

- 每个主题使用独立目录，主题 README 作为入口。
- 每个学习单元单独成文，只解释一个问题，附返回目录和相关内容链接。
- 图片等配套资源放在同级 assets/ 目录，由对应学习单元引用。
