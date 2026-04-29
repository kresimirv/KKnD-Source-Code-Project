# MOBD System — TODO / Proposals

## Critical Renames

### Struct Renames

| Current | Proposed | Reason |
|---------|----------|--------|
| `LevelMobdSurface` | `MobdAnimBank` | Not a "surface" — it's a flat byte blob containing all animation data for one MobdId. "AnimBank" better describes its role as an animation data bank |
| `LevelMobd` | `MobdDirectory` or `MobdIndex` | It's a directory/index of all entity animation banks, indexed by MobdId |
| `mobd_lookup_offset_4` | `mobd_lookup_offset_damaged` | Used exclusively for building damaged states and aircraft landed states (see analysis below) |
| `_unit_mobd_anchor_unused` | Remove or mark `__deprecated` | Never referenced in code |
| `_unit_mobd_anchor_unknown` | `dock_point` or `service_point` | Used by tankers docking at Power Stations/Drill Rigs and vehicles entering Repair Bay. Provides world-space offset where visiting units navigate to |
| `UnitMobdAnchors.render` | `healthbar` | Only used to position healthbar/status bar sprite in render_state_handler_buildings and render_state_handler_vehicles |
| `_54_inside_mobd_ptr4` (in EntitySaveStruct) | `mobd_frame_offset` | It's the byte offset of current frame within `layers[0]->frames` |

### Field Renames in Entity

| Current | Proposed | Reason |
|---------|----------|--------|
| `anim_cursor` | `anim_frame_ptr` | It's a pointer into the frames array that advances each tick |
| `anim_timer` | `anim_frame_delay` | It's the delay counter before advancing to next frame |
| `anim_speed` | `anim_tick_rate` | More precise — controls how fast frames advance |

### Variable Renames

| Current | Proposed | Reason |
|---------|----------|--------|
| `g_current_lvl_mobd` | `g_mobd` or `g_mobd_directory` | Shorter, clearer |
| `g_current_lvl_mobd_verified` | `g_mobd_loaded` | Clearer boolean name |
| `dword_47965C` | Investigate — `MobdImageData*` global | Needs context on what image it holds |
| `g_47C6E0_mobd79_task` | `g_menu_ui_task` | MobdId 79 = menu UI elements |
| `g_turrent_grapeshot_cannon` | `g_turret_grapeshot_cannon` | Typo: "turrent" → "turret" |

### Function Renames

| Current | Proposed | Reason |
|---------|----------|--------|
| `entity_anim_load` | `entity_anim_set` | Loads and starts playing a single (non-directional) animation |
| `entity_anim_start` | `entity_anim_set_directional` | Loads a directional animation by group offset + direction index |
| `entity_anim_switch_direction` | `entity_anim_change_direction` | Preserves frame position while switching direction |
| `entity_anim_advance` | `entity_anim_transfer` | Transfers animation progress from old anim to new offset |
| `cursor_load_mobd` | `cursor_set_anim` | More consistent with entity naming |

## Enum Proposals

### CursorAnimOffset — for MobdId_Cursors offsets

```c
enum CursorAnimOffset {
  CursorAnim_Arrow         = 12,
  CursorAnim_HelpDefault   = 24,
  CursorAnim_HelpUnit      = 36,
  CursorAnim_Hover         = 48,
  CursorAnim_Move          = 144,
  CursorAnim_Repair        = 188,
  CursorAnim_Guard         = 216,
  CursorAnim_Follow        = 244,
  CursorAnim_AttackOwn     = 280,
  CursorAnim_Sabotage      = 292,
  CursorAnim_Sell          = 304,  // also airstrike targeting
  CursorAnim_Invalid       = 448,
  CursorAnim_AttackEnemy   = 572,
  CursorAnim_XMark         = 820,
  CursorAnim_XMarkAlt1     = 852,
  CursorAnim_XMarkAlt2     = 868,  // Mute_04 level only
};
```

### ExplosionAnimOffset — for MobdId_Explosions offsets

