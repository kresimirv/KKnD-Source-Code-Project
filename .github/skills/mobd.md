# MOBD System — Deep Investigation

## Overview

MOBD = "MOBile Data" — the sprite/animation data format for KKND. Each level file (.LVL) has a "MOBD" section containing all sprite sheets, animation sequences, collision shapes, and anchor points for every entity type in that level.

## Core Data Structures

### LevelMobd — The Top-Level Container

```c
struct LevelMobd {
  LevelMobdSurface *layers[1];  // variable-length; indexed by MobdId
};
```

Global: `g_current_lvl_mobd` — loaded from level file's "MOBD" section via `LVL_FindSection("MOBD")`.

**Indexed by `MobdId` enum (0–83, plus -1 for Invalid).** Each MobdId corresponds to one entity type's full sprite data. `g_current_lvl_mobd[mobd_id]` gives that entity's `LevelMobd` entry.

### LevelMobdSurface — Single Entity's Animation Bank

```c
struct LevelMobdSurface {
  MobdAnimFrame frames[1];  // variable-length flat byte blob
};
```

Accessed as: `g_current_lvl_mobd[mobd_id].layers[0]` — gets the only surface layer. No code ever accesses `layers[1]` or higher. (The `layers[MobdId_Mute_AlchemyHall]` in decompilation = `layers[0]` since that enum = 0.) The raw byte blob is then navigated via **byte offsets** to reach specific `MobdAnimation` structs within it.

### MobdAnimation — A Single Animation Sequence

```c
struct MobdAnimation {
  int anim_speed;               // playback speed (subtracted from anim_timer each tick;
                                // when timer goes negative, frame advances).
                                // 0 = don't override entity's current anim_speed.
                                // entity_anim_load: if (anim->anim_speed != 0) entity->anim_speed = anim->anim_speed;
  MobdAnimFrame *frames[1];     // variable-length frame pointer array.
                                // Sentinels (literal pointer values stored in-place):
                                //   (MobdAnimFrame*)NULL  = animation end (one-shot complete)
                                //   (MobdAnimFrame*)-1   = loop back to frames[0]
};
```

Located at a **byte offset** within `LevelMobdSurface`. All the `mobd_lookup_offset_*` fields in UnitStats are byte offsets into `layers[0]`.

### MobdAnimFrame — A Single Frame

```c
struct MobdAnimFrame {
  int x;                      // sprite draw offset X (pixels)
  int y;                      // sprite draw offset Y (pixels)
  int flags;
  MobdSprtImage *sprt;        // sprite image data
  BoxdCollisionShape *shape;  // collision shape for this frame
  SoundId sound_id;           // sound to play when this frame is displayed
  MobdPoint points[1];        // variable-length array of anchor points
};
```

### MobdPoint — Named Anchor Points

```c
struct MobdPoint {
  int id;     // anchor type identifier (0–5, -1 = list terminator)
  int x;      // offset X relative to entity position (fixed-point, same scale as entity->x)
  int y;      // offset Y relative to entity position (fixed-point, same scale as entity->y)
  int z;      // offset Z (rarely used)
};
```

Stored as variable-length array in each `MobdAnimFrame`, terminated by `id == -1`. Each frame can have different anchor positions (e.g., turret mount point moves with body animation).

**Coordinate system:** Anchors are world-space offsets relative to `entity->x`/`entity->y`. Usage: `world_pos = entity->pos + anchor->pos`. Same fixed-point scale (upper bits = tile, lower bits = sub-tile). Evidence: `unit->turret->entity->x = unit->entity->x + mobd_anchors.turret->x` (kknd.c:72738).

**Population (kknd.c:72708–72720):** Every game tick, for every unit, all 6 anchor slots are re-read from `anim_current_frame->points[]`. The `MobdPoint.id` field (0–5) indexes directly into `UnitMobdAnchors`:

```c
// Runs every tick for all units:
memset32(&unit->mobd_anchors, default_anchor, 6);
id = anim_current_frame->points[0].id;
for ( j = *id; j != -1; id += 4 ) {  // id += 4 = sizeof(MobdPoint)/4
    if ( j < 6 )
        *((&unit->mobd_anchors.turret) + j) = id;  // direct slot index
    j = id[4];  // next point's id
}
```

### UnitMobdAnchors — Anchor Slot Mapping

