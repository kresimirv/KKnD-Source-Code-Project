# KKND Game State Machine

The game does NOT use a traditional switch-based state dispatcher. Instead, three nested `do/while` loops share a single guard variable `g_game_state`. Value `0` means "keep ticking current loop." Non-zero values break out and trigger the appropriate transition logic.

## GameState Enum (kknd.h L2214)

```c
enum KKND::GameState : unsigned __int32
{
    GameState_MainMenu       = 0,  // "keep running current loop"
    GameState_NextMission    = 1,  // "victory / load — advance campaign"  (UNNAMED in binary)
    GameState_CampaignEnd    = 2,  // "defeat / campaign over"             (UNNAMED in binary)
    GameState_PreviousScreen = 3,  // "exit to main menu / quit"
};
```

Only `0` and `3` have names in the decompilation. Values `1` and `2` are inferred from context.

## MissionOutcome Enum (kknd.h L2220)

```c
enum KKND::MissionOutcome : unsigned __int32
{
    MissionOutcome_NotStarted = 0,
    MissionOutcome_Defeat     = 1,
    MissionOutcome_Victory    = 2,
    MissionOutcome_Abandoned  = 6,
};
```

---

## Loop Architecture

Three nested loops, all using the same pattern:
```c
g_game_state = GameState_MainMenu;
do {
    // tick all subsystems
    GAME_AdvanceEntityAnimations();
    GAME_AdvanceEntityPositions();
    INPUT_ProcessKeyboard();
    INPUT_ProcessMouse();
    SOUND_ProcessSounds();
    FADE_Update();
    TASK_ExecuteScheduledTasks();  // ← tasks SET g_game_state to break out
    GAME_UpdateCamera();
    REND_DoFrame();
    TIME_tick();
} while (!g_os_quit_signal_received && g_game_state == GameState_MainMenu);
```

Tasks running inside `TASK_ExecuteScheduledTasks` set `g_game_state` to a non-zero value to signal the exit condition.

### Loop 1: Main Menu
Runs the menu UI (campaign select, multiplayer setup, options). Exits when player picks a mission or quits.

### Loop 2: Mission Complete Screen
Shown between super-level missions (levels 16–25). Displays score/stats, waits for player input.

### Loop 3: In-Game (RTS Gameplay)
The actual real-time strategy game. Ticks AI, unit logic, network sync. Exits on victory, defeat, surrender, or quit.

---

## Full Program Flow

