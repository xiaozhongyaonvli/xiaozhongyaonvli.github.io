---
title: AI Function Calling 出现幻觉怎么办？接口幂等 + 人工防护实战指南
description: "最近在搞 AI Agent 的项目，让大模型去调外部 API。结果发现一个很头疼的问题 — 模型有时候会\"编\"参数、调不存在的函数、甚至伪造调用结果。查了一圈资料才意识到，这就是 Function Calling 场景下的幻觉问题，而..."
published: 2025-12-08
tags:
    - AI
    - Function Calling
    - 幻觉
    - 幂等
category:
    - AI 开发
draft: false
---
> 最近在搞 AI Agent 的项目，让大模型去调外部 API。结果发现一个很头疼的问题 — 模型有时候会"编"参数、调不存在的函数、甚至伪造调用结果。查了一圈资料才意识到，这就是 Function Calling 场景下的幻觉问题，而且比普通的文本幻觉危险得多，因为它会直接触发真实的系统操作。

## 先搞清楚：Function Calling 幻觉到底长什么样？

普通的 LLM 幻觉就是"一本正经胡说八道"，但在工具调用场景下，幻觉的表现形式更具体，也更危险：

**1. 幻觉函数名 — 调一个根本不存在的工具**

比如你只定义了 `get_weather` 和 `send_email` 两个工具，模型可能会自信地调用一个 `turn_on_TV()` — 你压根没提供这个能力。有开发者反馈，让 LLM 写代码时它甚至会调用 `Server::terminate_debuggee()` 这种不存在的方法。

**2. 幻觉参数值 — 参数是编出来的**

这是最常见的情况。有人做了个餐厅点餐机器人，用 Function Calling 查菜单数据，结果模型时不时会返回菜单里根本没有的菜品价格和配料。数据是 JSON 明明白白给它的，它偏要"创造性发挥"。

**3. 伪造执行结果 — 没成功但说成功了**

研究表明，单 Agent 系统在操作失败时可能会 claim success，自己给自己编一个"操作已完成"的回复。这要是发生在转账、下单这种场景，后果不堪设想。

**4. 重复调用 — 同一个函数被调三次**

OpenAI 社区有开发者反馈，GPT 的 Function Calling 会出现同一个函数被重复执行 3 次的情况，每次给出相同的回复。

> 这里有个关键的认知：**工具调用的本质是文本补全 + JSON 约束**。模型并不是真的"理解"了你的 API，它只是在根据上下文生成最可能的 JSON 结构。所以它会猜、会编、会瞎填 — 这不是 bug，这是 LLM 的工作原理。

## 为什么 Function Calling 比普通幻觉更危险？

OpenAI 的研究论文指出了一个根本原因：**标准的训练和评估流程奖励"猜测"而不是"承认不确定"**。当模型遇到不确定的问题时，它不会停下来说"我不知道"，而是生成统计上最可能的续写内容。

在问答场景中，猜错了顶多给个错误答案。但在 Function Calling 场景中：

- 猜错参数 → 可能触发错误的数据库操作
- 编造函数 → 系统报错，用户体验崩溃
- 重复调用 → 重复扣款、重复下单、重复发邮件
- 伪造成功 → 用户以为操作完成，实际啥都没发生

一个真实案例：某团队的 AI Agent 工作流在第 2 步失败了，但第 1 步已经创建了客户记录。重试时又创建了一条。一周后他们发现了 200 多条孤儿记录，每条记录的客户都收到了重复的账单通知。

## 防御策略一：接口幂等化 — 让重复调用不产生副作用

幂等性的意思是：**同样的请求执行一次和执行一百次，效果完全一样**。HTTP GET 天然幂等，但 POST（创建资源）、转账、发消息这些操作不是。当 AI Agent 因为幻觉或网络重试多次调用同一个接口时，幂等设计能救你一命。

### 方案 1：幂等 Key（最推荐）

核心思路：客户端生成一个唯一的 idempotency_key，服务端拿到后先查缓存，有记录就直接返回上次的结果，没有才真正执行。

