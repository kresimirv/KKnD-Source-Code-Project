# KKND Task System — Architecture Reference

The task scheduler is KKND's cooperative multitasking kernel. Every game entity, AI controller, UI widget, projectile, explosion, and menu screen runs as a Task.

---

## Core Structs

### Task (kknd.h:1225)

```c
struct KKND::Task {
    Task *next;                    // active/terminated list linkage
    Task *prev;
    TaskLocal *locals;             // singly-linked list of TASK_Alloc'd memory blocks
    TaskChannel channel;           // pub/sub channel for broadcasts
    int mode_turret;               // overloaded field (turret mode fn ptr in some contexts)
    int sleep;                     // frames to skip before next execution
    TaskKind kind;                 // 0=Coroutine, 1=Callback
    TaskNetzFlags netz_flags;      // multiplayer sync flags
    int flags_20;                  // runtime state flags (accumulates OR'd values each frame)
    int flags_24;                  // persistent flags (OR'd with flags_20)
    TaskYieldFlags wait_flags;     // what conditions to wait for (set by TASK_yield)
    int field_2C;                  // inverse wait mask (wake if (field_2C & ~flags_20) != 0)
    TaskMessage *message_queue;    // singly-linked message queue (for coroutines)
    MessageHandler message_handler;// synchronous handler (for callbacks)
    Entity *entity;                // the entity this task drives (can be NULL)
    void *ctx;                     // domain-specific context (Unit*, Turret*, UpgradeProcess*, etc.)
    TaskFn entry_point;            // handler function (for Callbacks) or coroutine fiber (for Coroutines)
};
```

### TaskMessage (kknd.h:1186)

```c
struct KKND::TaskMessage {
    TaskMessage *next;       // intrusive singly-linked list
    Task *task;              // sender
    TaskMessageType type;    // message ID (1497-1551 range + special 0xFFFFFFxx)
    void *payload;           // arbitrary data (usually Unit*, Entity*, or NULL)
};
```

Messages are pooled: `g_next_free_task_message` is a pre-allocated free list. `task_message_recycle()` returns a message to the pool.

---

## Two Execution Models

### TaskKind_Callback (kind == 1)

- **No fiber.** Scheduler calls `task->entry_point(task)` every eligible frame.
- Messages delivered **synchronously** — `TASK_send_message` calls `task->message_handler()` inline, no queuing.
- Used by: combat units, buildings, turrets, AI controllers.

### TaskKind_Coroutine (kind == 0)

- **Has a fiber** with its own stack (default size from `g_coroutine_default_stack_size`).
- Scheduler calls `TASK_ExecuteAsync(task->entry_point)` to resume the fiber.
- Messages **queued** onto `task->message_queue`, popped via `TASK_pop_message()`.
- `TASK_yield()` suspends fiber mid-function.
- Used by: OilPatch, Hut, explosions, projectiles, sidebar buttons, camera shake, UI dialogs, mission scripts.
- Handlers typically declared `__cdecl __noreturn` with `while(1) TASK_yield(...)` loops, or plain `__cdecl` that run to completion.

---

## Task Creation

Only two creation functions exist:

### TASK_CreateCallback (kknd.c:66078)

```c
Task *TASK_CreateCallback(TaskChannel chan, TaskFn fn);
```

- Pops task from `g_task_terminated_head` free list.
- Sets `kind = TaskKind_Callback`, `entry_point = fn`, `channel = chan`.
- Links to head of `g_task_active_head` doubly-linked list.
- Returns immediately — task won't execute until next scheduler tick.

### TASK_CreateFiber (kknd.c:66043)

```c
Task *TASK_CreateFiber(TaskChannel chan, TaskFn fn, size_t stack_size);
```

- Pops task from `g_task_terminated_head` free list.
- Sets `kind = TaskKind_Coroutine`.
- Creates a coroutine via `TASK_coroutine_create(coroutine_starter, stack_size)`.
- Stores `fn` in `g_task_creation_main`, task in `g_task_creation_arg`.
- **Immediately switches** to the new fiber via `TASK_ExecuteAsync` — the coroutine runs until its first `TASK_yield`.
- Returns after the new coroutine yields back.

