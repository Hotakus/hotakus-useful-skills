# Camera2D 像素抖动防护

## When to use

当配置 2D 像素游戏相机、排查画面抖动、或 Camera2D 跟随角色时出现 1px 偏移时使用。

## 场景设置

Camera2D `anchor_mode` 设为 `1`（DRAG_CENTER）——相机坐标代表屏幕中心，无需手动 viewport 偏移。

## 玩家脚本：position.round()

```gdscript
# player.gd
func _physics_process(_delta: float) -> void:
    var direction: Vector2 = Input.get_vector("move_left", "move_right", "move_up", "move_down")
    velocity = direction * move_speed
    move_and_slide()
    position = position.round()  # 消除 move_and_slide 浮点残差
```

## 相机跟随：_process() 不是 _physics_process()

```gdscript
# Camera2D 脚本（或父节点脚本）
extends Camera2D

@export var target: Node2D

func _process(_delta: float) -> void:
    if not target:
        return
    global_position = target.global_position  # 已整数，无需 round
```

### 场景树顺序陷阱

Godot 按树顺序执行 `_physics_process()`（父→子）。如果相机和玩家在不同节点且相机节点在玩家上方，`_physics_process` 读到的是玩家**上一帧**位置——产生一致的 1 帧视觉偏移。

用 `_process()`（在所有 physics 之后运行）规避此问题。

### 调度规则

| 条件 | 相机回调 |
|------|----------|
| 相机和玩家在不同节点（默认） | `_process()` |
| 相机脚本直接在玩家节点上 | `_physics_process()` |
| 启用 physics interpolation | `_process()`（读取插值坐标） |

## 关键规则

- MUST 设 `anchor_mode = 1`（DRAG_CENTER）——比手动 viewport 偏移更简单可靠
- 玩家 MUST 在 `move_and_slide()` 后 `position.round()` ——源头消除浮点抖动
- MUST 禁用 `position_smoothing_enabled`（手动设 position 时冲突）
- 玩家速度 MUST 对齐物理 TPS：`move_speed ÷ TPS = integer`（如 120 ÷ 60 = 2）
- Camera2D `Process Callback` 设 `Idle`（默认，配合 `_process()`）
- 避免非整数 drag_margin：`drag_margin × viewport_size` 必须为整数

## 与物理插值联动

```ini
[physics]
common/physics_interpolation=true
common/physics_jitter_fix=0.0
```

启用后：
- 所有移动逻辑 MUST 在 `_physics_process()`
- 相机在 `_process()` 跟随（读取插值坐标）
- 传送后调用 `reset_physics_interpolation()`

代价：约半帧输入延迟。复古像素游戏（需要逐帧精确）不推荐。
