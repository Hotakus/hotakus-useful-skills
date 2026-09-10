---
name: git-pr-issue-guidelines
description: PR 与 Issue 协作规范——创建 PR、编写 PR 描述、更新与推送 PR、合并 PR、创建 Issue 时的标题格式、描述风格、流程约束与环境调研准则。防止描述冗长、推送时机错误、依赖版本与运行环境不匹配。
license: MIT
metadata:
  tags: git, pr, issue, github, workflow
---

# Git PR 与 Issue 规范

规范创建 PR（Pull Request）与 Issue 时的标题、描述与协作流程。不覆盖 commit/push/tag 本身（见 `git-commit-guidelines`），聚焦 PR/issue 层的公开面输出。

## 1. 为什么需要

PR 与 Issue 是协作的公开面，审阅者和协作者只通过它了解你的工作。描述冗长会淹没核心信息；流程时机错误（未经确认的推送、合并）会打断协作节奏；依赖版本与环境不匹配会导致 CI 失败、运行时行为异常。本规范让 PR/issue 的输出与操作流程可预测、可审阅。

## 2. PR 创建

### 2.1 标题格式

标题复用 Conventional Commits 格式：`type(scope): description`，与分支承载的功能对应，不加句号。

**正确**：

```text
feat(auth): 支持 refresh token 轮换
fix(parser): 处理空输入报错
```

**错误**：

```text
add some stuff
更新一下
```

### 2.2 描述风格——只写功能，不写实现细节

描述只写**功能、用户价值与验证方式**。不写实现细节（内部函数名、校验逻辑、状态机流转、数据结构、依赖降级链）。实现细节属于代码与 commit message，不属于 PR 描述。

**正确**：

```markdown
## 功能

1. 新增「取消」按钮，点击后真实终止运行中的子任务，条目显示「已取消」状态
2. 会话 ID 支持一键复制到系统剪贴板

## 验证

- `tsc --noEmit` 通过
- `npm run build` 通过
```

**错误**：

```markdown
## 功能

1. 新增 cancelEntry() 函数，包含 6 层校验：sessionId 存在 → parentID 检查 →
   isDescendantOf 回溯 → status busy → session.abort → catch 兜底
2. settleOnIdle() 决策函数插入 tryMatchAndUpdate / setEntryMap 循环 /
   兜底匹配 / scan reconcile 等 4 个结算点……

## 验证

- IIFE 捕获 entry.cost 关闭竞态窗口……
```

### 2.3 验证方式必填

PR 描述末尾必须列出已验证项（类型检查、构建、测试）。未验证的项如实标注"未验证"，不得省略或夸大。

### 2.4 描述传参方式

描述包含空格、引号、反引号或括号等特殊字符时，使用 `--body-file`（描述写入临时文件后传入），禁止使用 `--body` 直接传内联字符串。

理由：Shell（尤其是 Windows PowerShell）会解析 `--body` 值中的引号与空格，导致参数被拆分、命令失败；`--body-file` 将描述内容与命令参数完全隔离，无转义风险。临时文件用后立即删除。创建 Issue 时同理。

**正确**：

```bash
# 描述写入临时文件后传入，内容与命令参数隔离，无转义问题
gh pr create --title "feat(auth): 新增登录接口" --body-file "$TMP/pr-body.md"
```

**错误**：

```bash
# 描述内含双引号 → 参数被 Shell 拆分，命令失败
gh pr create --body "描述含 \"引号\" 导致失败"
```

### 2.5 外部引用链接格式

PR/Issue 描述引用**其他仓库**的 issue/PR 时，使用 `https://redirect.github.com/<owner>/<repo>/issues/<number>` 格式，禁止使用 `https://github.com/...` 直链。

理由：`github.com` 直链会被 GitHub 自动生成 backlink（cross-reference），在被引用方的 issue/PR 时间线显示「mentioned this issue in ...」——已产生的 backlink **无法通过编辑源内容移除**（只能删除源资源或联系 support）。`redirect.github.com` 链接可正常点击跳转，但不产生 backlink（官方文档支持的方式，dependabot/Renovate 亦用此法）。同仓库引用（`#number`）不适用此规则。

**正确**：

```markdown
参考官方 issue [37276](https://redirect.github.com/example/opencode/issues/37276)
```

**错误**：

```markdown
# github.com 直链 → 产生 backlink，且无法撤回
参考官方 issue [37276](https://github.com/example/opencode/issues/37276)
```

### 2.6 中文内容编码安全（Windows PowerShell）

PR/Issue 描述含中文时，**禁止用 PowerShell `Set-Content` / `Out-File` 写 body 文件**：PowerShell 从 `gh` 捕获输出时按系统代码页（GBK）解码 UTF-8，写出的文件是损坏内容，`gh pr edit --body-file` 会把乱码提交到远端，且 **GitHub 无法撤回 body**（污染后只能靠对话历史/本地备份恢复）。

**正确**：

- 用编辑器的 write 工具（UTF-8 无 BOM）写 body 文件
- 或 node 脚本直写字节：`execSync('gh pr view N --json body --jq .body', {encoding:'utf8'})` + `writeFileSync`

**校验**：远端内容用 ASCII 安全指标验证（`--jq '.body | length'`、`--jq '.body | contains("ASCII关键词")'`）；**中文关键词经 PowerShell 管道会损坏导致误判**（contains 返回 false 不代表内容缺失）。

