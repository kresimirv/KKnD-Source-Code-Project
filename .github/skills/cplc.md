# CPLC — Compiled Placement System

CPLC = "Compiled Placement". Level file section storing pre-placed entity positions, spawn params, and camera. Used for both **in-game level entities** (units, buildings, oil patches, convoys) and **menu UI elements** (buttons, cursors, tooltips).

---

## Data Structures

### `CplcSpawnParams` ([kknd.h](kknd.h#L1770))

```c
struct CplcSpawnParams {
  MobdId mobd_id;                        // sprite bank id
  TaskFn task;                           // handler function pointer
  TaskKind task_kind;                    // Fiber vs Callback
  BOOL skip_spawning;                    // if true, entity unlinked but NOT spawned
  Entity *entity;                        // back-pointer to spawned Entity (set at spawn time)
  int _cplc_sawpn_params_field_14;       // AI build queue linked-list pointer (TYPO: "sawpn")
  int player_side_or_stru49_array_idx;   // owner player index (0..N)
  int spawn_param;                       // OVERLOADED: oil amount, convoy checkpoint, AI waypoints ptr, wander delay
  UnitType unit_id;                      // unit type for script lookup
  int owner_side;                        // used ONLY by Clanhall/Outpost/DetentionCenter/HoldingPens
  int _cplc_spawn_params_field_28;       // unknown
  int _cplc_spawn_params_field_2c;       // unknown
};
```

`spawn_param` meanings by entity type:
- **Oil patch**: `spawn_param * 500` = oil amount
- **Convoy/Tanker**: checkpoint index (if >= 15, used as checkpoint id)
- **AI wanderer unit**: countdown ticks before joining active hunt list
- **Building (Clanhall/Outpost)**: AI build queue / waypoint data pointer
- **General mobile unit**: if nonzero, sets `path_flags |= 0x80` (scripted path / wander)

### `CplcEntity` ([kknd.h](kknd.h#L1810))

```c
struct CplcEntity {
  TaskType task_type;           // indexes g_scripts[]
  int x, y, z;                 // world position (16.16 fixed-point)
  CplcEntity *next_x_sorted;   // X-sorted doubly-linked list
  CplcEntity *prev_x_sorted;
  CplcEntity *next_y_sorted;   // Y-sorted doubly-linked list
  CplcEntity *prev_y_sorted;
  CplcSpawnParams spawn_params; // inline spawn params
};
```

Stored in TWO independent sorted doubly-linked lists (X and Y). When spawned, entity is **unlinked from both** — it can never spawn twice.

### `CplcEntityInViewport` ([kknd.h](kknd.h#L1824))

```c
struct CplcEntityInViewport {
  CplcEntityInViewport *next, *prev;
  Entity *entity;               // back-pointer to spawned Entity
};
```

**Frame allocator pattern** — 2000 pre-allocated slots (`0x5DC0` bytes). `_head` is free-list pointer; `_tail` is active-list head. Reset to pool base on layer switch/restore. NOT a true pool (no per-slot free list). Slots returned to free list when entity exits viewport.

### `LevelCplcSurface` ([kknd.h](kknd.h#L1959))

```c
struct LevelCplcSurface {
  int size;                     // byte size of entity data (excluding this field)
  CplcEntity *next_x_sorted;   // head of X-sorted list (leftmost)
  CplcEntity *prev_x_sorted;   // tail of X-sorted list (rightmost)
  CplcEntity *next_y_sorted;   // head of Y-sorted list (topmost)
  CplcEntity *prev_y_sorted;   // tail of Y-sorted list (bottommost)
};
```

### `LevelCplc` ([kknd.h](kknd.h#L1969))

```c
struct LevelCplc {
  LevelCplcSurface *layers;    // array indexed by layer id (MenuId / surface index)
};
```

### Entity fields ([kknd.h](kknd.h#L1378-L1380))

```c
CplcSpawnParams *cplc_spawn_params;  // non-NULL → entity placed from CPLC data
CplcEntity *cplc_meta;               // back-pointer to CplcEntity record
CplcEntityInViewport *cplc_view;     // viewport tracking node (NULL if exited viewport)
```

---

## Global Variables

### Core State

| Variable | Type | Purpose |
|---|---|---|
| `g_current_lvl_cplc` | `LevelCplc*` | Pointer to "CPLC" section from `LVL_FindSection("CPLC")` |
| `g_current_lvl_cplc_verified` | `int` | 1 if CPLC loaded & camera found; 0 otherwise |
| `g_current_lvl_cplc_layer` | `LevelCplcSurface*` | Currently active layer header |
| `g_current_lvl_cplc_layer_size` | `int` | `layer->size + 4` (total byte count) |
| `g_current_lvl_cplc_backup` | `LevelCplcSurface*` | malloc'd copy of original layer (for restore) |

