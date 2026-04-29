# MAPD — Background/Scrolling-Image Subsystem

## Overview

MAPD = "Map Data". It is the level file section (`"MAPD"`) containing tiled scrolling background images. Used for **two distinct purposes**:

1. **Menu backgrounds** — each menu screen has its own MAPD layer (static, non-scrolling, sometimes scrolled via camera Y animation)
2. **Level terrain** — the gameplay map background (2 layers: ground + shroud/fog overlay)

The MAPD section is an **array of `LevelMapd` entries**, each indexed by `MenuId`. Each entry points to a `LevelMapdSurface` which holds a palette and one or more `MapdScrlImage`s.

---

## Data Structures

### File-Level (in .lvl)

```
LevelHunk
  └─ sections[] → LevelHunkSection { name[4], ptr }
       └─ "MAPD" → LevelMapd[] (array, indexed by MenuId)
            └─ LevelMapdSurface* layers
                 ├─ num_images
                 ├─ images[num_images] → MapdScrlImage*
                 ├─ num_palette_entries
                 └─ palette (256-entry RGB palette)
```

### `LevelMapd` (kknd.h:2025)
```c
struct LevelMapd {
    LevelMapdSurface *layers;  // single pointer; array of LevelMapd indexed by MenuId
};
```
**Key insight**: `LVL_FindSection("MAPD")` returns `LevelMapd*` — an array. Code accesses `mapd[mapd_id].layers` to get the surface for a given menu/context.

### `LevelMapdSurface` (kknd.h:2016)
```c
struct LevelMapdSurface {
    int num_images;               // number of scroll images in this surface
    MapdScrlImage *images[1];     // flexible array of image pointers
    int num_palette_entries;
    infantr palette;              // 256-entry palette (PaletteEntry[256])
};
```
**DECOMPILATION CONCERN**: `num_palette_entries` field after `images[1]` — for multi-image surfaces, this layout is wrong. Likely `images[]` is a true flexible array and the palette fields follow after all image pointers.

### `MapdScrlImage` (kknd.h:1990)
```c
struct MapdScrlImage {
    Blitter renderer;         // function pointer: render_mode_scrl_draw
    int tile_x_size;          // width of each tile in pixels
    int tile_y_size;          // height of each tile in pixels
    int num_x_tiles;          // horizontal tile count
    int num_y_tiles;          // vertical tile count
    MapdScrlImageTile *tiles[16];  // tile pointers (flexible array, can be >16)
};
```
Total image size: `tile_x_size * num_x_tiles` x `tile_y_size * num_y_tiles` pixels.

The `renderer` field is a `Blitter` function pointer called with mode 0=draw, 1=get_width, 2=get_height.

### `MapdScrlImageTile` (kknd.h:1983)
```c
struct MapdScrlImageTile {
    int flags;                // bit 0: unused?, bit 1: transparency mode
    unsigned char pixels[1];  // raw pixel data (tile_x_size * tile_y_size bytes)
};
```
Tile can be NULL in the tile array → that tile position is empty/transparent.

### `MapdCamera` (kknd.h:1975)
```c
struct MapdCamera {
    int x;   // 8.8 fixed-point world X
    int y;   // 8.8 fixed-point world Y
    int z;   // unused in practice (always 0)
};
```
Camera position is in **8-bit fixed-point** (value >> 8 = pixel coordinate).

### `MapdRenderNode` (kknd.h:2031)
```c
struct MapdRenderNode {
    MapdRenderNode *next;     // doubly-linked list
    MapdRenderNode *prev;
    RenderNode *job;          // render pipeline node
    MapdScrlImage *scrl;      // the scroll image to render
    int z;                    // z-order for sorting
};
```
Pool of 16 nodes allocated at init. Linked-list free pool + active list.

---

## Global State

