# KKND Projectile System

Projectiles are independent entities with their own coroutine tasks. Each `UnitProjectileType` defines a complete weapon — visual, flight behavior, damage, and impact effect. Nine distinct projectile archetypes exist.

## Core Architecture

### Struct: `KKND::UnitProjectileType` (kknd.h L2812)
| Field | Type | Purpose |
|-------|------|---------|
| `mobd_id` | MobdId | Projectile sprite resource |
| `task_fn` | void(__cdecl __noreturn *)(Task*) | Coroutine that drives flight + impact |
| `mobd_lookup_offset_travel` | ptrdiff_t | Animation offset during flight (-1 = none) |
| `mobd_lookup_offset_impact` | ptrdiff_t | Animation offset for explosion/impact |
| `speed` | int | Travel speed (world units/tick) |
| `damage_infantry` | int | Base damage vs infantry |
| `damage_vehicle` | int | Base damage vs vehicles |
| `damage_building` | int | Base damage vs buildings |
| `radius` | int | Splash radius (0 = single-target) |
| `_projectile_type_field_24` | int | Unknown |

### Global State
- `g_num_active_projectiles` — hard cap **200**. Checked before every spawn.
- `g_num_active_explosions` — cap **20**. For smoke trails / visual FX.

---

## Lifecycle

### Spawn (Ground Units — `unit_mode_attack`)
1. Check `g_num_active_projectiles < 200`
2. `entity_create_ex(proj->mobd_id, unit->entity, proj->task_fn, TaskKind_Coroutine, unit->mobd_anchors.turret)`
3. Set `entity->ctx = projectile` (UnitProjectileType data)
4. Set `task->ctx = unit->locked_target` (aim target)
5. Set `_80_attacker_unit_or_... = firing_unit` (kill credit)
6. Compute damage: `base + (base * g_veterancy_damage_mod[vet] >> 8)`
7. Send `TaskMessage_Attacked` to target (triggers aggro)
8. Yield for reload: `reload_time - (reload_time * g_veterancy_reload_mod[vet] >> 8)`

### Spawn (Turrets — `turret_mode_fire`)
Same as above but:
- Spawns at `turret->projectile_spawn_anchor`
- Uses `attachment->projectile_type` instead of `unit->stats->projectile`
- Z offset: `turret_entity->z + 768`

### Spawn (Bombers — `unit_mode_bomber_4016B0`)
- Uses `TaskKind_Callback` (not coroutine) — fires on timer
- `task->ctx = nullptr` — **no target** (bombs drop vertically)
- `cplc_spawn_param = 15` — 15-tick interval between bombs
- `multi_purpose_field_3` = bombs remaining in run
- Uses `render_state_handler_aircraft_turret`

### Impact & Cleanup
1. Stop movement
2. Play impact animation (`mobd_lookup_offset_impact`)
3. Apply damage:
   - **Splash**: `entity_40D8B0(entity, proj->radius)` — hits all units in radius
   - **Single-target**: `TaskMessage_ReceiveDamage` to target (with accuracy check)
   - **Anti-air splash**: manual iteration of `g_unit_list_head` checking aircraft in radius
4. `entity_remove(entity)`
5. `--g_num_active_projectiles`

---

## Damage Formula

```
effective_damage = base_damage + (base_damage * g_veterancy_damage_mod[veterancy]) >> 8
```

Three independent channels: infantry, vehicle, building. Target's armor type determines which channel applies.

---

## Accuracy Formula

```
effective_accuracy = min(base_accuracy + g_veterancy_accuracy_bonus[vet], 99)
scatter_range = 100 - effective_accuracy
offset = multiplier * (rand % scatter_range - scatter_range / 2)
x_speed += offset
y_speed += offset
```

Multipliers vary by projectile type:
| Multiplier | Types |
|-----------|-------|
| 16 | Grenade, Rocket |
| 8 | Generic, Bow, Mech plasma |
| 4 | Giant Beetle |

