# KKND Entity System

Entity is the universal "body" of any visible object in the game — units, buildings, projectiles, explosions, terrain objects, UI elements, cursors, menu buttons. It handles position, velocity, animation, collision, and rendering. It does NOT know about game logic — that's the Task and Unit layers above it.

---

## Struct Layout ([kknd.h](kknd.h#L1345))

```c
struct KKND::Entity {
  Entity *next, *prev;                // active list / free pool linkage
  Entity *parent;                     // parent entity (turret→unit, projectile→turret)
  MobdId mobd_id;                     // sprite/animation bank index
  int x, y, z;                        // position (8.8 fixed-point, 8192 units per tile)
  int x_speed, y_speed, z_speed;      // velocity per tick
  int x_acceleration, y_acceleration, z_acceleration;
  int x_speed_limit, y_speed_limit, z_speed_limit;  // max speed per axis (default 0x7FFFFFFF)
  int x_drag, y_drag, z_drag;         // friction toward zero
  MobdAnimation *anim;                // current animation data
  MobdAnimFrame **anim_cursor;        // current position in frame pointer array
  MobdAnimFrame *anim_current_frame;  // dereferenced frame for current display
  BoxdCollisionShape *shape;          // collision shape (from current anim frame)
  BoxdCollisionHandler *collider;     // collision dispatch table
  int anim_speed;                     // ticks per frame advance
  int anim_timer;                     // countdown; when ≤0 advance frame
  RenderNode *rend;                   // rendering pipeline node
  Task *task;                         // cooperative task driving this entity
  CplcSpawnParams *cplc_spawn_params; // non-NULL = placed from level CPLC data
  CplcEntity *cplc_meta;              // back-pointer to CPLC entity record
  CplcEntityInViewport *cplc_view;    // viewport tracking node
  void *ctx;                          // POLYMORPHIC context (Unit*, Turret*, projectile params, packed type|player)
  void *_80_attacker_unit_or__stru29__sprite__initial_hitpoints;  // OVERLOADED tagged field
  int _80_unit_id;                    // stale-pointer validation (paired with _80_)
  BOOL is_on_collision_grid;          // collision grid dirty flag
  __int16 infantry_damage;            // projectile damage vs infantry
  __int16 vehicle_damage;             // projectile damage vs vehicles
  __int16 building_damage;            // projectile damage vs buildings
  __int16 _92_padding;               // NOT padding — used as repair counter in building code
};
```

Total size: 0x94 bytes (148 bytes). Pool: 2000 entities = `0x48440` bytes.

---

## Pool Lifecycle

### Initialization — `LVL_InitMobd()` ([kknd.c](kknd.c#L39240))

- `malloc(0x48440)` → `g_entity_pool` — 2000 contiguous Entity structs
- **Free list**: singly-linked via `->next`. Head: `g_entity_free_pool_head`. Sentinel: `&g_entity_free_pool_head`.
- **Active list**: doubly-linked circular via `->next/->prev`. Sentinel: `&g_entity_head`.
- Template `g_default_entity` initialized (all zeroed except speed_limits = `0x7FFFFFFF`, collider = `g_unit_collision_handlers`)

### Allocation — `entity_create()` ([kknd.c](kknd.c#L39305))

1. Pop from `g_entity_free_pool_head`
2. `REND_AddNode(entity, entity_render)` → allocate render node
3. `qmemcpy` from `g_default_entity` (template copy)
4. Append to active list tail
5. Set `mobd_id`, `task`, `rend`. Link `task->entity = entity`.
6. If `parent`: inherit position (and task if none provided)

### Extended Creation — `entity_create_ex()` ([kknd.c](kknd.c#L39345))

Creates Task first (fiber or callback based on `task_kind`), then calls `entity_create()`. Optionally offsets position by `parent_anchor` (MobdPoint).

### Creation from Unit Stats — `entity_create_by_unit_type()` ([kknd.c](kknd.c#L72932))

Bridges `g_unit_stats[type]` → Entity:
```c
result = entity_create_ex(stats.mobd_id, NULL, stats.task_fn, TaskKind_Callback, NULL);
result->x = x; result->y = y;
result->ctx = (void*)(type | (player_num << 16));  // packed
```

