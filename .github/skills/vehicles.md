# Vehicle Mode Workflow — Complete Reference

## Critical Discovery: Vehicles Share the Infantry Mode System

`UNIT_Handler_Vehicle` is literally:
```c
void __cdecl UNIT_Handler_Vehicle(KKND::Task *task)
{
    UNIT_Handler_Infantry(task);
}
```

**Vehicles and infantry use the EXACT same handler and mode functions (0x415540–0x419720).** The difference is purely in runtime branches checking `unit->stats->is_infantry` and `unit->stats->size` within shared mode functions.

---

## How Vehicles Differ from Infantry (in Shared Modes)

| Behavior | Infantry | Vehicle |
|----------|----------|---------|
| **Rotation** | Instant orientation change in `walk_setup` | Must enter `turning` mode (0x415A60) before walking |
| **Tile positioning** | 5 sub-positions per tile via `BOXD_adjust_unit_position_x/y` | Fixed center: 4096 (Regular) or 7424 (XL/Large) |
| **Idle fidget** | Random look-around / fidget anim after 50 ticks | None — vehicles just idle silently |
| **HP regen** | Veterans regenerate HP while idle | Never — vehicles don't regen |
| **ClearStraightLine path** | Never used — infantry always uses waypoint pathing | Uses `sub_417550` Bresenham direct-path scanner |
| **Walk proximity** | No early-arrival flag | Sets `path_flags |= 1` when < 1792 units from waypoint |
| **Orientation steering** | `unit_40D600` (infantry variant) | `unit_40D6F0` (vehicle variant — smoother curves) |
| **XL raycast** | N/A (infantry is always Small) | `BOXD_41C130` fires 3 parallel rays for wide vehicles |
| **Death** | `death_anim` → `cleanup` (fade out corpse) | `destroy_begin` → `destroyed_spin` (spinning wreckage forever) |
| **Walk setup turn** | Orient instantly, start walking | Need `unit_mode_415A60` (turning), `mode_turn_return = walk_tick` |

---

## Vehicle-Exclusive Modes

### Direct Path Scanner (Vehicle-Only)

When `unit_mode_attack_move` gets `BoxdRaycastResult_ClearStraightLine`:
- Infantry: uses `sub_417550` → NO — actually infantry never reaches this case
- Vehicle: enters the Bresenham scanner chain

```
sub_417550 (setup)              — compute initial distance, set scan heads
    ↓
sub_417670 (scan loop)          — advance CW/CCW probes, 4 iterations/tick
    ↓ (scan complete or wrapped)
sub_417810 (crawl setup)        — compute waypoint, set mode → scan_walk_tick
    ↓
unit_mode_417A20 (scan step)   ← → unit_mode_417BD0 (scan walk tick)
```

**`sub_417550`** — Bresenham Direct-Path Setup:
- Computes Manhattan distance from current tile to target tile
- Sets scan budget = 2× distance
- Initializes dual scan heads (CW and CCW offset by ±2 directions)
- Transitions → `sub_417670`

**`sub_417670`** — Dual-Direction Scanner (4 iter/tick):
- Advances two scan positions using tile stepper (`sub_44D250`)
- Tracks best intermediate waypoint (closest to target with distance improvement > half-steps)
- Terminates when budget exhausted or scan wrapped back to start
- Calls `sub_417810` on completion

**`sub_417810`** — Crawl Setup:
- If already at destination: → `unit_mode_416A70` (arrival)
- Otherwise: computes pixel waypoint, stores step budget, sets `mode = unit_mode_417BD0`
- Calls `unit_mode_417A20` for first iteration

**`sub_417980`** — Resume Crawl (after interruption):
- Same as `sub_417810` but skips destination check
- Used when restarting after obstacle avoidance

---

## Tanker Docking System (0x4444D0–0x444B40)

Solo tankers (economy units) shuttle oil between Drillrig and Power Station.

### Mode Catalog

