---
title: 做一套 Skill 评测优化工具（三）：停下来精读官方 skill-creator 2.0
description: "做一个零侵入、用户零感知的 skill 自动迭代优化工具，这是系列的第三篇。正要开始工具化，我又停了一下——官方 skill-creator 2.0 更新了，它早就不是当年那个『帮你创建 skill』的小工具，里面已经长出 eval / grader / benchmark / description 优化一整套骨架。我顺着它的主循环读下来：with-skill 对 baseline 的净增益、语义断言、train/test 防过拟合、grader 反过来批评评测本身……不少地方正好撞上我前两篇绕的问题。但它的姿态还是『用户主动要求改』，而我想要的是后台零感知自己找偏差、自己改写 SKILL.md。它给了我骨架，但还不是终点…"
published: 2026-05-30
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

> 这是 Skill 评测优化系列的第三篇。整个系列我想做的，是一个**零侵入、用户零感知的 skill 自动迭代优化工具**。前两篇先读了 OpenAI 那篇 skill eval 的文章，又手工跑通了一轮 `setup-demo-app` 的基线。正要开始工具化，我发现 Anthropic 官方的 `skill-creator 2.0` 更新了，也涵盖了相关内容，值得先停一下看。它已经不是早期那个「帮你创建 skill」的小工具，里面已经长出了一整套 eval / grader / benchmark / description optimization 的骨架。

## 一、为什么突然停下来读它

早期的 `skill-creator` 功能比较单一：主要是教你怎么创建 skill，怎么写 description，怎么组织 references 和 scripts。这也是我为什么叫它 2.0 哈哈——我之前看的那版只有写 skill 的功能。

这次再看，它已经明显变成了另一个东西。它不只是「写 skill」，而是在回答：

> 一个 skill 写完之后，怎么知道它真的变好了？

这句话正好对上前两篇反复绕的那个问题。

我想做的，不是一个「给 skill 打分」的小工具，而是一个**零侵入、用户零感知的 skill 自动迭代优化工具**。也就是说，用户日常用着用着，后台自动积累 case，自动发现 skill 哪里与用户需求不符，自动改 `SKILL.md`，再自动跑回归确认没有补 A 坏 B。

而 `skill-creator 2.0` 已经做了一些很接近的事情：

- 同一批 case 同时跑 with-skill 和 baseline
- 用 assertions + grader 做定量评估
- 用 viewer 给用户做定性 review
- 把多次 run 聚合成 benchmark
- 专门优化 description 的触发边界
- 用 train/test split 避免对少数 case 过拟合
- 用 blind A/B comparison 比较两个版本输出

有不少值得学习和参考的点。一个是它引入了 without-skill 和 baseline 这层对照（我觉得加入这个维度，特别是在模型飞速迭代和进步的当下非常适用，因为 skill 某些维度可能不需要约束也能达到既定目标了）。通过语义断言来做评判，这是个不错的点。train/test 的区分，避免过拟合。

但它不是最终形态。官方这套的姿态还是：

> 用户去主动感知、去要求修改某个 skill，人根据 trace 和 output 写 feedback。

而我想做的工具真正想要的是：

> 用户日常使用自然产生 case，后台零感知评测，工具自己找偏差，自己差异化改写 SKILL.md。

## 二、第一眼看到的主循环：它把「感觉变好了」变成了流程

`skill-creator` 的主线其实可以压成一行：

```text
draft → run with-skill+baseline → grade → aggregate → review → improve → re-run
```

拆开就是：

```text
draft（先写一个 skill 草稿）
→ run with-skill+baseline（同一批 case 同时跑「启用 skill」和「基线/旧版」）
→ grade（逐个 case 判分，给出 evidence）
→ aggregate（把多个 case / 多次 run 聚合成 benchmark）
→ review（人看定量结果和定性输出）
→ improve（根据反馈改 SKILL.md 或 description）
→ re-run（用同一套 case 重新跑，确认有没有变好或退化）
```

以前改 skill 很容易变成这样：

```text
改一段 prompt
手动试一下
感觉这次顺了
于是认为它变好了
```

但这里它强迫你把「感觉」落到一套流程里：

```text
同一批 case
同一套检查项
with-skill / baseline 对照
grader 给证据
benchmark 看整体
改完再跑
```

我前两篇一直在说 eval 之于 skill 有点像单元测试之于代码。读到这里，这个类比更实了。它不是证明 skill 写得漂亮，它是证明：

