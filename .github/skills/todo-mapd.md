# MAPD System — TODO / Proposals

## Rename Suggestions

### Types

| Current | Proposed | Reason |
|---------|----------|--------|
| `LevelMapdSurface` | `MapdLayerDef` or `MapdSurfaceDef` | "Surface" is ok but "Layer" better matches how it's used as an array element |
| `infantr` | `Palette256` | Nonsense decompiler name, it's just a 256-entry palette wrapper |
| `MapdScrlImage` | `MapdTiledImage` or `MapdScrollImage` | "Scrl" abbreviation inconsistent; full word better |
| `MapdScrlImageTile` | `MapdTile` | Simpler, unambiguous |
| `MapdRenderNode` | — (fine as-is) | |
| `MapdCamera` | — (fine as-is) | |

### Functions

| Current | Proposed | Reason |
|---------|----------|--------|
| `render_state_handler_mapd` | `MAPD_RenderStateHandler` | Match MAPD_ prefix convention |
| `mapd_camera_follow_target` | `MAPD_CameraFollowTarget` | Match MAPD_ prefix |
| `render_mode_scrl_draw` | `MAPD_BlitTiledImage` or `REND_BlitScrollImage` | Describes what it does |
| `render_mode_scrl_setup` | `MAPD_BlitSetup` or remove entirely | Function is a no-op stub (returns 1) |
| `palette_40E400` | `PAL_SetActive` or `PAL_Apply` | Sets the active palette |
| `sub_448360` | `MAPD_ShroudRenderHandler` | Copies ground position, sets shroud z |
| `sub_448390` | `MAPD_FogRenderHandler` | Copies ground position with tile offset for fog |
| `sub_44BD50` | `MAPD_SetAllTilesTransparent` or `MAPD_EnableTileBlending` | Sets flag bit 2 on all tiles |
| `sub_444080` | `MENU_InitAnimatedOverlay` or `MENU_StartMobd79Task` | Starts mobd79 animated overlay task for menus |

### Globals

| Current | Proposed | Reason |
|---------|----------|--------|
| `node[3]` | `g_mapd_bg_nodes[3]` or `g_mapd_active_layers[3]` | Too generic; clarify it's MAPD background slots |
| `g_fog_of_war_scrl_2` | `g_fog_of_war_source_image` | It's the original/source SCRL from the level file |
| `g_fog_of_war_scrl` | `g_fog_of_war_runtime_image` | It's the dynamically-built fog image |
| `g_fog_of_war_tile0..15` | Convert to array `g_fog_tile_types[16]` | 16 individual globals should be one array |
| `g_current_lvl_mapd_verified` | `g_mapd_has_data` | "verified" is misleading, it just means section was found |
| `mapd_camera_controller` | `g_mapd_camera_update_fn` | Naming convention: globals get g_ prefix |
| `mapd_render` | `g_mapd_render_state_fn` | Same |
| `dword_47A01C` | Not a real variable — it's the memory right after `node[3]` | Used as sentinel for cleanup loop end address |

---

## Struct Issues

### `LevelMapdSurface` Layout Is Wrong
```c
struct LevelMapdSurface {
    int num_images;
    MapdScrlImage *images[1];     // flexible array
    int num_palette_entries;      // ← this is WRONG position
    infantr palette;
};
```
**Problem**: For surfaces with >1 image, `num_palette_entries` would overlap `images[1]`. The actual binary layout is likely:
```c
struct LevelMapdSurface {
    int num_images;
    MapdScrlImage *images[num_images];  // variable length
    // followed by:
    int num_palette_entries;
    PaletteEntry palette[num_palette_entries];
};
```
Or more likely: palette always at a fixed offset. Code accesses it as `*(_DWORD*)Section + 16)` which is `layers + 16 bytes` — that's `num_images(4) + images[0](4) + images[1](4) + images[2](4)` = 3 image pointers before palette. Needs investigation on actual level files.

### `MapdScrlImage.tiles[16]` May Be Too Small
The fog system creates images with `(map_tiles_x + 4) * (map_tiles_y + 4)` tiles. The `tiles[16]` declaration is only for the **source template** tiles (16 fog variants). The runtime fog image is `malloc`'d separately. However, level background images may also have more than 16 tiles. This should be `tiles[]` (flexible array member).

### `LevelMapd` Is Suspiciously Simple
```c
struct LevelMapd {
    LevelMapdSurface *layers;
};
```
This is just a pointer. The MAPD section is an **array of these pointers** — one per MenuId slot. Could simplify to `LevelMapdSurface **mapd_layers` if we're being honest about what it is.

---

## Decompilation Bugs

