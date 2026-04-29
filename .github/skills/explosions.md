# Explosions & Gore System

**Original sources**: `C:\k\Scripts\Schrap.cpp` (gore/debris), `C:\k\Scripts\Building.cpp` (building explosions), `C:\k\Scripts\Projectl.cpp` (nuke)

## Overview

5 distinct death/explosion visual effect types, all using `MobdId_Explosions` (MOBD 22) for animation frames. Two throttle pools limit simultaneous effects:

| Pool | Global | Max | Used By |
|------|--------|-----|---------|
| Explosions | `g_num_explosions` | 10 | Fire/explosion anims |
| Debris/blood | `g_num_explosions_2` | 30 | Gore chunks, blood drops, nuke fragments |

---

## 1. GORE — Infantry Death (`Schrap.cpp`)

**Trigger**: `unit_mode_419560()` (L26954) — universal death handler, when `stats->is_infantry` is true.

### Flow
```
unit_mode_419560(unit)
  ├─ play random death scream (5 variants: SoundId_174–178)
  ├─ switch entity MOBD to MobdId_Explosions
  ├─ entity_anim_load(568 for Survivors, 636 for Evolved)  ← corpse death anim
  ├─ unit_438D90(unit) → spawn 6x task_bloodsplats
  └─ wait 60 ticks → remove entity
```

### `task_bloodsplats` (L54321) — Small blood droplet
- MOBD: `MobdId_69` (dedicated gore sprite sheet)
- Picks 1 of 5 blood spray anims (`dword_46BC84[5]`)
- Launches upward: `z_speed = rand(127) + 336`, gravity = `-10`
- Falls until `z < 0` → removes itself
- No landing splat (just disappears)

### Proposed Names
| Current | Proposed |
|---------|----------|
| `unit_mode_419560` | `unit_mode_death` |
| `unit_438D90` | `gore_spawn_debris` |
| `task_bloodsplats` | `task_blood_droplet` |
| `MobdId_69` | `MobdId_Gore` |
| `dword_46BC84` | `g_blood_droplet_anims` |

---

## 2. VEHICLE EXPLOSION + DEBRIS (`Schrap.cpp`)

**Trigger**: `unit_mode_419560()` for non-infantry, non-DireWolf units.

### Flow
```
unit_mode_419560(unit)  [vehicle branch]
  ├─ unit_explode_1(unit) → task_explosion_1 (medium fireball)
  ├─ unit_438D90(unit) → spawn 24x sub_438C20 (large debris chunks)
  ├─ show unit idle sprite (wreckage frame)
  └─ wait 60 ticks → remove
```

### `sub_438C20` (L54344) — Large flying debris chunk
- MOBD: `MobdId_69` (gore sheet, 9 chunk variants in `dword_46BC60[9]`)
- Random XY speed from `g_blood_x_speeds[8]` / `g_blood_y_speeds[8]` ± random offset
- Launch up: `z_speed = rand(127) + 352`
- Gravity: `-9` for small chunks (index ≤ 2), `-5` for larger (slower fall)
- **On landing**: switches to `MobdId_Explosions` anim 412 (blood pool/splat)
- Plays splat anim to completion → removes

### `task_explosion_1` (L72312) — Medium fireball
- Plays `SoundId_3` (generic explosion boom)
- Uses `render_state_handler_explosion` (additive/overlay rendering)
- Anim frame offset: **144** (medium explosion)
- Plays once → removes

### Proposed Names
| Current | Proposed |
|---------|----------|
| `sub_438C20` | `task_gore_chunk` |
| `unit_explode_1` | `explosion_spawn_medium` |
| `task_explosion_1` | `task_explosion_medium` |
| `dword_46BC60` | `g_gore_chunk_anims` |
| `g_blood_x_speeds` | `g_debris_x_speeds` |
| `g_blood_y_speeds` | `g_debris_y_speeds` |

---

## 3. BUILDING EXPLOSION — Multi-stage Cascade (`Building.cpp`)

**Trigger**: `unit_mode_building_on_death()` (L7706)

### Flow
```
unit_mode_building_on_death(unit)
  ├─ unit_explode_1 → task_explosion_1 (immediate medium fireball)
  ├─ GAME_ShakeCamera()  ← screen shake!
  ├─ [cleanup: production, turret, task channels]
  ├─ unit_explode_2 → task_explosion_2 (BIG cascade)
  │    ├─ spawn task_explosion_3 (delayed ground explosion at +130 ticks)
  │    ├─ spawn 8x task_explosion_4 (random small explosions, staggered)
  │    └─ apply terrain scars in 3×3 tile grid via scar_apply
  ├─ unit_438D90(unit) → 24x sub_438C20 (debris chunks)
  └─ wait 165 ticks → remove building
```

### `task_explosion_2` (L7533) — Big cascading explosion coordinator
- Spawns `task_explosion_3` (delayed large ground boom)
- Spawns 8× `task_explosion_4` (small scattered blasts) with random timing
- Total duration capped at ~120 ticks
- After all sub-explosions: applies `scar_apply` in 3×3 grid (x ± 0x2000, y ± 0x2000)

### `task_explosion_3` (L7512) — Delayed ground explosion
- Waits **130 ticks** after building starts dying
- Anim 0 (LARGE explosion) + `SoundId_3`
- Offset y+2048 from parent

### `task_explosion_4` (L7488) — Small random explosion
- Random position within ±0x2000 of parent on both axes
- Anim **220** (small explosion)
- Elevated at z=0x4000
- `SoundId_3`

### Proposed Names
| Current | Proposed |
|---------|----------|
| `unit_mode_building_on_death` | `building_mode_death` |
| `unit_explode_2` | `explosion_spawn_cascade` |
| `task_explosion_2` | `task_explosion_cascade` |
| `task_explosion_3` | `task_explosion_delayed_large` |
| `task_explosion_4` | `task_explosion_scatter_small` |

