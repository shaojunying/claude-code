# System Prompt 设计哲学

> **前置阅读**: [architecture.md](./architecture.md) 中"系统提示构建"小节介绍了提示词的基本组装流程。本文深入分析 system prompt 的**设计思想**——为什么这样分层、为什么这样措辞、以及背后的工程权衡。

---

## 目录

1. [总览：Prompt 是一个产品](#1-总览prompt-是一个产品)
2. [分层缓存架构](#2-分层缓存架构)
3. [静态与动态的分界线](#3-静态与动态的分界线)
4. [Section 记忆化系统](#4-section-记忆化系统)
5. [编码风格约束：对抗 LLM 本能](#5-编码风格约束对抗-llm-本能)
6. [工具使用优先级：架构级要求](#6-工具使用优先级架构级要求)
7. [行动安全框架：量两次切一次](#7-行动安全框架量两次切一次)
8. [安全行为雕塑](#8-安全行为雕塑)
9. [输出效率指令](#9-输出效率指令)
10. [内部与外部用户的差异](#10-内部与外部用户的差异)
11. [知识注入策略：按需而非预加载](#11-知识注入策略按需而非预加载)
12. [FRC：教模型"记笔记"](#12-frc教模型记笔记)
13. [隐身模式](#13-隐身模式)
14. [设计原则总结](#14-设计原则总结)
15. [核心文件速查](#15-核心文件速查)

---

## 1. 总览：Prompt 是一个产品

Claude Code 的 system prompt 不是一个简单的指令字符串，而是一个**精心工程化的产品**。它需要同时满足：

```
┌──────────────────────────────────────────────────────────────┐
│                   System Prompt 的多重职责                     │
│                                                              │
│  ① 定义身份  → "You are Claude Code, Anthropic's CLI..."    │
│  ② 约束行为  → 编码风格、安全边界、输出长度                    │
│  ③ 教授技能  → 工具使用方法、Git 操作规范                     │
│  ④ 注入上下文 → 环境信息、记忆、MCP 服务器                    │
│  ⑤ 优化成本  → 分层缓存，减少重复 token 消耗                  │
│  ⑥ 保障安全  → 网络安全准则、行动风险评估                     │
└──────────────────────────────────────────────────────────────┘
```

这六个职责之间存在张力——更多的上下文意味着更好的行为，但也意味着更高的 token 成本和更低的缓存命中率。整个 prompt 架构就是在这个张力中寻找平衡。

---

## 2. 分层缓存架构

### 2.1 三层缓存模型

Claude Code 对 system prompt 实施了**三层缓存策略**：

```
┌─────────────────────────────────────────────────────────┐
│              API 请求的 Prompt 块分布                     │
│                                                         │
│  Block 1: Attribution Header                            │
│  ┌───────────────────────────────────┐                  │
│  │ 计费归因 + 请求指纹               │  cacheScope: null │
│  │ （每次请求都不同）                 │  ← 从不缓存       │
│  └───────────────────────────────────┘                  │
│                                                         │
│  Block 2: System Prompt - 静态部分                       │
│  ┌───────────────────────────────────┐                  │
│  │ 身份 + 规则 + 编码指南 +          │  cacheScope:      │
│  │ 安全准则 + 工具指引 + 风格        │  'global'         │
│  │                                   │  ← 跨用户/跨会话  │
│  │ （所有用户完全相同）               │     共享缓存      │
│  └───────────────────────────────────┘  TTL: 1 小时     │
│                                                         │
│  Block 3: System Prompt - 动态部分                       │
│  ┌───────────────────────────────────┐                  │
│  │ 会话指引 + 记忆 + 环境 +          │  cacheScope:      │
│  │ 语言 + MCP + Skills              │  'org' 或 null    │
│  │                                   │  ← 组织级或无缓存  │
│  │ （每个会话/用户不同）              │                   │
│  └───────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

### 2.2 缓存范围决策

`splitSysPromptPrefix()` 函数（`src/utils/api.ts`）根据上下文选择不同的缓存策略：

```
splitSysPromptPrefix(systemPrompt, options)
          │
          ▼
┌──────────────────────────────────────────────┐
│ 有 MCP 工具?                                  │
│ ├── 是 → 跳过全局缓存                         │
│ │        全部使用 cacheScope: 'org'            │
│ │        （MCP 工具集因用户而异）               │
│ │                                             │
│ └── 否 → 检查: 是否在 1P API + 有资格?         │
│          ├── 是 → 静态部分: 'global'           │
│          │        动态部分: null               │
│          │                                    │
│          └── 否 → 全部 cacheScope: 'org'      │
│                   （3P 提供商无全局缓存）       │
└──────────────────────────────────────────────┘
```

### 2.3 防缓存抖动

为了防止 GrowthBook A/B 实验在会话中途改变代码路径、导致缓存失效：

```
会话启动时:
  ① 评估 TTL 资格 → 锁定到 Bootstrap State
  ② 评估 allowlist → 锁定到 Bootstrap State
  ③ 后续所有请求使用锁定值

结果: 同一会话内缓存行为 100% 一致
```

---

## 3. 静态与动态的分界线

### 3.1 边界标记

System prompt 中有一个**关键标记**将内容分为两部分：

```
__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__
```

位于 `src/constants/prompts.ts` 第 573 行。源码中有明确警告：

> WARNING: Do not remove or reorder this marker without updating cache logic in:
> - src/utils/api.ts (splitSysPromptPrefix)
> - src/services/api/claude.ts (buildSystemPromptBlocks)

### 3.2 分界原则

**边界之前（静态区）**——对所有用户、所有会话完全相同的内容：

| 顺序 | Section | 内容 |
|------|---------|------|
| 1 | Identity | "You are Claude Code, Anthropic's official CLI..." + 网络安全准则 |
| 2 | System Rules | 工具权限、输出格式、hook 系统 |
| 3 | Doing Tasks | 编码实践准则 |
| 4 | Actions Safety | 行动风险评估框架 |
| 5 | Using Your Tools | 工具使用指引 |
| 6 | Tone & Style | 沟通风格 |
| 7 | Output Efficiency | 输出简洁性要求 |

**边界之后（动态区）**——因会话、用户、项目而异的内容：

| Section | 为什么是动态的 |
|---------|--------------|
| Session Guidance | 依赖当前启用的 Agent/Tool 集合 |
| Memory (CLAUDE.md) | 每个项目、每个用户不同 |
| Environment Info | Git 状态、OS、工作目录 |
| Language | 用户语言偏好 |
| Output Style | 可选的解释模式/学习模式 |
| MCP Instructions | MCP 服务器随时连接/断开 |
| Scratchpad | 会话特定的临时目录 |
| FRC Guidance | 根据 Feature Flag 决定是否启用 |
| Token Budget | 动态预算控制 |

### 3.3 为什么这个分界如此重要

一个直觉性的理解：

```
假设静态部分有 3000 tokens，每次 API 调用都相同。

没有缓存分界:
  每次调用 = 3000 tokens × 输入价格 = $$$

有缓存分界:
  第一次调用 = 3000 tokens × 输入价格（写入缓存）
  后续调用  = 3000 tokens × 缓存读取价格（便宜 10 倍）

跨用户全局缓存:
  所有用户共享同一个缓存前缀 → 命中率极高
  相当于 Anthropic 为所有 Claude Code 用户分摊了静态 prompt 成本
```

---

## 4. Section 记忆化系统

### 4.1 两种 Section 类型

`src/constants/systemPromptSections.ts` 定义了两种 Section 工厂：

```typescript
// 安全类型：计算一次，缓存到 /clear 或 /compact
systemPromptSection('memory', () => loadMemoryPrompt())

// 危险类型：每轮重新计算，会破坏 prompt 缓存
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => getMcpInstructions(),
  'MCP servers connect/disconnect between turns'  // ← 必须说明原因
)
```

**为什么要强制写原因？** 因为未缓存的 Section 每轮重计算，如果值发生变化就会**破坏 prompt 缓存**。这是一个有意的摩擦设计——迫使开发者思考是否真的需要每轮更新。

### 4.2 缓存清除时机

```
用户输入 /clear  ──→ clearSystemPromptSectionState() ──→ 所有 Section 重新计算
用户输入 /compact ──→ clearSystemPromptSectionState() ──→ 所有 Section 重新计算
普通对话轮次       ──→ 使用缓存值（除了 DANGEROUS_ 标记的）
```

---

## 5. 编码风格约束：对抗 LLM 本能

这是 system prompt 中最有"产品洞察"的部分。它反映了 Anthropic 团队从大量实际使用中发现的核心问题：**LLM 天然倾向于过度工程化**。

### 5.1 四条反直觉规则

源码 `prompts.ts` 中的关键指令：

```
❌ "Don't add features, refactor code, or make 'improvements' beyond what was asked"
❌ "Don't add error handling for scenarios that can't happen"
❌ "Don't create helpers or abstractions for one-time operations"
❌ "Three similar lines of code is better than a premature abstraction"
```

### 5.2 为什么需要这些约束

```
没有约束的 LLM 行为:

用户: "修复这个 null pointer bug"

LLM 的冲动:
  ① 修复 bug ✅
  ② 顺便重构了周围 3 个函数 ❌
  ③ 加了完善的错误处理 ❌
  ④ 抽取了一个通用工具函数 ❌
  ⑤ 添加了 JSDoc 注释 ❌
  ⑥ 改了变量命名 ❌

结果: 一个 5 行的 bugfix 变成了 200 行的 PR
      代码审查负担增加 10 倍
      引入新 bug 的风险增加
```

### 5.3 具体约束条目

| 约束 | 目的 |
|------|------|
| "A bug fix doesn't need surrounding code cleaned up" | 防止 scope creep |
| "Only validate at system boundaries" | 防止过度防御编程 |
| "Don't design for hypothetical future requirements" | 防止 YAGNI 违反 |
| "Don't add docstrings, comments, or type annotations to code you didn't change" | 防止无关噪音 |
| "Avoid backwards-compatibility hacks like renaming unused _vars" | 防止残留物 |

这些规则的共同哲学：**做被要求的事，仅此而已**。

---

## 6. 工具使用优先级：架构级要求

### 6.1 专用工具优先于 Bash

```
"Do NOT use Bash to run commands when a relevant dedicated tool is provided."

具体映射:
  读文件: Read ← 而非 cat/head/tail
  编辑文件: Edit ← 而非 sed/awk  
  创建文件: Write ← 而非 echo >/cat <<EOF
  搜索文件: Glob ← 而非 find/ls
  搜索内容: Grep ← 而非 grep/rg
```

### 6.2 为什么这不是偏好而是架构要求

```
使用 Bash 读文件:
  bash("cat src/main.ts")
  ↓
  权限系统: 只看到 "Bash 命令"，无法细粒度控制
  UI 渲染: 显示为纯文本输出
  用户体验: 要翻看终端输出理解操作

使用 Read 工具:
  Read("src/main.ts")
  ↓
  权限系统: 知道这是 "读取文件" 操作
  UI 渲染: 语法高亮 + 行号
  用户体验: 一眼看懂操作内容
```

专用工具让**权限系统、UI 渲染、操作审计**都能正常工作。这是整个工具架构的基础假设。

### 6.3 并行调用指引

```
"If you intend to call multiple tools and there are no dependencies between them,
 make all independent tool calls in parallel."

实际效果:
  搜索 3 个文件 → 3 个 Glob 并行 → 总耗时 = max(单次) ≈ 单次
  而非: Glob → 等 → Glob → 等 → Glob → 总耗时 = 3 × 单次
```

---

## 7. 行动安全框架：量两次切一次

### 7.1 核心哲学

```
"Carefully consider the reversibility and blast radius of actions."

"The cost of pausing to confirm is low, while the cost of an unwanted action
 (lost work, unintended messages sent, deleted branches) can be very high."
```

### 7.2 三色分类

```
🟢 可自由执行 — 本地、可逆
   编辑文件、运行测试、读取代码

🟡 需要确认 — 对外可见或共享状态
   git push、创建 PR、发消息、评论 Issue

🔴 高度警惕 — 破坏性、难逆转
   rm -rf、force-push、删除分支、drop table
   git reset --hard、覆盖未提交变更
```

### 7.3 关键细节

```
"A user approving an action (like a git push) once does NOT mean that they
 approve it in all contexts"

含义: 每次都要重新评估上下文，不要因为用户之前批准过就自动执行
```

这和权限系统的 "session memory" 形成了有趣的张力——技术上可以记住授权，但 prompt 要求模型在**语义层面**每次重新判断。

---

## 8. 安全行为雕塑

### 8.1 网络安全准则

位于 `src/constants/cyberRiskInstruction.ts`，由 Safeguards 团队维护：

```
"Assist with authorized security testing, defensive security,
 CTF challenges, and educational contexts."

"Refuse requests for destructive techniques, DoS attacks,
 mass targeting, supply chain compromise, or detection evasion
 for malicious purposes."

"Dual-use security tools require clear authorization context:
 pentesting engagements, CTF competitions, security research,
 or defensive use cases."
```

### 8.2 设计哲学

这段指令的**措辞非常精确**——不是简单的"不准做坏事"，而是：

```
允许:
  ✅ 授权的渗透测试
  ✅ 防御性安全
  ✅ CTF 竞赛
  ✅ 教育目的

拒绝:
  ❌ 破坏性技术
  ❌ DoS 攻击
  ❌ 大规模目标攻击
  ❌ 供应链攻击
  ❌ 恶意目的的检测逃避

灰色地带（双用途工具）:
  ⚠️ C2 框架、凭证测试、漏洞利用开发
  → 必须有明确的授权上下文
```

源码注释要求修改此指令必须经过 Safeguards 团队审批。这意味着每个词都经过了安全评估校准。

---

## 9. 输出效率指令

### 9.1 外部用户版本

```
"Go straight to the point. Try the simplest approach first without going
 in circles. Do not overdo it. Be extra concise."

"Keep your text output brief and direct. Lead with the answer or action,
 not the reasoning. Skip filler words, preamble, and unnecessary transitions."

"If you can say it in one sentence, don't use three."
```

### 9.2 内部用户版本（ant-only）

内部版本有**更具体的数字约束**：

```
"Keep text between tool calls to ≤25 words"
"Keep final responses to ≤100 words"
```

以及更高级的写作指引：

```
"Write for humans who scan: inverted-pyramid structure"
"Avoid jargon, prefer concrete examples"
"Flowproof prose — clear structure"
```

### 9.3 为什么要区分？

```
内部用户 (Anthropic 工程师):
  • 熟悉 Claude 的行为模式
  • 希望极简输出，快速迭代
  • 可以接受"只给答案不解释"
  • 有数字约束帮助控制 token 消耗

外部用户:
  • 可能是第一次用 AI 编码助手
  • 需要适度的解释帮助理解
  • "extra concise" 但不是 "telegraphic"
  • 没有硬性字数限制
```

---

## 10. 内部与外部用户的差异

通过 `process.env.USER_TYPE === 'ant'` 编译时判断：

### 10.1 内部独有功能

| 特性 | 说明 |
|------|------|
| **数字化输出约束** | ≤25 词/工具间隔，≤100 词/最终回复 |
| **验证合约** | 非平凡工作要求独立验证 agent |
| **忠实结果报告** | "Never suppress test failures or errors to make output seem cleaner" |
| **误解提前暴露** | "Surface misconceptions early — collaborator, not executor" |
| **严格注释规范** | "Default to writing no comments. Only add one when the WHY is non-obvious" |
| **模型覆盖** | GrowthBook 配置的内部模型参数 |
| **反馈渠道** | 推荐使用 `/issue` 或 `/share` 报告问题 |

### 10.2 启示

外部用户看到的 Claude Code **不是最优版本**。内部版本通过更严格的约束获得了：
- 更简洁的输出（减少 token 浪费）
- 更可靠的结果（强制验证）
- 更诚实的报告（禁止美化错误）

---

## 11. 知识注入策略：按需而非预加载

### 11.1 传统方式的问题

```
传统 (大多数 AI 编码助手):
  System Prompt = 基础 Prompt + 所有技能 + 所有记忆
  → 每次 API 调用携带全部知识
  → 浪费 token，占据宝贵的上下文窗口

Claude Code 的方式:
  System Prompt = 基础 Prompt + 动态边界 + 环境信息
  Skills  = SkillTool.call()     → 需要时通过 tool_result 注入
  Memory  = CLAUDE.md            → 按目录层级惰性加载
  MCP     = getMcpInstructions() → 只加载已连接服务器的说明
```

### 11.2 CLAUDE.md 的加载层次

```
优先级（由低到高）:
  /etc/claude-code/CLAUDE.md      ← 系统管理员配置
  ~/.claude/CLAUDE.md              ← 用户全局配置
  <project-root>/CLAUDE.md         ← 项目级
  <project-root>/.claude/CLAUDE.md ← 项目配置级
  .claude/rules/*.md               ← 规则目录
  当前目录/CLAUDE.md               ← 目录级
```

越靠近工作目录的文件优先级越高（后加载覆盖先加载）。

### 11.3 Skills 的发现式注入

```
用户输入 → 技能发现预取 (prefetch)
         → 每轮作为 "Skills relevant to your task:" 附件投递
         → 不在 system prompt 中占位

用户明确需要 → SkillTool.call("commit")
             → 技能内容通过 tool_result 注入
             → 使用完毕后可被 FRC 清理
```

---

## 12. FRC：教模型"记笔记"

### 12.1 Function Result Clearing 机制

当对话持续较长时间，旧的工具调用结果会被自动清理以节省上下文空间：

```
"Old tool results will be automatically cleared from context to free up space.
 The N most recent results are always kept."
```

### 12.2 配套的"抢救"指令

```
"When working with tool results, write down any important information you might
 need later in your response, as the original tool result may be cleared later."
```

**这个设计非常精妙**：它不是被动地让信息丢失，而是**主动教模型在信息新鲜时提取关键内容**。就像一个人知道笔记本会被扔掉，所以在看到重要内容时立刻抄写要点。

---

## 13. 隐身模式

### 13.1 什么是 Undercover Mode

当 Claude Code 在**公开/开源仓库**中工作时：

```
isUndercover() → true
  → 从 system prompt 中移除所有模型名称和版本号
  → 防止未发布的模型信息泄露到公开 commit/PR 中
```

### 13.2 为什么需要

```
场景:
  Anthropic 工程师用 Claude Code 给开源项目提交 PR
  Commit 消息或代码注释中包含 "Generated by Claude Opus 4.7"
  → 泄露了尚未发布的模型名称

Undercover Mode:
  自动检测 repo 是否为公开仓库
  是 → 擦除所有模型标识
  默认保守开启（宁可误开也不误关）
```

---

## 14. 设计原则总结

从 system prompt 的整体设计中，可以提炼出以下原则：

| 原则 | 体现 |
|------|------|
| **缓存优先设计** | 静态/动态分界、Section 记忆化、防抖动锁定 |
| **最小权限** | 编码约束（只做被要求的事）、安全准则（只允许授权的） |
| **渐进披露** | 知识按需注入而非预加载、Skills 发现式投递 |
| **Fail Closed** | 工具默认不安全需权限、安全准则默认拒绝灰色地带 |
| **内观性** | FRC 教模型记笔记、Prompt 本身指导模型如何使用工具 |
| **可维护性** | 源码注释要求修改安全指令需审批、边界标记有强制更新要求 |
| **量两次切一次** | 行动安全框架、可逆性优先 |

---

## 15. 核心文件速查

| 文件 | 职责 |
|------|------|
| `src/constants/prompts.ts` | **核心**: System prompt 组装，静态/动态分界 |
| `src/constants/systemPromptSections.ts` | Section 缓存/记忆化工厂 |
| `src/constants/cyberRiskInstruction.ts` | 网络安全准则（Safeguards 团队维护） |
| `src/constants/outputStyles.ts` | 输出风格定义（默认/解释/学习模式） |
| `src/utils/api.ts` | `splitSysPromptPrefix()` — 缓存范围分配 |
| `src/services/api/claude.ts` | `buildSystemPromptBlocks()` — API 块构建 |
| `src/context.ts` | 环境上下文收集（Git 状态、系统信息） |
| `src/memdir/memdir.ts` | MEMORY.md 加载与截断 |
| `src/utils/claudemd.ts` | CLAUDE.md 发现 + @include 处理 |
| `src/utils/betas.ts` | 全局缓存范围资格判断 |
| `src/utils/undercover.ts` | 隐身模式检测 |

---

> **总结**: System Prompt 的设计哲学可以用一句话概括：**用最少的 token 传达最精准的约束，同时让不变的部分尽可能被缓存**。这不仅是 prompt engineering，更是 prompt 层面的性能工程——每个 Section 的位置、每个指令的措辞、甚至每个换行，都在静态缓存命中率和动态灵活性之间做着精确的权衡。