Used for spawning buildings from mobile units. The building handler unpacks `ctx` to get type+player.

### Removal — `entity_remove()` ([kknd.c](kknd.c#L39372))

1. Mark render node for deletion (`rend->flags |= 0x80000000`)
2. Clear CPLC back-references
3. Unlink from active list (prev/next surgery)
4. Push onto free list head

### `sub_426F90` — Entity Remove + Task Terminate ([kknd.c](kknd.c#L39395))

Like `entity_remove()` but ALSO calls `task_445310(task)` to terminate associated task. Used when entity owns its task and both die together. `entity_remove()` is for cases where the task handles its own cleanup.

### Teardown — `MOBD_Deinit()` ([kknd.c](kknd.c#L39935))

Just `free(g_entity_pool)`. No per-entity cleanup — the entire 2000-entity slab is freed at once.

---

## Animation System

### Data Model

```
MobdAnimation {
    int anim_speed;           // default ticks per frame
    MobdAnimFrame *frames[];  // pointer array: frames, NULL=end, -1=loop
}

MobdAnimFrame {
    int x, y;                 // sprite render offset
    int flags;
    MobdSprtImage *sprt;      // the image to draw
    BoxdCollisionShape *shape; // collision shape this frame
    SoundId sound_id;         // sound to play when frame displays
    MobdPoint points[];       // anchor points (for events, projectile origins)
}
```

### How `anim`, `anim_cursor`, `anim_current_frame` Work

- **`anim`** — base pointer to `MobdAnimation` struct. The frame array starts at offset 4 (`anim->frames[0]`).
- **`anim_cursor`** — iterator into frame pointer array. Initialized to `(MobdAnimFrame**)anim` (points at `anim_speed` field). First `entity_anim_tick` call advances past it to `frames[0]`.
- **`anim_current_frame`** — `*anim_cursor`, the frame data currently displayed and used for collision shape.

### Per-Frame Tick — `entity_anim_tick()` ([kknd.c](kknd.c#L39828))

1. If `anim_cursor == NULL` → skip
2. Subtract `anim_speed` from `anim_timer`
3. If `anim_timer < 0` (high bit set) → advance cursor:
   - Next frame `NULL` → **animation ended**. Set `task->flags_20 |= 0x10000000`. Null anim. Return.
   - Next frame `-1` → **loop**. Reset cursor to `anim->frames`. Set flags.
   - Otherwise → advance cursor. Set `anim_current_frame = *cursor`, update `shape`, set `is_on_collision_grid = 1`.
   - If frame has `sound_id` → `entity_play_sound`
   - If frame has `points[0].id` and task waits on `0x40000` → send anchor point event

### Animation Functions