---

## 4. NUKE — Full Weapon System (`Projectl.cpp`)

**Trigger**: Aircraft (Wasp/Bomber) fire nuke projectile → `proj_mode_nuke` state machine.

### Nuke Struct (12 bytes)
```c
struct Nuke {
    Entity *entity;
    Task *task;
    void (*mode)(Nuke *);  // state function pointer
};
```

### State Machine Flow
```
proj_mode_nuke (L6467) — coroutine tick dispatcher
  │
  ├─ nuke_mode_init (L6453)
  │    entity falls: z_speed=512, z_accel=-34
  │    play SoundId_163 (launch/whistle)
  │    → nuke_mode_falling
  │
  ├─ nuke_mode_falling (L6446)
  │    wait until z <= 0
  │    → nuke_mode_impact
  │
  └─ nuke_mode_impact (L6416)
       stop all movement
       play SoundId_10 (massive explosion)
       switch to MobdId_Explosions
       entity_anim_load(proj->mobd_lookup_offset_impact)
       sub_439180(entity) → spawn 20x sub_438F50 (fire fragments)
       entity_40D8B0(entity, radius) → damage all units in blast radius
       → nuke_mode_finalize → cleanup
```

### `sub_439180` (L54559) — Nuke fragment spawner
- Disables parent physics (`mode_null`)
- Spawns up to **20** fire fragment tasks (`sub_438F50`)
- Limited by `g_num_explosions_2 > 30`

### `sub_438F50` (L54460) — Nuke fire fragment
- Random type 0–7 from `dword_46BC98[8]` (building rubble anims)
- Staggered delay: type < 4 → 60 + rand(63) ticks; type ≥ 4 → 30 + rand(63) ticks
- Random scatter position ±128 pixels from parent
- Each fragment applies `scar_apply` at its position!
- Plays rubble/fire anim to completion → removes

### Proposed Names
| Current | Proposed |
|---------|----------|
| `proj_mode_nuke` | `proj_mode_nuke` (already good) |
| `nuke_mode_401DC0` | `nuke_mode_falling` |
| `nuke_mode_401D30` | `nuke_mode_impact` |
| `sub_439180` | `nuke_spawn_fire_fragments` |
| `sub_438F50` | `task_nuke_fire_fragment` |
| `dword_46BC98` | `g_nuke_fragment_anims` |

---

## 5. DireWolf Death — Special Case

Uses its own MOBD (`MobdId_Mute_DirewWolf`, anim offset 1216) for death anim but still spawns gore via `unit_438D90` → 6× `task_bloodsplats` (same as infantry).

---

## 6. WarMastadont — Hybrid Death

Gets BOTH chunk types: 8× `sub_438C20` (large chunks) + 8× `task_bloodsplats` (blood droplets). Double gore for big beast.

---

## 7. Rocket Trail Puffs — `task_rocket_proj_explosion` (L52253)

Smoke puffs spawned during homing rocket flight (every 12 ticks, max 20 active). Anim 412 (small puff). Separate throttle via `g_num_active_explosions`.

---

## 8. Aircraft Crash Smoke — `task_439050_explosion` (L54500)

Spawned by `unit_4390F0` during aircraft falling animation. Medium explosion anim (144) + `SoundId_3`. Used for trailing smoke as planes spiral down.

---

## MOBD Animation Frame Map (`MobdId_Explosions`)

| Offset | Visual | Used By |
|--------|--------|---------|
| 0 | Large fireball | Building delayed explosion, aircraft crash |
| 144 | Medium fireball | Vehicle/building immediate explosion, aircraft smoke |
| 220 | Small scattered fireball | Building cascade sub-explosions |
| 412 | Blood pool / smoke puff | Gore chunk landing splat, rocket trail |
| 568 | Survivor infantry death | Infantry gore (Survivors) |
| 636 | Evolved infantry death | Infantry gore (Evolved) |

## MOBD Animation Frames (`MobdId_69` / MobdId_Gore)

| Array | Count | Content |
|-------|-------|---------|
| `dword_46BC60[9]` | 9 | Gore chunk flight anims (index ≤ 2 = small, > 2 = large) |
| `dword_46BC84[5]` | 5 | Blood droplet spray anims |
| `dword_46BC98[8]` | 8 | Nuke fire fragment / rubble anims |

---

## Throttle System

```c
// Pool 1: Fireballs
BOOL can_have_more_explosions() {      // L54303
    if (g_num_explosions > 10) return 0;
    ++g_num_explosions;
    return 1;
}
void explosion_finished() {
    if (g_num_explosions > 0) --g_num_explosions;
}

// Pool 2: Gore/debris/fragments — manual increment/decrement
// g_num_explosions_2 > 30 → stop spawning
```

---

## Complete Death Handler Dispatch (`unit_mode_419560`)

```
unit dies →
  ├─ is_infantry? → gore path (death anim + 6 blood droplets)
  ├─ is DireWolf? → own death anim + 6 blood droplets
  ├─ is WarMastadont? → own death + 8 chunks + 8 droplets
  └─ else (vehicle)? → medium fireball + 24 debris chunks

building dies → building_mode_death (separate handler)
  → immediate fireball + cascade (9 sub-explosions) + 24 chunks + scars + camera shake
```

---

## Sound Summary

| SoundId | Context |
|---------|---------|
| `SoundId_3` | Generic explosion boom (all fireballs) |
| `SoundId_9` | Aircraft crash impact |
| `SoundId_10` | Nuke impact (massive boom) |
| `SoundId_163` | Nuke launch/falling whistle |
| `SoundId_174–178` | Infantry death screams (5 random) |