| Variable | Type | Purpose |
|----------|------|---------|
| `g_current_lvl_mapd` | `LevelMapd*` | Pointer to MAPD section of current level |
| `g_current_lvl_mapd_num_layers` | `int` | Number of MAPD entries (MenuId slots) |
| `g_current_lvl_mapd_verified` | `int` | 1 if MAPD section found & initialized |
| `g_mapd_camera` | `MapdCamera` | Current camera position (8.8 fixed-point) |
| `g_mapd_camera_target` | `Entity*` | Entity camera follows (NULL = no auto-follow) |
| `mapd_camera_controller` | `fn ptr` | Camera update function (default: `mapd_camera_follow_target`) |
| `mapd_render` | `fn ptr` | Render state handler (default: `render_state_handler_mapd`) |
| `g_mapd_render_node_pool` | `MapdRenderNode*` | Allocated pool of 16 render nodes |
| `g_mapd_render_node_next_free` | `MapdRenderNode*` | Free list head |
| `g_mapd_render_node_head/tail` | `MapdRenderNode*` | Active render node list |
| `g_mapd_initialized` | `int` | Set during LVL_RunLevel |
| `node[3]` | `MapdRenderNode*[3]` | Global background render slots: [0]=ground, [1]=shroud, [2]=unused |
| `g_fog_of_war` | `MapdRenderNode*` | Fog-of-war render node (separate from node[]) |

---

## Core Functions

### `LVL_InitMapd()` @ 0x439230
- Called from `LVL_RunLevel()` during level load sequence
- Finds `"MAPD"` section via `LVL_FindSection`
- Counts number of MAPD layers (entries where `layers` pointer is non-NULL)
- Allocates pool of 16 `MapdRenderNode`s (0x140 = 16 * 0x14 bytes)
- Sets up free-list as circular doubly-linked list
- Initializes camera to (0,0,0), controller to `mapd_camera_follow_target`

### `MAPD_Draw(mapd_id, image_id, z)` @ 0x439300
- **mapd_id**: MenuId index into LevelMapd array — selects which background set
- **image_id**: Index into that surface's `images[]` — selects which scrolling image
- **z**: Z-order value for render sorting
- Validates: mapd_id ≤ num_layers, image_id ≤ num_images, free node available
- Allocates a `MapdRenderNode` from pool, creates a `RenderNode` via `REND_AddNode()`
- Links the MapdScrlImage and z into the node
- Returns the MapdRenderNode pointer (stored in `node[]` globals)

### `MAPD_RemoveRenderable(node)` @ 0x4393C0
- Marks render job for deletion (sets bit 31 of flags)
- Unlinks from active list, returns to free pool

### `render_state_handler_mapd(mapd, node)` @ 0x439200
- Default render-state updater for MAPD backgrounds
- Sets `cmd.image = mapd->scrl`
- Sets screen position: `cmd.x = -(camera.x >> 8)`, `cmd.y = -(camera.y >> 8)`
- This converts world-space camera to screen-space offset for the scrolling image

### `GAME_UpdateCamera()` @ 0x4393F0
- Called each frame in game loop
- If `g_mapd_camera_target` exists, calls `mapd_camera_controller(&g_mapd_camera, target)`

### `mapd_camera_follow_target(camera, target)` @ 0x439190
- Sets camera to center on target entity position minus half-screen offset

### `MAPD_Deinit()` @ 0x439420
- Frees the render node pool

---

## Rendering Pipeline: Scrolling Image Draw

### `render_mode_scrl_draw(cmd, mode)` @ 0x412860
The **Blitter** function stored in `MapdScrlImage.renderer`. Handles 3 modes:
- **mode 0**: Draw — tiles the image onto screen with viewport clipping
- **mode 1**: Return total width (`num_x_tiles * tile_x_size`)
- **mode 2**: Return total height (`num_y_tiles * tile_y_size`)

Drawing logic (mode 0):
1. Check if image is within viewport bounds
2. Calculate which tiles are visible (from negative offsets = camera position)
3. Iterate visible tiles row-by-row, column-by-column
4. For each non-NULL tile: call pixel blitter (`dword_4785E0` or `dword_478A10` based on transparency flag)
5. Tile flag bit 0 selects transparent vs opaque blitter

### `REND_get_width(cmd)` / `REND_get_height(cmd)` @ 0x4125D0 / 0x4125F0
- Call the blitter with mode 1/2 to get total image dimensions
- **Decompiler bug**: Shows no arguments passed to `image->renderer()` but actually passes cmd+mode

---

## Background Types

