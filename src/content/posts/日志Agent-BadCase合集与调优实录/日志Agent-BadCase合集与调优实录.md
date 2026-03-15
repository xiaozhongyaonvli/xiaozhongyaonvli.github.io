---
title: 日志 Agent 上线后遇到的 Bad Case 合集：从 RAG 召回到幻觉，逐个拆解
description: "之前写过日志 Agent 的架构设计，这篇专门聊上线后遇到的各种 Bad Case。做 AI Agent 项目最大的感受就是——架构设计只占 30%，剩下 70% 的时间都在跟 Bad Case 搏斗。上线第一周准确率只有 65%，经..."
published: 2026-01-05
tags:
    - AI Agent
    - RAG
    - 调优
    - Bad Case
category:
    - 项目实战
draft: false
---
> 之前写过日志 Agent 的架构设计，这篇专门聊上线后遇到的各种 Bad Case。做 AI Agent 项目最大的感受就是——架构设计只占 30%，剩下 70% 的时间都在跟 Bad Case 搏斗。上线第一周准确率只有 65%，经过两个月的迭代才稳定到 90%+。把踩过的坑和调优思路记录一下，希望对做类似项目的人有参考价值。

## 先说方法论：遇到 Bad Case 别急着改 Prompt

上线第一天就收到研发反馈"结论不对"，我的第一反应是改 Prompt。改了，好了一个 case，但另外三个 case 又坏了——这就是**回归问题**。

后来摸索出一套流程：

```
Bad Case 进来
  ↓
第一步：归类——是哪种不好？
  - RAG 没召回 / 召回了但不相关
  - Pipeline 漏判 / 误判
  - ReAct 推理链断了
  - 模型幻觉（编造日志内容）
  - JSON 输出格式崩了
  - 结论太笼统 / 不可操作
  ↓
第二步：定位——在哪个节点出的问题？
    （看完整的 trace log，每个节点的输入输出都有记录）
    ↓
第三步：最小改动修复
    ↓
第四步：跑回归测试集（100+ 标注 case），确认没退步
    ↓
第五步：灰度 10% → 观察 → 全量
```

> 调优优先级我的经验是：**Prompt > 检索参数 > 流程编排 > 换模型**。Prompt 改动成本最低，效果最立竿见影。换模型是最后手段——贵且不一定管用。

## Bad Case 1：RAG 召回失败——明明有历史案例，就是搜不到

### 现象

输入："用户反馈语音合成没声音"

知识库里明明有这条历史案例：
> "2025-11-03，大量用户对话无响应，根因：TTS Pod OOM，解决方案：重启 Pod 并扩容"

但 RAG 节点没命中，confidence 只有 0.3，直接跳过走了 Pipeline，多花了 15 秒和一堆 Token 才定位到同样的问题。

### 定位过程

把 query 和知识库条目的 embedding 拿出来手动算了一下余弦相似度：

```python
query = "语音合成没声音"
doc   = "大量用户对话无响应，根因：TTS Pod OOM"

similarity = cosine_similarity(
    embedding_model.encode(query),
    embedding_model.encode(doc)
)
```

0.31，确实很低。问题出在**语义表达差异太大**——用户说"语音合成没声音"，知识库写的是"对话无响应"。虽然人能理解这是同一类问题，但 embedding 模型觉得这两句话不太相关。

### 解决方案

**方案一：给知识库条目加多角度描述（最有效）**

原来一条案例只存一种表述，改成存多个：

```
原始：
  "大量用户对话无响应，根因：TTS Pod OOM"

改进后：
  "大量用户对话无响应，根因：TTS Pod OOM"
  + 别名："语音合成无声音" "TTS 没有回复" "语音服务卡住"
  + 关键词标签：TTS, 语音, 无响应, OOM, 内存
```

检索时不只匹配主描述，别名和标签也参与向量检索。这样同一条案例在向量空间里有多个"入口点"，不同表述都能命中。

**方案二：调低相似度阈值**

从 0.7 降到 0.55。听起来简单，但要小心——调太低会引入大量噪音（不相关的结果混进来），所以同时加了 **metadata 过滤**：

```python
results = pgvector.similarity_search(
    query_embedding = encode("语音合成没声音"),
    top_k = 5,
    threshold = 0.55,
    filter = { "service": "TTS" }  # 只在 TTS 相关案例里搜
)
```

