---
title: AI 时代，程序员的价值到底在哪？
date: 2026-03-24 16:15:00
tags:
  - AI
  - AI Agent
  - Claude Code
categories:
  - AI 开发
---

# AI 时代，程序员的价值到底在哪？

> 最近一直在想这个问题。AI 写代码越来越强了，GitHub Copilot、Claude Code、Cursor… 一个比一个猛。那我们这些写代码的人，以后靠什么吃饭？说说我的想法。

## 先看一个事实：AI 确实在抢活儿

不是危言耸听，数据摆在这：

- Stanford 的一项研究发现，到 2025 年中，22-25 岁软件开发者的就业率比 2022 年峰值**下降了近 20%**
- Salesforce 的 Marc Benioff 直接宣布 2025 年不再招新的软件工程师，理由是 AI 带来的生产力提升已经够了
- Anthropic CEO Dario Amodei 也说过，入门级岗位"在自动化的瞄准范围内"

但也有不同声音——AWS 负责人 Matt Garman 说"用 AI 取代初级开发者是我听过最蠢的想法之一"，因为新人便宜、学得快、对公司长期发展很重要。

所以现实是：**AI 不会消灭程序员这个职业，但会淘汰不会用 AI 的程序员。**

## 我觉得程序员的核心价值在哪

### 1. 从"写代码的人"变成"指挥 AI 的人"

这个转变我感受特别深。

O'Reilly 上有篇文章把这个变化描述得很好：程序员的角色正在从 **Conductor（指挥家）** 进化成 **Orchestrator（编排者）**。

- **Conductor 阶段**：你跟一个 AI 配合，盯着它写代码，一步步调整 prompt，不对就改——大部分人现在在这个阶段
- **Orchestrator 阶段**：你同时管理**多个 AI agent**，给它们分配不同任务，让它们并行工作，你负责定方向、做质量把控、整合结果

我之前看到 Claude Code 的团队模式（Agent Teams），一个 session 当 team lead 来协调工作、分配任务，其他 agent 各干各的，还能互相通信。有人说他一晚上开了 12 个 Claude agent，一个重构组件、一个写测试、一个更新文档、一个做性能优化，早上起来收到一个 10000 多行改动的 PR。

这不就是新时代的 "10x 工程师" 吗？不是你写代码快 10 倍，而是你能**同时指挥 10 个 AI 帮你干活**。

### 2. 拆解问题的能力比写代码更重要

AI 现在写单个函数、改个 bug 已经很溜了。但面对一个复杂的业务需求，它自己搞不定。

比如："给我们的电商系统加一个秒杀功能"——这一句话，AI 没法直接干。你得拆：

- 库存怎么扣减？用 Redis 预扣还是数据库乐观锁？
- 流量怎么削峰？消息队列用 Kafka 还是 RocketMQ？
- 前端限流怎么做？按钮置灰还是验证码？
- 订单超时怎么处理？延迟队列还是定时扫描？

每一个子问题可以丢给 AI，但**拆解本身**需要你对业务和技术都足够理解。这才是程序员不可替代的地方。

> Anthropic 2026 年的 Agentic Coding 报告里也提到：工程师们在实践中发现，越是"概念上困难的"或"依赖设计决策的"任务，越倾向于自己做或者跟 AI 协作完成，而不是完全交给 AI。

### 3. 判断力和审美

AI 生成的代码能跑，但不一定好。它不知道：

- 这个架构在高并发下扛不扛得住
- 这个方案是不是过度设计了
- 这段代码有没有安全漏洞
- 这个技术选型跟团队的技术栈搭不搭

**"能跑"和"能上线"之间差了十万八千里。** 判断 AI 输出的好坏、做取舍、拍板决策——这些是 AI 自己做不了的。

就像 Steve Yegge 说的，我们现在不缺更聪明的模型，缺的是**更好的编排层**——而这个编排层就是人。

### 4. 业务理解和沟通

代码是手段，业务才是目的。

产品经理说"用户反馈下单太慢了"，你得判断是前端渲染慢、接口响应慢、还是数据库查询慢。这需要对整个链路有认知，不是 AI 读几段代码就能下结论的。

而且很多时候需求本身就是模糊的，你得跟产品、设计、测试反复对齐。这种人与人之间的沟通协调，AI 短期内替代不了。

