---
title: Cursor 使用技巧：Rules 配置 + Vibe Coding 实战心得
date: 2025-12-24 09:00:00
tags:
  - Cursor
  - AI 工具
  - IDE
categories:
  - 效率工具
---

# Cursor 使用技巧：Rules 配置 + Vibe Coding 实战心得

> 最近把 Cursor 当主力编辑器用了一段时间，感觉最影响效率的不是 AI 模型本身多聪明，而是**你怎么给它上下文**。Rules 系统是 Cursor 最核心的配置能力之一，搞明白 User Rules 和 Project Rules 的区别，写代码的体验直接上一个台阶。

## 先说说什么是 Vibe Coding

"Vibe Coding"是 Andrej Karpathy（前 Tesla AI 总监）在 2025 年初提出的概念，大意就是：你用自然语言描述你想要什么，AI 来写代码，你来"感觉"对不对。

Cursor 就是目前 Vibe Coding 最主流的工具之一。它本质上是一个魔改版的 VS Code，内置了 AI 能力——你可以跟它对话、让它直接改代码、跑命令、甚至自动操作多个文件。

但 AI 不是万能的，你不告诉它项目的技术栈、代码风格、业务上下文，它就只能给你生成"通用但不一定对"的代码。这就是 Rules 的作用——**给 AI 定规矩**。

## Cursor 的三种模式

在讲 Rules 之前，先理解 Cursor 的三种交互模式，因为 Rules 在不同模式下都会生效：

| 模式 | 用途 | 特点 |
|------|------|------|
| **Ask** | 问问题、做规划 | 只读，不改代码。适合让 AI 先帮你想方案 |
| **Agent** | 让 AI 自主编码 | 能跨文件编辑、跑终端命令、自主决策。主力模式 |
| **Manual** | 手动控制 | AI 只在你指定的地方改，不会自己"发挥" |

> 我现在的习惯是：**先用 Ask 模式让 AI 规划方案，确认思路没问题了再切 Agent 模式让它动手**。比一上来就让 Agent 干活靠谱得多，因为 Agent 模式下它有时候会"自由发挥"，改了些你不想改的东西。

还有个 **YOLO 模式**（在 Agent 模式下开启），AI 执行命令不用你逐个确认。适合原型开发、个人练手，**千万别在生产代码里开**。

## Rules 系统详解

Rules 就是给 AI 的"行为准则"，本质上是加在 LLM 上下文开头的指令文本。Cursor 支持两个层级的 Rules：

### 一、User Rules（用户级 / 全局规则）

**位置：** Cursor Settings → Rules → User Rules

**作用范围：** 所有项目，所有对话

**适合放什么：** 跟具体项目无关的通用偏好

直接在设置里的文本框写就行，比如：

```
- 始终使用中文回复
- 回答要简洁，不要废话
- 写代码时加上关键注释，但不要过度注释
- 优先使用 TypeScript 而非 JavaScript
- 不要随意更改我没提到的文件
- 修复 bug 时不要引入新的模式或技术栈
```

> 这些规则会注入到每一次 AI 交互的上下文中，所以别写太长。我一般控制在 10-15 条以内，太多了反而浪费 token，AI 的"注意力"会被分散。

### 二、Project Rules（项目级规则）

**位置：** 项目根目录下的 `.cursor/rules/` 文件夹

**作用范围：** 仅当前项目

**适合放什么：** 技术栈、框架约定、代码风格、业务上下文

这是 Cursor 2025 年推出的新系统，每条规则是一个 `.mdc` 文件（MDC = Markdown Components，一种带 frontmatter 的 Markdown 格式）。

#### .mdc 文件长什么样？

