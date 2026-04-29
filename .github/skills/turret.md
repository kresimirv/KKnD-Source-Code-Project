# KKND Turret System

Turrets are independent sub-entities that run their own coroutine task, owned by a parent unit. Three distinct turret subsystems exist, sharing core architecture but diverging in combat behavior.

## Core Architecture

### Struct: `KKND::Turret`
| Field | Type | Purpose |
|-------|------|---------|
| `task` | Task* | Turret's own coroutine |
| `entity` | Entity* | Turret sprite entity |
| `owner` | Unit* | Parent unit that owns this turret |
| `target` | Unit* | Current aim target |
| `mode` | TurretMode (fn ptr) | Current state — called each tick |
| `current_mobd_frame` | int | Current rotation frame index |
| `reload_timer` | int | Ticks until next shot allowed (decremented by task loop) |
| `volley_remaining` | int | Shots left in current volley |
| `volley_reload_time` | int | Ticks between volleys (full reload) |
| `projectile_spawn_anchor` | Vec2 | World position where projectiles spawn |
| `attachment` | UnitAttachment* | Static config (rotation speed, fire delay, etc.) |
| `target_unit_id` | int | Tracked target's unique ID (stale-check) |

### Struct: `KKND::UnitAttachment`
Static turret configuration (one per unit type):
| Field | Purpose |
|-------|---------|
| `mobd_id` | Turret's sprite MOBD |
| `mode_attach` | Init function when turret attached |
| `rotation_speed` | How fast turret rotates (frames/tick) |
| `fire_delay` | Ticks between individual shots in a volley |
| `reload_time` | Ticks for full volley reload |
| `volley_size` | Shots per volley |
| `mobd_lookup_offset_idle` | Sprite offset for idle animation |
| `mobd_lookup_offset_attack` | Sprite offset for attack animation |
| `projectile_type` | → UnitProjectileType (damage/speed/visual) |

### Struct: `KKND::UnitProjectileType`
| Field | Purpose |
|-------|---------|
| `mobd_id` | Projectile sprite |
| `task_fn` | Projectile behavior coroutine |
| `mobd_lookup_offsets` | Animation offsets |
| `speed` | Travel speed |
| `damage_infantry` | Base damage vs infantry |
| `damage_vehicle` | Base damage vs vehicles |
| `damage_building` | Base damage vs buildings |
| `radius` | Splash radius |

Damage formula: `base_damage + (base_damage * g_veterancy_damage_mod[owner->veterancy]) >> 8`

---

## System 1: Passive Turret (Building/Mobile Unit Body Turret)

**Used by**: Buildings with cosmetic turrets, units that don't independently fire.
**Task**: `task_turret` (line 5778) — simple loop: `turret->mode(turret)` each tick.
**Init**: `turret_mode_init` (line 72334)
**Message handler**: `MessageHandler_Turret` — only handles `TaskMessage_DestroyAttachment`.
**Render**: `render_state_handler_unit_turret` — positions via `owner->mobd_anchors.turret` offset.

### Mode Catalog

| Address | Proposed Name | Purpose |
|---------|--------------|---------|
| `turret_mode_init` (72334) | `turret_mode_init` | Start anim, → idle |
| `turret_mode_idle` (72357) | `turret_mode_passive_track` | Rotate to match owner orientation, loops |
| `turret_mode_destroyed` (72374) | `turret_mode_destroy` | Remove entity, terminate task, free memory |

### State Machine
```
turret_mode_init
    ↓
turret_mode_passive_track ←──┐
    │ (loops each tick)      │
    └────────────────────────┘
    │ (TaskMessage_DestroyAttachment)
    ↓
turret_mode_destroy [TERMINAL]
```

---

## System 1b: Aircraft Turret

**Used by**: Barrage Craft, helicopters.
**Task**: Same `task_turret` but with different mode functions.
**Render**: `render_state_handler_aircraft_turret` — uses world position directly (not anchor offset), applies Z parallax.
**Message handler**: `MessageHandler_TurretCleanup` (mission-fail cleanup only).

### Mode Catalog

