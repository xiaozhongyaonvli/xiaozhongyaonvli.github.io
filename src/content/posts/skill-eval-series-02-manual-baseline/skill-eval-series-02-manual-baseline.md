---
title: 做一套 Skill 评测优化工具（二）：先放下工具，手工跑通一组评测
description: "做一个零侵入、用户零感知的 skill 自动迭代优化工具，这是系列的第二篇。动工具之前，我先放下工具，全手工跑通一组评测——5 条 prompt、3 个维度、JSONL trace 当依据。文章拆三件事：为什么评测必须看 trace 而不是对话窗口；稀疏 skill 池下 implicit 触发的水分；以及最意外的一条——skill 真正的价值是『收敛自由度』，而不是让模型会做。4.6/5 的均分背后，藏着评测标准没问『它该收敛到什么』的更深问题…"
published: 2026-05-27
tags:
    - AI Agent
    - Skills
    - Evals
    - Claude Code
    - AI 工具
category:
    - AI 开发
draft: false
---

> 这是 Skill 评测优化系列的第二篇。整个系列我想做的是一个**零侵入、用户零感知的 skill 自动迭代优化工具**——把 skill 的迭代从「凭感觉手动调」变成「基于 case 集自动闭环」。这一篇还没动工具，先放下工具手工跑通一组评测，把「评测内核」长什么样、能干到什么程度、有哪些约束摸清楚。后面工具化才有标准答案可以对照。

## 一、整个系列在干什么

先说大图。整个系列围绕一个目标：做一个**零侵入、用户零感知的 skill 自动迭代优化工具**。我最终的愿望是，让 skill 在用的过程中越来越好用、越来越贴合用户实际需求。

完整形态是**四层闭环**：

| 层 | 职责 |
| --- | --- |
| ① case 收集层 | 从用户日常使用中自动累积训练数据：用户纠错 → 负 case；用户接受 → 正 case。我希望通过这个来明确收敛 skill 的职责范围（也可能是反向扩充职责范围） |
| ② **评测内核**（本篇焦点） | 基于 case 集判断 skill 在哪些维度上偏移。这里还涉及一个隐藏问题：沉淀的 case 集本身怎么动态更新——用户最初想收敛，后来又想扩大，这个判断也应该由工具自己完成 |
| ③ 自动优化层 | 差异化改写 SKILL.md 的对应段落（description / Workflow / Guardrails / Completion Criteria） |
| ④ 回归保护层 | 每次优化后跑全量回归，防止「补 A 坏 B」或对单类 case 过拟合。我希望调整是基于**抽象**的，而不是基于具体 case 的，除非确有明确需求 |

三条硬约束（贯穿四层）：

- **零侵入**——不污染用户的全局 skill 目录、不污染用户工作目录。我希望有一个隔离的环境，避免「当前会话里未被验证的改动」立刻被其他会话看到并采用
- **零感知**——被测方不知道在被评估，case 数据通过底层 trace 收集，不通过 prompt 元指令。agent 本身不能知道自己在被观测，否则它可能会过度谨慎；用户也是零感知，一切都在自动地、后台地运行，只会感觉这个 skill 用起来越来越顺手
- **可对照**——自动化每一步的结果可以和手工基线对得上