> 这次改动之后，同一批场景没有悄悄坏掉。

## 三、with-skill vs baseline：这个对照比想象中更重要

这里我比较感兴趣的，是 baseline 概念的引入。

官方不是只跑 with-skill，而是一定要跑 baseline。新建 skill 时：

```text
with_skill vs without_skill（我认为这是非常非常重要的一点，这是简化 skill 的重要依据来源）
```

改进已有 skill 时：

```text
new_skill vs old_skill
```

这个设计很有价值。

因为一个 case 在 with-skill 下通过，并不说明 skill 有价值。可能不用 skill，模型自己也能做得一样好。

它会逼出一个问题：

> 如果某个维度上 with-skill 和 without-skill 表现一样，那 skill 里关于这个维度的内容是不是冗余的？

这个判断不一定总成立。因为有些 skill 的价值不是让模型「能做」，而是让它「稳定地按某个方式做」。但 baseline 至少给了一个追问的入口：这段 skill 到底提供了什么净增益？

更危险的是反过来：

```text
with_skill fail
without_skill pass
```

这就说明 skill 不是帮忙，而是在某类场景拖累模型。这必须被当成 regression，不能只看平均分。

官方还强调 with-skill 和 baseline 要在同一 turn 内并行 spawn。这个细节很工程。因为模型状态、服务负载、随机性都会影响结果。如果两版隔很久跑，你很难说差异到底来自 skill，还是来自外部波动。

这也直接映射到第 ④ 层回归保护：以后每次自动改写 skill，都要尽量在同一环境下跑改前/改后，不能随便拿两个时间点的结果硬比。（或者考虑排除环境的影响，比如 duration-ms 这个应该只需跟当前同时跑的 baseline 比，而不能直接复用 baseline 之前的数据。）

## 四、assertions 评估评分是另一个让我惊喜的点

官方倾向用 `assertions`。

好的 assertion 要客观可验证，名字还要清楚，让 viewer 里扫一眼就知道它检查什么。

比如：

```text
The output includes X.
The skill used script Y.
The final response includes the command to rerun the app.
```

这里的断言并不是程序性断言，而是语义断言（这会不会就是新时代的断言？语义断言？不写固定的检查代码，而是给出语义要求）。它检查的是一次 agent run 是否满足某个可验证条件：

- 有没有产出某个文件
- 有没有包含某个内容
- 有没有调用某个脚本
- 有没有走某个关键步骤

而我要做的「维度」更像：

- setup 顺序清不清楚
- scope 有没有收住
- handoff 质量好不好

这些不是简单的 pass/fail。比如 handoff 可以 5 分，也可以 3 分，也可以完全没有。它更像 rubric。

现在的判断是：

| 类型 | 适合干什么 |
| --- | --- |
| assertion | 检查有没有做、是否满足硬条件 |
| rubric 维度 | 判断做得多好、偏差在哪个方向 |

```text
触发没触发 / 文件有没有 / 工具有没有用 → 二元 PASS/FAIL
流程完整度 / scope 控制 / handoff 质量 → 1-5 维度评分
```

## 五、timing 和 viewer：两个看似边角，但很真实的设计

官方会在 subagent 完成时抓 timing：

- `total_tokens`
- `duration_ms`

它特别提醒：这是唯一一次能拿到的机会。错过了就没了。

这个其实主要是成本管控的关注。因为一个 skill 不是只要 pass rate 高就行。如果它让 token 翻倍、耗时翻倍，那就要问值不值。以后 benchmark 里一定不能只有质量分，还要有时间、token、tool calls、errors、variance。

另一个细节是 viewer。官方跑完后会生成一个网页，让用户逐个 case 看结果并留下 feedback。空 feedback 就认为用户觉得 OK。

之前 claude 推出说 ai 时代可能更适合给人看的是 html 而不是 md，这里也可以看他们确实这么干的哈哈。我个人看法，网页当然能更好地编排、更好地展现，但感觉需要注意 token 更多的消耗，token 是否能更便宜（路由让 haiku 模型这种来做？或许是个办法）。

## 六、四条改进原则：这部分真的挺精华

下面是官方关于 improving skill 的四条原则，基本无法压缩概括，已经很精简和高密度了。

