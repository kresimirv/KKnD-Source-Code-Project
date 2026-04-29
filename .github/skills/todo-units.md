# Units TODO — Naming Issues & Decompilation Fixes

## 1. Unnamed / Misnamed Functions

| Current Name | Suggested Name | Evidence |
|---|---|---|
| `unit_44C890` | `unit_free` | Unlinks from active list, pushes to `g_unit_free_head`. Standard pool release pattern. |
| `unit_mode_415540` | `unit_mode_enter_idle` | Snaps position to tile, sets anim to idle, then `mode = unit_mode_idle`. One-shot setup before idle loop. |
| `unit_mode_417360` | `unit_mode_move_straight` | Beeline movement toward `order_next_waypoint`. Checks distance, transitions to `mode_return` on arrival. |
| `unit_mode_416060` | `unit_mode_move_scan` | Wall-following obstacle avoidance movement (CW/CCW scanning). |
| `unit_mode_402AB0` | `unit_mode_under_construction` | Building waiting for construction-stage messages. Transitions to `mode_arrive` on completion. |
| `unit_mode_403650` | `unit_mode_building_idle` | Building idle loop — plays idle/damaged anim, decrements cooldown. |
| `unit_mode_4034B0` | `unit_mode_building_snap_grid` | Snaps building to grid after plant. One-shot → `unit_mode_building_idle`. |
| `unit_mode_4035C0` | `unit_mode_building_pre_plant` | Snap + transition to `unit_mode_403540` (plant check). |
| `unit_mode_403540` | `unit_mode_building_plant_check` | Checks `unit_425820` then triggers `mode_arrive`. |
| `unit_mode_bomber_401800` | `unit_mode_bomber_approach` | Bomber flying toward target (name partially guessed from context). |
| `unit_mode_bomber_401600` | `unit_mode_bomber_attack_run` | Bomber in attack pass. |
| `unit_mode_bomber_4016B0` | `unit_mode_bomber_turn_around` | Bomber reversing for next pass. |
| `unit_mode_bomber_on_death` | OK | Already named. |
| `unit_mode_beast_enclosure_402440` | `unit_mode_factory_producing` | Beast enclosure producing a unit (likely shared pattern). |
| `unit_mode_beast_enclosure_on_completed` | OK | Already named. |
| `unit_mode_blacksmith_402870` | `unit_mode_research_active` | Blacksmith researching. |
| `unit_mode_blacksmith_on_completed` | OK | Already named. |
| `unit_mode_clanhall_on_completed` | OK | Already named. |
| `unit_mode_building_finalize` | OK | Already named. |
| `unit_mode_building_on_death` | OK | Already named. |
| `unit_mode_destroyed` | OK | Already named. |
| `unit_mode_destroy` | `unit_mode_cleanup` | Final cleanup (remove entity, terminate). Disambiguate from `unit_mode_destroyed`. |
| `sub_4157F0` | `unit_mode_idle_fidget` | Called from `unit_mode_idle` when `idle_fidget_timer > 50` and RNG triggers. Infantry idle animation variant. |

---

## 2. Unnamed Globals

| Current Name | Suggested Name | Evidence |
|---|---|---|
| `g_unit_free_head` | OK (or `g_unit_free_list`) | Consistent with `g_aircraft_free_list_head` pattern |
| `g_task_terminated_head` | `g_task_free_list` | This IS the task free list. "terminated" is misleading — tasks go here when freed, not when "terminated" in game sense. |
| `dword_477350` | `g_coroutine_free_list` | Already documented in lifecycle-todo. |
| `g_47DCD8_mobd_anchors` | `g_default_mobd_anchors` | Default anchor points copied to every new unit. |
| `dword_465630[4]` | `g_veterancy_hp_regen_divisors` | Used as `(hitpoints << 8) / dword_465630[veterancy]` — higher rank = faster regen. |
| `dword_47CFC8[255]` | `g_orientation_to_direction_index` | Maps 256 orientations → 32 direction indices for speed/anim lookup. |
| `dword_4731A8[]` | `g_direction_cos_table` | X speed multiplier per direction: `speed * table[dir] >> 6`. |
| `dword_4731C8[32]` | `g_direction_sin_table` | Y speed multiplier per direction (negated). |
| `g_next_entity_id` | OK | Monotonic ID counter. |
| `g_opportunity_timer` | OK | Frame counter for staggered opportunity-fire checks. |

---

## 3. Structural Issues

### 3a. TASK_yield Semantics in Callbacks — RESOLVED

`TASK_yield(task, Task_Sleep, N)` inside Callback-mode handlers (e.g. `unit_destroy`, `unit_on_damage_ex`) does NOT do a fiber suspend. It sets `task->wait_flags = yield_flags` and `task->sleep = N`, then **returns normally**. Post-yield code executes **immediately, same frame**.

