---
name: reference-first-coding
description: 先查参考再写代码——在实现不熟悉的 API、库、框架、算法或协议前，查阅官方文档、GitHub 真实代码示例、社区最佳实践的标准化工作流。适用于所有编程语言和领域。
license: MIT
metadata:
  tags: research, documentation, code-example, api, workflow
---

# Reference-First Coding

在实现任何不熟悉的 API、库、框架、算法或协议前，**必须先查阅权威参考**，不得凭空猜测 API 签名或行为。

## 1. 为什么需要

LLM 最大的固有缺陷之一：**对训练数据之外的 API 会自信地编造**（hallucinate）。具体表现为：

- 凭空捏造不存在的函数签名或参数
- 使用已废弃/删除的 API
- 忽略版本间的 breaking changes
- 遗漏必要的 import/初始化步骤
- 给出理论上可行但实际不兼容的代码

先查参考再写代码，是消除这四类错误的最有效手段。

## 2. 触发条件

**必须加载此 Skill 的场景**（任一匹配即触发）：

- 当前任务涉及 **未在对话中确认过的第三方库/框架** 的 API 调用
- 需要实现 **未写过或不确定细节** 的算法、协议、数据格式
- 使用 **新版本** 的已知库（如 next.js 14 → 15，有 breaking changes）
- 调试 **原因不明的运行时错误**，怀疑是 API 用法错误
- 任务描述中提及的库/框架，其 API 设计你不是 100% 确定

**不需要加载此 Skill 的场景**：
- 使用内置语言标准库（Python `os`、`list`、Go 标准库等——这些在训练数据中足够覆盖）
- 纯逻辑实现（不依赖外部 API）
- 已经在前序对话中确认过的 API

## 3. 核心工作流

### 3.1 第一步：拆解未知点

在动手写代码前，列出所有**不确定的 API 调用或依赖**。

**正确**：
```
任务：用 NextAuth.js 实现 GitHub OAuth 登录
未知点：
- NextAuth.js 的 providers 配置格式
- GitHub OAuth 回调 URL 设置
- Session 回调的 token 参数类型
```

**错误**：
```
// 直接开始写代码，遇到问题再查
import { NextAuth } from 'next-auth/next'  // 这个路径对吗？
```

### 3.2 第二步：查官方文档（首选）

使用 `context7` 工具链获取最新官方文档：

```
1. context7_resolve_library_id(query="NextAuth.js", libraryName="next-auth")
   → 获取 /nextauthjs/next-auth

2. context7_query_docs(libraryId="/nextauthjs/next-auth", query="GitHub provider configuration options")
   → 返回最新的 API 签名、参数、示例代码
```

**规则**：
- **优先查官方文档**，其次才是第三方博客/教程
- 如果用户指定了版本号，必须在 `resolve_library_id` 中找对应版本
- `query` 参数要具体（"GitHub provider options with scope" 而非 "how to use"）
- 一次查一个具体问题，不要一次性问"怎么用这个库"

### 3.3 第三步：搜真实代码示例

使用 `grep_app_searchGitHub` 搜索生产环境中的真实用法：

```
grep_app_searchGitHub(
  query="NextAuth({",         // 真实代码中的 API 调用模式
  language=["TypeScript", "TSX"],
  matchCase=true
)
```

**搜索策略**：

| 目标 | 搜索关键词 | 语言过滤 |
|:----|:----------|:--------|
| 库的 import 方式 | `from 'next-auth'` 或 `require('next-auth')` | TypeScript, JavaScript |
| API 调用签名 | 函数名 + `({` 如 `NextAuth({` | 根据项目语言 |
| 配置模式 | 配置对象的关键字段，如 `providers: [` | 根据项目语言 |
| 初始化代码 | 框架的入口文件模式 | 根据框架 |

**规则**：
- 搜 API **调用方式**而非搜关键词：`prisma.user.create({` 而非 `Prisma create user`
- 搜 3~5 个结果即可，目的是确认共性模式而非穷举
- 关注 star 数高的仓库或知名项目
- 对比多个示例的一致性——如果不同示例用法不同，说明有版本差异或其他配置因素

### 3.4 第四步：交叉验证

将以下来源的信息进行交叉比对：

1. **官方文档**（context7）→ 签名和参数定义
2. **GitHub 示例** → 实际调用方式
3. **websearch** → 社区最佳实践和常见坑

