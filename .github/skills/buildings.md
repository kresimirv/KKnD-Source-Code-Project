# Building Mode Workflow — Complete Reference

## Overview

Buildings in KKND use the same mode-function-pointer system as units. Each building has a `UNIT_Handler_*` that calls `unit->mode(unit)` every tick. Buildings fall into distinct categories based on their init pattern and role.

Original source: `C:\k\Scripts\Building.cpp` (confirmed via embedded debug strings).

---

## Building Categories

| Category | Examples | Init Pattern | Has Production | Key Trait |
|----------|----------|--------------|----------------|-----------|
| **Factory** | Outpost, Clanhall, MachineShop, BeastEnclosure, Blacksmith | `unit_building_init` + `unit_402BB0` | Yes | Upgrade levels, production sidebar |
| **Infrastructure** | PowerStation, RepairBay, ResearchLab | `unit_building_init` (no callbacks) | No | Passive service buildings |
| **Defense** | Tower, MachinegunNest, GrapeshotTower, RotaryCannon | Custom init (no `unit_building_init`) | No | Turret-only, no garrison |
| **Resource** | Drillrig, OilPatch | Custom per-type | No | Economy backbone |
| **Neutral/Mission** | Prison, TechBunker, Hut | Custom | No | Scripted triggers |
| **Deployer** | MobileBase, MobileDerrick | Vehicle-style + deploy | No | Transforms into building |

---

## Shared Building Modes (0x402AB0–0x403720)

| Address | Name | Purpose |
|---------|------|---------|
| 0x402AB0 | `building_mode_under_construction` | No-op sentinel — building is being built (identity-check mode) |
| 0x4034B0 | `building_mode_grid_snap_ai` | Snap to grid, register footprint, → operational idle |
| 0x403540 | `building_mode_placement_validate` | Validate placement (radius check), claim for player, → mode_arrive |
| 0x4035C0 | `building_mode_grid_snap_player` | Snap to grid, register footprint, → placement_validate |
| 0x403650 | `building_mode_operational_idle` | Main alive-and-running loop: play idle anim, yield 1 tick |
| 0x403720 | `building_mode_finalize` | Play completion sound, teardown grid, remove entity, terminate task |
| 0x403780 | `building_mode_on_death` | Explode, shake camera, cleanup production/turret, → finalize |

---

## Construction Sequence (3-Stage Animation)

### Setup
```c
unit_402BB0(unit, arrive_mode):
    // Start construction animation frame 0
    entity_anim_start(entity, mobd_lookup_offset_attack, 0);
    // Notify game loop of construction progress
    TASK_send_message(task, TaskMessage_AdvanceConstructionStage, unit, g_game_update_loop_task);
    // Extract grid anchor from MOBD animation data
    // Register building footprint on collision grid
    unit_build_40DD00(unit);
    // Enter construction wait mode
    unit->mode = building_mode_under_construction;  // (0x402AB0)
```

### Cost Deduction Loop (runs in game loop task)
```
Each tick:
    remaining_cost -= cost_per_tick
    ratio = (remaining_cost << 8) / total_cost

    if ratio crosses 171 (67% done) → send Stage 1
    if ratio crosses 85 (33% done)  → send Stage 2
    if remaining_cost ≤ 0           → send Stage Complete
```

`cost_per_tick = (cost << 8) / (60 * build_time)` — 60 ticks/second.

### Stage Handler (in `MessageHandler_BuildingDefault`)
```c
if (mode == building_mode_under_construction && msg == AdvanceConstructionStage):
    Stage 1: play construction anim frame 1
    Stage 2: play construction anim frame 2
    Stage Complete: unit->mode = unit->mode_arrive  // → completion mode
```