| Address | Proposed Name | Purpose |
|---------|--------------|---------|
| `mode_turret_4010E0` (5794) | `turret_mode_aircraft_init` | Set initial anim, → snap mode |
| `mode_turret_4010C0` (5784) | `turret_mode_aircraft_snap` | Snap orientation to owner direction each tick |

### State Machine
```
turret_mode_aircraft_init
    ↓
turret_mode_aircraft_snap ←──┐
    │ (loops, no combat)     │
    └────────────────────────┘
```

---

## System 2: Tower Turret (Static Defense)

**Used by**: Guard Tower, Machinegun Nest, Cannon Tower, Missile Battery, Rotary Cannon, Plasma Cannon.
**Task**: `task_unit_turret` / `task_turret_attach` (line 68448) — calls `turret->mode(turret)` AND decrements `reload_timer` each tick.
**Init**: `tower_turret_init` (line 68394)
**Message handler**: `MessageHandler_TowerTurret` — handles `TaskMessage_AttackOrder` (player-issued).
**Render**: `render_state_handler_unit_turret`.
**Target scanning**: `sub_4479F0` (line 68461) — grid-based scan with diplomacy check, range check, LOS check.

### Target Acquisition (`sub_4479F0` → `turret_scan_for_enemy`)
1. Compute tile grid: `±(firing_range >> 5)` tiles around owner
2. For each tile, check up to 5 ground unit slots
3. Also scan `g_aircraft_list_head` linked list
4. Skip: destroyed, non-enemy (diplomacy check), zero-threat decorations
5. Distance check: `MATH_vec2_length() < stats->firing_range`
6. LOS check: `unit_test_line_of_sight_bresenham()`
7. Return first valid target

### Mode Catalog (Standard Tower)

| Address | Proposed Name | Purpose |
|---------|--------------|---------|
| `turret_mode_447C40` (68617) | `turret_mode_scan_idle` | Scan for enemies, sleep 90 ticks if none |
| `turret_mode_attack_target` (68637) | `turret_mode_rotate_to_target` | Rotate toward target, validate |
| `turret_mode_447DD0` (68691) | `turret_mode_start_attack_anim` | Play attack animation, → fire check |
| `turret_mode_447E20` (68706) | `turret_mode_fire_check` | Wait for reload_timer=0, validate target |
| `turret_mode_447FA0` (68787) | `turret_mode_fire` | Spawn projectile, decrement volley, set reload |
| `sub_448110` (68856) | `turret_mode_fire_alt` | Alternate fire-cycle (dry fire / reset) |
| `turret_mode_448160` (68884) | `turret_mode_post_volley_reacquire` | After full volley, wait reload, re-engage |

### Mode Catalog (Missile Battery Extension)

| Address | Proposed Name | Purpose |
|---------|--------------|---------|
| `turret_mode_4482D0_missile_battery` (68953) | `turret_mode_missile_post_volley` | Sleep 15, start rotate to fixed fire position |
| `turret_mode_448290` (68942) | `turret_mode_missile_rotate_to_fire_pos` | Rotate to SW (fixed launch direction) |
| `turret_mode_448230` (68923) | `turret_mode_missile_raise_launcher` | Play raise-tubes animation, yield until done |

### State Machine (Standard Tower)
```
turret_mode_scan_idle
    │ (target found by scan)
    ↓
turret_mode_rotate_to_target ──── (target lost) ──→ scan_idle
    │ (aligned + in range + LOS)
    ↓
turret_mode_fire_check ─────────── (target lost) ──→ scan_idle
    │ (reload_timer == 0)
    ↓
turret_mode_fire
    │ (volley_remaining > 0)  → fire_check (continue volley)
    │ (volley exhausted)
    ↓
turret_mode_post_volley_reacquire
    │ (reload done + target valid) → fire_check
    │ (target lost)                → scan_idle
```

### State Machine (Missile Battery)
```
scan_idle → rotate_to_target → fire_check → fire
    ↓ (volley exhausted)
turret_mode_missile_post_volley (sleep 15 ticks)
    ↓
turret_mode_missile_rotate_to_fire_pos (rotate to SW)
    ↓
turret_mode_missile_raise_launcher (raise anim, yield)
    ↓
turret_mode_start_attack_anim
    ↓
turret_mode_fire_check → (resume fire loop)
```