Lower multiplier = more accurate weapon.

---

## Projectile Archetypes

### 1. `proj_mode_nuke` — Beast Enclosure Nuke

**Flight**: Pure vertical ballistic. No XY movement. `z_speed=512`, `z_acceleration=-34` (gravity).
**Impact**: Splash damage via `entity_40D8B0`. Plays explosion at z=768.
**Special**: Uses sub-mode FSM (unique among projectiles). 4 internal states.
**Sound**: SoundId_163 (launch), SoundId_10 (impact).
**Used by**: Beast Enclosure superweapon (Mutant tech 4).

#### Nuke Sub-Modes
| Function | Proposed Name | Purpose |
|----------|--------------|---------|
| `nuke_mode_init` | `nuke_mode_launch` | Set z_speed/accel, play sound, → flight |
| `nuke_mode_401DC0` | `nuke_mode_flight` | Wait for z ≤ 0 (hit ground), → explode |
| `nuke_mode_401D30` | `nuke_mode_explode` | Stop movement, play explosion, splash damage, → cleanup |
| `nuke_mode_finalize` | `nuke_mode_cleanup` | Decrement counter, remove entity, terminate |

```
nuke_mode_launch → nuke_mode_flight → nuke_mode_explode → nuke_mode_cleanup [TERMINAL]
```

---

### 2. `proj_mode_grenade` — Ballistic Arc

**Flight**: Parabolic arc in 3D. Computes travel_time from distance/speed, sets `z_acceleration = -512 / (travel_time/2)` so grenade lands exactly at target.
**Impact**: Splash damage. Random explosion sound.
**Special**: Tumble animation (random spin speed). Accuracy scatter with 16× multiplier (least accurate weapon).
**Sound**: Random from `dword_46BB80[rand%2]` on impact.
**Used by**: Grenadier units.

---

### 3. `proj_mode_rocket` — Linear + Smoke Trail

**Flight**: Straight line. Speed halved (`>> 1`) from nominal. Smoke trail child entity every 12 ticks.
**Impact (Ground)**: Splash + screen shake via `sub_4389A0`. Explosion render.
**Impact (Air)**: Iterates all aircraft units, hits those in radius. Anti-air splash.
**Special**:
- Detects airborne targets (`entity->z >= 5120`) → uses aircraft render handler
- If parent has turret: launch offset z+5120, y+2560
- Player color tint on rocket entity
- Child entity `task_rocket_proj_explosion` = smoke puff (anim 412, capped at 20 active)
**Sound**: SoundId_14 (inferred from mech rocket path).
**Used by**: Rocket launcher infantry, SAM sites, missile turrets, Mech (every 3rd shot).

---

### 4. `proj_mode_flamethrower` — Multi-Particle Stream

**Flight**: Controller spawns 8 child `task_flamethrower_proj` entities, 5 ticks apart.
**Each flame**: Linear flight toward target with random jitter (±8 world units). Uses yield flag `0x80000002` (collision-active = deals damage during flight, not just on impact).
**Impact**: No explicit explosion. Flames ARE the damage.
**Special**:
- Stops early if target destroyed or parent dies
- No splash radius — damage applied per-particle via collision flag
**Sound**: SoundId_86 per flame particle.
**Used by**: Flamethrower units (Survivors Flamethrower, ATV Flamethrower).

---

### 5. `proj_mode_giant_beetle` — Acid Volley

**Flight**: Controller spawns 1 lead + 6 follow-up `task_giant_beetle_proj` children. Each child is ballistic arc (same formula as grenade).
**Each glob**: Anim 1784 (acid sprite). Tumble anim. Loops `yield(1)` until `z < 0`.
**Impact**: Per-glob splash damage.
**Special**:
- Lead projectile aimed accurately (4× scatter = most accurate projectile)
- Follow-ups add random velocity jitter (`4 * (rand & 0x1F) - 64`) — creates shotgun-like spread
- 2-tick delay between each child spawn
**Sound**: SoundId_161 (impact).
**Used by**: Giant Beetle (Mutant).

