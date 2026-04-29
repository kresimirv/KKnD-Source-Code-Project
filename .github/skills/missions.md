# KKND Mission System

Missions are configured via the `g_lvl_desc[68]` table — a static array of `LevelDesc` structs indexed by `g_current_lvl_id`. Covers campaign (30 missions), multiplayer (15 maps + 3 duplicates), and expansion super-levels (20 missions).

## LevelDesc Struct (kknd.h L2295)

```c
struct KKND::LevelDesc {
    const char *lvl_filename;
    const char *wav_filename;               // background music WAV
    const char *vbc_filename;               // briefing movie (.vbc)
    __int16 player_starting_cash;
    __int16 ai_starting_cash;
    __int16 superlvl_ai_cash;               // AI cash for super-level allies
    __int16 superlvl_ai_cost_reduction;     // fixed-point 8.8 multiplier (256 = full price)
    __int16 max_tech_level;                 // 0-5, caps research tier
    __int16 _level_desc_field_16;           // unknown
    unsigned int disabled_units_mask;       // bits disable specific units/upgrades
    int victory_condition_bits;             // bitmask defining win/lose conditions
    AiAirstrikeConfig superlvl_ai_airstrike_count_enemy;
    AiAirstrikeConfig superlvl_ai_airstrike_count_ally;
};
```

---

## Level IDs

### Campaign (indices 0–29)

| Idx | ID | Name | File |
|-----|-----|------|------|
| 0 | `Surv_01` | The Next Generation | surv_01.lvl |
| 1 | `Surv_02` | Build An Outpost | surv_02.lvl |
| 2 | `Surv_03` | Withstand The Raiding Party | surv_03.lvl |
| 3 | `Surv_04` | Rescue The Scout | surv_04.lvl |
| 4 | `Surv_05` | Toll Gate | surv_05.lvl |
| 5 | `Surv_06` | Exterminate The Village | surv_06.lvl |
| 6 | `Surv_07` | Protect The Convoy | surv_07.lvl |
| 7 | `Surv_08` | Back To The Beach | surv_08.lvl |
| 8 | `Surv_09` | Rescue The General | surv_09.lvl |
| 9 | `Surv_10` | Occupation Force | surv_10.lvl |
| 10 | `Surv_11` | Give Me Liberty | surv_11.lvl |
| 11 | `Surv_12` | Surgical Strike | surv_12.lvl |
| 12 | `Surv_13` | Hold The Bridge | surv_13.lvl |
| 13 | `Surv_14` | The Rout Retreat | surv_14.lvl |
| 14 | `Surv_15` | Beseiged | surv_15.lvl |
| 15 | `Mute_01` | The Return Of The Slugs | mute_01.lvl |
| 16 | `Mute_02` | Repel Them Repel Them | mute_02.lvl |
| 17 | `Mute_03` | Bubblin Crude | mute_03.lvl |
| 18 | `Mute_04` | Raid The Fort | mute_04.lvl |
| 19 | `Mute_05` | Ambush | mute_05.lvl |
| 20 | `Mute_06` | Seige | mute_06.lvl |
| 21 | `Mute_07` | Release The Prisoners | mute_07.lvl |
| 22 | `Mute_08` | Smash The Convoy | mute_08.lvl |
| 23 | `Mute_09` | Fight For Territory | mute_09.lvl |
| 24 | `Mute_10` | It's The End Of The World | mute_10.lvl |
| 25 | `Mute_11` | Close Encounters | mute_11.lvl |
| 26 | `Mute_12` | Big Attack | mute_12.lvl |
| 27 | `Mute_13` | Battle For The Islands | mute_13.lvl |
| 28 | `Mute_14` | The Final Assault | mute_14.lvl |
| 29 | `Mute_15` | Counter Attack | mute_15.lvl |

### Multiplayer (indices 30–47)
| Idx | File | Notes |
|-----|------|-------|
| 30–44 | mlti_01..mlti_15.lvl | 15 unique maps |
| 45–47 | mlti_10.lvl (×3) | Duplicates (padding) |

### Expansion Super-Levels (indices 48–67)
| Idx | File | Campaign |
|-----|------|----------|
| 48–57 | surv_16..surv_25.lvl | Survivor Xtreme |
| 58–67 | mute_16..mute_25.lvl | Evolved Xtreme |

---

## Cash System