**覆盖式写操作前**：先确认源内容完整可恢复（GitHub 无 body 历史），再执行 `gh pr edit`。

## 3. PR 更新与推送

### 3.1 推送前询问

执行 `git push`（含更新 PR 分支）前，必须列出涉及的文件、commit 列表与预期结果，等待用户明确批准。推送不是"日常操作"，每次都是需要确认的写操作。

### 3.2 描述同步更新

PR 新增 commit 后，若描述未覆盖新功能或与现状不一致，更新描述；更新描述同样是写操作，先展示修改内容再执行。

### 3.3 实现完成后等待本地测试

新功能实现并创建 PR 后，停在实现分支等待用户的本地测试结果，不得自行切换分支、合并或推进下一步。分支切换本身也须经用户明确确认。用户 WIP 的还原、工作区收尾等动作，应在确认测试完成后再做。

## 4. PR 合并

### 4.1 合并前置条件

合并前必须确认全部满足：CI 检查全绿、无冲突（mergeable 为 MERGEABLE）、PR 描述与内容一致。任一不满足，先解决再合。

### 4.2 合并方式由用户决定

合并方式（`--merge` / `--squash` / `--rebase`）由用户决定，执行前先询问并说明差异：

| 方式 | 效果 | 适用场景 |
|------|------|---------|
| `--merge` | 保留全部提交 + 合并提交 | 仓库历史使用 merge commit、需要保留每步演进 |
| `--squash` | 压缩为单个提交 | 功能开发期提交零散，只关心合并结果 |
| `--rebase` | 线性重放提交，无合并提交 | 追求线性历史、feature 分支短小 |

## 5. Issue 创建

### 5.1 标题

标题格式：`type: 简短问题描述`（type 用 `bug` / `feat` / `question` / `docs` 等），一句话说清问题。

### 5.2 内容结构

Issue 必须包含：复现步骤（编号列表）、期望行为与实际行为、环境信息（OS、运行时版本、npm/node 版本、相关依赖版本）、相关日志或错误输出。环境信息缺失会导致问题无法复现。

**正确**：

```markdown
## 复现步骤
1. 安装依赖后运行 `npm run build`
2. 传入含中文的输入

## 期望行为
构建成功

## 实际行为
报错 `Cannot find module`

## 环境
- OS: Windows 11
- node: v22.18.0
- npm: 11.17.0
```

## 6. 环境与依赖调研

### 6.1 修改依赖前先调研运行环境

修改或新增依赖（尤其是版本声明）前，必须先调研目标项目的运行环境：CI 的 node 版本矩阵（`.github/workflows`）、`engines` 字段、`.nvmrc`。依赖版本必须匹配实际运行环境，不得照抄来源项目的版本声明。

**正确**：CI 使用 node 20/22 → `@types/node` 声明为 `^22`。

**错误**：照抄上游项目的 `@types/node ^26`，未检查本项目 CI 的 node 版本。

### 6.2 构建验证

依赖变更后必须完整验证：类型检查、构建、实际执行一次关键命令。仅凭安装成功不构成验证证据。

## 7. 快速参考

| 场景 | 规则 |
|------|------|
| PR 标题 | `type(scope): description`，不加句号 |
| PR 描述 | 只写功能与验证，不写实现细节 |
| 推送 / 合并 / 更新描述 | 先展示方案，等待批准 |
| 合并方式 | 询问用户，说明 merge/squash/rebase 差异 |
| Issue 复现 | 编号步骤 + 期望/实际行为 + 环境信息 |
| 依赖版本 | 先查 CI 环境矩阵，匹配实际 node 版本 |

## 8. 禁止事项

- 禁止在 PR 描述中写实现细节（函数名、内部逻辑、依赖降级链）
- 禁止未经确认执行 `git push`、PR 合并、描述更新
- 禁止在用户本地测试前自行切换分支、合并或推进下一步
- 禁止在 CI 未全绿、存在冲突时合并 PR
- 禁止替用户决定合并方式
- 禁止照抄来源项目的依赖版本而不调研本地环境
- 禁止省略 Issue 的环境信息与复现步骤
- 禁止在 PR 描述中遗漏验证项或夸大验证结果
- 禁止用 PowerShell `Set-Content` / `Out-File` 写含中文的 PR/Issue body 文件（乱码污染远端，无法撤回）

## 9. 自检要求

输出前请自检，如出现违反以上规则的输出，重新输出：

- [ ] PR/Issue 标题是否符合 `type(scope): description` 或 `type: 描述`？
- [ ] PR 描述是否只写了功能与验证，不含实现细节？
- [ ] 验证项是否真实可复现，无夸大？
- [ ] 描述含特殊字符（空格、引号、反引号）时是否使用了 `--body-file` 而非 `--body`？
- [ ] 引用其他仓库 issue/PR 时是否使用了 `redirect.github.com` 而非 `github.com` 直链？
- [ ] 推送、合并、描述更新前是否已展示方案并等待批准？
- [ ] 合并方式是否由用户决定？
- [ ] Issue 是否包含复现步骤、期望/实际行为、环境信息？
- [ ] 依赖版本是否已对照 CI/engines 环境核实？
- [ ] 含中文的 body 文件是否用 write 工具（或 node 脚本）写入，而非 PowerShell `Set-Content` / `Out-File`？
- [ ] 实现完成后是否停在实现分支等待用户本地测试，未自行切换分支或合并？
