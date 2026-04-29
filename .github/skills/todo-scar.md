# Scar → `Scar` (Terrain Battle Damage)

**Original source**: `C:\k\Scripts\Scar.cpp` (confirmed via rand call at L54234)

## Identity

`Scar` is a **terrain scar** — a battle damage decal placed on the ground when explosions/projectiles hit. Pool of 40 with LRU eviction. 8 severity levels (repeated hits to same tile intensify the scar).

## Struct (kknd.h L3340)

```c
struct KKND::Scar {          // → Scar
    Scar *next;              // linked list next
    Scar *prev;              // linked list prev
    ptrdiff_t mobd_frame;      // → damage_level (0–7, indexes into scar anim table)
    Entity *entity;            // visual decal entity on terrain
};
```
Size: 16 bytes. Pool: 40 nodes (640 bytes total).

## Globals

| Current | Proposed | Purpose |
|---------|----------|---------|
| `g_Scar_pool` | `g_scar_pool` | malloc'd array of 40 Scar nodes |
| `g_Scar_list_free_head` | `g_scar_free_head` | Singly-linked free list |
| `g_Scar_list_head` | `g_scar_active_head` | Doubly-linked active list head (sentinel) |
| `g_Scar_list_tail` | `g_scar_active_tail` | Active list tail (oldest = LRU eviction target) |

## Functions

| Current | Proposed | Line | Purpose |
|---------|----------|------|---------|
| `Scar_init()` | `scar_pool_init` | 54172 | Alloc 40 nodes, chain free list, init active list |
| `sub_4389A0(x, y)` | `scar_apply` | 54194 | Add or intensify terrain scar at world position |

## `scar_apply` Logic (sub_4389A0)

1. Check terrain tile at `(x, y)` allows scarring
2. Random 19% chance (`rand < 0x1388`) to actually place scar
3. **If tile already has a scar**: increment `damage_level` (mod 8), update entity animation
4. **If new scar needed**:
   - Allocate from free list (or evict LRU tail node)
   - Create entity with `MobdId_Explosions`
   - Set `render_state_handler_terrain_detail`
   - Insert at head of active list (most recent)
5. `dword_46BBE8[8]` maps damage_level → specific MOBD animation frame (increasingly blackened ground)

## Callers

All from explosion/projectile impact code:
- Explosion task (L7591) — scars grid of tiles around blast
- Rocket impact (L52432, L52903)
- Mech impact (L53007)
- Generic projectile (L53551)
- Another explosion context (L54483)

## Key Design

- **LRU eviction**: When pool is full, tail node (oldest scar) is recycled. This means oldest battle damage disappears as new explosions happen.
- **Progressive damage**: Same tile hit multiple times → `damage_level` increases → more intense scar sprite. 8 levels total.
- **Render**: Uses `render_state_handler_terrain_detail` — draws at tile position relative to camera, beneath units.
- **19% spawn rate**: Not every explosion scars the terrain — prevents visual clutter.

## Naming

| Current | Proposed |
|---------|----------|
| `Scar` | `Scar` |
| `sub_4389A0` | `scar_apply` |
| `Scar_init` | `scar_pool_init` |
| `mobd_frame` | `damage_level` |
| `dword_46BBE8` | `g_scar_anim_frames` |