```
PROGRAM START
    │
    ├── GAME_WaitScreen()        // Load engine, splash screen
    ├── MOVIE_Play(0)            // Intro movies (mh_fmv.vbc, intro.vbc)
    │
    ▼
┌─────────────────────────────────────────────────┐
│  LABEL_5: OUTER RESTART POINT                    │
│                                                  │
│  Setup multiplayer state                         │
│  GAME_PrepareSuperLvl(menuId)                    │
│                                                  │
│  ┌── LOOP 1: MAIN MENU ──┐                      │
│  │  Menu tasks running    │                      │
│  └────────────┬───────────┘                      │
│               │                                  │
│  ┌────────────┼───────────────────┐              │
│  │ state=3    │ state=1 or 2      │              │
│  │ (quit)     │ (start game)      │              │
│  ▼            ▼                   │              │
│ EXIT   ┌── CAMPAIGN LOOP ───────────────────┐    │
│        │                                    │    │
│        │  if (!loading && normal level):     │    │
│        │      MOVIE_Play(1) — briefing      │    │
│        │                                    │    │
│        │  if (super-level):                  │    │
│        │      ┌── LOOP 2: MISSION COMPLETE ─┐│   │
│        │      └──────────────────────────────┘│   │
│        │                                    │    │
│        │  sub_422F60() — load game level     │    │
│        │                                    │    │
│        │  ┌── LOOP 3: IN-GAME PLAY ────────┐│   │
│        │  │  RTS engine ticking             ││   │
│        │  └──────────────┬──────────────────┘│   │
│        │                 │                   │    │
│        │     ┌───────────┼───────────┐       │    │
│        │     │           │           │       │    │
│        │   st=1        st=2        st=3      │    │
│        │  (victory)   (defeat)  (quit/back)  │    │
│        │     │           │           │       │    │
│        │     └─────┬─────┘           │       │    │
│        │           ▼                 │       │    │
│        │     sub_423320()            │       │    │
│        │      │        │             │       │    │
│        │      │ret=1   │ret=0        │       │    │
│        │      │        │             │       │    │
│        │      ▼        ▼             │       │    │
│        │   continue   MOVIE_Play(2)  │       │    │
│        │   campaign   (ending movie) │       │    │
│        │      │        │             │       │    │
│        │      │        └─── goto LABEL_5 ────┼───►│
│        │      │                      │       │    │
│        │      └──► next iteration ───┘       │    │
│        │         (or st=3 → goto LABEL_5) ───┼───►│
│        └─────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

---

## State Transition Table

### Who Sets What

| Value | Set By | Context |
|-------|--------|---------|
| `0` (MainMenu) | Main loop code | At start of each `do/while` — reset for re-entry |
| `1` (NextMission) | Victory handler | Won a campaign mission (not final) |
| `1` (NextMission) | Victory handler | Won final mission (Surv_15/Mute_15) + sets `dword_47A18C=1` |
| `1` (NextMission) | Load game UI | Player loaded a saved game |
| `1` (NextMission) | Line 59243 | Another play-next trigger |
| `2` (CampaignEnd) | Defeat handler | Lost a mission / surrender |
| `2` (CampaignEnd) | Line 50725 | Multiplayer campaign end |
| `3` (PreviousScreen) | Demo victory | Demo build → back to menu |
| `3` (PreviousScreen) | Quit/Abandon button | In-game menu "Quit Game" |
| `3` (PreviousScreen) | UI menu buttons | Various menu "Back" actions |
| `3` (PreviousScreen) | Super-level victory | Bonus missions → back to menu |
| `3` (PreviousScreen) | Multiplayer end | All players done → back to menu |
| `3` (PreviousScreen) | Error/failsafe | Entity creation failure, fiber creation failure |
| `3` (PreviousScreen) | Line 36335 | Menu/task handler |

### Post-Mission Evaluation (`sub_423320`)

| Condition | Action | Returns |
|-----------|--------|---------|
| `state==1` + loading | Restore save, set level from save | `1` (continue) |
| `state==1` + normal level | Increment `g_current_lvl_id` | `1` (continue) |
| `state==1` + final mission | Set ending-movie flag | `0` (exit → MOVIE_Play(2)) |
| `state==2` + normal level | Preserve current level for retry | `1` (continue — replay) |
| `state==2` + super-level | Invalidate prev_level | `return state==2` → `1` |
| `state==3` / other | — | `0` (exit to main menu) |

---

## Victory/Defeat Handler (`sub_424CE0`)

The in-game popup task that shows "VICTORY" or "DEFEAT" and then sets state:

```c
// After displaying outcome animation:
if (g_is_demo_build)
    g_game_state = PreviousScreen;              // demo → menu

else if (defeat)
    g_game_state = 2;                           // campaign defeat

else if (normal campaign level)
    if (level == Surv_15 OR level == Mute_15)
        g_game_state = 1;                       // final mission won
        dword_47A3DC = 1;                       // trigger ending cutscene
    else
        sub_4224B0();                           // save progress/stats
        g_game_state = 1;                       // next mission

else (super-level / bonus)
    g_game_state = PreviousScreen;              // back to menu