```c
struct UnitMobdAnchors {
  MobdPoint *turret;                   // id=0: turret attachment point
  MobdPoint *rally;                    // id=1: rally/spawn point for produced units
  MobdPoint *render;                   // id=2: healthbar/status bar render offset
  MobdPoint *grid;                     // id=3: building tile-grid alignment origin
  MobdPoint *_unit_mobd_anchor_unused; // id=4: never accessed by name in code
  MobdPoint *_unit_mobd_anchor_unknown;// id=5: dock/service point (tankers, repair bay); optional
};
```

#### Anchor Details

**turret (id=0):** Positions turret entity relative to body. Updated every tick: `turret->entity->x = unit->entity->x + mobd_anchors.turret->x`. Also passed to `entity_create_ex` as initial position. Different frames can shift the turret (e.g., recoil).

**rally (id=1):** World position where produced units spawn. Used by buildings: `spawn_x = unit->entity->x + mobd_anchors.rally->x`. Also used for escort waypoints.

**render (id=2):** Healthbar/status bar drawing offset. Used by `render_state_handler_buildings` and `render_state_handler_vehicles`:
- Buildings: `cmd.x = (render->x + entity->x - camera.x - 8448) >> 8` — the 8448 = 33×256 is a half-building-tile centering constant
- Vehicles: `cmd.x = (render->x + entity->x - camera.x - 4096) >> 8` — 4096 = 16×256 = half-tile centering
- Infantry: does NOT use render anchor (hardcoded 2048 = 8×256 offset, infantry are small)

**grid (id=3):** Building footprint alignment anchor. Snaps building to tile grid:
```c
entity->x = ((grid->x + entity->x) & 0xFFFFE000) - grid->x + 4096;
entity->y = ((grid->y + entity->y) & 0xFFFFE000) - grid->y + 4096;
```
`0xFFFFE000` = mask to tile boundary (tile = 0x2000 = 8192 units). `4096` = half-tile center. The anchor tells where the grid origin is relative to sprite center — necessary because large buildings have their pivot point off-center.

Also used by `unit_build_40DD00` to calculate which map tiles the building footprint occupies for collision placement.

**_unit_mobd_anchor_unused (id=4):** Populated by the generic loop if any frame defines id=4, but never accessed by name anywhere in code. Truly dead/vestigial.

**_unit_mobd_anchor_unknown (id=5):** **Dock/service point.** Used by:
- Oil tankers docking at Power Stations: `order_target_x = powerstation->entity->x + powerstation->mobd_anchors._unit_mobd_anchor_unknown->x` (kknd.c:65488)
- Oil tankers docking at Drill Rigs: same pattern (kknd.c:65566)
- Vehicles entering Repair Bay: `order_target_x = repairBay->entity->x + repairBay->mobd_anchors._unit_mobd_anchor_unknown->x` (kknd.c:27625, with `UnitOrder_RepairBayDock`)
- Fallback if anchor is (0,0): hardcoded offsets used instead (-8192, 0x2000 for power stations; -7168, 11520 for drill rigs)

**Proposed rename:** `_unit_mobd_anchor_unknown` → `dock_point` or `service_point`

#### Speculation on id=4 and id=5

Given the existing anchors (turret mount, spawn/rally, healthbar, grid origin, dock point):
- **id=4** may have been planned as a **smoke/damage effect point** — where smoke particles spawn on damaged buildings. Or a **shadow anchor** for aircraft shadow rendering. Cut/unused during development.

### MobdSprtImage — Sprite Rendering

```c
struct MobdSprtImage {
  Blitter blitter;    // function pointer to pixel blitter
  int flags;          // &1 = flip horizontally
  MobdImageData *bitmap;
};
```

### MobdImageData — Raw Pixel Data

```c
struct __unaligned MobdImageData {
  int width;
  int height;
  unsigned __int8 format;    // 0=raw palette indices, 2=RLE compressed
  unsigned __int8 pixels[1]; // variable-length pixel data
};
```

### MobdOrientation — Direction Enum (256-value angle system)

```c
enum MobdOrientation : unsigned __int32 {
  MobdOrientation_N   = 0,     // 12 compass directions defined
  MobdOrientation_NNE = 16,    // but full 256-value range used internally
  MobdOrientation_NE  = 32,
  MobdOrientation_E   = 64,
  MobdOrientation_SE  = 96,
  MobdOrientation_SSE = 112,
  MobdOrientation_S   = 128,
  MobdOrientation_SSW = 144,
  MobdOrientation_SW  = 160,
  MobdOrientation_W   = 192,
  MobdOrientation_NW  = 224,
  MobdOrientation_NNW = 240,
};
```

