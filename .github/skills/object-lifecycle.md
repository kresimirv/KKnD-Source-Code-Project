# KKND Object Pool Allocation Pattern

## Core Design

Game uses **pre-allocated pool + intrusive free list + doubly-linked active list** for all game objects. No per-object malloc/free at runtime — everything comes from fixed-size pool allocated at init.

## The Four Pointers

Every pooled object type uses up to 4 global pointers (or struct members for embedded pools like AiController):

| Suffix | Role | Notes |
|--------|------|-------|
| `xxx_pool` | Base pointer to malloc'd array | Used only for `free()` on cleanup. Never changes after init. |
| `xxx_free` / `xxx_free_list_head` / `xxx_free_pool` | Head of singly-linked free list | Nodes popped from here on alloc, pushed back on release. |
| `xxx_head` / `xxx_list_head` | Head of doubly-linked active list | First active object. |
| `xxx_tail` / `xxx_list_tail` | Tail of doubly-linked active list | Last active object. Sentinel trick (see below). |

### Naming Inconsistency (Decompilation Artifact)

Free list naming varies across subsystems — all mean the same thing:
- `g_aircraft_free_list_head` — full explicit name
- `g_oil_patch_free_pool` — "free_pool" variant  
- `g_viewports_free` — shortest form
- `g_sounds_free` — shortest form
- `ai->build_free` — struct member, short form
- `selection_free_pool` — struct member, "free_pool" variant
- `g_prod_free_list` — "free_list" without "head"
- `g_entity_free_pool_head` — mixed "free_pool_head"

**Canonical naming convention to adopt:** `xxx_free` for struct members, `g_xxx_free_list` for globals.

## Init Pattern

```c
BOOL xxx_list_init() {
    Type *v0 = (Type *)malloc(COUNT * sizeof(Type));
    g_xxx_pool = v0;
    if (!v0) return 0;
    
    // Wire free list: each node->next points to next element
    g_xxx_free = v0;  // free list starts at pool[0]
    for (int i = 0; i < COUNT - 1; ++i) {
        v0[i].next = &v0[i + 1];
        v0 = g_xxx_pool;  // decompiler artifact, reload base
    }
    g_xxx_pool[COUNT - 1].next = nullptr;  // terminate free list
    
    // Sentinel trick: head/tail both point to &head (empty circular list)
    g_xxx_head = (Type *)&g_xxx_head;
    g_xxx_tail = (Type *)&g_xxx_head;
    return 1;
}
```

## Alloc Pattern (Pop from free list)

```c
node = g_xxx_free;
if (g_xxx_free)
    g_xxx_free = g_xxx_free->next;
else
    node = nullptr;
```

## Insert into Active List (Prepend to head)

```c
node->next = g_xxx_head;
node->prev = (Type *)&g_xxx_head;
g_xxx_head->prev = node;
g_xxx_head = node;
```

## Remove from Active List + Return to Free List

```c
// Unlink from doubly-linked active list
node->next->prev = node->prev;
node->prev->next = node->next;

// Push onto free list (singly-linked)
node->next = g_xxx_free;
g_xxx_free = node;
```

## Cleanup Pattern

```c
void xxx_list_free() {
    free(g_xxx_pool);
}
```

## Sentinel Trick

Active list uses **self-referencing sentinel**: `head` and `tail` globals are adjacent in memory. On init:
```c
g_xxx_head = (Type *)&g_xxx_head;
g_xxx_tail = (Type *)&g_xxx_head;
```
This makes `&g_xxx_head` act as a dummy node where:
- `g_xxx_head` is the dummy's `next` field
- `g_xxx_tail` is the dummy's `prev` field

Empty list check: `g_xxx_head == (Type *)&g_xxx_head`

## Node Struct Requirements

Every pooled node struct must have `next` and `prev` as first two members (for doubly-linked active list). Free list reuses `next` only.

## Known Pool Instances

### Global pools (g_ prefix)
| Object | Pool | Free | Head | Tail |
|--------|------|------|------|------|
| Aircraft | `g_aircraft_pool` | `g_aircraft_free_list_head` | `g_aircraft_list_head` | `g_aircraft_list_tail` |
| OilPatch | `g_oil_patch_pool` | `g_oil_patch_free_pool` | `g_oil_patch_list_head` | `g_oil_patch_list_tail`* |
| BuildingsTracker | `g_buildings_tracker_pool` | `g_buildings_tracker_` | `g_buildings_tracker_list_head` | `g_buildings_tracker_list_tail` |
| Sidebar | `g_sidebar_pool` | `g_sidebar_free_list_head` | `g_siderbar_active_list_head` | `g_siderbar_active_list_tail` |
| SidebarButton | `g_sidebar_button_pool` | `g_sidebar_button_free_list_head` | `g_sidebar_button_active_list_head` | `g_sidebar_button_active_list_tail` |
| Coroutine | `g_coroutine_pool` | `dword_477350` (UNNAMED) | `g_coroutine_list_head` | — |
| RenderViewport | `g_viewports_pool` | `g_viewports_free` | — | — |
| GameEvent | `g_game_event_pool` | `g_game_event_free_pool` | — | — |
| Sound | `g_sounds_pool` | `g_sounds_free` | `g_sounds_head` | — |
| CplcEntityInViewport | `g_cplc_entities_in_viewport_pool` | — | `g_cplc_entities_in_viewport_head` | — |
| Entity | — | `g_entity_free_pool_head` | `g_entity_head` | — |
| Production | `g_prod_pool` | `g_prod_free_list` | — | — |
| ProductionOption | `g_prod_options_pool` | `g_prod_options_free_list` | — | — |
| UnitEscort | — | `g_escort_free_list_head` | — | — |
| TileCollisions | `g_tile_collisions_pool` | — | `g_tile_collisions_head` | — |

### AiController embedded pools (struct members, per-AI-player)
Each AI player's `AiController` struct contains multiple sub-pools for different AI node types:
- `build_pool` / `build_free` / `build_head` / `build_tail`
- `drillrig_pool` / `drillrig_free` / `drillrig_head` / `drillrig_tail`
- `tanker_pool` / `tanker_free` / `tanker_head` / `tanker_tail`
- `powerplant_pool` / `powerplant_free` / `powerplant_head` / `powerplant_tail`
- `enemy_pool` / `enemy_free` / `enemy_head` / `enemy_tail`
- Various unnamed `_ai_controller_XX_` pools following same pattern

### CursorState embedded pool (struct members)
- `selection_pool` / `selection_free_pool` / `selection_head` / `selection_tail`

## Decompilation Mistakes to Watch For

1. **`g_buildings_tracker_`** — truncated name, should be `g_buildings_tracker_free` or `g_buildings_tracker_free_list`
2. **`dword_477350`** — unnamed coroutine free list head, should be `g_coroutine_free`
3. **`g_oil_patch_list_tail`** — likely exists but may not have been found yet (adjacent to `g_oil_patch_list_head`)
4. **`enemy_tail` before `enemy_head`** in AiController — reversed order vs all other pairs, likely decompiler field offset misidentification
5. **Reload of pool base inside init loops** (`v0 = g_xxx_pool`) — decompiler artifact from optimized register reuse, not meaningful
