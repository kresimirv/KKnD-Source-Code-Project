# BOXD — TODO / Rename Proposals / Concerns

## Rename Proposals

### Functions

| Current Name | Proposed Name | Reason |
|---|---|---|
| `BOXD_40DA90` | `BOXD_update_tile_on_move` | Called during movement tick to update tile occupancy when entity moves between tiles |
| `BOXD_40DE80` | `BOXD_place_unit_world_coords` | Wrapper that converts world coords to tile coords before calling BOXD_place_unit (XL-aware) |
| `BOXD_40DF50` | `BOXD_update_friendly_flags` | Updates flags2 friendly bits for unit's tile(s), XL-aware |
| `BOXD_40E000` | `BOXD_classify_area_for_pathing` | Like BOX_classify_tile_for_pathing but handles XL 2×2 area and tracks nav_obstacle |
| `BOXD_40E1B0` | `BOXD_place_unit_xl` | XL placement across 2×2 grid, returns pathing classification on failure |
| `BOXD_40ED00` | `BOXD_classify_tile_simple` | Simpler version of BOX_classify_tile_for_pathing, used during scanning (returns 0/1/2/3 not the enum) |
| `BOXD_40EDF0` | `BOXD_is_tile_passable_ignoring` | Checks if tile blocked, ignoring two specific units (self + target) |
| `BOXD_40EE10` | `BOXD_find_building_at_tile` | Returns building-type unit at tile, checks CantPlaceBuilding linkage |
| `BOXD_40EE70` | `BOXD_is_tile_free_for_building` | Returns true if tile empty and no CantPlaceBuilding flag |
| `BOXD_40F230` | `BOXD_set_friendly_flag` | Sets or clears friendly flag in flags2 for given unit's slot |
| `BOXD_413860` | `BOXD_find_nearby_clear_tile` | Emergency placement: diamond-scan for clear tile with LOS |
| `BOXD_413A90` | `BOXD_find_nearby_clear_tile_spiral` | Fallback spiral search for any passable tile |
| `BOXD_41C130` | `BOXD_line_of_sight_check` | Size-dispatching LOS check (regular/small → 41C190, XL → 41C250) |
| `BOX_classify_tile_for_pathing` | `BOXD_classify_tile_for_pathing` | Consistent prefix (currently missing 'D') |
| `sub_41BA30` | `BOXD_raycast_x_major` | X-major Bresenham raycast leg |
| `sub_41BC60` | `BOXD_raycast_y_major` | Y-major Bresenham raycast leg |
| `sub_41BE90` | `BOXD_raycast_classify_step` | Per-step classifier during raycast walk |
| `sub_41C060` | `BOXD_raycast_finalize` | Raycast result assembly |
| `BOXD_41C190` | `BOXD_los_check_regular` | LOS for regular/small units |
| `BOXD_41C250` | `BOXD_los_check_xl` | LOS for XL units |
| `BOXD_41C660` | `BOXD_los_bresenham_x_major` | LOS X-major walk |
| `BOXD_41C890` | `BOXD_los_bresenham_y_major` | LOS Y-major walk |

### Structures

| Current Name | Proposed Name | Reason |
|---|---|---|
| `LevelBoxdSurface` | `BoxdGridLayer` | It's a spatial grid layer, not a "surface" |
| `BoxdCollisionBucket` | `BoxdSpatialHashEntry` | More accurate — it's a spatial hash bucket entry |
| `BoxdTile` | **DELETE** | Not a real struct — IDA artifact. Replace with `BoxdCollisionBox**` (NULL-terminated pointer array per cell) |

### Struct Fields

| Location | Current | Proposed | Reason |
|---|---|---|---|
| `BoxdCollisionBox._boxd_collision_box_field_c` | `_field_c` | `h` or `depth` | Likely the Z/depth extent (3D AABB has 6 values: x,y,z for min + w,h,d for extents). Currently only x,y are used as 2D but the field exists |
| `BoxdTile._boxd_tile_field_c` | `_field_c` | Same as above — probably depth/Z | |
| `Entity.is_on_collision_grid` | OK | Consider `is_collidable` | More semantic — flag means entity participates in collision detection |

### Enums

| Current Name | Proposed Name | Reason |
|---|---|---|
| `BoxdCollisionType_0` through `_7` | See table below | Should be named by terrain shape |
| `BoxCollisionAxis` | Acceptable | But values named `BoxCollisionDirection_*` — inconsistent with enum name |

**Proposed BoxdCollisionType names:**

