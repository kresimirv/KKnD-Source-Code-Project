# LVL — Level System

## Overview

KKND levels are binary dumps stored in `.lvl` files using an IFF-like (Interchange File Format) chunked structure. A level file contains two sequential chunks:

1. **DATA hunk** — The actual level data (pointers, tile data, entity placement, sprites, etc.) stored as raw memory image
2. **RLLC hunk** — Relocation/pointer fixup table used to patch embedded pointers in the DATA hunk

On load, pointers within DATA are file-relative offsets. `HUNK_FixPointers` walks the RLLC table and patches each offset into a valid memory address relative to the allocated DATA buffer.

---

## File Format (IFF Chunks)

Each chunk in the file:
```
Offset  Size  Description
0x00    4     Tag (4 ASCII chars, big-endian) — e.g. "DATA", "RLLC"
0x04    4     Size (big-endian u32) — byte count of payload following
0x08    ...   Payload data (Size bytes)
```

`LVL_ReadHunk` reads the 8-byte header, byte-swaps the size from big-endian to little-endian, mallocs the payload, and reads it in.

---

## Level Types

### wait.lvl
- Loading/splash screen level
- Loaded once at startup: `LEVELS\{resolution}\wait.lvl`
- Contains MAPD (background image) only
- Displayed during initial load with palette animation

### supspr.lvl (Super Sprites)
- Shared sprite asset library: `LEVELS\{resolution}\supspr.lvl`
- Contains MOBD hunk with all shared animation frames (units, UI elements)
- Loaded once per game session
- Its MOBD section is **substituted** into every gameplay/menu level via `LVL_SubstHunk`

### sprites.lvl
- Alternate name for supspr.lvl used by FMV briefing loader and multiplayer load path
- Same purpose: `LEVELS\{resolution}\sprites.lvl`
- **May be same file with different name in different contexts**

### super.lvl (Super Level / Menu Level)
- Menu system level: `LEVELS\{resolution}\super.lvl`
- Contains MAPD (multiple menu backgrounds indexed by MenuId), MOBD (substituted from supspr.lvl), BOXD, CPLC
- CPLC layers contain UI entity placements for each menu screen
- MAPD layers indexed by MenuId: 0=MainMenu, 1=Multiplayer, 2=NewCampaign, etc.

### Mission Levels (surv_XX.lvl, mute_XX.lvl)
- Actual gameplay levels
- Filename stored in `g_lvl_desc[level_id].lvl_filename`
- Path: `LEVELS\{resolution}\{filename}`
- Contains all hunks: MAPD (terrain + fog tiles), MOBD (substituted), BOXD (collision grid), CPLC (entity placements)

### Multiplayer Levels (mlti_XX.lvl)
- LevelId 48–67 (LevelId_Surv_16 through LevelId_Mute_25)
- Same structure as mission levels
- No VBC briefing video (vbc_filename is NULL)

### fmv.lvl
- Mission briefing level: `LEVELS\{resolution}\fmv.lvl`
- Loaded for pre-mission FMV playback
- Contains MAPD (briefing background) and uses substituted MOBD from supspr.lvl

---

## Resolution Directories

