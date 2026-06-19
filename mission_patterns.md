# What I Learned Studying 6 Artemis Cosmos Missions

Patterns and techniques discovered by reading: HereThereBeMonsters, Infinite_Cosmos, WalkTheLine, Lucky_13, theta_quadrant, and MiningDays — in addition to the canonical SecretMeeting example.

---

## 1. World-Building: The Tile Map System

HereThereBeMonsters and theta_quadrant use a tile-map system for structured procedural worlds. You define named "decks" of terrain prefabs, map them to characters, and fill from an ASCII art string. theta_quadrant creates 16 named sectors (Rhapsody, Carina, Polaris, etc.) this way:

```mast
tile_map = maps_tile_map_create(0, 100_000, 1_000, 2_000)

nebula_deck = maps_deck_create()
nebula_deck.add_card(prefab_terrain_nebula_sphere, {"density": 0.5, "cluster_color": "red"})

black_hole_deck = maps_deck_create()
black_hole_deck.add_card(prefab_black_hole, {"gravity_radius": 10_000})

tile_map.map_deck("n", nebula_deck)
tile_map.map_deck("b", black_hole_deck)

res = await tile_map.fill(map_art, x_count=100)
```

The ASCII art string defines layout. Spaces and unmapped characters are skipped. Cell sizes can vary (5,000–25,000 units) for different density regions.

---

## 2. Subfolder Organization for Large Missions

Large missions keep `story.mast` as a minimal entry point and move `@map/` labels into a `maps/` subfolder. Infinite_Cosmos, Lucky_13, and MiningDays all do this. External Python helper modules (`.py` files in the mission folder) can be imported directly from MAST since the mission directory is on `sys.path`.

```
MyMission/
├── story.mast                  # minimal — just imports
├── story.json
├── maps/
│   ├── mission_map.mast        # contains the @map/ label
│   └── prefabs.mast            # local prefab labels
├── terrain_helpers.py          # importable Python utilities
└── mission_helpers.py
```

---

## 3. Terrain: Higher-Level Helper Functions

Beyond the manual `terrain_spawn()` loop from SecretMeeting, there are procedural helpers that handle scatter math internally:

```mast
# Asteroid fields
terrain_spawn_asteroid_box(cx, cy, cz, size_x=40000, size_z=15000, density_scale=3.0, density=3, height=2000)
terrain_spawn_asteroid_sphere(cx, cy, cz, radius=7500, density_scale=2.0, density=2, height=2000)
terrain_spawn_asteroid_scatter(points, height=500)   # from a pre-computed point list

# Nebula
terrain_spawn_nebula_sphere(cx, cy, cz, radius=12000, cluster_color="red")

# Ring scatter — useful for minefields
points = scatter_ring(width, depth, cx, cy, cz, inner_r, outer_r, start_angle, end_angle)
for v in points:
    mine = terrain_spawn(v.x, v.y, v.z, None, None, "danger_1a", "behav_mine")
    mine.data_set.set("damage_done", 5)
    mine.data_set.set("blast_radius", 1000)
```

---

## 4. New MAST Language Features

### `promise_any` — Race two promises

Resolves with whichever promise finishes first. The canonical use is "button OR timeout":

```mast
choice = gui_button("Confirm")
result = await promise_any(choice, delay_sim(10))
```

If the player doesn't click within 10 seconds, the timeout wins.

### `for x while condition:` — Loop with exit condition

```mast
for x while d > 1000:
    await delay_sim(4)
    ->END if to_object(artemis_id) is None
    d = sbs.distance_id(artemis_id, target_id)
```

### `on change <expression>:` — Reactive GUI handlers

`on change` fires whenever an expression's value changes — it's not limited to simple variable names. theta_quadrant uses this to swap an alert image the moment the ship's data value changes:

```mast
on change get_data_set_value(ship_id, "red_alert", 0):
    image = "RedAlert" if get_data_set_value(ship_id, "red_alert", 0) >= 1 else "AllClear"
    on_screen.update(f"image:{get_mission_dir_filename(image)}")
    gui_represent(on_screen)
```

### `mast_task` — Current task reference

Inside a label, `mast_task` is the running task object. Store it in a shared variable, and external code (e.g. a debug comms menu) can call `.jump("label_name")` to redirect it:

```mast
shared main_story_task = mast_task   # store in @map body

# From a debug route:
main_story_task.jump("scene_two")    # skip to any chapter
```

---

## 5. Science Console: `extra_scan_source` Links

Link NPC objects to a player ship so the science officer can scan them:

