# 12 层渐进式包装架构

> **定位**: 这不是某个子系统的深入分析，而是一个**全局视角**——将 Claude Code 理解为从最简 Agent Loop 逐层叠加到工业级产品的 12 层包装（Progressive Harness）。每一层都建立在前一层之上，层层递进。

---

## 目录

1. [为什么需要这个视角](#1-为什么需要这个视角)
2. [全景图](#2-全景图)
3. [Layer 1: Agent Loop — 最小循环](#3-layer-1-agent-loop--最小循环)
4. [Layer 2: Tool Dispatch — 工具分发](#4-layer-2-tool-dispatch--工具分发)
5. [Layer 3: Planning — 先想后做](#5-layer-3-planning--先想后做)
6. [Layer 4: Sub-Agents — 子代理隔离](#6-layer-4-sub-agents--子代理隔离)
7. [Layer 5: Knowledge On Demand — 按需知识](#7-layer-5-knowledge-on-demand--按需知识)
8. [Layer 6: Context Compression — 上下文压缩](#8-layer-6-context-compression--上下文压缩)
9. [Layer 7: Persistent Tasks — 持久化任务](#9-layer-7-persistent-tasks--持久化任务)
10. [Layer 8: Background Execution — 后台执行](#10-layer-8-background-execution--后台执行)
11. [Layer 9: Agent Teams — 团队组建](#11-layer-9-agent-teams--团队组建)
12. [Layer 10: Team Protocols — 团队协议](#12-layer-10-team-protocols--团队协议)
13. [Layer 11: Autonomous Agents — 自治代理](#13-layer-11-autonomous-agents--自治代理)
14. [Layer 12: Worktree Isolation — 工作树隔离](#14-layer-12-worktree-isolation--工作树隔离)
15. [层间关系](#15-层间关系)
16. [从零构建的思维实验](#16-从零构建的思维实验)
17. [对照索引](#17-对照索引)

---

## 1. 为什么需要这个视角

我们已有的文档按**模块**组织（QueryEngine、Tool System、Permission System...），这很适合"我想了解某个子系统"的需求。但它缺少一种视角：

```
Q: 如果我要从零构建一个类似 Claude Code 的产品，应该先做什么、后做什么？
Q: 每一层机制解决了什么问题？没有它会怎样？
Q: Claude Code 的复杂度是怎么一步步长出来的？
```

**12 层渐进式包装**回答这些问题。它不是代码的物理结构，而是**功能的逻辑层次**——每一层都让系统从"能跑"进化到"好用"再到"强大"。

---

## 2. 全景图

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 12: Worktree Isolation    Git 工作树隔离，并行修改不同分支  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 11: Autonomous Agents     自驱动的空闲→领取→执行循环       │
├─────────────────────────────────────────────────────────────────┤
│ Layer 10: Team Protocols        统一的消息协议驱动所有协商        │
├─────────────────────────────────────────────────────────────────┤
│ Layer 9: Agent Teams            持久化队友 + 邮箱 + 权限同步     │
├─────────────────────────────────────────────────────────────────┤
│ Layer 8: Background Execution   Ctrl+B 后台化 + 完成通知         │
├─────────────────────────────────────────────────────────────────┤
│ Layer 7: Persistent Tasks       持久化任务图，带状态和依赖        │
├─────────────────────────────────────────────────────────────────┤
│ Layer 6: Context Compression    三层压缩 + FRC，透明无限对话      │
├─────────────────────────────────────────────────────────────────┤
│ Layer 5: Knowledge On Demand    Skills + CLAUDE.md 按需注入      │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Sub-Agents             Fork 隔离，主对话保持干净         │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Planning               先列步骤再执行                   │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Tool Dispatch          buildTool() 工厂 + 并行执行      │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Agent Loop             while(true) + tool_use 检查      │
└─────────────────────────────────────────────────────────────────┘
```

**规律**: 层 1-6 是单个 Agent 的能力增强；层 7-12 是多 Agent 协作的基础设施。

---

## 3. Layer 1: Agent Loop — 最小循环

### 解决的问题
> 如何让 LLM 不只是回答问题，而是能**持续行动**？

### 核心机制

```typescript
while (true) {
  response = await claude.sendMessage(messages)
  messages.push(response)

  if (!response.hasToolUse()) break  // ← 没有工具调用就停

  results = await executeTool(response.toolUse)
  messages.push(results)
  // 继续循环，让模型看到工具结果
}
```

**这就是全部。** 整个 Claude Code 的核心是这个极简循环。

### 为什么简单就够了

复杂性被推到了两个地方：
- **向内** → 工具层自己处理验证、权限、并发
- **向外** → 12 层包装在循环外叠加 production 特性

核心循环本身**从不改变**。添加新功能 = 添加新工具或新包装层。这是 **Open-Closed Principle** 在 Agent 系统中的实践。

### 关键源码
- `src/query.ts` — query() AsyncGenerator，Agent Loop 的实际实现
- `src/QueryEngine.ts` — 管理 messages[]、AbortController、会话状态

### 深入阅读
→ [modules-part1.md 模块 2-3](./modules-part1.md)

---

## 4. Layer 2: Tool Dispatch — 工具分发

### 解决的问题
> Agent Loop 只知道"有工具调用"和"执行工具"。如何让 45+ 工具统一管理、安全执行？

### 核心机制

`buildTool()` 工厂函数为每个工具创建统一的生命周期接口：

```
每个工具必须实现:
  name            → 工具名称
  description     → 工具描述（给模型看）
  inputSchema     → Zod 验证的输入 schema
  isEnabled()     → 当前环境是否可用
  canUseTool()    → 权限检查
  execute()       → 实际执行逻辑
  isConcurrencySafe() → 是否可并行
```

**Fail-Closed 默认值**——工具安全设计的典范：

```
新工具忘记声明并发安全性? → 默认串行执行（安全）
新工具忘记声明只读?       → 默认触发权限检查（安全）
安全相关工具?            → 必须显式覆盖分类器
```

### 并行执行

StreamingToolExecutor 将 API 返回的多个 tool_use blocks 分区处理：

```
API 返回 [Glob("*.ts"), Grep("TODO"), Read("main.ts")]
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
 Glob (并发安全)  Grep (并发安全)  Read (并发安全)
    │               │               │
    └───── Promise.all() ──────────┘
                    │
              结果追加到 messages[]
```

这就是为什么 Claude Code 搜索文件时感觉很快——多个搜索**真的在并行**。

### 关键源码
- `src/Tool.ts` — 工具接口定义和 buildTool() 工厂
- `src/tools/` — 45+ 工具实现

### 深入阅读
→ [modules-part1.md 模块 4](./modules-part1.md), [architecture.md 工具系统](./architecture.md)

---

## 5. Layer 3: Planning — 先想后做

### 解决的问题
> Agent 直接开干容易跑偏，尤其是多步骤任务。如何提高完成率？

### 核心机制

```
没有 Planning:
  用户: "重构认证系统"
  Agent: 直接开始改代码 → 改了一半发现方向错了 → 回退 → 再试 → ...

有 Planning:
  用户: "重构认证系统"
  Agent:
    Step 1: 分析现有认证流程      ✅
    Step 2: 列出需要修改的文件     ✅
    Step 3: 设计新的认证架构       ✅
    Step 4: 用户确认方案           ← 检查点
    Step 5: 按计划逐步实现         ✅
    Step 6: 运行测试验证           ✅
```

### 为什么这一层 ROI 惊人

仅仅添加"先列步骤再执行"的机制就使任务完成率大幅提升。这与学术界 Plan-and-Execute Agent 的研究结论一致——**Planning 是最便宜的"免费午餐"之一**。

### 实现方式

| 工具 | 作用 |
|------|------|
| `EnterPlanModeTool` | 进入计划模式，只读不写 |
| `ExitPlanModeTool` | 退出计划模式，开始执行 |
| `TodoWriteTool` / `TaskCreateTool` | 创建结构化任务清单 |

### 关键源码
- `src/tools/EnterPlanModeTool/`
- `src/tools/TaskCreateTool/`

### 深入阅读
→ [modules-part1.md 模块 4 工具分类](./modules-part1.md)

---

## 6. Layer 4: Sub-Agents — 子代理隔离

### 解决的问题
> 复杂任务需要大量工具调用，它们的输出会淹没主对话的上下文窗口。

### 核心机制

```
主代理                          子代理 (Fork)
┌──────────┐                   ┌──────────┐
│ 用户对话  │  spawn("研究X")   │ 全新的    │
│          │ ─────────────────▶│ messages[]│
│ 保持干净  │                   │ 独立执行  │
│          │                   │ 工具输出  │
│          │  ◄─ 只返回摘要 ──  │ 在这里堆积│
│          │                   │ 不影响主  │
└──────────┘                   └──────────┘
```

**关键价值**: 子代理的工具输出**不会回到主代理的上下文**。主代理只收到一个结构化的结果摘要。这解决了 Agent 系统中最常见的问题——**上下文污染**。

### 四种 Spawn 模式

| 模式 | 隔离级别 | 场景 |
|------|---------|------|
| default | messages[] 共享 | 简单委派 |
| fork | 全新 messages[] | 研究任务、多步实现 |
| worktree | 独立 Git 目录 + 全新消息 | 并行修改不同功能 |
| remote | 完全隔离（容器） | 远程执行 |

### 关键源码
- `src/tools/AgentTool/` — Agent 启动和 fork 逻辑
- `src/tools/AgentTool/forkSubagent.ts` — Fork 隔离实现

### 深入阅读
→ [modules-part2.md 模块 15](./modules-part2.md)

---

## 7. Layer 5: Knowledge On Demand — 按需知识

### 解决的问题
> 模型不可能预先知道所有项目的规范、框架用法、团队约定。如何在需要时注入知识？

### 核心机制

```
传统方式 (预加载):
  System Prompt = 基础 Prompt + 所有知识
  → 每次 API 调用都带全量知识 → 浪费 token

Claude Code (按需注入):
  System Prompt = 基础 Prompt + 最小上下文
  需要时: SkillTool.call("commit") → 知识通过 tool_result 注入
  项目级: CLAUDE.md → 按目录层级惰性加载
  MCP: 只加载已连接服务器的说明
```

### 三个知识来源

| 来源 | 加载时机 | 注入方式 |
|------|---------|---------|
| **Skills** | 用户触发或自动发现 | tool_result 注入到对话 |
| **CLAUDE.md** | 会话开始时按目录加载 | system prompt 动态区 |
| **MCP Instructions** | 服务器连接时 | system prompt 动态区 |

### 关键源码
- `src/skills/` — Skill 系统
- `src/memdir/` — 记忆系统和 CLAUDE.md 加载
- `src/constants/prompts.ts` — 按需 Section 组装

### 深入阅读
→ [modules-part2.md 模块 11, 14](./modules-part2.md), [system-prompt-design.md](./system-prompt-design.md)

---

## 8. Layer 6: Context Compression — 上下文压缩

### 解决的问题
> 对话越长，上下文窗口越满。如何让用户感觉"对话无限长"？

### 核心机制：三层递进压缩

```
对话进行中...
     │
     ▼ Token 计数接近阈值
┌──────────────────────────────────────┐
│ Layer 1: autoCompact（自动压缩）      │
│ 调用 Claude API 对旧消息生成摘要      │
│ 保留最近消息的完整保真度              │
│ 插入 compact_boundary 标记           │
└──────────────┬───────────────────────┘
               │ 仍然不够?
               ▼
┌──────────────────────────────────────┐
│ Layer 2: snipCompact（裁剪压缩）      │
│ 移除僵尸消息和过期标记                │
│ 清理无用的工具结果                    │
└──────────────┬───────────────────────┘
               │ 还不够?
               ▼
┌──────────────────────────────────────┐
│ Layer 3: contextCollapse（结构重组）   │
│ 重构上下文的组织方式                  │
│ 合并可合并的信息块                    │
└──────────────────────────────────────┘
```

**加上 FRC (Function Result Clearing)**：
- 只保留最近 N 个工具结果
- 更早的被自动清除
- Prompt 指导模型提前"记笔记"防止信息丢失

**用户感知**: System prompt 告诉模型 "The conversation has unlimited context through automatic summarization"——用户和模型都认为对话没有长度限制。

### 关键源码
- `src/services/compact/` — 压缩服务

### 深入阅读
→ [advanced-topics.md Topic 2](./advanced-topics.md)

---

## 9. Layer 7: Persistent Tasks — 持久化任务

### 解决的问题
> 层 1-6 解决了单个 Agent 的能力问题。但复杂项目需要**结构化的任务管理**——哪些做了、哪些没做、谁在做什么。

### 核心机制

```
基于文件的任务图:
~/.claude/tasks/{teamOrSessionId}/

Task #1: "实现搜索功能"
  ├── status: completed ✅
  ├── owner: researcher
  └── blocks: [#2, #3]

Task #2: "添加搜索 UI"
  ├── status: in_progress 🔄
  ├── owner: coder
  └── blockedBy: [#1]      ← 等 #1 完成

Task #3: "编写搜索测试"
  ├── status: pending ⏳
  ├── owner: (无)
  └── blockedBy: [#1]
```

### 从这一层开始，进入多 Agent 领域

任务本身不需要多 Agent——单个 Agent 也能用任务列表跟踪进度。但任务系统是多 Agent 协作的**基础设施**：Layer 9-11 的团队协作都建立在这个任务图之上。

### 关键源码
- `src/tasks/` — 任务类型定义
- `src/tools/TaskCreateTool/`, `TaskUpdateTool/`, `TaskListTool/`

---

## 10. Layer 8: Background Execution — 后台执行

### 解决的问题
> 长时间运行的任务（编译、测试、大规模重构）会阻塞用户交互。

### 核心机制

```
用户正在等待一个耗时操作...
     │
     │ 按 Ctrl+B
     ▼
┌──────────────────────────────────────┐
│ 当前任务转入后台                      │
│ • isBackgrounded = true              │
│ • 输出溢出到磁盘文件                  │
│ • UI 返回到全新提示符                 │
│                                      │
│ 用户可以继续其他工作                  │
│                                      │
│ 后台任务完成时:                       │
│ • 推送通知到 UI                      │
│ • 结果保留供查看                      │
└──────────────────────────────────────┘
```

### 任务类型

| 类型 | 用途 |
|------|------|
| `LocalShellTask` | Bash 命令后台执行 |
| `LocalAgentTask` | Agent 查询后台化 |
| `DreamTask` | 记忆整固（AI 空闲时自动整理记忆） |

### 关键源码
- `src/tasks/LocalShellTask/`
- `src/tasks/LocalAgentTask/`
- `src/tasks/DreamTask/`

---

## 11. Layer 9: Agent Teams — 团队组建

### 解决的问题
> 子代理（Layer 4）是一次性的——用完即销。复杂项目需要**持久化的队友**，它们有自己的身份、邮箱、和生命周期。

### 核心机制

```
Leader 调用 TeamCreateTool("my-team")
          │
          ▼
┌──────────────────────────────────────────┐
│ 创建持久化团队:                           │
│ ~/.claude/teams/my-team/                 │
│   ├── config.json    # 团队配置           │
│   ├── inboxes/       # 每人一个邮箱       │
│   │   ├── team-lead.json                 │
│   │   ├── researcher.json               │
│   │   └── coder.json                    │
│   └── permissions/   # 权限请求队列       │
│                                          │
│ 队友特征:                                 │
│ • 唯一 ID: "researcher@my-team"          │
│ • 独立 AbortController（Escape 不杀队友） │
│ • 独立权限模式                            │
│ • 持续运行直到被关闭                      │
└──────────────────────────────────────────┘
```

### 与 Layer 4 (Sub-Agents) 的区别

| 特性 | Sub-Agent (L4) | Team Member (L9) |
|------|----------------|-------------------|
| 生命期 | 单次任务 | 持续到团队解散 |
| 身份 | 匿名 | 有名有姓有颜色 |
| 通信 | 返回结果 | 双向邮箱 |
| 状态 | 运行→完成 | 运行→空闲→再运行→... |

### 关键源码
- `src/tools/TeamCreateTool/`, `TeamDeleteTool/`
- `src/utils/swarm/teamHelpers.ts`
- `src/utils/swarm/spawnInProcess.ts`

### 深入阅读
→ [swarm-multi-agent.md](./swarm-multi-agent.md)

---

## 12. Layer 10: Team Protocols — 团队协议

### 解决的问题
> 有了队友和邮箱，还需要**标准化的通信协议**。什么时候发消息、什么格式、如何处理权限请求和关闭请求？

### 核心机制：SendMessage 统一协议

```
SendMessageTool 是所有 Agent 间通信的唯一入口:

  ┌────────────────────────────────────┐
  │ 普通消息    → 文本或 XML 包装       │
  │ 权限请求    → permission_request   │
  │ 权限回复    → permission_response  │
  │ 关闭请求    → shutdown_request     │
  │ 关闭决策    → shutdown_approved/   │
  │              shutdown_rejected     │
  │ 空闲通知    → idle_notification    │
  │ 广播        → to: "*"             │
  └────────────────────────────────────┘
```

### 权限同步的双路径

```
Queue 路径（快速，同进程）:
  队友 ──→ Leader 的 React UI 队列 ──→ 用户审批 ──→ 回调返回

Mailbox 路径（兼容，跨进程）:
  队友 ──→ 邮箱文件 ──→ Leader 轮询 ──→ 审批 ──→ 写入响应文件 ──→ 队友轮询
```

### 优雅关闭协商

```
Leader: "任务完成了，可以关闭吗？"  → shutdown_request
Teammate 模型自己判断:
  "手头没活了"  → shutdown_approved → 安全退出
  "正在写文件"  → shutdown_rejected → 继续工作
```

### 关键源码
- `src/tools/SendMessageTool/`
- `src/utils/swarm/permissionSync.ts`
- `src/utils/teammateMailbox.ts`

### 深入阅读
→ [swarm-multi-agent.md §5-6](./swarm-multi-agent.md)

---

## 13. Layer 11: Autonomous Agents — 自治代理

### 解决的问题
> 有了团队和协议，Leader 成了瓶颈——每个任务都要 Leader 分配。能否让队友**自驱动**？

### 核心机制：空闲→领取循环

```
队友完成当前任务
     │
     ▼ 进入空闲状态
┌──────────────────────────────────────┐
│ waitForNextPromptOrShutdown()        │
│                                      │
│ 每 500ms 检查:                       │
│ ① 有来自 Leader 的消息? → 执行       │
│ ② 有关闭请求? → 协商                 │
│ ③ 有队友消息? → 响应                 │
│ ④ 任务列表有可领取的任务?             │
│    → tryClaimNextTask()              │
│    → 自动领取并开始执行!              │
└──────────────────────────────────────┘
```

**精妙之处**: Leader 只需创建任务列表，队友会**自动分配工作**。这将 Leader 从"分配每个任务"的瓶颈中解放出来：

```
传统方式:
  Leader: "A 做任务 1"  →  等 A 完成  →  "A 做任务 2"  →  ...（串行瓶颈）

自治方式:
  Leader: 创建任务 1, 2, 3, 4, 5
  Teammate A: 自动领取 #1 → 完成 → 自动领取 #4
  Teammate B: 自动领取 #2 → 完成 → 自动领取 #5
  Teammate C: 自动领取 #3 → 完成
  （Leader 该干嘛干嘛，不需要逐个分配）
```

### 关键源码
- `src/utils/swarm/inProcessRunner.ts` — 空闲循环与自动领取
- `src/coordinator/coordinatorMode.ts` — Coordinator 系统提示词

### 深入阅读
→ [swarm-multi-agent.md §4, §7](./swarm-multi-agent.md)

---

## 14. Layer 12: Worktree Isolation — 工作树隔离

### 解决的问题
> 多个 Agent 同时修改代码会产生冲突。如何让每个 Agent 有自己的"工作区"？

### 核心机制

```
Agent A: 修改 auth 模块            Agent B: 修改 search 模块
     │                                  │
     ▼                                  ▼
┌──────────────────┐            ┌──────────────────┐
│ Git Worktree A   │            │ Git Worktree B   │
│ ../project-auth/ │            │ ../project-search│
│                  │            │                  │
│ 独立的文件系统    │            │ 独立的文件系统    │
│ 独立的 Git 分支   │            │ 独立的 Git 分支   │
│ 不会互相干扰      │            │ 不会互相干扰      │
└────────┬─────────┘            └────────┬─────────┘
         │                               │
         └───── 各自 commit ─────────────┘
                     │
                     ▼
              合并回主分支
```

### 与前面层的配合

- Layer 4 (Sub-Agent) 可以在 worktree 中运行
- Layer 7 (Tasks) 通过任务 ID 绑定到特定 worktree
- Layer 9 (Teams) 的队友可以各自拥有 worktree

### 关键源码
- `src/tools/EnterWorktreeTool/`
- `src/tools/ExitWorktreeTool/`

### 深入阅读
→ [advanced-topics.md Topic 11](./advanced-topics.md)

---

## 15. 层间关系

### 15.1 依赖图

```
L12 Worktree ──────┐
L11 Autonomous ────┤
L10 Protocols ─────┤ 都依赖 L9 Teams
L9  Teams ─────────┤ 依赖 L7 Tasks + L4 Sub-Agents
L8  Background ────┤ 依赖 L7 Tasks
L7  Tasks ─────────┤ 依赖 L2 Tools
L6  Compression ───┤ 独立于其他层，嵌入 L1 循环
L5  Knowledge ─────┤ 依赖 L2 Tools (SkillTool)
L4  Sub-Agents ────┤ 依赖 L1 Loop (嵌套循环)
L3  Planning ──────┤ 依赖 L2 Tools (PlanTool)
L2  Tools ─────────┤ L1 的直接扩展
L1  Loop ──────────┘ 基础
```

### 15.2 每一层的"如果没有它会怎样"

| 层 | 如果去掉 | 后果 |
|----|---------|------|
| L1 | — | 没有 Agent |
| L2 | 只有 Bash | 无法权限控制、无法并行、UI 退化 |
| L3 | 直接执行 | 任务完成率降低约一半 |
| L4 | 结果堆在主对话 | 上下文污染，长任务后模型迷失 |
| L5 | 所有知识预加载 | token 浪费 + 缓存命中率下降 |
| L6 | 上下文硬上限 | 对话长度受限，长会话不可用 |
| L7 | 无任务追踪 | 多步工作容易遗漏或重复 |
| L8 | 阻塞执行 | 用户必须等着长任务跑完 |
| L9 | 一次性子代理 | 无法维持长期协作、无通信通道 |
| L10 | 无协议 | 队友之间无法协商、权限无法同步 |
| L11 | Leader 手动分配 | Leader 成为瓶颈、并行度降低 |
| L12 | 共享工作目录 | 多 Agent 同时改代码导致冲突 |

---

## 16. 从零构建的思维实验

如果你要从零构建一个类似的系统，推荐的优先级：

```
P0 — 最小可行 Agent（几天可完成）:
  ┌─────────────────────────────┐
  │ L1 Agent Loop               │  while(true) + tool_use
  │ L2 Tool Dispatch            │  Bash + Read + Edit + Grep
  │ L3 Planning                 │  先列步骤再执行
  └─────────────────────────────┘
  → 此时已经能完成大多数单文件编码任务

P1 — 单 Agent 增强（几周）:
  ┌─────────────────────────────┐
  │ L4 Sub-Agents               │  防止上下文污染
  │ L5 Knowledge On Demand      │  CLAUDE.md + Skills
  │ L6 Context Compression      │  autoCompact 为核心
  └─────────────────────────────┘
  → 此时能处理跨文件、长会话的复杂任务

P2 — 后台与持久化（几周）:
  ┌─────────────────────────────┐
  │ L7 Persistent Tasks         │  JSONL + 任务图
  │ L8 Background Execution     │  后台化 + 通知
  └─────────────────────────────┘
  → 此时用户不再被长任务阻塞

P3 — 多 Agent（数月，只在单 Agent 被证明不够时）:
  ┌─────────────────────────────┐
  │ L9  Agent Teams             │  邮箱 + 持久化队友
  │ L10 Team Protocols          │  SendMessage 统一协议
  │ L11 Autonomous Agents       │  自动领取
  │ L12 Worktree Isolation      │  并行隔离
  └─────────────────────────────┘
  → 此时能支持大规模并行协作
```

---

## 17. 对照索引

本文每一层在其他文档中有更深入的覆盖：

| Layer | 本文概述 | 详细文档 |
|-------|---------|---------|
| L1 Agent Loop | §3 | [modules-part1.md](./modules-part1.md) 模块 2-3 |
| L2 Tool Dispatch | §4 | [modules-part1.md](./modules-part1.md) 模块 4, [architecture.md](./architecture.md) |
| L3 Planning | §5 | [modules-part1.md](./modules-part1.md) 模块 4 |
| L4 Sub-Agents | §6 | [modules-part2.md](./modules-part2.md) 模块 15 |
| L5 Knowledge | §7 | [modules-part2.md](./modules-part2.md) 模块 11/14, [system-prompt-design.md](./system-prompt-design.md) |
| L6 Compression | §8 | [advanced-topics.md](./advanced-topics.md) Topic 2 |
| L7 Tasks | §9 | [modules-part1.md](./modules-part1.md) 模块 4 (TaskTools) |
| L8 Background | §10 | [advanced-topics.md](./advanced-topics.md) Topic 3 |
| L9 Teams | §11 | [swarm-multi-agent.md](./swarm-multi-agent.md) §3 |
| L10 Protocols | §12 | [swarm-multi-agent.md](./swarm-multi-agent.md) §5-6 |
| L11 Autonomous | §13 | [swarm-multi-agent.md](./swarm-multi-agent.md) §4, §7 |
| L12 Worktree | §14 | [advanced-topics.md](./advanced-topics.md) Topic 11 |

---

> **总结**: Claude Code 的架构优雅之处不在于单一技术突破，而在于**12 层机制的正确叠加顺序**。每一层都解决前一层留下的局限性，从"一个能调用工具的 while 循环"逐步进化为"一个能自治协作的多 Agent 平台"。理解这个递进关系，比理解任何单个模块都更能把握系统的全貌。
