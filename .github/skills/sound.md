# KKND Sound System

DirectSound-based audio with 8 fixed sound slots, flag-based thread synchronization, logarithmic volume/pan, and positional 3D sound via viewport-relative attenuation. Two modes: in-memory one-shot SFX and streaming background music via per-slot worker threads.

## Core Architecture

### DirectSound Setup
- `DirectSoundCreate(NULL, &g_ds, NULL)` — single IDirectSound interface
- Cooperative level: `DSSCL_NORMAL` (shares with other apps)
- Buffer flags: `DSBCAPS_CTRLVOLUME | DSBCAPS_CTRLPAN | DSBCAPS_STATIC | DSBCAPS_GETCURRENTPOSITION2` (= 232)

### Sound Slot Pool
8 pre-allocated `stru7_sound` slots (328 bytes each = 2624 total). Two linked lists:
- **Active list**: doubly-linked with sentinel node, holds playing/streaming sounds
- **Free list**: singly-linked via `next` pointer

### Global State
| Variable | Purpose |
|----------|---------|
| `g_ds` | LPDIRECTSOUND main interface |
| `g_sounds_pool` | Base allocation (8 × 0x148) |
| `g_sounds_head` | Active list head |
| `g_sounds_free` | Free list head |
| `g_sound_volumes[17]` | Log2 volume LUT (millibels) |
| `g_sound_pans[33]` | Log2 pan LUT (16 left + 1 center + 16 right) |
| `g_4690A8_sfx_volume` | Master SFX volume (default 16 = max) |
| `g_num_active_projectiles` | (used to limit SFX spawning) |
| `dword_47C5D4` | Current streaming sound slot ID |
| `dword_47C5D0` | Global pause flag |
| `dword_47C4E0` | Sound bank data pointer (level-loaded PCM) |
| `dword_47C5C0` | Sound bank loaded flag |

---

## Struct: `KKND::stru7_sound` (→ `SoundSlot`)

| Offset | Field | Proposed Name | Purpose |
|--------|-------|--------------|---------|
| 0x00 | `_stru7_sound_0` | `id` | Unique monotonic ID |
| 0x04 | `dsb` | `buffer` | IDirectSoundBuffer* |
| 0x08 | `_stru7_sound_8` | `owner_task` | Task for message callbacks |
| 0x0C | `file` | `stream_file` | File handle (streaming only) |
| 0x10 | `_stru7_sound_10` | `thread_handle` | _beginthread result |
| 0x14 | `_stru7_sound_14` | `type` | -2 = memory, -3 = streaming |
| 0x18 | `_stru7_sound_18` | `deferred_play_count` | Paused → play when unpaused |
| 0x1C | `volume` | `volume_index` | Current volume LUT index (0–16) |
| 0x20 | `_stru7_sound_20` | `vol_transition_ticks` | Fade ticks remaining |
| 0x24 | `_stru7_sound_24` | `vol_transition_step` | Volume step per tick (signed) |
| 0x28 | `pan` | `pan_index` | Current pan LUT index (0–32) |
| 0x2C | `_stru7_sound_2C` | `pan_transition_ticks` | Pan transition ticks remaining |
| 0x30 | `_stru7_sound_30` | `pan_transition_step` | Pan step per tick (signed) |
| 0x34 | `_stru7_sound_34` | `flags` | Bitfield (see below) |
| 0x38 | `next` | `next` | Next in list |
| 0x3C | `_stru7_sound_3C` | `prev` | Prev in list |
| 0x40 | `_stru7_sound_40` | `bytes_remaining` | Streaming: bytes left to read |
| 0x44 | `filename[32]` | `filename` | WAV path (streaming) |

### Flag Bits (`_stru7_sound_34`)
| Bit | Value | Meaning |
|-----|-------|---------|
| 2 | 0x04 | Paused |
| 3 | 0x08 | Busy (thread accessing buffer) |
| 4 | 0x10 | Loop mode |
| 5 | 0x20 | Stop requested / end-of-stream |
| 6 | 0x40 | Thread exited / done |

---

## Function Catalog

### Initialization & Cleanup

| Function | Line | Proposed Name | Purpose |
|----------|------|--------------|---------|
| `SOUND_init` | 54736 | `SOUND_init` | Create DirectSound, alloc pool, build LUTs |
| `SOUND_cleanup` | 55722 | `SOUND_cleanup` | Stop all, release DS, free pool |
| `sub_43A320` | 55573 | `SOUND_unload_bank` | Stop all sounds, free level sound bank (level transition) |