```python
def charge_customer(amount: int):
    bank_api.charge(amount)

def charge_customer(amount: int, idempotency_key: str):
    # 先查缓存，避免重复执行
    cached = redis.get(f"idempotent:{idempotency_key}")
    if cached:
        return json.loads(cached)  # 直接返回上次结果

    # 真正执行业务逻辑
    result = bank_api.charge(amount, key=idempotency_key)

    # 缓存结果，设置 24 小时过期
    redis.setex(f"idempotent:{idempotency_key}", 86400, json.dumps(result))
    return result
```

> 有个细节要注意：**不要缓存 500 错误的响应**。幂等缓存是为了防止成功操作的重复执行，如果把失败也缓存了，重试时会一直返回失败结果，反而更糟。

### 方案 2：状态机管控

对于多步骤的操作流程（比如"创建订单 → 扣库存 → 扣款 → 发通知"），用状态机来管理每一步的状态：

```python
def process_order(order_id: str, step: str):
    order = db.get_order(order_id)

    # 已经执行过这一步了，直接跳过
    if order.current_step >= STEP_ORDER[step]:
        return {"status": "already_done", "order": order}

    # 执行当前步骤
    execute_step(order, step)
    order.current_step = STEP_ORDER[step]
    db.save(order)
```

### 方案 3：去重校验

用 `(user_id, tool_name, params_hash)` 做联合去重 key，配合时间窗口：

```python
def deduplicate_tool_call(user_id, tool_name, params):
    dedup_key = f"{user_id}:{tool_name}:{hash(frozenset(params.items()))}"
    if redis.exists(dedup_key):
        return get_cached_response(dedup_key)
    # ... 执行并缓存
```

## 防御策略二：高危操作人工防护 — 关键时刻让人来拍板

再强的幂等设计也防不住模型调错接口的情况。对于删库、转账、发邮件等不可逆操作，**人工确认（Human-in-the-Loop）** 是最后一道防线。

### 分级管控：不是所有操作都需要人审

```
低风险（自动执行）：查询操作、读取数据、搜索
中风险（日志+异步审核）：修改用户信息、更新配置
高风险（实时人工确认）：转账、删除、发送外部消息
```

### 实现模式：拦截 → 暂停 → 确认 → 放行

```python
HIGH_RISK_TOOLS = ["transfer_money", "delete_record", "send_email", "drop_table"]

def execute_tool_call(tool_name, params, user_context):
    # 1. 参数校验（后面会讲）
    validated = validate_params(tool_name, params)
    if not validated.ok:
        return error_feedback_to_model(validated.error)

    # 2. 风险评估
    if tool_name in HIGH_RISK_TOOLS:
        # 暂停执行，发给人工审核
        approval = request_human_approval(
            tool_name=tool_name,
            params=params,
            user=user_context,
            timeout=300  # 5 分钟超时
        )
        if not approval.approved:
            return "操作已被管理员拒绝，原因：" + approval.reason

    # 3. 放行执行
    return call_tool(tool_name, params)
```

> 人工审核的成本没有想象中那么高。真正的高危操作本来就不应该频繁发生，如果你的 Agent 频繁触发人工审核，那说明 Prompt 或工具设计有问题。

### 影子运行（Shadow Mode）

对于新上线的 Agent，可以先用"影子模式"跑一段时间 — 所有工具调用都记录日志但不真正执行，人工审核通过后才正式切到生产环境。

## 防御策略三：参数校验 + 错误回馈闭环

**永远不要信任模型生成的参数。** 这是从事 Agent 开发后我最大的体会。

### 用 JSON Schema + enum 做硬约束

```python
tools = [{
    "type": "function",
    "function": {
        "name": "set_room_temperature",
        "description": "设置房间温度",
        "parameters": {
            "type": "object",
            "properties": {
                "room_id": {
                    "type": "string",
                    "enum": ["living_room", "bedroom", "kitchen"]  # 白名单！
                },
                "temperature": {
                    "type": "integer",
                    "minimum": 16,
                    "maximum": 30  # 范围限制
                }
            },
            "required": ["room_id", "temperature"]
        }
    }
}]
```