**Key difference**: `TASK_CreateCallback` is deferred (runs next tick). `TASK_CreateFiber` runs the handler immediately until first yield.

---

## TASK_yield — Polymorphic Behavior (kknd.c:66133)

```c
unsigned int TASK_yield(Task *task, TaskYieldFlags yield_flags, int sleep_ticks);
```

### On Coroutines (kind == 0):
1. Sets `task->wait_flags = yield_flags`, `task->sleep = sleep_ticks`.
2. Clears `task->flags_20 = 0`.
3. Calls `TASK_ExecuteAsync(g_coroutine_list_head)` — **fiber suspend**. Execution pauses here.
4. When resumed: clears `wait_flags`, returns `flags_20` (accumulated wake-up flags).

### On Callbacks (kind == 1):
1. Sets `task->wait_flags = yield_flags`, `task->sleep = sleep_ticks`.
2. Clears `task->flags_20 = 0`.
3. **Returns normally.** Code after yield runs immediately, same frame.
4. The sleep counter prevents scheduler from calling `entry_point` for N frames.

This means `TASK_yield` inside a Callback is a "schedule a gap" — not a suspension.

### Callback Yield In Practice — unit_destroy Example

```c
// Inside a Callback handler:
unit->hitpoints = 0;
TASK_yield(task, Task_Sleep, 1);   // sets sleep=1, returns immediately
unit->hitpoints = 0;               // executes NOW, same frame
unit->mode = next_mode;            // executes NOW, same frame
unit->destroyed = 1;               // executes NOW, same frame
// handler returns. Next frame: scheduler skips this task (sleep=1).
// Frame after: sleep expires, handler called again, runs new mode.
```

The yield does NOT pause execution — it marks the task to be **skipped for 1 frame** on the NEXT scheduler tick. All post-yield code runs immediately. This is a "schedule a gap" operation, not a suspension.

### Why Combat Units Use Callbacks, Not Coroutines

Combat units need to be **externally interruptible** every frame. Their handler does:
1. Call `unit->mode(unit)` — one tick
2. Check bounds, scan for opportunity targets, update order target

A coroutine can't be externally interrupted without complex cancellation logic. The mode-pointer pattern gives instant preemption — any external code can overwrite `unit->mode`.

Passive terrain objects (OilPatch, Hut) don't need this — they just sit there waiting for events, so coroutine with infinite yield loop is simpler.

### Confirmed Handler Classification

**Coroutine handlers** (declared `__cdecl __noreturn`, use `while(1) TASK_yield`):
- `UNIT_Handler_OilPatch` (kknd.c:29875), `UNIT_Handler_Hut`

**Callback handlers** (use `unit->mode(unit)` pattern, return each frame):
- `UNIT_Handler_Infantry`, `UNIT_Handler_General`, `UNIT_Handler_Bomber`
- All building handlers: Tower, Outpost, MachineShop, Clanhall, BeastEnclosure, etc.
- `UNIT_Handler_Tanker`, `UNIT_Handler_TankerConvoy`, `UNIT_Handler_MobileDerrick`

**Rule of thumb**: `__noreturn` + `while(1) TASK_yield` → Coroutine. Calls `unit->mode(unit)` and returns → Callback.

### TaskYieldFlags (kknd.h:1208)

| Flag | Value | Meaning |
|------|-------|---------|
| `Task_1` | 0x1 | Unknown wake condition |
| `Task_Yield_10000000` | 0x10000000 | Wake on animation/render complete |
| `Task_Terminate` | 0x20000000 | Task marked for termination |
| `Task_Wait` | 0x40000000 | Wake on message received |
| `Task_Sleep` | 0x80000000 | Wake after sleep counter expires |
| `Task_Yield` | 0xC0000000 | Combined sleep+wait — wake on any |

---

## Scheduler Loop — TASK_ExecuteScheduledTasks (kknd.c:66237)

Runs once per game frame. Iterates `g_task_active_head` linked list:

