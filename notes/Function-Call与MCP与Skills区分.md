---
title: Function Call、MCP、Skills —— 三个容易搞混的概念，一次理清
date: 2025-12-19 13:45:00
tags:
  - AI
  - Function Call
  - MCP
categories:
  - AI 开发
---

# Function Call、MCP、Skills —— 三个容易搞混的概念，一次理清

> 最近在折腾 AI Agent 相关的东西，这三个词反复出现：Function Calling、MCP、Skills。刚开始觉得"不都是让大模型调工具嘛"，但仔细研究下来发现它们解决的问题、工作层级、设计哲学完全不同。记录一下我的理解。

## 先来一个直觉类比

在动手拆解之前，先用一个不太严谨但很好理解的类比：

- **Function Calling** = 你教一个员工怎么**打电话**（拨号、说话、挂断），每次都得手把手教
- **MCP** = 公司装了一套**标准电话系统**，不管哪个员工都能直接用，不用每次都教
- **Skills** = 给员工一份**操作手册**（"接到客户投诉时，先安抚情绪，再查订单，最后给方案"），员工自己判断怎么执行

三者的层级不同：Function Calling 是底层能力，MCP 是标准化协议，Skills 是上层行为编排。

## Function Calling（函数调用）

### 是什么？

Function Calling 是 LLM 厂商提供的一种能力，让大模型不只是吐文本，还能"决定调用哪个函数、传什么参数"。2023 年 OpenAI 最先推出，现在 Anthropic、Google、Mistral 等主流厂商都支持。

> 有个关键点容易误解：**LLM 自己不执行函数**。它只是看了你的问题后说"我觉得应该调 `search_products` 这个函数，参数是 `{"keywords": ["laptop"]}`"，然后把这个 JSON 结构化输出丢给你的应用程序，由你的代码去真正执行。

### 工作流程

整个过程是一个"人-模型-工具"的循环：

```

 1. 用户: "帮我查一下北京今天的天气"            
                      ↓                         
 2. 应用把问题 + 工具定义列表 发给 LLM           
                     ↓                         
 3. LLM 分析意图，返回结构化 JSON:              
     {                                         
       "function": "get_weather",              
       "arguments": {"city": "北京"}            
     }                                         
                     ↓                         
  4. 应用代码拿到 JSON，执行真正的 API 调用       
     response = weather_api.get("北京")         
                     ↓                         
  5. 把执行结果喂回给 LLM                       
                     ↓                         
  6. LLM 结合结果生成自然语言回答:               
     "北京今天晴，最高温度26°C..."               
```

### 代码长什么样？

以 OpenAI 的 API 为例，你需要做两件事——**定义工具 schema** 和 **处理模型的调用决策**：

```python
# 1. 定义工具的 JSON Schema
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询指定城市的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，如'北京'、'上海'"
                    }
                },
                "required": ["city"]
            }
        }
    }
]

# 2. 调用 LLM，带上工具列表
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "北京天气怎么样？"}],
    tools=tools
)

# 3. LLM 返回的不是文本，而是函数调用指令
tool_call = response.choices[0].message.tool_calls[0]
# tool_call.function.name == "get_weather"
# tool_call.function.arguments == '{"city": "北京"}'

# 4. 你的代码去真正执行
result = call_weather_api(city="北京")

# 5. 把结果喂回去，让 LLM 组织最终回答
final = client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "user", "content": "北京天气怎么样？"},
        response.choices[0].message,  # 包含 tool_call 的消息
        {"role": "tool", "tool_call_id": tool_call.id, "content": result}
    ]
)
print(final.choices[0].message.content)
# → "北京今天晴，气温26°C，适合出行。"
```

### 局限性

Function Calling 的问题在于——**每家厂商的实现不一样**。OpenAI 的 `tools` 参数、Anthropic 的 `tool_use`、Google 的 `function_declarations`，格式都有差异。

如果你的应用只用一个模型，这不是问题。但如果你想支持多个模型（比如用户可以选 GPT-4 或 Claude），你就得**为每个模型写一套适配代码**。模型有 M 个，工具有 N 个，你就要维护 M × N 个集成——这就是所谓的 **M×N 问题**。

MCP 就是来解决这个问题的。

## MCP（Model Context Protocol）

### 是什么？

MCP 全称 Model Context Protocol（模型上下文协议），2024 年 11 月由 Anthropic 开源发布。一句话概括：**它是 AI 界的 USB-C 接口**。

USB-C 出现之前，不同设备用不同的充电线——安卓用 Micro-USB，苹果用 Lightning，笔记本用各种圆头。USB-C 出来之后，一根线走天下。

MCP 做的就是同样的事情：在 LLM 和外部工具之间定义一个**标准化的通信协议**，不管你是什么模型（Claude / GPT / Llama），不管你对接什么工具（GitHub / 数据库 / 搜索引擎），都用同一套协议来通信。