`g_angle_to_orientation[256]` maps 256 angles → 32 animation direction indices.
Formula: `g_angle_to_orientation[i] = ((i + 4) >> 3) & 0x1F`

## Animation API

### entity_anim_load(entity, offset)

Loads animation from `g_current_lvl_mobd[entity->mobd_id].layers[0] + offset`.
The offset addresses a `MobdAnimation` struct directly. Sets `anim_cursor` to start of frames array, triggers first tick.

Used for: single-animation entities (explosions, cursors, non-directional anims).

### entity_anim_start(entity, offset, idx)

Loads animation from `*(MobdAnimation**)(&layers[0]->frames[0].x + 4*idx + offset)`.
The offset addresses a **lookup table** — an array of `MobdAnimation*` pointers. `idx` selects which direction/variant.

Used for: directional units. `offset` = animation group (idle/move/attack), `idx` = `g_angle_to_orientation[orientation]` for direction.

### entity_anim_switch_direction(entity, offset, idx)

Same lookup as `entity_anim_start`, but preserves current frame position within the animation. Used when unit changes direction mid-animation without restarting the cycle.

### entity_anim_advance(entity, offset)

Transfers current animation progress from old animation to a new one at `offset`. Used for seamless animation transitions.

### entity_anim_tick(entity)

Advances animation by one frame. Frame list terminated by:
- `NULL` → one-shot animation complete (sets task flag 0x20000000)
- `-1` → loop back to first frame (sets task flag 0x10000000)
- normal pointer → advance to next frame

On each frame: updates `anim_current_frame`, `shape`, plays sound if `sound_id != 0`, and sends `TaskMessage_MouseHover|0x1` if frame has anchor point[0].id and task has wait flag 0x40000.

### entity_anim_advance_rotation(src_orientation, dst_orientation, step)

Rotates `*src_orientation` toward `dst_orientation` by `step` units. Uses shortest-path rotation around 256-value circle. Returns remaining angular distance. Used for smooth unit turning.

### entity_anim_clear(entity)

Nulls out `anim`, `anim_cursor`, `anim_current_frame`, `shape`.

## The Offset System — How It Works

Each entity type (MobdId) has its sprite data as a flat byte blob in `LevelMobdSurface`. Within this blob, different animation groups live at different byte offsets.

For **directional units** (infantry, vehicles, buildings), the layout is:

```
layers[0] + idle_offset   → MobdAnimation*[32]  (one per direction)
layers[0] + move_offset   → MobdAnimation*[32]  (one per direction)
layers[0] + attack_offset → MobdAnimation*[32]  (one per direction)
layers[0] + offset_4      → MobdAnimation*[32]  (damaged state for buildings)
```

For **non-directional entities** (explosions, cursors, UI):

```
layers[0] + offset → MobdAnimation  (single animation, loaded via entity_anim_load)
```

## UnitStats — mobd_lookup_offset Fields

Each `UnitStats` entry has 4 animation group offsets:

| Field | Purpose | Notes |
|-------|---------|-------|
| `mobd_lookup_offset_idle` | Standing/idle animation | Always present for units |
| `mobd_lookup_offset_move` | Walking/driving animation | -1 for buildings that can't move |
| `mobd_lookup_offset_attack` | Attack animation | -1 for turreted units (turret has own) |
| `mobd_lookup_offset_4` | Damaged/alternate state | Used for buildings (damaged appearance), aircraft (landed), mobile bases (deployed) |

### Complete UnitStats Offset Table