```
for each task in active list:
    if task has TERMINATE flag (0x20000000):
        → free task locals, discard message queue
        → unlink from active list
        → push onto g_task_terminated_head (recycle)
        → if coroutine: destroy fiber
        → continue

    if multiplayer lockstep mode && !task.netz_flags.EnabledForMultiplayer:
        → skip

    if task.sleep > 0:
        → decrement sleep
        → if sleep hits 0: set flag 0x80000000 in flags_20 and flags_24

    check wake conditions:
        if !wait_flags                                    → execute (no wait)
        OR if (wait_flags & flags_20) != 0                → execute (waited condition met)
        OR if (field_2C & ~flags_20) != 0                 → execute (inverse condition)
        else → skip this frame

    execute:
        if kind == Callback: TASK_ExecuteSync(task)       → task->entry_point(task)
        if kind == Coroutine: TASK_ExecuteAsync(entry_point) → resume fiber
```

**Important**: Terminated tasks are cleaned up **during** iteration, not after. The code backs up to `i->prev` before unlinking, so the loop continues correctly.

---

## Messaging System

### Point-to-Point: TASK_send_message (kknd.c:36110)

```c
BOOL TASK_send_message(Task *sender, TaskMessageType msg, void *payload, Task *receiver);
```

- **If receiver is Callback** and has `message_handler`: calls handler **synchronously inline**. No queue.
- **If receiver is Coroutine** (or Callback without handler): allocates message from pool, pushes onto `receiver->message_queue`.
- Sets wake flag `0x40000000` (Task_Wait) on receiver's `flags_20` and `flags_24`.

### Broadcast: TASK_broadcast_message (kknd.c:36149)

```c
BOOL TASK_broadcast_message(Task *sender, TaskMessageType msg, void *payload, TaskChannel channel);
```

- If `channel == TaskChannel_Everyone (0xFFFFFFFF)`: sends to ALL active tasks.
- Otherwise: sends only to tasks where `task->channel == channel`.
- Same delivery semantics per-task as `TASK_send_message`.

### Pop (Coroutines only): TASK_pop_message (kknd.c:36234)

```c
TaskMessage *TASK_pop_message(Task *task);
```

Pops head of `task->message_queue`. Caller must `task_message_recycle(msg)` after processing.

---

## Task Memory: TASK_Alloc / TASK_FreeLocal

```c
void *TASK_Alloc(Task *task, size_t size);     // malloc + link to task->locals
void TASK_FreeLocal(Task *task, void *mem);     // unlink from task->locals + free
```

Memory allocated via `TASK_Alloc` is linked to a specific task's `locals` list. When the task is terminated, the scheduler frees ALL locals automatically. This is a **task-scoped arena** pattern.

Critical implication: `TASK_FreeLocal(turret->owner->task, turret)` frees memory from the PARENT's locals — because turrets are allocated via `TASK_Alloc(unit->task, sizeof(Turret))`.

---

## Task Termination: TASK_Terminate (kknd.c:66219)

```c
void TASK_Terminate(Task *task);
```

- Sets terminate flag: `flags_20 |= 0x20000000`.
- If coroutine: immediately switches back to scheduler via `TASK_ExecuteAsync`.
- Actual cleanup happens in the scheduler loop (next frame): free locals, drain queue, unlink, push to terminated pool.

For coroutines that are plain `__cdecl` (not `__noreturn`): returning from the handler function returns into `coroutine_starter`, which calls `TASK_Terminate` automatically — **implicit self-termination**.

---

## Channels — Pub/Sub Architecture

Channels are 32-bit integers. A task has exactly one channel. Broadcasts reach all tasks on matching channel.

### Channel Categories

