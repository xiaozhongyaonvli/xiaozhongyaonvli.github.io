---
title: ACP 协议详解：和 MCP 什么关系？如果设计一个 AI IDE 这俩怎么配合？
description: "最近在研究多 Agent 协作的时候，发现除了 MCP 之外，还有个 ACP（Agent Communication Protocol）。刚开始以为是 MCP 的竞品，仔细看下来发现它们解决的压根不是同一层的问题。MCP 管的是\"Ag..."
published: 2026-01-07
tags:
    - ACP
    - MCP
    - AI 协议
category:
    - AI 开发
draft: false
---
> 最近在研究多 Agent 协作的时候，发现除了 MCP 之外，还有个 ACP（Agent Communication Protocol）。刚开始以为是 MCP 的竞品，仔细看下来发现它们解决的压根不是同一层的问题。MCP 管的是"Agent 怎么调工具"，ACP 管的是"Agent 之间怎么对话"。这篇先把 ACP 的核心概念捋一遍，跟 MCP 做个详细对比，最后用"设计一个 AI IDE"这个场景来说明两个协议到底怎么配合使用。

## 先聊问题：为什么需要 Agent 间通信协议？

现在做 AI Agent 的框架越来越多——LangChain、CrewAI、AutoGen、BeeAI... 每个框架内部的 Agent 协作都没问题，但**跨框架的 Agent 怎么协作**？

举个具体场景：

```
你的公司有三个 AI Agent：
  - 客服 Agent（用 LangChain 搭的）
  - 订单查询 Agent（用 CrewAI 搭的）
  - 退款审批 Agent（内部 Java 系统）

用户说："我上周买的东西质量有问题，要退款"

客服 Agent 需要：
  1. 调用订单查询 Agent → 查到订单信息
  2. 把订单信息和退款理由发给退款审批 Agent
  3. 拿到审批结果后回复用户

问题是：这三个 Agent 用不同框架写的，消息格式不一样，怎么通信？
```

以前的做法要么是硬编码适配（A 调 B 写一套，A 调 C 再写一套），要么全部塞到一个框架里（不现实，因为可能是不同团队甚至不同公司开发的）。

**ACP 就是来解决这个"Agent 间通信标准"问题的**——定义一套统一的消息格式和交互模式，不管 Agent 用什么框架实现的，都能互相对话。

## ACP 是什么？

ACP 全称 Agent Communication Protocol（Agent 通信协议），由 IBM Research 发起，现在已经捐给了 Linux Foundation 做开放治理。

一句话概括：**ACP 是 Agent 之间的"普通话"标准**。

类比一下：
- **MCP** = 定义了"人怎么使用工具"的标准（锤子怎么握、扳手怎么拧）
- **ACP** = 定义了"人和人之间怎么沟通"的标准（说什么语言、遵循什么对话规则）

### 核心设计原则

**1. 基于 REST**——不需要专门的 SDK 就能用

ACP 用的是标准 HTTP REST 接口，不是 JSON-RPC。这意味着你拿 curl、Postman 甚至浏览器就能跟 Agent 通信，入门门槛极低：

```bash
curl -X POST http://agent-server/runs \
  -H "Content-Type: application/json" \
  -d '{
    "agent_name": "order-query",
    "input": [{
      "parts": [{
        "content": "查询订单 ORD-2025-001 的状态",
        "content_type": "text/plain"
      }]
    }]
  }'
```

（跟 MCP 的 JSON-RPC 比起来，确实更"Web 开发者友好"。）

**2. 多模态原生支持**

ACP 用 MIME Type 标识内容类型，天然支持文本、图片、音频、视频、JSON 等任意格式，不需要协议层面的扩展：

```json
{
  "parts": [
    { "content": "这是用户上传的故障截图", "content_type": "text/plain" },
    { "content": "<base64图片数据>", "content_type": "image/png" },
    { "content": "{\"orderId\": \"ORD-001\"}", "content_type": "application/json" }
  ]
}
```

