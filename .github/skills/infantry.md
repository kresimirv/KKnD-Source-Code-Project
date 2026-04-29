# Infantry Mode Workflow — Complete Reference

## Overview

Infantry units use `UNIT_Handler_Infantry` as their task callback. Every frame it calls `unit->mode(unit)` — a function pointer that implements one tick of the current behavioral state. Mode transitions happen by assigning a new function pointer to `unit->mode`.

Original source: `C:\k\Scripts\Infantry.cpp` (confirmed via embedded debug strings).

**Address range:** 0x415540 – 0x419720 (all infantry-specific modes live in this block).

---

## Mode Catalog — Named & Explained

| Address | Current Name | Proposed Name | Purpose |
|---------|-------------|---------------|---------|
| 0x415540 | `unit_mode_415540` | **`infantry_mode_idle_setup`** | Snap to tile-center, start idle anim, setup HP regen, transition → idle |
| (inline) | `unit_mode_idle` | **`infantry_mode_idle_tick`** | Idle tick: fidget timer, veteran HP regen, opportunity scan, escort check |
| (inline) | `sub_4157F0` | **`infantry_mode_fidget`** | Random fidget: play anim or turn ±32°, sleep 80, return → idle_setup |
| (inline) | `unit_4158B0` | **`infantry_escort_setup`** | Snap to center, set mode → escort_tick |
| 0x415980 | `unit_mode_415980` | **`infantry_mode_escort_tick`** | Check escort target alive + distance, move if far |
| 0x415A60 | `unit_mode_415A60` | **`infantry_mode_turning`** | Rotate toward `target_orientation`, then → `mode_turn_return` |
| 0x415B90 | `unit_mode_attack_move` | **`infantry_mode_pathfind_decision`** | Bresenham raycast → decide next move phase |
| 0x415D30 | `unit_mode_415D30` | **`infantry_mode_ray_stack_pop`** | Pop ray stack entry, decide: scan obstacles or move |
| 0x416060 | `unit_mode_path_around_obstacles` | **`infantry_mode_obstacle_scan`** | CW/CCW wall-following scan loop (4 iterations/tick) |
| 0x4165C0 | `unit_mode_4165C0` | **`infantry_mode_walk_setup`** | Orient + start move anim, set mode → walk_tick |
| 0x416790 | `unit_mode_416790` | **`infantry_mode_walk_tick`** | Main per-frame movement toward waypoint |
| 0x416A70 | `unit_mode_416A70` | **`infantry_mode_snap_fine_setup`** | Compute tile-center (512 threshold), begin centering |
| 0x416CD0 | `unit_mode_416CD0` | **`infantry_mode_snap_fine_tick`** | Per-frame move to tile center (512 tolerance) |
| 0x416EB0 | `unit_mode_416EB0` | **`infantry_mode_snap_coarse_setup`** | Compute tile-center (1792 threshold), begin centering |
| 0x417100 | `unit_mode_417100` | **`infantry_mode_snap_coarse_tick`** | Per-frame move to tile center (1792 tolerance) |
| 0x417360 | `unit_mode_417360` | **`infantry_mode_direct_walk`** | Simple line-walk to waypoint, handles queued orders on arrival |
| 0x417A20 | `unit_mode_417A20` | **`infantry_mode_scan_step`** | Path-scan decision: next obstacle-scan iteration or give up |
| 0x417BD0 | `unit_mode_417BD0` | **`infantry_mode_scan_walk_tick`** | Walk tick during obstacle-scan phase |
| 0x417E60 | `unit_417E60` | **`infantry_mode_arrival_fallback`** | Arrival/stuck handler: set mode → repath, track stuck counter |
| (merged) | `unit_mode_attack_move_2` | **`infantry_mode_repath_tick`** | Re-entry point after arrival_fallback sleep (same code body) |
| 0x417FC0 | `unit_mode_417FC0` | **`infantry_mode_blocked_nudge`** | Blocked by unit: nudge sideways, wait, or overtake slower unit |
| 0x4181B0 | `unit_mode_4181B0` | **`infantry_mode_blocked_tile_wait`** | Destination tile occupied: wait for clearance or overtake |
| 0x418550 | `unit_mode_418550` | **`infantry_mode_attack_cooldown`** | Projectile throttle countdown → attack |
| (inline) | `unit_mode_attack` | **`infantry_mode_attack_fire`** | Fire projectile, orient to target, handle disengage, reload |
| 0x4187F0 | `unit_mode_4187F0` | **`infantry_mode_enter_building`** | Walk off-grid to building rally point, → mode_arrive |
| 0x418B30 | `unit_mode_418B30_technician_repair` | **`infantry_mode_technician_walk`** | Technician walks to target → self-destructs, spawns repair task |
| 0x418D20 | `unit_mode_418D20` | **`infantry_mode_saboteur_walk`** | Saboteur walks to target → sends sabotage message, self-destructs |
| 0x418E90 | `unit_mode_418E90` | **`infantry_mode_repairbay_exit`** | Slide out of repair bay (100-tick countdown), resume normal AI |
| 0x418FE0 | `unit_mode_418FE0` | **`infantry_mode_repairbay_healing`** | Being healed inside bay, costs cash, → exit when full |
| 0x419180 | `unit_mode_419180` | **`infantry_mode_repairbay_enter`** | Sliding into repair bay (100-tick countdown), → healing |
| 0x419230 | `unit_mode_419230` | **`infantry_mode_repairbay_rotate`** | Rotate to face SW before entering bay, → enter |
| 0x419420 | `unit_mode_419420` | **`infantry_mode_center_walk`** | Walk to current tile center (for tanker alignment), → pathfind |
| 0x419560 | `unit_mode_419560` | **`infantry_mode_death_anim`** | Play death effects, sounds, corpse; wait 60 ticks → cleanup |
| 0x419720 | `unit_mode_419720` | **`infantry_mode_cleanup`** | Remove from grid, entity, task — TERMINAL |
| (shared) | `unit_mode_destroy` | **`infantry_mode_destroy_begin`** | Stop, mark destroyed, remove from grid, → destroyed_spin |
| (shared) | `unit_mode_destroyed` | **`infantry_mode_destroyed_spin`** | Spinning wreckage (vehicles only, infinite loop) |