服务名是怎么来的？Query 预处理阶段，LLM 会从用户描述里提取涉及的服务名。"语音合成"→ 关联到 TTS 服务。有了这层过滤，即使阈值降低，噪音也不会太多。

**方案三：Query 重写的 few-shot 优化**

之前 Query 重写的 prompt 太泛了，模型重写出来的 query 经常"换汤不换药"。加了几个 few-shot 示例后好了很多：

```
示例1：
  原始查询：语音合成没声音
  重写为：TTS 服务无响应 语音输出失败

示例2：
  原始查询：机器人不说话了
  重写为：对话无响应 TTS 输出为空 语音合成失败

示例3：
  原始查询：用户说完话没反应
  重写为：ASR 识别后无 NLU 响应 对话流程中断
```

> 这个 case 教会我一件事：**RAG 的召回率，一半取决于检索算法，一半取决于知识库的内容质量**。你的知识库条目只用一种表述方式写，那用户换个说法就搜不到了。

## Bad Case 2：RAG 召回了，但不相关（噪音）

### 现象

输入："用户 A 的对话记录加载很慢"

RAG 检索结果 top 1：
> "2025-10-15，对话记录存储迁移导致查询变慢，根因：MongoDB 索引丢失"

confidence 0.72，被当成中置信度的辅助上下文传给了下游节点。但这次的问题根本不是 MongoDB 索引——是网络延迟导致的。这条错误的 RAG 上下文反而**误导了后续的分析**，System 级评估节点看到"MongoDB 索引丢失"这个历史信息后，优先往 MongoDB 方向查，浪费了好几轮。

### 解决方案

**优化 RAG 评估节点的 Prompt**，让它不只是打分，还要解释"为什么相关"：

```
之前的评估 prompt：
  "请给检索结果的相关性打分（0-1）"

改进后：
  "请判断检索结果与当前问题的相关性。对每条结果：
   1. 打分（0-1）
   2. 说明相关的原因（必须具体到哪个现象/服务/错误码匹配上了）
   3. 如果只是表面相似但根因可能不同，请标注'仅表面相似，慎用'

   特别注意：'加载慢'和'查询慢'可能由完全不同的原因导致，
   不要仅因为都包含'慢'就认为高度相关。"
```

另外加了一个**时间衰减因子**——3 个月前的案例权重自动降低。因为基础设施变化快，老案例的参考价值在下降。

## Bad Case 3：Pipeline 漏判——该在 System 级命中的，漏到了 ReAct

### 现象

输入："所有用户都无法使用语音功能"

System 级日志里明明有：
```
2025-12-01 14:23:01 ERROR [api-gateway] API-Key quota exceeded, all requests rejected
```

但 System 评估节点输出了 `isRootCause: false`，原因是模型觉得"不够确定——API-Key 配额超限不一定导致语音功能不可用"。

然后工单就漏到了 ReAct 节点，ReAct 花了 3 轮推理、调了 4 个工具，最后还是定位到了同一个问题。多花了 20 秒和几千 Token。

### 定位

看了 System 评估节点的完整输出：

```json
{
  "isRootCause": false,
  "conclusion": "API-Key 配额超限，但不确定是否影响全部语音功能",
  "confidence": 0.65,
  "severity": "medium"
}
```

模型的推理其实是对的——它确实不能 100% 确定 API-Key 超限就是根因。但在实际场景中，**System 级 ERROR + 全量用户受影响 = 几乎必然是全局故障**，不需要 100% 确定。

### 解决方案

调整 System 评估的 Prompt，降低确认门槛：

```
添加的约束：
  "判断标准调整：
   - 如果系统日志中有 ERROR 级别日志，且异常描述包含'所有用户'或'全部'等全量关键词，
     应优先认为是全局故障。设 isRootCause=true 并标注置信度。
   - 不需要 100% 确定因果关系。只要 ERROR 日志的内容能合理解释用户的异常现象，
     就标记为命中，并注明'建议确认'。
   - 宁可多命中一个误报让研发确认，也不要漏过一个真正的全局故障。"
```

同时加了一个**交叉验证**环节：Pipeline 命中后，追问模型一句"请解释：为什么这个错误会导致用户报告的现象？"如果模型能自洽地解释因果链，就确认命中；如果解释不通，再传给下一个节点。

