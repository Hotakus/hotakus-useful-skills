---
name: godot-2d-guidelines
description: Godot 4.x 2D 游戏开发规范——写 GDScript 代码、组织 2D 场景、配置物理与碰撞、处理输入、
             管理 TileMap/Camera2D/AnimationPlayer、搭建项目结构。适用于 Godot 4.3+ 2D 项目。
license: MIT
metadata:
  tags: godot, gdscript, 2d, game-development, godot4
---

## When to use

Use this skill whenever you are writing GDScript code, creating 2D scenes, setting up physics/collision,
organizing a Godot 4.x project, or working with TileMap / Camera2D / AnimationPlayer.
Always load before generating any `.gd` file.

## New project setup

```bash
# Create a new Godot 4.x project
godot --path ./my-game --editor
```

Minimal `project.godot`:

```ini
config_version=5
config/name="My 2D Game"
run/main_scene="res://scenes/main/main.tscn"
config/features=PackedStringArray("4.6")
```

> **⚠️ CRITICAL — Version mismatch warning:** The `config/features` version string MUST match the actual Godot binary version on the user's machine. If the project says `"4.3"` but the installed Godot is `4.6`, the editor will show **"This project was last edited in Godot 4.3"** and trigger a migration prompt on every open. Always run `godot --version` FIRST, then write the exact major.minor version (e.g., `"4.6"` for `4.6.2.stable`).
>
> Also note: the `"2D"` feature flag from older Godot versions may not be recognized by 4.6+ builds. If the editor warns about unsupported features, remove `"2D"` and keep only the version string. The project is still 2D — that's controlled via `[rendering]` and `[display]` settings, not this flag.

Recommended `.gitignore`:

```gitignore
# Godot 4+
.godot/
*.translation
export_presets.cfg

# Optional: Git LFS for large assets
# *.png filter=lfs diff=lfs merge=lfs -text
# *.ogg filter=lfs diff=lfs merge=lfs -text
```

MUST commit `.import` files (e.g. `player.png.import`) — they contain import metadata.
MUST NOT commit `.godot/` — auto-generated cache, large and unnecessary.

## Pixel art project setup

Use this section whenever the project is a pixel-art / low-resolution 2D game. Every setting below is battle-tested on Godot 4.3+.

### Complete `project.godot` template

```ini
[application]
config/name="My Pixel Game"
run/main_scene="res://scenes/main/main.tscn"
config/features=PackedStringArray("4.6")
config/icon="res://icon.svg"

[display]
window/size/viewport_width=640
window/size/viewport_height=360
window/size/window_width_override=1920
window/size/window_height_override=1080
window/stretch/mode="viewport"
window/stretch/aspect="keep"
window/stretch/scale_mode="integer"

[rendering]
textures/canvas_textures/default_texture_filter=0
2d/snap/snap_2d_transforms_to_pixel=true

[gui]
common/snap_controls_to_pixels=true
theme/default_font_antialiased=false
theme/default_font_multichannel_signed_distance_field=false
theme/default_font_oversampling=1.0
theme/default_font_subpixel_positioning=0
theme/default_font_hinting=0
```

### Resolution golden rule

Choose a base resolution that scales by **integer** to common 16:9 displays:

| Base | 1280×720 | 1920×1080 | 2560×1440 | 3840×2160 |
|------|:--------:|:---------:|:---------:|:---------:|
| 640×360 | 2× | 3× | 4× | 6× |
| 320×180 | 4× | 6× | 8× | 12× |
| 480×270 | — | 4× | — | 8× |

**640×360** is the recommended starting point — scales perfectly to all common 16:9 displays with no black bars.

### Stretch mode: `viewport` vs `canvas_items`

| | `viewport` | `canvas_items` |
|---|:---:|:---:|
| Mechanism | Render at base res, then integer-scale | Stretch canvas directly |
| Sub-pixel movement | ❌ | ✅ |
| Pixel-perfect guarantee | ✅ Full | ⚠️ Partial |
| Use case | Pure pixel art games | Mixed pixel + HD, 3D elements |