The sleep counter prevents the scheduler from calling `entry_point` for N subsequent frames. This is a "schedule a gap" — not a suspension.

**Decompiler is CORRECT here**: Code after `TASK_yield()` in Callbacks really does run sequentially same frame. No misleading flow.

**Note**: Some handlers (OilPatch, Hut) are true Coroutines despite being "unit handlers" — for those, `TASK_yield` genuinely suspends the fiber. Check `g_scripts[task_type]->kind` to distinguish.

**TODO**: Audit all `TASK_yield` calls — verify caller's `task->kind`. Callbacks: post-yield = same frame. Coroutines: post-yield = next resume.

### 3b. unit_create BUG annotation

```c
v1->type = g_scripts[entity_task_type][1].kind; // BUG casting to ScriptType4* would yield ->unit_type
```

This comment claims a bug in prison/clanhall unit creation. Verify whether this actually produces wrong results or is just a type confusion in decompiler output.

### 3c. Unit Pool — Missing `g_unit_list_tail` in Investigation

`g_unit_list_tail` exists (line 72565 init) but was not found in earlier grep. Verify it's actually used for tail-insertion anywhere, or if all inserts are head-prepend (making tail unused except as sentinel).

### 3d. mode_idle / mode_attacked Fields Unused?

`unit_create` sets `mode_idle = nullptr` and `mode_attacked = nullptr`. Grep for actual assignments:
- `mode_idle` — appears to NEVER be set to a non-null value in the codebase
- `mode_attacked` — appears to NEVER be set to a non-null value

These fields may be **dead code** — vestigial from earlier design where idle/attacked were stored. Current design just overwrites `mode` directly.

**TODO**: Verify `mode_idle` and `mode_attacked` are truly dead fields.

---

## 4. Mode Function Naming Convention

Propose standard prefixes:

| Prefix | Meaning |
|---|---|
| `unit_mode_enter_xxx` | One-shot setup, transitions to xxx loop |
| `unit_mode_xxx` | Per-frame loop/tick |
| `unit_mode_xxx_on_yyy` | Callback triggered by event yyy |
| `unit_mode_building_xxx` | Building-specific modes |
| `unit_mode_bomber_xxx` | Aircraft/bomber-specific modes |
| `unit_mode_tanker_xxx` | Tanker-specific modes |
| `unit_mode_derrick_xxx` | Mobile derrick-specific modes |

---

## 5. Handler Function Classification

| Handler | Unit Types | TaskKind |
|---|---|---|
| `UNIT_Handler_Infantry` | All infantry + vehicles (Rifleman through AutocannonTank) | Callback |
| `UNIT_Handler_General` | El Presidente, King Zog, Warlord | Callback |
| `UNIT_Handler_Bomber` | Bomber, Wasp | Callback |
| `UNIT_Handler_Tower` | All tower types | Callback |
| `UNIT_Handler_Outpost` | Outpost, Clanhall (post-plant) | Callback |
| `UNIT_Handler_MachineShop` | Machine Shop | Callback |
| `UNIT_Handler_Clanhall` | Clanhall (building) | Callback |
| `UNIT_Handler_BeastEnclosure` | Beast Enclosure | Callback |
| `UNIT_Handler_Blacksmith` | Blacksmith | Callback |
| `UNIT_Handler_PowerStation` | Power Station | Callback |
| `UNIT_Handler_Drillrig` | Drillrig | Callback |
| `UNIT_Handler_Tanker` | Tanker | Callback |
| `UNIT_Handler_TankerConvoy` | Tanker Convoy | Callback |
| `UNIT_Handler_MobileDerrick` | Mobile Derrick | Callback |
| `UNIT_Handler_Prison` | Detention Center, Holding Pens | Callback |
| `UNIT_Handler_RepairBuilding` | Repair Bay, Menagerie | Callback |
| `UNIT_Handler_ResearchBuilding` | Research Lab, Alchemy Hall | Callback |
| `UNIT_Handler_OilPatch` | Oil Patch (terrain object) | Callback |
| `UNIT_Handler_Scout` | Scout (mission-specific) | Callback |

**All unit handlers are Callbacks.** No unit uses coroutine execution.

---

## 6. Open Questions

- [ ] What is `unit->render_node` vs `unit->entity->rend`? Unit has its own separate render node (healthbar overlay?).
- [ ] `unit->sprt` (MobdSprtImage) on Unit struct — what renders through this? Selection circle? Healthbar?
- [ ] `MessageHandler_Scout` vs `MessageHandler_Turret` — Scout seems to be the default for mobile units, Turret for newly created. Verify transition.
- [ ] `EventHandler_Passive` — set on destroyed units. What does it do (drop all messages)?
- [ ] `unit->control_groups[8]` — 8 control group bits? Or 8 separate group memberships?
- [ ] Why 599 units and not 600? Off-by-one in pool init or intentional?