### Flowchart
```
unit_402BB0 (place building)
    │
    ▼
building_mode_under_construction (402AB0) — no-op, await stages
    │ (game loop deducts cost)
    ├── Stage 1 → anim frame 1 (33% scaffolding)
    ├── Stage 2 → anim frame 2 (67% structure)
    └── Complete → unit->mode = mode_arrive
                    │
                    ▼
            <completion_mode> (per-building type)
                    │
                    ▼
            building_mode_operational_idle (403650)
```

---

## Operational Idle Modes

### `building_mode_operational_idle` (0x403650) — Main Loop
```c
void unit_mode_403650(Unit *unit) {
    if (channel == TaskChannel_Drillrig)
        unit_410B00_drillrig(unit);     // update oil level bar

    if (player_num != 0) {              // AI buildings
        if (hp >= max_hp/3 || no damaged sprite)
            play idle anim;
        else
            play damaged anim (mobd_lookup_offset_4);
    } else {                            // Player 0 buildings
        play idle anim;
        unit->mode = building_mode_grid_snap_player;  // → snap → validate → finalize placement
    }

    decay cplc_spawn_param (attack alert cooldown);
    TASK_yield(task, Task_Sleep, 1);
}
```

**Player 0 chain:** `403650` → `4035C0` (snap) → `403540` (validate) → `mode_arrive`
**AI chain:** stays in `403650` permanently, shows damaged sprite when HP < 33%

### `building_mode_grid_snap_ai` (0x4034B0) — Pre-placed Building Init
```c
void unit_mode_4034B0(Unit *unit) {
    entity->is_on_collision_grid = 1;
    entity->x = tile_align(x, grid_anchor) + 4096;
    entity->y = tile_align(y, grid_anchor) + 4096;
    unit_build_40DD00(unit);            // register footprint
    unit->mode = building_mode_operational_idle;
    TASK_yield(task, Sleep, 1);
}
```

Used by CPLC-spawned (pre-placed) buildings to skip construction.

---

## Death Sequence

### `building_mode_on_death` (0x403780)
```c
void unit_mode_building_on_death(Unit *unit) {
    unit_explode_1(unit);              // explosion entity
    GAME_ShakeCamera();

    // If player-owned: destroy production, decrement tracker
    if (player_num == g_player_num) {
        sub_446860(state->prod);       // teardown sidebar
        buildings_tracker_decrement(unit);
    }

    unit->destroyed = 1;
    unit->state = nullptr;

    // Remove turret
    if (turret) {
        entity_remove(turret->entity);
        task_terminate(turret->task);
    }

    // Broadcast deselect
    TASK_send_message(task, TaskMessage_UnitDeselected, ...);
    TASK_broadcast_message(task, TaskMessage_UnitDeselected, TaskChannel_UnitLifecycle);

    // Notify dependents
    if (channel == TaskChannel_PowerPlant)
        broadcast TaskMessage_PowerPlantDown to TaskChannel_Tanker;
    if (channel == TaskChannel_Drillrig)
        broadcast TaskMessage_DrillrigDown to TaskChannel_Tanker;

    task->channel = TaskChannel_None;
    unit->unit_id = 0;

    unit_explode_2(unit);              // more VFX
    unit_438D90(unit);                 // rubble entity

    unit->mode = building_mode_finalize;
    TASK_yield(task, Task_Sleep, 165); // rubble visible for 165 ticks
}
```

### `building_mode_finalize` (0x403720)
```c
void unit_mode_building_finalize(Unit *unit) {
    SOUND_play(building_destroyed_sfx);
    unit_build_40DDD0(unit);           // remove from collision grid
    entity_remove(entity);
    TASK_Terminate(task);
    unit_44C890(unit);                 // free unit slot
}
```

---

## Factory Buildings — Common Pattern

### Init Template
```c
void UNIT_Handler_Factory(Task *task) {
    unit = unit_create(task);
    task->message_handler = MessageHandler_<Building>;
    unit_building_init(unit, garrison_strength, downgrade_fn, on_completed_fn);

    // Optional: set default production, register base, reveal fog

    if (!cplc_spawn_params)            // Fresh placement
        unit_402BB0(unit, completion_mode);  // → construction
    else                               // Pre-placed
        unit->mode = completion_mode;

    unit->mode(unit);
}
```

