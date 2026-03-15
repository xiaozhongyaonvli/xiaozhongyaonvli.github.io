---
title: Claude Code 深度使用手册：从入门到工程化的完整指南
description: "Claude Code 是 Anthropic 官方出的 CLI 编程工具。用了大半年，我对它的认知经历了几次跃迁——从\"会话式写代码\"到\"CLAUDE.md 驱动开发\"，再到现在用 Hooks、Agent Teams、GHA 搭建了..."
published: 2025-12-26
tags:
    - Claude Code
    - AI 工具
    - CLI
category:
    - 效率工具
draft: false
---
> Claude Code 是 Anthropic 官方出的 CLI 编程工具。用了大半年，我对它的认知经历了几次跃迁——从"会话式写代码"到"CLAUDE.md 驱动开发"，再到现在用 Hooks、Agent Teams、GHA 搭建了一套完整的 AI 辅助工程体系。这篇文章把我认为最有价值的功能和实践整理一遍，当手册查就行，不用一口气看完。

## 一、CLAUDE.md —— 一切的起点

在代码库里想高效用 Claude Code，**第一件事就是写好 CLAUDE.md**。它不是可选的配置文件，它是你和 Agent 之间最重要的"合同"。

### 1.1 层级结构

CLAUDE.md 有三个作用域层级，Claude 会自动加载当前路径上的所有层级：

| 位置 | 作用域 | 放什么 |
|------|--------|--------|
| `~/.claude/CLAUDE.md` | 全局 | 个人偏好——语言、代码风格、常用快捷方式 |
| 项目根目录 `CLAUDE.md` | 项目级 | 团队约定——框架、命名规范、测试命令、部署流程 |
| 子目录 `CLAUDE.md` | 目录级 | 特定模块的规则，比如 `src/auth/CLAUDE.md` 单独约束认证模块 |

大型 Monorepo 里这个层级结构非常有用——全局规则放根目录，各子包的特殊规则放各自目录，Claude 进入不同目录自动切换上下文。

### 1.2 怎么写好 CLAUDE.md

写了大半年总结出几个原则：

**先设限制，再写指南。** CLAUDE.md 应该从小处着手，根据 Claude 常犯的错误逐步添加。不要一上来就写百科全书——指令越多，遵循质量越均匀下降。目标控制在 200 行以内。

**不要 @ 引用文档。** 很多人喜欢在 CLAUDE.md 里 `@` 引用其他文件，但这会在每次运行时把整份文件塞进上下文。正确做法是告诉 Agent "什么时候"去读：

```markdown
遇到 FooBarError 时，请参阅 path/to/docs.md 获取排查步骤。

@path/to/docs.md  ← 每次都把整个文件塞进上下文
```

**不要只说"禁止"。** 纯粹的负面约束会让 Agent 在认为必须用它的时候左右为难。**永远提供替代方案**：

```markdown
禁止使用 --force 标志

禁止使用 --force 标志，改用 --force-with-lease，它更安全
```

**用 Linter 管代码风格，不要用 CLAUDE.md。** Claude 能读 `.eslintrc`、`.prettierrc`，格式化的事交给工具链，CLAUDE.md 只记录 Linter 管不了的东西——团队约定、架构决策、内部 API 用法。

**记录约定，不要记录 API。** Claude 能读 API 文档，但推断不出你们团队的模式。比如"我们的 Service 层不直接抛异常，统一返回 Result 类型"这种，必须写进 CLAUDE.md。

**把 CLAUDE.md 当压力测试。** 如果你的 CLI 命令复杂到需要在 CLAUDE.md 里写长篇说明，与其解释，不如写一个简单的 bash 包装器。CLAUDE.md 保持精简，反过来会迫使你优化工具链。

### 1.3 一个实际示例

```markdown

## 技术栈
- Python 3.12 + FastAPI
- 依赖管理：uv（不要用 pip）
- 测试：`make test`
- 类型检查：`make typecheck`
- 格式化：`make format`（会自动运行 ruff + black）

## 约定
- Service 层返回 Result[T, Error] 类型，不要直接 raise
- API 路由文件放在 src/api/routes/ 目录，一个资源一个文件
- 数据库迁移用 alembic，新建迁移后必须跑 `make test-db`

## 注意事项
- 遇到 ImportError 相关问题，参阅 docs/troubleshooting.md
- 禁止直接操作 prod 开头的环境变量文件，用 `./scripts/env.sh` 管理
```