**核对清单**：
- [ ] 官方文档的签名和 GitHub 示例中的调用方式一致吗？
- [ ] 是否有版本差异？（区分文档的版本号和示例项目的版本号）
- [ ] 是否有 import 路径的差异？
- [ ] 是否有被标记为 deprecated 的 API？

如果发现矛盾：
1. 优先相信官方文档的最新版本
2. 其次是近期（6个月内）的高 star GitHub 项目
3. 最后才是博客/教程

### 3.5 第五步：深度调研（可选）

对于特别复杂或不熟悉的技术栈，可派遣 `librarian` agent 进行独立调研：

```
task(
  subagent_type="librarian",
  load_skills=[],
  run_in_background=true,
  prompt="调研 [库名] 中 [功能X] 的实现方式，重点查：[1] 官方 API 签名 [2] GitHub 上的典型使用模式 [3] 常见坑和注意点"
)
```

当 `librarian` 完成后，合并其发现再开始编程。

## 4. 领域特定策略

### 4.1 前端（React / Next.js / Vue）

| 技术栈 | 文档来源 | 代码示例搜索技巧 |
|:-------|:--------|:---------------|
| React / Next.js | context7: `/vercel/next.js` | 搜 `use client` + API 名 |
| Vue / Nuxt | context7: `/vuejs/core` | 搜 `<script setup>` + API 名 |
| CSS / Tailwind | context7 + websearch | 搜 `className=` + 样式模式 |
| Remotion | `remotion-best-practices` Skill | 搜 `useCurrentFrame()` 等 Hook |

### 4.2 后端（Node.js / Python / Go）

| 技术栈 | 文档来源 | 代码示例搜索技巧 |
|:-------|:--------|:---------------|
| Express / Fastify | context7 + websearch | 搜 `app.get(`、`app.post(` |
| Prisma / TypeORM | context7: `/prisma/prisma` | 搜 `prisma.model.create({` |
| FastAPI / Flask | context7 | 搜 `@app.get(`、`@router.post(` |
| Gin / Echo (Go) | context7 + websearch | 搜 `r.GET(`、`e.POST(` |

### 4.3 嵌入式 / C / Rust

| 技术栈 | 文档来源 | 代码示例搜索技巧 |
|:-------|:--------|:---------------|
| ESP-IDF | 官方文档 + GitHub | 搜 `esp_` 函数前缀 |
| STM32 HAL | 芯片 datasheet + GitHub | 搜 `HAL_` 函数前缀 |
| Rust crates | context7 + crates.io | 搜 `use crate::` 模式 |

对于嵌入式，优先查芯片厂商的 **官方 datasheet / reference manual**，其次是 SDK 内的示例代码，最后才是第三方代码。

## 5. 禁止事项

- **禁止** 在不查官方文档的情况下直接写不熟悉的 API 调用代码
- **禁止** 只依赖训练数据中的旧版本 API 知识（库可能已有大版本更新）
- **禁止** 在 context7/GitHub 搜索结果为空时自行编造 API——换一种搜索词或换一个工具
- **禁止** 只查博客/教程而不验证官方文档（博客可能已过时）
- **禁止** 使用 `require` 和 `import` 混用（除非库明确支持，需在文档中确认）
- **禁止** 在同一个 `query_docs` 调用中塞多个问题——一次只查一个具体问题

## 6. 快速参考

| 步骤 | 工具 | 目标 | 耗时参考 |
|:----|:----|:----|:--------|
| ① 拆解未知点 | 脑内 / 笔记 | 列出所有不确定的 API | 30s |
| ② 查官方文档 | `context7_resolve_library_id` + `context7_query_docs` | 确认签名和参数 | 1~3min |
| ③ 搜代码示例 | `grep_app_searchGitHub` | 看真实调用方式 | 1~2min |
| ④ 交叉验证 | 对比 ②③ + `websearch` | 确认一致性/差异 | 1min |
| ⑤ 深度调研 | `librarian` agent | 复杂场景的独立调研 | 3~5min |

## 7. 自检要求

在输出代码前，逐项自检：

- [ ] 任务中涉及的所有第三方 API，是否都已查过官方文档或代码示例？
- [ ] 是否存在直接凭记忆写出的 API 调用，而未经任何来源验证？
- [ ] 上下文7/GitHub 搜索结果是否为空？如果为空，是否尝试了其他搜索策略？
- [ ] 使用的 API 签名是否和最新文档一致？
- [ ] 是否确认了 import/require 路径与库的实际导出一致？
- [ ] 对于有版本差异的库，是否确认了当前使用的版本对应的 API？

如有违反，回退到第 3 节重新执行对应步骤。
