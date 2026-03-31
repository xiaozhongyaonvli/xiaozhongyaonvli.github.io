---
title: 从 Prompt 工程到 Context 工程：我如何学习 Agent-Skills-for-Context-Engineering
date: 2026-03-31 10:00:00
tags:
  - AI
  - Agent
  - Context Engineering
  - Prompt Engineering
categories:
  - AI 开发
---

# 从 Prompt 工程到 Context 工程：我如何学习 Agent-Skills-for-Context-Engineering

在真正做长任务 agent 之前，我一直把问题理解为“提示词怎么写更好”。学完 `Agent-Skills-for-Context-Engineering` 这一轮之后，我的看法变了：很多看起来像“模型能力不行”的问题，其实是上下文工程问题。

这篇文章是我的学习复盘，基于我对 `Agent-Skills-for-Context-Engineering` 项目 README、skills 和 examples 的系统阅读与实践整理。

## 为什么我把这个项目当第一站

这个仓库最有价值的地方，不是“又一个 agent 工具包”，而是它把问题定义得非常干净：

- 进入模型上下文的到底有哪些信息
- 这些信息在长会话里如何失真
- 什么时候该压缩，什么时候该隔离，什么时候该落盘
- 多 agent 和 memory 在上下文层面各自解决什么问题

它研究的是一个系统如何管理“可见状态”，而不是只研究一段 prompt 文案。

## 我拿到的第一个关键转变

以前我会问：

- 这段 prompt 怎么改更强？

现在我先问：

- 这个回合里，模型到底看到了什么？
- 哪些 token 是高信号，哪些是噪声？
- 是否已经进入上下文退化区？

换句话说，我开始把上下文当有限的注意力预算来治理，而不是当无限容器去堆文本。

## 6 个核心学习点（也是我本轮主线）

这轮我只抓 6 个关键词，避免学散：

- `context-fundamentals`（上下文基础）
- `context-degradation`（上下文退化）
- `context-compression`（上下文压缩）
- `context-optimization`（上下文优化）
- `multi-agent-patterns`（多智能体模式）
- `memory-systems`（记忆系统）

### 1. Fundamentals（基础）：上下文不是 prompt，是完整推理状态

最基础但最重要的一点是，context 包含：

- system instructions
- tool definitions
- retrieved documents
- message history
- tool outputs

这直接解释了为什么“我只改了 prompt 但效果还是不稳”：因为你没治理的是其它四类输入。

我还补上了 `context-components.md` 里两个很实用的设计点：

- system prompt 最好用 XML 标签分段，例如 `<BACKGROUND_INFORMATION>`、`<INSTRUCTIONS>`、`<TOOL_GUIDANCE>`、`<OUTPUT_DESCRIPTION>`，这样模型在长上下文里更容易定位规则
- 指令需要做 Altitude Calibration：太低会变脆、太高会变空，最稳的是 heuristic-driven 的中间层写法（有步骤、有边界，但保留执行弹性）

### 2. Degradation（退化）：上下文退化是可诊断的，不是玄学

我最有收获的是五类退化模式：

- lost-in-middle（中段遗忘）
- poisoning（上下文污染）
- distraction（分心干扰）
- confusion（语境混淆）
- clash（上下文冲突）

这套分类把“模型怎么突然变笨了”拆成可定位问题。定位清楚之后，修复动作才不会乱用。

### 3. Compression（压缩）：目标不是每次省 token，而是任务总成本更低

这个项目反复强调 `tokens-per-task`，我非常认同。  
如果摘要导致关键文件路径、错误码、决策链丢失，后面反复 re-fetch，整体成本会更高。

所以我现在不会只看“单次 request 省了多少 token”，而是看整条任务链路里有没有出现额外重读、重复检索和重复推理。

压缩里我最重视两件事：

- 结构化摘要优于自由摘要
- artifact trail（产物轨迹）需要显式保存，不能靠摘要“顺便记住”

另外 `skills/context-compression/scripts/compression_evaluator.py` 这套评测给了我一个很实用的视角：  
它不是简单把“压缩前全文 + 压缩后全文”一起扔给同一个模型打一个总分，而是：

1. 从压缩前历史抽取 probe（事实、文件、决策、下一步）
2. 只把压缩后上下文提供给模型回答这些 probe
3. 再用评分器按多维标准（accuracy、artifact trail、continuity 等）打分