```markdown
---
description: Spring Boot 项目的 Service 层规范
globs: src/main/java/**/service/**/*.java
alwaysApply: false
---

# Service 层编码规范

- 所有 Service 类使用 @Service 注解
- 业务异常统一抛出自定义的 BizException
- 不要在 Service 层直接使用 HttpServletRequest/Response
- 涉及多表操作时使用 @Transactional，并指定 rollbackFor = Exception.class
- 方法命名：查询用 getXxx / listXxx / pageXxx，操作用 createXxx / updateXxx / removeXxx
- 复杂业务逻辑要拆分私有方法，单个方法不超过 50 行
```

frontmatter 里三个字段：
- `description`：规则描述，给 AI 和你自己看的
- `globs`：文件匹配模式（gitignore 风格），决定哪些文件触发这条规则
- `alwaysApply`：是否始终生效

#### 四种规则类型

这是 Rules 系统最核心的概念——通过 frontmatter 组合控制规则的触发方式：

**1. Always（始终生效）**

```markdown
---
description: 项目通用约定
alwaysApply: true
---
```

不管你在干什么，这条规则都会被塞进上下文。适合放项目级别的通用约定，比如技术栈说明、代码风格。

**2. Auto Attached（自动附加）**

```markdown
---
description: React 组件编写规范
globs: src/components/**/*.tsx
alwaysApply: false
---
```

只有当你在对话中引用了匹配 `globs` 模式的文件时，这条规则才会自动加载。比如上面这条，只有你聊到 `src/components/` 下的 `.tsx` 文件时才生效。

> 这个设计挺聪明的——**按需加载**，避免把所有规则一股脑塞给 AI 浪费 token。我现在 `.cursor/rules/` 里有十几个文件，但每次对话只会加载相关的那几条。

**3. Agent Requested（AI 自主判断）**

```markdown
---
description: 数据库迁移脚本编写规范，当需要创建或修改数据库表结构时参考
alwaysApply: false
---
```

没有 `globs`，`alwaysApply` 也是 false。AI 会根据 `description` 的内容判断当前任务是否需要这条规则。比如你让它"给用户表加一个 phone 字段"，它会自己去加载上面这条规则。

关键在于 **description 要写清楚**，不然 AI 不知道什么时候该用它。

**4. Manual（手动引用）**

需要你在聊天框里用 `@规则名` 手动触发：

```
@api-design-guide 帮我设计一个用户注册的 REST API
```

适合那些不常用但偶尔需要的规则，比如某个特定框架的迁移指南。

### 三、旧版 .cursorrules（已废弃，但还能用）

在 `.cursor/rules/` 系统出来之前，Cursor 用的是项目根目录下的 `.cursorrules` 文件，一个大文件塞所有规则。现在官方标记为 deprecated，建议迁移到新系统。

迁移思路很简单：把 `.cursorrules` 里的内容按主题拆分成多个 `.mdc` 文件，放到 `.cursor/rules/` 目录下。

## 实战：我的 Rules 目录结构

```
.cursor/
  rules/
    project-overview.mdc      # Always - 项目介绍、技术栈
    coding-style.mdc          # Always - 通用代码风格
    react-components.mdc      # Auto - 匹配 *.tsx
    api-routes.mdc            # Auto - 匹配 src/api/**
    database-migration.mdc    # Agent Requested - DB 相关
    testing-guide.mdc         # Agent Requested - 测试相关
    deployment.mdc            # Manual - 部署相关
```

`project-overview.mdc` 的内容示例：

```markdown
---
description: 项目概述和技术栈
alwaysApply: true
---

# 项目概述

这是一个电商后台管理系统。

## 技术栈
- 后端：Spring Boot 3.2 + MyBatis-Plus + MySQL 8.0 + Redis
- 前端：React 18 + TypeScript + Ant Design 5
- 构建：Maven（后端）、Vite（前端）
- 部署：Docker + Nginx

## 项目结构
- `/src/main/java/com/example/admin/` - 后端代码
- `/frontend/src/` - 前端代码
- `/sql/` - 数据库脚本

## 约定
- REST API 统一前缀 /api/v1
- 响应格式统一用 Result<T> 包装
- 分页查询统一用 PageQuery 和 PageResult
```

