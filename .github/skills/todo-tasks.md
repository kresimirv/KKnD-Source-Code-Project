# Task System — TODO / Issues / Renaming

---

## 1. Flag Fields — Need Explicit Enum Definitions

The 3 flag fields on Task (`flags_20`, `flags_24`, `field_2C`) and `TaskYieldFlags` use raw hex constants scattered through the codebase. All bits should be declared as proper enums.

### 1a. `flags_20` → rename to `event_flags` or `wake_signals`

Accumulates per-frame event signals. Cleared to 0 by `TASK_yield`. Tested by scheduler against `wait_flags` to decide wake.

**Proposed enum `TaskEventFlags`:**

```c
enum KKND::TaskEventFlags : unsigned __int32 {
    TaskEvent_ResourceDepleted       = 0x00000001,  // bit 0  — oil patch amount==0 (kknd.c:11296)
    TaskEvent_LeftViewport           = 0x00020000,  // bit 17 — CPLC viewport culling (kknd.c:10463)
    TaskEvent_AnimKeyframe           = 0x00040000,  // bit 18 — animation point trigger (kknd.c:39888)
    TaskEvent_Collision_PosZ         = 0x00080000,  // bit 19 — BOXD collision +Z (kknd.c:8600)
    TaskEvent_Collision_NegZ         = 0x00100000,  // bit 20 — BOXD collision -Z (kknd.c:8572)
    TaskEvent_Collision_PosY         = 0x00200000,  // bit 21 — collision +Y / slope (kknd.c:8550,8786,8838)
    TaskEvent_Collision_NegY_Floor   = 0x00400000,  // bit 22 — collision -Y / floor / ramps (kknd.c:8527,8653,8694)
    TaskEvent_Collision_PosX         = 0x00800000,  // bit 23 — collision +X (kknd.c:8504,8893)
    TaskEvent_Collision_NegX         = 0x01000000,  // bit 24 — collision -X (kknd.c:8482,8947)
    TaskEvent_ZSpeedCapped           = 0x02000000,  // bit 25 — entity physics z_speed limit (kknd.c:39670)
    TaskEvent_YSpeedCapped           = 0x04000000,  // bit 26 — entity physics y_speed limit (kknd.c:39611)
    TaskEvent_XSpeedCapped           = 0x08000000,  // bit 27 — entity physics x_speed limit (kknd.c:39562)
    TaskEvent_AnimCompleted          = 0x10000000,  // bit 28 — animation reached end/looped (kknd.c:39863,39901)
    TaskEvent_Terminate              = 0x20000000,  // bit 29 — TASK_Terminate() called (kknd.c:66227)
    TaskEvent_MessageReceived        = 0x40000000,  // bit 30 — TASK_send_message delivered (kknd.c:36142)
    TaskEvent_SleepExpired           = 0x80000000,  // bit 31 — sleep countdown hit 0 (kknd.c:66294)
};
```

**Bits 1-16 (0x2–0x10000)**: No usages found. Reserved/unused.

### 1b. `flags_24` → rename to `event_flags_sticky` or `event_flags_accumulated`

Pattern: every place that writes `flags_20` immediately does `flags_24 |= flags_20`. So `flags_24` is the historical OR of all `flags_20` values. Bits persist until explicitly cleared.

**Same enum as `flags_20`** — uses same bit meanings.

**Special clearing pattern**: `flags_24 &= ~0x10000000u` (clear AnimCompleted sticky bit) found at:
- kknd.c:6011 — bomber destroyed
- kknd.c:7733 — unit death handler
- kknd.c:11176 — mobile base despawn
- kknd.c:11864 — unit explode
- kknd.c:26969 — unit destroyed
- kknd.c:68755 — turret reload complete

Purpose: prevent stale "anim complete" from triggering behavior after mode/state change.

**TODO**: Replace all raw hex reads/writes of `flags_24` with the `TaskEventFlags` enum.

### 1c. `field_2C` → rename to `inverse_wake_mask`

Set by `TASK_yield` when NOT sleeping: `field_2C |= a3` (third arg to yield). Cleared to 0 when coroutine resumes.

Scheduler wakes task when `(field_2C & ~flags_20) != 0` — i.e., when a bit in the mask is ABSENT from current events. "Wake me when this condition STOPS being true."