## 那我们现在应该怎么做？

说点实际的：

**1. 把 AI 当队友，不是当工具**

不要只是用 Copilot 补全代码。试试让 AI 帮你做 code review、写测试、生成文档、分析性能瓶颈。用得越深，效率提升越大。

**2. 练习"分活儿"的能力**

刻意练习把一个大任务拆成多个独立的小任务，然后分给不同的 AI agent 并行执行。这就像带团队一样——你得想清楚哪些任务可以并行、哪些有依赖关系、怎么合并结果。

（Claude Code 现在已经支持 Agent Teams 了，可以用 `TeamCreate` 创建团队，多个 agent 并行工作，还能互相发消息协调。试试看，体验很不一样。）

**3. 深入理解原理，而不只是会用 API**

AI 越能写代码，"会调 API" 就越不值钱。但理解底层原理永远值钱——你得知道 Redis 为什么快、MySQL 索引怎么走、JVM GC 什么时候会 STW。这些知识决定了你能不能判断 AI 的输出是否靠谱。

**4. 给 AI 立规矩，让它符合团队风格**

这个我觉得特别被低估了。AI 写的代码能跑，但风格可能跟你团队格格不入——命名规范不对、错误处理方式不统一、甚至引入了团队不用的依赖。

怎么解决？**写约束文档**。

比如用 Claude Code 的话，可以在项目根目录写一个 `CLAUDE.md`，把团队的规范和偏好写进去：

```markdown
# CLAUDE.md

## 代码风格
- 使用 4 空格缩进，不用 tab
- 变量命名用 camelCase，常量用 UPPER_SNAKE_CASE
- 不要用 var，统一用 const/let

## 技术栈约束
- HTTP 客户端统一用 axios，不要用 fetch
- 状态管理用 Pinia，不要引入 Vuex
- 日期处理用 dayjs，不要用 moment

## 架构规范
- API 请求统一放在 src/api/ 目录下
- 组件拆分粒度：超过 200 行就考虑拆分
- 错误处理统一用全局拦截器，不要在每个请求里单独 catch
```

这个文件 AI 每次启动都会读，相当于给 AI 一个"团队新人入职手册"。

不止 Claude Code，其他工具也有类似机制——Cursor 有 `.cursorrules`，Copilot 也可以配 instruction 文件。本质上都是一样的：**把你脑子里的隐性知识显性化，让 AI 能遵守**。

> 这其实也是程序员价值的一种体现：你得知道什么是好的规范，才能把它教给 AI。AI 不会自己总结出"我们团队应该怎么写代码"，这得靠人来定义。

**5. 保持对业务的敏感度**

技术服务于业务。能把技术方案跟商业价值挂钩的人，比单纯写代码厉害的人更有竞争力。

## 小结

AI 时代程序员的价值，我觉得一句话就能概括：**从写代码的人，变成指挥 AI 写代码的人。**

具体来说就是四个关键词：**拆解、编排、约束、判断**。

- 拆解复杂问题为可执行的子任务
- 编排多个 AI agent 并行高效完成
- 用规范文档约束 AI 输出，让它符合团队风格
- 判断输出质量，做最终的技术决策

McKinsey 的研究说 80% 的编程工作在未来几年内仍然以人为核心。所以不用太焦虑，但确实需要主动适应。与其担心被 AI 替代，不如想想怎么让 AI 帮你变成那个 "10x 工程师"。

---

参考资料：
- [Conductors to Orchestrators: The Future of Agentic Coding - O'Reilly](https://www.oreilly.com/radar/conductors-to-orchestrators-the-future-of-agentic-coding/)
- [The 10x Engineer in 2026 - DEV Community](https://dev.to/hainanzhao/the-10x-engineer-in-2026-what-actually-matters-after-ai-changed-everything-1g9m)
- [Software developers are the vanguard of how AI is redefining work - World Economic Forum](https://www.weforum.org/stories/2026/01/software-developers-ai-work/)
- [Is There a Future for Software Engineers? - Brainhub](https://brainhub.eu/library/software-developer-age-of-ai)
- [Claude Code Agent Teams - Anthropic Docs](https://code.claude.com/docs/en/agent-teams)
- [Anthropic 2026 Agentic Coding Trends Report](https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf)
