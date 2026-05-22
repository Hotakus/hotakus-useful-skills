---
name: math-formula-guidelines
description: 数学公式编写规范——写数学公式、LaTeX 公式、Markdown 数学表达式时的正确写法，避免 GitHub 渲染兼容性问题。使用 $...$ 和 $$...$$ 定界符。
license: MIT
metadata:
  tags: math, latex, markdown, formula, github-markdown
---

# GitHub Markdown 数学公式编写规范

在 GitHub Markdown 中编写 LaTeX 数学公式时，严格遵循以下规则，确保公式在各平台正确渲染。

## 1. 行内公式的空格

**行内公式 `$...$`**：`$` 前后必须与相邻字符（中文、标点）用空格隔开。

- ✅ `电压 $V_m$ 越大`
- ❌ `电压$V_m$越大`

## 2. `\text{}` 不用于行内公式

**`\text{}` 在行内公式中不可靠**：把单位移出 `$` 分隔符。

- ✅ `$f = 200$ Hz`
- ❌ `$f = 200\ \text{Hz}$`

## 3. 块公式中含下划线时用 `\mathrm{}`

**`\text{}` 在 `$$` 块中含下划线时**：改用 `\mathrm{}`，下划线需转义。

- ✅ `\mathrm{sym\_err}`
- ❌ `\text{sym\_err}`

## 4. `\begin{aligned}` 内换行

**`\begin{aligned}` 内换行**：只用 `\\`，不附带间距参数。

- ✅ `\\`
- ❌ `\\[4pt]`

## 5. 运算符推荐写法

**`\operatorname{argmax}`** 优于 `\arg\max`，兼容性更好。

- ✅ `\operatorname{argmax}`
- ❌ `\arg\max`

## 6. 自检要求

输出前请自检，如出现违反以上规则的输出，重新输出。
