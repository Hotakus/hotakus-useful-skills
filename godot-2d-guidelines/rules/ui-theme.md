# UI Theme System

## Resource 模式

用 `class_name extends Resource` 创建 `.tres` 主题文件，双击在 Inspector 中编辑：

```gdscript
class_name UITheme
extends Resource

@export_group("Button", "btn_")
@export var btn_bg: Color
@export var btn_texture_normal: Texture2D    # 留空用纯色
@export var btn_margin_left: int = 4         # 九宫格
```

### 控件分组原则

每个控件分组包含：颜色（纯色方案）→ 纹理（可选，覆盖纯色）→ 九宫格裁切参数（四边独立）。

### 场景引用

```gdscript
var _theme: UITheme = preload("res://resources/ui_theme.tres")
func _ready() -> void:
    theme = _theme.build()
```

## StyleBox — 纯色 vs 纹理

`build()` 中判断：有纹理 → `StyleBoxTexture`（支持九宫格），无纹理 → `StyleBoxFlat`（程序绘制矩形）：

```gdscript
func _stylebox(tex: Texture2D, bg: Color, border: Color, ml, mr, mt, mb: int) -> StyleBox:
    if tex:
        var sb := StyleBoxTexture.new()
        sb.texture = tex
        sb.texture_margin_left = ml
        # ... 九宫格四边
        return sb
    var sb := StyleBoxFlat.new()
    sb.bg_color = bg
    sb.border_color = border
    sb.anti_aliasing = false  # 像素风格
    return sb
```

### 九宫格裁切

按控件分组独立设置（按钮和面板的边框宽度通常不同）：

```
素材 64×32，拉伸到 200×40（裁切 4px）：
┌──┬──────────┬──┐
│角│  边拉伸   │角│  ← 四角不变形
└──┴──────────┴──┘
```

## CanvasLayer — 分辨率分离

像素游戏 UI 应渲染在窗口分辨率下，避免跟随游戏低分辨率缩放而模糊：

```ini
# 场景中 CanvasLayer 节点
follow_viewport_enabled = false   # UI 用窗口分辨率（如 1280×720）
```

游戏用低分辨率 viewport（如 640×360），UI 用窗口分辨率 — 互不干扰。

## 锚点布局

去掉 CenterContainer，改用锚点 Panel：
- Panel `anchors_preset = 8`（居中），用户可在编辑器拖拽定位
- 内部用 VBoxContainer + MarginContainer 自动排列
- 换分辨率时锚点百分比自动适配位置
