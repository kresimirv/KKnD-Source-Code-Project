# CPLC — TODO: Renames, Issues, Suggestions

## Function Renames

| Current Name | Suggested Name | Line | Reason |
|---|---|---|---|
| `cplc_4060F0` | `cplc_restore_from_backup` | [kknd.c](kknd.c#L10253) | Restores layer from backup on mission transition. Name says nothing. |
| `cplc_entity_init` | `cplc_entity_spawn` | [kknd.c](kknd.c#L10311) | "init" is misleading — it UNLINKS from sorted lists AND creates Entity+Task. It's a spawn. |
| `cplc_4062E0` | `cplc_viewport_remove` | [kknd.c](kknd.c#L10378) | Removes viewport tracking node, returns to free pool. |
| `cplc_406320` | `cplc_viewport_update` | [kknd.c](kknd.c#L10394) | Per-frame viewport culling tick. Current name is raw address. |
| `sub_423320` | `game_mission_cleanup` | [kknd.c](kknd.c#L35831) | Calls `cplc_restore_from_backup`, `LVL_cleanup`, etc. |

---

## Global Variable Renames

### Unnamed Viewport Boundaries (PRIORITY — used in cplc_viewport_update)

| Current Name | Suggested Name | Address | Computation |
|---|---|---|---|
| `dword_47745C` | `g_cplc_spawn_right_x` | `0x47745C` | `camera.x + ((screen_width + 64) << 8)` |
| `dword_477460` | `g_cplc_spawn_bottom_y` | `0x477460` | `camera.y + ((screen_height - 64) << 8)` |
| `dword_477448` | `g_cplc_despawn_left_x` | `0x477448` | `camera.x - 24576` |
| `dword_477440` | `g_cplc_despawn_right_x` | `0x477440` | `camera.x + ((screen_width + 96) << 8)` |
| `dword_477454` | `g_cplc_despawn_top_y` | `0x477454` | `camera.y - 24576` |
| `dword_47743C` | `g_cplc_despawn_bottom_y` | `0x47743C` | `camera.y + ((screen_height - 32) << 8)` |

### Inverted Naming Convention

| Current Name | Suggested Name | Reason |
|---|---|---|
| `g_cplc_entities_in_viewport_tail` | `g_cplc_viewport_active_head` | This is the HEAD of the active list (first iterated). "tail" is confusing. |
| `g_cplc_entities_in_viewport_head` | `g_cplc_viewport_free_head` | This is the free-list head. "head" clashes with the actual active list head above. |

---

## Struct Field Issues

### Typo: `_cplc_sawpn_params_field_14`

**File:** [kknd.h](kknd.h#L1777)
**Issue:** "sawpn" → should be "spawn". Typo from decompiler annotation.
**Suggested name:** `ai_build_queue_next` (it's a linked-list pointer for AI build queue nodes)

### Unknown Fields to Investigate

| Field | Struct | Line | Notes |
|---|---|---|---|
| `_cplc_spawn_params_field_28` | `CplcSpawnParams` | [kknd.h](kknd.h#L1782) | Never seen written in investigated code. May be padding or unused. |
| `_cplc_spawn_params_field_2c` | `CplcSpawnParams` | [kknd.h](kknd.h#L1783) | Same — investigate if any code reads offset 0x2C. |

---

## Naming Confusion: `cplc_spawn_param` vs `cplc_spawn_params`

**This is the biggest naming issue in the CPLC system.**

| Name | Where | Type | Meaning |
|---|---|---|---|
| `entity->cplc_spawn_params` | Entity struct | `CplcSpawnParams*` | Pointer to CPLC origin data. Non-NULL = CPLC-placed. |
| `unit->cplc_spawn_param` | Unit struct | `int` | Overloaded timer/counter reused for bombers, buildings, AI, prisons |

These differ by ONE letter ("s") and have completely different semantics. The `unit->cplc_spawn_param` field is NOT really a CPLC spawn param — it's a **general-purpose countdown timer** that happens to be initialized from `spawn_params->spawn_param` at unit creation.

**Suggested rename for `unit->cplc_spawn_param`:**
- Option A: `multi_purpose_timer` (generic but honest)
- Option B: `spawn_countdown` (matches its primary CPLC use but misleading for bomber/building reuse)
- Option C: Keep as-is but add comments documenting each usage context

---

## Decompilation Bugs

### 1. `unit_create` — Wrong struct member access for buildings

**Line:** [kknd.c](kknd.c#L72838)
**Code:** `g_scripts[entity_task_type][1].kind`
**Issue:** This indexes `g_scripts` as if it were `ScriptType4[2]` array, reading offset 0x18 from base. For `ScriptType4` (size 0x24) that's `unit_type` (correct). For `ScriptType3` (size 0x18), this reads `script_type_3_field_14` instead.
**TODO:** Verify which TaskTypes used by Clanhall/Outpost/Prison/Pens use ScriptType3 vs ScriptType4.

### 2. Viewport pool "tail" is actually the active list head

**Lines:** [kknd.c](kknd.c#L10138), [kknd.c](kknd.c#L10225), [kknd.c](kknd.c#L10283)
**Issue:** `g_cplc_entities_in_viewport_tail` is iterated as the first element in every loop — it's the HEAD of the active/in-use list. The "free list" pointer is confusingly called `_head`. Either decompiler got them swapped, or the original dev used unconventional naming.

### 3. `sub_442BB0` — Wrong type cast

**Line:** ~63782 in kknd.c
**Issue:** Decompiler typed a local as `KKND::Task *cplc_spawn_params` when it's actually reading `entity->cplc_spawn_params` as a generic pointer. The `Task*` cast is bogus.

### 4. Loop termination bugs in building completion

**Lines:** ~8265, ~48800 in kknd.c
**Issue:** Loop termination in `unit_mode_clanhall_on_completed` and `unit_mode_outpost_on_completed` compares against unrelated data (`g_unit_collision_handlers`, `dword_468FDC`). Likely decompiler artifact from incorrect array bounds detection.

---

## Missing Investigations

### TODO: Trace CplcSpawnParams.task and CplcSpawnParams.task_kind

These fields store handler function pointer and fiber/callback kind. Are they redundant with `g_scripts[task_type]`? If so, they may be legacy or level-editor overrides. Need to check if `cplc_entity_spawn` ever uses `spawn_params.task` instead of `g_scripts[task_type]->handler`.

### TODO: Trace CplcSpawnParams.mobd_id

Same question — is `spawn_params.mobd_id` ever used, or does `cplc_entity_spawn` always use `g_scripts[task_type]->mobd_id`? If redundant, the spawn_params copy may be level-editor data that the runtime ignores.

### TODO: Investigate skip_spawning flag

`cplc_entity_spawn` checks `if (!cplc->spawn_params.skip_spawning)`. When is this flag set? Is it in the level file data, or set dynamically? Could be used for conditional entity placement (difficulty levels, multiplayer).

### TODO: Map CPLC_select layer IDs to menu screens

The layer indices passed to `CPLC_select` correspond to `MenuId` enum values. Full mapping:

| Layer ID | Used in | Likely Menu |
|---|---|---|
| 0 | Main menu, various fallbacks | Main/Title |
| 1 | Multiple places | Unknown menu |
| 2 | [kknd.c](kknd.c#L58722) | NewCampaign |
| 4 | [kknd.c](kknd.c#L59437) | Unknown |
| 5 | [kknd.c](kknd.c#L59517) | Unknown |
| 6 | [kknd.c](kknd.c#L59598) + many | Multiplayer? |
| 7 | [kknd.c](kknd.c#L61186) | Unknown |
| 8 | [kknd.c](kknd.c#L61092) | Unknown |
| 9 | [kknd.c](kknd.c#L60993) | Unknown |
| 10 | [kknd.c](kknd.c#L61253) | Unknown |
| 11 | [kknd.c](kknd.c#L58866) | PlayMission |
| 12 | [kknd.c](kknd.c#L58403) | Credits |
| 14 | [kknd.c](kknd.c#L58951) | NewMissions |
| 15 | [kknd.c](kknd.c#L59036) | Unknown |

**TODO:** Cross-reference with MenuId enum to complete this table.

### TODO: Verify `g_outpost_spawn_params` / `g_clanhall_spawn_params` fields

These BSS globals are zero-initialized sentinels. When `unit_create` reads `owner_side` from them, it gets 0 — is that correct? Does player 0 always own mobile-base-planted buildings? Or should these globals be initialized with the planting unit's player_num?

### TODO: Understand the HP reduction on Mute_09 / Surv_21

[kknd.c](kknd.c#L7237): CPLC buildings with `player_num == 0` on two specific levels get HP reduced to 1/5. Is player 0 neutral? Or is this a specific scenario mechanic (pre-damaged neutral buildings the player must capture)?

### TODO: Investigate `LevelCplcSurface.size` interpretation

The `size` field stores byte count of entity data EXCLUDING the size field itself. Code does `size + 4` to get total. But the sorted list pointers (next_x/prev_x/next_y/prev_y) are also in the struct — does `size` include those 16 bytes? If yes, actual entity data = `size - 16` bytes. If no, entities start right after `prev_y_sorted`.

---

## Naming Convention Suggestions

### CPLC function naming pattern
All CPLC core functions should follow: `cplc_<noun>_<verb>` or `CPLC_<Verb>`:
- `cplc_entity_spawn` (not `cplc_entity_init`)
- `cplc_viewport_update` (not `cplc_406320`)
- `cplc_viewport_remove` (not `cplc_4062E0`)
- `cplc_restore_from_backup` (not `cplc_4060F0`)
- `CPLC_select` — already good
- `LVL_cplc_init` — already good (follows LVL_ prefix convention)

### Global naming convention
All CPLC globals already follow `g_cplc_*` or `g_current_lvl_cplc_*`. The unnamed `dword_*` globals should follow `g_cplc_spawn_*` / `g_cplc_despawn_*` pattern.

### The "two rectangles" globals should be grouped
Consider a struct:
```c
struct CplcViewportBounds {
  int spawn_left, spawn_right, spawn_top, spawn_bottom;
  int despawn_left, despawn_right, despawn_top, despawn_bottom;
};
```
But since these are guessed-type globals at fixed addresses, renaming individual dwords is more practical.