```mast
link(artemis_id, "extra_scan_source", whale_watcher_id)
link(artemis_id, "extra_scan_source", friendly_station_id)
```

Multiple objects can be linked to the same ship. HereThereBeMonsters uses this heavily to surface story lore through science scans.

---

## 6. Lifeform Hosting

A lifeform NPC can be hosted aboard a player ship as crew:

```mast
ensign_rachel.host = artemis_id
```

The lifeform then appears in interior views and comms associated with that ship.

---

## 7. Audio

HereThereBeMonsters plays audio files synchronized to comms scenes:

```mast
sbs.play_audio_file(0, get_mission_audio_file("audio/SD02C0166"), 1.0, 1.0)
```

`get_mission_audio_file(path)` resolves relative to the mission's `media/` folder. The mission controls audio with a shared config flag so players can disable it:

```mast
shared AUDIO_ENABLED = True
if AUDIO_ENABLED:
    sbs.play_audio_file(0, get_mission_audio_file(audio_path), 1.0, 1.0)
```

---

## 8. Debug Pattern (`debug.mast`)

HereThereBeMonsters separates all dev-only code into a `debug.mast` file. It stores the main story task in a shared variable and builds a comms menu that can jump to any narrative chapter — invaluable for testing long missions:

```mast
# In @map body (story.mast):
shared main_story_task = mast_task

# In debug.mast:
//comms/chapters if main_story_task
    + "scene_distress_call_one":
        main_story_task.jump("scene_distress_call_one")
    + "salvage":
        main_story_task.jump("salvage")
    + "retrieve_recording":
        main_story_task.jump("retrieve_recording")

//enable/comms if is_dev_build()    # gate behind dev build check
```

---

## 9. Console Filtering with `linked_to`

`linked_to(ship_id, "consoles")` returns the set of console clients linked to a specific ship. Combine with role sets to target messages precisely:

```mast
consoles = linked_to(artemis_id, "consoles") & all_roles("console, comms")
for c in to_object_list(consoles):
    sbs.send_story_dialog(c.client_id, name, message, face, "#444")
```

---

## 10. Custom Console Labels

Missions can add new `@console/` labels beyond the LegendaryMissions defaults. theta_quadrant adds an alert condition console with a live-updating display:

```mast
@console/alert_condition !0 ^10 "Alert Condition"
    " Display the alert status of this ship
    gui_section(style="area: 0, 0, 100, 76;")
    ship_id = sbs.get_ship_of_client(client_id)
    image = "RedAlert" if get_data_set_value(ship_id, "red_alert", 0) >= 1 else "AllClear"
    on_screen = gui_image_keep_aspect_ratio_center(get_mission_dir_filename(image))

    on change get_data_set_value(ship_id, "red_alert", 0):
        image = "RedAlert" if get_data_set_value(ship_id, "red_alert", 0) >= 1 else "AllClear"
        on_screen.update(f"image:{get_mission_dir_filename(image)}")
        gui_represent(on_screen)

    await gui()
```

---

## 11. Fleet Wave Variations

WalkTheLine selects enemy races based on difficulty with weighted random choice — harder difficulties introduce tougher but rarer enemy types:

```mast
=== spawn_wave
    enemy_races = ["Kralien", "Torgoth", "Arvonian", "Ximni"]
    weights = (100-6*DIFFICULTY, DIFFICULTY*2, DIFFICULTY*2, DIFFICULTY*2)
    enemy = random.choices(enemy_races, weights=weights)[0]
    fleet_pos = Vec3.rand_in_sphere(10000, 30000, False, True) + next_pos
    prefab_spawn(prefab_fleet_raider, {
        "race": enemy,
        "fleet_difficulty": DIFFICULTY,
        "START_X": fleet_pos.x,
        "START_Y": fleet_pos.y,
        "START_Z": fleet_pos.z
    })
    ->END
```

---

## 12. Mastlib Usage by Mission Type

| Mission Type | Typical Mastlib Set |
|---|---|
| Narrative / story-driven (few enemies) | ai, comms, consoles, damage, docking, prefabs, science_scans |
| Standard multi-console combat | + fleets, upgrades, basic_player_destroy |
| Full sandbox / venue play | + commerce, hangar, internal_comms, gamemaster, gamemaster_comms, operator |
| No LegendaryMissions at all | sbslib only — handle everything manually |

**Observed counts across missions:**
- MiningDays: 0 mastlibs (fully custom cockpit game)
- HereThereBeMonsters: ~11 mastlibs (narrative, no fleets/commerce)
- theta_quadrant: ~16 mastlibs (adds side_missions, skips commerce/internal_comms)
- WalkTheLine, Infinite_Cosmos, Lucky_13: ~18 mastlibs (full sandbox suite)