| # | Name | MobdId | attack | move | idle | offset_4 |
|---|------|--------|--------|------|------|----------|
| 0 | Rifleman | Surv_Rifleman | 1408 | 1472 | 1344 | -1 |
| 1 | Berserker | Mute_Berserker | 1600 | 1664 | 1536 | -1 |
| 2 | Flamer | Surv_Flamer | 1408 | 1472 | 1344 | -1 |
| 3 | Pyromaniac | Mute_Pyromaniac | 1408 | 1472 | 1344 | -1 |
| 4 | SWAT | Surv_Swat | 1408 | 1472 | 1344 | -1 |
| 5 | Shotgunner | Mute_Shotgunner | 1472 | 1536 | 1408 | -1 |
| 6 | Sapper | Surv_Sapper | 2176 | 2240 | 2112 | -1 |
| 7 | Rioter | Mute_Rioter | 1536 | 1600 | 1472 | -1 |
| 8 | El Presidente | Surv_General | 1408 | 1472 | 1344 | -1 |
| 9 | King Zog | Mute_Chieftan | 1408 | 1472 | 1344 | -1 |
| 10 | Saboteur | Surv_Saboteur | 1408 | 1472 | 1344 | -1 |
| 11 | Vandal | Mute_Vandal | 1408 | 1472 | 1344 | -1 |
| 12 | Technician | Surv_Technician | -1 | 1088 | 960 | -1 |
| 13 | Mekanik | Mute_Mekanik | -1 | 1088 | 960 | -1 |
| 14 | RPG Launcher | Surv_RpgLauncher | 1472 | 1536 | 1408 | -1 |
| 15 | Bazooka | Mute_Bazooka | 1728 | 1792 | 1664 | -1 |
| 16 | Sniper | Surv_Sniper | 1408 | 1472 | 1344 | -1 |
| 17 | Crazy Harry | Mute_CrazyHarry | 1408 | 1472 | 1344 | -1 |
| 18 | General | Surv_General | 1408 | 1472 | 1344 | -1 |
| 19 | Warlord | Mute_Chieftan | 1408 | 1472 | 1344 | -1 |
| 20 | Scout | Surv_Rifleman | 1408 | 1472 | 1344 | -1 |
| 21 | Derrick (Surv) | Surv_MobileDerrick | -1 | 860 | 732 | -1 |
| 22 | Derrick (Mute) | Mute_MobileDerrick | -1 | 832 | 704 | -1 |
| 23 | Oil Tanker (Surv) | Surv_OilTanker | -1 | 832 | 704 | -1 |
| 24 | Oil Tanker (Mute) | Mute_OilTanker | -1 | 832 | 704 | -1 |
| 25 | Convoy Tanker | Surv_OilTanker | -1 | 832 | 704 | -1 |
| 26 | Dirt Bike | Surv_DirkBike | 960 | 1024 | 896 | -1 |
| 27 | Dire Wolf | Mute_DirewWolf | 1712 | 1776 | 1648 | -1 |
| 28 | 4x4 Pickup | Surv_4x4Pickup | -1 | 1024 | 896 | -1 |
| 29 | Bike & Sidecar | Mute_BikeAndSidecar | -1 | 1024 | 896 | -1 |
| 30 | ATV | Surv_Atv | -1 | 1024 | 896 | -1 |
| 31 | Monster Truck | Mute_MonsterTruck | -1 | 1024 | 896 | -1 |
| 32 | ATV Flamethrower | Surv_AtvFlamethrower | -1 | 1024 | 896 | -1 |
| 33 | Giant Scorpion | Mute_GiantScorpion | 1536 | 1600 | 1472 | -1 |
| 34 | Anaconda Tank | Surv_AnacondaTank | -1 | 1216 | 1088 | -1 |
| 35 | War Mastodon | Mute_WarMastadont | -1 | 1728 | 1600 | -1 |
| 36 | Barrage Craft | Surv_BarrageCraft | -1 | 832 | 832 | -1 |
| 37 | Giant Beetle | Mute_GiantBeetle | 2488 | 2552 | 2424 | -1 |
| 38 | Autocannon Tank | Surv_AutocannonTank | -1 | 1408 | 1280 | -1 |
| 39 | Missile Crab | Mute_MissileCrab | -1 | 1728 | 1600 | -1 |
| 40 | Mobile Outpost | Surv_MobileOutpost | -1 | 960 | 832 | 768 |
| 41 | Clanhall Wagon | Mute_ClanhallWagon | -1 | 832 | 704 | 640 |
| 42 | Orville ultralight | Surv_AnacondaTank | 0 | 0 | 0 | 0 |
| 43 | Wasp | Mute_Wasp | 832 | 896 | 768 | 704 |
| 44 | Bomber | Surv_Bomber | 764 | 828 | 700 | 636 |
| 45 | Kamikaze Rocket | Surv_AnacondaTank | 0 | 0 | 0 | 0 |
| 46 | Drill Rig (Surv) | Surv_Drillrig | 428 | -1 | 364 | 300 |
| 47 | Drill Rig (Mute) | Mute_Drillrig | 392 | -1 | 328 | 264 |
| 48 | Power Station (Surv) | Surv_PowerStation | 452 | 516 | 388 | 324 |
| 49 | Power Station (Mute) | Mute_PowerStation | 452 | 516 | 388 | 324 |
| 50 | Detention Center | Surv_DetentionCenter | 344 | 408 | 280 | 216 |
| 51 | Holding Pen | Mute_HoldingPen | 344 | 408 | 280 | 216 |
| 52 | Guard Tower | Surv_GuardTower | 608 | 672 | 544 | 544 |
| 53 | Machinegun Nest | Mute_MachinegunNest | 572 | 636 | 508 | 508 |
| 54 | Cannon Tower | Surv_CannonTower | 892 | 956 | 828 | 764 |
| 55 | GrapeShot Cannon | Mute_GrapeshotTower | 764 | 828 | 700 | 636 |
| 56 | Missile Battery | Surv_MissileBattery | 628 | 692 | 564 | 500 |
| 57 | Rotary Cannon | Mute_RotaryCannon | 956 | 1020 | 892 | 892 |
| 58 | Outpost | Surv_Outpost | 1232 | 1296 | 1168 | 1104 |
| 59 | Clanhall | Mute_Clanhall | 968 | 1032 | 904 | 840 |
| 60 | Machine Shop | Surv_MachineShop | 452 | 516 | 388 | 324 |
| 61 | Blacksmith | Mute_Blacksmith | 416 | 480 | 352 | 288 |
| 62 | Beast Enclosure | Mute_BeastEnclosure | 452 | 516 | 388 | 324 |
| 63 | Repair Bay | Surv_RepairBay | 512 | 576 | 448 | 384 |
| 64 | Menagerie | Mute_Menagerie | 428 | 492 | 364 | 300 |
| 65 | Research Lab | Surv_ResearchLab | 416 | 480 | 352 | 288 |
| 66 | Alchemy Hall | Mute_AlchemyHall | 416 | 480 | 352 | 288 |
| 67 | Tech Bunker | TechBunker | 412 | 476 | 348 | -1 |
| 68 | Hut | Hut | -1 | 528 | 400 | 336 |
| 69–73 | Wall/UFO | Surv_AnacondaTank | 0 | 0 | 0 | 0 |
| 74 | Gort | Gort | 2684 | 2748 | 2620 | -1 |
| 75 | G.O.R.N. | Surv_AnacondaTank | -1 | 1216 | 1088 | -1 |
| 76 | Plasma Tank | PlasmaTank | -1 | 1176 | 1048 | -1 |
| 77 | Sentinel Droid | SentinelDroid | -1 | 1920 | 1792 | -1 |
| 78 | Mech | Mech | -1 | 1560 | 1432 | -1 |
| 79–81 | Satellite/FOBS/Reactor | Surv_AnacondaTank | 0 | 0 | 0 | 0 |
| 82 | Hut (capturable) | Hut | 464 | 528 | 400 | -1 |
| 83 | Tech Bunker (capturable) | TechBunker | 412 | 476 | 348 | -1 |
| 84 | Infiltrator | Mute_Chieftan | 1408 | 1472 | 1344 | -1 |
| 85–87 | Timebomb/Tree/Airstrike | various | 0 | 0 | 0 | 0 |
| 88 | Sentinel (invalid) | Invalid | 0 | 0 | 0 | 0 |

