---
name: git-commit-guidelines
description: Git 提交与推送规范——提交(commit)、修改提交信息(amend)、推送(push)、上传代码时的拆分策略、Conventional Commits 提交格式、分支选择、分步提交流程。防止破坏性修改。
license: MIT
metadata:
  tags: git, version-control, commit, workflow, conventional-commits
---

# Git 提交规范

在进行代码修改和版本提交时，必须严格遵循以下准则，确保提交历史清晰、可追溯、易于回滚。

**前置条件**：执行任何 git 写操作（commit / amend / push / tag）前，必须先完整阅读本规范并逐条对照。

## 1. 提交拆分原则

将所有修改拆分成有意义的逻辑批次，每个批次只实现一个明确的修改意图。

不允许在一个批次中混杂多个无关的修改意图（例如不要把"修复 bug"和"调整格式"放在一起），否则必须拆成两个批次，除非这些修改发生在同一行代码中，这种情形可以合并为一个批次。

## 2. 提交标题格式

每个 commit 只允许**一行标题**，格式为：

```
type(scope): description
```

禁止添加 body（正文、详细说明、列表）。

**正确**：

```text
feat(auth): 添加登录接口
```

**错误**：

```text
feat(auth): 添加登录接口

- 新增 refresh token 轮换
- 补充错误码映射
```
（多行 body 违反单行规则，禁止）

其中：
- **type**：修改类型，必须从下列清单中选择：`feat`、`fix`、`refactor`、`docs`、`style`、`test`、`chore`、`perf`、`ci`。
- **scope**：影响范围，用简短英文单词说明受影响的模块、文件或组件名（若无法明确，可省略，但括号必须保留）。
- **description**：用一句简明中文概括本批次的修改内容（一句话总结，不加句号）。

如果某个修改是删除文件，输出 `fix(scope): remove file_name` 即可。

### 2.1 修改提交信息（amend）

`git commit --amend` 同样遵守单行标题规则。

- 仅允许对**本地未推送**的 commit 使用 amend；已推送的禁止改写历史
- amend 修改 message 时，标题格式与第 2 节一致，只写一行，禁止 body
- amend 并入新改动时，确认新改动与 commit 意图一致；意图不同则拆为独立 commit

## 3. 输出规则

多个批次按逻辑顺序依次输出，每行一个标题，批次之间用空行分隔。

整个回复只能包含这些标题行，不要包含任何代码块、解释、总结或其他文字。

示例输出格式（仅用于说明格式，真实回复中不应出现示例）：

```text
feat(auth): 添加登录接口
fix(parser): 处理空输入报错
chore(deps): 更新 lodash 版本
```

## 4. 注意事项

- 优先使用中文提交。
- 不要破坏文件。
- 不要偷懒跳过必要步骤。
- 不要使用 patch 方式修改文件。

## 5. 分步提交流程

当多个逻辑独立的改动需要拆分为多个 commit 时，按以下流程执行：

### 5.1 规划阶段

先整体规划各 commit 分别涉及哪些文件和哪些改动。

### 5.2 跨文件场景

跨文件 → 逐文件 `git add <file>` + `git commit`。

### 5.3 同文件交织场景

同文件交织 → `git add -p <file>` 交互选取 hunk，选完一组立即 commit。

### 5.4 无交互终端场景

如果 `git add -p` 因无交互终端不可用 → 在动手改文件之前先分好组，改一组提交一组，不要改完了再拆。

### 5.5 禁止行为

- 禁止把改动全部揉成一个大 commit 再回头拆。
- 禁止用管道给 `git add -p` 喂脚本。

## 6. 推送规范

提交推送到远端时，必须遵循以下规则：

- **用户未指定分支时，始终推送到 `master` 分支**。不得自行创建其他分支名（如 `main`、`develop` 等）。
- 推送命令统一为 `git push origin master`（或等效的 `git push`，前提是本地已跟踪 `origin/master`）。
- 用户明确指定了其他分支名时，以用户指定为准。

### 6.1 远程协议

GitHub 推送优先使用 SSH，避免 HTTPS 环境下的连接不稳定：

```bash
# 检查当前远程协议
git remote -v

# 若为 HTTPS → 切为 SSH
git remote set-url origin git@github.com:<user>/<repo>.git
```

- MUST 优先使用 SSH 远程（`git@github.com:...`）
- HTTPS 仅作为 SSH 不可用时的备用方案

### 6.2 Tag 管理

`npm version` 或 `git tag` 创建的 tag 是**静态锚点**——它固定在创建时的 commit 上，不会自动跟随后续 `git commit` 向前移动。当 tag 创建后又追加了修复性提交（如 CI 配置修正、lockfile 重建等），重推 tag 前必须执行两步验证：

```bash
# 1. 确认 tag 是否指向最新 commit
git log --oneline -1 HEAD
git log --oneline -1 <tag>

# 2. 若不一致 → 将 tag 移动到 HEAD，再推送
git tag -f <tag>
git push origin :<tag>      # 删除远端旧 tag
git push origin <tag>       # 推送新 tag
```

- MUST 在重推 tag 前先运行 `git log --oneline <tag>` 比对 HEAD，确认 tag 指向最新 commit
- MUST 用 `git tag -f` 更新本地 tag，再删远端、重推，确保远端 tag 与本地一致
- 禁止直接 `git push origin :<tag> && git push origin <tag>` 而不先检查 tag 指向的 commit

## 7. 自检要求

输出前逐项核查：

- [ ] 执行 git 写操作前是否完整阅读了本规范？
- [ ] 标题格式为 `type(scope): description`，一句简明中文、不加句号？
- [ ] commit 信息只包含一行标题，无 body？
- [ ] 每个 commit 只实现一个明确修改意图？
- [ ] amend 仅用于本地未推送 commit，且标题符合单行规则？
- [ ] 推送目标分支为用户指定分支（未指定则为 master）？