Known values passed as `a3`:
- `1` — wake when `TaskEvent_ResourceDepleted` is no longer set (most common, used by sidebar, cursor, UI)
- `2` — seen at kknd.c:9985

**TODO**: Clarify what `field_2C = 1` and `field_2C = 2` actually mean semantically in each call site. The "wake on absence" idiom is confusing — document each pattern.

### 1d. `TaskYieldFlags` — Complete Map

Current enum is incomplete. Full verified map:

```c
enum KKND::TaskYieldFlags : unsigned __int32 {
    Task_ResourceDepleted    = 0x00000001,  // wake on resource depletion event
    Task_AnimCompleted       = 0x10000000,  // wake when animation finishes (bit 28)
    Task_Terminate           = 0x20000000,  // marks task for removal (bit 29)
    Task_Wait                = 0x40000000,  // wake on message received (bit 30)
    Task_Sleep               = 0x80000000,  // wake after sleep countdown (bit 31)
    Task_Yield               = 0xC0000000,  // Wait | Sleep — wake on either
};
```

**TODO**: Rename `Task_1` to `Task_ResourceDepleted`. Rename `Task_Yield_10000000` to `Task_AnimCompleted`.

**Compound flags observed in TASK_yield calls:**
- `0xC0000000` (`Task_Yield`) = `Task_Wait | Task_Sleep` — common in sidebar/UI, yields with both sleep timer AND message wake
- `0x50000000` = `Task_AnimCompleted | Task_Wait` — used in button animation, wakes on either anim done or message
- `0x80000002` = `Task_Sleep | 0x2` — seen in projectile code (kknd.c:52479), sleep with extra flag

**TODO**: Document all compound flag combinations and their semantic meaning.

---

## 2. Naming Renames

### 2a. Task Struct Fields

| Current | Proposed | Reason |
|---------|----------|--------|
| `flags_20` | `event_flags` | Accumulates per-frame event signals, tested against `wait_flags` |
| `flags_24` | `event_flags_sticky` | Historical OR of all `event_flags` values |
| `field_2C` | `inverse_wake_mask` | Scheduler wakes when `(mask & ~event_flags) != 0` |
| `wait_flags` | OK as-is, or `yield_flags` | Set by TASK_yield, cleared on resume |
| `mode_turret` (offset 0x10) | `field_10` or investigate | Decompiler guessed wrong — not always turret-related |

### 2b. TaskYieldFlags Enum Members

| Current | Proposed | Reason |
|---------|----------|--------|
| `Task_1` | `Task_ResourceDepleted` | Bit 0 = resource patch depleted signal |
| `Task_Yield_10000000` | `Task_AnimCompleted` | Bit 28 = animation finished event |
| `Task_Terminate` | OK | Bit 29 |
| `Task_Wait` | `Task_WakeOnMessage` | Bit 30 = message received wake |
| `Task_Sleep` | `Task_WakeOnTimer` | Bit 31 = sleep countdown expired wake |
| `Task_Yield` | `Task_WakeOnAny` | 0xC0000000 = message OR timer |

### 2c. Global Variables

| Current | Proposed | Reason |
|---------|----------|--------|
| `g_script_handlers[352]` | `g_fn_ptr_serialization_table` | Used exclusively for save/load fn ptr→index mapping |
| `g_47A3CC` | `g_mission_controller_task` | Stores mission outcome controller task pointer |
| `EventHandler_Passive` | `MessageHandler_Noop` | Assigned to dead units to silently drop messages |

### 2d. ScriptType Structs

| Current | Proposed | Reason |
|---------|----------|--------|
| `ScriptType4` | `ScriptDef` (base) | Smallest variant, 24 bytes, common layout |
| `ScriptType2` | `ScriptDefEx1` | 32 bytes = base + 2 extra fields |
| `ScriptType1` | `ScriptDefEx2` | 40 bytes = base + 4 extra fields |
| `ScriptType3` | `ScriptDefWide` | 48 bytes, different `unit_type` offset — BREAKING layout difference |

### 2e. TaskMessageType Aliases

These are overloaded by context. Should add alias comments or split:

