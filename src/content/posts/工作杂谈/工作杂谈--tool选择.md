---
title: 工作杂谈--tool选择
description: "最近在排查 tool 的误召回问题，因为前面刚做过 skill 优化工具，所以我第一反应还是回到 desc、schema、case 这些东西上。顺着这个问题，又开始补 ToolBench 和 BFCL 这类工具评测项目，记录一下我的理解..."
published: 2026-06-13
tags:
    - AI
    - AI 工具
    - Function Calling
    - ToolBench
    - BFCL
category:
    - AI 开发
draft: false
---
> 最近在排查 tool 的误召回问题，因为前面刚做过 skill 优化工具，所以我第一反应还是回到 desc、schema、case 这些东西上。顺着这个问题，又开始补 ToolBench 和 BFCL 这类工具评测项目，记录一下我的理解。

## Tool 误召回的排查思路

记录 tool 的误召回的排查思路。

因为之前才做了 skill 优化的工具，所以我的第一反应就是看 tool 的 desc：

- 比较错误召回的 tool 和预期召回的 tool，两者的 desc 是否有重叠。
- 看 tool 是否明确了职责范围。

再一个 tool 和 skill 不同的点在于：

- skill 只传 name 和 desc；
- tool 多一个 schema。

如果预期工具没有触发的话，可以看一下是否 case 并不满足预期工具的 schema。比如有些参数规定了 min 最少 1 个，结果这个参数都没传入。

还有一个，如果上参考图片/视频，需要考虑 vlm 或者多模态 llm，确保模型能在理解用户上传的媒体文件，而不只是根据 url 去选择工具。

## 接下来准备学什么

接下来借此机会开始对 tool 进行深度的学习。

熟悉常见的，我直接就写摘要，锻炼下自己的摘要一篇论文的能力。

依旧安利我自己的写的 skill：

- https://github.com/xiaozhongyaonvli/skill-lab/tree/main/repo-theme-extract

能快速指定项目中筛选整理你感兴趣的方向并落阅读指南文档。

## 第一个学习 ToolBench

资料：

- 论文地址：https://arxiv.org/abs/2307.16789
- 开源项目地址：https://github.com/openbmb/toolbench

2023 年的研究，感觉大家应该都对这种方式比较熟悉了。

大致流程：

1. 建立工具召回训练/评测集（case 和对应的预期 tools doc）。
2. 训练检索模型（query -> top-k tool docs，其中非目标 tool 被临时当做反例）。
3. 线上时 query -> top-k tool。

召回评测标准，简单来说就是：

- 正确工具是否被召回；
- 召回比例多少；
- 排名是否靠前。

和 rag 那套评测一样的指标。

## 第二个学习 BFCL

资料：

- 开源项目地址：https://github.com/ShishirPatil/gorilla/tree/main/berkeley-function-call-leaderboard

这个项目我认为有些比较新的角度和观点，大致摘要下。

### 好角度 1：工具评测不能都是一个标准

而要分类去评测：

- 类 1. simple ast：一个 query 对应一个工具，只需判断函数名和参数是否正确。
- 类 2. Multiple ast：多个必要工具并且有依赖关系。
- 类 3. Parallel ast：多个工具调用，但是工具之间没有严格的先后顺序。
- 类 4. Relevance：应该调用工具时，有无调用工具。
- 类 5. Irrelevance：不应该调用工具时，模型有没有乱调用工具。

我觉得一个是关注了：不该调用时，是否没有调用；二是关注了多工具调用时，并行还是依赖两种情况。

都是容易忽略的点，但是工具错误调用比文本幻觉的危险大，还是应该引起重视。

### 好角度 2：test case 不只构造 query 和标准答案

它的 test case 不仅构造 query 和标准答案，同时还构造 tool 列表。

这个构造列表可以：

- 包含正确工具；
- 故意缺失工具；
- 放相似干扰工具；
- 用来测试选择、拒识和鲁棒性。

我感觉这种测试对应工具系统的稳定性真的挺重要的。

比如新的工具的插入，或者某些工具异常被迫下线、无法调用，这个时候这些例子实际上就会出现。

### 好角度 3：checker 也要跟着分类

这个其实比较常见或者说容易想到了。

前面分类了 case，实际后续的 checker 也需要针对每个类去做。该调用和不该调用两个可以快速地评测下，其他的也是构造自己的 checker 即可。



实习中遇到的解决了顺手记录下，写的比较随意哈哈