## 二、上下文管理 —— 200k token 怎么花

强烈建议在每个会话中至少跑一次 `/context`，看看 200k token 的上下文窗口是怎么被分配的。

在 Monorepo 里，全新会话的基线成本大约是 **20k token（10%）**，剩下 180k 用于实际工作——而这很快就会被填满。

### 三种上下文管理策略

**策略一：`/clear` + `/catchup`（推荐日常使用）**

切换任务时直接 `/clear` 清空上下文，然后用自定义命令重新加载必要上下文：

```markdown
<!-- .claude/commands/catchup.md -->
请读取我当前 git 分支中所有已更改的文件，了解当前的工作进展。
```

`/clear` 清除状态 → `/catchup` 重新定位 → 继续工作。简单可靠。

**策略二：`/compact`（谨慎使用）**

上下文超过 80% 时可以用 `/compact` 压缩。可以指定保留重点：

```
/compact retain the error handling patterns and the test structure
```

但自动压缩过程不透明，容易丢信息。我遇到过压缩后 Claude 把关键上下文"忘"了的情况，所以只在上下文确实快满、又不想丢失当前工作流时才用。

**策略三："记录并清除"（处理大型任务）**

大功能做到一半上下文快满了，让 Claude 先把进展导出：

```
请把当前的计划、已完成部分、待办事项输出到 .claude/progress.md
```

然后 `/clear`，新会话读这个文件继续。相当于给 Agent 创建了一个**外部记忆**。

## 三、SubAgent 与多 Agent 并发

### 3.1 SubAgent 基础

一个复杂任务需要大量 token 处理过程，但最终输出可能只有几百 token。SubAgent 的核心价值就是：**把处理过程的 token 消耗隔离到独立上下文窗口，只把最终结论返回给主 Agent**。

```
主 Agent（保持清爽）
   Task("分析 src/auth/ 的代码质量")  → 子 Agent A → 返回摘要
   Task("检查 API 路由一致性")         → 子 Agent B → 返回摘要
   Task("运行测试并汇总失败项")         → 子 Agent C → 返回摘要
```

**前台 vs 后台：**
- **前台**：主 Agent 等待子任务完成再继续。适合结果之间有依赖的场景
- **后台**：子任务后台跑，你继续和主 Agent 聊别的。按 `Ctrl+B` 可以把正在前台跑的任务切到后台

**并发上限**：最多 10 个并发任务。超过会排队。

**省钱技巧**——主 Agent 用好模型做决策，SubAgent 用快模型执行：

```bash
export CLAUDE_CODE_SUBAGENT_MODEL="claude-sonnet-4-20250514"
```

### 3.2 SubAgent 的坑

**坑一：上下文被"关起来"了。** 创建了专用 SubAgent 后，相关知识对主 Agent 隐藏了。主 Agent 无法整体思考，被迫调用 SubAgent 才能获取信息。

**坑二：强迫 Agent 按人类的工作流走。** 自定义 SubAgent 等于你在指令"必须如何委派任务"，而委派本来就是 Agent 该自己决定的。

所以我的建议是：**把关键上下文放在 CLAUDE.md 里，让主 Agent 用内置的 Task 功能自行决定何时、如何委派**。对于固定流水线任务（安全审计、格式化检查），专用 SubAgent 没问题。

### 3.3 Agent Teams —— 多 Agent 协作

Agent Teams 是更高级的多 Agent 模式。和 SubAgent 的区别：

| 特性 | SubAgent | Agent Teams |
|------|---------|-------------|
| 通信 | 只向主 Agent 汇报 | 成员间可以直接通信 |
| 任务管理 | 无共享任务 | 共享任务列表 |
| 上下文 | 在单会话内 | 各自独立上下文窗口 |
| 适用场景 | 聚焦执行 | 需要协作的复杂项目 |

适合的场景：大规模并行分析、多角度代码审查（安全 + 性能 + 测试覆盖同时进行）、多假设并行验证调试。

### 3.4 三种并行开发方式

**方式一：会话内 Task 并行（最常用）**

一个会话里用 Task 生成多个子 Agent 并行跑。适合代码审查、探索大型代码库。

**方式二：多终端 + Session 隔离**

开多个终端窗口，各跑独立 Session：

```bash
claude --session "feature-auth"
claude --session "feature-payment"
```