```
没有 MCP:                        有了 MCP:

Claude → GitHub 适配器           Claude
Claude → Slack 适配器            GPT-4   → MCP 协议 → GitHub MCP Server
GPT-4  → GitHub 适配器           Llama                → Slack MCP Server
GPT-4  → Slack 适配器                                 → DB MCP Server
Llama  → GitHub 适配器
Llama  → Slack 适配器            M×N → M+N
```

### 架构：三层结构

MCP 的架构灵感来自 LSP（Language Server Protocol，VS Code 用的那个语言服务协议），采用 **Host - Client - Server** 三层结构：

```
MCP Host (Claude Desktop / Cursor 等)
  MCP Client A          MCP Client B
      ↓                     ↓
  JSON-RPC 2.0          JSON-RPC 2.0
      ↓                     ↓
  MCP Server (GitHub)   MCP Server (数据库)
```

- **Host**：宿主应用，就是用户直接交互的那个 AI 应用（比如 Claude Desktop、Cursor IDE）。它负责管理 LLM 的上下文、决定什么时候调工具
- **Client**：连接管理器，每个 Client 对应一个 Server。负责会话管理、能力协商、消息路由
- **Server**：工具的具体实现方。GitHub 写一个 MCP Server，Slack 写一个，数据库写一个... 每个 Server 只需要实现一次，所有 Host 都能用

通信用的是 **JSON-RPC 2.0**，传输层支持两种方式：
- **STDIO**：本地进程间通信，Server 跑在本机
- **HTTP + SSE**：远程通信，Server 部署在云端

### 三大原语（Primitives）

MCP Server 能向 Host 暴露三种东西：

**1. Tools（工具）** —— AI 可以调用的动作

```json
{
  "name": "create_issue",
  "description": "在 GitHub 仓库创建一个 Issue",
  "inputSchema": {
    "type": "object",
    "properties": {
      "repo": {"type": "string"},
      "title": {"type": "string"},
      "body": {"type": "string"}
    }
  }
}
```

这个最好理解，就是 Function Calling 里的"函数"，但是用 MCP 标准格式描述的。

**2. Resources（资源）** —— 提供上下文数据

```
resource://github/repos/my-project/README.md
resource://database/users/schema
```

Resources 是被动的数据源，不需要 AI 主动"调用"，而是 Host 可以拉取它来补充上下文。比如打开一个代码仓库时，自动把 README 和目录结构加载到上下文里。

**3. Prompts（提示模板）** —— 预定义的交互模式

```json
{
  "name": "code_review",
  "description": "代码审查模板",
  "arguments": [
    {"name": "language", "description": "编程语言"},
    {"name": "code", "description": "待审查的代码"}
  ]
}
```

Prompts 是预定义的提示词模板，帮助 AI 在特定场景下用更好的方式处理任务。

> 我觉得这个三层原语的设计挺清晰的——Tools 负责"做事"，Resources 负责"给信息"，Prompts 负责"教方法"。

### MCP 实际配置长什么样？

以 Claude Desktop 为例，在配置文件里加一段 JSON 就能接入一个 MCP Server：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
    }
  }
}
```

配置好之后，Claude 就自动多了"读写 GitHub"和"访问本地文件"的能力。不需要写任何适配代码。

### MCP vs Function Calling 的关系

这两者不是替代关系，更像是**不同层级**的东西：

- Function Calling 是**模型层面的能力**——"我能理解你给我的工具定义，并生成调用参数"
- MCP 是**协议层面的标准**——"不管什么模型，都用这套格式来描述和调用工具"

实际上 MCP Server 暴露的 Tools，最终还是通过 Function Calling 的机制被 LLM 使用的。MCP 没有替代 Function Calling，而是在它之上加了一层标准化封装。

## Skills（技能）

### 是什么？

Skills 跟前面两个不太一样——它不是调 API，而是给 AI 加载一份**"操作手册"**。

具体来说，一个 Skill 就是一个 Markdown 文件（加上可选的脚本/资源），里面写着："在什么场景下、按什么步骤、用什么风格来完成任务"。

比如 Claude Code 里的 Skills，本质就是 `.claude/commands/` 目录下的 `.md` 文件：

```markdown
---
name: code-review
description: 对提交的代码进行审查
---

# 代码审查流程

1. 先阅读变更的文件，理解改动目的
2. 检查以下方面：
   - 是否有明显的 bug
   - 命名是否清晰
   - 是否有安全隐患（SQL 注入、XSS 等）
   - 是否有不必要的复杂度
3. 给出具体的修改建议，附带代码示例
4. 用中文输出，语气友好但专业
```

用户输入 `/code-review` 的时候，这段 prompt 就被加载到 AI 的上下文窗口里，AI 按照这个"手册"来执行任务。

### 跟 Function Calling / MCP 的本质区别

**Function Calling 和 MCP 解决的是"能不能做"的问题**——让 AI 有能力去查数据库、调 API、读文件。

**Skills 解决的是"怎么做"的问题**——教 AI 用什么方式、什么流程、什么风格来完成任务。

```
Function Calling / MCP:  "AI，你可以调用 get_weather 这个函数"
                          → AI 知道了自己能查天气

