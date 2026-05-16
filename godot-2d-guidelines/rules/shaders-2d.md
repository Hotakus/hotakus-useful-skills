# 2D 着色器

Godot 2D shader 基础、CanvasItem 着色器模式、2D 光照与后处理效果规范。

## When to use

When writing `.gdshader` files for 2D sprites, applying visual effects (outline, glow, dissolve),
configuring 2D lights, or optimizing shader performance. Load this rule when
`shader`, `gdshader`, `canvas_item`, `2D light`, or `shader_material` is mentioned.

## CanvasItem shader — minimal template

```gdshader
shader_type canvas_item;

uniform vec4 modulate_color : source_color = vec4(1.0);
uniform float time_scale = 1.0;

void fragment() {
    // Sample the sprite texture
    vec4 tex_color = texture(TEXTURE, UV);

    // Apply modulation
    COLOR = tex_color * modulate_color;
}
```

Key built-ins:

| Variable | Type | Description |
|----------|------|-------------|
| `TEXTURE` | `sampler2D` | The sprite's texture |
| `UV` | `vec2` | Normalized texture coordinates (0-1) |
| `COLOR` | `vec4` | Output color — must be set in `fragment()` |
| `TIME` | `float` | Elapsed time in seconds (engine time) |
| `VERTEX` | `vec2` | Vertex position (in `vertex()`) |

FORBIDDEN: `shader_type spatial` in 2D projects — use `canvas_item`.
FORBIDDEN: assuming `COLOR` has a default value — always set it explicitly.

## Common 2D shader patterns

### Outline / stroke

```gdshader
shader_type canvas_item;

uniform vec4 outline_color : source_color = vec4(0.0, 0.0, 0.0, 1.0);
uniform float outline_width : hint_range(0.0, 10.0) = 1.0;

void fragment() {
    vec4 tex_color = texture(TEXTURE, UV);
    vec2 texel_size = 1.0 / vec2(textureSize(TEXTURE, 0));

    // Sample neighbors
    float alpha = tex_color.a;
    alpha += texture(TEXTURE, UV + vec2(outline_width * texel_size.x, 0.0)).a;
    alpha += texture(TEXTURE, UV + vec2(-outline_width * texel_size.x, 0.0)).a;
    alpha += texture(TEXTURE, UV + vec2(0.0, outline_width * texel_size.y)).a;
    alpha += texture(TEXTURE, UV + vec2(0.0, -outline_width * texel_size.y)).a;

    // Outline where neighbor has alpha but current pixel doesn't
    float outline = clamp(alpha - tex_color.a, 0.0, 1.0);
    COLOR = mix(tex_color, outline_color, outline);
}
```

### Dissolve / burn

```gdshader
shader_type canvas_item;

uniform float progress : hint_range(0.0, 1.0) = 0.0;
uniform sampler2D noise_texture : filter_nearest;
uniform float edge_width : hint_range(0.0, 0.5) = 0.1;
uniform vec4 edge_color : source_color = vec4(1.0, 0.5, 0.0, 1.0);

void fragment() {
    vec4 tex_color = texture(TEXTURE, UV);
    float noise = texture(noise_texture, UV).r;

    float cutoff = progress;
    float edge = cutoff + edge_width;

    // Dissolve
    float visibility = smoothstep(cutoff - 0.01, cutoff, noise);

    // Glow edge
    float edge_glow = smoothstep(cutoff, edge, noise) * (1.0 - smoothstep(edge, edge + 0.01, noise));

    vec4 final_color = tex_color;
    final_color.a *= visibility;
    final_color.rgb = mix(final_color.rgb, edge_color.rgb, edge_glow * edge_color.a);

    COLOR = final_color;
}
```

### Grayscale / desaturation

```gdshader
shader_type canvas_item;

void fragment() {
    vec4 tex_color = texture(TEXTURE, UV);
    float gray = dot(tex_color.rgb, vec3(0.299, 0.587, 0.114));
    COLOR = vec4(vec3(gray), tex_color.a);
}
```

## Performance rules

- Use `filter_nearest` on noise/pattern textures when pixel-perfect sampling is intended
- Avoid `texture()` calls in loops — sample once, process, output
- Prefer `mix()` over manual branching (`if` in shader) when possible
- Set `render_mode unshaded` on materials that don't need lighting
- Use `hint_range` on uniforms for editor sliders

FORBIDDEN: `discard` in `canvas_item` shaders (not supported).
FORBIDDEN: heavy computation in `vertex()` — 2D sprites only have 4 vertices, keep it trivial.

## 2D lighting basics

```gdscript
# PointLight2D — configure in _ready()
@onready var light: PointLight2D = $PointLight2D

func _ready() -> void:
    light.texture_scale = 2.0
    light.energy = 0.8
    light.color = Color(1.0, 0.9, 0.7)  # warm light

# WorldEnvironment for global 2D settings
# Add WorldEnvironment node → Environment resource → adjust glow, adjustments
```

| Light2D type | Use |
|-------------|-----|
| `PointLight2D` | Torch, lantern, muzzle flash |
| `DirectionalLight2D` | Sunlight, global ambient |
| `LightOccluder2D` | Shadow casting — attach to StaticBody2D |

FORBIDDEN: more than 8 visible `PointLight2D` nodes per viewport — each light is a separate render pass.
FORBIDDEN: `LightOccluder2D` with complex polygon shapes — keep occluder shapes simple (≤ 8 vertices).
