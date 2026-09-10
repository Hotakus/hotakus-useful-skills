---
name: file-tool-usage-guidelines
description: 文件与 shell 工具使用规范——读取、检索、写入、编辑文件及执行 shell/npm 命令时的工具选择、编码安全与临时文件纪律。避免用 PowerShell 读取/检索文件、编码乱码、误删依赖。
license: MIT
metadata:
  tags: file-ops, tooling, encoding, shell, windows
---

# 文件与 shell 工具使用规范

规范在操作文件（读取 / 检索 / 写入 / 编辑）与执行 shell 命令时的工具选择与纪律。不绑定具体项目，适用于所有含文件与 shell 操作的会话。

## 1. 为什么需要

内置工具与 shell 是两类操作文件的手段。选错工具会引入三类故障：

- **编码损坏**：Windows 控制台按 GBK 解码 UTF-8，用 PowerShell 读 / 写含中文的文件会乱码或写坏文件。
- **检索失真**：shell 转义层使复杂匹配（引号、正则、多条件）易碎，产生 SyntaxError 或静默失败。
- **误操作**：用 shell 直接改文件、删依赖目录，绕过审计且不可撤销。

因此确立一条主轴：**读与检索归内置工具，shell 只用于"执行命令"。**

## 2. 读取与检索：只用内置工具

**读取文件内容一律用 `read`，检索内容一律用 `grep`。**

理由：内置工具按 UTF-8 处理、无 shell 转义层；`read` 原生支持 offset/limit，`grep` 原生支持正则与文件过滤。

| 需求 | 工具 |
|------|------|
| 查看文件内容（全文 / 行号范围 / 片段 / 验证编辑结果） | `read` |
| 确认某内容是否存在 / 位于何处 | `grep` |
| 按文件名找文件 | `glob` |

**错误**：用 PowerShell `Get-Content` 读文件、`Select-String` 检索——控制台按 GBK 解码 UTF-8 导致中文乱码，且转义层易碎。

不得为了"同时做定位行号 + 截断 + 多条件输出"而改走 PowerShell 脚本读取文件。

### 2.1 shell 的适用范围

**`bash` 仅用于无内置专用工具的操作**：git、npm 查询类、进程管理、文件系统结构操作（建目录 / 列目录 / 删除）、执行构建或测试。

命令一旦涉及"读文件内容或检索内容"，就拆分出来交给 `read` / `grep`。

### 2.2 禁止内联脚本做多断言

**禁止用内联 `node -e` / `python -c` 做多断言文本检查。**

理由：跨 shell 引号转义易碎，Windows 下反复出现 SyntaxError 或静默失败。

**正确**：组合断言（多条件 OK/FAIL 汇总）用 `write` 写临时脚本到系统临时目录 → 执行 → 用完删除。

**错误**：把多条件检查塞进一条内联 `node -e "..."`——引号嵌套导致解析失败。

## 3. 写入与编辑：优先外科手术式修改

**写入文件用 `write`（UTF-8 无 BOM）；修改文件用 `edit`（精确替换），不重写整个文件。**

**禁止用 PowerShell 写文件**：`Set-Content` / `Out-File` 从外部命令捕获输出时按系统代码页（GBK）解码 UTF-8，会写坏含中文的文件；且覆盖式写操作无法撤回。

编辑纪律：

- **分步**：修改上下文距离过大时，分步修改，避免一次改太多。
- **查重**：编辑前检查目标范围内是否已有重复项或冲突内容，避免引入重复定义。
- **批量替换三步法**：① 唯一占位符保护例外 → ② `replaceAll=true` 全量替换 → ③ 占位符恢复原文。
- **影响面**：使用 `replaceAll` 前检查影响范围，误伤面积大时改用逐行 `edit`。

## 4. 编码安全：不依赖可能乱码的显示字符

**Windows（GBK 系统代码页）下，PowerShell stdout 的非 BMP 字符（emoji、✅❌⚠️）会损坏为 `?`；`Get-Content` 读 UTF-8 文件会乱码。**

- 命令输出标记用 **ASCII**（OK / FAIL / FOUND / MISSING），不用 emoji。
- 验证 / 匹配逻辑用 **ASCII 模式**（如 `Contains("...")`）或字节级检查，不依赖可能乱码的显示字符。
- 读取含中文文件优先用 `read` / `grep`（UTF-8 假设）；确需 PowerShell 读取时按文件实际编码显式指定（`-Encoding UTF8` 或 `-Encoding Default`）。
- 不主动变更文件编码（如 UTF-8 变 GBK），除非用户允许。

## 5. shell 与 npm

**npm / node 包管理优先用 `npm` 命令；禁止用 PowerShell 直接删除 `node_modules` 或插件缓存目录。**

**禁止在 agent 中执行 npm 环境配置操作**：切换 registry（`npm config set registry`、nrm）、安装全局包（`npm install -g`）、执行 `npm install`（项目依赖由用户侧安装）。

如确需 npm 命令，仅限**查询类**：`npm list`、`npm view`、`npm info` 等。

## 6. 临时文件

**临时文件写入系统临时目录（`$env:TMP` 或 `/tmp`），用完立即删除。**

## 7. 快速参考

| 场景 | 工具 |
|------|------|
| 读文件内容 | `read` |
| 检索内容 / 定位 | `grep` |
| 按文件名查找 | `glob` |
| 写文件 | `write` |
| 改文件 | `edit`（精确替换） |
| git / npm 查询 / 进程 / 目录操作 / 构建测试 | `bash` |
| 组合断言脚本 | `write` 临时脚本 → 执行 → 删除 |

## 8. 禁止事项

- 禁止用 PowerShell `Get-Content` / `Select-String` / `findstr` 读取或检索文件。
- 禁止用 PowerShell `Set-Content` / `Out-File` 写文件。
- 禁止用内联 `node -e` / `python -c` 做多断言检查。
- 禁止用 PowerShell 直接删除 `node_modules` 或插件缓存目录。
- 禁止在 agent 中执行 npm 环境配置（registry 切换、`npm install`、全局安装）。
- 禁止主动变更文件编码格式。
- 禁止在命令输出 / 验证中用 emoji 做标记。

## 9. 自检要求

输出前请自检，如出现违反以上规则的输出，重新输出。

- [ ] 读文件内容是否用了 `read`（而非 PowerShell `Get-Content`）？
- [ ] 检索内容是否用了 `grep`（而非 PowerShell `Select-String`）？
- [ ] 写 / 改文件是否用了 `write` / `edit`（而非 PowerShell）？
- [ ] bash 命令是否只做"执行命令"，未夹带读文件 / 检索？
- [ ] 命令输出标记是否用 ASCII（未用 emoji）？
- [ ] 组合断言是否用临时脚本（未用内联 `node -e`）？
- [ ] npm 命令是否仅限查询类？
- [ ] 临时文件是否写入系统临时目录并已删除？
