---
title: RAG 评估方案 — 怎么判断你的 RAG 到底行不行
date: 2025-12-10 19:00:00
tags:
  - RAG
  - AI
  - 评估方法
categories:
  - AI 开发
---

# RAG 评估方案 — 怎么判断你的 RAG 到底行不行

> 搭个 RAG demo 其实很快，LLM + 向量库一接就能跑起来。但跑起来之后呢？怎么知道它检索的对不对、生成的准不准？这块我之前一直比较模糊，最近系统看了一圈资料，整理一下。

## 为什么需要专门评估 RAG？

先说一个让我有点意外的数据：有研究表明，**检索准确率只能解释大约 60% 的 RAG 最终质量**，剩下的取决于生成阶段对上下文的利用能力。也就是说，光看"检索到的文档对不对"是远远不够的。

RAG 系统容易出的问题其实挺多的：

- **幻觉**：LLM 无视检索到的上下文，自己编了一个答案
- **上下文忽略**（Google DeepMind 管这叫 "context neglect"）：明明检索到了正确信息，但模型就是不用，转头用自己的参数知识回答
- **Lost in the Middle**：关键信息在文档中间位置，模型只关注开头和结尾，中间的直接跳过了
- 检索精度不够，召回一堆无关文档，噪声太大

所以评估不能只看最终答案对不对，得把检索和生成拆开来看。

## 两大评估维度：检索 vs 生成

这个是核心思路 — **分阶段评估**。

### 检索质量（Retrieval）

回答的是："系统找到有用信息了吗？"

几个常见指标：

| 指标 | 说明 |
|------|------|
| **Precision@k** | 检索回来的 top-k 个文档里，有多少是真正相关的 |
| **Recall@k** | 所有相关文档中，被成功检索到的比例 |
| **Hit Rate** | top-k 里是否至少包含一个相关文档（二元判断） |
| **NDCG@k** | 不光看检索到没有，还看相关文档是不是排在前面 |
| **MRR** | 第一个相关文档出现在第几位 |

我的理解是，Precision 和 Recall 是基础，NDCG 关注排序质量（这个在有 reranker 的场景特别重要），Hit Rate 适合做快速的 smoke test。

### 生成质量（Generation）

回答的是："模型的回答靠谱吗？"

这块又可以分成两种评估方式：

**有参考答案的（reference-based）**：
- 语义相似度打分
- LLM-as-judge 对比正确答案

**无参考答案的（reference-free）**：
- 忠实度（Faithfulness）：回答是否基于检索到的上下文
- 完整性：是不是回答完了所有问题
- 幻觉检测

> 无参考评估这个思路挺好的，因为在实际场景中你不可能为每个问题都准备标准答案。RAGAS 框架的一大卖点就是很多指标不需要 ground truth。

## RAGAS 的核心指标（重点）

RAGAS（Retrieval-Augmented Generation Assessment Suite）是目前最主流的 RAG 评估框架，开源的。它定义了几个非常经典的指标，我觉得每个做 RAG 的都应该了解一下。

### 1. Faithfulness（忠实度）

衡量的是：**生成的回答有多少内容是能从检索上下文中推导出来的**。

简单说就是检测幻觉。计算逻辑大概是这样的：

```
Faithfulness = 能从上下文推导的陈述数 / 回答中的总陈述数
```

分值 0~1，越高越好。比如回答里有 5 句话，其中 4 句能在检索到的文档里找到依据，那 Faithfulness = 0.8。

有个变体叫 **FaithfulnesswithHHEM**，用的是 Vectara 的 HHEM-2.1-Open 模型（基于 T5 的分类器）来检测幻觉，不需要再调用 LLM，适合生产环境用，又快又省钱。

### 2. Answer Relevancy（答案相关性）

衡量的是：**回答和问题有多相关**。

这个指标的计算方式挺有意思的 — 它不是直接比较问题和答案，而是反过来：先从答案反推出 N 个"可能的问题"，然后算这些反推问题和原始问题的**余弦相似度的均值**。