第一条是从 feedback 泛化。用户熟悉这几个 case，所以和用户在少数 case 上迭代很快。但 skill 不能只对这几个 case 有效。不要因为一个失败 case 就堆一堆压迫式的 `MUST / ALWAYS / NEVER`。如果某个 issue 反复不过，可以换方法试试。成本低，换个角度也许就撞上了。重点是不要只在一个僵硬方向上越拧越紧。

第二条是保持 prompt 简洁。要读 transcript，不要只看 final output。如果 skill 让模型在某些动作上花很多时间但没产出，那段指令可能就该删。这个思路很像删 dead code：不是写得越多越好，每一段都要拉得动业务。

第三条是解释 why。今天的 LLM 很聪明，有一定 theory of mind。一个好的 harness 加上「为什么这样做」的解释，往往比死板命令更有效。规则不应该只是：

```text
NEVER do X.
```

而应该尽量写成：

```text
Avoid X because it expands scope and makes the handoff harder to verify.
```

第四条是找跨 case 的重复劳动。如果多个 case 里 agent 都临时写了类似 helper script，或者都走了同一个多步流程，那这种可复用能力应该沉淀进 `scripts/`。

这个点之前想得不够。原来更多想着「自动改 SKILL.md 文字」，但这里提醒了一件事：skill 优化不只是改文字，也可能是把反复出现的脚本、模板、验证器沉淀下来。

## 七、description 优化：这里已经有自动优化的雏形了

最接近「自动优化」的部分是 description optimization。

它先明确一个机制：

```text
skill 出现在 Claude 的 available_skills 列表里
Claude 先只看到 name + description
Claude 根据它们决定是否查询 skill
```

所以 description 不是普通摘要。它更像一个触发分类器的特征文本。

它要同时处理两类 case：

| 类型 | 含义 |
| --- | --- |
| should-trigger | 应该触发 |
| should-not-trigger | 不应该触发 |

这里最有价值的是 near-misses。should-not-trigger 里最好的负样本不是八竿子打不着的任务，而是「差一点就该触发，但其实不该触发」的相似任务。它们共享关键词和概念，但实际需要别的工具。

这一下就对上了 case 收集层：

> 用户纠错「不该调用这个 skill」，其实就是天然 near-miss。

skill 已经触发了，但用户实际不想它调用。这个负 case 比人工编一个有价值太多。

官方还提醒：case 不能太简单。Claude 只会在自己觉得需要 skill 时才查 skill。简单的一步 query，即使 description 完美匹配，模型也可能自己解决，不会触发 skill。

所以 eval query 必须真实、具体、有场景、有复杂度。坏例子是：

```text
Format this data.
```

好一点的是：

```text
我老板发了一个 Q4 sales final FINAL v2.xlsx，让我加一列 profit margin，
收入在 C 列，成本在 D 列，顺便做个图给周会用。
```

这也会让一些简单 skill 的价值变得可疑。如果一个 skill 处理的都是模型自己能轻松完成的事，with-skill 和 without-skill 差距又不大，那它可能没必要存在。除非它的价值不是补能力，而是约束产出格式和流程。

## 八、train/test split：这不只是测试，更像训练

官方把 case 集分成 train/test，而且按 should_trigger 分层。

优化时用 train 结果指导 description 改写，最后选 best description 看 test 分数。

这不是普通「跑测试」思路，更像把机器学习里的反过拟合逻辑迁移到 prompt / skill 优化里。

还有一个细节：它喂历史尝试时，只喂 description 版本 + 分数，并告诉 LLM 不要重复结构相似的尝试。

这是为了防止卡在 local minimum。也就是每一轮都只是小修小补：

```text
Use this skill for PDF forms...
Use this skill for filling PDF forms...
Use this skill for completing PDF forms...
```

看起来一直在改，但结构没变，失败模式还在。

这个设计对自动改写特别重要。以后不能每轮只是再加一句限制，也要允许换结构、换表达、甚至把文字指令变成脚本。

## 九、schemas：这些 JSON 其实是系统的骨头

`references/schemas.md` 表面上是在列字段，实际把整个系统怎么流动讲清楚了。

`evals.json` 是评测用例集，里面有 prompt、expected_output、files、expectations。

`history.json` 记录版本树，包括当前最佳版本和每个版本的得分。不过它更像单链版本记录，不是完整的多版本回归系统。

`grading.json` 是单次 run 的评分输出。除了 expectation 是否通过和 evidence，它还有两个我特别关注的字段：