### Struct: `KKND::MissionCashTable` (kknd.h L2313)
```c
struct KKND::MissionCashTable {
    int cash[7];    // one per player slot (0=neutral, 1-6=players)
};
```
Global: `g_cash` at kknd.c L5615.

### Initialization
| Mode | Source |
|------|--------|
| Campaign (player) | `g_lvl_desc[level].player_starting_cash` |
| Campaign (AI enemy) | `g_lvl_desc[level].ai_starting_cash` |
| Campaign (AI ally, super-level) | `g_lvl_desc[level].superlvl_ai_cash` |
| Demo (AI) | `dword_464DC0[g_difficulty_mult]` |
| Kaos/Multiplayer | `atoi(off_46E420[dword_47C5FC])` — preset table cycled in lobby |

### Cash Table (Campaign)

| Level | Player | AI | Notes |
|-------|--------|-----|-------|
| surv_01 | 1000 | 5000 | |
| surv_02 | 4500 | 5000 | Economic mission |
| surv_03 | 1000 | 5000 | |
| surv_04 | **0** | 5000 | No starting cash (rescue mission) |
| surv_05 | 5000 | 100 | AI is weak |
| surv_06 | **0** | 50 | |
| surv_07 | 4000 | 5000 | Convoy escort |
| surv_08 | 5000 | 1000 | |
| surv_09 | 1000 | 100 | Rescue mission |
| surv_10 | 6500 | 1000 | |
| surv_11 | 5000 | 1000 | |
| surv_12 | **0** | 50 | Assassination mission |
| surv_13 | 4000 | 100 | |
| surv_14 | 5000 | 5000 | |
| surv_15 | 6000 | 1000 | Final mission |
| mute_01 | 5000 | 5000 | |
| mute_02 | 2600 | 1000 | |
| mute_03 | 2500 | 500 | |
| mute_04-15 | 5000-7000 | 50-5000 | Varies |

---

## Player System

### Player Slots
- **Player 0**: Neutral/Gaia — never an enemy, owns decorative units
- **Players 1–6**: Active slots. Human player defaults to `g_player_num = 1`
- Up to 6 AI players supported (one `AiController` each)

### Race Enum (kknd.h L2705)
```c
enum KKND::Race : unsigned __int32 {
    Race_Survivors = 1,
    Race_Evolved   = 2,
};
```

Each player has a `player_race` and each unit has `unit->player_num`. The AI controller stores `player_race` at offset 0x2A4.

---

## Diplomacy System

### Struct: `KKND::MissionDiplomacyTable` (kknd.h L2319)
```c
struct KKND::MissionDiplomacyTable {
    BOOL is_ally[7][7];    // is_ally[A][B] = 1 means B is allied to A
};
```
Global: `g_diplomacy` at kknd.c L5653.

### `diplomacy_is_enemy(player_num, unit)` (kknd.c L73251)
```c
BOOL diplomacy_is_enemy(int player_num, Unit *unit) {
    if (unit->type == UnitType_TechBunker) return FALSE;  // always capturable
    if (unit->type == UnitType_Hut)        return TRUE;   // always hostile
    other = unit->player_num;
    return other != 0 && !g_diplomacy.is_ally[player_num][other];
}
```

**Rules**:
- Player 0 (neutral) units are **never** enemies
- TechBunkers are **never** enemies (capturable by anyone)
- Huts are **always** enemies (raiding targets)
- Otherwise: enemy if not in ally table

### Alliance Formation
When AI detects a non-enemy unit: sets `g_diplomacy.is_ally[ai][other] = 1` and purges stale enemy entries. Alliance is **one-directional** per entry but typically set mutually during level setup.

---

## Victory Conditions

### Condition Bits (`victory_condition_bits`)

| Bit(s) | Hex | Name | Effect |
|--------|-----|------|--------|
| 0-1 | `0x03` | `VictoryBit_KillAllEnemies` | Win when no enemy combat units remain |
| 2 | `0x04` | `VictoryBit_Timer` | Win at 108000 ticks (≈60 min at 30 tps) |
| 4 | `0x10` | `VictoryBit_Cash` | Win if cash ≥ 5000; lose if bankrupt + no power |
| 8-9 | `0x300` | `VictoryBit_PlayerMustSurvive` | Lose if player has zero combat units |

### Common Combinations