> 这里的核心权衡是：**宁可误报不可漏报**。Pipeline 误报的成本很低（研发看一眼就知道不对，几秒钟的事），但漏报的成本很高（多走 ReAct 浪费 20 秒和几千 Token，而且 ReAct 不一定更准）。

## Bad Case 4：Pipeline 误判——把不是根因的当成根因了

### 现象

输入："用户 B 的语音偶尔卡顿"

System 级日志里恰好有一条：
```
2025-12-01 14:00:05 ERROR [scheduler] Cron job 'daily-report' execution failed
```

System 评估节点把这条错误当成了根因，输出：
> "根因：定时任务执行失败，可能影响服务稳定性"

但实际上这个定时任务是跑日报的，跟用户语音卡顿完全没关系。模型看到 ERROR 就往上面靠了。

### 定位

这属于**模型"看到 ERROR 就激动"的问题**。ERROR 日志和用户异常之间不一定有因果关系，但评估 Prompt 里当时只写了"如果有 ERROR 日志就重点关注"，没要求模型验证因果关系。

### 解决方案

在评估 Prompt 里加**因果验证步骤**：

```
改进后的评估流程（写在 Prompt 里）：
  "Step 1: 找到所有 ERROR/WARN 级别日志
   Step 2: 对每条 ERROR，判断它所在的服务/组件是否与用户的异常现象直接相关
           - 用户异常是'语音卡顿' → 相关服务：ASR、NLU、TTS、音频处理
           - 定时任务(scheduler)、日报生成(report) 等与语音链路无关的 ERROR 应排除
   Step 3: 只有与异常现象所在链路直接相关的 ERROR 才考虑为候选根因
   Step 4: 对候选根因，解释因果链：这个错误 → 导致什么 → 最终导致用户感知到什么"
```

说白了就是教模型做**因果链推理**，而不是简单的"有 ERROR = 有问题"。

## Bad Case 5：ReAct 推理链断裂——模型选了不相关的工具

### 现象

输入："用户 C 的对话响应延迟约 5 秒"

ReAct 的推理过程：

```
Step 1:
  Thought: 响应延迟可能是服务端处理慢，先查 TTS 服务日志
  Action: service_log_query(serviceName="TTS", timeRange="1h")
  Observation: TTS 日志正常，无 ERROR

Step 2:
  Thought: TTS 正常，可能是 NLU 处理慢
  Action: service_log_query(serviceName="NLU", timeRange="1h")
  Observation: NLU 日志正常

Step 3:
  Thought: 语音链路都正常...那可能是网络问题？
  Action: resource_monitor(serviceName="api-gateway", metric="network")   ← ❌ 该工具不支持查网络指标
  Observation: 工具返回空结果

Step 4:
  Thought: 查不到网络信息，那我再试试查...
  Action: resource_monitor(serviceName="api-gateway", metric="network")   ← ❌ 重复调用
  Observation: 还是空

Step 5（达到上限）: 未能定位根因
```

实际根因是用户的设备 ID 对应的 ASR 模型加载了一个大的定制模型，推理耗时比标准模型长 3 秒。但 ReAct 完全没往 ASR 方向查，而且步骤 3-4 还陷入了重复调用的死循环。

### 定位

两个问题：
1. 工具描述不够清晰，模型不知道 `resource_monitor` 只能查 CPU/GPU/内存，不能查网络
2. 没有防重复调用的机制

### 解决方案

**问题一：优化工具描述**

```
之前：
  resource_monitor: "查询服务资源使用情况"

改进后：
  resource_monitor: "查询服务资源使用情况。
    支持的 metric: cpu, gpu, memory。
    不支持: network, disk, connection。
    如需查网络相关信息，请使用 trace_query 查调用链耗时。"
```

工具描述里不只写"能做什么"，还要写**"不能做什么"和"替代方案"**。这对 ReAct 模式下的模型决策特别重要。

**问题二：加重复检测和强制终止**

```python
if is_duplicate_action(current_action, history):
    # 相同工具 + 相同参数 → 不执行，直接注入提示
    inject_message = "你刚才已经调用过相同的工具和参数，结果为空。请换一个工具或换一个查询角度。"
    history.append({"role": "system", "content": inject_message})
    continue
```

**问题三：在 ReAct Prompt 里加推理引导**