| Address | Name | Purpose |
|---------|------|---------|
| 0x4444D0 | `tanker_mode_goto_drillrig_init` | Trampoline: → navigate to drillrig, sleep 5 |
| 0x4444F0 | `tanker_mode_find_drillrig` | Broadcast on Drillrig channel, find/assign drillrig |
| 0x444590 | `tanker_mode_find_powerstation` | Broadcast on PowerPlant channel, find/assign target |
| 0x444630 | `tanker_mode_rotate_at_drillrig` | Rotate to SW at drillrig dock |
| 0x4446A0 | `tanker_mode_oil_exhausted` | Sentinel: oil patch ran dry mid-load |
| 0x4446B0 | `tanker_mode_load_oil` | Load 1 oil/tick (max 500), low-oil warnings |
| 0x4447C0 | `tanker_mode_goto_powerstation` | Clear drillrig slot, move to powerstation |
| 0x4448C0 | `tanker_mode_goto_drillrig` | Validate drillrig, move to it |
| 0x4449D0 | `tanker_mode_rotate_at_powerstation` | Rotate to SW at powerstation dock |
| 0x444A40 | `tanker_mode_reverse_into_dock` | Back up into powerstation (80 ticks) |
| 0x444AB0 | `tanker_mode_drive_out_of_dock` | Drive forward out (80 ticks), → goto drillrig |
| 0x444B40 | `tanker_mode_unload_oil` | Unload 20 oil/tick → cash, sleep 20/batch |

### Constants

| Parameter | Value |
|-----------|-------|
| Tanker capacity | **500 oil** |
| Load rate | **1 oil/tick** |
| Unload rate | **20 oil per batch** |
| Cash per batch | **20** |
| Dock/undock time | **80 ticks** |
| Docking orientation | **SW (MobdOrientation_SW)** |
| Drillrig anchor offset | (-7168, +11520) default |
| Powerstation anchor offset | (-8192, +8192) default |
| Low-oil warning threshold | 5000, then every 500 below 2500 |

### Tanker Lifecycle Flowchart

```
              ┌─────── LOADING SIDE ────────┐     ┌────── UNLOADING SIDE ──────┐
              │                              │     │                            │
spawn ───→ find_drillrig (4444F0)            │     │  find_powerstation (444590)│
              │                              │     │         │                  │
              ▼                              │     │         ▼                  │
        goto_drillrig (4448C0)               │     │  goto_powerstation (4447C0)│
              │ (arrive)                     │     │         │ (arrive)         │
              ▼                              │     │         ▼                  │
        rotate_at_drillrig (444630)          │     │  rotate_at_powerstation    │
              │ (facing SW)                  │     │  (4449D0)                  │
              ▼                              │     │         │ (facing SW)      │
        load_oil (4446B0)                    │     │         ▼                  │
              │ 1 oil/tick                   │     │  reverse_into_dock (444A40)│
              │                              │     │         │ (80 ticks)       │
              ├─ full (500) ─────────────────┼─────┼────→    ▼                  │
              │                              │     │  unload_oil (444B40)       │
              └─ oil exhausted → sentinel    │     │         │ 20 oil/batch     │
                  (4446A0)                   │     │         │                  │
                                             │     │         ▼ (empty)          │
              ┌──────────────────────────────┼─────┼── drive_out_of_dock       │
              │                              │     │  (444Ab0, 80 ticks)        │
              ▼                              │     │                            │
        goto_drillrig (4448C0) ◄─────────────┘     └────────────────────────────┘
              │
              └─── CYCLE REPEATS ───
```

### Dock Slot Contention

Before docking, tanker enters `unit_mode_tanker_419390` (wait-for-slot):
- Checks building `state[6]` (docked unit reference)
- If empty or dead → claims slot, proceeds to dock
- If occupied by valid unit → sleeps 30, retries
- Ensures only ONE tanker docks at a building at a time

---

## Tanker Convoy System (Mission-Specific)

Used in campaign missions (Surv07 "Protect the Convoy", Mute08 "Smash the Convoy").

### Mode Catalog

| Address | Name | Purpose |
|---------|------|---------|
| 0x405680 | `convoy_mode_parked` | Terminal: tanker parked/escaped. Sets sentinel. |
| 0x405690 | `convoy_mode_countdown` | Count down timer, then remove from grid → parked |
| (inline) | `unit_405750_tanker_convoy` | Advance checkpoint or exit map |

### Handler: `UNIT_Handler_TankerConvoy`

- Creates unit with `TaskChannel_Tanker`, `MessageHandler_TankerConvoy`
- Initializes from CPLC spawn params (checkpoint index)
- Orientation: West
- Order: `UnitOrder_TankerConvoyNextCheckpoint`
- Each tick: dispatches mode + decrements `multi_purpose_field_3`

### Convoy Lifecycle

```
spawn → unit_mode_attack_move → [arrive at checkpoint]
    │
    ▼
unit_4054D0 → unit_405750_tanker_convoy
    │
    ├─ more checkpoints → update waypoint, unit_mode_attack_move (loop)
    │   (Surv07: 18 waypoints, Mute08: 15 waypoints)
    │
    └─ last checkpoint → exit sequence:
        set mode = convoy_mode_countdown (405690)
        orientation = West, speed = (-64, 0)
        multi_purpose_field_3 = 600 (countdown)
        increment g_num_convoy_tankers_en_route
        trigger mission victory/progress check
            │
            ▼ (600 ticks)
        convoy_mode_parked (405680) — TERMINAL
```