| Range | Purpose | Examples |
|-------|---------|---------|
| 0x0 | No channel | `TaskChannel_None` — most tasks start here |
| 0x1–0x19 | Menu/UI flows | MainMenu, MultiplayerProvider, TextInput, etc. |
| 0x9876 | Unit lifecycle events | `TaskChannel_UnitLifecycle` |
| 0xBABA–0xCACA | Sidebar UI | SidebarButton, SidebarHelp, SidebarPlaceholder |
| 0xEAEA | Unit type tag | `TaskChannel_Units` — all units register here |
| 0xCA000000+ | Building-specific | One per building type (Outpost, Clanhall, etc.) |
| 0xDA000000+ | Ingame menu UI | Ingame menu controller, sliders, buttons, dialogs |
| 0xFFFFFFFF | Everyone | `TaskChannel_Everyone` — reaches ALL tasks |

### Key Channel Flows

**TaskChannel_UnitLifecycle (0x9876)** — the AI awareness bus:
- **Senders**: Every dying unit broadcasts `TaskMessage_UnitDeselected`; every spawning unit broadcasts `TaskMessage_UnitCreated`.
- **Listeners**: AI controllers (`MessageHandler_AiController_General`, Mute05, Mute08). They receive unit birth/death events synchronously and update tracking lists.
- **Note**: Despite the name, `UnitDeselected` on this channel means "unit removed from game," not UI deselection.

**TaskChannel_Tanker (0xCA00000F)** — economic disruption bus:
- **Senders**: Drillrig/PowerPlant death code broadcasts `TaskMessage_DrillrigDown` / `TaskMessage_PowerPlantDown`.
- **Listeners**: All tanker units. They react by seeking new drillrig/power station.

**TaskChannel_Units (0xEAEA)** — passive type tag:
- All units set `task->channel = TaskChannel_Units` during init (kknd.c:72875).
- Buildings immediately overwrite with their specific channel (PowerPlant, Outpost, etc.).
- **Nobody broadcasts to this channel.** It's used as a type check: `task->channel == TaskChannel_Units`.

**Building channels (0xCA000000 range)** — self-notification:
- Building sets its task to its specific channel (e.g., `TaskChannel_Outpost`).
- On construction completion, `TaskMessage_BuildingComplete` is broadcast on that channel.
- The building's own task receives it and transitions to production-ready state.
- On death, building clears channel to `TaskChannel_None` to stop receiving.

---

## Task Chaining Patterns

There is **no explicit parent pointer** in the Task struct. Parent-child relationships use domain structs and messages.

### Pattern 1: Domain Struct Link (Building → Turret)

```
Building creates turret:
  turret = TASK_Alloc(unit->task, sizeof(Turret))   // alloc from PARENT
  entity_create_ex(..., task_unit_turret, Callback)   // creates child task+entity
  turret->task = child_task
  turret->owner = unit                                // child → parent
  unit->turret = turret                               // parent → child

Building destroys turret:
  TASK_send_message(NULL, DestroyAttachment, NULL, turret->task)
  → MessageHandler_Turret sets turret->mode = turret_mode_destroyed
  → Next tick: turret_mode_destroyed() calls:
      entity_remove(turret->entity)
      TASK_Terminate(turret->task)
      TASK_FreeLocal(turret->owner->task, turret)   // frees from PARENT
```

### Pattern 2: Fire-and-Forget (Unit → Projectile)

```
Unit fires:
  entity_create_ex(proj.mobd_id, unit->entity, proj.task_fn, Coroutine, turret_anchor)
  entity->ctx = projectile_type_params
  task->ctx = locked_target_unit
  ++g_num_active_projectiles

Projectile runs as coroutine:
  calculate trajectory → TASK_yield(sleep=flight_time)
  on impact: area damage or targeted damage
  entity_remove(entity)
  --g_num_active_projectiles
  return  → coroutine_starter calls TASK_Terminate
```

No parent reference kept. Projectile self-terminates. Global counter tracks active count (max 200).

### Pattern 3: Channel Broadcast (PowerStation → Tanker)

```
PowerStation spawns tanker:
  entity_create_by_unit_type(Tanker, x, y+offset, player)
  (no direct reference stored)

Tanker init:
  task->channel = TaskChannel_Tanker
  tanker_state->powerstation = <found by proximity scan>

PowerStation dies:
  TASK_broadcast_message(task, PowerPlantDown, NULL, TaskChannel_Tanker)

All tankers receive PowerPlantDown:
  → seek new PowerStation or go idle
```