### 1. Wait Screen Background
**File**: `wait.lvl`
**Code**: `GAME_WaitScreen()` @ ~0x422690
```c
mapd = LVL_FindSection("MAPD");
palette_40E400(mapd->layers->palette.entries);  // index 0
node[0] = MAPD_Draw(MenuId_Main, 0, 0);        // layer 0, image 0
g_mapd_camera.y = 125440;                       // 490 pixels down (scrolled)
// ... render 180 frames ...
MAPD_RemoveRenderable(node[0]);
```
Single static background, camera Y-scrolled to show bottom portion. Shown during initial load.

### 2. Menu Backgrounds (super.lvl)
**File**: `super.lvl` (shared menu level)
**Code**: `GAME_PrepareSuperLvl(mapd_id)` @ 0x422860
The MAPD section in super.lvl contains **16 surfaces** (one per MenuId). Each menu screen:

1. Removes previous `node[0]`
2. Looks up `MAPD[menu_id]` palette → calls `palette_40E400()`
3. Calls `MAPD_Draw(menu_id, 0, -10)` → image 0, z=-10 (behind UI)
4. Calls `CPLC_select(menu_id)` to activate matching UI entity set

| MenuId | Value | Screen | z |
|--------|-------|--------|---|
| MenuId_Main | 0 | Main menu | 0 or -10 |
| MenuId_Multiplayer | 1 | Multiplayer menu | -10 |
| MenuId_NewCampaign | 2 | New Campaign | -10 |
| MenuId_3 | 3 | (unknown) | -10 |
| MenuId_IpxSetup | 4 | IPX Network Setup | -10 |
| MenuId_Serial | 5 | Serial Connection | -10 |
| MenuId_Modem | 6 | Modem Setup | -10 |
| MenuId_HostGame | 7 | Host Game | -10 |
| MenuId_JoinGame | 8 | Join Game | -10 |
| MenuId_AddModem | 9 | Add Modem | -10 |
| MenuId_ModemPhonebook | 10 | Modem Phonebook | -10 |
| MenuId_PlayMission | 11 | Play Mission | -10 |
| MenuId_Credits | 12 | Credits | -10 |
| MenuId_MissionComplete | 13 | Mission Complete | -10 |
| MenuId_NewMissions | 14 | New Missions | -10 |
| MenuId_Kaos | 15 | Kaos Screen | -10 |

**Menu transition pattern** (each menu task):
```
FADE_out → MAPD_RemoveRenderable(node[0]) → yield 3 frames
→ palette_40E400(MAPD[id].palette) → node[0] = MAPD_Draw(id, 0, -10)
→ CPLC_select(id) → yield 1 frame → FADE_in
```

### 3. FMV Mission Briefing Background (fmv.lvl)
**File**: `fmv.lvl`
**Code**: `MOVIE_Play()` @ ~0x422AF0
Uses **faction-dependent** MAPD index:
```c
if (g_current_lvl_id >= LevelId_Mute_01) {
    // Mutants: use MAPD[0] (MenuId_Main)
    node[0] = MAPD_Draw(MenuId_Main, 0, 0);
    v8 = *(PaletteEntry**)LVL_FindSection("MAPD");        // palette from layer 0
} else {
    // Survivors: use MAPD[1] (MenuId_Multiplayer)
    node[0] = MAPD_Draw(MenuId_Multiplayer, 0, -10);
    v8 = *((PaletteEntry**)LVL_FindSection("MAPD") + 1);  // palette from layer 1
}
```
Two faction-specific backgrounds in fmv.lvl, selected by current campaign side.

### 4. Level Gameplay Ground (mission .lvl)
**File**: Each mission's .lvl
**Code**: Game start @ ~0x422F70
```c
Section = LVL_FindSection("MAPD");
palette_40E400((infantr*)(*Section + 16));       // raw offset: layers->palette
node[0] = MAPD_Draw(MenuId_Main, 0, 0);         // ground layer, image 0, z=0
node[1] = MAPD_Draw(MenuId_Main, 1, 0x20000000);// shroud layer, image 1, z=0x20000000
node[1]->job->render_state_handler = sub_448360; // custom handler: follows node[0] position
```
- **node[0]**: Main terrain ground (z=0, lowest)
- **node[1]**: Shroud/darkness overlay (z=0x20000000, rendered on top of ground but under units)
- Shroud render handler (`sub_448360`) copies node[0]'s screen position, sets z=0xFFFFF

