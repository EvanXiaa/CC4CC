# Claude Code Context Management 深度分析

本文档基于对 Claude Code 源码的逆向分析，详细解读其上下文管理机制。

---

## 目录

1. [Agent Context 组装过程](#1-agent-context-组装过程)
2. [Context Compact 压缩机制](#2-context-compact-压缩机制)
3. [CONTEXT_COLLAPSE 模式推断](#3-context_collapse-模式推断)
4. [关键问题与解答](#4-关键问题与解答)

---

## 1. Agent Context 组装过程

### 1.1 整体架构

Claude Code 的上下文组装分为两个主要部分：

```
┌─────────────────────────────────────────────────────────────┐
│                      API Request                             │
├─────────────────────────────────────────────────────────────┤
│  system: [System Prompt]                                     │
│          ├─ 基础系统提示（静态）                               │
│          ├─ git 状态（memoized，只计算一次）                   │
│          └─ 动态注入（cache breaker，ant-only）               │
│                                                              │
│  messages: [User Context]                                    │
│            ├─ CLAUDE.md 内容                                  │
│            ├─ 当前日期                                        │
│            └─ 其他配置文件                                     │
│                                                              │
│           [Conversation History]                             │
│            ├─ 原始对话消息                                    │
│            ├─ Tool results                                   │
│            ├─ Attachments (Memory, Skills, etc.)             │
│            └─ Compact summaries / Collapsed spans            │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 System Prompt 组装

**核心代码位置：** `query.ts:449-451`, `context.ts`

```typescript
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

**组件分解：**

| 组件 | 来源 | 特性 |
|------|------|------|
| `systemPrompt` | 入参 | 静态基础提示 |
| `git 状态` | `context.ts:36-111` | **memoized**，整个会话只计算一次 |
| `系统注入` | `context.ts:22-34` | 运行时动态插入 |
| `Cache Breaker` | `context.ts:130-147` | ant-only 调试注入 |

**Git 状态内容：**
- 当前分支名称
- 主分支检测
- `git status` 输出（截断到 2000 chars）
- 最近 commits 历史

### 1.3 User Context 注入

**关键区分：** User Context **不进入 system prompt**，而是作为 messages 数组的首条消息注入。

```typescript
// query.ts:660
prependUserContext()  // 注入 CLAUDE.md + 日期
```

**这样做的原因：**
- System prompt 有缓存优化，不适合频繁变化
- User context 包含项目特定信息，可能随项目变化
- 作为 user message 更符合 API 语义

### 1.4 Messages 数组流水线

每轮 queryLoop 的 messages 处理流水线：

```
原始 messages
      │
      ▼
┌─────────────────────────────────────────┐
│ getMessagesAfterCompactBoundary()       │  截取 compact 边界之后的消息
└────────────────────┬────────────────────┘
                     ▼
┌─────────────────────────────────────────┐
│ applyToolResultBudget()                 │  每条消息的 tool result 大小限制
└────────────────────┬────────────────────┘
                     ▼
┌─────────────────────────────────────────┐
│ applySnipCompact()                      │  历史裁剪，释放 token
└────────────────────┬────────────────────┘
                     ▼
┌─────────────────────────────────────────┐
│ applyMicrocompact()                     │  缓存优化 + 消息归并
└────────────────────┬────────────────────┘
                     ▼
┌─────────────────────────────────────────┐
│ contextCollapse.applyCollapsesIfNeeded()│  细粒度上下文折叠 (feature-gated)
└────────────────────┬────────────────────┘
                     ▼
┌─────────────────────────────────────────┐
│ autoCompactIfNeeded()                   │  全量 LLM 摘要压缩
└────────────────────┬────────────────────┘
                     ▼
              messagesForQuery
                     │
                     ▼
               发送给 API
```

### 1.5 Memory 注入机制

**异步预取模式：**

```typescript
// query.ts:301-304 - 循环开始时启动
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(messages, toolUseContext)

// query.ts:1599-1614 - Tool 执行后消费
const memoryAttachments = filterDuplicateMemoryAttachments(
  await pendingMemoryPrefetch.promise,
  toolUseContext.readFileState,  // 去重
)
```

**去重机制：**
- `readFileState` 记录模型已访问的文件路径
- Memory 注入时过滤已存在的路径
- 注入后立即标记，防止重复注入

---

## 2. Context Compact 压缩机制

### 2.1 三层压缩体系

Claude Code 采用三层渐进式压缩策略：

```
┌─────────────────────────────────────────────────────────────┐
│ Level 1: Snip Compact                                       │
│   - 无 LLM 调用                                              │
│   - 物理删除中间旧消息                                        │
│   - 最轻量，速度最快                                          │
├─────────────────────────────────────────────────────────────┤
│ Level 2: Micro Compact                                      │
│   - Time-based: 清空旧 tool result 内容                      │
│   - Cached MC: 服务端 cache_edit，本地消息不变                │
│   - 不调用主模型                                              │
├─────────────────────────────────────────────────────────────┤
│ Level 3: Auto Compact                                       │
│   - Session Memory Compact: 用后台 SM 文件作摘要              │
│   - LLM Compact: 独立 forked agent 生成全量摘要               │
│   - 最重，消耗 API tokens                                    │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Level 1: Snip Compact

**触发时机：** 每轮循环的消息预处理 pipeline 中最先执行

**实现：** 按 `groupMessagesByApiRound()` 分组，从头丢弃最老的组

**关键变量：** `snipTokensFreed` 记录释放的 token 数，用于后续阈值计算的修正

### 2.3 Level 2: Micro Compact

#### 路径 A: Time-Based Microcompact

**触发条件：**
```
当前时间 - 最后一条 assistant 消息时间戳 > gapThresholdMinutes
```

**原理：** 服务端 cache 已冷，直接清空旧 tool result 内容

**操作：**
- 保留最近 `keepRecent` 条
- 其余 content → `"[Old tool result content cleared]"`
- 直接 mutate message content

#### 路径 B: Cached Microcompact (核心路径)

**原理：** 服务端 cache 是热的，通过 `cache_edits` 告诉服务端删除 tool_result

**关键设计：**
- **本地 messages 数组不变**
- 构建 `cache_edits` 指令块，在 API 调用时携带
- 服务端返回 `cache_deleted_input_tokens` 确认节省量

**可压缩的工具类型（COMPACTABLE_TOOLS）：**

```typescript
const COMPACTABLE_TOOLS = new Set<string>([
  'Read',        // FileReadTool
  'Bash',        // ShellTool
  'PowerShell',  // PowerShellTool
  'Grep',        // GrepTool
  'Glob',        // GlobTool
  'WebSearch',   // WebSearchTool
  'WebFetch',    // WebFetchTool
  'Edit',        // FileEditTool
  'Write',       // FileWriteTool
])
```

**选择标准：** 输出结果大、输入参数小、结果消费后历史价值递减。AgentTool、SkillTool 等决策链重要的工具不在此列。

**触发条件（计数阈值触发）：**

```
触发 = 累计注册的 compactable tool results >= triggerThreshold

配置参数（来自 GrowthBook）:
- triggerThreshold: 触发阈值（如 10）
- keepRecent: 保留最近 N 个（如 5）

示例:
  Turn 1-9:  注册 9 个 → 9 < 10，不触发
  Turn 10:   注册 10 个 → 10 >= 10，删除前 5 个，保留最近 5 个
  Turn 11:   注册 11 个 → 删除前 6 个，保留最近 5 个
```

**前置条件：**

```typescript
// 必须同时满足
feature('CACHED_MICROCOMPACT')           // Feature flag 开启
isCachedMicrocompactEnabled()            // GrowthBook 开关
isModelSupportedForCacheEditing(model)   // 模型支持 cache editing
isMainThreadSource(querySource)          // 是主线程（非 forked agent）
```

**状态追踪：**
```typescript
cachedMCState = {
  registeredTools: Set<string>,    // 已注册的工具 ID
  toolOrder: string[],             // 工具历史顺序
  deletedRefs: Set<string>,        // 已删除的工具
  pinnedEdits: PinnedCacheEdits[], // 已发送的 cache_edits
}
```

**cache_edits 数据结构：**

```typescript
type CachedMCEditsBlock = {
  type: 'cache_edits'
  edits: { type: 'delete'; cache_reference: string }[]
}

// 示例
{
  type: 'cache_edits',
  edits: [
    { type: 'delete', cache_reference: 'toolu_01ABC123' },
    { type: 'delete', cache_reference: 'toolu_01DEF456' }
  ]
}
```

**cache_reference 机制：**

```typescript
// claude.ts:3201 - 使用 tool_use_id 作为 cache_reference
msg.content[j] = Object.assign({}, block, {
  cache_reference: block.tool_use_id
})
```

`cache_reference` 的作用是让服务端**直接定位** cached KV entry，而不依赖位置匹配。

**Pinned Edits 机制（必须重新发送）：**

每次请求都必须携带所有历史的 `pinnedEdits`，因为：
- 消息内容仍在 messages 数组中
- 如果不带 `cache_edits`，服务端会重新看到被删除的内容
- 会导致 cache miss → 重新计算 → 重新计费

```typescript
// 每次请求前重新插入
for (const pinned of pinnedEdits ?? []) {
  insertBlockAfterToolResults(msg.content, pinned.block)
}
```

### 2.4 Level 3: Auto Compact

#### 触发决策树

```typescript
autoCompactIfNeeded()
  │
  ├─ [熔断器] consecutiveFailures >= 3 → return false
  │
  ├─ shouldAutoCompact()
  │   ├─ querySource == 'session_memory'/'compact'/'marble_origami' → false
  │   ├─ CONTEXT_COLLAPSE 启用 → false
  │   └─ tokenCount >= autoCompactThreshold → true
  │
  ├─ [优先] trySessionMemoryCompaction()
  │
  └─ compactConversation()  // 传统 LLM 摘要
```

#### 阈值计算

```typescript
effectiveWindow = contextWindow(model) - 20_000  // 预留摘要输出空间
autoCompactThreshold = effectiveWindow - 13_000  // 触发阈值
blockingLimit = effectiveWindow - 3_000          // 硬保护阈值
```

#### Session Memory Compact

**条件：** `tengu_session_memory` 且 `tengu_sm_compact` feature flags 开启

**核心思路：** 不发起额外 LLM 调用，使用后台维护的 SessionMemory 文件作为摘要

**执行流程：**
1. 等待后台 SM 提取完成
2. 找到上次摘要边界 `lastSummarizedMessageId`
3. 计算保留哪些原始消息（满足 minTokens, minTextBlockMessages）
4. 用 SM 内容构建 CompactionResult

#### 传统 LLM Compact

**完整执行步骤：**

```
① PreCompact Hooks → 可注入自定义压缩指令
② stripImagesFromMessages() → 图片替换为占位符
③ runForkedAgent(summaryRequest) → 独立 agent 生成摘要
   - 若 PTL: truncateHeadForPTLRetry() 从头丢弃最老组
④ readFileState.clear() → 清空文件读取缓存
⑤ createPostCompactFileAttachments() → 恢复最多 5 个文件
⑥ 重建工具/Agent/MCP delta attachments
⑦ processSessionStartHooks('compact') → 恢复 CLAUDE.md 等
⑧ executePostCompactHooks()
⑨ 构建 CompactionResult
```

**压缩后消息结构：**
```
[boundaryMarker]         ← SystemCompactBoundaryMessage
[summaryMessage]         ← LLM 生成的摘要
[...messagesToKeep]      ← SM-compact 专用：保留的原始消息
[...attachments]         ← 恢复的文件 + delta
[...hookResults]         ← session-start hook 注入
```

### 2.5 熔断器机制

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// 连续失败 3 次后停止尝试
if (tracking.consecutiveFailures >= 3) {
  return { wasCompacted: false }
}
```

**目的：** 防止在上下文无法恢复时反复消耗 API tokens

---

## 3. CONTEXT_COLLAPSE 模式推断

> **注意：** `src/services/contextCollapse/` 目录在源码中不存在（被 feature gate 完全移除），以下分析基于类型定义和代码注释推断。

### 3.1 核心概念

CONTEXT_COLLAPSE（代号 `marble_origami`）是一种**细粒度的渐进式上下文折叠策略**，取代传统的 autocompact 全量压缩。

**核心差异：**

| 维度 | Autocompact | CONTEXT_COLLAPSE |
|------|-------------|------------------|
| 压缩粒度 | 全量摘要 | 分段折叠 |
| 信息保留 | 低（单条摘要） | 高（多条细粒度摘要） |
| 触发频率 | 一次（大爆炸） | 多次（渐进式） |
| 恢复能力 | 无 | drain 暂存队列 |
| 状态管理 | 无状态 | commit log + snapshot |

### 3.2 数据结构

```typescript
// 已提交的折叠记录（append-only）
type ContextCollapseCommitEntry = {
  type: 'marble-origami-commit'
  collapseId: string              // 16位折叠ID
  summaryUuid: string             // 摘要占位符 UUID
  summaryContent: string          // <collapsed id="...">text</collapsed>
  summary: string                 // 纯文本摘要
  firstArchivedUuid: string       // 被折叠范围的起始 UUID
  lastArchivedUuid: string        // 被折叠范围的结束 UUID
}

// 暂存队列快照（last-wins）
type ContextCollapseSnapshotEntry = {
  type: 'marble-origami-snapshot'
  staged: Array<{
    startUuid: string
    endUuid: string
    summary: string
    risk: number                  // 风险评分
    stagedAt: number
  }>
  armed: boolean                  // spawn 触发器是否就绪
  lastSpawnTokens: number         // 上次 spawn 时的 token 数
}
```

### 3.3 阈值阶梯

```
┌────────────────────────────────────────────────────────────┐
│                    Token 使用量                             │
│                                                            │
│  0% ──────────────────────────────────────────────── 100%  │
│                                                            │
│                                    ┌── 90% commit-start    │
│                                    │   (开始暂存折叠)        │
│                                    │                        │
│                                    │   ┌── 93% autocompact  │
│                                    │   │   (被 collapse 抑制) │
│                                    │   │                    │
│                                    │   │   ┌── 95% blocking │
│                                    │   │   │   (阻止 spawn)  │
│                                    │   │   │                │
│                                    │   │   │   ┌── 97% PTL  │
│                                    │   │   │   │   (API 拒绝) │
└────────────────────────────────────┴───┴───┴───┴────────────┘
```

### 3.4 执行流程

```
每轮 queryLoop 开始
      │
      ▼
检查 token 使用量
      │
      ├─ < 90%  → 不触发
      │
      ├─ >= 90% 且 armed=true 且 token 有足够增量
      │     └─ spawn ctx-agent（后台 haiku）
      │     └─ 生成摘要，加入 staged 队列
      │
      └─ >= 95%
            └─ 阻止新 spawn
            └─ 已有的 staged 可以 commit
```

### 3.5 消息投影机制

**关键设计：** collapse **不修改 REPL 的消息数组**，而是通过 `projectView()` 进行读取时投影。

```
原始 REPL 消息数组（完整历史）
         │
         ▼ projectView() 重放 commit log
         │
Collapsed View（折叠后的视图）
         │
         ▼ 作为 messagesForQuery 发送给 API
```

**投影后消息结构：**

```
[s1] user: {
       type: 'user',
       isMeta: true,  // 不显示在对话视图中
       content: "<collapsed id=\"0000000000000001\">摘要内容...</collapsed>"
     }
[m16] user: "帮我实现注册功能"
[m17] assistant: "我来..."
...
```

### 3.6 413 恢复路径

```
API 返回 413 prompt-too-long
      │
      ▼
[1] CONTEXT_COLLAPSE: recoverFromOverflow()
      │
      ├─ drained.committed > 0 ?
      │     └─ transition = 'collapse_drain_retry'
      │     └─ continue（重试本轮）
      │
      ▼
[2] REACTIVE_COMPACT: tryReactiveCompact()
      │
      ├─ 成功 ?
      │     └─ transition = 'reactive_compact_retry'
      │     └─ continue
      │
      ▼
[3] 无法恢复 → 返回 prompt_too_long 错误
```

### 3.7 与 Autocompact 的互斥

```typescript
// autoCompact.ts:215-223
if (feature('CONTEXT_COLLAPSE')) {
  if (isContextCollapseEnabled()) {
    return false  // autocompact 完全退出
  }
}
```

---

## 4. 关键问题与解答

### Q1: Micro Compact 的 Cached MC 路径为什么不修改本地消息？

**A:** 核心原因是**保护 prompt cache**。

- 服务端的 prompt cache 是热的，旧 tool result 仍在 cache 中
- 如果修改本地消息内容，会导致整个 cached prefix 失效
- 通过 `cache_edits` 告诉服务端删除特定 tool_result，**在服务端原地节省 tokens**
- 本地消息数组不变，下次请求仍能命中其他 cached tokens

```
传统方式（修改本地消息）：
  messages 变化 → 整个 cache 失效 → 全部 cache_creation

Cached MC 方式：
  messages 不变 + cache_edits → 仅删除指定 tool_result → 其他 cache 仍命中
```

### Q2: Memory 注入如何去重？

**A:** 通过 `readFileState` 双重追踪。

```typescript
// attachments.ts:2520-2535
export function filterDuplicateMemoryAttachments(
  attachments: Attachment[],
  readFileState: FileStateCache,
): Attachment[] {
  return attachments.map(attachment => {
    if (attachment.type !== 'relevant_memories') return attachment
    // 过滤已存在的路径
    const filtered = attachment.memories.filter(m => !readFileState.has(m.path))
    // 注入后立即标记
    for (const m of filtered) {
      readFileState.set(m.path, { content: m.content, ... })
    }
    return { ...attachment, memories: filtered }
  })
}
```

**关键顺序：先过滤，再标记。** 如果在 prefetch 期间就写入 `readFileState`，过滤时所有路径都会显示"已在 context 中"，导致每次 memory 都被丢弃。

### Q3: ToolUseSummary 是什么？与压缩有关吗？

**A:** **与压缩无关**，是一个独立的展示层功能。

**用途：** 为 SDK/移动端客户端提供工具执行进度的轻量摘要标签。

**工作方式：**
- 每轮工具批次执行完后，**异步调用 Haiku** 生成摘要
- 摘要格式类似 git commit subject（~30 字符）
- 例如：`"Fixed NPE in UserService"`、`"Searched in auth/"`

**时机优化：**
```
工具批次执行完毕
  → 异步启动 haiku 调用（不等待）
  → 进入下一轮主模型流式响应
  → 主模型响应期间，haiku 并行完成
  → 下一轮循环开始时 yield 这个 summary
```

Haiku 的延迟完全被主模型响应时间掩盖，对总延迟贡献为零。

### Q4: CONTEXT_COLLAPSE 模式下 query messages 是什么样的？

**A:** 采用**读取时投影**，原始 REPL 数组不变。

**投影机制：**
```
原始 REPL 消息数组（完整历史，不变）
         │
         ▼ projectView() 重放 commit log
         │
Collapsed View（折叠后的视图）
         │
         ▼ 作为 messagesForQuery 发送给 API
```

**投影后结构示例：**

假设原始有 20 条消息，经过 2 次折叠：

```
原始: [m0] ... [m5] ... [m10] ... [m15] ... [m19]

Commit 1: 折叠 m0-m5
Commit 2: 折叠 m6-m10

投影后:
[s1] user: { isMeta: true, content: "<collapsed id='...'>摘要1</collapsed>" }
[s2] user: { isMeta: true, content: "<collapsed id='...'>摘要2</collapsed>" }
[m11] user: "继续..."
[m12] assistant: ...
...
[m19] assistant: ...
```

**关键特性：**
- `<collapsed>` 占位符是 `isMeta: true`，对用户不可见但对 API 可见
- 每段摘要独立，保留更多结构信息
- 恢复时重放 commit log 重建视图

### Q5: 分段摘要如何判断边界？多久摘要一次？

**A:** 边界判断和触发频率基于以下机制。

**边界判断（推断）：**

1. **API Round 边界**：以 API 响应为分段单位，保持 tool_use/tool_result 配对完整
2. **Risk 评分**：每个 staged span 有 `risk` 字段，低风险段优先折叠
3. **Token 密度**：优先折叠 token 密度高但信息密度低的段（如大型 tool_result）

**触发频率：阈值阶梯机制**

```
Token 使用量
  0% ──────────────────────────────────────────── 100%
                              │
                              ├── 90% commit-start
                              │    (开始 spawn ctx-agent)
                              │
                              ├── 95% blocking
                              │    (阻止新 spawn)
                              │
                              └── 97% PTL
                                   (API 拒绝)
```

**Spawn 控制：**
- `lastSpawnTokens` 记录上次 spawn 时的 token 数
- 只有当 token 增量超过阈值时才再次 spawn
- 避免在 90%-95% 区间频繁 spawn ctx-agent

**ctx-agent：** 后台运行的 Haiku agent（querySource = `marble_origami`），负责生成分段摘要。

---

## 附录：关键文件索引

| 文件 | 行数 | 主要职责 |
|------|------|----------|
| `query.ts` | ~1,729 | 消息流水线、循环控制、压缩触发 |
| `QueryEngine.ts` | ~46K | LLM API 调用引擎 |
| `Tool.ts` | ~29K | 工具类型定义、权限模型 |
| `context.ts` | ~200 | System/User context 组装 |
| `services/compact/autoCompact.ts` | ~350 | Autocompact 触发逻辑 |
| `services/compact/compact.ts` | ~750 | LLM 压缩实现 |
| `services/compact/microCompact.ts` | ~530 | Micro compact 实现 |
| `services/compact/sessionMemoryCompact.ts` | ~630 | SM-based compact |
| `types/logs.ts` | ~330 | 数据结构定义（含 collapse entries） |

---

*文档生成时间: 2026-04-01*
*基于 Claude Code 源码逆向分析*