---

### 6. `proj_mode_mech` — Dual-Mode Weapon

**Mode selection**: `turret->volley_remaining % 3`
- ≠ 0 → **Plasma bolt** (2 out of every 3 shots)
- == 0 → **Rocket** (every 3rd shot)

**Plasma bolt**:
- Straight line, 8× scatter, anim 1152
- Launch z: `parent->z + 2560`
- Impact: anim 2084, screen shake, splash damage
- Sound: SoundId_87

**Rocket**:
- Identical to `proj_mode_rocket` — smoke trail, anti-air capability, player tint
- Speed halved, smoke every 8 ticks
- Sound: SoundId_14

**Used by**: Mech (Survivors faction tech 4).

---

### 7. `proj_mode_machinegun` — Hitscan/Instant Hit

**Flight**: **None**. Entity teleports to target position + random scatter (±16px).
**Damage**: Accuracy roll: `rand % 100 < (accuracy + vet_bonus)` → `TaskMessage_ReceiveDamage`. Single-target only.
**Special**:
- Spawns `task_machinegun_proj` child — muzzle flash that tracks turret/anchor position
- Infantry uses anim offset 2248, vehicles use 2184
- AutocannonTank excluded from muzzle flash (special case)
- 33% chance ricochet sound
- Rifleman/Shotgunner/DireWolf: 33% chance SoundId_190 (bullet whiz)
- 10-tick delay before impact visual

**Per-unit sounds**:
| Unit | Sound |
|------|-------|
| Shotgunner, DireWolf | SoundId_17 |
| CrazyHarry, ATV | SoundId_8 |
| DirkBike, 4x4Pickup, MonsterTruck | SoundId_5 |
| BikeAndSidecar | SoundId_7 |
| AutocannonTank | SoundId_6 |
| Default | SoundId_4 |

**Used by**: All ballistic kinetic weapon units (Rifleman, Shotgunner, DireWolf, CrazyHarry, ATV, DirkBike, 4x4Pickup, MonsterTruck, BikeAndSidecar, AutocannonTank).

---

### 8. `proj_mode_bow` — Arrow/Dart

**Flight**: Linear. Player color tint on arrow entity.
**Damage**: Accuracy roll (same as machinegun): `rand % 100 < (accuracy + vet_bonus)`. Single-target only. No splash.
**Special**:
- If `speed == 0`: instant hit (melee edge case?)
- 8× scatter multiplier
**Sound**: SoundId_89 (fire), SoundId_90 (impact).
**Used by**: Bow-wielding Mutant units (Berserker/Crazy Harry ranged variant).

---

### 9. `proj_mode_generic` — Catch-All Linear Projectile

**Flight**: Straight line, 8× scatter. Launch z: `parent->z + 2560`. Collision disabled (`g_null_collision`).
**Impact**: Splash damage + screen shake. Explosion anim.
**Special**: Per-unit sound selection.

**Per-unit sounds**:
| Unit | Fire Sound | Impact Sound |
|------|-----------|--------------|
| GiantScorpion | SoundId_161 | SoundId_161 |
| PlasmaTank, SentinelDroid | SoundId_87 | Random explosion |
| Others | SoundId_9 | Random explosion |

**Used by**: Giant Scorpion, Plasma Tank, Sentinel Droid, and any unit not assigned a specialized projectile type.

---

## Helper Functions

| Function | Proposed Name | Purpose |
|----------|--------------|---------|
| `entity_40D8B0` | `projectile_apply_splash_damage` | Apply damage to all units in radius |
| `sub_4389A0` | `camera_screen_shake` | Shake screen on heavy impact |
| `task_rocket_proj_explosion` | `task_smoke_trail_puff` | Smoke trail child entity (anim 412) |
| `task_machinegun_proj` | `task_muzzle_flash` | Muzzle flash visual tracking parent anchor |
| `task_flamethrower_proj` | `task_flame_particle` | Single flame in a stream |
| `task_giant_beetle_proj` | `task_acid_glob` | Single acid glob in a volley |
| `sub_435D40` | (unknown — between explosion task and rocket) | Likely render helper |