---

## Mode Categories

### Idle & Passive
```
infantry_mode_idle_setup (415540)    — one-shot setup: snap, idle anim, HP regen init
infantry_mode_idle_tick (idle)       — tick loop: fidget, regen, opportunity scan
infantry_mode_fidget (4157F0)        — random look-around, sleep 80
```

### Escort System
```
infantry_escort_setup (4158B0)       — snap & transition to tick
infantry_mode_escort_tick (415980)   — distance check, re-follow if far
```

### Movement — Primary Path
```
infantry_mode_pathfind_decision (attack_move)  — raycast, branch by result
infantry_mode_walk_setup (4165C0)              — orient, start walk, → walk_tick
infantry_mode_walk_tick (416790)               — per-frame move, arrive → re-pathfind
infantry_mode_turning (415A60)                 — rotate, → mode_turn_return
infantry_mode_direct_walk (417360)             — no-pathfind straight-line walk
```

### Movement — Obstacle Avoidance
```
infantry_mode_ray_stack_pop (415D30)           — pop ray entry, decide next action
infantry_mode_obstacle_scan (416060)           — CW/CCW wall-follow, 4 iter/tick
infantry_mode_scan_step (417A20)               — scan phase decision node
infantry_mode_scan_walk_tick (417BD0)          — walk during scan phase
```

### Movement — Position Correction
```
infantry_mode_snap_fine_setup (416A70)         — compute center, threshold 512
infantry_mode_snap_fine_tick (416CD0)          — move to center, tolerance 512
infantry_mode_snap_coarse_setup (416EB0)       — compute center, threshold 1792
infantry_mode_snap_coarse_tick (417100)        — move to center, tolerance 1792
infantry_mode_center_walk (419420)             — walk to tile center (tanker align)
```

### Movement — Blocked States
```
infantry_mode_blocked_nudge (417FC0)           — nudge sideways, wait, overtake
infantry_mode_blocked_tile_wait (4181B0)       — wait for tile clearance
infantry_mode_arrival_fallback (417E60)        — stuck detection + sleep
infantry_mode_repath_tick (attack_move_2)      — re-entry after fallback sleep
```

### Combat
```
infantry_mode_attack_cooldown (418550)         — throttle countdown (max 200 projectiles global)
infantry_mode_attack_fire (attack)             — fire, reload, check range/disengage
```

### Special Actions
```
infantry_mode_enter_building (4187F0)          — walk to building rally → mode_arrive
infantry_mode_technician_walk (418B30)         — walk to target → self-destruct + spawn repair
infantry_mode_saboteur_walk (418D20)           — walk to target → sabotage + self-destruct
```

### Repair Bay Sequence
```
infantry_mode_repairbay_rotate (419230)        — face SW
infantry_mode_repairbay_enter (419180)         — slide in (100 ticks)
infantry_mode_repairbay_healing (418FE0)       — heal loop (costs cash)
infantry_mode_repairbay_exit (418E90)          — slide out (100 ticks)
```

