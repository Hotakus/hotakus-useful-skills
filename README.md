<div align="center">
<strong>
    <h1>Hotakus Useful Skills</h1>
    Hotakus 的实用 AI SKILLs —— 编码规范、游戏开发、嵌入式测试、文档撰写等领域的最佳实践
</strong>

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Skills](https://img.shields.io/badge/skills-10-orange.svg)]()
[![LLM](https://img.shields.io/badge/LLM-Ready-green.svg)]()
</div>

## 技能列表

| 技能 | 说明 |
|------|------|
| **godot-2d-guidelines** | Godot 4.x 2D 游戏开发——场景组织、物理碰撞、Camera2D、AnimationTree、TileMap、像素渲染 |
| **c-coding-guidelines** | C 语言编码规范——命名约定、函数设计、宏定义、static/const 正确性 |
| **code-comment-guidelines** | 代码注释规范——注释如初写，只描述功能与设计理由 |
| **doc-writing-guidelines** | 文档撰写规范——技术文档结构、排版、语言风格 |
| **embedded-testing-guidelines** | 嵌入式测试——分层策略、Mock、CI 集成 |
| **git-commit-guidelines** | Git 提交规范——Conventional Commits、拆分策略、分步流程 |
| **karpathy-guidelines** | LLM 编码防错准则——简单优先、外科手术式修改、可验证目标 |
| **math-formula-guidelines** | 数学公式编写——LaTeX 写法、GitHub 渲染兼容 |
| **reference-first-coding** | 先查参考再写代码——API/库/框架/算法的官方文档查阅、代码示例搜索、交叉验证工作流 |
| **skill-design-guidelines** | Skill 设计指南——领域定义、规则设计、自检清单 |

## 安装

```bash
# 克隆到 Claude Skills 目录
git clone https://github.com/Hotakus/hotakus-useful-skills.git ~/.claude/skills
```

## 结构

```
skills/
├── README.md
├── godot-2d-guidelines/          ← 最完善的技能，含 10 个独立规则文件
│   ├── SKILL.md
│   └── rules/
│       ├── animation.md
│       ├── autoloads.md
│       ├── camera2d.md
│       ├── fonts.md
│       └── ...
└── <其他技能>/
    └── SKILL.md
```

## 使用

在 Claude Code 中，当任务匹配技能描述时自动加载。也可以手动触发：

```
/skill godot-2d-guidelines
```

## 许可

[MIT](LICENSE)
