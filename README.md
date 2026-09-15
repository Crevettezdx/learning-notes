# Learning Notes

用于持续沉淀个人技术学习笔记。按技术领域组织，每个学习单元聚焦一个问题，以结构图配合精简说明。

## Programming Foundations

- [Python asyncio 学习笔记](./programming/python/asyncio/README.md)：梳理协程、Task、Future、事件循环与完成通知如何协作。

## Agent Infra

- [Agent 沙箱学习笔记](./agent-infra/sandbox/README.md)：12 个独立学习单元，覆盖基础架构、冷启动、预热池、快照与组合策略，另附术语速查和项目引子。

## Model Architecture

- [DeepSeek-V4.1-Flash 架构学习笔记](./model-architecture/deepseek-v4.1-flash/README.md)：围绕长任务 Agent，梳理 CED、CSA2、分层稀疏索引、SWA Bounded Replay、后训练与系统启示。

## 目录约定

```text
learning-notes/
├── README.md
├── programming/
│   └── python/
│       └── asyncio/                  # Python asyncio 运行机制专题
├── agent-infra/
│   └── sandbox/                      # Agent 沙箱专题
└── model-architecture/
    └── deepseek-v4.1-flash/          # 模型架构专题
        ├── README.md                 # 主题目录与阅读路径
        ├── 00-论文总览.md
        ├── …                         # 独立学习单元
        └── assets/                   # 正文引用图片
```

- 每个主题使用独立目录，主题 README 作为入口。
- 每个学习单元单独成文，只解释一个问题，附返回目录和相关内容链接。
- 图片等配套资源放在同级 assets/ 目录，由对应学习单元引用。
