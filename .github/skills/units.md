# KKND Unit System

## Architecture: Three-Layer Object Model

Every game unit is composed of three linked objects:

```
UnitStats (static template)     ──→ defines WHAT the unit IS
    ↓ (unit->stats)
Unit (game logic object)        ──→ defines WHAT the unit DOES (state, orders, modes)
    ↓ (unit->entity, unit->task)
Entity (sprite/render object)   ──→ defines HOW it LOOKS and WHERE it IS
    ↓ (entity->rend)
RenderNode (draw job)           ──→ actual rendering pipeline node
```

### Linkage

```
Task.ctx         → Unit
Task.entity      → Entity
Unit.task        → Task
Unit.entity      → Entity
Unit.stats       → &g_unit_stats[unit->type]
Entity.task      → Task
Entity.ctx       → Unit (or other owner — Turret, OilPatch, etc.)
Entity.rend      → RenderNode (draw job in render list)
```

---

## UnitStats — Static Template (g_unit_stats[89])

One entry per `UnitType` enum value. Never modified at runtime. Defines:

| Field | Purpose |
|-------|---------|
| `mobd_id` | Which MOBD sprite bank to use |
| `task_fn` | The UNIT_Handler function (e.g. `UNIT_Handler_Infantry`) |
| `name` | Display name ("Rifleman", "War Mastadont") |
| `cost` | Build cost |
| `hitpoints` | Max HP |
| `speed` | Movement speed |
| `reload_time` | Weapon reload time |
| `turning_speed` | Rotation speed |
| `view_range` / `firing_range` | Detection/attack ranges |
| `accuracy` | Hit chance |
| `can_crush` | Can crush infantry |
| `is_infantry` | Infantry pathing rules (5 per tile) |
| `mobd_lookup_offset_attack/move/idle/4` | Animation set offsets |
| `attachment` | Turret definition (UnitAttachment*) |
| `projectile` | Projectile type |
| `size` | Map tile footprint (Small=512/Regular=128/Large=4096) |
| `race` | Survivors/Evolved |
| `ai_threat_weight` / `ai_strategic_value` | AI decision weights |
| `factory` | Which building produces this unit |
| `production_time` | Build time in ticks/60 |

---

## Entity — Sprite/Physics Layer

The "body" of any visible thing. NOT unit-specific — explosions, terrain objects, UI elements all use Entity.

**Key responsibilities:**
- Position (x, y, z) in world coords (13-bit tile = 8192 units per tile)
- Velocity (x_speed, y_speed, z_speed)
- Animation state (anim, anim_cursor, anim_current_frame, anim_speed, anim_timer)
- Collision (shape, collider, is_on_collision_grid)
- Rendering (rend → RenderNode → draw pipeline)
- CPLC link (cplc_meta, cplc_spawn_params — level placement data)

**Entity does NOT know about unit logic.** Its `ctx` field points back to owner, but Entity code doesn't interpret it. The rendering system only needs Entity + RenderNode.

---

## Unit — Game Logic Layer

The "brain". Only exists for gameplay-relevant actors (not decorations/particles).

**Pool:** 599 slots pre-allocated (`g_unit_pool`). Free list: `g_unit_free_head`. Active list: `g_unit_list_head/tail`.

**Key responsibilities:**
- Orders (order, order_target, order_next_waypoint_*)
- Pathfinding (scan_pathing, path_flags, ray_* arrays)
- Combat (locked_target, opportunity_target, hitpoints, veterancy)
- Mode functions (see below)
- Player ownership (player_num)
- Escort system (escort_list_head/tail)

---

## Task — Execution Context

Every Entity gets a Task. Task determines HOW the entity's logic runs:

| TaskKind | Execution Model | Used By |
|----------|----------------|---------|
| `TaskKind_Callback` | `entry_point(task)` called every frame by scheduler | Most combat units (infantry, vehicles, buildings) |
| `TaskKind_Coroutine` | Cooperative fiber, can `TASK_yield()` to suspend | OilPatch, Hut, explosions, camera shake, UI scripts, AI tasks |

**Critical insight:** Most unit handlers (UNIT_Handler_Infantry, etc.) are **Callbacks** — called every frame by the task scheduler. Some passive objects (OilPatch, Hut) are **Coroutines** — suspended fibers that yield in infinite loops. The `g_scripts[]` table determines which kind each handler uses.

---

## The Mode System — Cooperative State Machine