### Playback

| Function | Line | Proposed Name | Purpose |
|----------|------|--------------|---------|
| `SOUND_play` | 54870 | `SOUND_play` | Play in-memory SFX from sound bank |
| `sub_439AA0` | 55090 | `SOUND_play_stream` | Start streaming WAV from file |
| `SOUND_thread` | 55161 | `SOUND_stream_thread` | Background thread — fills ring buffer |
| `entity_408800_play_sound` | 12417 | `SOUND_play_positional` | Play with viewport-relative pan/volume |

### Control

| Function | Line | Proposed Name | Purpose |
|----------|------|--------------|---------|
| `SOUND_ProcessSounds` | 55589 | `SOUND_tick` | Per-frame: transitions, cleanup finished |
| `sub_43A2B0` | 55510 | `SOUND_stop` | Stop/pause a sound by slot ID |
| `sub_43A800` | 55876 | `SOUND_set_stream_volume` | Set volume on current streaming sound |
| `sub_43A850` | 55902 | `SOUND_destroy_slot` | Teardown slot: signal thread, release buffer, return to free list |

### Helpers

| Function | Line | Proposed Name | Purpose |
|----------|------|--------------|---------|
| `sub_43A710` | 55833 | `SOUND_parse_wav_memory` | Parse in-memory RIFF → format + data ptrs |
| `FILE_read_waveformatex` | 55761 | `SOUND_parse_wav_file` | Parse WAV file → WAVEFORMATEX + data size |

---

## Volume System

### Logarithmic LUT
17-entry table computed at init: `g_sound_volumes[i] = log2(i * 0.0588) * 1000`

Index 0 = silence (-10000 dB), index 16 = full volume (0 dB). Master volume `g_4690A8_sfx_volume` defaults to 16.

### Volume Transitions (Fades)
Per-tick stepped fade. Each tick:
1. Read current hardware volume via `GetVolume`
2. Reverse-map to nearest LUT index (binary search by min abs diff)
3. Add `vol_transition_step` (signed index offset)
4. Set new volume from LUT
5. Decrement `vol_transition_ticks`; at 0, send `TaskMessage_SoundVolumeTransitionComplete`

### Pan System
33-entry table (0=hard left, 16=center, 32=hard right). Same log2-scaled millibel values, positive for left bias, negative for right.

Pan transitions work identically to volume transitions.

---

## Positional Audio (`SOUND_play_positional`)

Viewport-relative stereo positioning:

### Pan Calculation
```
half_viewport_w = max(viewport->clip_w << 7, 256)
offset_x = entity->x - camera.x - half_viewport_w

if |offset_x| <= half_viewport_w:  (on-screen)
    pan = 16 ± 16 * (|offset_x| >> 8) / (half_viewport_w >> 8)
if offset_x < -half_viewport_w:    (off-screen left)
    pan = 0, volume *= (offset_x + 2*half_viewport_w) / half_viewport_w
if offset_x > half_viewport_w:     (off-screen right)
    pan = 32, volume *= (2*half_viewport_w - offset_x) / half_viewport_w
```

### Volume Attenuation (Y-axis)
```
half_viewport_h = viewport->clip_h << 7
offset_y = entity->y - camera.y - half_viewport_h

if off-screen vertically:
    volume *= (clip_h<<8 - |offset_y|) / half_viewport_h
```

Final volume clamped to `[2, g_4690A8_sfx_volume]`.

---

## Streaming Model

Used for background music (level WAV files loaded from `g_lvl_desc[level].wav_filename`).

### Ring Buffer
Size: `3 × wfx.nAvgBytesPerSec` (3 seconds of audio at native sample rate).

### Thread Flow
```
SOUND_play_stream(filename, loop, volume, pan, task)
    ↓
_beginthread(SOUND_stream_thread, 0x1000, slot)
    ↓
Open file → Parse RIFF/WAVE header
    ↓
CreateSoundBuffer (3s ring buffer)
    ↓
Initial fill: Lock → Read → Unlock → Play(LOOPING)
    ↓
Main loop (100ms Sleep intervals):
    ├── Check stop flag (0x20) → break
    ├── Check pause flag (0x04) → skip fill
    ├── Set busy (0x08)
    ├── GetCurrentPosition → calculate fillable region
    ├── Lock → FILE_read → Unlock
    ├── Clear busy (0x08)
    └── If EOF:
        ├── If looping (0x10): seek to start, continue
        └── If one-shot: drain remaining buffer → Stop → cleanup
    ↓
Set done flag (0x40) → _endthread()
```

