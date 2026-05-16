# Autoload 与信号总线

Godot Autoload 单例的设计原则、信号总线（EventBus）模式与场景切换规范。

## When to use

When creating Autoload singletons, designing global signal routing, implementing
scene transitions, or managing shared game state. Load this rule when
`Autoload`, `singleton`, `event bus`, `signal bus`, or `SceneManager` is mentioned.

## Core principles

### When to use Autoload

Use Autoload when the system:
- Manages state entirely internally (task system, dialogue system)
- Must be globally accessible from any scene
- Should exist independently of any specific scene

Do NOT use Autoload for:
- Data that belongs in a `Resource` file (`.tres`)
- Functionality that only one scene needs
- Things that can be a `class_name` with `static func` instead

### Single responsibility

Each Autoload does ONE thing:

```text
# project.godot — correct Autoload list
GlobalConfig (Node)     → read-only configuration, loaded from .tres
EventBus (Node)         → global signal routing only
GameState (Node)        → runtime save/load state
SceneManager (Node)     → scene transitions, loading screens
AudioManager (Node)     → music/SFX playback
```

FORBIDDEN: giant `Global.gd` that manages config + state + audio + scenes.
FORBIDDEN: Autoload order that creates circular dependencies (e.g., EventBus before GlobalConfig must be correct).

## EventBus pattern

A dedicated Autoload that routes signals between decoupled systems:

```gdscript
# autoloads/event_bus.gd
extends Node

# Player events
signal player_died
signal player_health_changed(current: int, max_health: int)

# Game events
signal game_paused
signal game_resumed
signal level_completed(level_name: String)

# UI events
signal coin_collected(amount: int)
signal screen_shake_requested(intensity: float, duration: float)
```

Usage from any node:

```gdscript
# Emit — anywhere in the project
func take_damage(amount: int) -> void:
    health -= amount
    EventBus.player_health_changed.emit(health, max_health)
    if health <= 0:
        EventBus.player_died.emit()

# Connect — in _ready()
func _ready() -> void:
    EventBus.coin_collected.connect(_on_coin_collected)
    EventBus.player_died.connect(_on_player_died)

func _on_coin_collected(amount: int) -> void:
    coin_count += amount
    update_ui()
```

FORBIDDEN: direct cross-scene `get_node()` calls — use EventBus instead.
FORBIDDEN: emitting EventBus signals during node `_exit_tree()` without checking `is_inside_tree()`.
FORBIDDEN: forgetting to disconnect EventBus connections in `_exit_tree()`:

```gdscript
func _exit_tree() -> void:
    EventBus.coin_collected.disconnect(_on_coin_collected)
```

## SceneManager pattern

```gdscript
# autoloads/scene_manager.gd
extends Node

signal scene_changing(from: String, to: String)
signal scene_changed(to: String)

var current_scene: String
var loading_screen_path: String = "res://scenes/ui/loading_screen.tscn"

func change_scene(scene_path: String) -> void:
    scene_changing.emit(current_scene, scene_path)

    # Optional: show loading screen
    if ResourceLoader.exists(loading_screen_path):
        var loading_screen: PackedScene = load(loading_screen_path)
        get_tree().root.add_child(loading_screen.instantiate())

    # Wait for loading screen to appear, then change
    await get_tree().process_frame
    get_tree().change_scene_to_file(scene_path)
    current_scene = scene_path
    scene_changed.emit(scene_path)
```

FORBIDDEN: `get_tree().change_scene_to_file()` called from multiple places — centralize in SceneManager.
FORBIDDEN: scene path strings scattered across the project — define as constants.

```gdscript
# Better: centralize scene paths
# autoloads/scene_paths.gd (or as constants in SceneManager)
class_name ScenePaths

const MAIN_MENU: String = "res://scenes/ui/main_menu.tscn"
const GAMEPLAY: String = "res://scenes/main/main.tscn"
const GAME_OVER: String = "res://scenes/ui/game_over.tscn"
```

## GameState pattern

```gdscript
# autoloads/game_state.gd
extends Node

signal coins_changed(new_amount: int)

var coins: int = 0
var current_level: int = 1

func add_coins(amount: int) -> void:
    coins += amount
    coins_changed.emit(coins)

func reset() -> void:
    coins = 0
    current_level = 1
```

FORBIDDEN: directly modifying `GameState` properties without going through setter methods — signals won't fire.
FORBIDDEN: storing scene-specific state in GameState — each scene manages its own state.

## Autoload ordering

```text
# MUST be in this order — dependencies run top to bottom
GlobalConfig    → (no dependencies)
EventBus        → (no dependencies)
GameState       → (may depend on GlobalConfig)
AudioManager    → (may depend on GlobalConfig)
SceneManager    → (may depend on GameState, EventBus)
```

FORBIDDEN: Autoload A depending on Autoload B's `_ready()` having run — order in `project.godot` doesn't guarantee `_ready()` completion order. Use `await` or signals for cross-Autoload initialization.

```gdscript
# In SceneManager._ready() — safe pattern
func _ready() -> void:
    # Wait one frame to ensure all Autoloads are ready
    await get_tree().process_frame
    EventBus.game_paused.connect(_on_game_paused)
```