### Solo vs Convoy Comparison

| Aspect | Solo Tanker | Convoy Tanker |
|--------|-------------|---------------|
| Handler | `UNIT_Handler_Vehicle` → Infantry | `UNIT_Handler_TankerConvoy` (custom) |
| Order | `UnitOrder_TankerDock` | `UnitOrder_TankerConvoyNextCheckpoint` |
| Movement | Attack-move to building anchors | Attack-move through scripted waypoint array |
| Purpose | Economy (oil shuttle loop) | Mission objective (escort/destroy) |
| End state | Loops forever | Drives offscreen, parks |
| Arrival | `unit_mode_tanker_419390` (dock slot) | `unit_405750_tanker_convoy` (next checkpoint) |

---

## Mobile Derrick Modes (0x406CC0–0x406EB0)

### Mode Catalog

| Address | Name | Purpose |
|---------|------|---------|
| 0x406CC0 | `mobile_derrick_mode_post_move` | Install message handler, → idle_setup |
| 0x406DC0 | `mobile_derrick_mode_arrive` | Scan for oil patch on tile, begin deployment rotation |
| 0x406EB0 | `mobile_derrick_mode_deploy` | Transform into drillrig building entity |
| (shared) | `unit_mode_mobile_base_despawn` | Self-destruct (remove unit from world) |

### Handler: `UNIT_Handler_MobileDerrick`

- On init: sets `mode_arrive = mobile_derrick_mode_arrive`
- Channel: `TaskChannel_MobileDerrick`
- Initial movement: `unit_4172D0` → `unit_mode_417360` (direct walk)

### Mobile Derrick Lifecycle

```
spawn → unit_4172D0 (direct walk setup) → unit_mode_417360 (walk tick)
    │ (arrive at oil patch)
    ▼
mobile_derrick_mode_arrive (406DC0)
    │
    ├─ oil patch found on tile → rotate to SW
    │     │ (rotation complete)
    │     ▼
    │   mobile_derrick_mode_deploy (406EB0)
    │     │ spawns new drillrig entity at position
    │     │ transfers hitpoints
    │     ▼
    │   unit_mode_mobile_base_despawn — TERMINAL (self-destruct)
    │
    └─ no oil patch → unit_mode_415540 (idle, await new order)
```

---

## Drillrig Modes (0x4081C0–0x408330)

Drillrigs are buildings, not mobile vehicles, but they use the unit system.

| Address | Name | Purpose |
|---------|------|---------|
| 0x4081C0 | `drillrig_mode_drill_anim` | Play deployment animation (turret hide → reveal) |
| 0x408240 | `drillrig_mode_skip_anim` | Fast-deploy (from save/CPLC, no animation) |
| 0x408260 | `drillrig_mode_finalize` | Register with oil patch, begin operation |
| 0x408330 | `drillrig_mode_on_death` | Notify tankers, → building death |

### Drillrig Lifecycle

```
Init (placed fresh):
    drillrig_mode_drill_anim (4081C0) — hide turret, play drill animation
        ↓ (anim done)
    drillrig_mode_finalize (408260) — register on oil patch, enable pumping
        ↓
    unit_mode_403650 (building operational) + unit_build_40DD00

Init (from save/CPLC):
    drillrig_mode_skip_anim (408240) — skip directly
        ↓
    drillrig_mode_finalize → unit_mode_4034B0 (building idle/pump)

Death:
    drillrig_mode_on_death (408330)
        broadcasts TaskMessage_DrillrigDown on TaskChannel_Tanker
        → unit_mode_building_on_death (rubble)
```

---

## Mobile Base Modes (0x4278C0, 0x427C30)

Mobile bases deploy into stationary Outpost/Clanhall buildings.

| Address | Name | Purpose |
|---------|------|---------|
| 0x4278C0 | `mobile_base_mode_post_move` | Restore message handler, → idle_setup |
| 0x427C30 | `mobile_base_mode_plant_complete` | Finalize building, spawn new entity, self-destruct |

### Mobile Base Lifecycle

```
spawn → unit_init_mobile_base → unit_4172D0 (direct walk)
    │ (arrive at position)
    ▼
mobile_base_mode_post_move (4278C0) — restore handler → unit_mode_415540 (idle)
    │ (player orders deploy)
    ▼
unit_mobile_base_plant — play build animation
    │ (animation frames via sub_427BB0 → sub_427BF0)
    ▼
mobile_base_mode_plant_complete (427C30)
    │ calls unit_build_40DDD0
    │ creates Outpost/Clanhall entity
    │ transfers hitpoints
    ▼
self-destruct — TERMINAL
```

