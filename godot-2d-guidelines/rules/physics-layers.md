# 物理层与碰撞设计

Godot 2D 物理层（Layers）和掩码（Masks）的设计策略与配置规范。

## When to use

When configuring `collision_layer` / `collision_mask` on physics bodies, designing
hitbox/hurtbox systems, or setting up `RayCast2D` queries. Load this rule when
`collision_layer`, `collision_mask`, `physics layer`, or `layer design` is mentioned.

## Core concepts

- **Layer** (`collision_layer`): what this body **IS** — which layer(s) it belongs to
- **Mask** (`collision_mask`): what this body **DETECTS** — which layer(s) it checks against

A collision happens when: `bodyA.collision_layer & bodyB.collision_mask != 0` **AND** `bodyB.collision_layer & bodyA.collision_mask != 0`

## Layer assignment strategy

Assign layers by **entity category**, not by individual object:

```text
Layer 1: World geometry (walls, floors, platforms)
Layer 2: Player
Layer 3: Enemies
Layer 4: Player projectiles
Layer 5: Enemy projectiles
Layer 6: Pickups / items
Layer 7: Triggers / detection zones
Layer 8: Moving platforms
```

### What each entity sets

| Entity | `collision_layer` | `collision_mask` |
|--------|:---:|:---:|
| Player | 2 | 1 + 3 + 5 + 6 + 7 + 8 |
| Enemy | 3 | 1 + 2 + 4 |
| Player bullet | 4 | 3 |
| Enemy bullet | 5 | 2 |
| Pickup | 6 | 2 |
| World wall | 1 | — (or all if needed) |

## Code pattern: human-readable layer indices

```gdscript
extends CharacterBody2D

# Export layer INDICES (1-based, human-readable)
@export var body_layer: int = 2      # "player"
@export var detect_layers: Array[int] = [1, 3, 5, 6, 7, 8]

func _ready() -> void:
    # Convert index to bit value
    collision_layer = 1 << (body_layer - 1)

    # Convert mask array to bitmask
    var mask: int = 0
    for idx in detect_layers:
        mask |= 1 << (idx - 1)
    collision_mask = mask
```

FORBIDDEN: hardcoding raw bit values (`collision_layer = 4`). Use the index pattern — it's self-documenting.

## Hitbox / Hurtbox pattern

```gdscript
# Hurtbox (Area2D) — receives damage
# collision_layer: none (doesn't need to "be" anything)
# collision_mask: layers that can hit this entity
extends Area2D

func _ready() -> void:
    # ONLY detect enemy projectiles and enemy bodies
    collision_mask = (1 << 4) | (1 << 2)  # layer 5 + layer 3
    area_entered.connect(_on_area_entered)

func _on_area_entered(area: Area2D) -> void:
    if area is Hitbox:
        get_parent().take_damage(area.damage)
```

```gdscript
# Hitbox (Area2D) — deals damage
# collision_layer: identify what this IS (e.g., "player weapon")
# collision_mask: none (doesn't detect — the hurtbox detects it)
extends Area2D

@export var damage: int = 10

func _ready() -> void:
    collision_layer = 1 << 6  # layer 7: "weapon hitboxes"
    collision_mask = 0        # passive — detected by hurtboxes
```

## RayCast2D queries

```gdscript
@onready var ray: RayCast2D = $RayCast2D

func _ready() -> void:
    # Only collide with world geometry
    ray.collision_mask = 1 << 0  # layer 1

func _physics_process(delta: float) -> void:
    if ray.is_colliding():
        var collider: Object = ray.get_collider()
```

For manual physics queries:

```gdscript
var query: PhysicsPointQueryParameters2D = PhysicsPointQueryParameters2D.new()
query.position = target_pos
query.collision_mask = 1 << 0  # only layer 1
query.collide_with_bodies = true
query.collide_with_areas = false

var space_state: PhysicsDirectSpaceState2D = get_world_2d().direct_space_state
var results: Array[Dictionary] = space_state.intersect_point(query)
```

FORBIDDEN: not setting `collision_mask` on `RayCast2D` — it defaults to all layers, causing false positives.
FORBIDDEN: using `collision_mask = 0` when you need the ray to actually collide with something.

## Performance notes

- Each layer checked costs CPU — don't put all entities on the same layer
- Use `collision_mask = 0` on bodies that should never collide (purely visual)
- Disable `monitoring` on `Area2D` when not needed
- Use `collision_priority` to resolve overlap conflicts deterministically

## Common pitfalls

FORBIDDEN: thinking layer/mask is symmetric. `bodyA.mask` must include `bodyB.layer` AND `bodyB.mask` must include `bodyA.layer` for two-way collision.
FORBIDDEN: assigning layers by individual object ("sword layer 3", "spear layer 4"). Group by category.
FORBIDDEN: using layers 1-20 without a documented plan. Document the layer map in a project README or constants file.