| Function | Line | Purpose |
|---|---|---|
| `entity_anim_load` | [kknd.c](kknd.c#L39679) | Load anim from MOBD offset. Reset cursor, set timer=-1 (advance immediately). |
| `entity_anim_start` | [kknd.c](kknd.c#L39699) | Load anim by index within MOBD sub-table (offset + 4*idx). |
| `entity_anim_advance` | [kknd.c](kknd.c#L39720) | **Switch anim base, preserve cursor offset.** Used for direction changes mid-animation. |
| `entity_anim_switch_direction` | [kknd.c](kknd.c#L39772) | Like `anim_advance` but selects by index. |
| `entity_anim_clear` | [kknd.c](kknd.c#L39822) | Null all anim fields. Stop animation. |
| `entity_anim_tick` | [kknd.c](kknd.c#L39828) | Per-frame cursor advance. |
| `GAME_AdvanceEntityAnimations` | [kknd.c](kknd.c#L39920) | Iterate all entities, call `entity_anim_tick`. Respects multiplayer sync. |

---

## Physics System — `GAME_AdvanceEntityPositions()` ([kknd.c](kknd.c#L39460))

Called once per game tick. Processes every active entity. Three identical axis pipelines (X, Y, Z):

### Per-Axis Pipeline

1. **Position integration**: `entity->x += entity->x_speed`
2. **Collision detection**: If speed ≠ 0, calls `BOXD_collide_entity()` for that axis
3. **Drag (friction)**: Decelerates speed toward zero:
   - `speed > 0` → `speed -= drag`, clamp to 0
   - `speed < 0` → `speed += drag`, clamp to 0
4. **Acceleration + speed limiting**:
   - `speed += acceleration`
   - If `|speed| > speed_limit` → clamp to ±speed_limit, set task flag

### Task Flags on Speed Limit

| Axis | Flag | Meaning |
|---|---|---|
| X | `0x08000000` | X speed limit reached |
| Y | `0x04000000` | Y speed limit reached |
| Z | `0x02000000` | Z speed limit reached |

Set on `task->flags_20` and OR'd into `task->flags_24`. Tasks can wake on these to detect cruising speed.

### Collision Integration

- `BOXD_rebuild_collision_grid()` ([kknd.c](kknd.c#L9011)) — called at start of physics tick. Rebuilds spatial hash grid from entity shapes.
- `BOXD_collide_entity()` ([kknd.c](kknd.c#L9157)) — per-entity AABB check against grid. Calls mover/obstacle response callbacks.
- If neither X nor Y moved, collision still runs for `is_on_collision_grid` dirty flag update.

---

## Collision System

### `entity->shape`

Pulled from `anim_current_frame->shape` every time animation advances. Contains one or more `BoxdCollisionBox` entries defining physical footprint. Null when animation is cleared or entity has no collision.

### `entity->collider`

Dispatch table indexed by `BoxdCollisionType`. Each entry: `{collides_with_categories, category, mover_response, obstacle_response}`.

| Table | Purpose | Assigned By |
|---|---|---|
| `g_unit_collision_handlers[20]` | Default — units, buildings. Handles solid walls, floors, ramps, slopes, corners, cursor. | `g_default_entity` template |
| `g_ui_collision_handlers` | UI entities — menu screens, buttons | Overwritten on `g_default_entity.collider` before UI entity creation |
| `g_null_collision` | Zero-initialized — disables all collision | Projectiles, dead entities, special objects |

---

## Rendering Pipeline

### Entity → RenderNode → Draw

1. **`entity_create()`** calls `REND_AddNode(entity, entity_render)` → allocates `RenderNode`, links to staging draw list
2. **`RenderNode`** holds: `payload` (Entity*), `render_state_handler` (fn ptr), `cmd` (RenderCommand: x, y, z, image, viewport, palette, flags)
3. **Render state handler** called each frame to update `cmd` from entity state:

| Handler | Where Set | Z-Sorting |
|---|---|---|
| `render_state_handler_entity` | `LVL_InitMobd` ([kknd.c](kknd.c#L39252)) | Flat: `z = entity->z` |
| `render_state_handler_default` | Menu/escort init ([kknd.c](kknd.c#L72582)) | Isometric: `z = (entity->z + entity->y) >> 8` |

Both convert entity world position to screen coords: `screen_x = (entity->x >> 8) - (camera->x >> 8) - frame->x`

4. **`REND_DoFrame()`** — calls all render state handlers, sorts by Z, merges lists, draws to screen
5. **Removal**: `rend->flags |= 0x80000000` marks for deletion

### Specialized renderers override `render_state_handler` on their RenderNode:
- `render_state_handler_aircraft_turret`, `render_state_handler_buildings`, `render_state_handler_explosion`

---

## `ctx` Field — Polymorphic Context

| Context | Stored Type | Pattern |
|---|---|---|
| **Unit entity** | `Unit*` | `unit->entity->ctx = unit` |
| **Turret entity** | `Turret*` | `turret->entity->ctx = turret` |
| **Projectile entity** | `UnitProjectileType*` | Projectile params struct. `entity->parent->ctx` chains to firing unit. |
| **entity_create_by_unit_type** | Packed `type \| (player_num << 16)` | Unpacked by building handler for type+ownership. |
| **Explosion** | Params pointer or NULL | Stack-allocated or heap explosion params |

---

## `_80_` Fields — Tagged Union

`_80_attacker_unit_or__stru29__sprite__initial_hitpoints` + `_80_unit_id` form a **tagged pair** with three usage contexts:

| Context | `_80_` stores | `_80_unit_id` stores |
|---|---|---|
| Projectile/turret fire | `Unit*` (attacking unit) | `unit->unit_id` (stale-pointer check) |
| Building spawn (outpost/drillrig) | `(void*)hitpoints` (initial HP as int) | Unused (0) |
| UI/Menu entities | Sprite/stru29 pointer | Unused |

### Stale-pointer validation:
```c
attacker = (Unit*)entity->_80_...;
if (attacker && attacker->unit_id == entity->_80_unit_id) { /* still valid */ }
```

Units can die and memory gets reused. The `unit_id` check ensures the pointer still refers to the original unit.

---

## `parent` Field — Parent-Child Relationships

| Relationship | Created By |
|---|---|
| Turret → Unit | `entity_create_ex(..., parent=unit->entity)` |
| Projectile → Turret | `entity_create_ex(..., parent=turret->entity)` |
| Child entity → Spawner | Building sub-entities, upgrade overlays |

In `entity_create()`: if parent exists, `x/y/z = parent->x/y/z`. If no task provided, inherits parent's task.

Projectiles use `entity->parent->ctx` to reach the **firing unit** through the turret→unit chain.

---

## Damage Fields (Projectiles)

`infantry_damage`, `vehicle_damage`, `building_damage` — `__int16` fields set on **projectile entities** only.

### Set: base + veterancy bonus
```c
entity->infantry_damage = LOWORD(projectile->damage_infantry) + (base * g_veterancy_mod[vet] >> 8);
```

### Read: target-type dispatch
```c
if (target_is_infantry)       damage = attacker->infantry_damage;
else if (target_has_speed)    damage = attacker->vehicle_damage;   // mobile
else                          damage = attacker->building_damage;  // static
```

---

## Positional Sound — `entity_408800_play_sound()` ([kknd.c](kknd.c#L12417))

Computes pan (0–32) from entity X relative to camera. Attenuates volume by distance from viewport center. Calls `SOUND_play()`.

**Suggested rename**: `entity_play_sound`

---

## Distance Helpers

| Function | Line | Method | Returns |
|---|---|---|---|
| `entity_is_in_radius` | [kknd.c](kknd.c#L16789) | Squared Euclidean `((dy>>8)² + (dx>>8)²) <= radius²` | BOOL |
| `entity_40D8B0` | [kknd.c](kknd.c#L16800) | Tile-based spatial query — iterates tiles in square, checks unit lists | int |

---

## `g_default_entity` — Mutable Template

Initialized in `mobd_init_default_entity()` ([kknd.c](kknd.c#L39267)). Copied to every new entity via `qmemcpy`.

**WARNING**: `g_default_entity.collider` is **mutated globally** before UI entity creation (set to `g_ui_collision_handlers`), then entities created in that context inherit UI collision. This is implicit global state coupling.

---

## Save/Load — `EntitySaveStruct` ([kknd.h](kknd.h#L3981))

Serializable subset of Entity:

| Field | Purpose |
|---|---|
| `mobd_id` | Sprite bank |
| `x`, `y`, `z_index` | Position |
| `x_speed`, `y_speed`, `z_speed` | Velocity |
| `mobd_offset` | MOBD-relative pointer for anim restoration |
| `_54_inside_mobd_ptr4` | Secondary MOBD pointer offset |
| `anim_speed` | Animation playback speed |

The save system converts runtime pointers to MOBD-relative offsets.

---

## Global Variables

| Variable | Type | Purpose |
|---|---|---|
| `g_entity_pool` | `Entity*` | Base of 2000-entity slab allocation |
| `g_entity_free_pool_head` | `Entity*` | Free list head (singly-linked) |
| `g_entity_head` / `g_entity_tail` | `Entity*` | Active list head/tail (doubly-linked, sentinel-based) |
| `g_default_entity` | `Entity` | Template struct copied to new entities |
| `g_current_lvl_mobd` | `LevelMobd*` | Parsed MOBD section from level file |
| `g_current_lvl_mobd_verified` | `int` | 1 if MOBD loaded and pool initialized |
| `entity_render` | `RenderStateUpdater` | Function pointer: `render_state_handler_entity` or `render_state_handler_default` |
| `g_unit_collision_handlers[20]` | `BoxdCollisionHandler[]` | Default collision dispatch table |
| `g_ui_collision_handlers` | `BoxdCollisionHandler[]` | UI collision dispatch table |
| `g_null_collision` | `BoxdCollisionHandler` | No-collision sentinel |
| `g_tile_collisions` | `BoxdCollisionBucket**` | Spatial hash grid for collision |
| `g_tile_collisions_pool` / `_head` | `BoxdCollisionBucket*` | Pool allocator for collision buckets |

---

## Function Index

### Creation / Destruction

| Function | Line | Purpose |
|---|---|---|
| `LVL_InitMobd` | [kknd.c](kknd.c#L39240) | Allocate entity pool, init free list, setup template |
| `MOBD_Deinit` | [kknd.c](kknd.c#L39935) | Free entire entity pool slab |
| `mobd_init_default_entity` | [kknd.c](kknd.c#L39267) | Initialize `g_default_entity` template |
| `entity_create` | [kknd.c](kknd.c#L39305) | Allocate entity from pool, link to active list |
| `entity_create_ex` | [kknd.c](kknd.c#L39345) | Create entity + task together |
| `entity_create_by_unit_type` | [kknd.c](kknd.c#L72932) | Create entity from UnitStats table |
| `entity_remove` | [kknd.c](kknd.c#L39372) | Remove entity, return to pool |
| `sub_426F90` | [kknd.c](kknd.c#L39395) | Remove entity + terminate task (→ rename `entity_remove_with_task`) |

### Animation

| Function | Line | Purpose |
|---|---|---|
| `entity_anim_load` | [kknd.c](kknd.c#L39679) | Load animation from MOBD offset |
| `entity_anim_start` | [kknd.c](kknd.c#L39699) | Start animation by index |
| `entity_anim_advance` | [kknd.c](kknd.c#L39720) | Switch anim base, preserve cursor offset |
| `entity_anim_switch_direction` | [kknd.c](kknd.c#L39772) | Switch direction by index, preserve cursor |
| `entity_anim_clear` | [kknd.c](kknd.c#L39822) | Stop animation, null fields |
| `entity_anim_tick` | [kknd.c](kknd.c#L39828) | Per-frame animation advance |
| `GAME_AdvanceEntityAnimations` | [kknd.c](kknd.c#L39920) | Tick all entity animations per game frame |

### Physics / Collision

| Function | Line | Purpose |
|---|---|---|
| `GAME_AdvanceEntityPositions` | [kknd.c](kknd.c#L39460) | Physics tick: velocity, acceleration, drag, collision |
| `BOXD_rebuild_collision_grid` | [kknd.c](kknd.c#L9011) | Rebuild spatial hash each tick |
| `BOXD_collide_entity` | [kknd.c](kknd.c#L9157) | Per-entity collision resolution |

### Queries / Helpers

| Function | Line | Purpose |
|---|---|---|
| `entity_find_by_mobd_id` | [kknd.c](kknd.c#L39430) | Linear search active list by mobd_id |
| `entity_is_in_radius` | [kknd.c](kknd.c#L16789) | Squared Euclidean 2D distance check |
| `entity_40D8B0` | [kknd.c](kknd.c#L16800) | Tile-based spatial proximity query |
| `entity_408800_play_sound` | [kknd.c](kknd.c#L12417) | Positional sound playback |

### Rendering

| Function | Line | Purpose |
|---|---|---|
| `render_state_handler_entity` | [kknd.c](kknd.c#L39206) | Default entity renderer (flat Z) |
| `render_state_handler_default` | [kknd.c](kknd.c#L72668) | Menu entity renderer (isometric Z) |
| `REND_AddNode` | [kknd.c](kknd.c#L57999) | Allocate render node for entity |

---

## Cross-References

- [units.md](units.md) — Unit struct built on top of Entity (three-layer model)
- [task.md](task.md) — Task system driving entity logic
- [cplc.md](cplc.md) — CPLC spawn system (entity->cplc_spawn_params, cplc_meta, cplc_view)
- [object-lifecycle.md](object-lifecycle.md) — Entity pool as part of broader pool system