### Sorted List Cursors

| Variable | Type | Purpose |
|---|---|---|
| `g_current_lvl_cplc_layer_x_sorted_base` | `CplcEntity*` | Head of remaining (unspawned) X-sorted list |
| `g_current_lvl_cplc_layer_x_sorted_right` | `CplcEntity*` | Right cursor — tracks viewport right edge |
| `g_current_lvl_cplc_layer_x_sorted_left` | `CplcEntity*` | Left cursor — tracks viewport left edge |
| `g_current_lvl_cplc_layer_y_sorted_base` | `CplcEntity*` | Head of remaining (unspawned) Y-sorted list |
| `g_current_lvl_cplc_layer_y_sorted_bottom` | `CplcEntity*` | Bottom cursor — tracks viewport bottom edge |
| `g_current_lvl_cplc_layer_y_sorted_top` | `CplcEntity*` | Top cursor — tracks viewport top edge |
| `g_cplc_camera` | `CplcEntity*` | CplcEntity with `TaskType_Camera` — initial camera pos |

### Viewport Boundaries

| Variable | Address | Computation | Purpose |
|---|---|---|---|
| `g_cplc_viewport_left_x` | `0x477428` | `camera.x - 0x4000` | Spawn zone left edge |
| `g_cplc_viewport_top_y` | `0x477434` | `camera.y - 0x4000` | Spawn zone top edge |
| `dword_47745C` | `0x47745C` | `camera.x + ((screen_width + 64) << 8)` | Spawn zone right edge |
| `dword_477460` | `0x477460` | `camera.y + ((screen_height - 64) << 8)` | Spawn zone bottom edge |
| `dword_477448` | `0x477448` | `camera.x - 24576` | Despawn zone left edge |
| `dword_477440` | `0x477440` | `camera.x + ((screen_width + 96) << 8)` | Despawn zone right edge |
| `dword_477454` | `0x477454` | `camera.y - 24576` | Despawn zone top edge |
| `dword_47743C` | `0x47743C` | `camera.y + ((screen_height - 32) << 8)` | Despawn zone bottom edge |

Two **nested rectangles**: inner (spawn) and outer (despawn). ~32–96 pixel hysteresis margin prevents spawn/despawn flicker at edges.

### Viewport Pool

| Variable | Type | Purpose |
|---|---|---|
| `g_cplc_entities_in_viewport_pool` | `CplcEntityInViewport*` | Base of 2000-entry pool |
| `g_cplc_entities_in_viewport_head` | `CplcEntityInViewport*` | Free list head |
| `g_cplc_entities_in_viewport_tail` | `CplcEntityInViewport*` | Active list head (NAMING IS INVERTED) |

### Synthetic Spawn Params

| Variable | Type | Purpose |
|---|---|---|
| `g_outpost_spawn_params` | `CplcSpawnParams` | BSS zero-init sentinel for Mobile Base → Outpost |
| `g_clanhall_spawn_params` | `CplcSpawnParams` | BSS zero-init sentinel for Mobile Base → Clanhall |

---

## Functions

### Core CPLC Functions

