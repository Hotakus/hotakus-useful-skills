# Window Management

## 无边框全屏

MUST NOT 用 `Window.MODE_MAXIMIZED` — 它会接管窗口尺寸，阻止 viewport 拉伸系统重新计算缩放比率。

正确做法：

```gdscript
func _apply_borderless_fullscreen() -> void:
    var window := get_window()
    var idx: int = window.current_screen
    window.mode = Window.MODE_WINDOWED
    window.position = DisplayServer.screen_get_position(idx)
    window.size = DisplayServer.screen_get_size(idx)
    window.borderless = true
```

## 多显示器

MUST 用 `window.current_screen` 获取当前窗口所在屏幕：

```gdscript
var idx: int = window.current_screen
var pos: Vector2i = DisplayServer.screen_get_position(idx)   # 该屏左上角
var size: Vector2i = DisplayServer.screen_get_size(idx)      # 该屏尺寸
```

FORBIDDEN: 无参 `DisplayServer.screen_get_position/size()` — 默认返回主屏信息，窗口在副屏时会跳到主屏。

窗口居中同理：

```gdscript
func _center_window() -> void:
    var idx: int = get_window().current_screen
    var sp: Vector2i = DisplayServer.screen_get_position(idx)
    var ss: Vector2i = DisplayServer.screen_get_size(idx)
    var deco: Vector2i = DisplayServer.window_get_size_with_decorations()
    DisplayServer.window_set_position(sp + (ss - deco) / 2)
```

## 编辑器嵌入模式

编辑器默认嵌入游戏窗口，这会限制窗口操作（报 "Embedded window can't be moved" 等错误）。

测试窗口功能时需切到独立模式：

**永久设置**：`Editor Settings → Run → Window Placement → Game Embed Mode → Disable Embed`

**临时设置**：Game 标签 → 右上角 ⋮ → 取消 "Embed Game on Next Play"

## 分辨率约束

窗口模式下分辨率下拉正常使用。全屏时分辨率下拉应禁用（`resolution_option.disabled = true`），因为窗口已铺满屏幕、分辨率选择无意义。

切换到全屏时自动保存当前窗口尺寸，切回窗口模式时恢复。