Instead of a traditional FSM with switch/case, KKND uses **function pointer swapping** on `unit->mode`.

### How it works

Every frame, the task scheduler calls `UNIT_Handler_Infantry(task)`, which does:
```c
unit->mode(unit);   // call current mode function
```

Mode functions are `void __fastcall fn(Unit *unit)` — they execute ONE TICK of behavior, then return. To transition states, they write a new function pointer:
```c
unit->mode = unit_mode_idle;            // transition to idle
unit->mode = unit_mode_attack_move;     // transition to attack-move
unit->mode = unit->mode_arrive;         // transition to stored "on arrival" callback
```

### Mode Slots

| Slot | Purpose | Set By |
|------|---------|--------|
| `mode` | **Current active mode** — called every frame | Any code that transitions state |
| `mode_idle` | Stored idle mode (rarely used directly) | Handler init |
| `mode_arrive` | **On-arrival callback** — where to go when movement/construction completes | Before starting move/build |
| `mode_attacked` | Reaction to being attacked (rarely used as field) | |
| `mode_return` | **Post-action return** — where to go after target dies or gets lost | Before starting attack/escort |
| `mode_turn_return` | **Turn-back mode** — building capture logic | Building init |

### The Pattern

```
[1] unit idle: mode = unit_mode_idle
[2] player orders move: mode = unit_mode_attack_move
     sets mode_return = unit_mode_415540 (idle setup)
[3] pathfinding kicks in: mode = unit_mode_417360 (straight-line move)
[4] arrives at waypoint: mode = unit->mode_return (→ idle setup)
[5] idle setup runs once: mode = unit_mode_idle (back to idle)
```

For buildings:
```
[1] construction starts: mode = unit_mode_402AB0 (construction anim)
     sets mode_arrive = specific_building_idle_mode
[2] construction message received: mode = unit->mode_arrive
[3] building is now active
```

### Why This Replaces FSM

- **No giant switch**: each state is its own function. Clean separation.
- **Composable**: `mode_return` and `mode_arrive` let you "chain" behaviors without the caller knowing what comes next.
- **Interruptible**: any code can swap `unit->mode` to hijack the unit instantly (e.g. `mode = unit_mode_destroyed`).
- **Stack-like nesting**: move-to-target sets `mode_return = previous_mode`, creating implicit call stack.

### Key Mode Functions (Infantry/Vehicle)

| Mode Function | Purpose |
|---|---|
| `unit_mode_415540` | Idle setup — snap to tile, set anim, then → `unit_mode_idle` |
| `unit_mode_idle` | Idle tick — regen HP, fidget timer, check escort |
| `unit_mode_attack_move` | Main movement/attack decision loop |
| `unit_mode_417360` | Straight-line movement (beeline to waypoint) |
| `unit_mode_416060` | Obstacle scanning movement (wall-following) |
| `unit_mode_destroyed` | Death sequence |
| `unit_mode_destroy` | Cleanup and remove |

### Key Mode Functions (Buildings)

| Mode Function | Purpose |
|---|---|
| `unit_mode_402AB0` | Under construction (waiting for completion messages) |
| `unit_mode_403650` | Building idle (play idle/damaged anim) |
| `unit_mode_4034B0` | Post-plant snap to grid |
| `unit_mode_building_on_death` | Explode, cleanup, free |
| `unit_mode_building_finalize` | Remove entity, terminate task, free unit |
| `unit_mode_beast_enclosure_on_completed` | Factory becomes operational |
| `unit_mode_blacksmith_on_completed` | Research building becomes operational |

---

## Coroutines vs Callbacks — The Two Execution Models

### Callback (TaskKind_Callback) — Most combat units

```
Scheduler iterates active task list every frame:
  → for each task where kind == Callback:
      task->entry_point(task)    // UNIT_Handler_Xxx
          → unit->mode(unit)    // one tick of current state
      // returns immediately
```

No fiber suspension. Handler runs, does one frame of work, returns.

### Coroutine (TaskKind_Coroutine) — Terrain objects, explosions, scripts, UI

```
Fiber with own stack. Can call TASK_yield(task, flags, sleep_ticks).
Yield suspends fiber mid-function. Scheduler resumes it later.
```