**3. 异步优先**

Agent 的任务经常是长时间运行的（比如"分析这份 50 页的合同"），ACP 的默认模式就是异步——发出请求后拿到一个 `taskId`，然后轮询或订阅结果：

```
同步模式（简单任务）：
  POST /runs/wait → 等着拿结果

异步模式（长任务）：
  POST /runs      → 返回 { "run_id": "xxx" }
  GET  /runs/xxx  → 查询状态：running / completed / failed

流式模式（实时更新）：
  POST /runs/stream → SSE 推送中间结果
```

> 这三种模式的选择权在调用方。简单的查询用同步，复杂的分析用异步，需要实时反馈的用流式。

## ACP 的架构

### 核心概念

```
ACP 生态

  Agent Client (调用方)
      ↓ HTTP
  ACP Server (协议代理)
      ↓
  Agent Registry (Agent 注册表)
      ↓
  Agent A (LangChain)    Agent B (CrewAI)    Agent C (自定义)
```

- **Agent Client**：调用方，想跟某个 Agent 对话的人/系统
- **ACP Server**：协议代理，负责路由消息、管理 Agent 注册、处理认证授权
- **Agent Registry**：Agent 的"通讯录"，记录每个 Agent 的能力、地址、状态
- **Agents**：实际干活的 Agent，可以用任何框架实现，只要遵守 ACP 的消息格式

### Agent 的发现机制

跟 MCP 的 `tools/list` 类似，ACP 的 Agent 也需要"自我介绍"。但 ACP 的做法是通过 **Agent 元数据** 来注册：

```json
{
  "name": "order-query-agent",
  "description": "查询订单状态、物流信息、退换货进度",
  "version": "1.2.0",
  "capabilities": ["order-status", "logistics-tracking", "return-status"],
  "input_content_types": ["text/plain", "application/json"],
  "output_content_types": ["text/plain", "application/json"],
  "modalities": ["sync", "async", "stream"]
}
```

有意思的是，ACP 支持**离线发现**——Agent 不需要在线才能被发现，它的元数据可以打包在分发包里，通过注册表静态索引。这对 Serverless 架构很友好（Agent 可以被按需拉起，不需要常驻）。

### Agent 生命周期

```
INITIALIZING → ACTIVE → DEGRADED → RETIRING → RETIRED

INITIALIZING  启动中
ACTIVE        正常工作
DEGRADED      能用但有问题（比如依赖的服务不稳定）
RETIRING      正在下线，不接新请求
RETIRED       已下线
```

> 这套生命周期管理在 MCP 里是没有的。MCP 的 Server 要么在要么不在，没有"降级"的概念。ACP 考虑到了生产环境里 Agent 可能部分可用的情况。

## ACP vs MCP：详细对比

这是大家最关心的。先说结论：**它们不是竞品，是互补的，解决不同层的问题**。

### 定位差异

```
ACP 的领域 → Agent ←→ Agent 通信（多 Agent 协作、任务委派）
MCP 的领域 → Agent ←→ Tool/Data 通信（调工具、查数据、读资源）
```

用一个具体例子说明：

```
场景：用户问"帮我查一下北京今天的天气，然后订一张明天去上海的机票"

这里涉及两种协议：

1. "天气 Agent" 和 "机票 Agent" 之间怎么协作？
   → 需要 ACP（Agent 间通信）
   → 天气 Agent 把结果发给机票 Agent，说"用户查了天气，现在需要订机票"

2. "天气 Agent" 怎么调天气 API？"机票 Agent" 怎么调订票接口？
   → 需要 MCP（Agent 调工具）
   → 天气 Agent 通过 MCP 调用 weather_api 工具
   → 机票 Agent 通过 MCP 调用 booking_api 工具
```

### 技术层面对比