Skills:                   "AI，当用户问天气时，先查当天的，再查未来3天的，
                           用表格格式展示，如果有恶劣天气要特别标红提醒"
                          → AI 知道了该怎么查天气、怎么呈现
```

### Skills 的特点

- **不执行代码**，只注入上下文和指令，AI 自己判断怎么用
- **纯本地**，不需要网络调用，不依赖外部服务
- **轻量**，就是几个 Markdown 文件，不需要搭建任何基础设施
- **非确定性**——因为是 AI 自己解读执行的，同样的 Skill 可能每次执行结果略有不同

> 有个很好的概括：MCP tool 面对的挑战是"选哪个工具、什么时候用"，而 Skill 面对的挑战是"选哪个技能、什么时候用、**怎么用**"。多了一个"怎么用"的维度，因为这取决于 AI 自己对指令的理解。

## 三者对比

| 维度 | Function Calling | MCP | Skills |
|------|-----------------|-----|--------|
| **本质** | 模型能力（结构化输出） | 标准化协议（通信规范） | 行为指令（知识注入） |
| **解决什么** | 让 LLM 能调用外部函数 | 统一工具接入标准 | 教 AI "怎么做"特定任务 |
| **谁定义的** | 各 LLM 厂商 | Anthropic 发起的开放标准 | 开发者/用户自己写 |
| **执行方式** | 应用代码执行函数 | MCP Server 处理请求 | AI 自己解读执行 |
| **确定性** | 高（结构化输入输出） | 高（API 调用有明确结果） | 低（AI 解读可能有偏差） |
| **可移植性** | 厂商锁定 | 跨模型通用 | 主要在 Claude 生态 |
| **部署成本** | 低，写几个函数 | 中，需要运行 MCP Server | 极低，写几个 .md 文件 |
| **网络依赖** | 看函数实现 | 有（Client-Server 通信） | 无（本地注入） |

用一张图来看它们的层级关系：

```
Skills 层（行为编排、流程定义、风格约束）
  "当用户要求 X 时，按照 Y 流程来做"
        ↓
MCP 协议层（标准化工具接入、跨模型通用）
  Host ←→ Client ←→ Server
        ↓
Function Calling 层（底层模型能力）
  LLM 理解工具定义 → 输出结构化 JSON
```

## 实际开发中怎么选？

**场景一：简单的单模型应用**

比如你就用 GPT-4 做个客服机器人，需要查订单、查物流。直接用 Function Calling 就行了，最简单。

**场景二：多模型 / 多工具的平台**

你的产品需要支持 Claude 和 GPT，同时对接十几个外部服务。这时候上 MCP——工具端写一次 MCP Server，不管换什么模型都能用。

**场景三：团队有固定的工作流**

比如代码审查、文档生成、发布 SOP 这些重复性任务，用 Skills 把流程固化下来。团队成员不需要每次都重新描述需求。

**场景四：复杂 Agent 系统（通常三者都用）**

一个成熟的 AI Agent 往往是三层都用到的：
1. 底层通过 Function Calling 让模型具备调工具的能力
2. 中间层用 MCP 标准化工具接入，方便扩展和维护
3. 上层用 Skills 编排复杂工作流，定义"什么场景用什么工具、怎么用"

> 我现在做的博客发布工具就是这么干的——MCP Server 提供"同步文章到平台"的能力（工具层），Skill（`/write-blog`）定义了"搜索资料→写笔记→保存→发布"的完整流程（行为层）。两者配合，一个命令搞定整个发布流程。

## 容易搞混的点

**1. "MCP 是不是就是升级版的 Function Calling？"**

不是。Function Calling 是模型的能力，MCP 是通信协议。MCP 底层还是依赖 Function Calling 来让模型识别和调用工具，但它多做了一层标准化封装。就像 HTTP 和浏览器的关系——浏览器（模型）能发请求（Function Calling），HTTP（MCP）规定了请求的格式标准。

**2. "Skills 不就是 System Prompt 吗？"**

差不多但更结构化。Skill 有元数据（名字、描述、触发条件），可以按需加载（不是每次都塞进上下文），可以带附件（脚本、模板）。你可以理解为"可管理的、模块化的 System Prompt"。

**3. "我该学哪个？"**

都学，但优先级不同。Function Calling 是基础，必须理解。MCP 是趋势，2025 年 OpenAI、Google 都已经接入了，正在成为行业标准。Skills 是锦上添花，在用 Claude Code 这类工具时特别有用。

## 小结

三句话概括：Function Calling 让 AI "能做事"，MCP 让 AI "标准化地做事"，Skills 让 AI "按你定义的方式做事"。理解了这个层级关系，后面不管是接 API、搭 MCP Server 还是写 Skill，都不会搞混了。