```text
claims
eval_feedback
```

`claims` 是从被测结果往回抽信号。它关注模型实际承诺了什么、做了什么、有没有证据支撑。它和 assertion 不一样：

```text
assertion = 评测者预先规定要检查什么
claim     = 模型实际输出里自己声称了什么
```

`eval_feedback` 是 grader 反过来批评 eval 本身，比如弱 assertion、覆盖缺失、不可验证。

这两个字段很重要。因为它们不是只评估被测模型，而是在评估评测系统自己。换句话说，外层 harness 也要自进化。

`benchmark.json` 是多 run 聚合，里面会保留 skill name、model、runs、configuration、pass rate、timing、tokens、delta。这里 model 很重要，不同模型即使用同一个 skill，最后结果也可能不同。

`comparison.json` 是 blind A/B 对比输出。`analysis.json` 是后置分析，说明 winner 为什么赢。

不过这里有一个分歧。官方 `analysis.json` 很自然地会写 winner 的长处、loser 的短处、loser 的改进建议。但如果流程是 A/B 后直接选择 winner 作为 best-known version，那默认「优化 loser」不一定有意义。所以我觉得是否应该分为多个维度去对比 win/lose。

更合理的方向是：

```text
winner 进入 best
loser 作为失败实验复盘
把可迁移教训用于下一轮优化 best
```

尤其需要按维度看。一次改动通常是为了增强某一方面，但风险是导致其它维度退化。analysis 应该帮我们看清：哪个维度赢了，哪个维度退了。

## 十、数据流

这套系统的数据流大概是这样：

```text
                  evals.json  ──┐
                                │
                                ▼
                        (subagent 执行)
                                │
                                ▼
   metrics.json + timing.json (执行副产物)
                                │
                                ▼
                  ┌─────  grader  ─────┐
                  ▼                    ▼
            grading.json         (claims + eval_feedback)
                  │
                  │ (多 run 聚合)
                  ▼
            benchmark.json  ←─── analyzer (生成 notes)
                  │
                  │ (双版本 blind 对比)
                  ▼
            comparison.json ──→ analyzer ──→ analysis.json
                                                  │
                                                  │ (improvement_suggestions)
                                                  ▼
                                           回到 SKILL.md（人工修改）
                                              │
                                              ▼
                                         history.json (版本树)
```

这里的职责划分得很清楚：

- `grading.json` 负责单 run
- `benchmark.json` 负责多 run 聚合
- `comparison.json` 负责 blind A/B
- `analysis.json` 负责解释为什么赢
- `history.json` 负责记录版本演进

## 十一、grader：最该复用的原子

`grader` 是一个 subagent。它读取一次 run 的 transcript 和 outputs。

它的主任务是：

```text
核对预设 expectations
每条 expectation 给出 PASS / FAIL + 引文证据
```

但它还有一个隐藏任务：

```text
反过来批评 eval 本身
弱 assertion / 覆盖缺失 / 不可验证 都要标出来
```

这里最重要的是：

> 通过一个弱 assertion 比 fail 更糟，因为它制造虚假信心。

这个判断很对。fail 至少告诉你哪里没过；弱 assertion pass 会让你以为系统很稳，但其实只是评测没抓住问题。

grader 的流程是：

1. 读 transcript
2. 看 outputs
3. 逐条评估 expectations
4. 抽取并核对隐含 claims
5. 读 user_notes
6. 反过来审视 eval 本身
7. 写 grading 结果
8. 带上 metrics 和 timing

这里有几个原则可以直接拿走：

- 不能只信 transcript，要看实际 output
- 默认 FAIL，只有明确证据才 PASS
- 无法核对的 claim 要标成 unverifiable
- user_notes 可能暴露 expectations 都 pass 但实际有问题
- eval_feedback 只在明确 gap 时提，不要凑数

这几乎就是未来「维度判分 agent」的底层形态。区别是，输入要从 expectations 字符串列表，升级成：

```text
用户 prompt
SKILL.md
一次 run 的 trace
outputs
维度 rubric
用户后续反应（接受/纠错/未评价）
```

输出则不只是 pass/fail，还要有每个维度的 1-5 分、claims 核对、对 rubric 本身的反馈。

## 十二、run_eval.py：零感知触发评测的工程样板

`run_eval.py` 做的是 description 触发评测。