| 维度 | MCP | ACP |
|------|-----|-----|
| **发起方** | Anthropic（2024.11） | IBM Research（2025.05） |
| **解决什么** | Agent 怎么调工具/数据 | Agent 之间怎么通信 |
| **传输协议** | JSON-RPC 2.0 | RESTful HTTP |
| **通信模式** | 同步为主 + SSE 流式 | 同步 / 异步 / 流式 三种 |
| **状态管理** | 无状态（单次请求） | 有状态（Session 支持） |
| **内容格式** | JSON 为主 | MIME Type 多模态 |
| **发现机制** | `tools/list` 获取工具列表 | Agent Registry 注册表 |
| **生命周期** | 无（Server 在/不在） | 完整生命周期管理 |
| **治理** | Linux Foundation | Linux Foundation |
| **生态规模** | 10000+ MCP Server | 早期阶段 |

### 消息格式对比

**MCP 的调用方式（JSON-RPC）：**

```json
// Agent 通过 MCP 调用一个工具
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "city": "北京" }
  },
  "id": 1
}
```

MCP 的消息是"调用函数"的语义——我要调哪个工具、传什么参数。它不关心调用方是谁、有什么背景。

**ACP 的调用方式（REST）：**

```json
// Agent A 给 Agent B 发消息
POST /runs
{
  "agent_name": "weather-agent",
  "input": [
    {
      "parts": [
        {
          "content": "请查询北京今天的天气，用户需要这个信息来决定明天的出行计划",
          "content_type": "text/plain"
        }
      ],
      "role": "user"
    }
  ],
  "session_id": "session-abc-123"
}
```

ACP 的消息是"对话"的语义——我是谁、我的上下文是什么、我期望你做什么。有 Session 概念，能维持跨消息的状态。

> 我理解的差异核心是：MCP 把被调用方当"工具"，是无脑执行的；ACP 把被调用方当"Agent"，是有自主判断能力的。这决定了它们的消息格式一个像 RPC 调用，一个像对话交流。

### 什么时候用 MCP，什么时候用 ACP？

| 场景 | 选择 | 原因 |
|------|------|------|
| Agent 需要查数据库、调 API | MCP | 这是工具调用，MCP 的强项 |
| Agent 需要读文件、搜索网页 | MCP | 同上，工具/资源接入 |
| 两个 Agent 需要协作完成一个任务 | ACP | 需要对话、委派、状态跟踪 |
| 多个 Agent 组成工作流 | ACP | 需要任务编排和生命周期管理 |
| 单 Agent + 多工具 | MCP 就够了 | 不需要 Agent 间通信 |
| 多 Agent + 多工具 | MCP + ACP | MCP 管工具接入，ACP 管 Agent 协作 |

**一个实际的多 Agent 系统架构：**

```
用户请求
  ↓
编排 Agent（Orchestrator）
  → ACP → 分析 Agent → MCP → 日志系统
  → ACP → 检索 Agent → MCP → 知识库
  → ACP → 执行 Agent → MCP → 工单系统
```

Agent 之间用 ACP 协作（"我查到了这些信息，你来分析一下"），每个 Agent 各自通过 MCP 调用自己需要的工具。两层协议各管各的。

## 番外：ACP 和 A2A 合并了

2025 年还有个重要事件：IBM 的 ACP 和 Google 的 A2A（Agent-to-Agent）在 Linux Foundation 下**合并**了。

A2A 是 Google 在 2025 年 4 月发布的，也是解决 Agent 间通信的协议。两个协议目标一样，各有各的思路：

- ACP 更偏向**本地优先**（local-first），适合同一集群内的 Agent 协作
- A2A 更偏向**跨组织**，适合不同公司/供应商的 Agent 在公网上协作

最后两边决定合力而不是内耗，ACP 团队从 2025 年 9 月开始并入 A2A 项目。现在 A2A 是 Agent 间通信的统一标准，继承了 ACP 的一些设计理念（REST 原生、多模态、异步优先）。