| Value | Gameplay Name | Menu Name |
|-------|--------------|-----------|
| 1512 | `TaskMessage_UnitDeselected` | Same — but on `TaskChannel_UnitLifecycle` means "unit died/removed" |
| 1526 | `TaskMessage_Infiltrate` | `TaskMessage_OpenBriefing` |
| 1527 | `TaskMessage_Follow` | `TaskMessage_RestartLevel` |
| 1528 | `TaskMessage_Retreat` | `TaskMessage_CloseDialog` |
| 1549 | `TaskMessage_SpawnTanker` | `TaskMessage_LoadGame` |

**TODO**: Add `TaskMessage_UnitRemoved` alias for 1512 when used on UnitLifecycle channel. Consider `TaskMessage_MissionVictory` alias for 1524 (`MoveOrder`) when sent to `g_mission_controller_task`.

### 2f. TaskChannel Names

| Current | Issue |
|---------|-------|
| `TaskChannel_unused_tanker_broadcast` (0xCA000012) | Actually used — TowerReady (kknd.c:68333) and SpawnTanker (kknd.c:52072) broadcast on it. Rename to `TaskChannel_BuildingEvents` or investigate further |

---

## 3. Decompilation Bugs

### 3a. `g_scripts` type cast bug (kknd.c:72838)

```c
v1->type = g_scripts[entity_task_type][1].kind;
// BUG: casting to ScriptType4* would yield ->unit_type
```

Indexes `g_scripts` as `ScriptType4[2]`, reading offset 0x18 from base. For `ScriptType4` (24 bytes) that correctly lands on `unit_type`. For `ScriptType1`/`ScriptType2`/`ScriptType3` with different sizes this reads garbage.

**Risk**: Save/load may write wrong unit_type for entities with non-ScriptType4 descriptors.

### 3b. `ScriptType3::unit_type` offset mismatch

`ScriptType3` has `unit_type` at offset 0x20, others at 0x14. The CPLC spawner casts everything to `ScriptType4*` — accessing `->unit_type` on ScriptType3 entries reads `script_type_3_field_14` instead.

**TODO**: Verify which TaskTypes use ScriptType3. Check if any CPLC entity uses a ScriptType3 descriptor.

### 3c. `mode_turret` field at Task offset 0x10

Decompiler named this `mode_turret` but it's at a generic offset. Need to check ALL reads/writes:
- Is it only used by turret tasks?
- Or is it a general-purpose field that got named after one usage?

**TODO**: Grep `mode_turret` across all usages, determine if rename needed.

---

## 4. Structural Questions

### 4a. `flags_20` bits 1-16 — any usage?

No usages found for bits 1 through 16 (0x2 through 0x10000). Either:
- Truly unused / reserved
- Used in code sections the decompiler didn't emit
- Used in inline assembly or data-driven checks not visible in C

**TODO**: Search for any raw reads of `*(int*)(task + 0x20)` with bitmasks in that range, possibly in un-decompiled sections.

### 4b. `field_2C` values 1 vs 2

Most `TASK_yield(task, Task_Yield, N)` calls pass `N=1`. One call at kknd.c:9985 passes `N=2`. The inverse-wake check `(field_2C & ~flags_20) != 0` means:

- `field_2C=1`: wake when bit 0 (ResourceDepleted) is NOT set in flags_20
- `field_2C=2`: wake when bit 1 is NOT set in flags_20

Since bit 1 has no known setter, `field_2C=2` would mean "always wake immediately" (bit 1 is never set, so `2 & ~flags_20` is always nonzero). This might be intentional — a "yield once then resume" pattern.

**TODO**: Verify kknd.c:9985 context — is `field_2C=2` equivalent to "yield for one frame"?

### 4c. Return value of TASK_yield for Callbacks

For coroutines, `TASK_yield` returns `flags_20` after resume — callers check `& 0x40000000` (got message) or `& 0x10000000` (anim done).

For callbacks, `TASK_yield` returns `flags_20` immediately (no suspend). Since yield clears `flags_20 = 0` just before, the return value is always 0 for callbacks (unless the Task_Sleep with ticks=0 path triggers).

**TODO**: Confirm no callback code inspects TASK_yield return value expecting meaningful data.

### 4d. `TaskNetzFlags` field and multiplayer gating

The scheduler skips tasks without `TaskNetzFlags_EnabledForMultiplayer` when `g_netz_synchronized_lockstep_mode` is set.

**TODO**: Find where `netz_flags` is set. Which tasks are multiplayer-synchronized vs. local-only?