```

---

## Movie Playback Points

| Call | Movies | When |
|------|--------|------|
| `MOVIE_Play(0)` | `mh_fmv.vbc`, `intro.vbc` | Game startup |
| `MOVIE_Play(1)` | Level's `fmv.lvl` (briefing) | Before each normal campaign mission |
| `MOVIE_Play(2)` | `survout.vbc` or `evolvout.vbc` | After winning Surv_15/Mute_15 (ending) |

---

## Game Loading Functions

| Function | Purpose |
|----------|---------|
| `GAME_WaitScreen()` | Init engine, load `wait.lvl` (splash) |
| `GAME_PrepareSuperLvl(menuId)` | Load super-level for menus (campaign select, etc.) |
| `sub_422F60()` | Load actual RTS mission (terrain, units, AI, sidebar) |
| `GAME_load()` | Restore saved game state from disk |
| `LVL_cleanup()` | Unload current level data |

---

## In-Game Exit Triggers

### Player-Initiated
| Action | Sets | Outcome |
|--------|------|---------|
| "Quit Game" (in-game menu) | `state=3`, `outcome=Abandoned` | Back to main menu |
| "Surrender" (single-player) | `state=2` | Defeat evaluation |
| ESC → Back (menus) | `state=3` | Back to previous screen |

### Game-Initiated
| Trigger | Sets | Outcome |
|---------|------|---------|
| Victory condition met | `state=1`, `outcome=Victory` | Next mission or ending |
| Defeat condition met | `state=2`, `outcome=Defeat` | Retry or menu |
| Network desync/error | `state=3` | Back to menu |
| Engine failure (entity/fiber) | `state=3` | Emergency exit |

---

## Key Design Patterns

1. **Single variable, nested loops**: Unlike typical game engines with an explicit state machine + switch dispatcher, KKND uses one `g_game_state` variable that all three nested loops share. Non-zero breaks the innermost active loop.

2. **Tasks drive transitions**: The loops themselves just tick subsystems. All actual game logic (victory checks, menu buttons, AI) runs inside `TASK_ExecuteScheduledTasks` and sets `g_game_state` when something decisive happens.

3. **Reuse of value 0**: "MainMenu" doesn't mean "we're in the main menu." It means "keep the current loop running." Every loop phase resets to 0 before entering.

4. **Campaign progression via `g_current_lvl_id++`**: Winning a mission simply increments the level index. The campaign is a linear sequence of level IDs (Surv_01 → Surv_15 or Mute_01 → Mute_15).

5. **Super-levels (16–25)**: These are menus/screens implemented as RTS levels (with entities, animations, sprites). The menu system IS the game engine running at a different "level."

6. **No pause state**: There's no `GameState_Paused`. Pausing is handled by stopping the tick (`TASK_ExecuteScheduledTasks` skipping), not by changing `g_game_state`.

---

## Naming Suggestions

| Current | Proposed | Rationale |
|---------|----------|-----------|
| `GameState_MainMenu` | `GameState_Running` | Means "keep current loop active," not specifically "main menu" |
| `(unnamed) 1` | `GameState_NextMission` | Advance campaign |
| `(unnamed) 2` | `GameState_Defeat` | Mission lost / campaign evaluation |
| `GameState_PreviousScreen` | `GameState_ExitToMenu` | Clearer intent |
| `sub_423320` | `GAME_PostMissionEvaluate` | Post-mission decision point |
| `sub_422F60` | `GAME_LoadLevel` | Loads actual gameplay level |
| `sub_424CE0` | `task_mission_outcome_popup` | Shows Victory/Defeat UI and sets state |
| `sub_4224B0` | `GAME_SaveCampaignProgress` | Saves stats after victory |
| `g_mission_outcome` | (keep) | Clear enough |
| `dword_47A18C` | `g_play_ending_movie` | Flag for MOVIE_Play(2) |
| `dword_47A3DC` | `g_ending_movie_triggered` | Set in victory handler for final mission |
| `GAME_PrepareSuperLvl` | `GAME_LoadMenuLevel` | Loads menu-as-level |
| `GAME_WaitScreen` | `GAME_InitAndSplash` | Init + splash display |

---

## Decompilation Issues

1. **`GameState_MainMenu` misnomer**: The enum name implies "menu state" but it actually means "current loop continues." This is confusing because ALL three loops (menu, mission-complete, gameplay) use the same value.

2. **Missing enum values 1 and 2**: IDA couldn't resolve names. The binary likely had `GameState_1` and `GameState_2` as literal constants without debug symbols.

3. **`sub_423320` overloaded logic**: Returns different values based on both `g_game_state` AND level type (normal vs super-level). Hard to follow without renaming.

4. **`dword_47A18C` vs `dword_47A3DC`**: Two separate "ending movie" flags. One is checked in MOVIE_Play(2), the other is set by the victory handler. Likely one is "should play" and the other is "has been triggered" but the relationship is unclear from decompilation alone.