**MUST use `viewport`** for pure pixel art games. `canvas_items` allows sub-pixel positions that fight against pixel snapping, causing jitter.

**MUST use `keep` aspect** for pixel art — `expand` shows extra game area on wider screens, breaking intentional framing.

### Texture filter

`rendering/textures/canvas_textures/default_texture_filter=0` (Nearest) is the **single most important pixel art setting**. Without it, all textures will be blurry.

> **⚠️ CRITICAL:** Changing the default filter does NOT retroactively apply to already-imported textures. After setting this, select all textures in the FileSystem dock, set Filter to "Nearest" in the Import panel, and click **Reimport**.

### Snap settings (Godot 4.3+)

```
rendering/2d/snap/snap_2d_transforms_to_pixel = true   ← ENABLE (uses round() since 4.3)
rendering/2d/snap/snap_2d_vertices_to_pixel = false     ← DEPRECATED (moved to Advanced since 4.3)
```

`snap_2d_transforms_to_pixel` was stabilized in Godot 4.3 (PR #87297) — changed from `floor()` to `round()`, eliminating 1px offset jitter. The old `snap_2d_vertices_to_pixel` is deprecated and should NOT be enabled alongside transforms.

### Physics & movement speed alignment

For pixel-perfect movement without jitter, movement speed MUST align with physics tick rate:

```
move_speed ÷ physics_ticks_per_second = integer
```

Good speeds (60 TPS): 30, 60, 120, 180, 240
Bad speeds: 50, 70, 100, 150

If the desired speed doesn't align, change `physics_ticks_per_second` to match:
```gdscript
Engine.physics_ticks_per_second = 50  # then use speed = 50, 100, 150...
```

### Physics interpolation (Godot 4.3+)

Enable for high-refresh displays (120Hz+). Off by default for simplicity:

```ini
[physics]
common/physics_interpolation=true
common/physics_jitter_fix=0.0
```

When enabled:
- ALL movement logic MUST be in `_physics_process()`
- Camera follows in `_process()` (reads interpolated position)
- Call `reset_physics_interpolation()` after teleporting

**Trade-off:** Adds ~half-frame input lag. Not recommended for retro pixel games where frame-perfect timing matters.

### Camera2D — 关键规则

- Camera2D `anchor_mode = 1`（DRAG_CENTER），玩家 `position.round()` 在 `move_and_slide()` 后
- 相机在 `_process()` 跟随（非 `_physics_process()`），避免场景树顺序导致的 1 帧偏移
- 禁用 `position_smoothing_enabled`（手动设 position 时冲突）
- 完整说明见 [./rules/camera2d.md](./rules/camera2d.md)

### GUI pixel alignment

```ini
[gui]
common/snap_controls_to_pixels=true
```

像素游戏字体渲染配置见 [./rules/fonts.md](./rules/fonts.md) —— 全局关闭抗锯齿/MSDF/subpixel/hinting，字体尺寸必须匹配原生像素尺寸。

### Import settings for pixel art assets

| Setting | Value | Why |
|---------|-------|-----|
| Filter | Nearest | No blurring |
| Compression | Lossless | No artifacts on low-res textures |
| Mipmaps | Off | Not needed for 2D pixel art |
| Fix Alpha Border | On | Prevents dark edges on transparent sprites |

### Self-check checklist

Before running, verify:
- [ ] `stretch/mode = viewport`, `scale_mode = integer`, `aspect = keep`
- [ ] `default_texture_filter = 0` (Nearest) + all textures reimported
- [ ] `snap_2d_transforms_to_pixel = true`, `vertices = false`
- [ ] `move_speed ÷ physics_tps = integer`
- [ ] Camera in `_process()`, `position_smoothing_enabled = false`, `position.round()` on player
- [ ] `gui/snap_controls_to_pixels = true`
- [ ] `.import` files committed, `.godot/` in `.gitignore`

## GDScript core rules

### Code order — strictly follow this sequence

Every `.gd` file MUST organize members in the official Godot order:

```gdscript
# 01. @tool, @icon, @static_unload
# 02. class_name
# 03. extends
# 04. ## doc comments
# 05. signals
# 06. enums
# 07. constants
# 08. static variables
# 09. @export variables
# 10. regular public variables
# 11. @onready variables
# 12. _static_init()
# 13. remaining static methods
# 14. _init() → _enter_tree() → _ready() → _process() → _physics_process()
# 15. overridden custom methods
# 16. remaining methods
# 17. inner classes
```

### Naming — mandatory conventions

| Category | Convention | Example |
|----------|-----------|---------|
| Files / folders | `snake_case` | `player.gd`, `enemy_ai/` |
| `class_name` | `PascalCase` | `Player`, `HealthComponent` |
| Variables / functions | `snake_case` | `move_speed`, `take_damage()` |
| Constants | `CONSTANT_CASE` | `MAX_HEALTH`, `DEFAULT_GRAVITY` |
| Enums (name) | `PascalCase` | `enum State` |
| Enums (members) | `CONSTANT_CASE` | `{IDLE, RUNNING, JUMPING}` |
| Signals | `snake_case` past-tense | `health_changed`, `player_died` |
| Private members | `_` prefix | `_current_state`, `_apply_gravity()` |
| Node names in scene | `PascalCase` | `Camera2D`, `PlayerSprite` |

### Static typing — MANDATORY

Every variable, parameter, and return type MUST have an explicit type hint.

```gdscript
# Correct
var health: int = 100
var velocity: Vector2 = Vector2.ZERO
func apply_damage(amount: int) -> void:
    health -= amount

# FORBIDDEN — missing type hints
var health = 100
func apply_damage(amount):
    health -= amount
```

Use `:=` only when the type is obvious from the immediate right-hand side:

```gdscript
# OK — type is visually obvious
var direction := Vector2.RIGHT
var sprite := $Sprite2D as Sprite2D

# FORBIDDEN — reader cannot infer type from this line
var result := complex_function()
```

### Node access — cache, don't search

```gdscript
# Correct: @onready caches the node reference once
@onready var sprite: Sprite2D = $Sprite2D
@onready var anim_player: AnimationPlayer = $AnimationPlayer

# FORBIDDEN: $ in _process() or _physics_process() — string lookup every frame
func _process(delta: float) -> void:
    $Sprite2D.rotation += delta  # never do this
```

When `get_node()` cannot infer the concrete type, use `as`:

```gdscript
@onready var health_bar := get_node("UI/HealthBar") as ProgressBar
```

### preload vs load

```gdscript
# preload — compile-time, for CONSTANT paths — PREFERRED
const PlayerScene: PackedScene = preload("res://scenes/player/player.tscn")
const BulletData: Resource = preload("res://resources/bullet_data.tres")

# load — runtime, for DYNAMIC paths only
var dynamic_scene: PackedScene = load("res://levels/%s.tscn" % level_name)
```

FORBIDDEN: `load()` for paths known at compile time.
FORBIDDEN: `preload()` inside functions.
FORBIDDEN: `load()` to initialize a `const`.

### _ready vs _init vs _process

| Callback | When | What to do |
|----------|------|------------|
| `_init()` | Object created | Initialize own properties; scene tree NOT available |
| `_ready()` | Node enters scene tree | Access children, connect signals, one-time setup |
| `_process(delta)` | Every visual frame | Visual updates, UI polling |
| `_physics_process(delta)` | Fixed timestep (60Hz default) | Movement, physics, `move_and_slide()` |

FORBIDDEN: `get_node()` or `$` in `_init()`.
FORBIDDEN: physics logic (movement, collision) in `_process()`.
FORBIDDEN: calling `move_and_slide()` outside `_physics_process()`.

```gdscript
func _physics_process(delta: float) -> void:
    velocity.y += gravity * delta
    velocity.x = direction * speed
    move_and_slide()
```

### Signals — emit, don't poll

```gdscript
# Declare typed signals
signal health_changed(new_health: int)
signal player_died

# Connect in code (preferred over editor connections for clarity)
func _ready() -> void:
    health_component.health_changed.connect(_on_health_changed)

func _on_health_changed(new_health: int) -> void:
    if new_health <= 0:
        player_died.emit()

# One-shot disconnection for dynamic nodes
func _on_area_entered(body: Node2D) -> void:
    body.tree_exited.connect(_cleanup, CONNECT_ONE_SHOT)
```

FORBIDDEN: string-based `connect("signal_name", self, "_handler")` — use `signal.emit()` and `signal.connect(callable)`.
FORBIDDEN: forgetting to disconnect signals when nodes are freed.

### class_name usage

Use `class_name` for:
- Shared data types: `class_name PlayerData extends Resource`
- Reusable components: `class_name HealthComponent extends Node`
- Base classes for inheritance: `class_name Enemy extends CharacterBody2D`

FORBIDDEN: `class_name` on scripts used in a single scene only.
FORBIDDEN: `class_name` names colliding with built-in Godot classes.
FORBIDDEN: `class_name` on inner classes — they cannot be serialized by `ResourceSaver`.

### Properties — add before adding to tree

```gdscript
# Correct: set properties BEFORE add_child
var node: Node2D = Node2D.new()
node.name = "Enemy"
node.position = Vector2(100, 200)
add_child(node)

# FORBIDDEN: add_child before setting properties — triggers redundant updates
var node: Node2D = Node2D.new()
add_child(node)
node.position = Vector2(100, 200)
```

## Scene tree patterns

### Composition over inheritance

Prefer composing focused nodes over deep inheritance:

```text
# Good: Player composed of small focused components
# Player (CharacterBody2D)
#   ├── Sprite2D
#   ├── CollisionShape2D
#   ├── HealthComponent (Node)
#   ├── StateMachine (Node)
#   └── Hurtbox (Area2D)

# Bad: Deep inheritance chain
# CharacterBody2D → Player → Warrior → Knight → DragonKnight
```

### Scene exports as PackedScene — dependency injection

```gdscript
# Export scenes for configuration, not hardcoded paths
@export var bullet_scene: PackedScene
@export var explosion_scene: PackedScene

func shoot() -> void:
    var bullet: Node2D = bullet_scene.instantiate()
    get_tree().current_scene.add_child(bullet)
```

FORBIDDEN: hardcoded `preload("res://...")` for scene references that should be configurable.

### Scene root convention

Scene root node type MUST match `extends`:

| Script | Scene root type |
|--------|----------------|
| `extends CharacterBody2D` | `CharacterBody2D` |
| `extends Area2D` | `Area2D` |
| `extends Node2D` | `Node2D` |
| `extends Control` | `Control` |

### Scene coupling — by preference

When a child scene needs to communicate with its parent or siblings, use (in order of preference):

1. **Signals** — safest, for "response" behavior (parent connects to child's signal)
2. **Callable properties** — for "command" behavior (parent injects a callable)
3. **Node/Resource references** — when the dependency is structural and intentional
4. **NodePath** — last resort, breaks reusability

FORBIDDEN: `get_node("../../Sibling")` — hardcoded relative path, breaks if tree changes.

## Physics & movement

### Physics body selection

| Body type | Use when |
|-----------|---------|
| `CharacterBody2D` | Script-driven movement with collision response (player, NPCs, enemies) |
| `RigidBody2D` | Physics-simulated objects (crates, balls, debris) |
| `StaticBody2D` | Immovable objects (walls, floors, platforms) |
| `Area2D` | Hitboxes, hurtboxes, detection zones, pickups |
| `AnimatableBody2D` | Moving platforms that push `CharacterBody2D` |

### CharacterBody2D movement — standard pattern

```gdscript
extends CharacterBody2D

@export var move_speed: float = 200.0
@export var jump_velocity: float = -400.0

var gravity: float = ProjectSettings.get_setting("physics/2d/default_gravity")

func _physics_process(delta: float) -> void:
    # Gravity
    if not is_on_floor():
        velocity.y += gravity * delta

    # Horizontal input with deceleration
    var direction: float = Input.get_axis("move_left", "move_right")
    if direction:
        velocity.x = direction * move_speed
    else:
        velocity.x = move_toward(velocity.x, 0, move_speed)

    # Jump — single-frame check
    if Input.is_action_just_pressed("jump") and is_on_floor():
        velocity.y = jump_velocity

    move_and_slide()
```

FORBIDDEN: directly setting `position` on `CharacterBody2D` — use `velocity` + `move_and_slide()`.
FORBIDDEN: reading `Input` for movement in `_process()` — use `_physics_process()`.

### Collision layers & masks — bitwise values

```gdscript
# layer = what this body IS (bit position)
# mask  = what this body DETECTS (bit position)

# Use exported bit indices (human-readable), convert to bit values at runtime
@export var body_layer: int = 1    # player
@export var detect_mask: int = 2   # enemies (bit 2)

func _ready() -> void:
    collision_layer = 1 << (body_layer - 1)
    collision_mask = 1 << (detect_mask - 1)
```

See [./rules/physics-layers.md](./rules/physics-layers.md) for full layer design strategy.

### Area2D for hitboxes

```gdscript
extends Area2D

func _ready() -> void:
    area_entered.connect(_on_area_entered)
    body_entered.connect(_on_body_entered)

func _on_body_entered(body: Node2D) -> void:
    if body is Player:
        body.take_damage(10)
```

## Input handling

### Input Map actions — define once, reference by name

```gdscript
# Correct — uses Input Map action names
var direction: float = Input.get_axis("move_left", "move_right")
if Input.is_action_just_pressed("jump"):
    jump()

# FORBIDDEN — hardcoded key codes
if Input.is_key_pressed(KEY_SPACE):
    jump()
```

### Input callback selection

| Method | Use for |
|--------|---------|
| `_input(event)` | Raw events, BEFORE propagation; `get_viewport().set_input_as_handled()` to consume |
| `_unhandled_input(event)` | Events NOT handled by Control nodes; game world click detection |
| `_gui_input(event)` | Control nodes only |
| `Input.is_action_pressed()` in `_physics_process()` | Continuous input (movement) |
| `Input.is_action_just_pressed()` | Single-frame actions (jump, attack, interact) |
| `Input.get_axis(neg, pos)` | Two-key axis (left/right, up/down) |
| `Input.get_vector(neg_x, pos_x, neg_y, pos_y)` | Four-key directional vector |

```gdscript
# _unhandled_input for world click detection
func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseButton and event.pressed and event.button_index == MOUSE_BUTTON_LEFT:
        var click_pos: Vector2 = get_global_mouse_position()
        _handle_click(click_pos)
```

FORBIDDEN: `_input()` without consuming events (will block UI).
FORBIDDEN: polling `Input.is_key_pressed(KEY_*)` instead of using Input Map actions.

## TileMap

### TileMapLayer (Godot 4.3+)

```gdscript
extends TileMapLayer

func _ready() -> void:
    var cells: Array[Vector2i] = get_used_cells()
    set_cell(Vector2i(5, 10), source_id=0, atlas_coords=Vector2i(3, 2))

    # Read custom data
    var tile_data: TileData = get_cell_tile_data(Vector2i(5, 10))
    if tile_data:
        var is_solid: bool = tile_data.get_custom_data("solid")
```

### TileMapPattern for procedural generation

```gdscript
func place_room(origin: Vector2i, pattern: TileMapPattern) -> void:
    set_pattern(origin, pattern)

func create_pattern(cells: Array[Vector2i]) -> TileMapPattern:
    var pattern: TileMapPattern = TileMapPattern.new()
    for cell in cells:
        pattern.set_cell(cell, source_id=0, atlas_coords=Vector2i(5, 3), alternative_tile=0)
    return pattern
```

FORBIDDEN: calling `set_cell()` in `_process()` — TileMap updates are expensive.
FORBIDDEN: `alternative_tile` < 0 in `TileMapPattern.set_cell()` — cells appear empty.
FORBIDDEN: stacking navigation meshes from multiple TileMapLayers — they merge incorrectly in 2D.

### Performance: bake navigation off TileMap

After designing the TileMap, bake navigation to a `NavigationRegion2D` and disable the TileMap's navigation layer.

## Camera2D

```gdscript
extends Camera2D

func _ready() -> void:
    # Smooth follow — use built-in smoothing, not manual lerp
    position_smoothing_enabled = true
    position_smoothing_speed = 5.0

    # World bounds
    limit_left = 0
    limit_right = 2048
    limit_top = 0
    limit_bottom = 1536

    # Optional: drag margins (0.0-1.0, fraction of screen)
    # drag_left_margin = 0.2
    # drag_right_margin = 0.2
    # drag_horizontal_enabled = true
```

Key properties:

| Property | Type | Description |
|----------|------|-------------|
| `position_smoothing_enabled` | `bool` | Enable smooth follow |
| `position_smoothing_speed` | `float` | Pixels per second |
| `limit_left/right/top/bottom` | `int` | World bounds (pixels) |
| `drag_*_margin` | `float` | 0.0–1.0 fraction of screen before camera drags |
| `anchor_mode` | `enum` | `ANCHOR_MODE_FIXED_TOP_LEFT` or `ANCHOR_MODE_DRAG_CENTER` |

FORBIDDEN: manually lerping `Camera2D.position` in `_process()` — use `position_smoothing_enabled`.
FORBIDDEN: using `global_position` to read camera position — use `get_screen_center_position()`.
FORBIDDEN: not setting `limit_*` when the world has finite bounds.

## Animation

### AnimatedSprite2D vs AnimationPlayer

| Feature | `AnimatedSprite2D` | `AnimationPlayer` |
|---------|:---:|:---:|
| Sprite sheet playback | ✅ | ✅ (also supports sprites) |
| Property animation (position, modulate, etc.) | ❌ | ✅ |
| Method calling via keyframes | ❌ | ✅ `Call Method` track |
| Audio sync | ❌ | ✅ Audio track |
| Blending / state machine | ❌ | ✅ `AnimationTree` |

Rule: use `AnimatedSprite2D` for sprite-only playback. Use `AnimationPlayer` whenever you animate anything beyond sprites.

FORBIDDEN: using `AnimatedSprite2D` when you also need property animation — use `AnimationPlayer` for everything instead.

### AnimationPlayer — standard usage

```gdscript
@onready var anim_player: AnimationPlayer = $AnimationPlayer

func play_animation(anim_name: String) -> void:
    if anim_player.current_animation != anim_name:
        anim_player.play(anim_name)

func _on_animation_finished(anim_name: StringName) -> void:
    match anim_name:
        "attack":
            play_animation("idle")
        "death":
            queue_free()
```

> `play()` does NOT apply immediately — it takes effect on the next frame. Use `advance(0)` after `play()` to force immediate update.

### AnimationTree for state-based animation

```gdscript
@onready var anim_tree: AnimationTree = $AnimationTree
@onready var state_machine: AnimationNodeStateMachinePlayback = anim_tree.get("parameters/playback")

func travel_to(state_name: String) -> void:
    state_machine.travel(state_name)

func _physics_process(delta: float) -> void:
    # Blend tree parameters
    anim_tree.set("parameters/IdleWalkRun/blend_position", direction.length())
```

See [./rules/state-machines.md](./rules/state-machines.md) for animation + logic state machine patterns.

## Project structure

### Recommended layout

```text
project/
├── project.godot
├── scenes/                  # .tscn files by feature
│   ├── main/main.tscn
│   ├── player/player.tscn
│   └── enemies/slime.tscn
├── scripts/                 # Autoloads, utilities (no .tscn)
│   ├── autoloads/
│   │   ├── event_bus.gd
│   │   └── game_state.gd
│   └── utils/
├── resources/               # .tres data definitions
│   ├── enemies/
│   └── items/
├── assets/                  # Raw assets
│   ├── sprites/
│   ├── audio/
│   └── fonts/
├── shaders/                 # .gdshader files
└── tests/                   # GUT test scripts
```

### File naming

| File type | Convention | Example |
|-----------|-----------|---------|
| Scene | `snake_case.tscn` | `player.tscn`, `main_menu.tscn` |
| Script (paired with scene) | Same folder, same name | `scenes/player/player.tscn` + `scenes/player/player.gd` |
| Standalone script | `snake_case.gd` | `scripts/utils/math_utils.gd` |
| Resource | `snake_case.tres` | `resources/player_stats.tres` |
| Shader | `snake_case.gdshader` | `shaders/outline.gdshader` |

### Autoload rules

Single-responsibility Autoloads. Each Autoload does ONE thing:

```text
# project.godot Autoload list — order matters!
GlobalConfig (Node)     → read-only configuration
EventBus (Node)         → global signal routing
GameState (Node)        → runtime game state
SceneManager (Node)     → scene transitions
```

FORBIDDEN: giant "Global.gd" that manages everything.
FORBIDDEN: using Autoload for data that belongs in a `Resource` file.
See [./rules/autoloads.md](./rules/autoloads.md).

### Custom Resources (ScriptableObject pattern)

```gdscript
# bot_stats.gd
class_name BotStats
extends Resource

@export var health: int = 100
@export var speed: float = 200.0
@export var damage: int = 10
```

```gdscript
# In a node script
@export var stats: BotStats

func _ready() -> void:
    if stats:
        print(stats.health)  # typed auto-completion
```

## CLI commands

```bash
# Run the project
godot --path ./my-game

# Run headless
godot --path ./my-game --headless

# Run a specific scene
godot --path ./my-game scenes/main/main.tscn

# Export
godot --path ./my-game --export-release "Windows" build/game.exe

# Run GUT tests
godot --path ./my-game -s addons/gut/gut_cmdln.gd -gdir=res://tests -gexit
```

## Advanced topics

- State machines: [./rules/state-machines.md](./rules/state-machines.md)
- Physics layers & collision design: [./rules/physics-layers.md](./rules/physics-layers.md)
- Autoloads & signal bus: [./rules/autoloads.md](./rules/autoloads.md)
- 2D shaders: [./rules/shaders-2d.md](./rules/shaders-2d.md)
- Performance optimization: [./rules/performance.md](./rules/performance.md)
- Testing with GUT: [./rules/testing-gut.md](./rules/testing-gut.md)
- AnimationTree & Player: [./rules/animation.md](./rules/animation.md)
- UI Theme Resource: [./rules/ui-theme.md](./rules/ui-theme.md)
- Pixel Font Rendering: [./rules/fonts.md](./rules/fonts.md)
- Window Management: [./rules/window-management.md](./rules/window-management.md)
- Camera2D jitter prevention: [./rules/camera2d.md](./rules/camera2d.md)
- Pixel art design constraints: [./rules/pixel-art-design.md](./rules/pixel-art-design.md)