Used for things with **temporal flow** that's easier to express linearly:
```c
void __cdecl __noreturn task_explosion(Task *task) {
    // play explosion anim
    TASK_yield(task, Task_Sleep, 10);  // wait 10 frames
    // spawn debris
    TASK_yield(task, Task_Sleep, 20);  // wait 20 frames
    // remove entity
    entity_remove(task->entity);
    TASK_Terminate(task);
}
```

### Why Combat Units Use Callbacks, Not Coroutines

Combat units need to be **externally interruptible** every frame. Their handler does:
1. Call `unit->mode(unit)` — one tick
2. Check bounds
3. Scan for opportunity targets
4. Update order target tracking

A coroutine can't be externally interrupted without complex cancellation logic. The mode-pointer pattern gives instant preemption — any external code can just overwrite `unit->mode`.

Passive terrain objects (OilPatch, Hut) don't need this — they just sit there waiting for events, so coroutine with infinite yield loop is simpler.

### TASK_yield Behavior — Callbacks vs Coroutines

`TASK_yield` is a **polymorphic function** — same call, different behavior depending on `task->kind`.

#### TASK_yield implementation (verified from code):

```c
unsigned int TASK_yield(Task *task, TaskYieldFlags yield_flags, int sleep_ticks) {
    // ... message wake-up logic ...

    if (yield_flags & Task_Sleep) {
        if (sleep_ticks)
            task->sleep = sleep_ticks;
        // ... early exit if sleep==0 ...
    }

    task->flags_20 = 0;
    task->wait_flags = yield_flags;

    if (task->kind == TaskKind_Coroutine) {
        TASK_ExecuteAsync(g_coroutine_list_head);   // FIBER SUSPEND — switches to scheduler
        task->wait_flags = 0;                        // runs when resumed
        task->field_2C = 0;
    }
    // For Callbacks: just returns normally. No suspension.
    return task->flags_20;
}
```

#### Scheduler tick loop (verified from code):

```c
for (task = g_task_active_head; task != sentinel; task = task->next) {
    // 1. Decrement sleep counter
    if (task->sleep) {
        task->sleep--;
        if (task->sleep == 0)
            task->flags_20 |= 0x80000000;  // set "sleep expired" flag
    }

    // 2. Check if task should execute this frame
    wait_flags = task->wait_flags;
    if (!wait_flags || (wait_flags & task->flags_20) || ...)  {
        if (task->kind == TaskKind_Callback)
            TASK_ExecuteSync(task);          // task->entry_point(task)
        else
            TASK_ExecuteAsync(task->entry_point);  // resume fiber
    }
    // Otherwise: skip this task (sleeping or waiting)
}
```

#### For Coroutines (kind == 0):
`TASK_yield` calls `TASK_ExecuteAsync` which **switches fibers** — execution suspends mid-function and resumes at the same point later. True cooperative multitasking.

#### For Callbacks (kind == 1):
`TASK_yield` sets `task->wait_flags` and `task->sleep`, then **returns normally**. Code after the yield call **executes immediately, same frame**. The sleep counter prevents the scheduler from calling `entry_point` for N frames on subsequent ticks.

This means in `unit_destroy`:
```c
unit->hitpoints = 0;
TASK_yield(task, Task_Sleep, 1);   // sets sleep=1, returns immediately
unit->hitpoints = 0;               // executes NOW, same frame
unit->mode = next_mode;            // executes NOW, same frame
unit->destroyed = 1;               // executes NOW, same frame
// handler returns. Next frame: scheduler skips this task (sleep=1).
// Frame after: sleep expires, handler called again, runs new mode.
```

The yield does NOT pause execution — it marks the task to be **skipped for 1 frame** on the NEXT scheduler tick. All post-yield code runs immediately. This is a "schedule a gap" operation, not a suspension.

#### Some handlers ARE true Coroutines

Not all UNIT_Handlers are Callbacks. The `g_scripts[]` table (indexed by `TaskType`) stores `kind` per script. The CPLC level spawner reads it:
```c
v6 = g_scripts[cplc->task_type];
v8 = v6->kind ? TASK_CreateCallback(...) : TASK_CreateFiber(...);
```

**Confirmed Coroutine handlers** (declared `__cdecl __noreturn`, use `while(1) TASK_yield`):
- `UNIT_Handler_OilPatch` — spawned as `TaskKind_Coroutine` (line 29875)
- `UNIT_Handler_Hut` — `__noreturn`, eternal yield loop

