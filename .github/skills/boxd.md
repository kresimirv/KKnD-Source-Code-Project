# BOXD — Collision & Tile Occupancy System

## Overview

BOXD is KKND's collision/bounding-box subsystem. It does **two distinct jobs**:

1. **Physics collision grid** — 3D AABB overlap detection for entity-vs-entity and entity-vs-terrain-shape collisions (ramps, floors, walls, slopes, corners). Runs every tick for moving entities.
2. **Tile occupancy map** — 2D grid tracking which units occupy which tiles, used for pathfinding classification and infantry slot management.

The name "BOXD" comes from the level file section (`LVL_FindSection("BOXD")`) containing pre-authored terrain collision shapes.

---

## Level Data Structures

### LevelBoxd (from file)

```c
struct LevelBoxd {
    LevelBoxdSurface *layers[1];  // only layer[0] is ever used
};

struct LevelBoxdSurface {
    int max_collision_buckets;    // pool size for per-tick collision pairs
    int world_to_tile_x;         // bit-shift: world_x >> this = tile_x (always 13 in practice)
    int world_to_tile_y;         // bit-shift: world_y >> this = tile_y (always 13 in practice)
    int num_tiles_x;             // grid width (same as terrain grid in practice)
    int num_tiles_y;             // grid height
    BoxdTile *tiles[1];          // array of pointers, one per grid cell → NULL-terminated shape list
};
```

### BoxdCollisionBox (used by both entities and terrain)

```c
struct BoxdCollisionBox {
    BoxdCollisionType type;  // handler index (0–19, or negative specials)
    int x;                   // min-X (relative to entity, OR absolute for terrain)
    int y;                   // min-Y
    int _field_c;            // unused/padding (possibly min-Z in 3D)
    int z;                   // max-X
    int w;                   // max-Y
};
```

**For entity shapes:** x/y/z/w are offsets relative to entity position. World AABB = `entity->x + box->x` through `entity->x + box->z`.

**For terrain shapes (from BOXD level section):** x/y/z/w are **absolute world coordinates**. Terrain init checks overlap via `box->x < tile_world_x + 0x2000 && box->z > tile_world_x` (kknd.c:17906).

**Usage in BOXD_collide_entity overlap test:**
```c
// Obstacle AABB (world-space):
obs_min_x = obstacle->entity->x + obs_box->x;
obs_max_x = obstacle->entity->x + obs_box->z;
obs_min_y = obstacle->entity->y + obs_box->y;
obs_max_y = obstacle->entity->y + obs_box->w;
// Overlap: obs_min < mover_max && obs_max > mover_min (both axes)
```

### Terrain Grid (runtime, derived from BOXD)

```c
struct TerrainTile {
    TerrainTileFlags1 flags1;   // occupancy + passability
    TerrainTileFlags2 flags2;   // friendly-slot tracking + specials
    char _pad_2;
    char _pad_3;
    Unit *units[5];             // up to 5 infantry or 1 vehicle/building in slot 0
};
```

**Grid relationship:** The terrain grid dimensions are derived from the BOXD grid:
```c
g_map_num_tiles_x = boxd.num_tiles_x << (world_to_tile_x - 13);
g_map_num_tiles_y = boxd.num_tiles_y << (world_to_tile_y - 13);
```

If `world_to_tile_x == 13` (the standard value), shift = 0 → **terrain grid and BOXD collision grid are the same size** (1:1 tile mapping). The code *supports* BOXD tiles being larger (coarser) than terrain tiles — the terrain init loop iterates `1 << (world_to_tile_x - 13)` terrain tiles per BOXD tile — but in practice all observed levels use `world_to_tile_x = 13`, making them identical grids.