### Pattern 4: Global Pointer (Mission Script → Conditions)

```
Mission controller:
  g_47A3CC = task  // store self in global
  TASK_CreateFiber(None, victory_checker, 0)
  while(1) {
    TASK_yield(Sleep, 30)
    pop messages → check for MoveOrder(=victory) or MissionFailed
  }

Victory checker:
  poll game conditions each tick
  when met: TASK_send_message(NULL, MoveOrder, NULL, g_47A3CC)
  TASK_Terminate(task)
```

### Pattern 5: Trampoline (AI Task Init)

```
AI task created with entry_point = ai_task_40B700_general_ai
First tick:
  task->entry_point = ai_task_409770   // swap to real tick handler
  task->message_handler = MessageHandler_AiController_General
  // ... init AI state ...
  TASK_yield(task, Task_Sleep, 1)
Subsequent ticks:
  scheduler calls ai_task_409770 directly (the new entry_point)
```

Used by all AI controller variants. First frame does init and swaps handler.

### Pattern 6: Mode-Based FSM (Upgrade → Building)

```
Building starts upgrade:
  entity_create_ex(..., sub_4381A0, Callback)  // upgrade overlay task
  upgrade->entity_1 = building_entity          // parent ref via entity
  upgrade->mode = upgrade_mode_init
  task->channel = TaskChannel_UpgradeProgress
  task->message_handler = MessageHandler_UpgradeProcess

Each tick:
  upgrade_mode_init checks production cost remaining
  transitions through visual update modes

Building cancels:
  TASK_broadcast_message(..., UpgradeCancelled, NULL, TaskChannel_UpgradeProgress)
  → MessageHandler_UpgradeProcess sets cancelled=1
  → Next tick: upgrade_mode_init detects cancelled → self-terminates

Upgrade complete:
  Production system signals via shared memory (entity->ctx)
  upgrade_mode_init detects cost==0 → self-terminates
  Building receives TaskMessage_UpgradeComplete separately
```

---

## g_script_handlers[352] — Handler Lookup Table (kknd.c:771)

Flat array of 352 `void*` function pointers. Contains ALL handler functions: unit handlers, mode functions, message handlers, turret modes, projectile handlers, UI tasks, etc.

**Purpose**: Save/load serialization. When saving, the code searches this table to convert function pointers to indices. When loading, it uses indices to restore function pointers.

**NOT the same as** `g_scripts[]` (the ScriptType descriptor array used by CPLC spawner).

### Table Organization (indices 0-351)

```
[0]     NULL
[1-14]  Unit handlers: Outpost, Tanker, MachineShop, Clanhall, BeastEnclosure,
        PowerStation, Drillrig, OilPatch, MobileDerrick, RepairBuilding(x2),
        ResearchBuilding(x2), Blacksmith
[15-19] Tower handlers (5 variants)
[20-22] Special mission tasks: sub_433060, sub_424CE0, sub_425400
[23-27] General, TankerConvoy, Scout, Prison(x2)
[28-29] Turret, Bomber
[30+]   Mode functions, message handlers, turret modes, projectile handlers...
        (mixed, no strict ordering)
[~340+] Tower modes, turret modes, turret init/idle/destroyed, MessageHandler_Turret
```

---

## g_scripts[] — Script Type Descriptors (binary data)

`g_scripts` is an array of `ScriptType4*` pointers, one per `TaskType` (up to 196). Points to ROM data in the executable's data segment (not emitted as C source by decompiler).

Used ONLY by CPLC spawner (`cplc_entity_init`) and save/load unit type resolution.

### ScriptType Struct Variants

All share the same first 3 fields at offsets 0/4/8:

| Struct | Size | Extra Fields | handler signature |
|--------|------|--------------|-------------------|
| `ScriptType4` | 24B | field_C, field_10, unit_type | `__cdecl __noreturn` |
| `ScriptType2` | 32B | + field_18, field_1C | `__cdecl` |
| `ScriptType1` | 40B | + field_18..field_24 | `__cdecl` |
| `ScriptType3` | 48B | field_14..field_1C before unit_type, + field_24..field_2C | `__cdecl` |