> 所以现在的全景图是：**MCP 管 Agent↔Tool，A2A（含 ACP 的精华）管 Agent↔Agent**。两者互补，不冲突。

## 整个协议栈长什么样？

把 MCP、ACP/A2A 放在一起看，AI Agent 的协议栈正在形成一个清晰的分层：

```
应用层
  Agent 框架（LangChain / CrewAI / AutoGen）
        ↓
Agent 间通信层（A2A / 原 ACP）
  Agent 发现 / 任务委派 / 消息路由 / 状态管理
        ↓
Agent-工具通信层（MCP）
  工具调用 / 资源读取 / Prompt 模板
        ↓
模型层
  LLM（Claude / GPT / Gemini / 开源模型）
        ↓
基础设施层
  HTTP / WebSocket / gRPC / SSE
```

每一层解决不同的问题，互不替代。之前写过 Function Calling、MCP、Skills 三者的区分，现在再加上 ACP/A2A 这一层，整个拼图就更完整了。

## 落地场景：如果让我设计一个 AI IDE

光聊协议太抽象，拿一个具体场景来说明——如果让我从零设计一个 AI IDE（类似 Cursor、Windsurf），ACP 和 MCP 分别扮演什么角色。

### 先看现在的 AI IDE 是怎么做的

**Cursor 的 Shadow Workspace**

Cursor 最核心的技术叫 Shadow Workspace（影子工作区）——在后台开一个隐藏的编辑器窗口，AI 在这个隐藏窗口里改代码、跑 Language Server 做 Lint 检查，确认没问题之后才把结果给用户看。

```
用户的编辑器窗口（正常使用）
  ↓ AI 提议修改
Shadow Window（隐藏窗口）
  → 应用代码修改
  → 跑 Language Server（类型检查、Lint）
  → 收集错误信息
  → 有错？→ AI 自我修正 → 重新检查
  → 没错？→ 把修改呈现给用户
```

用户感知到的是"AI 一次就给了正确的代码"，实际上 AI 可能在影子窗口里改了好几轮才通过 Lint。

**Windsurf 的 Cascade Engine**

Windsurf 走的另一条路——它的 Cascade 引擎先把代码库做静态分析，生成**依赖图**（AST 级别）。AI 改代码的时候不是"盲改"，而是知道"改了 A 文件，B 和 C 文件会受影响"，然后多文件同时修改。

> 两者的设计哲学差异挺明显的：Cursor 偏保守（改尽量少的代码、增量修复），Windsurf 偏激进（诊断确认后大范围改）。

### AI IDE 的核心难题：代码幻觉

AI IDE 最大的问题就是**幻觉**——编造不存在的 API、引用不存在的包、写出语法正确但逻辑完全错的代码。代码幻觉尤其阴险，因为它可能**语法完全正确、命名看起来合理**：

```python
from pandas import DataFrame
df = DataFrame(data)
df.smart_fillna(method="contextual")  # ← 这个方法不存在，AI 编的
```

LLM 本质上是概率模型——它不是在"查文档"，它是在"猜下一个 token"。当训练数据里某个模式出现频率高，它就倾向于生成类似的东西，哪怕在当前上下文里是错的。

防幻觉大概分三层：

**第一层：生成前——给足上下文（Grounding）**

```
做法：
  1. RAG 检索项目文档和依赖库文档 → AI 基于真实文档生成
  2. 代码库 embedding 索引 → 改 UserService 前先检索它的定义和使用方式
  3. LSP 信息注入 → 把 Language Server 的类型信息、方法签名喂给 AI
```

> 思路就是：**与其让 AI 回忆，不如让 AI 查文档**。

**第二层：生成后——自动验证**

```
验证手段：
  1. LSP Lint 检查 → 类型错误、未定义变量、import 不存在
  2. 编译/解释检查 → 编译型语言直接尝试编译
  3. 单元测试 → AI 生成代码的同时生成测试，跑一遍
  4. 多次生成 + 投票 → 多个独立生成结果一致，大概率是对的
```