**Confirmed Callback handlers** (use `unit->mode(unit)` pattern):
- `UNIT_Handler_Infantry`, `UNIT_Handler_General`, `UNIT_Handler_Bomber`
- All building handlers: Tower, Outpost, MachineShop, Clanhall, BeastEnclosure, etc.
- `UNIT_Handler_Tanker`, `UNIT_Handler_TankerConvoy`, `UNIT_Handler_MobileDerrick`

**Rule of thumb**: If handler has `__noreturn` and `while(1) TASK_yield` loop → Coroutine. If handler calls `unit->mode(unit)` and returns → Callback.

---

## Unit Creation Flow

```
entity_create_by_unit_type(type, x, y, player_num)
  → entity_create_ex(stats.mobd_id, NULL, stats.task_fn, TaskKind_Callback, NULL)
      → TASK_CreateCallback(TaskChannel_None, task_fn)  // creates Task, links to active list
      → entity_create(mobd_id, task, NULL)              // allocs Entity from pool, links to task
      → returns Entity with task->entity = entity, entity->task = task

First frame: UNIT_Handler_Infantry(task) called by scheduler
  → task->ctx == NULL (no unit yet!)
  → unit = unit_create(task)           // alloc Unit from pool
  → unit_init_default(unit)            // set position, register on collision grid
  → unit_turret_init(unit)             // create turret sub-entity if needed
  → unit_rendering_default(unit)       // setup healthbar render node
  → unit->mode(unit)                   // first mode tick
```

---

## Unit Destruction Flow

```
unit->hitpoints reaches 0 (or sabotage succeeds)
  → unit->mode = on_death function
    → on_death plays explosion, broadcasts messages
    → sets unit->destroyed = 1, unit->unit_id = 0
    → eventually: entity_remove(entity), TASK_Terminate(task), unit_44C890(unit)

unit_44C890(unit):
  → unit->unit_id = 0
  → render_node disabled
  → unlink from active list (next/prev surgery)
  → push onto free list (unit->next = g_unit_free_head; g_unit_free_head = unit)
```

---

## Message System

Tasks communicate via messages. Two delivery modes:

1. **Direct** (`TASK_send_message`): If target is Callback → immediate `message_handler()` call. If Coroutine → queue message, set wake flag.
2. **Broadcast** (`TASK_broadcast_message`): Send to all tasks on a `TaskChannel`.

Key channels:
- `TaskChannel_Units` — all units
- `TaskChannel_UnitLifecycle` — creation/destruction notifications
- `TaskChannel_Tanker`, `TaskChannel_Drillrig`, `TaskChannel_PowerPlant` — resource chain

For Callback tasks, `message_handler` is called inline during send — **synchronous delivery**. This is how buildings learn about construction progress, attacks, etc.

---

## Naming Conventions & Decompilation Notes

| Pattern | Meaning |
|---|---|
| `UNIT_Handler_Xxx` | Task entry point / per-frame handler |
| `unit_mode_XXXXXX` | Mode function (unnamed — address as name) |
| `unit_mode_xxx_name` | Mode function (named — we figured it out) |
| `MessageHandler_Xxx` | Message callback |
| `unit_init_default` | First-frame setup for all units |
| `unit_building_init` | First-frame setup for buildings |
| `unit_turret_init` | Create turret sub-entity |
| `unit_create` | Alloc Unit from pool, basic init |
| `unit_44C890` | Free Unit back to pool (unnamed! → suggest `unit_free` or `unit_destroy_final`) |
| `entity_create_ex` | Create Entity+Task, link together |
| `entity_create_by_unit_type` | Convenience: create entity from UnitType index |

See [units-todo.md](units-todo.md) for naming suggestions, decompilation mistakes, and open questions.

---

## Pool Summary

| Object | Pool Size | Pool Var | Free Var | Active Head/Tail |
|---|---|---|---|---|
| Unit | 599 | `g_unit_pool` | `g_unit_free_head` | `g_unit_list_head` / `g_unit_list_tail` |
| Entity | 2000 | `g_entity_pool` | `g_entity_free_pool_head` | `g_entity_head` / `g_entity_tail` |
| Task | 2000 | `g_tasks` | `g_task_terminated_head` | `g_task_active_head` / `g_task_active_tail` |
| Coroutine | 2000 | `g_coroutine_pool` | `dword_477350` | `g_coroutine_list_head` |
| Escort | 500 | `g_escort_pool` | `g_escort_free_list_head` | `g_escort_active_list_head/tail` |
