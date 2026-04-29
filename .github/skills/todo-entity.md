# Entity System — TODO: Renames, Issues, Suggestions

## Function Renames

| Current Name | Suggested Name | Line | Reason |
|---|---|---|---|
| `sub_426F90` | `entity_remove_with_task` | [kknd.c](kknd.c#L39395) | Removes entity AND terminates its task. Unlike `entity_remove` which only recycles entity. Raw address name. |
| `entity_408800_play_sound` | `entity_play_sound` | [kknd.c](kknd.c#L12417) | Address in name. Plays positional sound for entity. |
| `entity_40D8B0` | `entity_find_nearby_in_tiles` | [kknd.c](kknd.c#L16800) | Address in name. Tile-radius spatial query. |
| `task_445310` | `task_terminate_immediate` | [kknd.c](kknd.c#L39401) | Called from `sub_426F90`. Address name. Terminates task. |
| `mobd_init_default_entity` | `entity_init_default_template` | [kknd.c](kknd.c#L39267) | Prefixed `mobd_` but operates on `g_default_entity`. Entity prefix more accurate. |
| `LVL_InitMobd` | — keep but document | [kknd.c](kknd.c#L39240) | Name implies only MOBD init, but also allocates entity pool. Could be `LVL_InitMobd_and_EntityPool` but that's ugly. Document dual purpose instead. |
| `render_state_handler_default` | `render_state_handler_entity_isometric` | [kknd.c](kknd.c#L72668) | "default" is confusing when there's also `render_state_handler_entity`. One is flat Z, one is isometric Z. |

---

## Global Variable Renames

| Current Name | Suggested Name | Reason |
|---|---|---|
| `entity_render` | `g_entity_render_handler` | Global function pointer, should follow `g_` convention |

---

## Struct Field Issues

### `_80_attacker_unit_or__stru29__sprite__initial_hitpoints`

**Problem**: Name is a concatenation of all observed usages. Unreadable. 56 characters.

**Proposal**: Rename to `_80_context_ptr` or make it a proper union:
```c
union {
    void *raw;
    Unit *attacker_unit;       // projectile/turret context
    int initial_hitpoints;     // building spawn context (cast from void*)
    void *ui_sprite;           // UI/menu context
} _80;
```

But since this is decompiled code, the simpler rename to `attacker_or_hp` or even just `_80_data` with a comment block documenting the 3 usages is more practical.

### `_80_unit_id`

**Problem**: Only meaningful when `_80_` stores a `Unit*`. Should be documented as stale-pointer validator.

**Suggested rename**: `_80_validation_id` — makes the purpose clearer.

### `_92_padding`

**Problem**: Named "padding" but actively used as a **repair iteration counter** in building code ([kknd.c](kknd.c#L26428)).

**Suggested rename**: `repair_counter` or `_92_counter`. Need more investigation to see if other usages exist.

**TODO**: Grep for all reads/writes to offset 0x92 in Entity struct to confirm full usage.

### `ctx` as `void*`

**Problem**: No type safety. Every usage requires casting.

**Not fixable** without major refactoring in decompiled code. Document the type table (done in entity.md).

---

## Decompilation Issues

### 1. `anim_cursor` initialization points to wrong type

**Location**: `entity_anim_load` ([kknd.c](kknd.c#L39679))
```c
entity->anim_cursor = (MobdAnimFrame **)anim;  // anim is MobdAnimation*
```

**Issue**: `anim_cursor` is typed as `MobdAnimFrame**` but initialized to point at `MobdAnimation` struct, which starts with `int anim_speed`. The first `entity_anim_tick` call advances past it to `frames[0]`. The cast is technically wrong for the initial value — it points at an int, not a `MobdAnimFrame*`.

**Not a bug** — the code is correct at runtime since the cursor is always advanced before dereferencing. But it confuses anyone reading the struct types.

### 2. `g_default_entity.collider` global mutation

**Location**: [kknd.c](kknd.c#L35456), [kknd.c](kknd.c#L35603), [kknd.c](kknd.c#L35766)
```c
g_default_entity.collider = g_ui_collision_handlers;  // before UI entity creation
```

**Issue**: The template is globally mutated to give UI entities different collision behavior. If not restored, gameplay entities created after would get wrong colliders. **Implicit coupling** — relies on strict ordering of entity creation contexts.

**TODO**: Verify that `g_default_entity.collider` is always restored to `g_unit_collision_handlers` before gameplay entity creation begins.

### 3. `sub_426F90` returns `Entity*` (prev pointer) — unusual

**Location**: [kknd.c](kknd.c#L39395)

The function returns `this->prev` — used for iterator-safe removal during list traversal. Caller gets the previous entity to continue iteration. This is intentional but undocumented.

### 4. `entity_create_ex` — LABEL_8 spaghetti

**Location**: [kknd.c](kknd.c#L39345-L39370)

The decompiler generated a `goto LABEL_8` flow. Clean version:
```c
if (task_entry) {
    task = task_kind ? TASK_CreateCallback(None, entry) : TASK_CreateFiber(None, entry, 0);
    if (!task) return NULL;
} else {
    task = NULL;
}
entity = entity_create(mobd_id, task, parent);
if (entity && parent && parent_anchor) {
    entity->x = parent_anchor->x + parent->x;
    // etc.
}
return entity;
```

---

## Missing Investigations

### TODO: Verify `is_on_collision_grid` semantics

Set to 1 in many places: animation tick, physics tick, entity creation. Appears to be a dirty flag for collision grid, but writes seem scattered. Investigate whether it's a genuine dirty flag or has other semantics.

### TODO: Trace all users of `entity_find_by_mobd_id`

Linear scan of 2000-entity list — potentially expensive. Currently only found called once (for `MobdId_Cursors`). Verify this is the only usage. If called in hot paths, it's a performance issue.

### TODO: Map all collision handler response functions

`g_unit_collision_handlers[20]` contains 20 entries with functions like `BOXD_collide_solid`, `BOXD_collide_floor`, `BOXD_collide_ramp_ltr`, `BOXD_collide_slope_left`, `BOXD_collide_corner_right`, `BOXD_collide_cursor`. These define the physics response behavior but aren't documented yet.

### TODO: Investigate `entity->z` usage

Z is used for both rendering (Z-sorting) and gameplay (height, projectile arcs). The two render handlers (`render_state_handler_entity` vs `render_state_handler_default`) use different Z formulas:
- Flat: `z = entity->z`
- Isometric: `z = (entity->z + entity->y) >> 8`

Document which contexts use which renderer and why.

### TODO: Document MobdPoint anchor point system

`MobdAnimFrame` has `MobdPoint points[]` with an `id` field. When `points[0].id` is set and task waits on `0x40000`, an event fires. This is used for turret fire points, projectile spawn origins, etc. Full documentation needed.

### TODO: Investigate `EntitySaveStruct._54_inside_mobd_ptr4`

This field name suggests it's at offset 0x54 inside something, storing a MOBD-relative pointer. Its purpose in save/load is unclear — the decompiler gave it a descriptive but opaque name.

### TODO: Count entity types by collision table

Three collision tables exist: `g_unit_collision_handlers`, `g_ui_collision_handlers`, `g_null_collision`. Categorize all entity types by which collider they use. This maps the collision "ecosystem".

---

## Naming Convention Suggestions

### Entity functions should follow `entity_<verb>` pattern
- `entity_create`, `entity_remove` — already good
- `entity_create_ex` — OK (extends entity_create)
- `entity_anim_*` — good namespace
- `sub_426F90` → `entity_remove_with_task`
- `entity_408800_play_sound` → `entity_play_sound`
- `entity_40D8B0` → `entity_find_nearby_in_tiles`

### Physics/frame-tick functions follow `GAME_Advance*` pattern
- `GAME_AdvanceEntityPositions` — good
- `GAME_AdvanceEntityAnimations` — good

### Render handlers follow `render_state_handler_*` pattern
- `render_state_handler_entity` — gameplay (flat Z)
- `render_state_handler_default` → `render_state_handler_entity_isometric`
