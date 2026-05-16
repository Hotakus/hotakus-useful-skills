# 性能优化

Godot 2D 项目性能优化规范，覆盖渲染、物理、脚本与资源管理。

## When to use

When optimizing frame rate, reducing draw calls, managing object pools, profiling
bottlenecks, or improving `_process()` / `_physics_process()` performance.
Load this rule when `performance`, `optimize`, `frame rate`, `draw call`, `batching`,
or `object pool` is mentioned.

## Core principle

**Profile before optimizing.** Don't guess where the bottleneck is.

```gdscript
# Quick CPU profiling
var start: int = Time.get_ticks_usec()
expensive_operation()
var elapsed: int = Time.get_ticks_usec() - start
print("took %d us" % elapsed)
```

Use the editor's **Debugger → Profiler** tab for full frame analysis.

## _process / _physics_process optimization

### Disable when not needed

```gdscript
func _ready() -> void:
    # Start disabled — enable only when needed
    set_process(false)
    set_physics_process(false)

func start_moving() -> void:
    set_physics_process(true)

func stop_moving() -> void:
    set_physics_process(false)
```

### Reduce node count

Every node with `_process()` enabled propagates the call to children.
Remove nodes from the scene tree when inactive:

```gdscript
# Instead of hide() + pause — remove from tree (keep reference)
var cached_node: Node

func deactivate() -> void:
    remove_child(cached_node)   # no _process() calls

func activate() -> void:
    add_child(cached_node)      # restored
```

### Process priority

Set `process_priority` to control execution order (higher = later).
Group identical priorities to avoid overhead:

```gdscript
# In _ready()
process_priority = 100  # Execute after lower-priority nodes
```

### Move heavy work out of _process

```gdscript
# Bad — allocates array every frame
func _process(delta: float) -> void:
    var nearby: Array[Node2D] = []
    for body in get_tree().get_nodes_in_group("enemies"):
        if body.global_position.distance_to(global_position) < 200:
            nearby.append(body)

# Good — use Area2D with signals
func _on_detection_area_body_entered(body: Node2D) -> void:
    if body.is_in_group("enemies"):
        nearby_enemies.append(body)

func _on_detection_area_body_exited(body: Node2D) -> void:
    nearby_enemies.erase(body)
```

FORBIDDEN: `get_tree().get_nodes_in_group()` in `_process()` or `_physics_process()`.
FORBIDDEN: array allocations, `Vector2.distance_to()` on large sets every frame.
FORBIDDEN: `_process()` with empty body when not needed — disable it.

## Rendering optimization

### Draw call reduction

- Use `SpriteFrames` with sprite sheets — multiple frames on one texture = 1 draw call
- Combine small textures into atlas textures
- Set `visible = false` (not `modulate.a = 0`) for hidden sprites — Godot skips invisible nodes
- Use `CanvasItem.draw_*()` methods instead of many individual `Sprite2D` nodes for simple shapes

### Batching

Godot automatically batches `Sprite2D` nodes that:
- Share the same texture
- Share the same material
- Are consecutive in the scene tree
- Have no `modulate` / `self_modulate` differences

FORBIDDEN: interleaving sprites with different textures when they could be grouped.

### Visibility culling

Nodes outside the camera view are automatically culled.
Keep the scene tree shallow and wide — deep nesting adds culling overhead.

## Physics optimization

### Collision shape complexity

```gdscript
# Good — simple shapes
CollisionShape2D.shape = CircleShape2D  (cheapest)
CollisionShape2D.shape = RectangleShape2D (cheap)
CollisionShape2D.shape = CapsuleShape2D (cheap)

# Expensive — concave polygon with many vertices
CollisionShape2D.shape = ConcavePolygonShape2D (use only for static world)
```

### Physics tick rate

Default is 60 Hz. Lower for less responsive games:

```gdscript
# Project Settings → Physics → Common → Physics Ticks Per Second
Engine.physics_ticks_per_second = 30  # 2x performance gain, less responsive
```

When lowering physics rate, interpolate visual positions for smoothness.

### Disable sleeping bodies

```gdscript
var query: PhysicsPointQueryParameters2D = PhysicsPointQueryParameters2D.new()
query.exclude_sleep_bodies = true  # skip non-moving bodies
```

## Object pooling

For frequently created/destroyed objects (bullets, particles, enemies):

```gdscript
# object_pool.gd
class_name ObjectPool
extends Node

@export var scene: PackedScene
@export var initial_size: int = 20

var _pool: Array[Node] = []

func _ready() -> void:
    for i in initial_size:
        var obj: Node = scene.instantiate()
        obj.process_mode = Node.PROCESS_MODE_DISABLED
        obj.visible = false
        add_child(obj)
        _pool.append(obj)

func acquire() -> Node:
    for obj in _pool:
        if obj.process_mode == Node.PROCESS_MODE_DISABLED:
            obj.process_mode = Node.PROCESS_MODE_INHERIT
            obj.visible = true
            return obj

    # Pool exhausted — expand
    var obj: Node = scene.instantiate()
    add_child(obj)
    _pool.append(obj)
    return obj

func release(obj: Node) -> void:
    obj.process_mode = Node.PROCESS_MODE_DISABLED
    obj.visible = false
```

FORBIDDEN: `queue_free()` + `instantiate()` in rapid succession (bullets, particles) — use object pooling.

## Resource loading

```gdscript
# Background loading for large scenes/assets
var loader: ResourceLoader = ResourceLoader.load_threaded_request("res://large_level.tscn")

func _process(delta: float) -> void:
    var progress: Array = []
    var status: ResourceLoader.ThreadLoadStatus = ResourceLoader.load_threaded_get_status("res://large_level.tscn", progress)
    match status:
        ResourceLoader.THREAD_LOAD_LOADED:
            var scene: PackedScene = ResourceLoader.load_threaded_get("res://large_level.tscn")
            get_tree().change_scene_to_packed(scene)
```

FORBIDDEN: `load()` on large resources in `_ready()` without considering load time — use threaded loading or `preload`.

## Common pitfalls checklist

- [ ] All `_process()` / `_physics_process()` callbacks are disabled when idle
- [ ] No `get_tree().get_nodes_in_group()` inside frame callbacks
- [ ] Collision shapes are simple (circle/capsule/rectangle, not concave polygon)
- [ ] Bullets/particles use object pooling, not `instantiate()` + `queue_free()`
- [ ] Large scenes are loaded via `ResourceLoader.load_threaded_request()`
- [ ] `visible = false` used instead of `modulate.a = 0` for hidden sprites
- [ ] `process_priority` set for ordering-critical nodes
- [ ] Physics tick rate considered (can lower from 60 Hz if framerate is tight)