---

## Naming Conventions

| Current Name | Proposed Name | Rationale |
|--------------|--------------|-----------|
| `proj_mode_nuke` | `task_projectile_nuke` | Task function (not a mode despite name) |
| `proj_mode_grenade` | `task_projectile_grenade` | Task function |
| `proj_mode_rocket` | `task_projectile_rocket` | Task function |
| `proj_mode_flamethrower` | `task_projectile_flamethrower` | Controller task |
| `proj_mode_giant_beetle` | `task_projectile_acid_volley` | Controller task |
| `proj_mode_mech` | `task_projectile_mech_dual` | Dual-mode task |
| `proj_mode_machinegun` | `task_projectile_hitscan` | Hitscan behavior |
| `proj_mode_bow` | `task_projectile_arrow` | Arrow behavior |
| `proj_mode_generic` | `task_projectile_linear` | Generic linear flight |
| `task_flamethrower_proj` | `task_flame_particle` | Child particle |
| `task_giant_beetle_proj` | `task_acid_glob` | Child glob |
| `task_machinegun_proj` | `task_muzzle_flash` | Visual-only child |
| `task_rocket_proj_explosion` | `task_smoke_trail_puff` | Smoke visual child |
| `nuke_mode_init` | `nuke_mode_launch` | Descriptive of action |
| `nuke_mode_401DC0` | `nuke_mode_flight` | Waiting for ground hit |
| `nuke_mode_401D30` | `nuke_mode_explode` | Explosion + damage |
| `nuke_mode_finalize` | `nuke_mode_cleanup` | Resource cleanup |
| `g_num_active_projectiles` | `g_projectile_count` | Shorter, clear |
| `g_num_active_explosions` | `g_explosion_count` | Shorter, clear |
| `entity_40D8B0` | `projectile_apply_splash_damage` | Describes action |
| `sub_4389A0` | `camera_screen_shake` | Screen shake effect |

---

## Key Design Patterns

1. **One task_fn per weapon archetype**: `UnitProjectileType->task_fn` is the entire weapon behavior. No shared base class — each archetype is a standalone coroutine.

2. **Controller + child pattern**: Complex weapons (flamethrower, giant beetle, mech rocket) spawn child entities that are themselves mini-projectiles. Controller manages timing and early-termination.

3. **Hitscan vs physics**: Two fundamental categories:
   - **Physics projectiles** (nuke, grenade, rocket, generic, bow): fly through world, hit on arrival
   - **Hitscan** (machinegun): teleport to target, accuracy roll determines hit/miss

4. **Accuracy as miss-chance**: Accuracy doesn't affect trajectory of physics projectiles (they always reach target area). Instead, scatter is added to the velocity vector — higher accuracy = less scatter = projectile lands closer to target center.

5. **Single-target vs splash**: Weapons either use splash radius (`entity_40D8B0`) or accuracy-roll single-target (`TaskMessage_ReceiveDamage`). Never both. Flamethrower is special — uses collision-active flight flag.

6. **Anti-air special case**: Rockets have dual impact logic. Ground targets get normal splash. Air targets trigger manual unit-list scan checking aircraft types specifically.

7. **200 projectile cap**: Hard global limit. When reached, units simply can't fire. This prevents performance degradation in large battles.

8. **Veterancy affects everything**: Damage (+%), accuracy (+flat), reload time (-%). All three compound to make veteran units significantly more dangerous.

9. **Kill credit via `_80_attacker_unit_or_...`**: Projectile carries reference to firing unit. When splash damage kills, credit goes to original attacker (for veterancy gain and kill tracking).

10. **No projectile-projectile interaction**: Projectiles cannot hit other projectiles. Only units are valid damage targets.