| Value | Current | Proposed |
|---|---|---|
| 0 | BoxdCollisionType_0 | `BoxdCollisionType_UnitRoot` |
| 1 | BoxdCollisionType_1 | `BoxdCollisionType_Solid` |
| 2 | BoxdCollisionType_2 | `BoxdCollisionType_Floor` |
| 3 | BoxdCollisionType_3 | `BoxdCollisionType_RampLTR` |
| 4 | BoxdCollisionType_4 | `BoxdCollisionType_RampRTL` |
| 5 | BoxdCollisionType_5 | `BoxdCollisionType_SlopeLeft` |
| 6 | BoxdCollisionType_6 | `BoxdCollisionType_SlopeRight` |
| 7 | BoxdCollisionType_7 | `BoxdCollisionType_FloorAlt` |

---

## Concerns & Decompilation Issues

### 1. BoxdCollisionState aliasing in BOXD_collide_solid

The function accesses `a3[1].x1` and `a4[1].x1` — treating `BoxdCollisionState*` as an array. This is how the decompiler represents access to Z-axis values beyond the struct boundary. **Likely decompilation artifact:** The real struct probably has more fields, OR the caller passes a larger buffer and the function reads past the declared end.

**Proposed fix:** `BoxdCollisionState` should have 6 fields. The current struct has `x1,y1,z1,x2,y2,z2` — but the code accesses PAST z2. Check if `a3[1].x1` = offset 24 = a 7th int. Might need to expand the struct or treat as a flat `int[8]` buffer.

**Update:** Looking more carefully — the switch cases in collide_solid map:
- NegativeX: uses a3->y1, a3->y2, a4->y1, a4->y2 → these ARE the X extent
- NegativeY: uses a3->z1, a3->z2, a4->z1, a4->z2 → these ARE the Y extent
- NegativeZ: uses a3->x2, a3[1].x1, a4->x2, a4[1].x1 → THIS IS THE Z EXTENT

**Conclusion:** The struct layout is wrong! It should be:
```c
struct BoxdCollisionState {
    int type;    // or padding? (what we call x1)
    int min_x;   // y1
    int min_y;   // z1
    int max_x;   // x2
    int max_y;   // y2
    int max_z;   // z2
    int min_z;   // what's accessed as [1].x1
};
```

OR the naming is just confusing and the 6 fields represent min/max of 3 axes but accessed in non-obvious order. Need to verify against how they're computed in `BOXD_collide_entity`.

### 2. BoxdTile struct is a decompilation error

`BoxdTile` as defined by IDA is **entirely wrong**. The actual data at `tiles[cell_index]` is a **NULL-terminated array of `BoxdCollisionBox*` pointers** (one per terrain collision shape overlapping that cell).

In `LVL_InitBoxdTerrain` (kknd.c:17880–17953):
```c
v10 = *tiles;                          // first BoxdCollisionBox* in array
type = (int *)(*tiles)->type;          // dereferences .type AS A POINTER → BoxdCollisionBox
v20 = (_DWORD *)v25->type;            // same: reads through it as BoxdCollisionBox*
// v20[0]=type, v20[1]=x_min, v20[2]=y_min, v20[4]=x_max, v20[5]=y_max
...
type = (int *)v10->x;                  // .x is next pointer in the array (offset 4 = next slot)
v10 = (BoxdTile *)((char *)v10 + 4);  // advance by 4 bytes = next pointer
```

**Reality:** Each `tiles[i]` is a `BoxdCollisionBox**` — pointer to array of BoxdCollisionBox pointers. The IDA-generated `BoxdTile` struct should be deleted and replaced with `BoxdCollisionBox**`.

The terrain shapes use **absolute world coordinates** (not entity-relative). Overlap test: `shape->x < tile_world + 0x2000 && shape->z > tile_world` confirms min/max semantics.

**Action:** Delete `BoxdTile` struct, change `LevelBoxdSurface.tiles` type to `BoxdCollisionBox **tiles[1]`, rename `g_tiles` to `g_terrain_shapes`.

### 3. Dual grid ~~confusion~~ (RESOLVED)

~~Two grids exist with similar names at different resolutions.~~

**RESOLVED:** In practice, `world_to_tile_x` is always 13, so both grids have identical dimensions. The difference is purely conceptual:
- `g_tiles[]` (static): pre-authored terrain collision shapes (loaded from BOXD section, used only during terrain init to mark passability flags). **NOTE:** `g_tiles` is referenced but never dereferenced in `BOXD_collide_entity` — dead/vestigial code from an older design where terrain shapes were checked at runtime.
- `g_terrain[]` (runtime): unit occupancy tracking. Used by pathfinding and placement.