The `side_missions` mastlib adds structured optional objectives — only theta_quadrant among these uses it.

---

## 13. Developer Tools as Missions (modding_tools)

modding_tools is a fundamentally different kind of mission: a suite of developer tools with no gameplay at all. No sides, no combat, no game end. It demonstrates several advanced GUI patterns.

**Key characteristics:**
- No `@map/` label — entry point is a plain `==` label
- No LegendaryMissions mastlibs — only the sbslib
- `sim_create()` / `sim_resume()` called directly to bring up a minimal simulation
- The "mission" is entirely a multi-pane GUI tool

**Multi-pane GUI architecture:**

```mast
==== editor_start
    gui_row()
    with gui_sub_section("col-width: 4em;"):
        lb_menu = gui_list_box(menu_items, "row-height: 20px;", select=True)
        on change lb_menu.value:
            menu = lb_menu.get_selected_index()
            jump editor_start        # rebuild GUI when menu changes

    content = gui_sub_section()      # named content placeholder

    match menu:
        case 0: jump pane_load
        case 1: jump pane_edit

    jump finish_layout

==== pane_load
    with content:                    # inject content into the placeholder
        gui_row()
        gui_button("Load")
    jump finish_layout

==== finish_layout
    await gui()
```

**New patterns observed:**
- `gui_style_def("style string")` — reusable style variable
- `gui_list_box`, `gui_drop_down`, `gui_checkbox`, `gui_int_slider` — full widget set
- `on gui_click` — click handler (vs `on gui_message` for value changes)
- `data={"key": val}` on widgets — injects local variables into handlers; `__ITEM__` is the triggering widget
- `gui_represent(widget)` — re-render a single widget without rebuilding the whole GUI
- `gui_reroute_server(label)` — rebuild the entire GUI from a new label
- `match/case` — Python 3.10+ syntax works in MAST
- `~~ {dict} ~~` — inline Python for complex dict literals the parser can't handle
- `load_json_data` / `save_json_data` — JSON file I/O from MAST
- `copy_clipboard(text)` — write to system clipboard
- Triple-quoted strings (`"""text"""`) as inline GUI text labels

---

## 14. No-LegendaryMissions: MiningDays

MiningDays uses no mastlibs at all — only the `sbslib`. It handles everything itself:
- `gui_reroute_server("start_server")` and `gui_reroute_clients("launch_to_cockpit")` for console routing
- `player_spawn()` + `sbs.assign_client_to_ship(client_id, ship_id)` for player ship assignment
- A custom cockpit GUI built from scratch with `gui_console("cinematic")`, `gui_layout_widget("3dview")`
- Custom prefab labels in `maps/prefabs.mast` with `metadata:` blocks for data-driven ship configuration

This is the right approach for single-player / non-standard console missions that don't fit the LegendaryMissions framework.

---

## 15. PyMAST — Python Generator Missions (`remote_mission_pick`)

`remote_mission_pick` demonstrates a third kind of mission: no `.mast` file at all. All logic lives in `script.py` using **PyMAST** — Python generator functions decorated with `@label()`.

**Core imports:**

```python
from sbs_utils.mast.label import label
from sbs_utils.mast.maststory import MastStory
from sbs_utils.mast.mast_node import MastDataObject
from sbs_utils.procedural.execution import AWAIT, jump, get_shared_variable, set_shared_variable
from sbs_utils.procedural.timers import timeout
```

**Page class uses `MastStory()` (empty) instead of `story_file`:**

```python
class SimpleAiPage(StoryPage):
    story = MastStory()
    main_server = main_gui
    main_client = main_gui
```

**`yield AWAIT(...)` replaces `await`; `yield jump(fn)` replaces `jump label`:**

```python
@label()
def main_gui():
    yield AWAIT(gui({"start": start}))

@label()
def start():
    yield AWAIT(gui({"back": main_gui}, timeout=timeout(10)))
    yield jump(main_gui)
```

**GUI callbacks** are Python functions registered with `gui_message_callback(widget, fn)` instead of MAST's `on gui_message`.

**`sbs.run_next_mission(name)`** loads and starts a completely different mission folder — the mechanism that makes `remote_mission_pick` useful as a launcher.

**Bug found**: `get_mission_list()` only checked `description.txt` (deprecated). Fixed to check `description.yaml` first using `from sbs_utils import yaml` and `yaml.safe_load(f)`, extracting `Category` and `Description` keys.