Levels live under `LEVELS\320\` or `LEVELS\640\` determined by `sub_40E3E0()`:
- If screen width (`dword_478AA0`) >= 640 → "640"
- Otherwise → "320"

---

## Core Data Structures

### LevelHunk
```c
struct LevelHunk {
    LevelHunkSection *sections[1];  // flexible array of section pointers
};
```
Top-level container after loading. `sections[0]` points to the first `LevelHunkSection` in the DATA payload. The array is NULL-terminated.

### LevelHunkSection
```c
struct LevelHunkSection {
    char name[4];   // 4-char tag: "MAPD", "MOBD", "BOXD", "CPLC"
    void *ptr;      // pointer to section data (fixed up on load)
};
```
Array of name+pointer pairs. Iterated by `LVL_FindSection(name)` which does `strncmp(name, section.name, 4)`.

### LevelDesc
```c
struct LevelDesc {
    const char *lvl_filename;           // "surv_01.lvl"
    const char *wav_filename;           // "surv2.wav" — background music
    const char *vbc_filename;           // "heads01.vbc" — briefing video (NULL for multiplayer)
    __int16 player_starting_cash;       // player money
    __int16 ai_starting_cash;           // AI money
    __int16 superlvl_ai_cash;           // bonus AI cash on super-level difficulty
    __int16 superlvl_ai_cost_reduction; // multiplier for AI production cost reduction (0-256, /256)
    __int16 max_tech_level;             // tech level cap (1-5)
    __int16 _level_desc_field_16;       // unknown / padding
    unsigned int disabled_units_mask;   // bitmask: which units are disabled for this mission
    int victory_condition_bits;         // bitmask: win conditions to check
    AiAirstrikeConfig superlvl_ai_airstrike_count_enemy;  // AI airstrike config vs player
    AiAirstrikeConfig superlvl_ai_airstrike_count_ally;   // AI airstrike config when allied
};
```
68-element table: `g_lvl_desc[68]`. Indexed by `LevelId`. Entries 0–14 = Survivors campaign, 15–29 = Mutants campaign, 48–67 = multiplayer maps.

---

## Level Loading Workflow

### 1. System Initialization (`LVL_SysInit`)
```
stru2_init()       — allocate render handler cache pool
FILE_init()        — filesystem init
REND_create_window — DirectDraw window
SOUND_init()       — DirectSound
```

### 2. Wait Screen Load
```
LVL_LoadLevel("wait.lvl")
LVL_RunLevel(g_wait_lvl)
  → MAPD_Draw, show splash, animate 180 frames
LVL_cleanup()
```

### 3. Super Level Preparation (`GAME_PrepareSuperLvl`)
```
LVL_LoadLevel("supspr.lvl") → g_supspr_lvl
LVL_LoadLevel("super.lvl")  → g_level
LVL_SubstHunk(g_level, g_supspr_lvl, "MOBD")  — inject shared sprites
LVL_RunLevel(g_level)        — initialize all subsystems
MAPD_Draw(mapd_id)           — render menu background
CPLC_select(mapd_id)         — activate menu UI entities
```

### 4. Mission Level Load (`sub_422F60`)
```
if (level_id != prev_level_id):
    LVL_LoadLevel("sprites.lvl") → g_supspr_lvl  (if not already loaded)
    LVL_LoadLevel(g_lvl_desc[id].lvl_filename) → g_level
    LVL_SubstHunk(g_level, g_supspr_lvl, "MOBD")
LVL_RunLevel(g_level)
  → Init subsystems: Tasks, Timer, Render, Sound, MAPD, MOBD, BOXD, CPLC, Fade, Window