### Death & Destruction
```
infantry_mode_death_anim (419560)              — death FX, sound, corpse, 60-tick wait
infantry_mode_cleanup (419720)                 — remove from world (TERMINAL)
infantry_mode_destroy_begin (destroy)          — mark destroyed, off grid, → spin
infantry_mode_destroyed_spin (destroyed)       — spinning wreckage (vehicles)
```

---

## Flowcharts

### Master State Machine
```
                              ┌──────────────────────────────────────┐
                              │         IDLE CLUSTER                  │
                              │                                      │
    spawn/arrive ──→ idle_setup ──→ idle_tick ←──→ fidget            │
                              │       │    │                         │
                              │       │    └─── escort_setup → escort_tick
                              └───────┼──────────────────────────────┘
                                      │ (order received / target found)
                                      ▼
                    ┌─────────────────────────────────────┐
                    │      PATHFINDING DECISION           │
                    │      (pathfind_decision)            │
                    │                                     │
                    │  Raycast → 6 outcomes:              │
                    │    ClearWithWaypoints ──→ walk_setup│
                    │    UnitObstacle ───────→ ray_pop    │
                    │    NoWaypoints ────────→ walk_setup │
                    │    TerrainObstacle ────→ ray_pop    │
                    │    ClearStraightLine ──→ direct_walk│
                    │    InvalidState ───────→ fallback   │
                    └──────────┬──────────────────────────┘
                               │
            ┌──────────────────┼──────────────────────┐
            ▼                  ▼                      ▼
    ┌──────────────┐   ┌──────────────┐      ┌─────────────┐
    │  walk_setup  │   │  ray_pop     │      │ direct_walk │
    │  (4165C0)    │   │  (415D30)    │      │ (417360)    │
    └──────┬───────┘   └──────┬───────┘      └──────┬──────┘
           │                  │                     │
           ▼                  ▼                     ▼
    ┌──────────────┐   ┌──────────────┐      mode_return
    │  walk_tick   │   │obstacle_scan │      or pathfind
    │  (416790)    │   │  (416060)    │
    └──────┬───────┘   └──────┬───────┘
           │                  │
           ├─ arrived → pathfind_decision (loop)
           ├─ blocked → blocked_nudge / blocked_tile_wait
           └─ turn needed → turning → walk_tick
```

### Movement Walk Loop
```
    walk_setup
        │
        ├─ infantry: set orientation directly → walk_tick
        └─ vehicle: → turning → walk_tick (via mode_turn_return)
              │
              ▼
    walk_tick (every frame)
        │
        ├─ within 768 of waypoint ──→ pathfind_decision (next segment)
        ├─ strayed > 0x10000 ───────→ pathfind_decision (recalculate)
        ├─ fully blocked ───────────→ blocked_nudge (wait/nudge)
        ├─ non-infantry wrong facing → turning (mode_turn_return = walk_tick)
        └─ normal: apply speed, advance position
```

### Obstacle Scan Loop
```
    ray_pop (pop ray stack)
        │
        ├─ stack empty + push_through + clear tile → walk_setup
        ├─ stack empty → arrival_fallback (give up)
        ├─ entry has clear LOS → walk_setup
        └─ entry blocked → obstacle_scan
                                │
    obstacle_scan (CW/CCW wall-follow, 4 iter/tick)
        │
        ├─ found clear tile → walk_setup (resume movement)
        ├─ circled back to self → snap_fine → arrival_fallback (disperse)
        ├─ iterations exhausted → ray_pop (try next entry)
        └─ 4 iterations done this tick → return (continue next frame)
                                │
    scan_step ←→ scan_walk_tick (alternate pair)
        │
        ├─ at best tile → snap_fine → idle_setup
        ├─ exhausted → pathfind_decision
        ├─ next step found → scan_walk_tick
        └─ fallback → arrival_fallback
```

### Blocked Resolution
```
    blocked_nudge (417FC0)
        │
        ├─ nudge succeeds (space clear) → mode_return (resume walking)
        ├─ passable / patience=0 → arrival_fallback (repath)
        └─ slower unit ahead → pathfind_decision (overtake)

    blocked_tile_wait (4181B0)
        │
        ├─ tile cleared → walk_setup
        ├─ passable now → arrival_fallback
        └─ slower unit → pathfind_decision (overtake)
```

### Combat Flow
```
    target_found (via opportunity scan or order)
        │
        ▼
    (unit_418290 — pre-attack setup, inline)
        │
        ├─ non-infantry facing wrong → turning (mode_turn_return = attack_fire)
        ├─ infantry at edge of tile → snap_coarse (mode_return = attack_fire)
        └─ ready → attack_cooldown
                    │
                    ▼ (countdown + projectile cap check)
    attack_fire
        │
        ├─ target lost/out of range → disengage (sub_4133D0)
        ├─ target is ally → disengage
        ├─ attack-move + far from dest → pathfind_decision
        └─ normal: fire projectile, yield reload_time, loop
```