`enum` 是限制模型"自由发挥"的利器。对于类别型字段，一定要用 enum；对于数值型字段，一定要设 min/max。

### 校验失败时：喂回模型，而不是崩溃

```python
def validate_and_execute(tool_name, params):
    # 校验参数
    errors = validate_params(tool_name, params)
    if errors:
        # 关键：把错误信息返回给模型，让它自己修正
        return {
            "role": "tool",
            "content": f"参数校验失败：{errors}。请检查后重新调用。"
        }
    # 校验通过，正常执行
    return execute_tool(tool_name, params)
```

（这个"错误回馈 → 模型自修正"的闭环非常重要，比直接抛异常然后死掉好太多了）

## 防御策略四：架构层面的幻觉抑制

除了在接口层做防护，从架构设计上也可以大幅降低幻觉概率。

### 语义工具筛选 — 别一股脑把所有工具都给模型

研究表明 **工具数量越多，幻觉概率越高**。解决办法：先用向量相似度筛选出跟用户请求相关的工具，只把少量工具传给模型。

测试数据很惊人：这种方法**减少了 86.4% 的工具调用错误，同时节省了 89% 的 token 消耗**。

```python
from sentence_transformers import SentenceTransformer
import faiss

model = SentenceTransformer('all-MiniLM-L6-v2')
tool_embeddings = model.encode([t["description"] for t in all_tools])
index = faiss.IndexFlatL2(tool_embeddings.shape[1])
index.add(tool_embeddings)

def filter_tools(user_query, top_k=5):
    query_vec = model.encode([user_query])
    _, indices = index.search(query_vec, top_k)
    return [all_tools[i] for i in indices[0]]
```

### 工具描述要写成"指令"，不是"文档"

模型读 docstring 来决定调哪个工具。含糊的描述会增加幻觉概率：

```python
"""获取数据"""

"""
查询指定客户最近 5 条客服工单记录。
当用户询问历史投诉或服务记录时使用此工具。
不要用于金融交易或账户余额查询。
"""
```

### 多 Agent 交叉验证

单个 Agent 幻觉了没人发现。解决办法：让多个 Agent 扮演不同角色互相校验：

- **Executor**：执行操作
- **Validator**：验证参数和结果的合理性
- **Critic**：挑毛病，质疑可疑输出

这种模式在高风险场景（金融、医疗）里特别有价值。

## 实战清单：一个工具上线前的 Checklist

最后整理一个实用的检查清单，每次给 Agent 新增工具时过一遍：

| 检查项 | 说明 |
|--------|------|
| 参数是否有 enum/range 约束？ | 能用白名单就不要让模型自由填写 |
| 接口是否幂等？ | 特别是写操作，必须支持安全重试 |
| 校验失败是否有错误回馈？ | 返回可读的错误信息，而非直接报异常 |
| 高危操作是否有人工确认？ | 转账、删除、发消息等操作必须有审批流 |
| 工具描述是否足够清晰？ | 写明用途、适用场景、禁止场景 |
| 工具数量是否过多？ | 考虑语义筛选，减少模型的选择压力 |
| 是否有完整的调用日志？ | 出了问题要能回溯 |
| 输入是否做了安全过滤？ | 防路径穿越、SQL 注入等 |

## 小结

说到底，Function Calling 场景下的幻觉问题不是靠"写个好 Prompt"就能解决的。Prompt 是建议，不是约束 — 模型可以无视它。真正靠谱的防御是**在代码层和架构层做硬约束**：接口幂等保证重复调用无副作用，参数校验把住入口，高危操作交给人类拍板，多 Agent 交叉验证兜底。核心理念就一句话：**永远不要信任模型的输出，像对待用户输入一样对待 LLM 的工具调用。**