Common layout:
```c
offset 0: MobdId mobd_id        // MOBD sprite resource ID
offset 4: TaskFn handler         // entry point function
offset 8: TaskKind kind           // 0=Coroutine, 1=Callback
...
offset varies: UnitType unit_type // what unit type this script produces
```

**Decompilation issue**: `ScriptType3` has `unit_type` at offset 0x20 instead of 0x14 like the others. The decompiler casts everything to `ScriptType4*`, which works for fields at offsets 0/4/8 but gives wrong `unit_type` for ScriptType3 entries.

**Bug at kknd.c:72838**: `g_scripts[entity_task_type][1].kind` — This indexes `g_scripts` as if it were `ScriptType4[2]` array, reading offset 0x18 relative to base. For ScriptType4 that's `unit_type` (correct). For other sizes, this is wrong.

---

## CPLC Level Spawning

CPLC = "Compiled Placement" — level file section containing entity positions.

### Data Structure

```c
struct CplcEntity {
    TaskType task_type;    // indexes into g_scripts[]
    int x, y, z;           // world position
    CplcEntity *next_x_sorted, *prev_x_sorted;  // X-sorted linked list
    CplcEntity *next_y_sorted, *prev_y_sorted;  // Y-sorted linked list
    CplcSpawnParams spawn_params;
};
```

Entities stored in two sorted linked lists (X and Y). Camera scrolling walks these lists to find entities entering the viewport.

### Spawn Flow

```
LVL_cplc_init()
  → Parse "CPLC" section from level file
  → Build X-sorted and Y-sorted linked lists
  → Find Camera entity for initial viewport

cplc_init_all() / cplc_init_subset()
  → For each TaskType: walk sorted list, call cplc_entity_init()

cplc_entity_init(cplc):
  → script = g_scripts[cplc->task_type]
  → if script->kind == Coroutine: TASK_CreateFiber(None, script->handler, 0)
  → if script->kind == Callback:  TASK_CreateCallback(None, script->handler)
  → entity = entity_create(script->mobd_id, task, NULL)
  → entity->x = cplc->x; entity->y = cplc->y; entity->z = cplc->z
  → entity->cplc_spawn_params = &cplc->spawn_params
```

### Viewport Streaming

As camera scrolls, boundary code walks X/Y sorted lists to find newly-visible `CplcEntity` nodes and calls `cplc_entity_init()` to spawn them on demand. Entities leaving viewport are NOT despawned — CPLC is one-way spawning.

---

## Naming Issues & Decompilation Artifacts

See [tasks-todo.md](tasks-todo.md) for naming suggestions, decompilation bugs, and flag field documentation.

---

## Key Globals

| Global | Type | Purpose |
|--------|------|---------|
| `g_task_active_head` | `Task*` | Head of doubly-linked active task list (sentinel-based) |
| `g_task_terminated_head` | `Task*` | Head of singly-linked free/recycled task list |
| `g_next_free_task_message` | `TaskMessage*` | Head of pre-allocated message pool free list |
| `g_coroutine_list_head` | `Coroutine*` | Head of coroutine scheduling list |
| `g_coroutine_default_stack_size` | `size_t` | Default fiber stack size |
| `g_task_creation_arg` | `Task*` | Temp: task being initialized (for `coroutine_starter`) |
| `g_task_creation_main` | `TaskFn` | Temp: handler function (for `coroutine_starter`) |
| `g_num_active_projectiles` | `int` | Active projectile count (max 200) |
| `g_scripts[]` | `ScriptType4*[]` | Script descriptors indexed by TaskType (binary data) |
| `g_script_handlers[352]` | `void*[]` | All handler fn ptrs for save/load serialization |
| `g_netz_synchronized_lockstep_mode` | `int` | Multiplayer sync flag — gates task execution |
| `g_game_update_loop_task` | `Task*` | Main game loop task — receives many point-to-point messages |
| `g_47A3CC` | `Task*` | Mission outcome controller task |