| Value | Hex | Meaning | Used By |
|-------|-----|---------|---------|
| 771 | `0x303` | Kill all + player must survive | Most missions |
| 768 | `0x300` | Player must survive only (special objectives) | surv_04, surv_07 |
| 775 | `0x307` | Kill all + timer win | surv_14 |
| 787 | `0x313` | Kill all + cash win/lose | surv_02 |
| 1795 | `0x703` | Kill all + extra flag (0x400) | surv_09 |

### Victory Task (`sub_425400`)

Runs every 30 ticks. Checks:
1. `sub_4250E0()` / `sub_424FD0()` — scans `g_unit_list_head` for combat-relevant units (type ≤ 66 or 74–78)
2. Returns 2-bit result: bit 0 = player has units, bit 1 = enemy has units
3. If player dead (bit 0 == 0) + `0x300` set → **DEFEAT**
4. If cash check active + cash ≥ 5000 → **VICTORY**
5. If cash check active + bankrupt + no power station → **DEFEAT**
6. If enemy dead (bit 1 == 0) + kill-all active → **VICTORY**
7. If timer active + ticks ≥ 108000 → **VICTORY**

### Per-Mission Death Conditions (`mission_check_on_death_victory_conditions`)

Called on every unit death. Mission-specific critical-unit checks:

| Level | Trigger | Result |
|-------|---------|--------|
| surv_02 | Derrick/Drillrig/PowerStation/Tanker dies | DEFEAT |
| surv_03 | Derrick/Drillrig/PowerStation/Outpost dies | DEFEAT |
| surv_04 | Scout dies | DEFEAT |
| surv_07 | 3+ convoy tankers destroyed | DEFEAT |
| surv_09 | General dies | DEFEAT |
| surv_12 / mute_12 | Warlord dies | **VICTORY** (assassination!) |
| mute_02 | Mute_Drillrig dies | DEFEAT |
| mute_07 | 5+ convoy tankers destroyed | **VICTORY** (destroy convoy!) |
| mute_13 | King Zog dies | DEFEAT |
| surv_18 (super) | El Presidente dies | DEFEAT |

---

## Tech Level System

`max_tech_level` in `LevelDesc` — caps available research/upgrades:

| Value | Available |
|-------|-----------|
| 0 | No production buildings available |
| 1 | Basic infantry only |
| 2 | Basic vehicles unlocked |
| 3 | Mid-tier units (RPG, tanks) |
| 4 | Advanced units (heavy vehicles) |
| 5 | All tech unlocked (superweapons) |

### `disabled_units_mask` — Per-unit Disable Bits

Bitmask where set bits disable specific units/upgrades for that level. Checked against each buildable option's mask:
```c
if ((g_lvl_desc[level].disabled_units_mask & option_mask) == 0)
    // unit available
```

Notable levels with restrictions:
| Level | Mask | Meaning |
|-------|------|---------|
| surv_02 | `0x1C001F` | Many basic units disabled (economic focus) |
| surv_03 | `0x0B001D` | Limited early units |
| surv_05 | `0x000010` | One specific unit disabled |
| surv_21 | `0x002000` | One mid-tier disabled |
| mute_02 | `0x000077` | Heavy restrictions (defensive mission) |
| mute_03 | `0x000015` | Some early units disabled |
| mute_22 | `0x002000` | Same as surv_21 |

---

## Kaos Mode (Multiplayer/Skirmish)

"Kaos" is KKND1's name for multiplayer/skirmish. Uses `TaskChannel_KaosMenu` (8) and `TaskChannel_KaosLobbyWatch` (9).

### Key Differences from Campaign
| Aspect | Campaign | Kaos |
|--------|----------|------|
| Level selection | Linear `g_current_lvl_id++` | Player picks from 15 maps |
| Cash | `player_starting_cash` from table | Preset table cycled in lobby |
| Briefing FMV | `vbc_filename` per level | None (NULL) |
| Victory | Per-level `victory_condition_bits` | Standard kill-all (0x303) |
| Tech level | Varies (0–5) per level | Always 5 (all unlocked) |
| Disabled units | Per-level mask | 0 (all available) |
| AI | Per-level AI type | General AI only |
| Players | 1 human + N AI | Up to 6 players (network) |

### Kaos Lobby Flow
1. Host opens lobby (`MenuId_Kaos = 15`)
2. Map selection: cycles `off_46C318[0..14]` (15 maps)
3. Faction pick: Survivors or Evolved via `g_demo_faction`
4. Cash preset: `off_46E420[dword_47C5FC]` — 8 presets cycled with UI button
5. Start: `sub_441CE0` launches game. Solo mode if `dword_47C6C0 == 15`
6. Network sync: fiber tasks coordinate start across all players