它的办法是：

1. 把 desc 写成 `.claude/commands/<random>.md`
2. 随机命名，避免和已有命令冲突
3. 启动新的 `claude -p` 进程跑真实 query
4. 读 `stream-json`
5. 看 `tool_use` 是否引用这个临时 command

它还用了 `--include-partial-messages`，这样工具调用刚开始就能拿到事件，可以早返回。

每个 query 跑三次。正样本触发率大于等于 0.5 才算通过；负样本则要低于 0.5。这是在降低单次随机性的影响。

最值得复用的是它的姿态：

> 被测 Claude 不知道自己在被测。

它只是正常收到一个 query，正常决定要不要用 skill。这正好是零感知原则。未来不一定照搬 command 文件法的每个细节，但这个思路必须保留。

## 十三、improve_description.py：失败要分类，不能混成一锅

`improve_description.py` 会根据 `run_eval.py` 的结果，让 LLM 改写一次 description。

它先把失败分成两类：

| 类型 | 含义 | 修复方向 |
| --- | --- | --- |
| failed triggers | 该触发但没触发 | 召回不足，description 信号不够 |
| false triggers | 不该触发却触发了 | 过度泛化，description 边界太宽 |

这两类问题修法是相反的。召回不足要加强相关意图的信号，误触发要收窄边界。如果混在一起喂给 LLM，很容易越改越糊。

它还明确禁止列举具体 query，要求抽象成更广的 user intent。

这个点对我非常重要。因为自动优化最容易犯的错就是：看到一个失败 case，就把这个 case 的字面特征写进 skill。这样短期可能过了，长期一定过拟合。

不过如果考虑将 case 持久化，需要加上时间的注脚，如果发生 case 冲突，应以最近的为准。

自动优化层也应该分类型：

```text
触发偏差 → 改 description
流程偏差 → 改 Workflow
scope 越界 → 改 Guardrails
handoff 差 → 改 Completion Criteria
```

而不是一股脑「优化 SKILL.md」。

## 十四、analyzer：模式发现不是建议，是把异常显形

`analyzer.md` 有两个角色。

第一个是 post-hoc analyzer。blind comparison 后，它读 winner/loser 的 skill 和 transcript，分析为什么 winner 赢。

它会看：

- 指令是否更清楚具体
- 示例覆盖是否更好
- script/tool 使用模式是否更稳定
- 边界 case 和错误处理是否更完整
- agent 是否真的 follow 了 skill

这里的 script/tool 使用模式，我理解成两层：

```text
script：skill 有没有沉淀自己的脚本，什么时候要求 agent 用，脚本承担什么稳定能力
tool：skill 有没有指导 agent 用哪些工具，按什么顺序用，失败时怎么 fallback
```

loser 的弱点可能就是缺了关键 tool，逼着 agent 做 workaround。比如本该有 `validate_output.py`，结果没有，agent 每次临时写一段不稳定脚本。

第二个角色是 benchmark result analyzer。它看多 run 数据，做模式发现。

这里的「模式」不是设计模式，而是：

> 在多次 eval 结果里，找出单个 case 看不出来的规律、异常和重复现象。

它对每条 expectation 跨所有 run 看：

| 模式 | 含义 |
| --- | --- |
| 两个 config 都 pass | 可能不区分 skill 价值 |
| 两个 config 都 fail | 可能 assertion 坏了，或超出能力 |
| with-skill 总 pass / without-skill 总 fail | skill 在这里清晰加分 |
| with-skill 总 fail / without-skill 总 pass | skill 在这里反而拖累 |

这里的职责分离很关键：benchmark analyzer 只报观察，不直接给改进建议。因为「发现模式」和「决定怎么改」是两件事。维度报告也应该这样，先把偏差显形，再交给优化层判断怎么改。

## 十五、aggregate_benchmark.py：delta 才是 baseline 对比的本质

`aggregate_benchmark.py` 把多个 `grading.json` 聚合成 `benchmark.json`。

最关键的是 delta：

```text
with_skill - baseline = 净收益
```

单看 with-skill 的 pass rate 没意义。要看它相比 without-skill 或 old-skill 多了什么、少了什么。

这就是 baseline 对比的本质。

也因为有 delta，我们才能看到：

- skill 在哪些维度真的加分
- 哪些维度没有区别
- 哪些维度反而退化
- 质量提升是否值得额外 token 和时间