**第三层：人机协作——标记 + 审查**

AI 生成的代码标记来源、Code Review 时重点看 AI 生成的部分、IDE 里对 AI 生成的 import 自动高亮（如果该 import 在项目里从未出现过）。

### 我的 AI IDE 架构设计：MCP + ACP 两层协议

现在把前面聊的 ACP 和 MCP 落到 IDE 这个场景里，架构就是这样的：

```
AI IDE 整体架构

用户交互层
  编辑器 UI / 终端 / 聊天面板 / Diff 视图
        ↓
Agent 编排层（用 A2A/ACP 协作）
  → A2A → 代码生成 Agent
  → A2A → 代码审查 Agent
  → A2A → 测试生成 Agent
  → A2A → 文档/解释 Agent
        ↓
MCP 工具接入层（每个 Agent 都能调）
  文件系统 MCP Server    Git MCP Server      终端 MCP Server
  LSP MCP Server         代码搜索 MCP Server  文档检索 MCP Server
        ↓
验证层（Shadow Workspace）
  LSP 检查 / 编译验证 / 测试运行 / 幻觉检测
```

**MCP 层——让 AI "看得见" 项目**

MCP 在这里解决的是 Agent → 工具/数据 的连接。每个 Agent 都能通过 MCP 调用：

- **文件系统 MCP Server**：读写文件、搜索文件、查看目录结构
- **Git MCP Server**：查看 diff、commit 历史、分支信息
- **LSP MCP Server**（防幻觉核心）：获取类型定义、方法签名、诊断信息
- **终端 MCP Server**：执行命令、运行测试
- **文档检索 MCP Server**：RAG 检索项目文档、依赖库文档
- **代码搜索 MCP Server**：语义搜索 + 关键词搜索

其中 **LSP MCP Server** 是对防幻觉最有价值的：

```
AI 想调用 user.getName()

没有 LSP 信息时：
  → AI 从训练数据"回忆"，觉得 User 类应该有 getName() → 可能是幻觉

有 LSP 信息时：
  → MCP 调用 LSP → LSP 返回 User 类的完整方法列表
  → AI 看到 User 类只有 getFullName() 没有 getName()
  → AI 生成 user.getFullName() → 准确
```

**用真实的 LSP 数据替代模型的"记忆"**，这是代码场景下 Grounding 最直接的手段。

> 为什么用 MCP 而不是直接写 API？因为 MCP 的 `tools/list` 机制让工具可以自描述——新增一个 MCP Server 不需要改 Agent 代码。这跟 LSP 的思路一脉相承：**把 M×N 问题变成 M+N 问题**。

**A2A/ACP 层——多个 Agent 各司其职**

单 Agent 做所有事情 context 会爆炸，而且不同任务需要的"思维模式"不一样——写代码需要创造力，Code Review 需要谨慎，写测试需要找边界条件。所以拆成多个 Agent，用 A2A/ACP 协作：

```
用户说："帮我实现用户注册功能"

编排 Agent（Orchestrator）
  ↓
  → A2A → 代码生成 Agent
            → MCP → 文件系统（读现有代码）
            → MCP → LSP（获取类型信息）
            → MCP → 文档检索（查 ORM 文档）
            → 生成代码 → 通过 A2A 发给审查 Agent
  ↓
  → A2A → 代码审查 Agent
            → 收到代码 → MCP → LSP（跑 Lint 检查）
            → 检查安全漏洞（SQL 注入？XSS？）
            → 有问题 → A2A 反馈给生成 Agent，要求修改
            → 没问题 → A2A 发给测试 Agent
  ↓
  → A2A → 测试生成 Agent
            → MCP → 终端（跑测试）
            → 测试不过 → A2A 反馈给生成 Agent
            → 测试通过 → A2A 通知编排 Agent → 呈现给用户
```