## Vibe Coding 实用技巧

除了 Rules，再记录几个用 Cursor 过程中摸索出来的技巧：

### 1. 先规划再执行

```
   直接一句话让 Agent 干：
   "给这个系统加上权限管理功能"

   分两步：
   Ask 模式："我想给系统加权限管理，请帮我规划一下方案，包括表设计、接口设计、前端改动"
   → 确认方案
   Agent 模式："按照上面的方案，先创建数据库表和实体类"
```

> 一步一步来比一口吃个胖子靠谱太多了。AI 一次改太多文件的时候很容易顾此失彼。

### 2. 善用 @docs 索引文档

Cursor Settings → Features → Docs 里可以添加文档链接，AI 会索引这些文档内容。

比如你用了 某个模块 新版，但 AI 的训练数据可能只到旧版，直接加个文档链接就能让它"学会"新版 API。框架有大版本升级的时候特别有用。

### 3. 频繁用 Checkpoint

Cursor 有一个"恢复到检查点"的功能（在 Agent 对话里），AI 每次改动都会生成一个快照。发现改错了直接回滚，比 Ctrl+Z 可靠得多。

配合 Git 使用更好——**每完成一个小功能就 commit 一次**。AI 写代码速度快，翻车速度也快，有 commit 兜底心里踏实。

### 4. 约束 AI 的"发挥空间"

在 User Rules 里加上这几条能避免很多问题：

```
- 只修改我提到的文件，不要动其他文件
- 修复 bug 时优先在现有方案上修复，不要引入新框架或新模式
- 如果你不确定某个改动是否必要，先问我
- 不要删除已有的注释和文档
```

（不加这些的话，你让它修个 bug，它可能顺手把你的目录结构重构了...）

### 5. 模型选择

不同任务选不同的模型：

| 任务类型 | 推荐模型 | 原因 |
|---------|---------|------|
| 方案规划、架构设计 | Claude Opus / o3 | 推理能力强 |
| 日常编码、功能开发 | Claude Sonnet / Gemini 2.5 Pro | 速度和质量平衡 |
| 简单补全、重复性修改 | 默认模型就行 | 省 token |

### 6. 新对话 = 新开始

一个对话聊太久，上下文会越来越乱。完成一个任务后开个新对话，AI 的表现会好很多。别舍不得——旧对话的历史不会丢，随时可以回去看。

## 踩坑记录

**坑 1：Rules 不生效**

写了规则但 AI 好像没读。排查后发现是 `globs` 写错了——用的是 `*.tsx` 而不是 `**/*.tsx`，只匹配了根目录下的文件。glob 模式里 `**` 表示任意层级的子目录，别忘了。

**坑 2：Agent 模式改了不该改的文件**

让它修一个组件的样式，结果它把全局的 CSS 变量也改了，导致其他页面样式全崩了。后来在 User Rules 里加了"只修改我提到的文件"这条，好了很多。

**坑 3：旧 .cursorrules 和新 .cursor/rules/ 冲突**

两个都有的时候，行为不太可预测。建议彻底迁移到新系统，删掉旧的 `.cursorrules` 文件。

## 社区资源

找 Rules 模板不用自己从头写：

- **cursor.directory** — 社区维护的规则集合，按技术栈分类，直接复制
- **awesome-cursorrules**（GitHub） — PatrickJS 维护的规则模板仓库
- **dotcursorrules.com** — 另一个规则分享网站

> 我的做法是先从社区找一个跟自己技术栈接近的模板，然后根据项目实际情况改。比自己从零开始写效率高很多。

## 小结

Cursor 的 Rules 系统其实就两层：User Rules 管"你是谁"（全局偏好），Project Rules 管"这个项目怎么写"（项目约定）。四种规则类型（Always / Auto / Agent Requested / Manual）控制加载时机，核心是**按需给 AI 提供上下文，而不是一股脑全塞进去**。把 Rules 配好了，AI 写出来的代码才是"懂你项目"的代码。