```c
enum ExplosionAnimOffset {
  ExplosionAnim_Default        = 0,
  ExplosionAnim_Building       = 220,
  ExplosionAnim_InfantrySurv   = 568,
  ExplosionAnim_InfantryMute   = 636,
  ExplosionAnim_MuzzleTurret   = 2184,
  ExplosionAnim_MuzzleInfantry = 2248,
};
```

### SidebarAnimOffset — for MobdId_Sidebar offsets

```c
enum SidebarAnimOffset {
  SidebarAnim_QueuePanel    = 1980,
  SidebarAnim_QueueOverflow = 2160,  // "10+" display
  SidebarAnim_QueueDigit    = 2276,  // frames 0-8 = digits
  SidebarAnim_ProgressBar   = 2312,  // frames 0-15 = fill level
};
```

### MenuAnimOffset — for MobdId_IngameMenuUi / MobdId_45

```c
enum MenuAnimOffset {
  MenuAnim_ButtonBase     = 96,
  MenuAnim_ButtonPressed  = 708,
  MenuAnim_ButtonCycle    = 720,
  MenuAnim_ButtonClosed   = 1692,
  MenuAnim_ButtonOpening  = 1704,
  MenuAnim_ButtonOpened   = 1716,
  MenuAnim_ButtonChild    = 1824,
};
```

### MissionOutcomeAnimOffset

```c
enum MissionOutcomeAnimOffset {
  OutcomeAnim_Background = 0,
  OutcomeAnim_Overlay1   = 12,
  OutcomeAnim_Overlay2   = 24,
};
```

## Structural Concerns

### 1. `mobd_lookup_offset_4` is definitely "damaged building state"

Evidence:
- Only used by buildings (offset_4 != -1 for all buildings, == -1 for all non-building units)
- Also used by aircraft (Wasp=704, Bomber=636) — likely "landed" state
- Mobile Outpost (768) and Clanhall Wagon (640) use it — likely "deployed" state
- In `unit_mode_idle` code: if `health < max_health/2 && mobd_lookup_offset_4 != -1`, play offset_4 animation
- **Rename to `mobd_lookup_offset_damaged`** (or `mobd_lookup_offset_alt_state` to cover aircraft/mobile bases)

### 2. Placeholder/dummy units using MobdId_Surv_AnacondaTank

Units 42, 45, 69–73, 75, 79–81, 85, 87 all use `MobdId_Surv_AnacondaTank` with offsets 0/0/0/0. These are:
- Orville ultralight, Kamikaze Rocket — unimplemented/cut content
- Wall ×4, UFO, Satellite/FOBS/Reactor, Timebomb, Airstrike — abstract/placeholder units

They reuse Anaconda's MobdId as a dummy. The `0,0,0,0` offsets mean "use whatever is at the start of the data" — effectively broken/invisible or shows the first animation.

**Proposal**: Add `MobdId_Placeholder` or `MobdId_None` instead of abusing Anaconda's MobdId.

### 3. Barrage Craft has move==idle (both 832)

`mobd_lookup_offset_move=832, mobd_lookup_offset_idle=832` — same animation for both. Likely intentional (barrage craft hovers, looks same moving or still).

### 4. Directional animation group sizes vary by unit class

**Fact:** Infantry use 16-direction groups (64 bytes), vehicles use 32-direction groups (128 bytes).

Evidence:
- Rifleman: idle=1344, attack=1408, move=1472 → gaps of 64 bytes = 16 entries
- Anaconda: idle=1088, move=1216 → gap of 128 bytes = 32 entries
- `entity_anim_start` does `4*idx + offset` where idx comes from `g_angle_to_orientation` (range 0–31)
- `g_angle_to_orientation` formula: `((i+4) >> 3) & 0x1F` for i=0..255 → produces 0–31

**Unresolved:** Infantry groups are only 16 entries wide, but `g_angle_to_orientation` can produce indices 0–31. With `4*31 = 124` this overflows a 64-byte group into the next one. Infantry must use a different orientation mapping or only even indices. Needs binary verification.

### 5. UnitAttachment/UnitProjectileType definitions missing