### Offset Patterns Observed

**Standard Infantry** (most common):
- idle=1344, move=1472, attack=1408 (gap of 64 between each = 16 pointers × 4 bytes)
- Some infantry share the same MobdId sprite set, reusing offsets

**Vehicles (no turret)**: idle/move only, attack=-1 (turret handles attacking)

**Buildings**: all 4 offsets present, offset_4 = damaged state. Gap between groups varies by building complexity.

**The 64-byte group size** = 16 direction pointers × 4 bytes = one directional animation group (covering 16 of the 32 possible orientations, with mirroring via sprite flip flag).

## Hardcoded Magic Offsets (Non-Stats)

These are raw numeric offsets used in `entity_anim_load`/`entity_anim_start` that don't come from UnitStats fields:

### MobdId_Explosions (22) Offsets

| Offset | Context | Meaning |
|--------|---------|---------|
| 0 | Generic explosion (unit death, building death) | Default explosion |
| 220 | `task_explosion_2` | Building destruction explosion |
| 568 | Infantry death (Survivors race) | Survivor infantry death |
| 636 | Infantry death (Evolved race) | Evolved infantry death |
| 2184 | Machinegun projectile (turreted) | Turreted muzzle flash |
| 2248 | Machinegun projectile (infantry) | Infantry muzzle flash |