每个 Session 完全隔离。适合同时开发多个不相关功能。注意同时改同一批文件会有冲突风险——用 Git Worktree 隔离。

**方式三：Headless 模式批量脚本（大规模操作）**

```bash
#!/bin/bash
for mod in $(find src/modules -maxdepth 1 -type d); do
  claude -p "在 $mod 中将所有 lodash 导入替换为 es-toolkit，确保测试通过" \
    --output-format json \
    --max-turns 20 &
done
wait
echo "全部完成"
```

最暴力也最可控。注意 API 速率限制。

## 四、Hooks —— 确定性的质量门禁

CLAUDE.md 里的规则是"应该做"，Hooks 是"必须做"。个人项目可以不用，团队协作和 CI 场景必备。

### 4.1 Hook 事件类型

| 事件 | 触发时机 | 典型用途 |
|------|---------|---------|
| **PreToolUse** | 工具执行前 | 安全拦截（exit code 2 阻止执行） |
| **PostToolUse** | 工具执行后 | 自动格式化、lint、运行测试 |
| **UserPromptSubmit** | 用户提交提示时 | 注入额外上下文 |
| **SessionStart** | 会话开始时 | 加载最新工单、代码规范检查清单 |
| **Stop** | Claude 完成响应时 | 桌面通知 |

### 4.2 Hook 类型

- **command**：执行 Shell 命令
- **http**：POST 到指定 URL（适合 webhook 集成）
- **prompt**：单轮 LLM 评估（轻量审查）
- **agent**：多轮 LLM 验证（深度审查）

### 4.3 几个实用 Hook

**提交时强制测试通过：**

PreToolUse 钩子包裹 `Bash(git commit)` 命令——检查测试是否通过，不通过就阻止提交，迫使 Claude 进入"测试-修复"循环。

**代码写入后自动格式化：**

PostToolUse 钩子在每次 Edit/Write 后自动运行 `prettier` 或 `gofmt`。

**防止写入敏感文件：**

PreToolUse 钩子阻止写入 `.env`、生产配置文件、`.git` 目录。

### 4.4 关键原则：不要在写入时阻断

**刻意不在 Edit 或 Write 操作上设阻断钩子**。在 Agent 执行计划的中途打断它，会让它困惑甚至"摆烂"。更好的做法是让它完成整个计划，在提交阶段检查最终结果。

> 就像代码审查应该在 PR 阶段做，而不是在工程师每写一行代码时都打断他。

配置方式：在 Claude Code 中输入 `/hooks` 打开交互式菜单，或直接编辑 `.claude/settings.json`。

## 五、Skills 与 MCP —— 能力扩展的两条路

### 5.1 Skills（自定义斜杠命令）

Skills 是 `.claude/skills/` 或 `.claude/commands/` 目录下的 Markdown 文件，定义了可复用的工作流。输入 `/` 即可触发。

我只保留了两个高频命令：

- `/catchup`：读取当前 git 分支的所有变更文件
- `/pr`：清理代码、暂存更改、准备 Pull Request

> 如果你有一长串复杂的斜杠命令，可能在制造反模式。Agent 的意义在于你可以用自然语言完成任何事。过多的"魔法命令"反而增加了学习成本。

Skills 支持 YAML frontmatter 控制行为——比如 `user-invocable: false` 表示只能由 Claude 自动触发，不对用户暴露。

### 5.2 MCP（Model Context Protocol）

MCP 是连接外部工具和数据源的标准协议。三种传输模式：

| 模式 | 延迟 | 适用场景 |
|------|------|---------|
| **stdio** | <5ms | 本地服务器（推荐默认） |
| **SSE** | 较高 | 远程服务器连接 |
| **HTTP Streamable** | 中等 | 生产环境云部署 |

添加 MCP 服务器：

```bash
claude mcp add brave-search -- npx -y @anthropic/mcp-server-brave-search

claude mcp add --transport sse my-server https://example.com/mcp/sse
```

### 5.3 Skills vs MCP 怎么选

我的经验是分场景：

| 场景 | 选择 | 理由 |
|------|------|------|
| 有状态的复杂环境（Playwright、数据库连接） | MCP | 需要管理连接和状态 |
| 无状态的工具调用（Jira、AWS、GitHub） | CLI + Skills | 更轻量，Agent 自己写脚本调用 |
| 可复用的工作流（代码审查、发布流程） | Skills | 流程固化，按需加载 |