### `REND_get_width` / `REND_get_height` — Missing Arguments
```c
return ((int (*)(void))image->renderer)();  // BUG: should pass (cmd, 1) or (cmd, 2)
```
Decompiler lost the arguments. The blitter signature is `int(RenderCommand*, int mode)` where mode 1=width, 2=height.

### `render_mode_scrl_setup` Is a No-Op
```c
int render_mode_scrl_setup() { return 1; }
```
Called in two places. May have been meaningful in debug/development builds, or it's a function pointer placeholder that got inlined elsewhere.

---

## Enum Proposals

### Fog Tile Types (new enum)
Based on fog tile indices 0-15 used in fog-of-war system:
```c
enum FogTileType {
    FogTile_FullyFogged = 0,      // tile[0] = NULL/transparent
    FogTile_FullyRevealed = 1,    // tile[1] = clear
    FogTile_EdgeN = 2,            // tile[2]..tile[15]: edge transitions
    FogTile_EdgeS = 3,
    FogTile_EdgeE = 4,
    FogTile_EdgeW = 5,
    FogTile_CornerNE = 6,
    FogTile_CornerNW = 7,
    FogTile_CornerSE = 8,
    FogTile_CornerSW = 9,
    FogTile_InnerNE = 10,
    FogTile_InnerNW = 11,
    FogTile_InnerSE = 12,
    FogTile_InnerSW = 13,
    FogTile_PeninsulaNS = 14,     // guesses for 14-15
    FogTile_PeninsulaNS2 = 15,
};
```
**NOTE**: Exact mapping of tiles 2-15 to edge/corner types needs verification from actual fog rendering or tile graphics.

### MapdNodeSlot (new enum for `node[]` indices)
```c
enum MapdNodeSlot {
    MapdSlot_Ground = 0,
    MapdSlot_Shroud = 1,
    MapdSlot_Reserved = 2,     // unused, always NULL
};
```

### MapdBlitMode (for Blitter mode parameter)
```c
enum MapdBlitMode {
    MapdBlit_Draw = 0,
    MapdBlit_GetWidth = 1,
    MapdBlit_GetHeight = 2,
};
```

### MapdTileFlags
```c
enum MapdTileFlags {
    MapdTile_Transparent = 1,  // bit 0: use transparent blitter
    MapdTile_Blended = 2,     // bit 1: set by sub_44BD50 on all gameplay tiles
};
```

---

## Architecture Observations

### MenuId Overload
`MenuId` serves as both:
- UI screen identifier (CPLC entity set selector)
- MAPD background layer index

For **menu levels** (super.lvl): these align perfectly — each MenuId maps to both its background and UI entities.

For **gameplay levels**: `MenuId_Main` (0) = ground terrain, `MenuId_Multiplayer` (1) = fog tile source. The enum names are completely misleading in gameplay context.

**Proposal**: Add comments or type aliases:
```c
typedef MenuId MapdLayerId;  // when used as MAPD array index
// MapdLayerId_Terrain = 0 (MenuId_Main)
// MapdLayerId_FogTiles = 1 (MenuId_Multiplayer)
```

### Z-Order Convention
| Layer | Z value | Purpose |
|-------|---------|---------|
| Menu background | -10 | Behind everything |
| Ground terrain | 0 | Base layer |
| Shroud | 0xFFFFF (~1M) | Darkness overlay, set by render handler |
| Fog of war | 0x10000000 (~268M) | Fog overlay above shroud |
| Shroud (from MAPD_Draw) | 0x20000000 (~537M) | Initial z, overridden by handler to 0xFFFFF |

### Camera Shake Integration
Camera shake is **applied directly to g_mapd_camera**, then subtracted for next frame's base calculation. This means shake affects world-to-screen conversion for all entities, not just the background. Clean design.

### Render Node Pool Limit
Only 16 MapdRenderNodes allocated. With 3 background slots + 1 fog = 4 active maximum in gameplay. Menus use 1. Plenty of headroom, but hard limit exists.

---

## Investigation Needed

1. **`dword_46580C`** — controls which blitter is used for tiles. When set, forces transparent blitter for all tiles. What sets it?
2. **`sub_439610(Buffer)`** — called before LVL_RunLevel in game start, takes a path. Sound/music related?
3. **Actual fog tile graphics** — need to verify which of tiles[0..15] are which edge transitions
4. **`LevelMapdSurface` true layout** — need to examine a hex dump of a .lvl MAPD section to confirm struct layout
5. **`node[2]`** — always NULL. Was it planned for a third layer? Parallax? Water overlay?
6. **Menu 3 (`MenuId_3`)** — never referenced in code. What background was assigned to it?
