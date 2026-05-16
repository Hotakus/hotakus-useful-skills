# 状态机模式

在 Godot 2D 游戏中实现角色状态机（State Machine）的三种模式。

## When to use

When implementing character states (idle, run, jump, attack, death), AI behaviors,
or animation-driven logic. Load this rule when `state machine`, `FSM`, or `enum State` is mentioned.

## Pattern 1: enum + match (推荐)

最简单的状态机，适合状态 ≤ 6 个的角色。

```gdscript
extends CharacterBody2D

enum State {IDLE, RUNNING, JUMPING, ATTACKING, HIT, DEAD}

var current_state: State = State.IDLE

func _physics_process(delta: float) -> void:
    match current_state:
        State.IDLE:
            _state_idle(delta)
        State.RUNNING:
            _state_running(delta)
        State.JUMPING:
            _state_jumping(delta)
        State.ATTACKING:
            _state_attacking(delta)
        State.HIT:
            _state_hit(delta)
        State.DEAD:
            _state_dead(delta)

func transition_to(new_state: State) -> void:
    current_state = new_state

func _state_idle(delta: float) -> void:
    # Apply gravity, check for input, transition
    if not is_on_floor():
        transition_to(State.JUMPING)
    elif Input.get_axis("move_left", "move_right") != 0:
        transition_to(State.RUNNING)
```

FORBIDDEN: huge `if/elif` chains instead of `match`.
FORBIDDEN: returning from `_physics_process()` inside a state handler — always let the match complete.

## Pattern 2: Node-based state machine

Each state is a `Node` child. Better for complex states with many methods.

```text
# StateMachine (Node)
#   ├── IdleState (Node)
#   ├── RunState (Node)
#   ├── JumpState (Node)
#   └── AttackState (Node)
```

```gdscript
# state.gd — base class
class_name State
extends Node

var parent: CharacterBody2D

func enter() -> void:
    pass

func exit() -> void:
    pass

func physics_update(delta: float) -> void:
    pass
```

```gdscript
# state_machine.gd
class_name StateMachine
extends Node

var current_state: State

func _ready() -> void:
    # Start with first child state
    for child in get_children():
        if child is State:
            child.parent = get_parent()
            change_state(child)
            break

func change_state(new_state: State) -> void:
    if current_state:
        current_state.exit()
    current_state = new_state
    current_state.enter()

func _physics_process(delta: float) -> void:
    if current_state:
        current_state.physics_update(delta)
```

FORBIDDEN: accessing sibling states directly — transitions go through `StateMachine.change_state()`.

## Pattern 3: AnimationTree StateMachine

When animation and logic states are tightly coupled, use `AnimationTree`'s built-in state machine.

```gdscript
@onready var anim_tree: AnimationTree = $AnimationTree
@onready var playback: AnimationNodeStateMachinePlayback = anim_tree.get("parameters/playback")

func travel_to(state_name: String) -> void:
    playback.travel(state_name)

func _ready() -> void:
    # Listen for transition end
    playback.start("Idle")

func _physics_process(delta: float) -> void:
    # Set blend tree parameters for smooth transitions
    anim_tree.set("parameters/IdleWalkRun/blend_position", direction.length())
    anim_tree.set("parameters/conditions/is_falling", not is_on_floor())

    # Trigger transitions via condition parameters
    if Input.is_action_just_pressed("attack"):
        anim_tree.set("parameters/conditions/attack_trigger", true)
```

Key `AnimationNodeStateMachinePlayback` methods:

| Method | Purpose |
|--------|---------|
| `travel(to_node)` | Smooth transition with blend |
| `start(node)` | Immediate switch, no blend |
| `stop()` | Stop playback |
| `get_current_node()` | Query current state name |

FORBIDDEN: calling `travel()` in `_process()` for states that drive `_physics_process()` logic — sync with physics tick.

## Choosing the right pattern

| Pattern | States | Logic complexity | Best for |
|---------|:---:|:---:|----------|
| enum + match | ≤ 6 | Simple | Player character, simple enemies |
| Node-based | 6+ | Complex, many methods per state | Bosses, complex AI |
| AnimationTree | 4–8 | Animation-driven | Animation-heavy characters |

## Transition validation

Always validate transitions — not every state should go to every other state:

```gdscript
func can_transition(from: State, to: State) -> bool:
    match from:
        State.DEAD:
            return false  # dead can't transition
        State.ATTACKING:
            return to == State.IDLE  # attack can only go to idle
        _:
            return true

func transition_to(new_state: State) -> void:
    if not can_transition(current_state, new_state):
        return
    current_state = new_state
```