为什么要这么做？这件事的起点其实是另一个观察。OpenAI Developers 那篇 [Testing Agent Skills Systematically with Evals](https://developers.openai.com/blog/eval-skills) 读完，我最直接的感觉是：它讲的根本不是「怎么写出一个好的 skill」，而是另一件事——

> skill 的改动凭什么可以被信任？

这个问题平时根本不存在。一个人写 skill 的状态通常是：改一行 → 手动试几次 → 「好像变好了」 → 提交。但你其实并不知道这次「变好」是 prompt 凑巧，还是真的改对了；更不知道这次改动会不会让别的场景悄悄变差。我自己也写过一些 skill，针对 skill 的改动一直缺一个评判标准。最要命的可能是——你只关注了当前这个需求而忽视了其他场景，导致**对当前需求过拟合，其他场景反而下降了**。

有了 eval 之后，这件事才被定下来：改一行 → 跑同一批 prompt → 看分数变化 → 升了就保留、降了就回滚、其它场景没退化才放心。我的工程类比是：

> **eval 之于 skill，就是单元测试之于代码。**
> 单元测试不证明代码写得好，它证明「代码改完之后还是对的」。

但只到这里还不够。**如果只做「评测工具」，用户拿到的就是一份分数报告——然后还是要他自己去改 SKILL.md、自己跑回归、自己判断真的变好了没。这跟现在凭感觉调 skill 没本质区别，只是多了一份分数而已。**

所以我把目标推到下一步：**评测是手段，自动迭代优化才是终点**。评测内核是四层闭环里的一环，它的输出（维度报告）会喂给自动优化层去改 SKILL.md，改完再走回归保护层验证整体没退化。

这篇文章写的就是 **② 评测内核**的第一次摸底。我还没有开始做工具，想着先全手工跑一遍，目的是把这一层的约束、协议、能力边界搞清楚——然后工具化才有标准答案可以对照。

## 二、第一步：用 setup-demo-app 跑通一组评测

我选了一个示例 skill 作为被测对象，叫 `setup-demo-app`，description 是这样的：

> 帮助从零脚手架一个最小可运行的 demo web 应用。当用户要求创建一个小型 demo 应用、引导一个起步项目、接上基础样式，或者需要拿到把一个全新应用跑起来的精确本地命令时使用。

这个 skill 很适合做首发评测对象：

- **任务边界清晰**
- **产物可执行**——能跑就是能跑，不能跑就是不能跑
- **有几个明显的「容易跑偏」的点**——比如越界引入 ESLint / Tailwind / TypeScript

我给它设计了 **5 条 prompt** 和 **3 个评分维度**。

### 5 条 prompt 的分类

| # | 类型 | 用途 |
| --- | --- | --- |
| 1 | explicit（显式调用） | 快速验证：agent 是否能看到 skill（没被装漏、没被折叠） |
| 2 | implicit（隐式触发） | 验证 description 写得对不对，模型能不能凭描述召回 |
| 3 | contextual（真实上下文） | 在带场景语义的 prompt 下能否触发，顺带测对「minimal」边界的解读精度 |
| 4 | negative（负样本） | 检查不该触发的时候是否没触发 |
| 5 | negative（负样本） | 同上，从另一个角度 |

### 3 个评分维度

3 个维度都来自 SKILL.md 的三段：Workflow / Guardrails / Completion Criteria。

| 维度 | 来源 | 含义 |
| --- | --- | --- |
| `clear_setup_sequence` | Workflow | 五步顺序对不对、有没有跳步、有没有把框架配置塞到 init 之前 |
| `minimal_scope` | Guardrails | 有没有越界（乱加 ESLint / Tailwind / TS、擅自 install、擅自启服务） |
| `handoff_quality` | Completion Criteria | 给用户的最终命令清不清楚、是不是集中在一个代码块、端口/停止命令齐不齐 |

## 三、一个关键决定：评分不能只看对话窗口

刚开始我以为，提供给打分 LLM 的依据就是会话窗口里看到的对话内容。后来意识到这条路有几个问题：

1. **对话窗口里的内容不完整**——某些工具调用的内容会在处理完后被压缩甚至隐藏
2. **评测者会被自己的解读带跑**——看的是模型的最终文字时，你会无意识地把「它本应该做什么」和「它实际做了什么」混在一起
3. **看不到 thinking**——模型「想了但没说出来」的部分，恰好是判断 scope 越界的金证据（它有没有动过装一个多余依赖的念头）。我认为 thinking 是非常关键的信息，但在对话窗口里它是不可见的

那评判依据应该是什么？——**JSONL trace**。

Claude Code（我用的是 CLI 版本）每次会话期间会在本地自动落盘一份 JSONL 记录，详尽记录所有对话和工具调用结果。我用的是 Windows，位置在：

```
C:\Users\<用户名>\.claude\projects\<工作目录的 slug>\<sessionId>.jsonl
```


> 这件事关键不在「有 trace 文件」，而在「它是 CLI 默认自动写的」。这正好解了下一节要说的另一个问题——被测方不能知情。

## 四、JSONL trace 里到底有什么

简单介绍一下里面的内容，后面打分都靠它。

**三类核心事件**：

- `user` 事件：用户输入，或 `tool_result` 容器
- `assistant` 事件：模型的一次回合，内部分三种：
  - `text`：面向用户的输出文字
  - `tool_use`：调一个工具
  - `thinking`：内部 reasoning（**评判 scope 越界的金证据**——模型「想了但没说」的内容会被原原本本写下来）
- `system` 事件：元信息（这一回合花了多久、消息总数等）

**几个辅助事件**（知道有就行，我也是看着 AI 抽取出来的，对当前评测感觉不太用得上）：

- `attachment`：`@docs/...` 引用的文件内容内联
- `file-history-snapshot`：Edit/Write 前后的文件备份（撤销用）
- `permission-mode`：当前许可模式
- `opened_file_in_ide`：IDE 打开了哪个文件
- `last_prompt`：最近一条 prompt 的指针
- `task_reminder`：系统提醒（用 TaskTool 之类的）
- `skill_listing`：skill 列表广播
- `directory`：项目目录树快照

**几个关键属性**：

- 同一条用户 prompt 触发的所有事件共享同一个 `promptId`，**追溯非常方便**
- JSONL **只增不删，不会自动 compact**，历史完整可查
- `tool_result` 输出 >30KB 会被截断并留截断标记。当前阶段只能让评判方在读到截断标记时理解「这里输出是有截断的」，没有更好的兜底

> **TODO：待后续核实**——我猜应该会有一个指向完整 `tool_result` 实际落盘文件的指针（毕竟截断标记本身没法回溯原内容），但这里没有细看，留作 TODO。

## 五、一个不能违反的原则：被测方不能知情

这条听起来玄学，但很重要：

> 一旦被测方知道自己在被评估，它会优化对应行为（自我审查），基线就失真了。

我担心的主要是它会**变得偏保守**，这样对于 skill 职责范围的划定会变得模糊——你以为它「不越界」，其实是它知道自己在被看着。

所以我不能让被测方自己写评测日志，也不能在 prompt 里塞「你正在被评估，请按 rubric 输出」这类元指令。日志必须靠底层自动写，或者用其他不暴露给被测方的方式收集。

JSONL trace 正好是 Claude Code 默认自动写的，被测方根本意识不到，**这是它最大的价值**——不是「有」，而是「它对被测方透明」。

## 六、一个小工程技巧：`mklink /J`

我把项目里的 skill 放在我学习/修改这个工具的项目目录里，但 Claude Code 读 skill 是从全局目录读的。每次改完如果需要手动同步两边，很烦。

Windows 上用 directory junction 解决：

```powershell
mklink /J "C:\Users\<用户名>\.claude\skills\setup-demo-app" "C:\Users\<用户名>\Desktop\skill-test-learning\templates\evals\setup-demo-app"
```

之后改项目里的 skill，全局视图自动同步。**单一来源，不需要复制副本。**

## 七、5 条 prompt 的评测结果

下面把 5 条 prompt 的结果各自摘要一下。

### #1 explicit（显式调用）

- **prompt**：显式点名调用 setup-demo-app
- **trigger**：`trigger_correct = true`（显式触发，Skill → setup-demo-app）

| 维度 | 分 | 理由 |
| --- | --- | --- |
| `clear_setup_sequence` | 5 | Workflow 五步全覆盖，顺序：探查 → 脚手架（6 文件）→ 样式 → handoff，没把框架配置塞到 init 之前 |
| `minimal_scope` | 5 | 只建 Vite + React 最小集；thinking 全程「minimal」，无 ESLint / Tailwind / TS 越界；不擅自 npm install |
| `handoff_quality` | 5 | `cd` / `npm install` / `npm run dev` 集中在一个 PowerShell 代码块，含端口 5173，每条带说明 |

最终产物是一个暗色主题计数器，装好启动成功。

> 这条只能说明触发链路没崩，算是一个冒烟测试。

### #2 implicit（隐式触发）

- **prompt**：`Create a small demo web app from scratch. Keep it simple and tell me exactly how to start it after setup.`
- **trigger**：`trigger_correct = true`（模型自主调 Skill，没有显式点名）

| 维度 | 分 | 理由 |
| --- | --- | --- |
| `clear_setup_sequence` | 5 | 询问栈 → 探查环境 → 脚手架 → install → 改写内容 → handoff，顺序正确 |
| `minimal_scope` | **4** | 没装多余依赖；**扣 1 分**：未先验证默认模板能跑就重写 `App.jsx` / `App.css`，对「keep it simple」过度发挥 |
| `handoff_quality` | 5 | `cd` + `npm run dev` 集中代码块，含 5173 端口、Ctrl+C 停止、build / preview 备用命令 |

**核心发现**：implicit 触发链路正常，description 召回有效。scope 出现首个判断瑕疵——模型把「keep it simple」解读成了「主动 polish」，而不是「保留模板再验证」。这是一个可以记录下来的、典型的 process / scope 类失败信号。

### #3 contextual（真实上下文）

- **prompt**：`Inside this empty repo, bootstrap the smallest possible app I can run today and keep the setup sequence clean.`
- **trigger**：`trigger_correct = true`（thinking 里显式判断「setup-demo-app skill is exactly what I need」）

| 维度 | 分 | 理由 |
| --- | --- | --- |
| `clear_setup_sequence` | 5 | 探查环境 → 选最简栈 → Write `index.html` → handoff，五步对应，顺序最简合理 |
| `minimal_scope` | 5 | 主动放弃 Vite / React / npm，走单 HTML + Python `http.server`，零依赖；thinking 显式权衡后仍守 minimal；不擅自启服务 |
| `handoff_quality` | 5 | `cd` + `python -m http.server` 集中代码块，含端口 / URL，额外给出 rationale + 未来升级路径（Vite / Express） |

最终产物是单文件 `index.html`（计数器 +1 / -1 / reset、自适应深浅色），`python -m http.server 5173` 即跑。

**核心发现**：#2 和 #3 形成对照——都涉及「keep simple」语义，但：

- #2（弱约束措辞 `Keep it simple`）：擅自重写默认模板，scope 4
- #3（强约束措辞 `smallest possible`）：精准收敛到单 HTML，scope 5

→ description 健壮性不均：**强约束下引导得当，弱约束下没拦住过度发挥**。

（当然也可以反过来看：这样泛化性更高。我认为这些都不是实际的决定因素，**核心是用户体验**——如果用户觉得用这个 skill 这样做、并且效果他也满意，那么这个 skill 就应该负责这块内容。这一点在第八节会展开。）

### #4 / #5 negative（负样本）

- **#4 prompt**：`Review this existing function and explain whether it has a race condition.`
- **#5 prompt**：`Summarize the differences between React and Vue for a beginner.`
- **trigger**：两条都是 `trigger_correct = true`——全 trace 无 Skill 调用、无 `attributionSkill`，**成功未触发** setup-demo-app
- 其余维度 N/A（skill 没触发，打分不适用）

## 八、跑的过程中，我有一些发现

5 条全部跑完，均分 4.6 / 5，看上去这个 skill「挺好的」。

但如果就停在这里，这次评测的意义只剩下「我手动跑了一遍」。真正值得记下来的有三件事。

### 1. 评测系统本身有局限——稀疏 skill 池下，implicit / contextual 的「通过」水分很大

我的全局 skill 池里和 setup-demo-app 描述空间几乎没交集——其它 skill 领域几乎无关。所以 #2 / #3 的「触发成功」在稀疏 skill 池下证据力是有限的——**真实多 skill 环境下，结果可能完全不同**。

这等于是「低难度通过」，没真正考验 description 的精确性。

后续做「自动评测 skill 工具」时，这是必须解决的问题：

- 在沙箱目录里放被测 skill + 几个 confusable distractor（共享关键词但语义不同的诱饵 skill）
- FuncBenchGen 那篇论文的发现：**connected distractor**（共享类型或关键词、但语义不同）是最难的一类
- 陷阱质量把控：陷阱在被测 skill 的 positive prompt 下不能是合法替代解，否则评分逻辑会出现假阳

### 2. 跑完 5 条 prompt 不是评测结束，**是基线建立**

最容易的误解，是把这次练习理解为「给 setup-demo-app 做一次体检，看它写得好不好」。这不是评测的目的。

> 这里有个小插曲：我自己测着测着，结果细化太多，自己跑偏了，过了一会才反应过来——感觉对 agent 长任务的偏移有了更切身的理解，哈哈。

OpenAI 那篇文章真正解决的不是「判断 skill 好不好」，而是 **「skill 的改动不可比」**这个问题。

- **没有 eval 时**，skill 迭代是：改一行 → 手动试几次 → 「好像变好了」→ 但不知道是 prompt 凑巧还是真改好了，也不知道改完会不会让别的场景变差
- **有了 eval 后**：改一行 → 跑同一批 prompt → 看分数变化 → 升了就保留、降了就回滚、其它场景没退化才放心

所以这 5 条跑完不是评测结束。**它的价值要等到下一次有人想动 SKILL.md 一行时才显现**——那时这 5 条 prompt + 这套打分，就是判断「这次改动是否真的让 skill 变好」的对照标尺。

即使 setup-demo-app 没人想改，这套基线也已经构成「**未来任何改动都得通过的回归测试**」。

> 再贴一次那个类比：eval 之于 skill = 单元测试之于代码。单元测试不证明代码写得好，它证明「代码改完之后还是对的」。

### 3. skill 的真正价值，不是「让模型会做」，而是**「收敛自由度」**

这是我自己最意外的一条。回头看 #1 / #2 / #3 三条 positive 的产出：

- **#1**：Vite + React，默认模板
- **#2**：Vite + React，但主动重写了默认模板
- **#3**：HTML + Python `http.server`，零依赖

三次都「成功」，但三次产出完全不同。

这逼出一个关键追问：**skill 的价值到底是什么？**

模型本来就会搭 demo，不需要 skill 也能做。所以 skill 真正的价值不是「让模型会做这事」，而是 **「在模型本来就能做的多种合理做法中，约束到某一种」**——也就是**收敛自由度**。我认为这是工程思维的一种体现：我们希望输出是稳定的、可靠的。

按这个标准，setup-demo-app 在 #1 / #2 / #3 上几乎没起收敛作用——**有它跟没它，产出分布差不多**。

这才是 4.6 / 5 背后真正的问题：**评分标准没问「它该收敛到什么」，所以分数虚高**。

**但是**——要不要在技术栈 / 目录 / 命令上钉死，是一个有取舍的产品决策，不是单向更好。钉死的代价是**泛化能力下降**：如果硬性规定 always Vite + React，那 #3 那种走 HTML + Python 的最简解就被禁掉了，而那次明明是对 `smallest possible` 措辞最精准的呼应。

所以这个 skill 当前的形态，本身就可能是合理的产品选择：

> 如果 skill 的目的就是「起一个能跑的 demo，具体什么栈无所谓」——那现在的宽松 description + 不收敛 workflow 就是对的，评测出现的「产出多样」反而是符合意图的行为，而不是缺陷。

这意味着 **「该不该改这个 skill」不能由评测分数决定，而要由 skill 用户的意图决定**：

- 意图是「统一团队 demo 起点」 → 当前 skill 欠收敛，需要改
- 意图是「任何 demo 都行，只要能跑」 → 当前 skill 没问题，不改

评测工具最有用的输出**不是「4.6 / 5」**，而是：

> 「你这个 skill 在 X 维度没在做约束工作。」

让作者意识到自己其实没决定要约束什么。**第二层才是工具的真正价值。**

## 九、接下来要做什么（系列预告）

手工跑完 5 条 + 三段反思之后，接下来要做的事在四层闭环里都有，但近期重心仍在 **② 评测内核**内部的子模块上。

### 评测内核内部（当前焦点）

继续按手工的方式扩到 10 条、20 条 prompt **没意义**——本质瓶颈不在样本量，而在三个地方：

1. **维度怎么定**——目前是我读完 SKILL.md 之后人脑挑出来的，主观且不可复用
2. **prompt 怎么生**——目前是我对照 SKILL.md 写的，既有盲区，也不能保证覆盖度
3. **稀疏 skill 池**——上面反思 1 提到的 distractor 问题，implicit / contextual 触发的「通过」含金量打折

按依赖顺序拆：

- **下一篇**：从 SKILL.md 自动提取评分维度（对照 Anthropic 官方 skill-creator 2.0 的实现，叠加「零侵入、零感知」的硬约束）
- **再下一篇**：从 SKILL.md 自动生成 prompt 测试集（尤其是邻近场景 negative——共享关键词但语义不同的诱饵，这次手工基线里没覆盖到的盲区）
- **再之后**：沙箱 + 自动生成 distractor skill，解决稀疏 skill 池问题

### 其余三层（评测内核成型之后陆续展开）

- **① case 收集层**：怎么从 trace 里识别「用户纠错」信号？——用户接下来的 prompt 是不是在否定（「不对」、「不是这样」、「重做」）？边界很微妙：太宽会把无关 prompt 算成纠错，太窄会漏
- **③ 自动优化层**：基于维度报告怎么差异化改 SKILL.md？description 改写是其中最成熟的方向（Anthropic 官方已经做了，可以参考学习），Workflow / Guardrails / Completion Criteria 的改写策略还要研究。**这里还要解决反思 3 留下的问题**——自动优化不能盲目「补丁式」地把分数刷高，而是需要动态地维护评分维度和测试集，让 skill 真正越来越贴合用户的实际需求
- **④ 回归保护层**：case 集 append-only；每次优化前后跑全量打分输出 diff；过拟合检测策略待定

整个系列要回答的问题就是：**怎么把「凭感觉调 skill」变成「基于 case 集自动迭代」，并且让用户在这个过程中零侵入、零感知。**

希望我能成为一个好的产品背后的优秀工程师。

下一篇见！