A2A 协议里的几个概念在 IDE 场景下特别有用：

- **Agent Card**：每个 Agent 声明自己的能力。编排 Agent 通过 Agent Card 知道"审查 Agent 能检查 Java 和 Python，但不支持 Rust"
- **Task 生命周期**（`submitted → working → completed/failed`）：用户可以在 IDE 面板里看到每个 Agent 的工作状态
- **Artifact**：Agent 的产出物。代码生成 Agent 的 artifact 是代码文件，测试 Agent 的 artifact 是测试报告

> 为什么用 A2A/ACP 而不是在一个 Agent 里用多轮 prompt？两个原因：1）每个 Agent 的 system prompt 可以针对性优化；2）Agent 之间的消息有 Session 概念，可以跟踪整个协作过程的状态。

**验证层——幻觉专项检测**

在 Shadow Workspace 的基础上，加一层专门针对 AI 幻觉的检测：

```python
class CodeHallucinationDetector:

    def check_imports(self, generated_code, project_deps):
        """检查 AI 生成的 import 是否在项目依赖中"""
        imports = extract_imports(generated_code)
        for imp in imports:
            if imp.package not in project_deps:
                yield Warning(f"引用了项目未安装的包: {imp.package}")

    def check_api_calls(self, generated_code, lsp_symbols):
        """检查 AI 调用的方法是否真实存在"""
        calls = extract_method_calls(generated_code)
        for call in calls:
            if call.method not in lsp_symbols.get(call.receiver_type, []):
                yield Warning(f"调用了不存在的方法: {call.receiver_type}.{call.method}")
```

这比单纯的 Lint 检查更有针对性——Lint 查语法和类型规范，幻觉检测查"这个东西在当前项目里存不存在"。

### 两层协议在 IDE 里的分工总结

| | MCP | A2A/ACP |
|--|-----|---------|
| 解决什么 | Agent 怎么使用 IDE 里的工具 | Agent 之间怎么协作 |
| 在 IDE 里的角色 | "手"——读文件、查类型、跑终端 | "嘴"——生成 Agent 和审查 Agent 对话 |
| 消息语义 | 函数调用：`tools/call("lsp_get_definitions", ...)` | 对话交流：`"我生成了这段代码，请审查安全性"` |
| 状态 | 无状态，调完就完 | 有状态，Session 贯穿整个任务 |

### 实际落地要考虑的

真要做还有几个现实问题：

- **延迟**：多 Agent 来回传消息肯定比单 Agent 慢。简单任务直接用单 Agent + MCP，复杂任务再启动多 Agent 协作
- **成本**：审查和测试 Agent 可以用便宜的小模型，只有代码生成 Agent 用大模型。大部分 MCP 工具调用不涉及 LLM，成本为零
- **上下文管理**：100 万行代码不可能全塞进 context，靠依赖图 + RAG 检索按需获取
- **安全**：MCP 工具有执行权限，所有操作需用户确认或做沙箱隔离

## 小结

ACP 和 MCP 的关系用一句话概括：**MCP 让 Agent 能"用工具"，ACP 让 Agent 能"找同事"**。MCP 管的是纵向的 Agent→工具接入（向下），ACP/A2A 管的是横向的 Agent↔Agent 协作（平级）。做单 Agent 应用只需要 MCP，做多 Agent 协作系统两个都要用。

用 AI IDE 这个场景来看就很直观——MCP 接入文件系统、LSP、终端等开发工具，让每个 Agent 能"看见"项目、"操作"代码；A2A/ACP 让代码生成、审查、测试等多个 Agent 协作配合，各司其职。两层协议各管一层，谁也替代不了谁。

现在 ACP 已经并入 A2A 成为 Linux Foundation 下的统一标准，加上 MCP 也进了 Linux Foundation，整个生态正在走向标准化——对开发者来说是好事，以后不用再为"选哪个协议"纠结了。
