# AnimationTree & AnimationPlayer

## Architecture

```
Player
├── Sprite2D (hframes/vframes)
├── AnimationPlayer (存放帧序列动画数据)
└── AnimationTree (状态机/混合树，引用 AnimationPlayer)
```

- `AnimationTree` 不存动画数据，只存状态机逻辑
- `AnimationTree.anim_player` 必须指向 `AnimationPlayer` 节点

## travel() — 核心规则

`travel()` 使用 A* 寻路，**故意不检查 advance_condition**（Godot C++ 源码，`_make_travel_path` 只过滤 DISABLED 过渡）。

```
travel("walk") → A* 寻路 → 只跳过 DISABLED → 无条件直达目标
```

### 两种控制模式互斥

| 模式 | Advance Mode | 代码 |
|------|:--:|------|
| travel 驱动 | `Enabled`（默认） | `state_machine.travel("walk")` |
| 条件驱动 | `Auto` + `advance_condition` | `anim_tree.set("parameters/conditions/moving", true)` |

MUST NOT 混用 — travel() 绕过 Auto 条件。

### advance_condition 定义

在过渡线的 Inspector → Advance Condition 中输入自定义字符串（如 `"moving"`）→ Godot 自动创建 `parameters/conditions/moving` 布尔参数 → 代码控制：

```gdscript
anim_tree.set("parameters/conditions/moving", direction.length() > 0)
```

### 代码模板

```gdscript
@onready var anim_tree: AnimationTree = $AnimationTree
@onready var sm: AnimationNodeStateMachinePlayback = anim_tree.get("parameters/playback")

func _ready() -> void:
    anim_tree.active = true
    sm.start("idle")

func _physics_process(delta: float) -> void:
    if direction.length() > 0:
        sm.travel("walk")
    else:
        sm.travel("idle")
```

## 轨道类型陷阱

`Sprite2D.frame` 是整数属性：

- MUST 用 **Property Track** 创建动画关键帧（离散值，无插值）
- FORBIDDEN 用 bezier 轨道 — AnimationTree 过渡时尝试帧间浮点插值，报 `Type mismatch between initial and final value: bool and float`

修复：删除旧轨道 → 点 + → Property Track → Sprite2D → frame → 重新插关键帧

## 精灵翻转

MUST 只在移动时更新 flip_h：

```gdscript
if direction.length() > 0:
    sprite.flip_h = direction.x < 0
# 静止时不更新，保持最后朝向
```

FORBIDDEN: 无条件 `sprite.flip_h = direction.x < 0`（静止时 direction.x=0 → 强制翻右）
FORBIDDEN: 在 AnimationPlayer 中为 flip_h 创建轨道