> MCP 的最佳角色应该是**安全网关**——管理认证、网络和安全边界，提供几个高层次的工具，然后让开。不要变成臃肿的 API 镜像。

## 六、Plan 模式 —— 大活必备

对于大型功能变更，Plan 模式让 Claude 先列计划、你确认了再动手：

```
> 我需要给用户系统加上 RBAC 权限控制，进入 plan 模式帮我规划一下

Claude: 好的，我的计划如下：
1. 创建 role/permission 数据表 (检查点 1)
2. 实现 RBAC 中间件 (检查点 2)
3. 改造现有 API 路由 (检查点 3)
4. 编写测试用例

你觉得这个方案可行吗？
```

定义好"检查点"很重要——这是 Claude 需要停下来让你验收的地方，避免一口气改太多发现方向不对。

用多了会培养出一种直觉：**提供多少最少的上下文，能让 Claude 给出好计划，又不会在实现阶段跑偏**。

## 七、Git Worktree —— 并行开发的安全网

多个 Agent 同时改代码，最怕文件冲突。Git Worktree 是解决方案：

```bash
git worktree add ../project-auth feature/auth
git worktree add ../project-payment feature/payment

cd ../project-auth && claude --session "auth"
cd ../project-payment && claude --session "payment"
```

每个 Worktree 是独立的工作目录，改代码互不影响。Claude Code 也内置了 Worktree 支持，可以用 `--worktree` 标志直接创建：

```bash
claude --worktree feature-name
```

SubAgent 也支持 Worktree 隔离（`isolation: worktree`），退出时无变更自动清理，有变更提示保留或删除。

**核心原则：读操作随便并行，写操作必须隔离。**

## 八、GitHub Action —— 可能是最被低估的功能

Claude Code GHA 的概念很简单：在 GitHub Actions 里运行 Claude Code。但正是这种简单性让它特别强大。

### 基本用法：自动 PR 审查

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

用 `/install-github-app` 可以一键设置，之后在 PR 评论里 `@claude` 就能触发。

### 进阶玩法：事件驱动的自动修复

不只是 PR 审查——可以构建"随处触发 PR"的工具：

- 从 **Slack** 触发："@claude 修复 #1234 这个 bug"
- 从 **Jira** 触发：工单状态变更自动生成实现 PR
- 从 **CloudWatch 告警** 触发：报警自动分析日志、定位问题、提交修复

GHA 的日志就是完整的 Agent 日志。定期审查这些日志，找出 Agent 常犯的错误，改进 CLAUDE.md。这形成了一个**数据驱动的飞轮**：

```
Agent 犯错 → 日志记录 → 发现模式 → 改进 CLAUDE.md / 工具 → Agent 更聪明
```

## 九、实用技巧速查

### 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+B` | 把前台任务切到后台 |
| `Esc` | 中断当前响应 |

### 常用命令

```bash
claude --continue

claude --resume

claude -p "你的提示" --output-format json

/context

/hooks
```

### 配置建议

```json
// settings.json
{
  // 调高超时，复杂命令不会被掐断
  "MCP_TOOL_TIMEOUT": 60000,
  "BASH_MAX_TIMEOUT_MS": 300000
}
```

### 安全建议

- 定期审计 `permissions` 中允许 Claude 自动运行的命令列表
- 将 `curl`、`wget`、`nc`、`ssh` 加入全局 deny 规则
- 审查第三方项目的 CLAUDE.md 和 `.claude/settings.json`（已有公开仓库中发现恶意注入的案例）

### Shell 别名（提效）

```bash
alias cc="claude"
alias cch="claude --model haiku"  # 用快模型处理简单任务
```

## 小结

Claude Code 的核心价值不是"跟 AI 聊天写代码"，而是**围绕 Agent 建立一套工程体系**：

- **CLAUDE.md** 定规矩——精简、具体、持续迭代
- **Hooks** 做质量门禁——确定性保障，不依赖 LLM 的"自觉"
- **Skills + MCP** 扩展能力——按需加载，不求大而全
- **SubAgent / Agent Teams** 并行加速——读操作随便并行，写操作必须隔离
- **GHA** 推到生产——自动审查、事件驱动修复、数据飞轮

把这些配好了，Claude Code 就不只是编码助手，而是团队工程系统里一个可审计、可自我改进的核心组件。