---

## System 3: Mobile Combat Turret (Unit Escort/Guard)

**Used by**: 4x4 Pickup, Bike & Sidecar, ATV, Monster Truck, ATV Flamethrower, Anaconda Cannon, War Mastadont, Autocannon Tank, Missile Crab, Sentinel Droid, Mech.
**Task**: `task_unit_turret` (same task with reload decrement).
**Init**: `unit_turret_init` (line 21951)
**Message handler**: `MessageHandler_TurretCleanup` (cleanup only — targeting comes from owner's state).
**Render**: `render_state_handler_unit_turret`.
**Targeting**: Reads target from owner's combat state (no independent scanning — relies on unit AI to provide target).

### Mode Catalog

| Address | Proposed Name | Purpose |
|---------|--------------|---------|
| `sub_448980` (69284) | `turret_mode_combat_init` | Start idle anim, → track mode |
| `sub_4489B0` (69297) | `turret_mode_combat_idle` | Track owner orientation, detect owner's target |
| `sub_448B40` (69386) | `turret_mode_combat_aim` | Rotate toward target until aligned |
| `sub_448BF0` (69420) | `turret_mode_combat_quickaim` | Instant orient to target, → fire |
| `sub_448C40` (69432) | `turret_mode_combat_fire` | Fire at target, volley loop, re-aim on switch |
| `sub_448E90` (69519) | `turret_mode_combat_destroy` | Remove entity, terminate, free |

### State Machine
```
turret_mode_combat_init
    ↓
turret_mode_combat_idle ←──────────── (target lost)
    │ (owner has valid target)
    ↓
turret_mode_combat_aim
    │ (aligned)
    ↓
turret_mode_combat_fire ←──────────── (target switched) → combat_aim
    │ (loops while target valid + in range)
    │ (target lost)
    ↓
turret_mode_combat_idle
```

---

## Projectile Creation (in `turret_mode_fire`)

1. Position: `turret->projectile_spawn_anchor` (world coords from turret sprite anchor)
2. Entity: `entity_create_ex(projectile_type->mobd_id, ...)`
3. Task: `projectile_type->task_fn` — independent coroutine drives projectile flight
4. Target: `task->ctx = turret->target`
5. Kill credit: `entity->attacker = turret->owner`
6. Damage: Three-channel (infantry/vehicle/building) + veterancy modifier
7. Z-offset: `turret_entity->z + 768`

---

## Known `g_turret_*` Attachment Globals

| Global | Unit | System |
|--------|------|--------|
| `g_turret_4x4_pickup` | 4x4 Pickup | 3 (Mobile Combat) |
| `g_turret_bike_and_sidecar` | Bike & Sidecar | 3 |
| `g_turret_atv` | ATV | 3 |
| `g_turret_monster_truck` | Monster Truck | 3 |
| `g_turret_atv_flamethrower` | ATV Flamethrower | 3 |
| `g_turret_anaconda_cannon` | Anaconda Cannon | 3 |
| `g_turret_war_mastadont` | War Mastadont | 3 |
| `g_turret_barrage_craft` | Barrage Craft | 1b (Aircraft) |
| `g_turret_autocannon_tank` | Autocannon Tank | 3 |
| `g_turret_missile_crab` | Missile Crab | 3 |
| `g_turret_guard_tower` | Guard Tower | 2 (Tower) |
| `g_turret_machinegun_nest` | Machinegun Nest | 2 |
| `g_turret_cannon_tower` | Cannon Tower | 2 |
| `g_turret_missile_battery` | Missile Battery | 2 (w/ missile extension) |
| `g_turret_rotary_cannon` | Rotary Cannon | 2 |
| `g_turret_plasma_cannon` | Plasma Cannon | 2 |
| `g_turret_sentinel_droid` | Sentinel Droid | 3 |
| `g_turret_mech` | Mech | 3 |

---

## Naming Conventions

| Current Name | Proposed Name | Rationale |
|--------------|--------------|-----------|
| `task_turret` | `task_turret_passive` | Distinguishes from combat turret task |
| `task_unit_turret` / `task_turret_attach` | `task_turret_combat` | Has reload decrement = combat capable |
| `turret_mode_idle` | `turret_mode_passive_track` | It tracks owner, not truly idle |
| `turret_mode_447C40` | `turret_mode_scan_idle` | Scans for enemies autonomously |
| `turret_mode_attack_target` | `turret_mode_rotate_to_target` | Action is rotation/alignment |
| `turret_mode_447DD0` | `turret_mode_start_attack_anim` | Intermediate animation trigger |
| `turret_mode_447E20` | `turret_mode_fire_check` | Waits for reload, validates target |
| `turret_mode_447FA0` | `turret_mode_fire` | Actually spawns projectile |
| `sub_448110` | `turret_mode_fire_alt` | Alternate fire path (no spawn) |
| `turret_mode_448160` | `turret_mode_post_volley_reacquire` | Post-volley target re-validation |
| `turret_mode_4482D0_missile_battery` | `turret_mode_missile_post_volley` | Missile battery specific post-volley |
| `turret_mode_448290` | `turret_mode_missile_rotate_to_fire_pos` | Rotates to fixed SW orientation |
| `turret_mode_448230` | `turret_mode_missile_raise_launcher` | Raises launcher tubes animation |
| `sub_448980` | `turret_mode_combat_init` | Mobile turret init |
| `sub_4489B0` | `turret_mode_combat_idle` | Mobile turret idle/owner track |
| `sub_448B40` | `turret_mode_combat_aim` | Rotate toward combat target |
| `sub_448BF0` | `turret_mode_combat_quickaim` | Instant orientation snap |
| `sub_448C40` | `turret_mode_combat_fire` | Mobile turret fire loop |
| `sub_448E90` | `turret_mode_combat_destroy` | Mobile turret destruction |
| `sub_4479F0` | `turret_scan_for_enemy` | Target acquisition function |
| `tower_turret_init` | `tower_turret_create` | Creates tower turret instance |
| `unit_turret_init` | `mobile_turret_create` | Creates mobile combat turret |
| `unit_attach_turret` | `turret_attach_to_unit` | Attaches turret to building unit |
| `mode_turret_4010C0` | `turret_mode_aircraft_snap` | Aircraft turret snap-to-owner |
| `mode_turret_4010E0` | `turret_mode_aircraft_init` | Aircraft turret initialization |
| `MessageHandler_Turret` | `turret_msg_passive` | Passive turret message handler |
| `MessageHandler_TowerTurret` | `turret_msg_tower_attack_order` | Handles player attack commands |
| `MessageHandler_TurretCleanup` | `turret_msg_cleanup` | Mission-end cleanup handler |
| `render_state_handler_unit_turret` | `turret_render_anchored` | Renders at owner anchor point |
| `render_state_handler_aircraft_turret` | `turret_render_world_pos` | Renders at world position with Z parallax |

---

## Key Design Patterns

1. **Mode-function-pointer state machine**: Same pattern as units — `turret->mode` is a function pointer called every tick.

2. **Task-per-turret**: Each turret has its own coroutine. This allows independent timing (sleep/yield) without blocking the parent unit.

3. **Two-tier reload**: `fire_delay` between individual shots in a volley, `reload_time` between full volleys. `volley_size` controls burst length.

4. **Three damage channels**: Infantry/vehicle/building — allows weapons to be strong vs one type.

5. **Missile Battery special path**: Unique among turrets — must rotate to fixed direction (SW) before firing, has raise-launcher animation. Represents a "guided missile" that always launches from same orientation.

6. **Mobile turret targeting**: Does NOT scan independently. Reads target from owner's unit AI state (owner's `order_target`). Only fires when owner is in combat mode.

7. **Tower turret targeting**: Scans independently via `turret_scan_for_enemy`. Can also receive explicit attack orders from player via message handler.

8. **Veterancy damage scaling**: `g_veterancy_damage_mod[]` lookup table applies percentage bonus to all damage channels.

9. **Isometric depth sorting**: Turret render uses `owner->entity->z + owner->entity->y` for Z-order — standard isometric formula (higher Y = further back = lower Z-order).