---

## Scout Spawn Mode (0x425920)

Not a vehicle per se, but uses a unique mode for delayed reveal.

| Address | Name | Purpose |
|---------|------|---------|
| 0x425920 | `scout_mode_init` | Countdown → claim for player, reveal fog, show marker |

```
spawn → scout_mode_init (countdown via multi_purpose_field_3)
    │ (timer expires)
    ▼
claim unit → reveal fog → broadcast UnitCreated → unit_mode_415540 (idle)
```

---

## Master Vehicle Mode Map (All Exclusive Modes)

```
┌──────────────── SHARED WITH INFANTRY ────────────────────────┐
│  All modes at 0x415540–0x419720 (see infantry.md)            │
│  Vehicles use same code, different branches on is_infantry   │
│                                                              │
│  KEY DIFFERENCES:                                            │
│  • Vehicles enter turning mode before walking                │
│  • Vehicles use sub_417550 Bresenham direct-path             │
│  • Vehicles have XL 3-ray pathfinding                        │
│  • Vehicles don't fidget, don't regen HP                     │
│  • Vehicle death = spinning wreckage (not corpse)            │
└──────────────────────────────────────────────────────────────┘

┌──────────────── TANKER ECONOMY (0x444xxx) ───────────────────┐
│  find_drillrig → goto_drillrig → rotate → load_oil           │
│  → goto_powerstation → rotate → reverse_dock → unload        │
│  → drive_out → goto_drillrig (LOOP)                          │
│                                                              │
│  Dock contention: unit_mode_tanker_419390 (wait for slot)    │
└──────────────────────────────────────────────────────────────┘

┌──────────────── TANKER CONVOY (0x405xxx) ────────────────────┐
│  checkpoint array → attack_move → arrive → next checkpoint   │
│  → ... → last checkpoint → drive offscreen → park            │
└──────────────────────────────────────────────────────────────┘

┌──────────────── MOBILE DERRICK (0x406xxx) ────────────────────┐
│  walk to oil → scan tile → rotate SW → deploy → self-destruct │
└───────────────────────────────────────────────────────────────┘

┌──────────────── DRILLRIG/BUILDING (0x408xxx) ─────────────────┐
│  drill anim → finalize → pump mode → (death → notify tankers) │
└───────────────────────────────────────────────────────────────┘

┌──────────────── MOBILE BASE (0x427xxx) ───────────────────────┐
│  walk to position → idle → plant → spawn building → destroy   │
└───────────────────────────────────────────────────────────────┘
```

---

## Naming Convention Proposal

Pattern: `vehicle_mode_{subsystem}_{action}`

Subsystem prefixes:
- `tanker` — solo economy tanker
- `convoy` — mission tanker convoy
- `derrick` — mobile derrick
- `drillrig` — drillrig building
- `base` — mobile base (outpost/clanhall deployer)
- `scout` — scout unit delayed-spawn

Examples:
- `vehicle_mode_tanker_load_oil`
- `vehicle_mode_tanker_rotate_at_dock`
- `vehicle_mode_convoy_countdown`
- `vehicle_mode_derrick_deploy`
- `vehicle_mode_base_plant_complete`

---

## Decompilation Artifacts & Mistakes

1. **`UNIT_Handler_Vehicle` is misleadingly named** — it's just a 1-line trampoline to `UNIT_Handler_Infantry`. There is NO separate vehicle handler logic. The naming suggests a distinct system that doesn't exist.

2. **`sub_417550` / `sub_417670` / `sub_417810` / `sub_417980`** — these are declared as `__thiscall` with raw `int this` parameter. IDA failed to recognize them as `__fastcall` with `KKND::Unit*`. They should be proper unit mode functions.

3. **`unit_mode_4446A0` (oil exhausted sentinel)** — sets `dword_477358 = 901223` and does nothing else. Likely a debug marker or partially-implemented handler that was never finished. In-game this causes the tanker to get stuck in an infinite no-op loop until the drillrig is reassigned externally.

4. **`unit_mode_tanker_convoy_405680`** — same pattern, sets `dword_477358 = 2371645`. Another debug/sentinel mode that functionally does nothing. Both use the same global as a "last mode reached" debug trace.

5. **Building modes (0x4034B0–0x403650)** are shared between drillrigs, factories, and all production buildings. They are NOT vehicle-specific despite appearing in the vehicle context. See the building/construction mode system for full details.
