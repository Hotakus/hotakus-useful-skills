---
name: math-formula-guidelines
description: 数学公式编写规范——写数学公式、LaTeX 公式、Markdown 数学表达式时，按目标渲染平台选择正确的定界符与写法，避免 GitHub 与 OpenCode 的渲染兼容性问题。
license: MIT
metadata:
  tags: math, latex, markdown, rendering, opencode
---

# 数学公式编写规范

在 GitHub Markdown、OpenCode TUI（CLI）、OpenCode Desktop/Web 中编写 LaTeX 数学公式时，严格遵循以下规则。

## 1. 先判定渲染目标平台

**输出数学公式前，先确认目标渲染平台，再按下表选择定界符。定界符选错会导致公式原样显示源码。**

| 目标平台 | 行内公式 | 块级公式 | 禁止使用 |
|---------|---------|---------|---------|
| GitHub Markdown | `$...$` | `$$...$$` | — |
| OpenCode TUI / CLI | 不支持 | 围栏块，语言标识 `math` / `latex` / `tex` | `$...$`、`$$...$$` |
| OpenCode Desktop / Web | `\(...\)` | `$$...$$` | 单 `$...$` |

依据：OpenCode v2.0.10（2026-09-19 发布）的 TUI 仅渲染围栏块内的 LaTeX（PR #45339，2026-08-26 合入 v2 分支，Scope 明确排除行内 `$...$` 与 `$$...$$`）。Desktop/Web 使用 KaTeX，仅识别 `\(...\)` 与 `$$...$$`。

## 2. OpenCode 环境规则

**在 OpenCode TUI / CLI 输出数学公式时，必须使用围栏代码块，禁止使用 `$` 定界符。**

**正确**：

````
```math
\int_{-\infty}^{+\infty} e^{-x^2} \, dx = \sqrt{\pi}
```
````

**错误**：

```
$$\int_{-\infty}^{+\infty} e^{-x^2} \, dx = \sqrt{\pi}$$
```

反例在 TUI 中原样显示源码，不做任何渲染。围栏块内直接书写 LaTeX 数学语法，块内不添加 `$` 定界符。

## 3. 公式写法细则

### 3.1 行内公式的空格（GitHub）

**行内公式 `$...$`**：`$` 前后必须与相邻字符（中文、标点）用空格隔开。

- ✅ `电压 $V_m$ 越大`
- ❌ `电压$V_m$越大`

### 3.2 `\text{}` 不用于行内公式（GitHub）

**`\text{}` 在行内公式中不可靠**：把单位移出 `$` 分隔符。

- ✅ `$f = 200$ Hz`
- ❌ `$f = 200\ \text{Hz}$`

### 3.3 块公式中含下划线时用 `\mathrm{}`（GitHub）

**`\text{}` 在 `$$` 块中含下划线时**：改用 `\mathrm{}`，下划线需转义。

- ✅ `\mathrm{sym\_err}`
- ❌ `\text{sym\_err}`

### 3.4 `\begin{aligned}` 内换行

**`\begin{aligned}` 内换行**：只用 `\\`，不附带间距参数。

- ✅ `\\`
- ❌ `\\[4pt]`

### 3.5 运算符推荐写法

**`\operatorname{argmax}`** 优于 `\arg\max`，兼容性更好。

- ✅ `\operatorname{argmax}`
- ❌ `\arg\max`

### 3.6 自检要求

输出前请自检，如出现违反以上规则的输出，重新输出。

- [ ] 是否先确认了目标渲染平台？
- [ ] 在 OpenCode TUI / CLI 中是否使用了围栏块，而非 `$` 定界符？
- [ ] 在 GitHub Markdown 中是否遵守了第 3.1-3.3 节的兼容性规则？