### 4. BOXD_40ED00 returns int, not BoxdPathingClassification

`BOXD_40ED00` returns 0/1/2/3 matching the enum values but typed as `int`. Likely same enum, just the decompiler didn't tag it.

### 5. g_unit_collision_handlers has 20 entries but BoxdCollisionType only goes to 7

The enum only defines 0-7 + special values (-3,-2,-1). But the handler table has 20 entries (indices 0-19). The enum needs extending to cover all terrain shape types actually used in MOBD/BOXD data.

### 6. `entity->collider` is array-accessed but typed as pointer

Code does `entity->collider[type].mover_response` — treating `collider` as an array base. The pointer always points to `g_unit_collision_handlers` (or `g_null_collision`). This is correct C but confusing — the struct type should document that it's used as array base.

### 7. UnitTilePosition_40 magic value

`UnitTilePosition_40 = 64` — used specifically during building placement to signal "mark as CantPlaceBuilding but don't set VehicleOrBuilding." Should be renamed to `UnitTilePosition_BuildingPlacement` or similar.

---

## Structural Proposals

### 1. Split BoxdCollisionType enum

Create two enums:
- `BoxdTerrainShapeType` — for the level file shapes (0-15+)
- `BoxdEntityCollisionMode` — for the special values (Skip, OffGrid, Always)

### 2. Rename BoxCollisionAxis enum values

Current: `BoxCollisionDirection_NegativeX` etc.
Either rename enum to `BoxCollisionDirection` or rename values to `BoxCollisionAxis_NegX`.

### 3. Document the task flags set by collision responses

`collide_solid` sets these task->flags_20 bits:
- 0x0080000 — blocked from PositiveZ direction
- 0x0100000 — blocked from NegativeZ direction
- 0x0200000 — blocked from PositiveY direction (landed on slope)
- 0x0400000 — blocked from NegativeY direction (hit floor/ceiling)
- 0x0800000 — blocked from PositiveX direction
- 0x1000000 — blocked from NegativeX direction

These should be an enum: `TaskCollisionFlags` or similar. Also `task->flags_24 |= task->flags_20` accumulates "ever-blocked" history.

### 4. Infantry slot position constants

Instead of raw magic numbers (2048, 4096, 6144), define:
```c
#define TILE_QUARTER  2048
#define TILE_CENTER   4096
#define TILE_THREE_Q  6144
```

### 5. Consider whether BOXD_40ED00 and BOX_classify_tile_for_pathing should merge