### Kaos Maps (indices 30–44)
All use: `player_cash=5000, ai_cash=5000, tech=5, disabled=0, victory=0x303`

---

## Mission Briefings

### FMV System
Each campaign level has a `vbc_filename` in `LevelDesc`:
- Survivor missions: `"heads01.vbc"` through `"heads15.vbc"`
- Evolved missions: `"headm01.vbc"` through `"headm15.vbc"`
- Super-levels: `NULL` (no briefing)
- Multiplayer: `NULL`

### Playback
`MOVIE_Play(1)` before each normal campaign mission:
1. Loads `fmv.lvl` from level directory (contains briefing text/graphics)
2. Plays corresponding `.vbc` file (talking head video)
3. Uses 320×240 viewport
4. Skipped if loading a save or entering a super-level

---

## Airstrike Configuration

Only super-levels (48–67) have airstrikes. Config per-level:

```c
struct AiAirstrikeConfig {
    __int16 count;           // total strikes allowed
    __int16 interval;        // initial countdown (ticks)
    unsigned __int16 rng_interval_min;
    unsigned __int16 rng_interval_max;
};
```

Reset formula: `new_interval = (rng_min + rand(rng_max)) >> 2`

| Level | Count | Initial | Min | Max |
|-------|-------|---------|-----|-----|
| surv_24 | 10 | 40000 | 25000 | 3000 |
| surv_25 | 10 | 30000 | 25000 | 3000 |
| mute_23 | 6 | 35000 | 30000 | 3000 |
| mute_24 | 4 | 19000 | 17000 | 1000 |
| mute_25 | 30 | 30000 | 25000 | 3000 |

---

## Naming Suggestions

| Current | Proposed | Rationale |
|---------|----------|-----------|
| `g_lvl_desc` | `g_level_table` | Clearer |
| `victory_condition_bits` | `victory_flags` | It's a bitmask |
| `_level_desc_field_16` | `unknown_field_16` | Unknown |
| `disabled_units_mask` | `unit_disable_mask` | Consistent casing |
| `sub_425400` | `task_victory_check` | Victory polling task |
| `sub_4250E0` | `victory_count_combat_units` | Standard variant |
| `sub_424FD0` | `victory_count_combat_units_superlvl` | Ally-aware variant |
| `mission_check_on_death_victory_conditions` | `mission_on_unit_death_check` | Shorter |
| `dword_47A3D0` | `g_victory_flags_cached` | Cached from LevelDesc |
| `dword_47DC68` | `g_mission_timer_ticks` | Tick counter for timer victories |
| `g_num_convoy_tankers_destroyed` | (keep) | Already named |
| `g_diplomacy` | (keep) | Clear |
| `diplomacy_is_enemy` | (keep) | Clear |
| `get_current_level_starting_cash` | `GAME_GetStartingCash` | Function naming style |
| `off_46E420` | `g_kaos_cash_presets` | String array of cash values |
| `off_46C318` | `g_kaos_map_names` | String array of map filenames |
| `dword_47C5FC` | `g_kaos_cash_preset_idx` | Current cash preset selection |
| `byte_47C654` | `g_kaos_map_idx` | Current map selection |

---

## Key Design Patterns

1. **Table-driven missions**: All config in one 68-entry static array. No per-mission code except death conditions.

2. **Victory as task**: A persistent coroutine polls every 30 ticks rather than checking synchronously. Death conditions use a hook callback for immediate response.

3. **Bitmask composition**: Victory bits compose orthogonally — any combination of kill-all + timer + cash + survive. Each bit adds an independent condition.

4. **Diplomacy is runtime-mutable**: `g_diplomacy.is_ally` can change during play (AI auto-alliances). Not just initial setup.

5. **Super-levels reuse mission infrastructure**: Expansion levels (48–67) are full missions that just happen to be accessed from the "Xtreme" menu rather than the linear campaign.

6. **Kaos = campaign engine minus story**: Same game loop, same victory task, same AI — just with all tech unlocked, preset cash, and no FMV.

7. **Player 0 immunity**: Neutral units can never be targeted by AI (diplomacy always returns "not enemy"). Decorative/environmental entities use player 0.

8. **Asymmetric death conditions**: Some missions trigger DEFEAT on critical unit death (protection missions), others trigger VICTORY (assassination missions like surv_12/mute_07). Same hook, different outcome direction.