```
Answer Relevancy = mean(cosine_similarity(original_question, generated_questions))
```

如果答案跑题了或者说了一堆废话，反推出的问题就会和原始问题差很远，分数自然就低。

（刚开始看这个计算方式我觉得有点绕，但仔细想想确实比直接做文本匹配更合理）

### 3. Context Precision（上下文精准度）

衡量的是：**相关的文档块是不是排在了不相关的前面**。

这个主要看 reranker 的效果。如果你检索回来 10 个 chunk，相关的 3 个排在第 1、2、3 位，那精准度就很高；如果相关的排在第 5、8、10 位，那就有问题了。

### 4. Context Recall（上下文召回）

衡量的是：**回答所需的信息是不是都被检索到了**。

注意这个指标**需要人工标注的 ground truth**，是 RAGAS 里唯一一个需要参考答案的指标。它会检查标准答案里的每个关键事实，看是不是都能在检索上下文中找到对应。

> 这四个指标组合起来其实覆盖面很全了：Context Precision + Context Recall 评估检索，Faithfulness + Answer Relevancy 评估生成。有问题的时候能很快定位到底是哪个环节出了岔子。

## 其他评估工具对比

除了 RAGAS，还有几个值得关注的：

**DeepEval**：思路很酷，把 LLM 评估当成单元测试来跑，能直接集成 PyTest。写法大概是这样：

```python
from deepeval.metrics import FaithfulnessMetric, AnswerRelevancyMetric
from deepeval.test_case import LLMTestCase
from deepeval import evaluate

test_case = LLMTestCase(
    input="什么是 RAG？",
    actual_output="RAG 是一种结合检索和生成的技术...",
    retrieval_context=["RAG（Retrieval-Augmented Generation）..."]
)

faithfulness = FaithfulnessMetric(threshold=0.7)
relevancy = AnswerRelevancyMetric(threshold=0.7)

evaluate([test_case], [faithfulness, relevancy])
```

（这种写法对后端开发者很友好，直接融入现有的测试流程就行）

**Giskard**：侧重安全性和鲁棒性评估，会帮你生成对抗性测试用例。

**LlamaIndex**：本身就是 RAG 框架，自带评估模块，和自己的 pipeline 集成最顺畅。

**RAGEval**（ACL 2025 论文）：学术界的方案，提出了 Completeness、Hallucination、Irrelevance 三个指标，和人工评估的一致性很高。

## 实战建议

踩坑总结 + 最佳实践：

**1. 从 chunking 开始排查**

RAG 效果不好，第一反应不应该是换模型，而是看 chunking 策略对不对。chunk 太大噪声多，太小又会丢失上下文。根据文档类型选择不同的分割策略。

**2. 用合成数据做冷启动**

没有真实用户数据的时候，可以从知识库自动生成 QA 对来做测试。RAGAS 和 DeepEval 都支持这个功能。

**3. 分阶段评估，别只看端到端**

端到端准确率 80% 看着还行，但可能是检索 95% + 生成 84%，也可能是检索 82% + 生成 98%。优化方向完全不同。

**4. LLM-as-judge 不是万能的**

LLM 打分有一定的不确定性，最好结合规则检查和人工抽检。比如先用 LLM 批量打分，然后人工 review 低分和边界 case。

**5. 在 CI/CD 里集成评估**

把评估跑进持续集成流程，每次改 prompt、换模型、调参数都自动跑一遍评估 suite，防止回归。DeepEval 的 PyTest 集成方式就很适合这个场景。

**6. 关注 Embedding 模型的选择**

可以参考 MTEB Leaderboard 选合适的 embedding 模型，确保它能捕捉你领域的语义信息。通用 embedding 在垂直领域可能表现很差。

## 小结

RAG 评估核心就是拆开看：检索用 Precision/Recall/NDCG，生成用 Faithfulness/Answer Relevancy。RAGAS 是目前最成熟的框架，四个指标基本够用。工程上记得把评估自动化，别等上线了才发现问题。