They have near-identical logic but return differently (BOXD_40ED00 doesn't handle "self already on tile" the same way). Could be one function with a flag parameter.

---

## Unknown/Uncertain Items

1. **BoxdCollisionBox._field_c** — Never read in any collision function. Might be dead/vestigial Z-extent that was planned but unused. Or padding for alignment.

2. **BoxdTile._field_c** — Same situation. The terrain init code only reads type, and compares fields as offsets. Need to verify if this is ever meaningful.

3. **Handler indices 8-12** — All map to `collide_solid`. Are these distinct terrain shapes in the art that just happen to have identical collision behavior? Or copy-paste?

4. **Handler index 17 (collides_with=5)** — collides with terrain(1) + unit-body(4). Used for projectile hit detection? Need to find what entities use this type index.

5. **Handler index 19 (category=4, obstacle=collide_solid)** — Unit body solid shape. Where is type=19 used in MOBD data? Likely the "hitbox" sub-shape for units that projectiles can hit.

6. **The `bump` parameter in BOXD_collide_entity** — Passed in but never used in the function body shown. Might affect behavior deeper in the call chain or be vestigial.

7. **`g_tile_collisions_head` advances in the inner loop of collide_entity** — This looks like a bug or artifact. The head pointer should only advance during rebuild, not during collision resolution. Might be the decompiler confusing loop counter with global state.

8. **TerrainTileFlags2 bit 0x20 — completely unused.** Never read, never written. The `& 0xE0` mask in clear operations preserves it as a side effect of wanting to keep bits 6+7 (CantPlaceBuilding + OilPatch). Bit 5 in flags2 has no purpose — likely reserved/dead.

9. **TerrainTile._pad_2 and _pad_3 are truly padding.** Never accessed anywhere in the codebase — not via struct fields, not via raw byte offsets, not via WORD/DWORD casts. The struct is 24 bytes: 1(flags1) + 1(flags2) + 2(pad) + 5×4(units) = 24. Pure alignment padding.

10. **`BOXD_place_unit` pos parameter is overloaded/dual-purpose:**
    - For infantry: `pos` is treated as boolean (0=enemy, 1=friendly) — shifted into flags2 as friendly-slot bit
    - For vehicles: `if (pos)` → set all 5 friendly bits; `if (!pos)` → clear friendly bits
    - Special value `UnitTilePosition_40 (=64)` → building shadow placement: clears flags1 (no physical block), sets CantPlaceBuilding in flags2
    - **Rename proposal:** Split into two concepts — `is_friendly` (bool) and building-specific placement mode. The pos=64 overload is confusing since enum is UnitTilePosition which implies a slot.

11. **Terrain init types 6/7/8 are NOT collision handler indices.** The `LVL_InitBoxdTerrain` outer loop filters `v12 == 6 || v12 == 7 || v12 == 8` to mark terrain occupancy flags. These are special "terrain marker" types — type 6 = Blocked, type 7 = Obstructed, type 8 = CantPlaceBuilding. Despite sharing the same `BoxdCollisionBox.type` field, they serve a completely different purpose from handler indices 0-19 used at runtime. In the handler table, index 6 = `collide_slope_right` and index 7 = `collide_floor` — totally unrelated! **Rename proposal:** These should be documented as `TerrainMarkerType_Blocked=6, TerrainMarkerType_Obstructed=7, TerrainMarkerType_CantPlace=8` in a separate enum context.

12. **flags1 bit 7 (VehicleOrBuilding) signed-char trick:** The pathing code checks `flags1 >= 0` (treating byte as signed char) to test "is NOT vehicle/building" — since bit 7 = 0x80 makes it negative when interpreted as signed. This is a C optimization pattern (`(char)flags1 < 0` ≡ `flags1 & 0x80`). Worth noting in code comments for clarity.

13. **TerrainTileFlags2_OilPatch (0x80) is write-only / dead.** Set once in `UNIT_Handler_OilPatch` (kknd.c:11359) but NEVER READ anywhere. Oil patch lookup (`oil_patch_407040_find`) uses a linked list with world-coordinate XOR match `((x ^ entity->x) & 0xFFFFE000) == 0` instead of tile flags. The bit is vestigial — possibly planned for minimap display or building placement exclusion but never hooked up. Safe to ignore.

14. **BOXD_40EE10 "find building at tile" uses v2[-1] backwards lookup.** When CantPlaceBuilding (0x40) is set on a tile, the function looks at the PREVIOUS tile in the flat array (`v2[-1]`) to find the building unit. This relies on multi-tile buildings storing their Unit* pointer on the first tile of their footprint, while subsequent tiles only get CantPlaceBuilding flag. The `map_y >= 1` guard prevents array underflow. **Note:** This only works correctly if CantPlaceBuilding tiles always follow the building's root tile in memory order (left-to-right, top-to-bottom). Buildings wider than 1 tile with gaps in their `collision_mask` could break this assumption.

15. **CantPlaceBuilding clear logic is type-restricted.** Only `!speed` OR `type == Outpost/Clanhall` clear the CantPlaceBuilding flag on removal (kknd.c:18480). Other unit types that set 0x40 (via UnitTilePosition_40) DON'T clear it on removal — potential leak? Or those tiles are cleared via some other mechanism. Needs investigation.

16. **BoxdPathingClassification enum is INCOMPLETE — missing value 4 (MovementSucceeded).** `BOXD_40DA90` (movement-tick tile update) and `BOXD_40E1B0` (XL placement) both return `4` to signal "tile placement succeeded, unit position updated." The enum only defines 0-3 (Impassable/PartiallyOccupied/Clear/FullyOccupied). All callers check `== 4` or `!= 4` as the success/failure branch:
    - kknd.c:23439, 23479, 23842: `if (BOXD_40DA90(unit) == 4)` → success path
    - kknd.c:24979, 25176: `if (BOXD_40DA90(unit) != 4)` → failure/reroute path
    - kknd.c:25878: `v2 = BOXD_40DA90(unit) == 4` → boolean success
    - Values 0-3 are only returned on FAILURE (placement failed → classify the blocking tile for pathfinding decision)
    - **Fix:** Add `Boxd_MovementSucceeded = 4` to enum. Function is really a dual-purpose "try-move-or-classify" — returns classification only when move fails, returns 4 when move succeeds. This makes it a tagged union of success(4) vs failure-reason(0-3). `BOXD_40ED00` (declared as `int` not enum) returns same {0,1,2,3} range — it's a pure classifier without the success path.