### Thread Termination
Main thread sets `0x20` flag, then busy-waits (`Sleep(20ms)` loop) until `0x40` appears. No mutexes/critical sections — pure flag-based synchronization.

---

## Sound Categories

### Weapon Fire Sounds
| SoundId | Proposed Name | Weapon |
|---------|--------------|--------|
| 3 | `SFX_Explosion_Small` | Generic explosion, building debris |
| 4 | `SFX_Rifle_Default` | Default rifle/generic ballistic |
| 5 | `SFX_MG_Vehicle_Light` | DirkBike, 4x4 Pickup, Monster Truck |
| 6 | `SFX_Autocannon` | Autocannon Tank |
| 7 | `SFX_MG_Sidecar` | Bike & Sidecar |
| 8 | `SFX_MG_Heavy` | Crazy Harry, ATV |
| 9 | `SFX_Explosion_Medium` | Bombs, aircraft crash, generic projectile |
| 10 | `SFX_Explosion_Large` | Nuke detonation, large impacts |
| 14 | `SFX_Rocket_Launch` | Rocket/missile fire |
| 17 | `SFX_Shotgun` | Shotgunner, Dire Wolf |
| 86 | `SFX_Flamethrower` | Flame particles |
| 87 | `SFX_Plasma` | Plasma Tank, Sentinel Droid, Mech plasma |
| 89 | `SFX_Bow_Fire` | Arrow launch |
| 90 | `SFX_Bow_Impact` | Arrow hit |
| 161 | `SFX_Acid_Spit` | Giant Beetle, Giant Scorpion |
| 163 | `SFX_Nuke_Launch` | Beast Enclosure nuke |
| 188 | `SFX_Bullet_Whiz` | Ricochet/whiz (33% chance) |

### UI/Event Sounds
| SoundId | Name | Context |
|---------|------|---------|
| 33 | `SFX_Surv_UnitReady` | Survivor unit production complete |
| 47 | `SFX_Surv_LowOilWarning` | Low oil notification |
| 54 | `SFX_Surv_ResearchStarted` | Research begun |
| 122 | `SFX_Mute_LowOilWarning` | Mutant low oil |
| 125 | `SFX_Mute_UnitReady` | Mutant production complete |
| 126 | `SFX_Mute_ResearchStarted` | Mutant research begun |
| 189 | `SFX_MobileBase_Plant` | Mobile base deploying |

### Unit Recall/Acknowledge Sounds
| SoundId | Name | Unit |
|---------|------|------|
| 160 | `SFX_Mute_WarMastadont_Recall` | War Mastadont selected/ordered |
| 165 | `SFX_Mute_DireWolf_Recall` | Dire Wolf |
| 168 | `SFX_Mute_GiantBeetle_Recall` | Giant Beetle |
| 170 | `SFX_Mute_MissileCrab_Recall` | Missile Crab |
| 172 | `SFX_Mute_GiantScorpion_Recall` | Giant Scorpion |
| 181 | `SFX_FuturisticUnit_Recall_1` | Futuristic tech unit acknowledge 1 |
| 182 | `SFX_FuturisticUnit_Recall_2` | Futuristic tech unit acknowledge 2 |

### Sound Lookup Tables (Random Selection)
| Table | Contents | Usage |
|-------|----------|-------|
| `dword_46BB80[2]` | {SoundId_9, SoundId_10} | Random explosion on grenade/rocket impact |
| `dword_46BB70[3]` | Likely {SoundId_9, SoundId_10, SoundId_3} | Random ricochet/impact variants |
| `dword_46BB60[3]` | Animation indices (not sounds) | Random explosion visual variants |

---

## Streaming Music Playback

Level music loaded from: `{app_root}\LEVELS\{g_lvl_desc[level].wav_filename}`

Call pattern:
```c
sprintf(Buffer, "%s\\LEVELS\\%s", g_app_root, g_lvl_desc[g_current_lvl_id].wav_filename);
sub_439AA0(Buffer, 1, dword_4690AC, 16, 0);  // loop=1, volume=master, pan=center
```

Music starts on level load, loops indefinitely (flag `0x10`), plays at master volume with center pan.

---

## Task Message Callbacks

| Message | Value | Trigger |
|---------|-------|---------|
| `TaskMessage_SoundPanTransitionComplete` | 0xFFFFFFFB (-5) | Pan fade finished |
| `TaskMessage_SoundVolumeTransitionComplete` | 0xFFFFFFFC (-4) | Volume fade finished |
| `TaskMessage_SoundCleanupComplete` | 0xFFFFFFFD (-3) | Sound slot destroyed |