### Completion Mode Template
```c
void completion_mode(Unit *unit) {
    unit->mode_arrive = nullptr;
    <building>_on_completed(unit);     // enable production sidebar
    show_turret();
    play_sound(construction_complete);
    task->channel = <TaskChannel>;

    if (cplc_spawn_params)
        unit->mode = building_mode_grid_snap_ai;   // pre-placed
    else
        unit->mode = building_mode_operational_idle; // player built

    building_mode_operational_idle(unit);
}
```

### on_completed Template
```c
void on_completed(Unit *unit) {
    if (player_num == g_player_num) {
        show_message("Building completed");
        prod = unit_enable_production(unit, ProductionType);
        state->prod = prod;
        // Enable units based on tech level
        production_enable_unit(prod, UnitType_X, mobd_icon);
        ++tracker.num_buildings_by_level[1];
    } else if (player_num == 0) {
        unit->mode_arrive = on_completed;  // defer
    }
}
```

### Upgrade Handler (in MessageHandler)
```c
case TaskMessage_UpgradeComplete:
    state->upgrade_level++;
    ++tracker.num_buildings_by_level[upgrade_level];
    if (upgrade_level > tracker.max_level)
        tracker.max_level = upgrade_level;
    // Unlock new units for this tier
    production_enable_unit(prod, UnitType_NewUnit, icon);
```

### Downgrade/Death Template
```c
void downgrade_production(Unit *unit) {
    --tracker.num_buildings_by_level[state->upgrade_level];
    // Recalculate max_level walking backwards
    sub_446860(state->prod);  // destroy sidebar
    state->prod = nullptr;
}

void on_death(Unit *unit) {
    downgrade_production(unit);
    unit_mode_building_on_death(unit);
}
```

---

## Factory Buildings — Specific Details

### Survivors

| Building | Channel | Garrison | Production | Upgrades |
|----------|---------|----------|------------|----------|
| **Outpost** | Outpost | 3 | Infantry | 5 levels — Rifleman → Pyromaniac → Vandal/Rioter → Bazooka → CrazyHarry |
| **MachineShop** | MachineShop | 2 | Vehicles | 4 levels — DirkBike/4x4 → ATV → MobileDerrick → Tanker |

### Evolved

| Building | Channel | Garrison | Production | Upgrades |
|----------|---------|----------|------------|----------|
| **Clanhall** | Clanhall | 3 | Infantry | 5 levels — Berserker → Pyromaniac → Vandal/Rioter → Bazooka → CrazyHarry |
| **BeastEnclosure** | BeastEnclosure | 2 | Vehicles | 4 levels — DireWolf → WarMastadont → GiantBeetle → MissileCrab |
| **Blacksmith** | Blacksmith | 2 | Vehicles | 3 levels — FlameATV → MonsterTruck → Tanker |

### Master Buildings (Outpost / Clanhall)

These control the ENTIRE tech tree:
- Own `g_sidebar_buildings_prod` (building construction sidebar)
- Own `g_sidebar_tower_prod` (defensive tower sidebar)
- Own `g_sidebar_aircraft_prod` (air unit sidebar)
- Control minimap visibility
- Register in `g_player_bases[]` (aircraft spawn anchor)
- On death: disable towers, aircraft, buildings sidebar, minimap

---

## Infrastructure Buildings

### Power Station
| Field | Value |
|-------|-------|
| Channel | `TaskChannel_PowerPlant` |
| Garrison | 2 |
| Production | None |
| Special | Spawns tanker on completion; notifies tankers on death via `TaskMessage_PowerPlantDown` |