### Repair Bay Sequence
```
    (order: dock at repair bay)
        │
        ▼
    pathfind_decision → ... → walk → arrival
        │
        ▼
    repairbay_rotate (face SW)
        │
        ▼
    repairbay_enter (slide in, 100 ticks, speed 64,-64)
        │
        ▼
    repairbay_healing (heal loop, deduct cash)
        │
        ├─ bay destroyed → death_anim
        └─ full HP → repairbay_exit (slide out, 100 ticks, speed -64,64)
                        │
                        ├─ had Move order → pathfind_decision
                        └─ otherwise → snap_fine → idle_setup
```

### Death Sequence
```
    (hitpoints ≤ 0)
        │
        ▼
    death_anim (play FX, sound, corpse)
        │ (sleep 60)
        ▼
    cleanup (BOXD_remove, entity_remove, TASK_Terminate, free unit slot)
        │
        ▼
    [REMOVED FROM WORLD]
```

---

## Key Conventions

### Mode Slots Pattern
```c
unit->mode              // current active state (called every frame)
unit->mode_return       // "where to go after this finishes" (most alignment/snap modes use this)
unit->mode_turn_return  // "where to go after turning completes" (turning mode reads this)
unit->mode_arrive       // "what to do on arrival at building" (enter_building sets this)
```

### Infantry vs Vehicle Differences (in shared modes)
- **Infantry** skip turning — orient instantly in `walk_setup`
- **Infantry** use `BOXD_adjust_unit_position_x/y` for 5-per-tile sub-positions
- **Infantry** have HP regen when veteran (idle only)
- **Infantry** have fidget animations in idle
- **Vehicles** must enter `turning` mode before walking (non-instant rotation)
- **Vehicles** use `unit_mode_destroyed_spin` for wreckage; infantry use `death_anim` → `cleanup`

### Stuck Detection (arrival_fallback / unit_417E60)
- Tracks `last_stuck_tile_x/y` + `stuck_timer`
- If same tile repeatedly: sleep grows (16*stuck_timer, max 240)
- After sleep: infantry do `snap_fine → repath`; vehicles just repath directly

### Disperse Timer / Push-Through Timer
- `disperse_timer`: when >0, partially-occupied tiles are treated as BLOCKED (avoid congestion)
- `push_through_timer`: when >0, partially-occupied tiles are treated as CLEAR (force through)
- Both decrement in `pathfind_decision` each frame

### Global Projectile Cap
- Max 200 projectiles active globally
- `attack_cooldown` waits if cap reached (resets counter to 10 each time)

---

## UNIT_Handler_Infantry Frame Loop

```c
void UNIT_Handler_Infantry(Task *task) {
    unit = task->ctx;
    if (!unit) { unit = unit_create(task); init; }

    unit->mode(unit);                    // execute current mode

    // bounds check — destroy if off-map
    if (entity out of map bounds)
        unit->mode = unit_mode_destroy;

    // staggered opportunity scan (1/64 frames per unit)
    if ((unit_id ^ g_opportunity_timer) & 0x3F == 0)
        unit_scan_for_target_of_opportunity(unit);

    // track order target position
    if (order_target valid)
        unit->_u1 = order_target->entity->position;
}
```

---

## Source File Mapping

All infantry modes come from `C:\k\Scripts\Infantry.cpp`. The decompiled addresses map to a single compilation unit. Helper functions (`sub_4157F0`, `unit_4158B0`, `unit_417E60`, `unit_418290`) are inlined or small helpers within the same file.

Shared utility modes (`unit_mode_destroy`, `unit_mode_destroyed`) are likely from a common include but linked into the same address range.

---

## Naming Convention Proposal

Pattern: `infantry_mode_{category}_{action}`

Categories:
- `idle` — passive states
- `move` — movement execution
- `path` — pathfinding decisions
- `scan` — obstacle scanning
- `snap` — position correction/alignment
- `block` — blocked/waiting states
- `attack` — combat
- `repair` — repair bay sequence
- `enter` — entering buildings
- `death` — destruction sequence
- `special` — technician/saboteur one-shots

Examples:
- `infantry_mode_idle_tick`
- `infantry_mode_move_walk_tick`
- `infantry_mode_path_decision`
- `infantry_mode_scan_obstacle_cw_ccw`
- `infantry_mode_snap_to_center_fine`
- `infantry_mode_block_nudge_sideways`
- `infantry_mode_attack_fire`
- `infantry_mode_repair_healing`
- `infantry_mode_death_anim`