### MobdId_Cursors (17) Offsets

| Offset | Context | Cursor Type |
|--------|---------|-------------|
| 12 | Default arrow | Normal pointer |
| 24 | Help mode (no target) | Question mark |
| 36 | Help mode (unit hovered) | Question mark + highlight |
| 48 | Unit hovered | Selection highlight |
| 144 | Move order | Move arrow |
| 188 | Repair cursor | Wrench |
| 216 | Guard/escort cursor | Shield |
| 244 | Follow cursor | Follow arrow |
| 280 | Attack friendly building | Attack icon |
| 292 | Sabotage/infiltrate | Sabotage icon |
| 304 | Sell building / airstrike target | Sell/target icon |
| 448 | Invalid / cannot-do | Red X |
| 572 | Attack enemy | Crosshair |
| 820 | X-mark on map | Map X marker |
| 852 | X-mark variant 1 (most levels) | Map X marker variant |
| 868 | X-mark variant 2 (Mute_04 only) | Map X marker variant |

### MobdId_Sidebar (10) Offsets

| Offset | Context | Element |
|--------|---------|---------|
| 1980 | Production queue panel | Queue display |
| 2160 | Queue size ≥10 label | "10+" counter display |
| 2276 | Queue count digit / airstrike counter | Single digit (frames 0–8) |
| 2312 | Production progress bar | Progress fill (frames 0–15) |

### MobdId_MissionOutcomeModal (15) Offsets

| Offset | Context | Element |
|--------|---------|---------|
| 0 | Main modal background | Layer 1 |
| 12 | Secondary overlay | Layer 2 |
| 24 | Tertiary overlay (loaded twice) | Layer 3 |

### MobdId_IngameMenuUi (30) / MobdId_45 Menu Button Offsets

| Offset | Context | Element |
|--------|---------|---------|
| 96 | Button default state | Base |
| 708 | Button pressed state | Pressed |
| 720 | Button state cycle | Idle/hover/press |
| 1692 | Button closed state | Menu button closed |
| 1704 | Button opening transition | Menu button opening |
| 1716 | Button opened state | Menu button open |
| 1824 | Button child animation | Lobby button child |

### Font MOBDs

| MobdId | Value | Usage |
|--------|-------|-------|
| MobdId_Font_26 (26) | Used via `g_current_lvl_mobd[26].layers[0]` | In-game UI font |
| MobdId_Font_27 (27) | Used via `g_current_lvl_mobd[27].layers[0]` | Tooltip/sidebar font |
| MobdId_FontItalic (80) | Used via `g_current_lvl_mobd[80].layers[0]` | Menu italic font |

## Entity ↔ MOBD Relationship

1. `entity_create(mobd_id, ...)` allocates Entity, sets `entity->mobd_id`
2. All animation lookups use `g_current_lvl_mobd[entity->mobd_id]` to find the sprite data
3. Different animation states (idle/move/attack/damaged) are at different byte offsets within the same MOBD
4. For directional animations: offset addresses a pointer table indexed by direction (0–31)
5. For non-directional: offset addresses a MobdAnimation struct directly

## Save/Load

`EntitySaveStruct` stores:
- `mobd_id` — which MOBD to load
- `mobd_offset` — byte offset of current animation within the MOBD
- `_54_inside_mobd_ptr4` — offset of current frame within layers[0]->frames
- `anim_speed` — current playback speed

On load, `GAME_load_entity()` reconstructs the animation state by recalculating pointers from these offsets.

## LevelMobd Loading

```
LVL_InitMobd() {
  g_current_lvl_mobd = LVL_FindSection("MOBD");  // parse MOBD section from .LVL file
  // allocate entity pool (2000 entities = 0x48440 bytes)
  // init default entity
}
```

All MOBD pointers are **relocatable** — stored as offsets in the level file and converted to absolute pointers during level loading.