palette_40E400(mapd.palette)
sidebar_init()
LVL_InitBoxdTerrain()        — build terrain grid from BOXD
GAME_InitUnits()             — create unit objects from CPLC
```

### 5. Level Teardown (`LVL_Deinit` / `LVL_cleanup`)
Reverse order of init:
```
render_mode_scrl_setup, FADE, sub_445B30, CPLC cleanup, BOXD cleanup,
MOBD_Deinit, MAPD_Deinit, sound cleanup, stru2 cleanup, timer, tasks
g_current_lvl = NULL
g_current_lvl_sections = NULL
```

---

## Hunk Substitution (`LVL_SubstHunk`)

Purpose: Replace a section pointer in one level (dst) with the corresponding data from another level (src).

Used exclusively to inject shared MOBD sprites from `supspr.lvl`/`sprites.lvl` into mission/menu levels.

Algorithm:
1. Walk src's sections array, find section matching `name` (4-char tag)
2. Walk dst's sections array, find section matching `name`
3. Overwrite dst's section pointer with src's data pointer

This means the mission level's own MOBD data (if any) is replaced by the shared sprite bank. The mission level doesn't need to duplicate sprite data.

---

## Pointer Fixup (`HUNK_FixPointers`)

The RLLC hunk contains a relocation table. Structure:
```
Header: section_count (DWORD) — total number of fixup entries
Entries: array of DWORDs, each encoding one fixup operation
```

Three fixup modes based on bit flags in each entry:

### Mode 1: Simple Pointer Relocation (bit 31 = 0, bit 30 = 0)
```
entry & 0x3FFFFFFF = offset into DATA
```
Action: `*(DWORD*)(data + offset) += (DWORD)data`

The value at that offset is a file-relative pointer. Add the base address to make it absolute.

### Mode 2: Batch Pointer Relocation (bit 31 = 0, bit 30 = 1)
```
entry & 0xBFFFFFFF = offset into DATA
next_entry = count of consecutive pointers
```
Action: For `count` consecutive DWORDs starting at offset, each gets `+= (DWORD)data`

Used for arrays of pointers (e.g., the `sections[]` array, `tiles[]`, `layers[]`).

### Mode 3: Render Handler Fixup (bit 31 = 1)
```
entry & 0x7FFFFFFF = offset into DATA
```
Action: Read the DWORD at that offset as a "hunk ID", look it up in `g_render_handlers` table, and replace with the function pointer for the renderer (`mode_render` / blitter function).

This patches embedded renderer IDs in sprite/image data with actual function pointers. The `g_render_handlers` table maps hunk IDs to init/draw/cleanup function triples.

The lookup uses a cache (`stru2` pool) to avoid repeated table scans for the same hunk ID.

---

## Section Types (Hunks)

### MAPD — Map/Background Data
```c
struct LevelMapd {
    LevelMapdSurface *layers;  // array of surface pointers, NULL-terminated
};
struct LevelMapdSurface {
    int num_images;
    MapdScrlImage *images[num_images];  // flexible
    int num_palette_entries;
    PaletteEntry palette[256];
};
```
- Multiple layers indexed by MenuId (for super.lvl) or by purpose (terrain=0, fog_source=1)
- Each image is a tiled scrolling bitmap with per-tile transparency flags
- Palette embedded per-layer

### MOBD — Mobile Object Data (Sprites/Animations)
```c
struct LevelMobd {
    LevelMobdSurface *layers[1];  // array indexed by MobdId
};
struct LevelMobdSurface {
    MobdAnimFrame frames[1];      // animation frame data
};
```
- Indexed by MobdId enum
- Contains all sprite animations: units, buildings, UI elements, fonts
- Usually substituted in from supspr.lvl rather than embedded in mission level

### BOXD — Collision/Terrain Grid
```c
struct LevelBoxd {
    LevelBoxdSurface *layers[1];  // typically single layer
};
struct LevelBoxdSurface {
    int max_collision_buckets;  // pool size for runtime collision
    int world_to_tile_x;       // world coord → tile X shift (log2 of tile width)
    int world_to_tile_y;       // world coord → tile Y shift
    int num_tiles_x;           // grid width
    int num_tiles_y;           // grid height
    BoxdTile *tiles[1];        // flexible array of tile pointers
};
struct BoxdTile {
    int type;                  // terrain type (passability flags)
    int x, y;                 // tile position
    int _boxd_tile_field_c;   // unknown
    int z;                    // elevation?
    int w;                    // width/weight?
};
```
- Defines the collision grid for pathfinding and terrain types
- `world_to_tile_x/y` are bit-shift values (not divisors) — world coord >> shift = tile index
- At runtime, `LVL_InitBoxdTerrain` builds a higher-resolution terrain grid from this data

### CPLC — Entity Placement Data
```c
struct LevelCplc {
    LevelCplcSurface *layers;  // array indexed by MenuId or layer ID
};
struct LevelCplcSurface {
    int size;                          // byte size of entity data following
    CplcEntity *next_x_sorted;        // head of X-sorted linked list
    CplcEntity *prev_x_sorted;        // tail of X-sorted linked list
    CplcEntity *next_y_sorted;        // head of Y-sorted linked list
    CplcEntity *prev_y_sorted;        // tail of Y-sorted linked list
};
struct CplcEntity {
    TaskType task_type;                // what kind of entity/script to spawn
    int x, y, z;                      // world position
    CplcEntity *next_x_sorted;        // doubly-linked list sorted by X
    CplcEntity *prev_x_sorted;
    CplcEntity *next_y_sorted;        // doubly-linked list sorted by Y
    CplcEntity *prev_y_sorted;
    CplcSpawnParams spawn_params;      // parameters for entity creation
};
```
- Pre-sorted doubly-linked lists for spatial queries (viewport culling)
- Layer 0 = gameplay entities for mission levels
- For super.lvl: each layer is a different menu screen's UI entities
- On init, the current layer is copied (backup) so it can be modified at runtime
- `CPLC_select(id)` switches to a different CPLC layer

---

## Global State

| Variable | Type | Purpose |
|----------|------|---------|
| `g_level` | `LevelHunk*` | Currently loaded level (mission or super) |
| `g_current_lvl` | `LevelHunk*` | Active running level (set by LVL_RunLevel) |
| `g_current_lvl_sections` | `LevelHunkSection*` | Sections array of active level |
| `g_wait_lvl` | `LevelHunk*` | Preloaded wait/splash level |
| `g_supspr_lvl` | `LevelHunk*` | Shared sprite level (supspr.lvl / sprites.lvl) |
| `g_current_lvl_id` | `LevelId` | Current level enum value |
| `g_prev_level_id` | `LevelId` | Previous level (for reload avoidance) |
| `g_current_lvl_mapd` | `LevelMapd*` | MAPD section pointer |
| `g_current_lvl_mobd` | `LevelMobd*` | MOBD section pointer |
| `g_current_lvl_boxd` | `LevelBoxd*` | BOXD section pointer |
| `g_current_lvl_cplc` | `LevelCplc*` | CPLC section pointer |
| `g_lvl_desc[68]` | `LevelDesc[]` | Static level metadata table |
| `g_level_loading_lock` | `BOOL` | Prevents subsystem callbacks during load |

---

## Level Lifecycle Summary

```
FILE_open(filename)
  │
  ├─ LVL_ReadHunk() → DATA (malloc'd binary blob)
  ├─ LVL_ReadHunk() → RLLC (malloc'd relocation table)
  │
  ├─ HUNK_FixPointers(DATA, RLLC)
  │     ├─ Simple fixup: ptr += base_addr
  │     ├─ Batch fixup: N consecutive ptrs += base_addr
  │     └─ Renderer fixup: replace hunk_id with function pointer
  │
  ├─ free(RLLC)  — no longer needed after fixup
  │
  └─ return DATA as LevelHunk*

LVL_SubstHunk(level, supspr, "MOBD")
  — Replace level's MOBD with shared sprite bank

LVL_RunLevel(level)
  — g_current_lvl = level
  — g_current_lvl_sections = level->sections[0]
  — Init chain: Tasks → Timer → Sound → Render → MAPD → MOBD → BOXD → CPLC → Fade → Window

[... game runs ...]

LVL_Deinit()
  — Reverse teardown of all subsystems
  — g_current_lvl = NULL
  — g_current_lvl_sections = NULL

kknd_mfree(level)  — free the DATA blob
```

---

## Key Design Patterns

### Binary Dump + Pointer Fixup
Level files are memory images — the DATA hunk can be loaded, fixed up, and used in-place without parsing. All structures, linked lists, and pointer arrays are pre-built in the file. The RLLC table just patches offsets to actual addresses. This is extremely fast to load but makes the format platform-specific (32-bit, little-endian data with big-endian IFF headers).

### Shared Sprite Injection
Rather than duplicate sprite data in every level file, all levels share a single sprite bank (`supspr.lvl`). The injection via `LVL_SubstHunk` is a pointer swap — O(n) scan of sections array, O(1) pointer write. Mission levels may still have their own MOBD sections for level-specific sprites, but these get overwritten.

### Level Reuse Optimization
`g_prev_level_id` tracks the last loaded level. If the same level is requested again (e.g., reload after death), the expensive `LVL_LoadLevel` + `LVL_SubstHunk` steps are skipped — only `LVL_RunLevel` re-initializes subsystems.

### g_level_loading_lock
Set during level load to prevent game subsystems (rendering, input, AI) from processing callbacks while data is in an inconsistent state.