### 5. Fog of War (runtime-generated from MAPD tiles)
**Code**: Fog init @ ~0x448560 (function containing `sub_44BD50` call)
```c
result = MAPD_Draw(MenuId_Multiplayer, 0, 0x10000000);  // MAPD layer 1 image 0
g_fog_of_war = result;
// Override render handler to sub_448390
// Clone the MapdScrlImage structure with expanded tile grid
// 16 fog tile types: tiles[0..15] from original SCRL image
// Runtime tile array sized: (map_tiles_x + 4) * (map_tiles_y + 4)
```
- Uses **MAPD layer 1 (MenuId_Multiplayer)** as source for fog tile graphics
- Creates a **runtime copy** of MapdScrlImage with dynamically updated tile pointers
- Fog render handler (`sub_448390`) offsets position by 2 tiles from ground position
- **16 tile types**: fully fogged, edge transitions, fully revealed (stored as tile indices 0-15)
- `g_fog_of_war_tile0` through `g_fog_of_war_tile15` hold pointers to each fog variant
- Tile grid updated each frame based on unit vision

### 6. Shroud Tile Masking (`sub_44BD50`)
```c
// For each tile in ground and shroud images:
//   tile->flags |= 2;  // set transparency bit
```
Called after level MAPD setup. Sets bit 1 on all tile flags in both ground and shroud images, enabling transparent blitting mode for the tile renderer.

---

## Camera System

### Fixed-Point Coordinates
Camera uses **8.8 fixed-point**: multiply pixel coords by 256 (`<< 8`) for world coords.

### Camera Bounds Clamping
```
x_min = 0
x_max = (image_width - screen_width + 32) << 8
y_min = 0
y_max = (image_height - screen_height) << 8
```
The `+32` on x accounts for right-side overflow margin.

### Camera Movement (in-game)
- **Keyboard scroll**: 1024 normal, 3840 fast (with shift/ctrl)
- **Mouse edge scroll**: Separate mouse-driven panning system
- **Camera shake**: Additive offset (`g_camera_shake_x/y`), subtracted before movement, re-applied after
- **Follow target**: `mapd_camera_follow_target()` centers on entity

### Camera in Menus
- Usually `g_mapd_camera = (0, 0)` — top-left of background
- Wait screen: `g_mapd_camera.y = 125440` (scrolled 490px down)
- Some menus animate camera Y for scrolling effects

---

## Coordinate Spaces

| Space | Range | Conversion |
|-------|-------|------------|
| World (fixed-point) | 0 to (mapsize << 8) | Standard game coordinates |
| World (pixel) | 0 to mapsize | world >> 8 |
| Screen | 0 to screen_w/h | world_pixel - camera_pixel |
| MAPD draw offset | negative camera | `-(camera >> 8)` |

Entity screen position: `screen_x = (entity.x >> 8) - (camera.x >> 8) - frame.x`

---

## Level File Lifecycle

```
LVL_LoadLevel(path)       → loads .lvl into memory
LVL_RunLevel(lvl)         → initializes all subsystems:
  └─ LVL_InitMapd()       → finds MAPD section, allocates render nodes
  └─ LVL_InitMobd()       → sprite data
  └─ BOXD_collisions_init → collision grid
  └─ LVL_cplc_init        → entity/UI templates
  └─ ... (tasks, timer, fade, etc.)

// Game uses:
LVL_FindSection("MAPD")  → returns LevelMapd* array
MAPD_Draw(id, img, z)    → creates renderable for a background
MAPD_RemoveRenderable()  → destroys renderable

LVL_Deinit()             → tears down in reverse:
  └─ MAPD_Deinit()       → frees render node pool
```

---

## MenuId Double-Duty Issue

`MenuId` enum serves **two unrelated purposes**:
1. Menu screen identifier (for CPLC UI entity sets)
2. MAPD layer index (background image selector)

In menus: MenuId maps 1:1 to both its background and UI set.
In levels: `MenuId_Main` (0) = ground, `MenuId_Multiplayer` (1) = fog source tiles.

The level MAPD array typically has only 2 entries (ground + fog tiles), while menu MAPD arrays have up to 16.
