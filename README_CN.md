# CC4CC - Claude Code 源码解析 Index

> **仅供教育分析** - 本仓库不包含任何已公开制品的源代码，仅包含用于教育目的的分析。本文档不包含任何实际代码。

**[English](./README.md) | 中文**



---

## 背景

在 **2026年3月31日**，Claude Code 通过 npm 分发包中的 source map 泄露，公开了最新版本的源代码。

虽然我们对这一失误感到遗憾，但许多研究人员对这款**可能是当前最佳编程代理**的设计方式充满好奇。

为了方便社区并避免各类教程的混乱，本仓库旨在收集使用 Claude Code 自身来分析其设计的现有分析成果。

### 扩展分析索引

每个模块都有一个 `analysis/` 子目录，用于存放社区对该模块的深度分析。主 [analysis/](./analysis/) 目录提供社区整体分析。

| 标题 | 作者 | 外部分析链接 |Description |
|----|-------|--|-------------|
  | How Claude Code Works | Windy3f |[link](https://github.com/Windy3f3f3f3f/how-claude-code-works)||
| i read claude code's entire source code. here's what anthropic got wrong.  | Rohan | [link](https://x.com/rohan_2502/status/2038927786228998194) |
| Claude Code v2.1.88 源码逆向分析 | 秋实 | [link](http://xhslink.com/o/8YaWaT3ZfDr)|  |
| - | - | - | *Waiting for contributions* |

---

## 目录结构

目录包含每个关键模块的详细分析。每个子目录专注于特定的架构组件：

```
├── README.md                 # 本文件 - 概览与导航
├── README_CN.md              # 中文版
├── analysis/                 # 扩展分析索引（所有模块）
│
├── tool-system/              # 工具系统架构分析
│   ├── README.md
│   └── analysis/             # 本模块的扩展分析
│
├── command-system/           # 斜杠命令系统分析
│   ├── README.md
│   └── analysis/
│
├── service-layer/            # 外部服务集成
│   ├── README.md
│   └── analysis/
│
├── bridge-system/            # IDE 集成桥接
│   ├── README.md
│   └── analysis/
│
├── permission-system/        # 权限与安全模型
│   ├── README.md
│   └── analysis/
│
├── context-management/       # 上下文窗口与内存管理
│   ├── README.md
│   └── analysis/
│
├── multi-agent/              # 多代理协调
│   ├── README.md
│   └── analysis/
│
├── plugin-system/            # 插件扩展性
│   ├── README.md
│   └── analysis/
│
├── skill-system/             # 技能与工作流系统
│   ├── README.md
│   └── analysis/
│
└── ui-architecture/          # 终端 UI 架构
    ├── README.md
    └── analysis/
```

---

## 模块概览

### [工具系统](./tool-system/)
所有代理操作的核心抽象。每个工具都是一个自包含模块，包含输入模式、权限模型和执行逻辑。
- **关键洞察**：动态注册支持延迟发现和懒加载

### [命令系统](./command-system/)
用户面向的斜杠命令，用于控制 Claude Code。
- **关键洞察**：集中式注册表配合基于模式的参数解析

### [服务层](./service-layer/)
外部集成，包括 API、MCP、OAuth 和 LSP 服务。
- **关键洞察**：抽象层隔离外部依赖

### [桥接系统](./bridge-system/)
CLI 与 IDE 扩展之间的双向通信。
- **关键洞察**：协议优先设计支持多种 IDE 客户端

### [权限系统](./permission-system/)
控制操作何时需要用户批准的安全层。
- **关键洞察**：声明式权限配合语义匹配

### [上下文管理](./context-management/)
Token 预算管理、压缩和内存持久化。
- **关键洞察**：基于优先级的保留策略保护关键信息

### [多代理系统](./multi-agent/)
子代理生成、团队管理和代理间通信。
- **关键洞察**：上下文隔离支持安全的并行工作

### [插件系统](./plugin-system/)
用于添加工具、命令和钩子的扩展框架。
- **关键洞察**：能力声明支持安全的插件加载

### [技能系统](./skill-system/)
用于领域特定功能的打包工作流。
- **关键洞察**：优秀的提示词是有效技能的核心

### [UI 架构](./ui-architecture/)
使用 Ink（终端版 React）构建的终端 UI。
- **关键洞察**：组件架构适合复杂 UI 的扩展

---

## 架构概要

### 高层架构

系统遵循**中心辐射架构**：

- 中央查询引擎协调 LLM 交互
- 模块化工具系统支持扩展
- 命令系统提供用户面向的功能
- 服务层处理外部集成
- 桥接系统实现 IDE 连接

### 关键设计模式

| 模式 | 应用 |
|------|------|
| 工具抽象 | 每个操作都是自包含的工具 |
| 权限优先 | 安全性内置于核心 |
| 上下文管理 | 对长运行会话至关重要 |
| 事件驱动 | 通过钩子实现扩展性 |
| 协议抽象 | IDE 无关的桥接设计 |

---

## 给构建者的关键启示

1. **模块化工具设计** - 每个工具都是自包含的，具有清晰的接口
2. **权限优先架构** - 安全考虑内置于核心
3. **上下文管理** - 对长运行会话至关重要
4. **扩展性** - 插件和技能系统允许定制
5. **IDE 集成** - 桥接模式支持丰富的集成
6. **多代理模式** - 子代理生成处理复杂任务

---

## 研究/所有权声明

- 本仓库是一个**教育和防御性安全研究档案**。
- 其存在是为了研究源码泄露、打包失败以及现代代理 CLI 系统的架构。
- 原始 Claude Code 源代码仍属于 **Anthropic** 所有。
- 本仓库**不隶属于、不由 Anthropic 认可或维护**。
- **不包含任何实际源代码** - 仅包含高层架构分析和教育性观察。

---

## 贡献指南

**我们欢迎各种形式的学习目的 Pull Request！**

### 如何贡献

每个模块在 `learn/` 下都有自己的子目录。我们鼓励向对应的子目录贡献内容：

| 模块 | 目录 | 贡献内容 |
|------|------|----------|
| 工具系统 | [tool-system/](./tool-system/) | 工具模式、特定工具分析 |
| 命令系统 | [command-system/](./command-system/) | 命令模式、参数解析 |
| 服务层 | [service-layer/](./service-layer/) | API 模式、集成分析 |
| 桥接系统 | [bridge-system/](./bridge-system/) | 协议分析、IDE 集成 |
| 权限系统 | [permission-system/](./permission-system/) | 安全模型、审批流程 |
| 上下文管理 | [context-management/](./context-management/) | 压缩算法、内存模式 |
| 多代理系统 | [multi-agent/](./multi-agent/) | 代理生成、通信模式 |
| 插件系统 | [plugin-system/](./plugin-system/) | 扩展模式、钩子系统 |
| 技能系统 | [skill-system/](./skill-system/) | 提示工程、工作流设计 |
| UI 架构 | [ui-architecture/](./ui-architecture/) | 组件模式、状态管理 |

### 欢迎的贡献类型

- **详细分析** 特定组件或模式
- **图表和可视化** 架构图示
- **设计模式解释** 配合示例
- **与其他框架对比**（LangChain、AutoGPT 等）
- **教育教程** 基于架构
- **文档翻译**
- **代码流程图**（不含实际代码）


**如何提交扩展分析：**

1. 在相应模块的 `analysis/` 目录中创建分析文件
   - 命名格式：`PR{编号}_{简短标题}.md`
   - 示例：`PR42_permission-flow.md`
2. 更新该模块 `analysis/README.md` 中的索引表格
3. 可选：更新主 [analysis/README.md](./analysis/README.md) 以获得跨模块可见性

**模块分析目录快速链接：**

| 模块 | 分析目录 |
|------|----------|
| 工具系统 | [analysis/](./tool-system/analysis/) |
| 命令系统 | [analysis/](./command-system/analysis/) |
| 服务层 | [analysis/](./service-layer/analysis/) |
| 桥接系统 | [analysis/](./bridge-system/analysis/) |
| 权限系统 | [analysis/](./permission-system/analysis/) |
| 上下文管理 | [analysis/](./context-management/analysis/) |
| 多代理系统 | [analysis/](./multi-agent/analysis/) |
| 插件系统 | [analysis/](./plugin-system/analysis/) |
| 技能系统 | [analysis/](./skill-system/analysis/) |
| UI 架构 | [analysis/](./ui-architecture/analysis/) |

### 贡献准则

请确保所有贡献：

1. **不包含实际源代码** - 仅限高层描述
2. **关注教育价值** - 帮助他人学习
3. **尊重知识产权** - 这是分析，不是复制
4. **链接相关模块** - 展示组件间的联系

### 开始贡献

1. 选择你感兴趣的模块目录
2. 阅读该目录中现有的 README.md
3. 找出空白或需要更深入分析的领域
4. 提交包含教育性贡献的 PR

---

## 许可证

本分析仅用于教育目的。原始 Claude Code 软件为 Anthropic 所有。