```
添加的约束：
  "推理原则补充：
   - 如果连续查了 2 个服务都正常，考虑从用户维度切入：
     用 user_log_query 查该用户的完整请求链路，看哪个环节耗时最长
   - 不要只查服务级日志，用户级日志里的 trace 信息往往更有价值
   - 如果某个工具返回空结果，说明该角度没线索，
     立刻换方向，不要重复调用同一个工具"
```

## Bad Case 6：幻觉——模型编造了不存在的日志内容

### 现象

Agent 输出的结论：
> "根因：用户日志显示错误码 5012（ASR 音频格式不支持），建议检查用户端音频编码配置"

但实际查了一下——错误码 5012 根本不存在！我们的错误码表里没有这个码。模型看日志信息不够充分，自己"编"了一个看起来合理的错误码。

### 定位

查了那次的输入日志，里面确实没有任何错误码。模型在上下文不足的时候，倾向于"编一个合理的故事"来填充，而不是诚实地说"信息不足"。

### 解决方案

**Prompt 加硬约束（最重要的一步）：**

```
在所有评估节点的 system prompt 末尾加：

"## 严格约束
1. evidence 字段必须是日志原文的精确复制，不允许任何改写或概括
2. 你只能引用上方 <log_data> 标签内提供的日志内容
3. 如果日志中没有出现某个错误码，你不得在结论中提及任何错误码
4. 如果现有信息不足以确定根因，你必须输出：
   { 'isRootCause': false, 'conclusion': '当前日志信息不足，建议补充查询' }
5. 绝对禁止编造任何日志条目、错误码、服务名或时间戳"
```

**输出校验层（Prompt 解决不了的，代码兜底）：**

```python
class HallucinationDetector:
    def check(self, output, input_logs):
        # 1. 检查 evidence 是否在原始日志中存在
        if output.get('evidence'):
            evidence_text = output['evidence']
            found = any(evidence_text in log for log in input_logs)
            if not found:
                # 证据在原始日志中找不到 → 可能是幻觉
                output['_warning'] = "结论中的证据在原始日志中未找到，可能不准确"
                output['confidence'] *= 0.5  # 降低置信度

        # 2. 检查提到的错误码是否在已知错误码表里
        error_codes = re.findall(r'错误码\s*(\d+)', str(output))
        for code in error_codes:
            if code not in KNOWN_ERROR_CODES:
                output['_warning'] = f"错误码 {code} 不在已知错误码表中，请核实"

        return output
```

> 幻觉是 LLM 应用里最阴险的问题——因为它说得**特别有信心**。用户看到"错误码 5012"这种具体信息，第一反应是相信而不是质疑。所以必须从 Prompt 约束 + 代码校验两个层面同时拦截。

## Bad Case 7：JSON 输出格式崩了——整个流程断裂

### 现象

某天突然批量出错，排查发现是 Query 预处理节点输出的不是 JSON，而是：

```
好的，我来帮你分析一下。以下是提取的结构化信息：

{
  "keyword": "对话无响应",
  "userId": "device_456"
}
```

前面多了一句"好的，我来帮你分析一下"，JSON 解析直接炸了。

### 原因

那天 OpenAI 更新了模型版本（从 gpt-4o-2024-08-06 到新版本），新版本在 JSON Mode 下偶尔会在 JSON 前面加一句客套话。之前不会出现的问题突然出现了——这就是 LLM 应用的"魅力"，**模型更新可能破坏你的系统，而且没有任何通知**。

### 解决方案

三层防御（之前就有，这次主要是加固了第二层）：

```python
def extract_json(raw: str) -> dict:
    # 1. 直接解析
    try: return json.loads(raw)
    except: pass

    # 2. 提取 ```json``` 代码块
    match = re.search(r'```(?:json)?\s*([\s\S]*?)```', raw)
    if match:
        try: return json.loads(match.group(1))
        except: pass

    # 3. 找第一个 { ... }（贪婪匹配最外层）
    match = re.search(r'\{[\s\S]*\}', raw)
    if match:
        try: return json.loads(match.group(0))
        except: pass

    # 4. 修复中文标点
    fixed = raw.replace('，', ',').replace('"', '"').replace('"', '"')
    fixed = re.sub(r',\s*}', '}', fixed)  # 尾部多余逗号
    try: return json.loads(fixed)
    except: pass

    raise JsonParseError(f"无法解析: {raw[:200]}")
```

另外锁定了模型版本号（用 `gpt-4o-2024-08-06` 而不是 `gpt-4o`），避免自动升级带来的意外。虽然拿不到最新优化，但至少行为稳定。