All `g_turret_*`, `g_attach_*`, `g_proj_*` globals are **external symbols** not defined in this translation unit. Their `mobd_lookup_offset_idle`, `mobd_lookup_offset_attack`, `mobd_lookup_offset_travel`, `mobd_lookup_offset_impact` values are unknown.

**TODO**: Find the .obj/.lib or other .c file containing these definitions to complete the offset mapping.

### 6. `entity_anim_start` offset semantics

```c
void entity_anim_start(Entity *entity, ptrdiff_t offset, ptrdiff_t idx);
```

`offset` = byte offset into `layers[0]` treated as flat buffer. `idx` = direction index (0–31). Actual lookup: `*(MobdAnimation**)(&layers[0]->frames[0].x + 4*idx + offset)`. The combination selects one `MobdAnimation*` from a direction lookup table at that offset.

### 7. MobdId enum gaps and unknowns

| MobdId | Value | Status |
|--------|-------|--------|
| MobdId_20 | 20 | Unknown — used by `task_4269B0_mobd_20_handler`, `script_431E60_mobd_20_input` |
| MobdId_23 | 23 | Unknown |
| MobdId_45 | 45 | Menu UI buttons/elements |
| MobdId_46 | 46 | Unknown — `script_442BB0_mobd46` |
| MobdId_69 | 69 | Unknown |
| MobdId_79 | 79 | Menu/lobby UI elements |
| MobdId_some_letters_and_symbols (6) | 6 | Special character set — used by `TaskType_MOBD_6_some_letters_and_symbols` |

**Proposals**:
- `MobdId_20` → `MobdId_TextInput` (used for text input UI)
- `MobdId_45` → `MobdId_MenuButtons`
- `MobdId_46` → needs investigation (`script_442BB0_mobd46`)
- `MobdId_79` → `MobdId_LobbyUi`
- `MobdId_some_letters_and_symbols` → `MobdId_SpecialChars`

### 8. `anim_speed = 0x20000000` magic number

Fixed-point speed value (2^29). Used for one-shot explosion animations = "fast playback".

**Proposal**: Define `ANIM_SPEED_FAST = 0x20000000`.

### 9. g_angle_to_orientation potential memory corruption

`dword_47CFC4` is 4 bytes before `g_angle_to_orientation`, so `dword_47CFC4[1..256]` maps to `g_angle_to_orientation[0..255]`. The `_47D3C4_bugged[++v1]` write may corrupt memory past the orientation array.

**TODO**: Verify in binary.

### 10. `MobdAnimFrame.flags` — dead/unused field

**Fact:** `MobdAnimFrame.flags` (offset 8 in struct, third `int` field) is **never read** anywhere in game code.

Evidence:
- `grep "anim_current_frame->flags"` = 0 matches across entire codebase
- `render_state_handler_entity` reads only `->x`, `->y`, `->sprt` from frame
- `render_state_handler_unit_turret` reads only `->x`, `->y`, `->sprt` from frame
- `entity_anim_tick` reads `->sound_id`, `->shape`, `->points[0].id` — never `->flags`
- The blitter (`render_mode_sprt_draw`) checks `RenderCommand.flags` and `MobdSprtImage.flags & 1` (flip), NOT `MobdAnimFrame.flags`
- One hit at line 66569 (`a2->frames[0].flags`) is UI layout code abusing MobdAnimFrame struct as generic data (treating fields as pixel dimensions, not actual flags)