### Repair Bay
| Field | Value |
|-------|-------|
| Channel | `TaskChannel_RepairBay` |
| Garrison | 1 |
| Production | None |
| Special | Units dock here for healing (see infantry.md repair bay sequence) |

### Research Lab
| Field | Value |
|-------|-------|
| Channel | `TaskChannel_ResearchLab` |
| Garrison | 1 |
| Production | Hidden (research queue in state block) |
| Special | Deferred message handler swap on completion |

---

## Defensive Towers

**NO `unit_building_init`** — completely different init pattern.

| Mode | Purpose |
|------|---------|
| `tower_mode_on_completed` | Sets debug state |
| `tower_mode_advance_construction` | Animation stage stepping |
| `tower_mode_on_death` | Explosion + → finalize |
| `tower_mode_finalize` | Remove from grid, cleanup (60-tick delay) |

Init:
- `unit_build_40DD00` for grid registration
- `tower_turret_init()` for turret
- Increments `g_num_towers`
- Progressive construction via `TaskMessage_AdvanceConstructionStage` to game loop

---

## Neutral/Mission Buildings

### Prison
| Field | Value |
|-------|-------|
| Garrison | 5 (highest!) |
| Channel | None |
| Special | On death: spawns prisoner units (level-specific behavior) |
| Modes | `prison_407300` (idle), `prison_release_prisoners` (spawn loop) |
| Surv_09 | Releases General unit |
| Xtreme | Releases 10 prisoners from hardcoded array |

### Tech Bunker
| Field | Value |
|-------|-------|
| Init | Custom (no `unit_building_init`) |
| Channel | None |
| Special | Proximity-triggered treasure chest |
| Modes | `tech_bunker_407950` (proximity scan), `tech_bunker_407690` (dispense reward) |
| Rewards | Units (El Presidente at param=10) or cash (1000–5000 RU) |
| Cooldown | 28800–54000 ticks (single-player: 5 ticks) |
| Detection | `unit_find_any_in_radius()` with radius from `multi_purpose_field_1` |

### Hut
| Field | Value |
|-------|-------|
| Init | Custom (no `unit_building_init`) |
| Channel | None |
| Special | Decorative destructible prop |
| Modes | `hut_init` → `hut_407E70` (align) → `hut_407DA0` (idle anim loop) |
| Config | `cplc_spawn_param` 0–4 controls facing direction |

### Oil Patch
| Field | Value |
|-------|-------|
| Init | Custom — NOT a unit (entity only) |
| Channel | None |
| Special | Terrain resource marker; amount = 500 × spawn_param |
| Handler | Enters infinite sleep loop (`TASK_yield(task, Sleep, 1000)` forever) |
| Structure | `KKND::OilPatch` with linked list |

---

## Drillrig (Building Form)

| Address | Mode | Purpose |
|---------|------|---------|
| 0x4081C0 | `drillrig_mode_drill_anim` | Deploy animation (hide turret → reveal) |
| 0x408240 | `drillrig_mode_skip_anim` | Fast deploy from CPLC (no animation) |
| 0x408260 | `drillrig_mode_finalize_deploy` | Register on oil patch, enable pumping |
| 0x408330 | `drillrig_mode_on_death` | Notify tankers → building death |

```
Fresh placement: drill_anim → finalize_deploy → building_mode_operational_idle
Pre-placed:     skip_anim  → finalize_deploy → building_mode_grid_snap_ai
Death:          on_death → broadcasts DrillrigDown → building_mode_on_death
```

Special: `unit_410B00_drillrig` called every tick from `building_mode_operational_idle` — renders oil level status bar (59-pixel bar, proportional to remaining oil / 100000).

---

## Aircraft (Bomber)

Not a building, but uses a unique handler (`UNIT_Handler_Bomber`). Source: `Aircraft.cpp`.