## Bad Case 8：耗时太长——不该进 ReAct 的进了 ReAct

### 现象

一批简单故障（如"某用户 ASR 报错 5001"），Pipeline 本应直接命中，但 User 级评估节点的置信度只到 0.7，没达到 0.8 的命中阈值，全漏到了 ReAct。ReAct 多花 15-20 秒才输出完全相同的结论。

### 定位

查了评估节点的输出，模型的分析其实是对的——定位到了 ASR 报错 5001，但它认为"还需要确认是个体问题还是全局问题"，所以没标 `isRootCause: true`。

模型太"谨慎"了。

### 解决方案

**分场景设置不同的命中阈值：**

```yaml
thresholds:
  system_eval:
    # 系统级故障一般影响面大，宁可多命中
    root_cause_confidence: 0.6

  user_eval:
    # 用户级日志里有明确 ERROR + 错误码 → 大概率就是根因
    root_cause_confidence: 0.65
    # 如果日志里有已知错误码映射，直接命中
    known_error_code_auto_resolve: true
```

另外在 User 评估 Prompt 里加了一条：

```
"如果用户日志中有明确的错误码（如 5001、5003 等），且该错误码在 RAG 提供的错误码映射中有对应含义，
 直接标记 isRootCause=true，不需要额外确认是否影响其他用户。"
```

改完之后这类简单 case 90% 都在 Pipeline 阶段就命中了，平均耗时从 25 秒降到 8 秒。

## 评估体系：怎么知道改好了没有

所有 Bad Case 修完不算完，得验证不会引入新问题。

### 离线评估集

从历史工单里挑了 120 条 case，按类型标注：

| 类型 | 数量 | 预期命中阶段 |
|------|------|------------|
| RAG 可直接命中 | 25 条 | RAG 短路 |
| System 级故障 | 30 条 | System 级 |
| User 级故障 | 40 条 | User 级 |
| 复杂故障需 ReAct | 15 条 | ReAct |
| 信息不足应返回"无法定位" | 10 条 | 兜底 |

每次改完 Prompt 或调参数后，跑一遍全量测试集。用另一个 LLM 做**自动评估**（判断输出结论和标注根因是否语义一致），再人工抽检 20%。

```python
eval_prompt = """
请判断以下两个根因结论是否指向同一个问题：
标注根因：{expected}
Agent 输出：{actual}

输出 JSON：
{ "match": true/false, "reason": "判断理由" }

注意：不要求文字完全一样，只要语义指向同一个根因即可。
"""
```

### 核心指标

| 指标 | 目标 | 当前 |
|------|------|------|
| 准确率（结论正确率） | > 85% | 91% |
| 阶段命中率（在预期阶段解决） | > 80% | 84% |
| 幻觉率 | < 3% | 1.5% |
| 平均耗时（Pipeline 路径） | < 15s | 9s |
| 平均 Token 消耗 | < 8000 | 6200 |

### 在线反馈闭环

研发用完 Agent 结论后有个反馈按钮。👎 的 case 自动进入"待复盘"队列，我每周批量分析一次，分类后进入下一轮调优迭代。

```
线上 Bad Case → 归类 → 修 Prompt / 补知识库 / 调参数
                         ↓
                   跑离线回归测试
                         ↓
                   灰度 10% 验证
                         ↓
                   全量上线
                         ↓
                   继续收集反馈 → 下一轮...
```

## 小结：几条调优心得

1. **遇到 Bad Case 先归类再动手**。不归类就改 Prompt，改一个好一个坏三个。
2. **Prompt 调优是性价比最高的手段**。大部分问题加几行约束或几个 few-shot 就能解决。
3. **RAG 召回问题一半出在知识库质量**。一条案例只有一种表述 = 只有一个"入口"，用户换个说法就搜不到。
4. **Pipeline 的判断阈值要分场景**。简单故障松一点（宁可误报），复杂判断严一点（避免噪音）。
5. **ReAct 的工具描述要写"不能做什么"**。模型不知道边界，就会在边界上反复碰壁。
6. **幻觉必须 Prompt + 代码双重拦截**。光靠 Prompt 约束不够，代码层校验 evidence 是否在原始数据中存在。
7. **锁定模型版本**。LLM 厂商的模型更新可能悄悄破坏你的系统。
8. **回归测试集是生命线**。没有回归测试，每次改动都是在赌博。