这样更接近真实场景：压缩后 agent 是否还能继续把任务做下去。

### 4. Optimization（优化）：优化不等于“做个总结”

这个项目给了很实用的优先级：

1. 先做 KV-cache 稳定前缀
2. 再做 observation masking
3. 还不够再 compaction
4. 最后才 partition到多 agent

这比“上下文长了就摘要”要工程化得多。

另外我现在会把 progressive disclosure 明确拆成两种模式来实现：

1. 技能激活模式：只有任务描述和某个 skill 足够相近时，才加载该 skill 的完整上下文，而不是开场全量加载所有 skill
2. 引用加载模式：对大块 tool output 先做引用化，原文落存，窗口里只保留引用 ID 和关键摘要；需要时再按 reference id 回读原始内容

这两种模式叠加，能同时控制首屏上下文体积和长会话的历史膨胀。

### 5. Multi-Agent：核心收益是上下文隔离，不是角色扮演

我之前也容易把多 agent 理解成“专家团队”。  
这个项目让我重新校正：多 agent 首要价值是 context isolation（上下文隔离）。

当一个窗口承载不了全部信息时，拆分上下文比继续堆文本更可靠。

### 6. Memory：先从简单层开始，复杂度按检索质量升级

这个项目对 memory的态度很克制：  
不是一上来就 graph/ temporal（时间维度）/ fancy infra（复杂基础设施），而是先问检索需求和失败模式。

我现在更认可这样的路线：

- 先文件系统和结构化落盘
- 再向量检索
- 再图和时间有效性

这和实际工程节奏更匹配。

而且文件系统的好处不只是“能存很多”，还有可发现性：

- 文件名本身就是语义索引（比如 `customer_pricing_rates.json` 比 `file1.json` 可检索性高很多）
- 时间戳是天然的新鲜度信号，便于 agent 优先读取最近更新的上下文产物

## 两个 example 带来的“落地感”

我重点看了 `digital-brain-skill` 和 `x-to-book-system`，它们的价值在于把原则转成结构。

`digital-brain-skill` 让我看到：

- progressive disclosure（渐进式披露）不只是概念，也可以体现在目录组织
- append-only 数据格式天然更友好于 agent（智能体）回放和追踪
- 模块隔离本质是在减少上下文污染

`x-to-book-system` 让我看到：

- supervisor 架构的关键不是“有个总指挥”，而是控制上下文流量
- raw data 不经过 orchestrator 全量上下文，而是先落文件再分阶段读取
- 多 agent + 文件系统 + memory layer 可以形成可扩展的上下文治理闭环

## 我现在判断一个 agent 系统的方式

学完这轮之后，我会先看这几件事：

- 输入到上下文的五类信息是否被分别治理
- 当前问题属于哪种 degradation 模式
- 是否已经有 observation masking 和 compaction 策略
- sub-agent 是否真的在做上下文隔离，还是只在做角色装饰
- memory 层是否和实际检索需求匹配，而不是“为了高级而高级”

这个判断框架比“有没有 memory”“窗口多大”“用了多少 agent”更有解释力。

## 给自己留的工程化清单

后续我做项目时，会优先做这 8 条：

1. 明确上下文预算和 compaction 触发阈值
2. 固定稳定前缀，提升 cache hit
3. 工具输出默认可 masking，保留可回读引用
4. 用结构化摘要保存任务状态和 artifact trail
5. 任务切换时显式 reset 或隔离上下文
6. 多 agent 只在上下文收益明显时拆分
7. memory 层从简单到复杂渐进升级
8. 优化后用评测验证，不靠体感宣布“变好了”

## 这一轮学习的阶段结论

如果只让我推荐一个“建立上下文工程思维”的起点，我会继续推荐这个项目。  
它的强项不在某个花哨实现，而在于给了你一套可迁移的语言体系：

- attention budget
- lost-in-middle
- compaction
- masking
- partitioning
- memory layering

这套语言能直接迁移到你后续学的 `planning-with-files`、`Vibe-Skills`、`LLMLingua`、`vLLM`、`Graphiti` 上。

## 结语

如果你也在做长任务 agent，我建议先建立一套上下文工程判断框架，再谈模型、工具和工作流。  
这套框架一旦建立起来，你会更快识别系统瓶颈，也更容易把优化投入放到真正影响效果的地方。