**Conclusion:** Field exists in binary MOBD file format (loaded from disk, pointers fixup'd by `HUNK_FixPointers`) but runtime code ignores it completely. Likely vestigial from development — may have once controlled per-frame flip/mirror/event triggers that got moved to `MobdSprtImage.flags` or removed.

**Possible original purposes (speculative):**
- Per-frame horizontal flip (moved to `MobdSprtImage.flags & 1`)
- Per-frame event trigger mask (removed, events now in `sound_id` + `points[]`)
- Padding/alignment field inserted by compiler/toolchain

**Proposal:** Rename to `_flags_unused` or add comment `///< FILE FORMAT FIELD — never read by game code`. Do NOT create enum — no values to enumerate.

**Related:** `MobdSprtImage.flags & 1` = flip horizontally (used by blitter). `RenderNode.flags & 0x40000000` = hidden. `RenderNode.flags & 0x80000000` = destroy. `RenderCommand.flags & 0xC0000000` = skip draw. `RenderCommand.flags & 0x10000000` = palette blend mode.

## Core Facts (for skill)

1. `g_current_lvl_mobd[mobd_id].layers[0]` = flat byte blob per entity type
2. All `mobd_lookup_offset_*` are byte offsets into this blob
3. `entity_anim_load` = non-directional, `entity_anim_start` = directional (offset + 4*direction_index)
4. Infantry = 16-direction groups (64 bytes), vehicles = 32-direction groups (128 bytes)
5. `MobdOrientation` is 0–255, mapped to 0–31 via `g_angle_to_orientation`
6. Frame sentinel: NULL=end, -1=loop
7. `UnitStats.mobd_lookup_offset_4` = damaged/alt state (buildings, aircraft, mobile bases)
8. Hardcoded offsets for Explosions, Cursors, Sidebar, Menu UI documented in enum proposals above

## Priority Actions

1. **HIGH**: Rename `mobd_lookup_offset_4` → `mobd_lookup_offset_damaged` everywhere
2. **HIGH**: Create `CursorAnimOffset` and `ExplosionAnimOffset` enums to replace magic numbers
3. **HIGH**: Create `SidebarAnimOffset` enum
4. **HIGH**: Fix typo `g_turrent_grapeshot_cannon` → `g_turret_grapeshot_cannon`
5. **MEDIUM**: Rename `LevelMobdSurface` → `MobdAnimBank`
6. **MEDIUM**: Rename `LevelMobd` → `MobdDirectory`
7. **MEDIUM**: Investigate MobdId_20, MobdId_46, MobdId_69, MobdId_79 to get proper names
8. **MEDIUM**: Find attachment/projectile global definitions for complete offset mapping
9. **LOW**: Investigate 16-vs-32 direction discrepancy for infantry vs vehicles
10. **LOW**: Verify g_angle_to_orientation initialization bug
11. **LOW**: Define `ANIM_SPEED_FAST` constant for `0x20000000`
15. **HIGH**: Rename `_unit_mobd_anchor_unknown` → `dock_point` in `UnitMobdAnchors` — confirmed used exclusively as tanker/vehicle docking position at Power Stations, Drill Rigs, and Repair Bays
12. **HIGH**: Define sentinel constants for MobdAnimation frame array: `#define MOBD_FRAME_END ((MobdAnimFrame*)0)` and `#define MOBD_FRAME_LOOP ((MobdAnimFrame*)-1)` — currently raw NULL/`-1` pointer casts scattered through entity_anim_tick
13. **MEDIUM**: Simplify `LevelMobd` struct — only `layers[0]` is ever accessed (`layers[MobdId_Mute_AlchemyHall]` = `layers[0]` since that enum = 0). Replace `LevelMobd { LevelMobdSurface *layers[1]; }` with plain `LevelMobdSurface*` typedef, making `g_current_lvl_mobd` a `LevelMobdSurface**` (array of pointers, indexed by MobdId)
14. **LOW**: Rename `MobdAnimFrame.flags` → `MobdAnimFrame._flags_unused` — field present in binary format but never read by game code (see Structural Concern #10)
16. **LOW**: Create `MobdImageFormat` enum for `MobdImageData.format` field:

```c
enum MobdImageFormat : unsigned __int8 {
  MobdImageFormat_Raw = 0,  // uncompressed palette indices (width × height bytes)
  MobdImageFormat_RLE = 2,  // RLE compressed; blitter requires width/height match cmd dimensions
};
```

Evidence: `render_mode_sprt_draw` (kknd.c:21496–21512) branches on format: `if (!format)` → raw path, `if (format != 2)` → skip (unsupported). Value 2 additionally requires image dimensions match exactly (`v7 != *v5 || v8 != v5[1]` → skip). Only values 0 and 2 observed.