| Function | Line | Suggested Name | Purpose |
|---|---|---|---|
| `LVL_cplc_init` | [kknd.c](kknd.c#L10089) | `LVL_cplc_init` | Parse "CPLC" section, alloc viewport pool, backup layer, init cursors, find camera |
| `CPLC_select` | [kknd.c](kknd.c#L10180) | `CPLC_select` | Switch to a different CPLC layer by index; used for menu screens |
| `cplc_4060F0` | [kknd.c](kknd.c#L10253) | **`cplc_restore_from_backup`** | Restore layer from backup copy, reset cursors and viewport pool |
| `cplc_entity_init` | [kknd.c](kknd.c#L10311) | **`cplc_entity_spawn`** | Unlink CplcEntity from sorted lists, create Entity+Task, link cplc pointers |
| `cplc_4062E0` | [kknd.c](kknd.c#L10378) | **`cplc_viewport_remove`** | Remove viewport node from active list, return to free pool |
| `cplc_406320` | [kknd.c](kknd.c#L10394) | **`cplc_viewport_update`** | Per-frame viewport culling tick — spawn entering entities, despawn leaving ones |
| `cplc_init` | [kknd.c](kknd.c#L10826) | `cplc_init` | Walk X-sorted list, spawn or consume all entities of given TaskType |
| `cplc_init_with_objects` | [kknd.c](kknd.c#L10818) | `cplc_init_with_objects` | Wrapper: `cplc_init(type, 1)` |
| `cplc_init_without_objects` | [kknd.c](kknd.c#L10894) | `cplc_init_without_objects` | Wrapper: `cplc_init(type, 0)` — unlink without spawning |
| `cplc_cleanup` | [kknd.c](kknd.c#L10900) | `cplc_cleanup` | Free viewport pool and backup, set verified=0 |

### Batch Init Functions

| Function | Line | Purpose |
|---|---|---|
| `cplc_init_all` | [kknd.c](kknd.c#L36449) | Loop 0..TaskType_Max, `cplc_init_with_objects` each (skip Camera). Full level startup. |
| `cplc_init_subset` | [kknd.c](kknd.c#L36456) | UI-only init: `cplc_init_with_objects` for UI TaskTypes, `cplc_init_without_objects` for everything else |

### Level Flow

| Function | Line | CPLC Usage |
|---|---|---|
| `GAME_PrepareSuperLvl` | [kknd.c](kknd.c#L35344) | Load super.lvl → `CPLC_select(mapd_id)` → `cplc_viewport_update()` |
| `sub_423320` (mission cleanup) | [kknd.c](kknd.c#L35831) | Calls `cplc_restore_from_backup()` for non-Xtreme levels |
| `LVL_Deinit` | [kknd.c](kknd.c#L28399) | Calls `cplc_cleanup()` |

---

## Data Flow

```
Level File (.lvl)
  └── "CPLC" section
        │
        ▼
  LVL_cplc_init() / CPLC_select(layer_id)
  ├── g_current_lvl_cplc = LVL_FindSection("CPLC")
  ├── g_current_lvl_cplc_layer = g_current_lvl_cplc[id].layers
  ├── Backup layer → g_current_lvl_cplc_backup
  ├── Init X/Y sorted list cursors
  └── Find TaskType_Camera → g_mapd_camera
        │
        ▼ (every frame)
  cplc_viewport_update()   [cplc_406320]
  ├── Phase 1: DESPAWN — walk active viewport list
  │   └── Entity outside despawn rect → set TaskEvent_LeftViewport flag, return slot to free pool
  ├── Phase 2: SPAWN BY X — compare viewport_left_x to previous frame
  │   ├── Camera right → walk x_sorted_right forward, spawn entities in spawn rect
  │   └── Camera left  → walk x_sorted_left backward, spawn entities in spawn rect
  └── Phase 3: SPAWN BY Y — compare viewport_top_y to previous frame
      ├── Camera down → walk y_sorted_bottom forward
      └── Camera up   → walk y_sorted_top backward
        │
        ▼ (for each entity entering viewport)
  cplc_entity_spawn()   [cplc_entity_init]
  ├── Unlink from X-sorted and Y-sorted doubly-linked lists
  ├── Check skip_spawning flag
  ├── g_scripts[task_type] → handler, mobd_id
  ├── Create Task (fiber or callback)
  ├── entity_create(mobd_id, task)
  ├── Set entity.x/y/z from CplcEntity
  ├── entity.cplc_spawn_params = &cplc->spawn_params (bidirectional)
  ├── Alloc CplcEntityInViewport → link entity.cplc_view
  └── Add to active viewport list
        │
        ▼ (task handler runs)
  unit_create(task)
  ├── If entity->cplc_meta exists:
  │   ├── Normal unit: type from g_scripts, player from player_side_or_stru49_array_idx
  │   ├── Building (Clanhall/Outpost/Prison/Pens): player from owner_side
  │   ├── cplc_spawn_param = spawn_params->spawn_param
  │   └── If spawn_param != 0: path_flags |= 0x80 (wander flag)
  └── If no cplc_meta: extract type/owner from entity->ctx bitmask
```

---

## CPLC vs Player-Built Discrimination

The key discriminator: `entity->cplc_spawn_params != NULL`

| Aspect | CPLC-placed | Player-built |
|---|---|---|
| Construction animation | **Skipped** | Played |
| "Building completed" message | **Suppressed** | Shown |
| Completion sound | **Suppressed** | Played |
| Post-completion mode | `unit_mode_4034B0` (grid snap → idle) | `unit_mode_403650` (normal idle) |
| Drillrig grid build | **Skipped** | Called |
| Power Station tanker offset | Y+0x6000 | X-0x2000, Y+0x2000 |
| HP on Mute_09 / Surv_21 | **Reduced to HP/5 - 1** (if player_num==0) | Normal |
| AI reads build queue | From spawn_params | N/A |

### Building handlers with CPLC branching

All follow same pattern: `if (!entity->cplc_spawn_params) { animate_construction(); return; } else { skip_to_completed_mode(); }`

| Handler | Line | Building |
|---|---|---|
| `UNIT_Handler_BeastEnclosure` | [kknd.c](kknd.c#L6628) | Beast Enclosure |
| `UNIT_Handler_Blacksmith` | [kknd.c](kknd.c#L6920) | Blacksmith |
| `UNIT_Handler_Clanhall` | [kknd.c](kknd.c#L8200) | Clanhall |
| `UNIT_Handler_MachineShop` | [kknd.c](kknd.c#L35026) | Machine Shop |
| `UNIT_Handler_Outpost` | [kknd.c](kknd.c#L48756) | Outpost |
| `UNIT_Handler_PowerStation` | [kknd.c](kknd.c#L52043) | Power Station |
| `UNIT_Handler_RepairBuilding` | [kknd.c](kknd.c#L53680) | Repair Building |
| `UNIT_Handler_ResearchBuilding` | [kknd.c](kknd.c#L53984) | Research Building |
| `UNIT_Handler_Drillrig` | [kknd.c](kknd.c#L12050) | Drillrig |

---

## `unit->cplc_spawn_param` — The Overloaded Timer Field

This field on `Unit` (NOT the pointer on `Entity`) is reused for many purposes:

| Context | Set to | Meaning |
|---|---|---|
| `unit_create` | `spawn_params->spawn_param` | Initial value from level data |
| Bomber handler | 15, counts to 0 | Bomb drop inter-shot cooldown |
| Building idle | 2000, counts to 0 | "Under attack" announcement cooldown |
| AI wanderer | Countdown from initial value | Ticks until unit joins active hunt list |
| Prison/Bunker | 10, counts to 0 | Spawn queue remaining units |

---

## Viewport Culling Algorithm Detail

`cplc_viewport_update` ([kknd.c](kknd.c#L10394)) uses **dual-axis incremental sweep**:

1. Maintains four sorted-list cursors (`x_sorted_left`, `x_sorted_right`, `y_sorted_top`, `y_sorted_bottom`) tracking the edges of what's been processed.
2. Each frame, compares new viewport position to previous.
3. Walks cursors in the direction of camera movement, spawning entities that fall within the spawn rect.
4. After spawning an entity (which unlinks it from both sorted lists), all four cursors are re-validated — if a cursor pointed at the spawned entity, it advances to the next valid node.
5. Entities are one-way: once spawned, they cannot be re-spawned from CPLC. The despawn phase only removes the viewport tracking node (sets `TaskEvent_LeftViewport` flag on the task), but the entity/task lives on.

### Two Rectangles

```
+--- despawn rect (outer) ----+
|   +-- spawn rect (inner) -+ |
|   |                        | |
|   |      viewport          | |
|   |                        | |
|   +------------------------+ |
+------------------------------+
```

Hysteresis: ~32–96 pixel gap prevents flicker. Entity enters spawn rect → spawned. Entity exits despawn rect → viewport tracking removed.

---

## Menu vs In-Game Usage

CPLC serves **dual purpose**:

### Menu (super.lvl)
- Multiple layers (indexed 0..15+), one per menu screen (Main, NewCampaign, Credits, etc.)
- `CPLC_select(menuId)` switches layers
- `cplc_init_subset()` spawns only UI TaskTypes (buttons, cursors, tooltips, fonts)
- Non-UI TaskTypes consumed without spawning via `cplc_init_without_objects()`

### In-Game (level .lvl)
- Single layer (layer 0, or selected by `CPLC_select` for specific levels)
- `cplc_init_all()` spawns everything (all TaskTypes except Camera)
- Viewport streaming: `cplc_viewport_update()` called every frame
- Camera entity (`TaskType_Camera`) consumed to set initial viewport position

### Layer Backup System
- On init/select, layer data is memcpy'd to `g_current_lvl_cplc_backup`
- On mission-complete transition (non-Xtreme levels): `cplc_restore_from_backup()` restores original sorted lists
- This allows the super.lvl to be reused across menu transitions without reloading

---

## Cross-References

- [task.md](task.md#L444) — CPLC Level Spawning section (brief overview)
- [ai.md](ai.md#L183) — Wanderer system (CPLC-spawned units with countdown)
- [units.md](units.md#L71) — Entity CPLC link fields
- [object-lifecycle.md](object-lifecycle.md#L125) — CplcEntityInViewport pool pattern
- [object-lifecycle-todo.md](object-lifecycle-todo.md#L101) — Frame allocator pattern note