| Mode | Purpose |
|------|---------|
| `bomber_401800` | Fly toward waypoint (main flight) |
| `bomber_4016B0` | Bomb run (drop projectiles, 15-tick cooldown) |
| `bomber_4017E0` | Loiter (sleep 40 ticks) |
| `bomber_401680` | Fly offscreen (set waypoint to map edge) |
| `bomber_on_death` | Crash sequence |

Spawn: random map edge, altitude 46080, fly to target.

---

## Master Flowchart — Building Lifecycle

```
┌──────────────── CONSTRUCTION ────────────────────────────────┐
│                                                              │
│  unit_402BB0 (place)                                         │
│      │                                                       │
│      ▼                                                       │
│  building_mode_under_construction (402AB0)  ← no-op wait     │
│      │                                                       │
│      │ (game loop deducts cost_per_tick)                     │
│      │                                                       │
│      ├── AdvanceStage(1) → anim frame 1                      │
│      ├── AdvanceStage(2) → anim frame 2                      │
│      └── AdvanceStage(Complete) → unit->mode = mode_arrive   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────── COMPLETION ──────────────────────────────────┐
│                                                              │
│  <completion_mode> (per building type)                       │
│      │                                                       │
│      ├── Enable production sidebar (factory buildings)       │
│      ├── Show turret                                         │
│      ├── Play "construction complete" sound                  │
│      ├── Set task channel                                    │
│      └── Transition → building_mode_operational_idle         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────── OPERATIONAL ─────────────────────────────────┐
│                                                              │
│  building_mode_operational_idle (403650)                     │
│      │                                                       │
│      ├── Drillrig: update oil level bar                      │
│      ├── AI: show damaged sprite if HP < 33%                 │
│      ├── Player 0: → grid_snap_player → validate → arrive   │
│      └── yield 1 tick (loop)                                 │
│                                                              │
│  UPGRADES (via message handler):                             │
│      UpgradeComplete → unlock new units at new tier          │
│                                                              │
│  PRODUCTION (factory buildings):                             │
│      UnitReady → spawn unit at rally point                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼ (hitpoints ≤ 0 or sabotage)
┌──────────────── DEATH ───────────────────────────────────────┐
│                                                              │
│  <on_death> (per building type)                              │
│      │                                                       │
│      ├── downgrade_production (remove sidebar, decrement)    │
│      └── building_mode_on_death (403780):                    │
│              explode, shake camera                            │
│              destroy production/turret                        │
│              broadcast PowerPlantDown/DrillrigDown            │
│              play rubble anim                                 │
│              │                                               │
│              ▼ (sleep 165 ticks — rubble visible)            │
│          building_mode_finalize (403720):                     │
│              teardown grid (unit_build_40DDD0)                │
│              entity_remove, TASK_Terminate                    │
│              free unit slot                                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Sabotage System

Saboteurs reduce `garrison_strength`. When it reaches 0, the building is captured/destroyed:

```c
void unit_on_sabotage(unit, saboteur_info, on_death_mode) {
    if (saboteur is same team)
        garrison_strength++ (repair, max 5);
    else {
        garrison_strength--;
        if (garrison_strength == 0)
            g_cash[saboteur_player] += building cost;  // refund to attacker
            unit->mode = on_death_mode;                // destroy building
    }
}
```

Building sabotage bar updated via `unit_building_update_sabotage_bar`.

---

## `unit_building_init` — State Allocation

```c
void unit_building_init(Unit *unit, garrison, downgrade_fn, on_completed_fn) {
    state = TASK_Alloc(task, 0x28);  // BuildingState (40 bytes)
    state->ctx = NULL;
    state->upgrade_level = 1;
    state->upgrade_timer = 0;
    state->garrison_strength = garrison;
    state->same_building_count = 0;
    state->prod = NULL;

    if (player == g_player_num)
        buildings_tracker_increment(type);

    // Mission-specific HP reduction (Mute_09, Surv_21)
    if (player==0 && special_level)
        unit->hitpoints = max_hp / 5 - 1;

    if (stats->attachment)
        unit_attach_turret(unit);

    unit_building_healthbar_init(unit);
    unit->mode_turn_return = downgrade_fn;
    unit->mode_return = on_completed_fn;
}
```

**Mode slot reuse for buildings:**
- `mode_turn_return` = downgrade/teardown function
- `mode_return` = on_completed function
- `mode_arrive` = completion transition target (set by `unit_402BB0`)

---

## `unit_build_40DD00` — Footprint Registration

Registers building on collision grid using `BuildingBlueprint`:
```c
int unit_build_40DD00(Unit *unit) {
    blueprint = g_building_blueprints[unit->type];
    origin_x = (grid_anchor_x + entity->x) >> 13;
    origin_y = (grid_anchor_y + entity->y) >> 13;

    mask = 0x80000000;
    for (y = 0; y < height; y++)
        for (x = 0; x < width; x++) {
            if (collision_mask & mask)
                BOXD_place_unit(unit, x, y, Slot0);      // solid
            else
                BOXD_place_unit(unit, x, y, Position_40); // overlay
            mask >>= 1;
        }
}
```

---

## Naming Convention Proposal

Pattern: `building_mode_{type}_{action}`

| Current Name | Proposed Name |
|-------------|---------------|
| `unit_mode_402AB0` | `building_mode_under_construction` |
| `unit_mode_4034B0` | `building_mode_grid_snap_ai` |
| `unit_mode_403540` | `building_mode_placement_validate` |
| `unit_mode_4035C0` | `building_mode_grid_snap_player` |
| `unit_mode_403650` | `building_mode_operational_idle` |
| `unit_mode_building_finalize` | `building_mode_cleanup_remove` |
| `unit_mode_building_on_death` | `building_mode_death_explode` |
| `unit_mode_beast_enclosure_402440` | `building_mode_beast_enclosure_completed` |
| `unit_mode_blacksmith_402870` | `building_mode_blacksmith_completed` |
| `unit_mode_clanhall_4042A0` | `building_mode_clanhall_completed` |
| `unit_mode_outpost_431680` | `building_mode_outpost_completed` |
| `unit_mode_machine_shop_*` | `building_mode_machineshop_completed` |
| `unit_mode_power_station_*` | `building_mode_powerstation_spawn_tanker` |
| `unit_mode_repair_building_437F30` | `building_mode_repairbay_completed` |
| `unit_mode_tech_bunker_407950` | `building_mode_techbunker_proximity_scan` |
| `unit_mode_tech_bunker_407690` | `building_mode_techbunker_dispense_reward` |
| `unit_mode_prison_release_prisoners` | `building_mode_prison_spawn_units` |

---

## Decompilation Artifacts

1. **`unit_mode_402AB0` writes sentinel** — not a bug. It's an identity-mode; the MODE POINTER ITSELF is the state (checked via `unit->mode == unit_mode_402AB0`). The sentinel write to `dword_477358` is likely a debug trace left by devs.

2. **`unit_mode_403650` player-0 chain** — looks like dead code path for EDITOR/development placement flow. In shipping game, buildings always go through `unit_402BB0` construction first. The `4035C0` → `403540` chain may only trigger for scenario editor pre-placed buildings.

3. **Mode slot reuse** — `mode_turn_return`, `mode_return`, `mode_arrive` have DIFFERENT meanings for buildings vs units:
   - Units: turning callback, return-after-action, arrival-at-destination
   - Buildings: downgrade function, on-completed function, construction-done transition

4. **`dword_477358`** — global debug trace variable written by multiple sentinel modes (`402AB0` writes 76354, `405680` writes 2371645, `4446A0` writes 901223). All three are "do nothing but mark we were here" modes.

5. **Garrison strength** — reused as sabotage hit-counter. Higher value = more saboteur hits needed to destroy. Not related to garrisoned units despite the name.