Sent to `owner_task` stored in slot offset 0x08. Used by game code to chain sound events (e.g., fade out → play next).

---

## Animation-Triggered Sounds

`MobdAnimFrame` struct (kknd.h L1600-1612) contains `SoundId sound_id` field. When animation system advances to a frame with `sound_id != 0`, it calls `entity_408800_play_sound` on that entity. This powers footstep sounds, attack grunts, and movement effects tied to specific animation frames.

---

## Key Design Patterns

1. **Fixed pool, no dynamic allocation**: 8 slots max. If all slots busy, `SOUND_play` returns 0 (sound dropped). No priority system.

2. **Flag-based thread sync**: No mutexes. Main thread signals via bit 0x20, polls bit 0x40. Thread signals via bit 0x08 (busy). Race window exists between flag check and buffer access — tolerated because DirectSound Lock/Unlock are themselves serialized.

3. **Coroutine-safe**: `SOUND_play` increments a re-entrant counter at entry (prevents coroutine interleaving during linked-list manipulation).

4. **Two sound sources per unit**: Units have `stats->projectile->task_fn` which plays fire sound, AND `MobdAnimFrame->sound_id` which plays anim-synced sounds. These are independent channels.

5. **Global pause**: `dword_47C5D0` flag defers all `Play()` calls. Sounds are queued (deferred_play_count incremented). On unpause, all deferred sounds trigger at once.

6. **Level-scoped sound bank**: PCM data loaded with level into `dword_47C4E0`. `SOUND_play` indexes directly into this bank. `SOUND_unload_bank` frees on level transition without tearing down DirectSound itself.

7. **Volume as index, not dB**: All game code passes volume as index (0–16). Only at the last moment does it convert via LUT to actual millibels for DirectSound. This keeps game logic integer-only.

8. **SoundId_17 anomaly**: Enum has `SoundId_17 = 117` (not 17). This is likely an IDA decompilation bug — the original value was probably 17 but IDA assigned the wrong constant. The shotgun fire code uses this value.

---

## Naming Conventions

| Current Name | Proposed Name | Rationale |
|--------------|--------------|-----------|
| `stru7_sound` | `SoundSlot` | It's a slot in the sound pool |
| `sub_439AA0` | `SOUND_play_stream` | Starts streaming playback |
| `SOUND_thread` | `SOUND_stream_thread` | Background streaming worker |
| `sub_43A2B0` | `SOUND_stop` | Stops a sound by ID |
| `sub_43A800` | `SOUND_set_stream_volume` | Sets volume on current stream |
| `sub_43A850` | `SOUND_destroy_slot` | Full teardown of a slot |
| `sub_43A320` | `SOUND_unload_bank` | Level transition cleanup |
| `sub_43A710` | `SOUND_parse_wav_memory` | In-memory RIFF parser |
| `FILE_read_waveformatex` | `SOUND_parse_wav_file` | File-based RIFF parser |
| `entity_408800_play_sound` | `SOUND_play_positional` | 3D positional sound |
| `g_4690A8_sfx_volume` | `g_sfx_master_volume` | Master volume index |
| `dword_47C5D4` | `g_current_stream_id` | Active streaming slot ID |
| `dword_47C5D0` | `g_sound_paused` | Global pause flag |
| `dword_47C4E0` | `g_sound_bank_data` | Level PCM data pointer |
| `dword_47C5C0` | `g_sound_bank_loaded` | Bank loaded flag |
| `_stru7_sound_0` | `id` | Slot unique ID |
| `_stru7_sound_8` | `owner_task` | Callback target |
| `_stru7_sound_10` | `thread_handle` | Worker thread |
| `_stru7_sound_14` | `type` | -2=memory, -3=stream |
| `_stru7_sound_18` | `deferred_play_count` | Paused queue count |
| `_stru7_sound_20` | `vol_fade_ticks` | Volume transition remaining |
| `_stru7_sound_24` | `vol_fade_step` | Volume step/tick |
| `_stru7_sound_2C` | `pan_fade_ticks` | Pan transition remaining |
| `_stru7_sound_30` | `pan_fade_step` | Pan step/tick |
| `_stru7_sound_34` | `flags` | State bitfield |
| `_stru7_sound_3C` | `prev` | Doubly-linked list prev |
| `_stru7_sound_40` | `bytes_remaining` | Stream bytes left |