**Key difference is conceptual, not dimensional:**
- **BOXD collision grid** (`g_tiles[]`, `g_tiles_num_x/y`): stores pre-authored terrain collision shapes (ramps, walls, floors). Indexed via `>> g_world_to_tile_x`. Used by physics collision system.
- **Terrain occupancy grid** (`g_terrain[]`, `g_map_num_tiles_x/y`): runtime unit tracking (who's on which tile). Always indexed via `>> 13` (hardcoded). Used by pathfinding and unit placement.

---

## Two Collision Systems

### System 1: Physics Collision Grid (Entity-vs-Entity/Terrain shapes)

**Lifecycle per tick:**
1. `BOXD_rebuild_collision_grid(entity_list)` — Iterates all entities, computes which BOXD tiles each entity's shape overlaps (including velocity), inserts into a spatial hash (linked-list buckets per tile).
2. `BOXD_collide_entity(entity, direction, bump)` — For each moving entity, checks overlapping bucket entries. If shapes intersect, dispatches to the appropriate collision response function.

**Key globals:**
- `g_tile_collisions` — `BoxdCollisionBucket**` array indexed by tile, each entry → linked list of overlapping entities
- `g_tile_collisions_pool` — Pre-allocated pool of `BoxdCollisionBucket` nodes (size 0x17700 = 96000 bytes)
- `g_tile_collisions_head` — Pool allocator cursor
- `g_tiles` — `BoxdCollisionBox**` array from BOXD level section (one per cell → NULL-terminated shape list). **NOTE:** Only used at init + tracked-but-never-read in BOXD_collide_entity (dead reference). Terrain passability is baked into g_terrain flags at init time; terrain entities handle runtime collisions via their MOBD shapes
- `g_world_to_tile_x/y` — Bit-shift for world→tile coordinate conversion
- `g_tiles_num_x/y` — BOXD grid dimensions

**BoxdCollisionBucket:**
```c
struct BoxdCollisionBucket {
    BoxdCollisionShape *shape;
    Entity *entity;
    BoxdCollisionBucket *next;
};
```

### System 2: Tile Occupancy Map (Pathfinding)

**Purpose:** Track which units sit on which terrain tiles for pathfinding decisions.

**Key functions:**
- `BOXD_place_unit(unit, map_x, map_y, pos)` → claims a tile slot
- `BOXD_remove_unit_from_tile(unit, map_x, map_y, pos)` → releases a tile slot
- `BOXD_remove_unit(unit, map_x, map_y, pos)` → XL-aware removal (2x2 grid)
- `BOX_classify_tile_for_pathing(unit, map_x, map_y, tile)` → returns passability classification

---

## Collision Shapes & Handlers

### BoxdCollisionShape

```c
struct BoxdCollisionShape {
    BoxdCollisionBox *box;  // NULL-terminated: array of shapes, last has box==NULL
};
```

Shape comes from the MOBD animation frame: `entity->shape = anim_current_frame->shape`. Changes every animation frame.

### BoxdCollisionBox

See definition in Level Data Structures section above. For entity shapes: x/y/z/w relative to entity pos. For terrain shapes from BOXD section: absolute world coords.

### BoxdCollisionType — Handler Index

```c
enum BoxdCollisionType {
    BoxdCollisionType_0 = 0,    // Root unit shape (collides_with=1, category=2)
    BoxdCollisionType_1 = 1,    // Solid wall
    BoxdCollisionType_2 = 2,    // Floor
    BoxdCollisionType_3 = 3,    // Ramp left-to-right
    BoxdCollisionType_4 = 4,    // Ramp right-to-left
    BoxdCollisionType_5 = 5,    // Slope left
    BoxdCollisionType_6 = 6,    // Slope right
    BoxdCollisionType_7 = 7,    // Floor (alt)
    // 8-12: Solid (various terrain shapes)
    // 13: Corner right
    // 14: Corner left
    // 15: Solid
    // 16: Empty (no collision — category=0, collides_with=0)
    // 17: Projectile target zone (collides_with=5, cat=0)
    // 18: Cursor hitbox (mover_response=BOXD_collide_cursor)
    // 19: Unit body blocker (collides_with=0, cat=4)
    BoxdCollisionType_Skip = -3,    // Skip this entity entirely (early-out, never inserted into grid)
    BoxdCollisionType_OffGrid = -2, // Not on collision grid (flying units — skipped in rebuild)
    BoxdCollisionType_Always = -1,  // Always collide (category = -1, matches any bitmask)
};
```

### g_unit_collision_handlers[20]

Global handler table. Each entity sets `entity->collider = g_unit_collision_handlers` (array base). The `BoxdCollisionBox.type` field indexes into this — so `entity->collider[box->type]` gives the handler for that shape.

| Index | collides_with | category | mover_response | obstacle_response |
|-------|---------------|----------|----------------|-------------------|
| 0 | 1 | 2 | NULL | NULL | ← Unit root shape |
| 1 | 0 | 1 | NULL | collide_solid |
| 2 | 0 | 1 | NULL | collide_floor |
| 3 | 0 | 1 | NULL | collide_ramp_ltr |
| 4 | 0 | 1 | NULL | collide_ramp_rtl |
| 5 | 0 | 1 | NULL | collide_slope_left |
| 6 | 0 | 1 | NULL | collide_slope_right |
| 7 | 0 | 1 | NULL | collide_floor |
| 8-12 | 0 | 1 | NULL | collide_solid |
| 13 | 0 | 1 | NULL | collide_corner_right |
| 14 | 0 | 1 | NULL | collide_corner_left |
| 15 | 0 | 1 | NULL | collide_solid |
| 16 | 0 | 0 | NULL | NULL | ← empty/disabled |
| 17 | 5 | 0 | NULL | NULL | ← projectile hit zone |
| 18 | 2 | 0 | collide_cursor | NULL | ← cursor hover detection |
| 19 | 0 | 4 | NULL | collide_solid | ← unit body blocker |

---

## Collision Dispatch — Step-by-Step Algorithm

### Phase 1: Spatial Hash Build (`BOXD_rebuild_collision_grid`)

Called once per tick before any collisions:

1. Clear entire `g_tile_collisions[]` to NULL (memset zero)
2. Reset pool allocator: `g_tile_collisions_head = g_tile_collisions_pool`
3. For each entity in global entity list:
   a. Skip if no shape (`entity->shape == NULL`)
   b. Read root type: `type = entity->shape->box->type`
   c. **Insertion gate:**
      - `type == BoxdCollisionType_OffGrid (-2)` → skip entirely (flying units)
      - `type >= 0` AND `entity->collider[type].category == 0` → skip (invisible to all, e.g. type 16)
      - Otherwise → insert (includes negative types -1 and -3... wait, -3 also skips — but code says `if type != -2 goto insert`)
      - **Actual logic:** `if (type >= 0) { if (!category) skip; else insert; } else if (type != -2) { insert; }`
      - So: type=-1 (Always) → inserted. type=-3 (Skip) → inserted into spatial hash BUT will be early-exited in collide_entity
   d. Compute swept AABB: extends shape bounds by `x_speed`/`y_speed` in movement direction
   e. Convert to tile range (clamped to grid bounds)
   f. For each tile in range: allocate bucket node from pool, link into that tile's chain

**Insertion stores:** the entity's entire `BoxdCollisionShape*` (pointer to shape array), not just the root box. This is important — collision testing iterates ALL sub-shapes.

### Phase 2: Position Update + Collision Check (per-entity)

The tick loop does:
```c
entity->x += entity->x_speed;
if (x_speed > 0) BOXD_collide_entity(entity, NegativeX, bump=1);
if (x_speed < 0) BOXD_collide_entity(entity, PositiveX, bump=1);

entity->y += entity->y_speed;
if (y_speed > 0) BOXD_collide_entity(entity, NegativeY, bump=1);
if (y_speed < 0) BOXD_collide_entity(entity, PositiveY, bump=1);

entity->z += entity->z_speed;

// If no speed at all — still check (gravity/overlap resolution):
if (!moved) BOXD_collide_entity(entity, NegativeX, bump=is_on_grid);
```

**Direction convention:** The `direction` param is the axis FROM WHICH the obstacle pushes back. Moving +X → obstacle on the right pushes NegativeX.

### Phase 3: Collision Detection (`BOXD_collide_entity`)

**Entry conditions:**
1. `g_current_lvl_boxd_verified` must be set (level loaded)
2. Entity must have a shape
3. Multiplayer netz check (skip disabled entities)

**Step 3a — Mover root shape setup:**
```c
box = entity->shape->box;          // root (first) box of mover
type = box->type;
if (type == BoxdCollisionType_Skip) return;   // EARLY OUT — mover is Skip
if (type < 0)
    collides_with = -1;            // Always: matches everything
else
    collides_with = entity->collider[type].collides_with_categories;
    if (collides_with == 0) return; // mover can't hit anything
```

**Step 3b — Compute mover world AABB:**
```c
mover_min_x = entity->x + box->x;
mover_max_x = entity->x + box->z;
mover_min_y = entity->y + box->y;
mover_max_y = entity->y + box->w;
```

**Step 3c — Determine tile scan range:**
Convert mover AABB to tile coordinates, clamp to grid bounds. This gives a rectangle of tiles to check.

**Step 3d — Iterate all bucket entries in tile range:**
For each `BoxdCollisionBucket` node in each tile:
- Skip self: `if (bucket->entity == entity) continue`
- Get obstacle's shape list: `obstacle_shape = bucket->shape`

**Step 3e — Iterate obstacle's sub-shapes:**
```c
for each sub-shape in obstacle's shape array (until box==NULL):
```

**Step 3f — Category/collides_with bitmask test (THE GATE):**
```c
obstacle_type = obstacle_shape->box->type;
if (obstacle_type >= 0)
    obstacle_category = entity->collider[obstacle_type].category;
else
    obstacle_category = -1;   // Always-type obstacle matches everything

if ((mover_collides_with & obstacle_category) == 0)
    continue;  // NO MATCH — these two shapes can't interact
```

**THIS IS THE KEY INSIGHT:** Both `collides_with` and `category` are looked up from the **MOVER's** handler table (`entity->collider`), not the obstacle's. This works because all entities share the same `g_unit_collision_handlers` array. The test is: "does my shape's outgoing mask include this obstacle shape's category?"

**Step 3g — AABB overlap test:**
```c
obs_min_x = obstacle->entity->x + obs_box->x;
obs_max_x = obstacle->entity->x + obs_box->z;
obs_min_y = obstacle->entity->y + obs_box->y;
obs_max_y = obstacle->entity->y + obs_box->w;

if (obs_min_x >= mover_max_x) continue;  // no X overlap
if (obs_max_x <= mover_min_x) continue;
if (obs_min_y >= mover_max_y) continue;  // no Y overlap
if (obs_max_y <= mover_min_y) continue;
// → OVERLAP CONFIRMED
```

**Step 3h — Response dispatch (TWO PATHS depending on mover root type):**

#### Path A: Mover root type >= 0 (NORMAL — most entities)

```c
handler = entity->collider;
response = handler[mover_box->type].mover_response;   // mover's own response
if (!response)
    response = handler[obstacle_type].obstacle_response;  // obstacle's response
if (response)
    moved = response(entity, obstacle->entity, direction, &mover_state, &obstacle_state);
```

**Logic:** Try mover's `mover_response` first. If NULL (which it is for type 0 — normal units), fall back to obstacle's `obstacle_response`. This means:
- Unit (type 0) hits wall (type 1) → mover_response[0]=NULL → obstacle_response[1]=collide_solid ✓
- Unit (type 0) hits floor (type 2) → mover_response[0]=NULL → obstacle_response[2]=collide_floor ✓
- Cursor (type 18) hits unit root (type 0) → mover_response[18]=collide_cursor → called ✓ (obstacle_response[0]=NULL is never reached)

#### Path B: Mover root type < 0 (ALWAYS type — special entities)

When the mover's root is negative (Always=-1), it enters a **nested loop** over the mover's OWN sub-shapes (starting from shape[1] onward):

```c
for each mover_sub in entity->shape[1..N]:  // skip root, iterate sub-shapes
    sub_type = mover_sub->box->type;
    // Reverse check: obstacle's category must match THIS sub-shape's collides_with
    if ((obstacle_category & entity->collider[sub_type].collides_with_categories) == 0)
        continue;
    // AABB test: mover_sub vs obstacle (world-space)
    if (!overlaps(mover_sub, obstacle_sub))
        continue;
    // Dispatch:
    response = entity->collider[mover_sub->box->type].mover_response;
    if (!response)
        response = entity->collider[obstacle_shape->box->type].obstacle_response;
    if (response)
        moved = response(entity, obstacle->entity, direction, &mover_sub_state, &obstacle_state);
```

**Why this exists:** Entities with `Always` root type have MULTIPLE collision sub-shapes that each need independent testing. The root shape (type=-1) serves only as the spatial-hash insertion key and bitmask override. The real collision work happens between individual sub-shapes and each obstacle.

**Use case:** A complex entity (e.g., a multi-part building or a cursor with multiple hit zones) where different sub-parts have different collision behaviors — one sub-shape might be a cursor hitbox (type 18) while another is a projectile zone (type 17).

**Step 3i — Post-response AABB refresh:**
If response returns TRUE (entity was moved):
```c
// Recompute mover's world AABB (entity position changed by push-out)
mover_min_x = entity->x + box->x;
mover_max_x = entity->x + box->z;
mover_min_y = entity->y + box->y;
mover_max_y = entity->y + box->w;
```
Then continues checking remaining obstacles with updated position.

---

## Category/CollidesWith Design Philosophy

The system uses **asymmetric bitmask filtering** to create one-way collision relationships:

| Role | collides_with | category | Meaning |
|------|---------------|----------|---------|
| Unit root (0) | 1 | 2 | "I hit things with category containing bit 1" / "Others see me as category 2" |
| Terrain shapes (1-15) | 0 | 1 | "I never initiate" / "Units can hit me (they want bit 1)" |
| Empty (16) | 0 | 0 | Invisible to everyone |
| Projectile zone (17) | 5 (=1+4) | 0 | "I hit terrain(1) AND unit bodies(4)" / "Nobody can hit me" |
| Cursor (18) | 2 | 0 | "I hit unit roots (category 2)" / "Nobody can hit me" |
| Unit body (19) | 0 | 4 | "I never initiate" / "Projectiles can hit me (they want bit 4)" |

**Category bits in use:** 1=terrain, 2=unit-root, 4=unit-body. `collides_with` is a bitmask selecting which categories to collide with.

**Collision proceeds when:** `(mover.collides_with & obstacle.category) != 0`

This means:
- Unit root (cw=1) vs terrain (cat=1): `1 & 1 = 1` → YES
- Unit root (cw=1) vs unit body (cat=4): `1 & 4 = 0` → NO (units don't push each other via physics)
- Cursor (cw=2) vs unit root (cat=2): `2 & 2 = 2` → YES
- Projectile (cw=5) vs terrain (cat=1): `5 & 1 = 1` → YES
- Projectile (cw=5) vs unit body (cat=4): `5 & 4 = 4` → YES
- Terrain (cw=0) vs anything: `0 & X = 0` → NEVER (terrain never initiates)

### Why `entity->collider` is an array pointer

`entity->collider` always points to `g_unit_collision_handlers[0]`. It's used as array base — `entity->collider[type]` gives the handler for shape type `type`. All entities share the same table. Non-collidable entities point to `&g_null_collision` (a single zeroed handler: collides_with=0, category=0, both responses=NULL) — this means indexing past it is undefined, but since non-collidable entities never reach the collision code (they're filtered by the rebuild gate: category=0 → not inserted into spatial hash), it's safe.

### BoxdCollisionHandler dispatch

```c
struct BoxdCollisionHandler {
    int collides_with_categories;
    int category;
    bool (*mover_response)(...);     // mover-driven (cursor, projectile)
    bool (*obstacle_response)(...);  // obstacle-driven (terrain shapes)
};
```

**Dispatch priority:** `mover_response` checked first; if NULL, falls back to `obstacle_response` of the target shape. In practice:
- Units colliding with terrain: mover=NULL → terrain's obstacle_response handles it
- Cursor colliding with units: mover=collide_cursor → cursor handles it
- **Never both:** No handler entry has both slots filled. Architecture supports it (mover would always win), but unused.

---

## Collision Response Functions

All share signature: `BOOL fn(Entity *mover, Entity *obstacle, BoxCollisionAxis axis, BoxdCollisionState *mover_state, BoxdCollisionState *obstacle_state)`

### BoxdCollisionState

```c
struct BoxdCollisionState {
    int x1, y1, z1;  // AABB min (world-space)
    int x2, y2, z2;  // AABB max (world-space)
};
```

### BoxCollisionAxis

```c
enum BoxCollisionAxis {
    NegativeX = 0,  // pushed left
    PositiveX = 1,  // pushed right
    NegativeY = 2,  // pushed up (Y = vertical/height)
    PositiveY = 3,  // pushed down
    NegativeZ = 4,  // pushed toward camera
    PositiveZ = 5,  // pushed away from camera
};
```

**NOTE:** The axis naming maps to an isometric 3D space where:
- X = left-right (screen horizontal)
- Y = up-down (height/altitude)
- Z = depth (screen vertical / into-screen)

### Response functions:
- **collide_solid** — Full 3D push-out. Moves entity out of penetration along the collision axis. Sets task flags (0x80000–0x1000000) indicating which direction was blocked.
- **collide_floor** — Only responds to NegativeY (falling onto floor). Clamps entity.y to floor surface. Zeroes y_speed.
- **collide_ramp_ltr** — Left-to-right ramp. Interpolates Y height based on X position within ramp bounds.
- **collide_ramp_rtl** — Right-to-left ramp. Same but reversed direction.
- **collide_slope_left** — Slope pushes entity upward (Y direction) based on its Y1 position relative to slope geometry.
- **collide_slope_right** — Mirror of slope_left.
- **collide_corner_right** — Hybrid: acts as floor (NegativeY axis) or wall (PositiveX axis).
- **collide_corner_left** — Mirror: floor (NegativeY) or wall (NegativeX).
- **collide_cursor** — Sends `TaskMessage_MouseHover` to mover's task. Returns 0 (no position adjustment).

---

## Tile Occupancy System

### TerrainTileFlags1 (passability + occupancy)

```c
bits [0:4] — Infantry slot mask (5 slots, one bit each)
bit 5     — Obstructed (terrain edges, landscape features)
bit 6     — Blocked (impassable; buildings with speed=0 also set this)
bit 7     — VehicleOrBuilding (single entity fills whole tile)
```

Combined: `0x60 = Impassible` (both Obstructed + Blocked)

### TerrainTileFlags2 (ownership + specials)

```c
bits [0:4] — Friendly infantry slot mask (mirrors flags1 slots)
bit 5     — UNUSED (never read or written; preserved accidentally by 0xE0 mask)
bit 6     — CantPlaceBuilding (multi-tile building footprint extension)
bit 7     — OilPatch (WRITE-ONLY — set but never queried; oil lookup uses linked list instead)
```

**Masks used in code:**
- `& 0x1F` — isolate friendly slot bits
- `& 0x40` — test/set CantPlaceBuilding
- `& 0xE0` — preserve upper 3 bits during clear (keeps CantPlaceBuilding + OilPatch + unused bit 5)
- `| 0x1F` — set all friendly slots (vehicle placement, friendly)
- `| 0x40` — mark CantPlaceBuilding

**Enemy detection trick:** `(flags1 ^ flags2) & 0x1F` — nonzero means enemy infantry present (occupied slots that aren't friendly).

### Infantry Slot System

Infantry units occupy one of 5 sub-positions within a tile:
- Slot 0: center (4096, 4096)
- Slot 1: top-left (2048, 2048)
- Slot 2: top-right (6144, 6144)
- Slot 3: bottom-left (6144, 2048)
- Slot 4: bottom-right (2048, 6144)

Wait — the actual position layout from `BOXD_adjust_unit_position_x/y`:

| Slot | X offset | Y offset |
|------|----------|----------|
| 0 (center) | 4096 | 4096 |
| 1 | 2048 | 2048 |
| 2 | 6144 | 6144 |
| 3 | 2048 | 6144 |
| 4 | 6144 | 2048 |

So the layout is:
```
     Slot1(2048,2048)       Slot4(6144,2048)

            Slot0(4096,4096)

     Slot3(2048,6144)       Slot2(6144,6144)
```

### g_next_infantry_slot[32]

Lookup table indexed by current `flags1 & 0x1F` bitmask. Returns next available slot when a unit's preferred slot is already taken. Sequence fills: 4→3→2→1→0→Invalid.

### UnitSize determines tile behavior

| UnitSize | Value | Tile footprint | Notes |
|----------|-------|----------------|-------|
| Small | 512 | 1/5 of tile | Infantry, uses slots |
| Regular | 128 | 1 tile | Vehicles, buildings |
| XL | 4096 | 2×2 tiles | Autocannon, missile crab, mobile outpost |

### BOXD_place_unit flow

1. Bounds check (map_x, map_y in range)
2. **Infantry path:**
   - If slots not full (`flags1 & 0x1F < 0x1F`), find available slot
   - If preferred slot taken, use `g_next_infantry_slot` lookup
   - Set flags1 bit, set flags2 friendly bit, store unit pointer
3. **Vehicle/Regular path:**
   - If infantry already on tile AND unit can_crush AND all infantry are enemies → crush them (send TaskMessage_Crushed), claim tile
   - Set flags1 to `0x9F` (all infantry bits + VehicleOrBuilding)
   - If building (speed=0): set Blocked flag, check CantPlaceBuilding
4. **XL path:** `BOXD_40E1B0` iterates 2x2 grid, calls place_unit on each

### BOXD_remove_unit_from_tile flow

1. Buildings: clear CantPlaceBuilding (flags2 bit 6)
2. **Vehicle/building** (flags1 & 0x80): clear to just Obstructed, zero all occupancy
3. **Infantry**: clear single slot bit in flags1, clear friendly bit in flags2, null unit pointer

---

## Pathfinding Classification

### BoxdPathingClassification

```c
enum BoxdPathingClassification {
    Boxd_Impassable = 0,          // terrain wall, no way through
    Boxd_PartiallyOccupied = 1,   // units present but slots available
    Boxd_Clear = 2,               // free tile
    Boxd_FullyOccupied = 3,       // all slots full / building present
};
```

### BOX_classify_tile_for_pathing logic

**Regular units:**
1. If terrain blocked (0x60) and NOT vehicle/building → Impassable
2. If unit already on tile → Clear (self not counted)
3. If no units on tile → Clear
4. If infantry present AND can_crush AND all are enemies → Clear (will crush)
5. Otherwise: check `(flags1 ^ flags2) & 0x1F` — nonzero = PartiallyOccupied (enemy infantry, might move), zero = FullyOccupied (friendly or full)

**Small (infantry):**
1. If terrain blocked and not vehicle → Impassable
2. If all 5 slots full: friendly present → FullyOccupied, enemy → PartiallyOccupied
3. Otherwise → Clear

**XL units:**
- Iterates all 4 tiles (2×2). If any tile Impassable → whole thing Impassable. Aggregates partial/full across all 4.

### BOXD_40DA90 — Movement-tick tile update

Called during unit movement. For regular units:
1. Compute new tile from `(entity.x + x_speed, entity.y + y_speed)`
2. If tile changed: try `BOXD_place_unit` on new tile
3. If successful: remove from old tile, update unit's map_x/map_y
4. If failed: classify the tile (return pathfinding info to caller)

For XL: remove from all old tiles, try placement at new position, restore if failed.

---

## Raycast System (BOXD_pathing_bresenham_raycast)

Used for pathfinding line-of-sight checks. Walks a Bresenham line from unit to target, classifying each tile along the way.

### BoxdRaycastResult

```c
enum BoxdRaycastResult {
    ClearWithWaypoints = 0,   // path clear, waypoints found
    UnitObstacle = 1,         // blocked by another unit
    NoWaypoints = 2,          // no waypoints generated
    TerrainObstacle = 3,      // blocked by terrain
    ClearStraightLine = 4,    // completely clear, straight line
    InvalidState = 5,         // error/fallback
};
```

The raycast separates into X-major and Y-major cases (standard Bresenham), calling `BOX_classify_tile_for_pathing` at each step. Records waypoints at direction changes.

### BOXD_41C130 — Line-of-sight passability check

Dispatches by unit size to `BOXD_41C190` (regular/small) or `BOXD_41C250` (XL). Uses same Bresenham approach but returns simple BOOL (clear/blocked).

---

## Initialization & Teardown

### BOXD_collisions_init()
1. Find "BOXD" section in level file
2. Allocate collision bucket pool (96KB)
3. Read grid parameters from layer[0]
4. Allocate tile-pointer array

### LVL_InitBoxdTerrain()
1. Find "BOXD" section
2. Compute terrain grid dimensions (finer than BOXD grid)
3. Allocate `g_terrain` array
4. Walk BOXD tiles, filter shapes with type 6/7/8 (terrain marker types — NOT collision handler indices):
   - Type 6 → `flags1 = Blocked|AllSlots (0x5F)` — fully impassable terrain
   - Type 7 → `flags1 = Obstructed|AllSlots (0x3F)` — obstructed (landscape edges)
   - Type 8 → `flags1 = None; flags2 = CantPlaceBuilding` — building exclusion zone
5. For each marker shape, AABB overlap test against terrain tile bounds to determine which tiles to flag
6. Initialize direction-offset table (`dword_478BE8[8]`)

### BOXD_terrain_cleanup()
Just `free(g_terrain)`.

### BOXD_cleanup()
Free collision bucket pool and tile-pointer array.

---

## Entity Lifecycle Integration

### Spawn
- Entity gets `collider = g_unit_collision_handlers` (for collidable units) or `collider = &g_null_collision` (for UI/non-physical entities)
- Shape assigned from MOBD animation frame
- `BOXD_place_unit` called to claim terrain tile
- `entity->is_on_collision_grid = 1` set

### Per-tick (moving entities)
1. `BOXD_rebuild_collision_grid` — rebuild spatial hash
2. Entity moves (x += x_speed, y += y_speed)
3. `BOXD_collide_entity` — resolve physics collisions
4. `BOXD_40DA90` — update tile occupancy

### Death/removal
- `BOXD_remove_unit` — clear from terrain tiles
- collider set to `&g_null_collision`
- shape may be set to NULL

### Building placement
- Uses `pos = UnitTilePosition_40` → special flag that sets CantPlaceBuilding
- Buildings with speed=0 set Blocked flag

---

## XL Unit Special Handling

XL units (2×2 tiles) have special paths everywhere:
- `BOXD_40E1B0` — Place across 2×2 grid
- `BOXD_remove_unit` — Remove from 2×2 grid
- `BOXD_40DF50` — Update friendly flags across 2×2 grid
- `BOXD_adjust_xl_movement_destination` — Snap movement target to valid 2×2 position
- `BOXD_40E000` — Scan 2×2 area for pathfinding classification
- `BOXD_41C250` — XL-specific line-of-sight check

XL position offset: entity center ± 4096 world units = 2×2 tile coverage.

---

## Key Coordinate Facts

- 1 terrain tile = 8192 (0x2000) world units
- World→tile: `>> 13` (world_to_tile_x/y from BOXD)
- Infantry sub-positions: offsets of 2048/4096/6144 within tile
- XL extent: ±4096 from center = spans 2 tiles

---

## BOXD_40F230 — Friendly Flag Update

Updates the `flags2` friendly bits for a unit's tile. Called when unit changes allegiance or moves. For infantry: sets/clears the specific slot bit. For vehicles: sets/clears all 5 bits (0x1F).

---

## BOXD_40EE10 — Find Building At Tile

Returns the building unit at (map_x, map_y), checking both the tile and the tile above (flags2 CantPlaceBuilding linkage). Only returns units in the building type range (Drillrig through AlchemyHall).

---

## BOXD_40EE70 — Is Tile Free For Building

Returns TRUE if tile has no units AND no CantPlaceBuilding flag. Used during building placement validation.

---

## BOXD_413860 — Emergency Placement (find nearby clear tile)

When a unit can't be placed at its current position (e.g., after spawn from building), scans nearby tiles in a diamond pattern to find a clear tile with line-of-sight back to origin.

## BOXD_413A90 — Fallback spiral placement

Last-resort placement: spiral search around entity position for any passable tile.
