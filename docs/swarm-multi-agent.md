# Swarm 多 Agent 协作系统深度解析

> **前置阅读**: [modules-part2.md 模块 15](./modules-part2.md) 介绍了 Coordinator 模式的基本概念。本文在此基础上，深入源码级别剖析整个 Swarm 系统的架构设计、通信协议和运行机制。

---

## 目录

1. [系统全景](#1-系统全景)
2. [核心概念与术语](#2-核心概念与术语)
3. [Team 生命周期](#3-team-生命周期)
4. [Agent 执行模型](#4-agent-执行模型)
5. [邮箱通信系统](#5-邮箱通信系统)
6. [权限同步机制](#6-权限同步机制)
7. [任务管理与自动领取](#7-任务管理与自动领取)
8. [优雅关闭协议](#8-优雅关闭协议)
9. [上下文隔离与内存管理](#9-上下文隔离与内存管理)
10. [后端抽象层](#10-后端抽象层)
11. [异常处理与崩溃恢复](#11-异常处理与崩溃恢复)
12. [设计亮点与权衡](#12-设计亮点与权衡)
13. [核心文件速查](#13-核心文件速查)

---

## 1. 系统全景

### 1.1 Swarm 是什么

Swarm（内部代号 **Tengu**）是 Claude Code 的多 Agent 协作框架。它允许一个 **Leader**（领导者）创建并管理多个 **Teammate**（队友），让它们并行执行任务。

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户终端 (TUI)                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Leader   │  │Teammate A│  │Teammate B│  │Teammate C│       │
│  │ 面板     │  │ 面板     │  │ 面板     │  │ 面板     │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │              │              │              │             │
│  ┌────┴──────────────┴──────────────┴──────────────┴──────┐     │
│  │                    AppState (共享状态)                    │     │
│  │   tasks: { taskA, taskB, taskC }                        │     │
│  │   teamContext: { teamName, leadAgentId, teammates }     │     │
│  └────────────────────────┬───────────────────────────────┘     │
│                           │                                      │
│  ┌────────────────────────┴───────────────────────────────┐     │
│  │              文件系统 (磁盘持久化)                        │     │
│  │  ~/.claude/teams/{teamName}/                            │     │
│  │    ├── config.json          # 团队配置                   │     │
│  │    ├── inboxes/             # 邮箱 (消息传递)            │     │
│  │    │   ├── team-lead.json                               │     │
│  │    │   ├── researcher.json                              │     │
│  │    │   └── coder.json                                   │     │
│  │    └── permissions/         # 权限请求                   │     │
│  │        ├── pending/                                     │     │
│  │        └── resolved/                                    │     │
│  └─────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 两种运行模式

| 特性 | In-Process（进程内） | Pane-Based（面板） |
|------|---------------------|-------------------|
| 隔离方式 | AsyncLocalStorage | 独立进程 (tmux/iTerm2) |
| 通信方式 | 内存队列 + 文件邮箱 | 纯文件邮箱 |
| 权限 UI | 复用 Leader 的对话框 | 邮箱中转 |
| 性能 | 低延迟，共享 API 客户端 | 完全隔离，各自初始化 |
| 当前状态 | **主要路径**（生产就绪） | 部分实现中 |

本文主要聚焦 **In-Process** 模式，因为它是当前的主要执行路径。

---

## 2. 核心概念与术语

### 2.1 角色模型

```
┌──────────────────────────────────────────┐
│   Team (团队)                             │
│                                          │
│   ┌─────────┐                            │
│   │ Leader  │  team-lead@my-team         │
│   │ (领导者) │  - 创建/销毁团队            │
│   └────┬────┘  - 分配任务                 │
│        │       - 审批权限请求              │
│        │       - 综合汇报结果              │
│   ┌────┴────────────────┐                │
│   │         │            │                │
│   ▼         ▼            ▼                │
│ ┌─────┐  ┌─────┐  ┌──────────┐          │
│ │队友A │  │队友B │  │队友C     │          │
│ │研究员│  │编码者│  │测试工程师 │          │
│ └─────┘  └─────┘  └──────────┘          │
│                                          │
│ 每个队友有:                               │
│ • 唯一 agentId: "name@teamName"          │
│ • 自己的邮箱 (inbox)                      │
│ • 独立的权限模式                          │
│ • 独立的 AbortController                  │
└──────────────────────────────────────────┘
```

### 2.2 核心数据结构

**TeammateIdentity** — 队友标识：

```typescript
type TeammateIdentity = {
  agentId: string          // "researcher@my-team"（全局唯一）
  agentName: string        // "researcher"
  teamName: string         // "my-team"
  color?: string           // UI 展示颜色
  planModeRequired: boolean // 是否强制 Plan 模式
  parentSessionId: string  // Leader 的会话 ID
}
```

**TeamFile** — 团队持久化配置 (`~/.claude/teams/{name}/config.json`)：

```typescript
type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string         // "team-lead@my-team"
  leadSessionId?: string      // Leader 的实际会话 UUID
  members: Array<{
    agentId: string           // "researcher@my-team"
    name: string              // 邮箱用名
    model?: string            // 模型覆盖
    prompt?: string           // 初始任务描述
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    cwd: string               // 工作目录
    backendType?: BackendType // 'in-process' | 'tmux' | 'iTerm'
    isActive?: boolean        // false = 空闲
    mode?: PermissionMode     // 当前权限模式
    subscriptions: string[]   // 邮箱订阅
  }>
}
```

---

## 3. Team 生命周期

### 3.1 创建团队

```
Leader 调用 TeamCreateTool
          │
          ▼
┌──────────────────────────────────┐
│ 1. 生成确定性 leadAgentId        │
│    formatAgentId("team-lead",    │
│                  teamName)       │
│                                  │
│ 2. 创建团队目录                   │
│    ~/.claude/teams/{teamName}/   │
│    ├── config.json               │
│    ├── inboxes/                  │
│    └── permissions/              │
│                                  │
│ 3. 初始化任务列表目录             │
│    ~/.claude/tasks/{teamName}/   │
│                                  │
│ 4. 注册到 AppState               │
│    appState.teamContext = {      │
│      teamName, leadAgentId,      │
│      teammates: Map()            │
│    }                             │
│                                  │
│ 5. 注册退出清理钩子               │
│    registerTeamForSessionCleanup │
└──────────────────────────────────┘
```

**关键设计**: Leader 的 `agentId` 是确定性的（基于团队名生成），不是随机 UUID。这使得队友始终知道如何找到自己的 Leader。

### 3.2 添加队友

当 Leader 通过 `AgentTool` 或 In-Process Backend 添加队友时：

```
Leader 调用 spawn(config)
          │
          ▼
┌─────────────────────────────────────────┐
│ spawnInProcessTeammate()                 │
│                                         │
│ ① 生成 agentId: "researcher@my-team"    │
│ ② 创建独立 AbortController              │
│    （不与 Leader 联动！）                 │
│ ③ 创建 TeammateContext                  │
│ ④ 注册 InProcessTeammateTaskState       │
│    到 AppState.tasks                    │
│ ⑤ 注册退出清理 handler                  │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ startInProcessTeammate()                 │
│                                         │
│ 进入 runInProcessTeammate() 循环         │
│ （详见第 4 节）                           │
└─────────────────────────────────────────┘
```

> **为什么 AbortController 独立？** 当用户按 Escape 停止 Leader 当前操作时，只会中断 Leader 的本轮对话，不会波及正在工作的队友。这让整个 Swarm 更稳健。

### 3.3 销毁团队

```
Leader 调用 TeamDeleteTool
          │
          ▼
┌──────────────────────────────────────┐
│ ① 检查所有非 leader 成员是否空闲       │
│    （有活跃成员时拒绝删除！）           │
│                                      │
│ ② 清理磁盘目录                        │
│    - 邮箱文件                         │
│    - 权限请求文件                      │
│    - 任务列表                         │
│    - Git worktree（如果有）            │
│                                      │
│ ③ 清除 AppState.teamContext           │
│                                      │
│ ④ 发出遥测: tengu_team_deleted        │
└──────────────────────────────────────┘
```

### 3.4 会话退出清理

当用户退出 Claude Code 或收到 `SIGINT`/`SIGTERM` 时：

```
cleanupSessionTeams()
          │
          ▼
┌───────────────────────────────────┐
│ ① 先杀所有 tmux/pane 队友         │  ← 防止僵尸进程
│ ② 中止所有 in-process 队友        │  ← abortController.abort()
│ ③ 删除团队目录                    │
│ ④ 清理 Git worktree              │
└───────────────────────────────────┘
```

---

## 4. Agent 执行模型

### 4.1 InProcessRunner 主循环

这是整个 Swarm 系统最核心的代码，位于 `src/utils/swarm/inProcessRunner.ts`（~1500 行）。

```
runInProcessTeammate(config)
│
├── 进入 AsyncLocalStorage 上下文
│   runWithTeammateContext(context, async () => {
│
│     while (!abortController.aborted && !shouldExit) {
│
│       ┌──────────────────────────────────────────┐
│       │ Phase 1: 执行当前 Prompt                  │
│       │                                          │
│       │ • 创建本轮 currentWorkAbortController     │
│       │   （Escape 只中断本轮，不杀队友）           │
│       │                                          │
│       │ • 检查是否需要上下文压缩                   │
│       │   （Token 超阈值时自动 compact）           │
│       │                                          │
│       │ • 调用 runAgent() 执行对话循环              │
│       │   - 使用 createInProcessCanUseTool()      │
│       │     进行权限检查                          │
│       │   - 流式处理 API 响应                     │
│       │   - 执行工具调用                          │
│       └──────────────────┬───────────────────────┘
│                          │
│                          ▼
│       ┌──────────────────────────────────────────┐
│       │ Phase 2: 进入空闲状态                     │
│       │                                          │
│       │ • 标记 isIdle = true                     │
│       │ • 触发 onIdleCallbacks                   │
│       │   （通知 Leader 无需轮询）                 │
│       │ • 发送 idle_notification 到 Leader 邮箱   │
│       │   内容: { reason, summary, taskId }      │
│       └──────────────────┬───────────────────────┘
│                          │
│                          ▼
│       ┌──────────────────────────────────────────┐
│       │ Phase 3: 等待下一个指令                    │
│       │                                          │
│       │ waitForNextPromptOrShutdown()             │
│       │ （每 500ms 轮询以下来源）                  │
│       │                                          │
│       │  优先级 1: pendingUserMessages            │
│       │           （来自 UI 注入的消息）            │
│       │                                          │
│       │  优先级 2: 邮箱中的 shutdown_request       │
│       │           （Leader 请求关闭）              │
│       │                                          │
│       │  优先级 3: 邮箱中 Leader 的消息            │
│       │           （新任务指令）                   │
│       │                                          │
│       │  优先级 4: 邮箱中其他队友的消息            │
│       │           （对等通信）                     │
│       │                                          │
│       │  优先级 5: 从任务列表中自动领取任务         │
│       │           tryClaimNextTask()              │
│       └──────────────────┬───────────────────────┘
│                          │
│                          ▼
│       ┌──────────────────────────────────────────┐
│       │ Phase 4: 处理结果                         │
│       │                                          │
│       │ if shutdown_request:                     │
│       │   → 交给模型决定是否同意关闭              │
│       │                                          │
│       │ if new_message:                          │
│       │   → 包装为 <teammate-message> XML         │
│       │   → 进入下一轮循环                        │
│       │                                          │
│       │ if aborted:                              │
│       │   → shouldExit = true，退出循环           │
│       └──────────────────────────────────────────┘
│
│   }) // end runWithTeammateContext
│
└── 返回 InProcessRunnerResult
```

### 4.2 队友状态机

```
                  spawn()
                    │
                    ▼
    ┌───────────────────────────┐
    │    running / isIdle=false │  ◄──── 正在执行 Prompt
    └─────────────┬─────────────┘
                  │ 完成一轮
                  ▼
    ┌───────────────────────────┐
    │    running / isIdle=true  │  ◄──── 等待新指令
    └──┬──────────┬──────────┬──┘
       │          │          │
  新消息到达    关闭请求    abort()
       │          │          │
       ▼          ▼          ▼
    running    模型决策    ┌─────────┐
    isIdle=    ┌──┴──┐    │ killed  │
    false      │     │    └─────────┘
     │       同意   拒绝
     │         │     │
     ▼         ▼     ▼
   (循环)   completed  (继续等待)
```

### 4.3 消息包装格式

队友之间的消息使用 XML 包装以保持结构化：

```xml
<!-- Leader 发给队友的消息 -->
<teammate-message teammate_id="team-lead" color="blue" summary="实现搜索功能">
  请开始实现 src/search.ts 中的搜索功能，要求支持正则表达式...
</teammate-message>

<!-- 队友之间的对等消息 -->
<teammate-message teammate_id="researcher" color="green" summary="发现 API 限制">
  注意：外部 API 有速率限制，每分钟最多 60 次请求...
</teammate-message>
```

---

## 5. 邮箱通信系统

### 5.1 存储结构

每个队友有一个**独立的邮箱文件**（JSON 数组）：

```
~/.claude/teams/{teamName}/inboxes/
├── team-lead.json       # Leader 的收件箱
├── researcher.json      # 研究员的收件箱
├── coder.json           # 编码者的收件箱
└── .lock                # 目录级文件锁
```

单条消息结构：

```typescript
type TeammateMessage = {
  from: string       // 发送者名称
  text: string       // 消息内容（纯文本或 JSON 序列化的结构化消息）
  timestamp: string  // ISO 8601 时间戳
  read: boolean      // 已读标记
  color?: string     // 发送者 UI 颜色
  summary?: string   // 5-10 字摘要
}
```

### 5.2 消息类型

邮箱中传递的不仅仅是文本消息，还有多种**结构化消息**（序列化为 JSON 字符串存储在 `text` 字段中）：

```
┌─────────────────────────────────────────────────────────┐
│                    邮箱消息类型                           │
│                                                         │
│  ┌───────────────────┐  ┌──────────────────────┐       │
│  │ 普通文本消息        │  │ permission_request   │       │
│  │ from → to 的对话   │  │ 工具权限请求          │       │
│  └───────────────────┘  └──────────────────────┘       │
│                                                         │
│  ┌───────────────────┐  ┌──────────────────────┐       │
│  │ permission_response│  │ shutdown_request      │      │
│  │ 权限审批结果        │  │ 请求队友优雅关闭      │       │
│  └───────────────────┘  └──────────────────────┘       │
│                                                         │
│  ┌───────────────────┐  ┌──────────────────────┐       │
│  │ shutdown_approved/ │  │ idle_notification     │      │
│  │ shutdown_rejected  │  │ 队友空闲通知          │       │
│  │ 关闭决策回复        │  │                      │       │
│  └───────────────────┘  └──────────────────────┘       │
│                                                         │
│  ┌───────────────────┐  ┌──────────────────────┐       │
│  │ sandbox_perm_req   │  │ sandbox_perm_resp    │       │
│  │ 沙箱网络访问请求    │  │ 沙箱权限审批结果      │       │
│  └───────────────────┘  └──────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

### 5.3 并发安全

邮箱的并发控制使用 `proper-lockfile` 库：

```
写入流程:
  ① 获取 .lock 文件锁（重试 10 次，退避 5~100ms）
  ② 读取当前 inbox JSON
  ③ 追加新消息
  ④ 原子写回
  ⑤ 释放锁

读取流程:
  无锁读取（允许读到略旧数据，因为队友会持续轮询）
```

### 5.4 SendMessageTool 路由

`SendMessageTool` 是所有 Agent 间通信的统一入口：

```
SendMessage(to, message)
         │
         ▼
┌──────────────────────────────────────────┐
│ 路由决策:                                 │
│                                          │
│ to = "*" (广播)?                          │
│   → 遍历所有队友，写入各自邮箱              │
│                                          │
│ to = 队友名 (in-process)?                 │
│   → 检查 AppState 中队友状态              │
│     ├── 运行中: pendingUserMessages.push()│
│     └── 已停止: resumeAgentBackground()  │
│                                          │
│ to = 队友名 (tmux/pane)?                  │
│   → writeToMailbox(to, message, team)    │
│                                          │
│ to = "bridge:..." / "uds:..."?           │
│   → 跨会话通信（远程连接）                 │
└──────────────────────────────────────────┘
```

---

## 6. 权限同步机制

这是 Swarm 系统中最复杂的子系统之一。当队友需要执行敏感工具（如 Bash、文件写入）时，需要 Leader 审批。

### 6.1 双路径设计

```
队友遇到需审批的工具调用
          │
          ▼
    canUseTool() 返回 'ask'
          │
          ▼
┌──────────────────────────────────────────────────────┐
│ 检查: Leader 的 UI 队列是否可用?                       │
│ (getLeaderToolUseConfirmQueue())                     │
│                                                      │
│          ┌────────┴────────┐                         │
│          │                  │                         │
│    ┌─────▼─────┐    ┌──────▼──────┐                 │
│    │ Path A:   │    │ Path B:     │                 │
│    │ 直接 UI   │    │ 邮箱中转    │                  │
│    │ (快速路径) │    │ (回退路径)  │                  │
│    └─────┬─────┘    └──────┬──────┘                 │
│          │                  │                         │
│          ▼                  ▼                         │
│   加入 Leader 的       序列化请求到                    │
│   ToolUseConfirm      Leader 邮箱                    │
│   队列 (React state)  每 500ms 轮询                  │
│          │            响应邮箱                        │
│          ▼                  ▼                         │
│   Leader UI 展示        Leader 读取                   │
│   对话框 (带队友标识)    pending/ 目录                 │
│          │                  │                         │
│          ▼                  ▼                         │
│   用户审批/拒绝          用户审批/拒绝                 │
│          │                  │                         │
│          ▼                  ▼                         │
│   回调直接返回           写入 resolved/               │
│   给队友               队友轮询获取结果                │
└──────────────────────────────────────────────────────┘
```

### 6.2 Path A: 直接 UI 路径（首选）

当队友与 Leader 在同一进程时，通过 **模块级注册表** 共享 Leader 的 React 状态：

```typescript
// leaderPermissionBridge.ts — 模块级桥接

// Leader 启动时注册:
registerLeaderToolUseConfirmQueue(setQueueFn)
registerLeaderSetToolPermissionContext(setContextFn)

// 队友需要权限时:
const setQueue = getLeaderToolUseConfirmQueue()
if (setQueue) {
  // 直接将请求加入 Leader 的 UI 队列
  setQueue(queue => [...queue, {
    assistantMessage,
    tool,
    input,
    workerBadge: { name: "researcher", color: "green" },  // ← 队友标识
    onAllow(updatedInput, permissionUpdates) {
      // 审批后同步权限规则回 Leader
      const setContext = getLeaderSetToolPermissionContext()
      setContext(newContext, { preserveMode: true })
    },
    onReject(feedback) { /* ... */ }
  }])
}
```

**优点**: 零延迟，复用 Leader 已有的丰富权限 UI（BashPermissionRequest, FileEditDiff 等）。

### 6.3 Path B: 邮箱回退路径

当直接 UI 不可用时（如 tmux 队友），使用文件系统：

```
权限请求存储:
~/.claude/teams/{teamName}/permissions/
├── pending/                    # 待处理请求
│   └── perm-1711929600-a7b3.json
├── resolved/                   # 已处理请求（自动清理）
│   └── perm-1711929100-c4d2.json
└── .lock
```

权限请求数据结构：

```typescript
type SwarmPermissionRequest = {
  id: string                  // "perm-{timestamp}-{random}"
  workerId: string            // 请求者 agentId
  workerName: string          // 请求者名称
  workerColor?: string
  teamName: string
  toolName: string            // "Bash", "Edit" 等
  toolUseId: string           // 原始工具调用 ID
  description: string         // 人类可读的操作描述
  input: Record<string, any>  // 工具参数
  permissionSuggestions: []   // 建议的权限规则
  status: 'pending' | 'approved' | 'rejected'
  createdAt: number
  // 审批后填充:
  resolvedBy?: 'worker' | 'leader'
  resolvedAt?: number
  feedback?: string           // 拒绝原因
  updatedInput?: Record<string, any>  // 修改后的参数
  permissionUpdates?: PermissionUpdate[]  // "始终允许" 规则
}
```

### 6.4 "始终允许" 规则同步

当用户对权限请求选择"始终允许"时，规则不仅需要保存到发起请求的队友，还需要同步到 Leader：

```
用户点击 "Always Allow"
          │
          ▼
┌───────────────────────────────────┐
│ ① 生成 PermissionUpdate 规则      │
│    { toolName, pathPattern, ... } │
│                                   │
│ ② 写入队友的本地权限配置           │
│    （队友后续不再询问同类操作）      │
│                                   │
│ ③ 同步到 Leader 的权限上下文       │
│    setToolPermissionContext()     │
│    使用 preserveMode: true       │
│    （不改变 Leader 自己的权限模式）  │
└───────────────────────────────────┘
```

### 6.5 自动清理

已处理的权限请求文件保留 1 小时后自动清理，防止磁盘累积：

```typescript
cleanupOldResolutions()  // 删除 resolved/ 下超过 1 小时的文件
```

---

## 7. 任务管理与自动领取

### 7.1 任务列表

每个团队有一个共享的任务列表，存储在 `~/.claude/tasks/{teamName}/`：

```typescript
type Task = {
  id: string
  subject: string        // 任务标题
  description: string    // 详细描述
  status: 'pending' | 'in_progress' | 'completed'
  owner?: string         // 领取者的 agentName
  blockedBy?: string[]   // 依赖的任务 ID
  blocks?: string[]      // 被此任务阻塞的任务 ID
}
```

### 7.2 自动领取机制

空闲的队友会**主动从任务列表中领取工作**，无需 Leader 手动分配：

```
队友进入空闲状态 (Phase 3)
          │
          ▼
tryClaimNextTask(taskListId, agentName)
          │
          ▼
┌──────────────────────────────────────┐
│ ① 读取任务列表                        │
│ ② 找到第一个满足条件的任务:            │
│    • status = 'pending'              │
│    • owner 为空                       │
│    • blockedBy 中无未完成任务          │
│ ③ 原子性地领取:                       │
│    owner = agentName                 │
│    status = 'in_progress'            │
│ ④ 格式化为 Prompt:                    │
│    "Complete all open tasks.          │
│     Start with task #3: ..."         │
│ ⑤ 返回给主循环作为下一轮输入           │
└──────────────────────────────────────┘
```

**这个设计很精妙**：队友变成了"自驱动的工人"——Leader 只需创建任务列表，队友会自动分配和执行。减少了 Leader 作为瓶颈的等待时间。

### 7.3 任务完成通知

队友完成任务后，通过 `idle_notification` 通知 Leader：

```json
{
  "type": "idle_notification",
  "agent_name": "researcher",
  "reason": "available",
  "completed_task_id": "3",
  "completed_status": "resolved",
  "summary": "搜索功能实现完成，支持正则和模糊匹配"
}
```

---

## 8. 优雅关闭协议

Swarm 不会粗暴地杀死队友，而是使用一个**协商式关闭协议**：

```
┌──────────┐                    ┌──────────┐
│  Leader   │                    │ Teammate │
└────┬─────┘                    └────┬─────┘
     │                               │
     │  SendMessage(type: shutdown_  │
     │  request, reason: "任务完成")  │
     │──────────────────────────────▶│
     │                               │
     │                               │  模型自主决策:
     │                               │  "我手头还有工作？"
     │                               │
     │                    ┌──────────┴──────────┐
     │                    │                      │
     │              同意关闭                 拒绝关闭
     │                    │                      │
     │    shutdown_approved│    shutdown_rejected │
     │◄───────────────────│   "我正在执行关键操作" │
     │                    │   ──────────────────▶│
     │                    │                      │
     │              abort()                 继续工作
     │              退出循环
     │              发送 idle_notification
     │                    │
     │    从团队中移除成员  │
     │◄───────────────────┘
```

**为什么需要协商？** 如果队友正在执行一个不可中断的操作（比如正在写入文件一半），直接杀掉可能导致数据损坏。协商让队友有机会完成当前操作后再退出。

**如果协商失败？** Leader 仍然可以通过 `kill()` 强制终止——这是最后手段，直接调用 `abortController.abort()`。

---

## 9. 上下文隔离与内存管理

### 9.1 AsyncLocalStorage 隔离

In-Process 队友在同一个 Node.js 进程中运行，使用 `AsyncLocalStorage` 隔离上下文：

```
┌──────────────────────────────────────────────────┐
│                  Node.js 进程                      │
│                                                    │
│  ┌──────────────────┐   ┌──────────────────┐      │
│  │ AsyncLocalStorage │   │ AsyncLocalStorage │      │
│  │ Context A:        │   │ Context B:        │      │
│  │ agentId=researcher│   │ agentId=coder    │      │
│  │ teamName=my-team  │   │ teamName=my-team  │      │
│  │                   │   │                   │      │
│  │ 自己的对话历史     │   │ 自己的对话历史     │      │
│  │ 自己的工具上下文   │   │ 自己的工具上下文   │      │
│  │ 自己的 AbortCtrl  │   │ 自己的 AbortCtrl  │      │
│  └──────────────────┘   └──────────────────┘      │
│                                                    │
│  共享:                                              │
│  • Anthropic API 客户端（高效复用连接）               │
│  • Leader 的 React 状态桥（权限 UI）                 │
│  • AppState（通过属性分区隔离）                       │
└──────────────────────────────────────────────────────┘
```

### 9.2 身份解析优先级

当代码需要知道"我是谁"时，按优先级查找：

```
① AsyncLocalStorage（In-Process 队友）    ← 最高优先级
② 环境变量 CLAUDE_CODE_AGENT_*（tmux 队友）
③ 参数传入的 teamContext（Leader 自身）
```

相关函数：
- `getAgentId()` — 返回当前 Agent 的完整 ID
- `getAgentName()` — 返回名称部分
- `isTeammate()` — 是否在 Swarm 中
- `isTeamLead(teamContext)` — 是否为 Leader

### 9.3 对话历史内存管理

一个关键的实际问题：大型 Swarm 中（比如 10 个队友各 50 轮对话），内存占用会迅速膨胀。解决方案：

```
┌─────────────────────────────────────────────────┐
│ 内存控制策略                                      │
│                                                  │
│ ① UI 消息上限: TEAMMATE_MESSAGES_UI_CAP = 50    │
│    appendCappedMessage() 超限时丢弃最旧消息       │
│                                                  │
│ ② 自动上下文压缩                                  │
│    Token 数超阈值 → compactConversation()         │
│    使用独立 ToolUseContext，不影响 Leader          │
│                                                  │
│ ③ 磁盘溢出                                       │
│    完整对话历史写入 JSONL 文件                     │
│    内存中只保留最近的片段                          │
│                                                  │
│ ④ 队友退出后清理                                  │
│    evictTerminalTask() → 3 秒后从 AppState 移除  │
│    PANEL_GRACE_MS → 30 秒后从 UI 移除            │
└─────────────────────────────────────────────────┘
```

---

## 10. 后端抽象层

### 10.1 后端检测与选择

系统在启动时自动检测可用的终端后端：

```
detectBestBackend()
    │
    ├── 在 tmux 内? ──────────────→ 使用 TmuxBackend
    │
    ├── 在 iTerm2 内?
    │   ├── it2 CLI 可用? ────────→ 使用 ITermBackend
    │   └── tmux 可用? ──────────→ 回退到 TmuxBackend
    │
    ├── tmux 可用? ──────────────→ 使用 TmuxBackend（外部会话）
    │
    └── 都不可用 ─────────────────→ markInProcessFallback()
                                    使用 InProcessBackend
```

**检测结果会被缓存**（`cachedDetectionResult`），整个会话生命周期内不变。

### 10.2 TeammateExecutor 接口

所有后端实现统一的 `TeammateExecutor` 接口：

```typescript
interface TeammateExecutor {
  spawn(config: SpawnConfig): Promise<SpawnOutput>
  sendMessage(agentId: string, message: string): Promise<void>
  terminate(agentId: string, reason?: string): Promise<void>
  kill(agentId: string): Promise<void>
  isActive(agentId: string): boolean
}
```

| 实现 | 文件 | 说明 |
|------|------|------|
| InProcessBackend | `backends/InProcessBackend.ts` | 同进程，AsyncLocalStorage 隔离 |
| TmuxBackend | `backends/TmuxBackend.ts` | tmux 分窗格 |
| ITermBackend | `backends/ITermBackend.ts` | iTerm2 原生分窗格 |

---

## 11. 异常处理与崩溃恢复

### 11.1 队友崩溃

```
队友在执行中抛出未捕获异常
          │
          ▼
┌────────────────────────────────────────┐
│ ① InProcessRunner catch 捕获异常       │
│ ② 设置 task.status = 'failed'         │
│ ③ 发送 idle_notification:              │
│    reason = 'failed'                   │
│    failure_reason = error.message      │
│ ④ Leader 收到通知，可决定:              │
│    • 重新 spawn 一个队友继续            │
│    • 手动接管该任务                     │
│    • 放弃该任务                        │
└────────────────────────────────────────┘
```

### 11.2 Leader 中断（Escape 键）

```
用户按 Escape
     │
     ▼
仅中断 Leader 的 currentWorkAbortController
     │
     ▼
队友不受影响！（因为 AbortController 独立）
     │
     ▼
Leader 回到提示符，可以:
• 查看队友状态
• 发送新消息给队友
• 等待队友完成
```

### 11.3 会话恢复

```
用户使用 /resume 恢复之前的会话
          │
          ▼
┌──────────────────────────────────────┐
│ reconnection.ts                      │
│                                      │
│ ① 读取之前的 TeamFile                │
│ ② 检查哪些队友还在运行               │
│ ③ 重建 TeamContext                   │
│ ④ 重新注册排面                       │
│ ⑤ 恢复通信通道                       │
└──────────────────────────────────────┘
```

---

## 12. 设计亮点与权衡

### 12.1 亮点

| 设计决策 | 为什么巧妙 |
|---------|-----------|
| **Leader ID 确定性生成** | 队友永远知道 Leader 的邮箱地址，无需额外发现机制 |
| **AbortController 独立** | Escape 只中断 Leader，不波及队友；每轮有独立 AbortController |
| **双路径权限** | 同进程走 React 桥（零延迟），跨进程走邮箱（兼容性） |
| **任务自动领取** | 队友空闲时自驱动，减少 Leader 瓶颈 |
| **协商式关闭** | 让模型自己决定是否安全退出，防止数据损坏 |
| **消息优先级** | shutdown > Leader 消息 > 队友消息，确保紧急指令优先处理 |
| **文件锁 + 轮询** | 简单可靠，无需 IPC 守护进程或数据库 |

### 12.2 权衡

| 权衡 | 现状 | 潜在改进方向 |
|------|------|------------|
| **轮询 vs 推送** | 500ms 轮询邮箱 | 可引入 IPC/事件通知减少延迟 |
| **文件系统通信** | JSON 文件 + lockfile | 大规模时可能成为瓶颈（>50 队友） |
| **单进程所有队友** | 共享 CPU/内存 | 可扩展到进程池并行 |
| **消息上限 50 条** | 丢弃旧消息节省内存 | 可能丢失上下文 |
| **权限缓存在文件** | 持久但每次需要 I/O | 可加内存缓存层 |

### 12.3 与 Coordinator Mode 的关系

```
┌─────────────────────────────────────────────────┐
│ Coordinator Mode (coordinatorMode.ts)            │
│ • 提供系统提示词模板                              │
│ • 定义 Leader 的行为准则（研究→综合→实现→验证）    │
│ • 控制 Worker 可使用的工具白名单                  │
│ • 概念层面的"指挥调度"                            │
│                                                  │
│           ↕ 构建在...之上                         │
│                                                  │
│ Swarm 基础设施 (本文档内容)                       │
│ • 团队生命周期管理 (TeamFile, cleanup)            │
│ • Agent 执行循环 (InProcessRunner)               │
│ • 通信协议 (邮箱, SendMessage)                   │
│ • 权限同步 (双路径)                               │
│ • 任务调度 (自动领取)                             │
│ • 实现层面的"如何协作"                            │
└─────────────────────────────────────────────────┘
```

简单说：**Coordinator Mode 是"为什么这样协作"，Swarm 是"如何实现协作"**。

---

## 13. 核心文件速查

| 文件 | 大小 | 职责 |
|------|------|------|
| `src/utils/swarm/inProcessRunner.ts` | ~1500 行 | **核心**: 队友执行主循环 |
| `src/utils/swarm/permissionSync.ts` | ~500 行 | 权限请求的创建、存储、解析 |
| `src/utils/swarm/leaderPermissionBridge.ts` | ~100 行 | Leader React 状态桥接 |
| `src/utils/swarm/spawnInProcess.ts` | ~200 行 | In-Process 队友生成逻辑 |
| `src/utils/swarm/teamHelpers.ts` | ~400 行 | 团队文件 CRUD、清理 |
| `src/utils/swarm/constants.ts` | ~50 行 | 常量定义 |
| `src/utils/swarm/reconnection.ts` | ~150 行 | 会话恢复上下文重建 |
| `src/utils/swarm/backends/registry.ts` | ~200 行 | 后端检测与缓存 |
| `src/utils/swarm/backends/InProcessBackend.ts` | ~150 行 | In-Process 执行器 |
| `src/utils/swarm/backends/types.ts` | ~80 行 | 接口与类型定义 |
| `src/utils/teammateMailbox.ts` | ~300 行 | 邮箱读写与锁管理 |
| `src/utils/teammate.ts` | ~200 行 | 身份解析函数集 |
| `src/utils/teammateContext.ts` | ~100 行 | AsyncLocalStorage 包装 |
| `src/coordinator/coordinatorMode.ts` | ~400 行 | Coordinator 系统提示词 |
| `src/tools/AgentTool/runAgent.ts` | ~200 行 | Agent 执行循环 |
| `src/tools/SendMessageTool/` | ~300 行 | 消息路由与结构化消息处理 |
| `src/tools/TeamCreateTool/` | ~150 行 | 团队创建 |
| `src/tools/TeamDeleteTool/` | ~100 行 | 团队销毁 |
| `src/tasks/InProcessTeammateTask/types.ts` | ~80 行 | 队友任务状态类型 |

### 推荐阅读顺序

1. **`teamHelpers.ts`** — 理解团队的数据模型
2. **`spawnInProcess.ts`** — 队友如何被创建
3. **`inProcessRunner.ts`** — 队友的核心执行循环 (**重点**)
4. **`teammateMailbox.ts`** — 通信协议
5. **`permissionSync.ts`** — 权限同步
6. **`SendMessageTool`** — 消息路由的全貌
7. **`coordinatorMode.ts`** — Leader 的行为准则

---

> **总结**: Swarm 系统的设计哲学是"用最简单的基础设施（文件系统 + 轮询）实现可靠的多 Agent 协作"。它没有选择复杂的 IPC 框架或消息队列，而是用 JSON 文件 + 文件锁 + 500ms 轮询构建了一个实用且可靠的分布式协通系统。这种务实的设计选择值得学习——在正确性和可调试性面前，"高级"技术不一定是更好的选择。
