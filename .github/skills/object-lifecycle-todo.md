# Object Lifecycle TODO — Inconsistencies & Improvements

Checklist of naming inconsistencies, decompiler artifacts, structural issues, and suggested renames for pool/free-list/active-list management across the codebase.

---

## 1. Naming Inconsistencies (Free List Pointer)

The free list pointer has wildly inconsistent naming. Should standardize to:
- **Globals**: `g_xxx_free_list`
- **Struct members**: `xxx_free`

| Current Name | Subsystem | Suggested Rename |
|---|---|---|
| `g_aircraft_free_list_head` | Aircraft | `g_aircraft_free_list` (drop "_head" — it's always a head) |
| `g_oil_patch_free_pool` | OilPatch | `g_oil_patch_free_list` |
| `g_entity_free_pool_head` | Entity | `g_entity_free_list` |
| `g_game_event_free_pool` | GameEvent | `g_game_event_free_list` |
| `g_viewports_free` | Viewport | `g_viewports_free_list` |
| `g_sounds_free` | Sound | `g_sounds_free_list` |
| `g_prod_free_list` | Production | OK as-is |
| `g_prod_options_free_list` | ProductionOption | OK as-is |
| `g_sidebar_free_list_head` | Sidebar | `g_sidebar_free_list` |
| `g_sidebar_button_free_list_head` | SidebarButton | `g_sidebar_button_free_list` |
| `g_escort_free_list_head` | Escort | `g_escort_free_list` |
| `g_file_free` | File | `g_file_free_list` |
| `g_buildings_tracker_` | BuildingsTracker | `g_buildings_tracker_free_list` (TRUNCATED!) |
| `dword_477350` | Coroutine | `g_coroutine_free_list` (UNNAMED!) |
| `selection_free_pool` | CursorState member | `selection_free` |

---

## 2. Naming Inconsistencies (Active List Head/Tail)

| Current Name | Subsystem | Issue | Suggested |
|---|---|---|---|
| `g_siderbar_active_list_head` | Sidebar | **TYPO**: "siderbar" | `g_sidebar_active_list_head` |
| `g_siderbar_active_list_tail` | Sidebar | **TYPO**: "siderbar" | `g_sidebar_active_list_tail` |
| `g_aircraft_list_head/tail` | Aircraft | OK — short form |  |
| `g_oil_patch_list_head` | OilPatch | OK |  |
| `g_buildings_tracker_list_head/tail` | BuildingsTracker | OK but verbose | `g_building_tracker_head/tail`? |
| `g_entity_head/tail` | Entity | OK — short form |  |
| `g_sounds_head` | Sound | Missing tail — uses `dword_47C398` as sentinel | Rename `dword_47C398` → `g_sounds_sentinel` or find real tail |
| `g_coroutine_list_head` | Coroutine | Missing tail | Singly-linked, no tail needed |

---

## 3. Unnamed / Truncated Globals (Decompiler Artifacts)

| Address/Name | True Identity | Evidence |
|---|---|---|
| `dword_477350` | `g_coroutine_free_list` | Used exactly like free list: pop on alloc, push on release. Init: `= g_coroutine_pool + 1` (pool[0] goes to list_head) |
| `g_buildings_tracker_` | `g_buildings_tracker_free_list` | Used at line 7885-7887 as free list pop; line 7937-7938 as free list push |
| `dword_47C398` | `g_sounds_list_tail` or sentinel | Adjacent to `g_sounds_head`, used as sentinel: `g_sounds_head = (stru7_sound*)&dword_47C398` |

---

## 4. Structural Inconsistencies

### 4a. Entity Free List Uses Circular Doubly-Linked (Non-Standard)

Unlike all other pools which use **singly-linked free lists**, Entity pool uses doubly-linked circular free list:
```c
// Entity init — circular, uses prev:
g_entity_free_pool_head = result;
result->prev = (Entity *)&g_entity_free_pool_head;
// ...
result->next = (Entity *)&g_entity_free_pool_head;  // circular termination
```
All others use `nullptr` termination and singly-linked `next` only.

**Suggestion**: Document this as intentional (Entity is larger, may need O(1) removal from free list?) or flag as potential over-engineering.

### 4b. Coroutine Pool Splits Allocation

Coroutine init does something unique — `pool[0]` goes to `g_coroutine_list_head` and free list starts at `pool[1]`:
```c
g_coroutine_list_head = g_coroutine_pool;      // pool[0] is "current" sentinel
dword_477350 = g_coroutine_pool + 1;           // free list starts at pool[1]
```
This is NOT a bug — pool[0] is the scheduler's "idle" coroutine. But it's undocumented.

### 4c. Sound Pool Uses Raw Offset Instead of `->next`

Sound alloc uses raw pointer arithmetic: `g_sounds_free = *((void**)g_sounds_free + 14)` — offset 56 (14*4). This means `next` is NOT the first struct member.

**Suggestion**: Verify `stru7_sound` layout — `next` may be at offset 56, or the struct is mistyped. Active list uses offset 56 for next too (`v11[14]`). Should define proper `next`/`prev` fields in struct.

### 4d. Escort Pool — Missing Active List Head/Tail

`g_escort_pool` and `g_escort_free_list_head` exist, but NO active list head/tail found. Escort nodes are embedded in Unit's `escort_list_head/tail` (per-unit lists, not global). This is correct — not a global active list.

### 4e. File Pool — Singly-Linked Only (No Active List)

`g_file_pool` / `g_file_free` — no head/tail for active list. File handles don't need iteration; just alloc/release. This is correct pattern for "allocation-only pools."

### 4f. GameEvent — Queue Pattern (Not Standard Pool)

`g_game_event_pool`/`g_game_event_free_pool` use singly-linked free list, but active items go into `g_game_event_queue` (FIFO queue, not doubly-linked list). Different lifecycle pattern.

### 4g. CplcEntityInViewport — Reset-Style Pool (No Free List)

`g_cplc_entities_in_viewport_pool` has a `_head` but no free list. Instead, `_head` is reset to `_pool` every frame. This is a **frame allocator** pattern, not a true pool. All slots are implicitly freed each frame.

### 4h. TileCollisions — Same Frame Allocator Pattern

`g_tile_collisions_pool` has `g_tile_collisions_head` that resets to pool base. No free list. Frame allocator.

---

## 5. Missing Cleanup Null-Outs

Some cleanups don't null the pointer after free (risk of use-after-free in debug):

| Subsystem | Cleanup | Nulls pool? | Nulls free? |
|---|---|---|---|
| Aircraft | `free(g_aircraft_pool)` | **NO** | **NO** |
| OilPatch | `free(g_oil_patch_pool)` | **NO** | **NO** |
| BuildingsTracker | `free(g_buildings_tracker_pool)` | **NO** | **NO** |
| Coroutine | `free(g_coroutine_pool)` | YES | YES |
| Entity | `free(g_entity_pool)` | YES | **NO** |
| Escort | `free(g_escort_pool)` | **NO** | **NO** |
| Viewport | `free(g_viewports_pool)` | **NO** | **NO** |

**Suggestion**: Standardize cleanup to null all 4 pointers (pool, free, head, tail). Coroutine does it right.

---

## 6. Pool Size Summary

| Pool | Count | Element Size (approx) | Total Bytes |
|---|---|---|---|
| Aircraft | 60 | 0x0C (12) | 0x2D0 |
| OilPatch | 256 | 0x18 (24) | 0x1800 |
| BuildingsTracker | 20 | 0x10 (16) | 0x140 |
| Entity | 2000 | 0x92 (146) | 0x48440 |
| Coroutine | 2000 | 0x10 (16) | 0x7D00 |
| Sidebar | 16 | 0x4C (76) | 0x4C0 |
| SidebarButton | 50 | 0x28 (40) | 0x7D0 |
| Escort | 500 | varies | varies |
| Viewport | 7 | varies | varies |
| Sound | 8 | 0x148 (328) | 0xA40 |
| GameEvent | 8 | varies | varies |
| File | 20 | varies | varies |

---

## 7. Proposed Rename Actions (Priority Order)

### Critical (broken/misleading names)
1. `dword_477350` → `g_coroutine_free_list`
2. `g_buildings_tracker_` → `g_buildings_tracker_free_list`
3. `g_siderbar_active_list_head` → `g_sidebar_active_list_head` (typo fix)
4. `g_siderbar_active_list_tail` → `g_sidebar_active_list_tail` (typo fix)
5. `dword_47C398` → `g_sounds_list_tail` (or `g_sounds_sentinel`)

### High (consistency)
6. `g_entity_free_pool_head` → `g_entity_free_list`
7. `g_oil_patch_free_pool` → `g_oil_patch_free_list`
8. `g_game_event_free_pool` → `g_game_event_free_list`
9. `selection_free_pool` → `selection_free`

### Medium (cosmetic)
10. Drop `_head` suffix from free list globals (they're always heads)
11. Add `_list` suffix to short-form globals: `g_viewports_free` → `g_viewports_free_list`
12. `g_sounds_free` → `g_sounds_free_list`
13. `g_file_free` → `g_file_free_list`

---

## 8. Patterns to Verify During Future Investigation

- [ ] Does `stru7_sound` really have `next` at offset 56? Or is struct layout wrong?
- [ ] Is Entity's circular free list intentional or decompiler artifact?
- [ ] `enemy_tail` appearing BEFORE `enemy_head` in AiController — verify field offsets
- [ ] `g_4795E8_pool` / `g_4795E8_tail` — stru2 pool completely unnamed, needs investigation
- [ ] Coroutine pool[0] as idle/scheduler coroutine — verify and document
- [ ] `_ai_controller_70`/`_ai_controller_74` in AiController — likely another head/tail pair for attacker sublist

---

## 9. Pool Lifecycle Variant Classification

| Pattern | Examples | Free List | Active List | Reset |
|---|---|---|---|---|
| **Full Pool** (alloc+free+iterate) | Aircraft, OilPatch, Buildings, Sidebar, Entity | Singly-linked | Doubly-linked sentinel | No |
| **Alloc-Only Pool** (alloc+free, no iterate) | File, Escort, Viewport, Production | Singly-linked | None (or per-owner) | No |
| **Queue Pool** (alloc+free+FIFO) | GameEvent | Singly-linked | Queue chain | No |
| **Frame Allocator** (bulk reset) | CplcEntityInViewport, TileCollisions | None | Linear head pointer | Reset to base each frame |
| **Coroutine Pool** (special) | Coroutine | Singly-linked | Scheduler-managed | No |
