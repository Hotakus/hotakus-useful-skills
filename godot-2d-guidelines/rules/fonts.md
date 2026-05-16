# Pixel Font Rendering

## 核心规则

MUST `font_size == 字体原生设计尺寸`。像素字体每个字形按固定像素网格设计，缩放即模糊——即使关了抗锯齿也一样。

```
Fusion Pixel 12px  → font_size = 12  ✅ 锐利
Fusion Pixel 12px  → font_size = 16  ❌ 模糊（非整数倍缩放）
```

## 全局设置（project.godot）

```ini
[gui]
theme/default_font_antialiased=false
theme/default_font_multichannel_signed_distance_field=false
theme/default_font_oversampling=1.0
theme/default_font_subpixel_positioning=0
theme/default_font_hinting=0
```

| 设置 | 值 | 原因 |
|------|:--:|------|
| `antialiased` | false | 像素字体不需要平滑 |
| `multichannel_signed_distance_field` | false | 低分辨率下 MSDF 产生色边 |
| `oversampling` | 1.0 | 1:1 采样，不缩放字体贴图 |
| `subpixel_positioning` | Disabled (0) | 强制整数像素坐标，拒绝 x=12.3 |
| `hinting` | None (0) | 像素字体原生尺寸下调 hint 反而破坏字形 |

## .tres 字体资源设置

```gdscript
[sub_resource type="SystemFont"]
font_names = PackedStringArray("Fusion Pixel 12px Proportional zh_hans")
hinting = 0                    # none — 像素字体原生尺寸不可调
subpixel_positioning = 0       # disabled — 拒绝亚像素
keep_rounding_remainders = true  # 多行文字防止累积偏移
msdf_pixel_range = 0           # 关闭 MSDF
msdf_size = 0                  # 关闭 MSDF
```

## UITheme 中的 font_size

```gdscript
@export var font_size: int = 12   # MUST 等于所选像素字体的原生尺寸
```

## 排查清单

如果字体仍然模糊：

1. 对比 `font_size` 和字体名中的设计尺寸（如 `12px`）
2. 检查 Label/Button 节点是否被单独覆盖了 font_size
3. 确认 CanvasLayer 的 `follow_viewport_enabled = false`
4. 运行时打印实际生效的字体设置确认编辑器没删配置
