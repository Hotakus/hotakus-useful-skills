# GUT 测试规范

Godot Unit Test (GUT) 框架的测试编写、组织与 CI 集成规范。

## When to use

When writing unit tests for GDScript code, setting up test directories, or configuring
CI for Godot projects. Load this rule when `test`, `GUT`, `unit test`, `assert`, or
`test-driven` is mentioned.

## Installation

```bash
# From Godot Asset Library: search "GUT"
# Or clone to addons/
git clone https://github.com/bitwes/Gut.git addons/gut
```

Enable the plugin: Project Settings → Plugins → GUT → Enable.

## Test directory structure

```text
project/
├── tests/
│   ├── unit/                  # Pure logic, no scene tree needed
│   │   ├── test_math_utils.gd
│   │   └── test_player_data.gd
│   ├── integration/           # Scene-dependent tests
│   │   ├── test_player_movement.gd
│   │   └── test_combat_system.gd
│   └── resources/             # Test fixtures (.tres, .tscn)
│       └── test_player.tscn
└── addons/gut/
```

Mirror `src/` structure: `scripts/utils/math_utils.gd` → `tests/unit/test_math_utils.gd`.

FORBIDDEN: test files outside `tests/` directory.
FORBIDDEN: test files without `test_` prefix.

## Test script template

```gdscript
# tests/unit/test_health_component.gd
extends GutTest

const HealthComponent = preload("res://scripts/components/health_component.gd")

var component: HealthComponent

func before_each() -> void:
    component = autoqfree(HealthComponent.new())
    component.max_health = 100
    component.current_health = 100

func after_each() -> void:
    component = null

func test_take_damage_reduces_health() -> void:
    component.take_damage(30)
    assert_eq(component.current_health, 70)

func test_damage_below_zero_emits_died_signal() -> void:
    watch_signals(component)
    component.take_damage(150)
    assert_signal_emitted(component, "died")

func test_heal_does_not_exceed_max() -> void:
    component.current_health = 90
    component.heal(20)
    assert_eq(component.current_health, 100)

func test_heal_negative_is_ignored() -> void:
    component.heal(-10)
    assert_eq(component.current_health, 100)
```

Key rules:
- Use `autoqfree()` for nodes created in `before_each()` — auto-freed after each test
- Use `watch_signals()` before emitting to track signal calls
- Test file naming: `test_<subject>.gd`
- Test function naming: `test_<what_happens_when_condition>()`
- One assertion concept per test (can have multiple `assert_*` for the same scenario)

## Common assertions

| Assertion | Use |
|-----------|-----|
| `assert_eq(actual, expected)` | Exact equality |
| `assert_ne(actual, unexpected)` | Not equal |
| `assert_true(condition)` | Boolean true |
| `assert_false(condition)` | Boolean false |
| `assert_almost_eq(actual, expected, tolerance)` | Float comparison |
| `assert_gt(a, b)` / `assert_lt(a, b)` | Greater/less than |
| `assert_between(value, low, high)` | Range check |
| `assert_signal_emitted(obj, "signal_name")` | Signal fired |
| `assert_signal_not_emitted(obj, "signal_name")` | Signal not fired |
| `assert_signal_emit_count(obj, "signal_name", count)` | Exact emit count |
| `assert_called(thing, "method")` | Method was called (on doubled object) |
| `assert_null(value)` / `assert_not_null(value)` | Null checks |

## Scene testing

```gdscript
# tests/integration/test_player_movement.gd
extends GutTest

var player: CharacterBody2D
var scene: Node

func before_each() -> void:
    scene = autoqfree(Node.new())
    add_child_autoqfree(scene)

    var player_scene: PackedScene = load("res://tests/resources/test_player.tscn")
    player = autoqfree(player_scene.instantiate())
    scene.add_child(player)

    # Wait one frame for _ready() to complete
    await wait_seconds(0.1)

func test_player_moves_right_when_input_pressed() -> void:
    # Simulate input
    Input.action_press("move_right")
    await wait_seconds(0.1)

    assert_gt(player.global_position.x, 0.0, "player should move right")

    Input.action_release("move_right")

func test_player_jumps_when_on_floor() -> void:
    Input.action_press("jump")
    await wait_seconds(0.1)

    assert_gt(player.velocity.y, -100.0, "player should have upward velocity")
    Input.action_release("jump")
```

FORBIDDEN: testing `_process()` directly — use `await wait_seconds()` or `await wait_frames()`.
FORBIDDEN: `queue_free()` in test cleanup — use `autoqfree()`.

## Doubles (Mocks/Stubs)

```gdscript
# Double a class to intercept method calls
func test_player_receives_damage_from_hitbox() -> void:
    var player: Player = autoqfree(Player.new())

    # Create a double of a hitbox that returns known damage
    var hitbox_double = double(Hitbox).new()
    stub(hitbox_double, "damage").to_return(25)

    # Simulate area entered
    player._on_hurtbox_area_entered(hitbox_double)

    assert_eq(player.current_health, 75)

func test_enemy_spawner_calls_spawn() -> void:
    var spawner = double(EnemySpawner).new()
    stub(spawner, "can_spawn").to_return(true)

    spawner.try_spawn()
    assert_called(spawner, "do_spawn")
```

## CLI execution

```bash
# Run all tests
godot --path ./my-game -s addons/gut/gut_cmdln.gd -gdir=res://tests -gexit

# Run specific directory
godot --path ./my-game -s addons/gut/gut_cmdln.gd -gdir=res://tests/unit -gexit

# Run with specific config
godot --path ./my-game -s addons/gut/gut_cmdln.gd -gconfig=res://.gutconfig.json -gexit
```

`.gutconfig.json` for CI:

```json
{
  "dirs": ["res://tests/"],
  "should_exit": true,
  "ignore_pause": true,
  "log_level": 2,
  "junit_xml_file": "test_results.xml"
}
```

## Test coverage targets

| Layer | Target | Environment |
|-------|:---:|-------------|
| Pure logic (math, data, state machines) | ≥ 80% | PC (no Godot runtime needed) |
| Scene-dependent (physics, input, signals) | Core paths | Godot editor or headless |
| Full integration (level loading, save/load) | Critical paths only | Godot editor |

## Common pitfalls

FORBIDDEN: calling `assert_*` after `await` without checking test state — the test may have timed out.
FORBIDDEN: using `yield()` (Godot 3) — use `await` (Godot 4).
FORBIDDEN: creating test scenes that depend on Autoload configuration not present in test environment — mock Autoloads with `double()`.
