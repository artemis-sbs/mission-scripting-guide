# Artemis Cosmos Mission Scripting Skills

A practical reference for writing missions and addons in MAST — the scripting language used by Artemis Cosmos. Organized by task, not by syntax.

---

## 1. What Is MAST?

MAST is a linear scripting language embedded in Python. Think "choose your own adventure" or BASIC — execution flows forward, jumps to labels, and runs until it ends. It is **not** object-oriented and does not use function calls with return values.

Key mental model:
- Code lives in **labels** (`== label_name ==`)
- Execution flows top-to-bottom and falls through into the next label unless stopped
- Use `jump` or `->END` to control flow
- Background work runs in **tasks** (spawned with `task_schedule`)
- Python is available inline — most expressions and statements work directly

`sim` and `sbs` (the engine API) are always available as globals. No import needed.

---

## 2. Mission File Layout

```
MyMission/
├── script.py           # Engine entry point — boilerplate, rarely changes
├── story.mast          # All mission logic
├── story.json          # Which libraries (.sbslib / .mastlib) to load
├── description.yaml    # Appears in the mission browser
├── settings.yaml       # Default values for difficulty, players, etc.
├── media/              # Images, skyboxes, music, audio files
└── maps/               # Optional — split large missions into subfolders
```

Large missions (Infinite_Cosmos, Lucky_13) keep `story.mast` minimal and put `@map/` labels in a `maps/` subfolder. Python helper modules (`.py` files) in the mission folder are importable directly from MAST.

### script.py (boilerplate)

```python
try:
    import sbslibs
    from sbs_utils.handlerhooks import *
    from sbs_utils.gui import Gui
    from sbs_utils.mast.maststorypage import StoryPage
    from sbs_utils.mast.mast import Mast

    class MyStoryPage(StoryPage):
        story_file = "story.mast"

    Mast.include_code = True   # show MAST source in runtime errors

    Gui.server_start_page_class(MyStoryPage)
    Gui.client_start_page_class(MyStoryPage)
except Exception as e:
    message = e
    def cosmos_event_handler(sim, event):
        import sbs
        sbs.send_gui_clear(event.client_id, "")
        sbs.send_gui_text(event.client_id, "", "text",
                          f"$text:sbs_utils runtime error^{message};", 0, 0, 80, 95)
        sbs.send_gui_complete(event.client_id, "")
```

### story.json

```json
{
    "sbslib": ["artemis-sbs.sbs_utils.v1.3.0.sbslib"],
    "mastlib": [
        "artemis-sbs.LegendaryMissions.consoles.v1.3.0.mastlib",
        "artemis-sbs.LegendaryMissions.damage.v1.3.0.mastlib",
        "artemis-sbs.LegendaryMissions.docking.v1.3.0.mastlib",
        "artemis-sbs.LegendaryMissions.fleets.v1.3.0.mastlib",
        "artemis-sbs.LegendaryMissions.prefabs.v1.3.0.mastlib",
        "artemis-sbs.LegendaryMissions.comms.v1.3.0.mastlib",
        "artemis-sbs.LegendaryMissions.ai.v1.3.0.mastlib"
    ]
}
```

### description.yaml

```yaml
format version: 1
Category: Standard
Category Priority: C
Visible Mission Name: My Mission
Description: Short description shown in the mission browser.
Keywords: combat, asteroids
```

Avoid apostrophes in `Visible Mission Name` — the YAML single-quote escape (`''`) may render literally in the engine.

### settings.yaml

```yaml
AUTO_START: false
DIFFICULTY: 5
PLAYER_COUNT: 1
PLAYER_SHIP_RESPAWN: false
GAME_TIME_LIMIT: 20

DOCKING:
    refuel_amount: 20
    refuel_delay: 2
    shield_delay: 2
    shield_coeff: 2
    torps_delay: 6
    interior_delay: 2
    interior_count: 2

GAMEMASTER:
    enable: true

NEW_DAMCONS: true          # brain-based damage control
CAN_CHANGE_CONSOLE: true   # allow console switching mid-mission

PLAYER_LIST:
    -   name: "Artemis"
        side: "tsn"
        ship: "tsn_light_cruiser"
        face: "terran"
    -   name: "Intrepid"
        side: "tsn"
        ship: "tsn_battle_cruiser"
        face: "terran"
```

---

## 3. MAST Syntax Essentials

### Labels

```mast
== label_name ==       # top-level label — any number of = signs (2+)
=== label_name         # trailing delimiter optional
```

Labels fall through into the next label by default. Use `->END` to stop.

### Inline labels

```mast
== outer ==
    setup_code()
--- loop               # inline label — local to the parent == scope
    await delay_sim(1)
    jump loop if still_running
    ->END
```

### Routes

```mast
//signal/enemy_spotted
    log("Enemy spotted")
    ->END
```

Routes are triggered by engine events, not execution flow. They do **not** fall through.

### Flow control

```mast
->END                           # end this task
->END if obj is None            # conditional end
jump label_name                 # jump to label
jump label_name if not done     # conditional jump
```

### Variables

```mast
x = 42                          # task-local variable
shared station_id = None        # accessible from all tasks
default count = 0               # set only if not already set
default shared DIFFICULTY = 5   # shared + only-if-not-set
```

Top-level `shared` statements run **only once** — converted to no-ops after first execution. The server runs first, so top-level shared assignments are server-initialised.

### await

```mast
await delay_sim(5)                          # wait 5 sim seconds
await delay_sim(minutes=2)                  # wait 2 minutes
await distance_less(id1, id2, 400)          # wait until two objects are close
await distance_point_less(id, Vec3(0,0,0), 400)
await signal_wait("enemy_destroyed")        # wait for a signal
await task_schedule(spawn_players)          # wait for a spawned task to finish
```

### Task scheduling

```mast
task_schedule(spawn_wave)                               # fire and forget
task_schedule(send_message, {"the_message": "Hello"})  # with data dict
await task_schedule(spawn_players)                      # wait for it to finish
gui_sub_task_schedule("watch")                          # sub-task, ends with GUI
```

### yield (prefabs and AI)

```mast
yield result to_id(obj)    # return a value to the caller
yield success              # prefab / brain succeeded
yield fail                 # prefab / brain failed
yield idle                 # brain: still running, check next tick
```

### Common routes

| Route | Triggered when |
|---|---|
| `//spawn` | Object spawned |
| `//comms` | Comms opened |
| `//signal/name` | `signal_emit("name")` called |
| `//shared/signal/name` | Signal, fires for all clients/tasks |
| `//damage/destroy` | Object destroyed |
| `//damage/killed` | Object killed |
| `//collision/passive` | Passive collision |
| `//dock/hangar` | Docking event |

---

## 4. Setting Up the World

### Sides and diplomacy

```mast
# Top-level (before @map) — runs once on the server
tsn = await prefab_spawn(prefab_side_generic, data={"key":"tsn", "name":"TSN", "color":"#07F"})
raider = await prefab_spawn(prefab_side_generic, data={"key":"raider", "name":"Raider", "color":"#F00"})

side_set_relations(tsn, raider, sbs.DIPLOMACY.HOSTILE)
side_set_relations(tsn, neutral_side, sbs.DIPLOMACY.NEUTRAL)

sim.set_diplomacy_color(sbs.DIPLOMACY.HOSTILE, "#F00")
sim.set_diplomacy_color(sbs.DIPLOMACY.NEUTRAL, "#077")
```

`sbs.DIPLOMACY` values: `HOSTILE`, `NEUTRAL`, `ALLIED`.

### Spawning NPCs

```mast
obj = npc_spawn(x, y, z, "Display Name", "role1, role2", "art_id", "behavior")
id  = to_id(obj)
```

The roles argument is a comma-separated list. **The first role is the side.** Additional roles are query tags.

```mast
station = npc_spawn(0, 0, 0, "Outpost Alpha", "tsn, station", "starbase_civil", "behav_station")
ship    = npc_spawn(100, 0, -2000, "Patrol One", "raider", "skaraan_fighter", "behav_npcship")

# Unpack a Vec3 as positional args
npc_spawn(*Vec3(1000, 0, 1000), "DS 1", "tsn, station", "starbase_command", "behav_station")
```

### Terrain — helper functions

```mast
# Asteroid fields
terrain_spawn_asteroid_box(cx, cy, cz, size_x=40000, size_z=15000, density_scale=3.0, density=3, height=2000)
terrain_spawn_asteroid_sphere(cx, cy, cz, radius=7500, density=2, height=2000)
terrain_spawn_asteroid_scatter(scatter_points, height=500)

# Nebula
terrain_spawn_nebula_sphere(cx, cy, cz, radius=12000, cluster_color="red")

# Manual asteroid with scaling
rock = terrain_spawn(x, y, z, None, "#,asteroid", a_type, "behav_asteroid")
rock.engine_object.steer_yaw = random.uniform(0.0001, 0.003)
sx = random.uniform(1.0, 2.0) + 3
rock.data_set.set("local_scale_x_coeff", sx)
rock.data_set.set("local_scale_y_coeff", sx)
rock.data_set.set("local_scale_z_coeff", sx)
rock.engine_object.exclusion_radius *= sx

# Ring scatter (minefields)
points = scatter_ring(width, depth, cx, cy, cz, inner_r, outer_r, start_angle, end_angle)
for v in points:
    mine = terrain_spawn(v.x, v.y, v.z, None, None, "danger_1a", "behav_mine")
    mine.data_set.set("damage_done", 5)
    mine.data_set.set("blast_radius", 1000)
```

### Tile map system

For large structured worlds, define terrain "decks" and fill from an ASCII art string:

```mast
tile_map = maps_tile_map_create(0, 100_000, 1_000, 2_000)  # id, map_size, cell_x, cell_z

nebula_deck = maps_deck_create()
nebula_deck.add_card(prefab_terrain_nebula_sphere, {"density": 0.5, "cluster_color": "red"})

asteroid_deck = maps_deck_create()
asteroid_deck.add_card(prefab_terrain_asteroid_cluster, {})

tile_map.map_deck("n", nebula_deck)
tile_map.map_deck("a", asteroid_deck)

res = await tile_map.fill(map_art, x_count=100)   # map_art is an ASCII string
```

Each character in `map_art` maps to a deck. Spaces and unmapped characters are skipped. Multiple sectors can use different cell sizes.

### Lifeforms (NPC faces / crew)

```mast
shared admiral = lifeform_spawn("Admiral Harkin", "ter #964b00 8 1;ter #fff 3 5;", "admiral")
face = get_face(admiral.id)

# Assign a random face to a station or ship
set_face(station_id, random_terran(civilian=True))

# Host a lifeform aboard a player ship (appears in interior views)
ensign_rachel.host = artemis_id
```

### Science scan links

Link NPCs to a player ship so the science officer can scan them:

```mast
link(artemis_id, "extra_scan_source", whale_watcher_id)
link(artemis_id, "extra_scan_source", friendly_station_id)
```

---

## 5. Player Ships

### With LegendaryMissions (recommended)

Set `PLAYER_CREATE_DEFAULT: true` in `settings.yaml`. The fleets mastlib pre-creates ships from `PLAYER_LIST`.

```mast
await task_schedule(spawn_players)                   # positions ships near a friendly station
await task_schedule(docking_standard_player_station) # wires up docking

player_list = to_object_list(role("__player__"))
player_name = player_list[0].name
```

### Without LegendaryMissions (manual)

```mast
PLAYER_CREATE_DEFAULT = False
PLAYER_COUNT = 1

//shared/signal/create_player_ships
    shared player = to_object(player_spawn(0, 0, 0, "Artemis", "tsn", "tsn_light_cruiser"))
    ->END

@map/my_mission "My Mission"
"  Mission description.
    # player is now available as a shared variable
    docking_set_docking_logic(player.id, role("tsn") & role("station"), docking_dock_with_friendly_station)
```

For a fully custom cockpit (no LegendaryMissions at all):
```mast
gui_reroute_server("start_server")
gui_reroute_clients("launch_to_cockpit")

=== start_server
    sbs.create_new_sim()
    # spawn world...
    sbs.resume_sim()

=== launch_to_cockpit
    sbs.assign_client_to_ship(client_id, ship_id)
    # build GUI...
    await gui()
```

---

## 6. Game Flow and Endings

### Game loop with a timer

```mast
@map/my_mission "My Mission"
"  Description.
    station_id = to_id(npc_spawn(0, 0, 0, "Home Base", "tsn, station", "starbase_civil", "behav_station"))
    await task_schedule(spawn_players)
    await task_schedule(docking_standard_player_station)

    defense_minutes = max(1, int(GAME_TIME_LIMIT))
    set_timer(0, "defense_timer", minutes=defense_minutes)
    END_GAME_DELAY = max(1, defense_minutes // 4)
    seconds_between_waves = END_GAME_DELAY * 60
    milestone = defense_minutes * 60 - seconds_between_waves
    min_left = defense_minutes

    game_end_condition_add(destroyed_all(role("__player__")), "All ships lost.", False)
    game_end_condition_add(destroyed_any(station_id), "Station destroyed.", False)

    task_schedule(spawn_wave)

--- game_loop
    await delay_sim(seconds_between_waves)
    seconds = get_time_remaining(0, "defense_timer")
    if seconds < milestone:
        task_schedule(spawn_wave)
        milestone -= seconds_between_waves
    jump game_loop if not is_timer_finished(0, "defense_timer")

    # Timer expired — mission won
    START_TEXT = "Mission complete! The station has been defended."
    GAME_STARTED = False
    GAME_ENDED = True
    sbs.play_music_file(0, "music/default/victory")
    signal_emit("show_game_results", None)
    ->END
```

### Promise-based end conditions

`game_end_condition_add` requires a **promise** object. These functions return promises:

```mast
game_end_condition_add(destroyed_all(role("__player__")), "All ships lost.", False)
game_end_condition_add(destroyed_any(station_id), "Station destroyed.", False)
game_end_condition_add(destroyed_any(phoenix_id), "Station destroyed.", False)

# Win condition added once the win is reachable
game_end_condition_add(
    distance_point_greater(amb_id, Vec3(0, 0, -5200), 500),
    "Mission complete!",
    True
)
```

**`is_timer_finished` returns a bool — NOT a promise.** Do not pass it to `game_end_condition_add`. Use the polling loop pattern above for timer-based wins.

### Await distance

```mast
await distance_less(id1, id2, 400)
await distance_point_less(id, Vec3(0, 0, -5200), 400)
```

### Timeout with promise_any

Race two promises — whichever finishes first wins:

```mast
choice = gui_button("Confirm")
result = await promise_any(choice, delay_sim(10))
# if player doesn't click within 10 seconds, the timeout resolves
```

---

## 7. Enemy Waves

### Basic wave spawn

```mast
=== spawn_wave
    fleet_diff = int(DIFFICULTY - 1)
    fleet_pos = Vec3.rand_in_sphere(3000, 6000, False, True)
    prefab_spawn("prefab_fleet_raider", {
        "race": "skaraan",
        "fleet_difficulty": fleet_diff,
        "START_X": fleet_pos.x,
        "START_Y": fleet_pos.y,
        "START_Z": fleet_pos.z
    })
    ->END
```

`fleet_difficulty` maps to fleet size (0–9). `Vec3.rand_in_sphere(min_r, max_r, include_y=False, flat=True)` gives a random point on a flat ring around the origin.

### Weighted race selection

Scale enemy variety with difficulty:

```mast
=== spawn_wave
    enemy_races = ["Kralien", "Torgoth", "Arvonian", "Ximni"]
    weights = (100 - 6*DIFFICULTY, DIFFICULTY*2, DIFFICULTY*2, DIFFICULTY*2)
    enemy = random.choices(enemy_races, weights=weights)[0]
    fleet_pos = Vec3.rand_in_sphere(10000, 30000, False, True)
    prefab_spawn(prefab_fleet_raider, {
        "race": enemy,
        "fleet_difficulty": int(DIFFICULTY),
        "START_X": fleet_pos.x,
        "START_Y": fleet_pos.y,
        "START_Z": fleet_pos.z
    })
    ->END
```

### Scatter relative to a position

```mast
next_pos = Vec3(5000, 0, 3000)
cluster_points = scatter_box(60, next_pos.x, next_pos.y, next_pos.z, 10000, 0, 10000, True)
terrain_spawn_asteroid_scatter(cluster_points, height=500)
```

---

## 8. Communication and Story

### NPC message to all screens and comms

```mast
await task_schedule(send_admiral_message, {"the_message": "Enemy incoming!"})

=== send_admiral_message
    default the_message = "You forgot to set the_message"
    face = get_face(admiral.id)
    sbs.send_story_dialog(0, admiral.name, the_message, face, "#444")

    for c in to_object_list(role("mainscreen") & role("console")):
        sbs.send_story_dialog(c.client_id, admiral.name, the_message, face, "#444")

    my_players = to_object_list(role("__player__") & role("tsn"))
    comms_message(the_message, my_players, admiral.id)
    ->END
```

### Comms routes

```mast
//comms if has_roles(COMMS_ORIGIN_ID, "__player__")
    + "Hail":
        << [green] "Hail"
            % Greetings, commander. How can I help?
            % Welcome. What do you need?
    + "Request Docking" //comms/docking
    + "Attack!" if side_are_enemies(COMMS_ORIGIN_ID, COMMS_SELECTED_ID):
        >> "Prepare to be boarded."

//comms/docking
    + "Back" //comms
    + "Request dock":
        comms_navigate("")
```

`<<` = incoming (NPC speaking). `>>` = outgoing (player speaking). `%` lines are chosen at random.

### Console filtering with linked_to

Target messages to only the consoles linked to a specific ship:

```mast
consoles = linked_to(artemis_id, "consoles") & all_roles("console, comms")
for c in to_object_list(consoles):
    sbs.send_story_dialog(c.client_id, name, message, face, "#444")
```

### Audio

```mast
if AUDIO_ENABLED:
    sbs.play_audio_file(0, get_mission_audio_file("audio/SD02C0166"), 1.0, 1.0)
```

`get_mission_audio_file(path)` resolves relative to the mission's `media/` folder. Provide an `AUDIO_ENABLED` shared setting so players can disable it.

---

## 9. Console Panels

### Basic custom console

```mast
@console/my_panel !0 ^10 "My Panel"
    " Short description
    gui_section(style="area: 0, 0, 100, 100;")
    gui_text("text: Hello World; justify: center;")
    await gui()
```

Format: `@console/name !priority ^sort_order "Display Name"`

### Live-updating panel (watch/repaint pattern)

Use `gui_sub_task_schedule` to run a watcher alongside the GUI. When state changes, the watcher calls `gui_task_jump` to force a repaint:

```mast
=== alert_panel
--- repaint
    ship_id = sbs.get_ship_of_client(client_id)
    alert_state  = get_data_set_value(ship_id, "red_alert", 0)
    shield_state = get_data_set_value(ship_id, "shields_raised_flag", 0)
    dock_state   = get_data_set_value(ship_id, "dock_state", 0)

    if alert_state == 1:
        color = "RED"
    elif shield_state == 1:
        color = "YELLOW"
    elif dock_state == "docked":
        color = "BLUE"
    else:
        color = "GREEN"

    prev_alert_state  = alert_state
    prev_shield_state = shield_state
    prev_dock_state   = dock_state

    gui_section(style=f"area: 10, 10, 90, 90; background:{color}")
    gui_text(f"text:CONDITION {color}; justify:center; font:gui-6; color:{color}")

    gui_sub_task_schedule("watch")   # watcher ends automatically when a new GUI is shown
    await gui()

--- watch
    await delay_sim(1)
    ->END if not object_exists(ship_id)
    alert_state  = get_data_set_value(ship_id, "red_alert", 0)
    shield_state = get_data_set_value(ship_id, "shields_raised_flag", 0)
    dock_state   = get_data_set_value(ship_id, "dock_state", 0)
    if alert_state != prev_alert_state or shield_state != prev_shield_state or dock_state != prev_dock_state:
        gui_task_jump("repaint")     # redirect the GUI task to repaint
    jump watch
```

**Key functions:**
- `get_data_set_value(ship_id, "property", default)` — read a live ship data value
- `object_exists(id)` — returns False if the object has been destroyed
- `gui_sub_task_schedule("label")` — sub-task on the GUI task, auto-cancelled on new GUI
- `gui_task_jump("label")` — redirect the console's GUI task (not the watcher itself)
- `get_mission_dir_filename("image.png")` — resolve a path relative to the mission folder

### Reactive GUI with on change

For simpler cases where the expression can be evaluated inline:

```mast
on_screen = gui_image_keep_aspect_ratio_center(get_mission_dir_filename("AllClear.png"))

on change get_data_set_value(ship_id, "red_alert", 0):
    image = "RedAlert" if get_data_set_value(ship_id, "red_alert", 0) >= 1 else "AllClear"
    on_screen.update(f"image:{get_mission_dir_filename(image)}")
    gui_represent(on_screen)

await gui()
```

---

## 10. GUI Patterns

### Tool-as-mission

A mission does not need gameplay. modding_tools is a developer tool — no sides, no players, no game end — just a server-side GUI. The entry point is a plain `==` label, and `sim_create()` / `sim_resume()` are called directly:

```mast
==== server_start
    sbs.suppress_client_connect_dialog(0)
    sbs.transparent_options_button(0, 1)
    sim_create()
    sim_resume()

    gui_section("area: 0, 0, 100, 100;")
    on gui_message(gui_button("Open Editor")):
        jump editor_show
    await gui()
```

No `@map/` label required. No `story.json` mastlibs needed beyond the sbslib.

### Reusable styles

Define a style string once and reference it by variable:

```mast
row_style  = gui_style_def("row-height: 1.5em; padding: 6px, 0, 2px, 6px;")
form_style = gui_style_def("area: 100-20em, 5em, 100, 100-2em; margin: 5px;")

gui_row(row_style)
gui_section(style=form_style)
```

### Multi-pane layout

Define a named content section, then fill it from different pane labels using `with content:`. Use `match/case` to route between panes:

```mast
==== editor_start
    gui_section("area: 0, 0, 100, 100;")
    gui_row("row-height: 2em; background: #1573")
    gui_text("text: MY EDITOR; font: gui-3; justify: center;")

    gui_row()
    with gui_sub_section("col-width: 4em;"):
        lb_menu = gui_list_box(menu_items, "row-height: 20px;", select=True)
        on change lb_menu.value:
            menu = lb_menu.get_selected_index()
            jump editor_start

    content = gui_sub_section()   # named content area

    match menu:
        case 0: jump pane_load
        case 1: jump pane_edit

    jump finish_layout

==== pane_load
    with content:
        gui_row()
        gui_text("text: Load a file;")
        bb = gui_button("Load")
        on gui_message(bb):
            jump do_load
    jump finish_layout

==== finish_layout
    await gui()
```

### Common GUI widgets

```mast
# Dropdown — binds to a variable, renders current value
todo = gui_drop_down("text: {menu}; list: arc, line, box, sphere", var="menu")
on gui_message(todo):
    jump scatter_client_common    # rebuild GUI on selection change

# List box with selection
lb = gui_list_box(items, "row-height: 1em; background: #1572;", select=True)
lb.set_selected_index(0, False)
on change lb.value:
    selected = lb.get_value()     # current selected item

# Multi-select list box
lb = gui_list_box(items, style, multi=True)
sel = lb.get_selected()           # list of selected items

# Checkbox
cb = gui_checkbox("text: {label}; state: {enable}", style)
on gui_message(cb):
    enable = not enable

# Integer slider
sl = gui_int_slider("low: 0; high: 10;", style)
on gui_message(sl):
    value = sl.value

# Icon (clickable)
ib = gui_icon("icon_index: 137; color: white;", style="click_tag: menu; click_background: #6666")
on gui_click(ib):
    jump server_start

# Face / avatar display
the_face = gui_face(face_string)
the_face.value = new_face_string  # update face
gui_represent(the_face)           # re-render it

# Text label (triple-quoted, inline)
"""Count"""
"""{variable} items loaded"""

# Blank spacer
gui_blank()
```

### `on gui_click` vs `on gui_message`

- **`on gui_message(element):`** — fires when the element's **value changes** (sliders, dropdowns, buttons pressed)
- **`on gui_click(element):`** — fires when the element is **clicked** (used for icons and elements with `click_tag`)

### Widget data dict and `__ITEM__`

Pass a `data={}` dict to a widget to inject local variables into its handler. `__ITEM__` refers to the widget that triggered the event:

```mast
for i, widget in enumerate(widgets):
    sl = gui_int_slider("low: 0; high: {widget['max']};", style, data={"windex": i})
    on gui_message(sl):
        avatar_cur[windex] = __ITEM__.value   # windex and __ITEM__ injected by data={}
        gui_represent(the_face)
```

### Updating a single widget

`gui_represent(widget)` re-renders one element without rebuilding the whole GUI:

```mast
name_input.value = new_name
gui_represent(name_input)
```

### Full GUI rebuild

`gui_reroute_server(label)` redirects all clients to a new label, rebuilding the GUI entirely. Use this when the layout itself needs to change:

```mast
on gui_message(delete_btn):
    delete_item(selected_id)
    gui_reroute_server(editor_start)   # rebuild from scratch
```

`gui_refresh("label_name")` does the same via a string label name.

### Inline buttons inside `await gui()`

One-shot buttons can be placed directly inside the `await gui()` call:

```mast
await gui():
    * "Apply":
        x = int(s_x)
        task_schedule(apply_scatter)

jump scatter_client_common   # runs after button is pressed
```

`*` = consumed after click. `+` = stays visible.

### JSON file I/O

```mast
data = load_json_data(get_mission_dir_filename("grid_data.json"))
save_json_data(get_mission_dir_filename("grid_data.json"), data)
save_json_data(filename + ".bak", data)    # backup before overwrite
```

### Clipboard

```mast
copy_clipboard(char_editor_build_face(race, avatar_enable, avatar_cur))
```

### `match / case`

Python 3.10+ match syntax works in MAST:

```mast
match menu:
    case "arc":
        jump edit_scatter_arc
    case "box":
        jump edit_scatter_box
    case "sphere":
        jump edit_scatter_sphere
```

### When to use `~~ ~~`

Use inline Python (`~~ expr ~~`) only for syntax the MAST parser cannot handle — complex dict/set literals are the main case:

```mast
g = ~~ {"x": pos_x, "y": pos_y, "name": item_name} ~~
# Do NOT use ~~ for regular function calls or assignments
```

---

## 11. Creating Addons

An addon is a subfolder with `__init__.mast`. It provides labels that become globally available to any mission that loads it.

### Folder structure

```
MyAddon/                        # the mission (dev harness + source)
├── script.py                   # standard boilerplate
├── story.mast                  # minimal test harness
├── story.json                  # sbslib + dependencies
├── __lib__.json                # declares the addon for the packager
└── my_addon/                   # THE ADDON ITSELF
    ├── __init__.mast           # imports content files
    ├── panels.mast
    └── helpers.mast
```

### `__lib__.json`

```json
{
    "version": "v1.3.0",
    "mastlib": ["my_addon"]
}
```

### `my_addon/__init__.mast`

```mast
import panels.mast
import helpers.mast
```

### Minimal test harness (`story.mast`)

```mast
PLAYER_CREATE_DEFAULT = False
PLAYER_COUNT = 1

//shared/signal/create_player_ships
    shared player = to_object(player_spawn(0, 0, 0, "Artemis", "tsn", "tsn_light_cruiser"))
    ->END

@map/test "Test Map"
"  A map to test the addon.
    npc_spawn(*Vec3(1000, 0, 1000), "DS 1", "tsn, station", "starbase_command", "behav_station")
    docking_set_docking_logic(player.id, role(player.side) & role("station"), docking_dock_with_friendly_station)
```

### Distribution

During development the addon folder lives in the mission directory and is found automatically. To share with other missions:
1. Package with `sbs.pyz` → produces `my_addon.mastlib`
2. Place in the shared `__lib__/` folder
3. Add `"artemis-sbs.my_addon.v1.0.0.mastlib"` to the target mission's `story.json`

---

## 12. Debug Techniques

### Chapter-jump menu

Store the main story task in a shared variable, then use comms to jump to any narrative chapter without replaying the whole mission:

```mast
# In your @map/ body:
shared main_story_task = mast_task

# In debug.mast:
//comms/chapters if main_story_task
    + "scene one":
        main_story_task.jump("scene_one")
    + "final battle":
        main_story_task.jump("final_battle")
    + "end":
        main_story_task.jump("game_end")

//enable/comms if is_dev_build()
```

### debug.mast pattern

Keep all dev-only code in a separate file so it is easy to exclude from a release build:

```mast
# __init__.mast
import panels.mast
import helpers.mast
# import debug.mast   # uncomment for dev builds
```

`is_dev_build()` returns True in development environments — use it to gate debug routes and tools.

---

## 13. LegendaryMissions Addon Reference

### Which mastlib provides what

| Mastlib | Key labels / features |
|---|---|
| `ai` | Brain behavior trees for NPC fleets |
| `comms` | Enemy taunt/surrender, player comms menus |
| `consoles` | Standard helm, weapons, science, engineering, comms, main screen |
| `damage` | `//damage/destroy` and `//damage/killed` handlers, wreck spawning |
| `docking` | `docking_standard_player_station`, `docking_dock_with_friendly_station` |
| `fleets` | `spawn_players`, `prefab_fleet_raider`, `prefab_fleet_empty` |
| `prefabs` | `prefab_side_generic`, station/terrain prefabs |
| `science_scans` | Science scan content for enemy ships |
| `upgrades` | Pickup collection handlers |
| `hangar` | Landing bay, bar, hangar comms |
| `internal_comms` | Crew department comms (sickbay, security, etc.) |
| `commerce` | Trading / economy system |
| `gamemaster` | GM console and spawn tools |
| `gamemaster_comms` | GM comms menus (messages, spawn, map) |
| `operator` | Operator/admin console for venue use |
| `side_missions` | Structured optional objectives |
| `basic_player_destroy` | Player ship destruction handling |

### Choose your mastlib set

| Mission type | Typical set |
|---|---|
| Narrative / story-driven | ai, comms, consoles, damage, docking, prefabs, science_scans |
| Standard combat | + fleets, upgrades, basic_player_destroy |
| Full sandbox / venue | + commerce, hangar, internal_comms, gamemaster, gamemaster_comms, operator |
| No LegendaryMissions | sbslib only — handle everything manually (see MiningDays) |

Minimum for a standard multi-console Artemis mission: **fleets, docking, prefabs, comms, consoles, damage**.

Truly optional (include only if you use the feature): autoplay, commerce, operator, gamemaster, gamemaster_comms, hangar, internal_comms, side_missions.

---

## 14. PyMAST — Python as the Mission Script

Some missions skip `.mast` files entirely and write mission logic in pure Python using **PyMAST** — a thin layer that turns Python generator functions into MAST labels. `remote_mission_pick` is the clearest example.

### When to use PyMAST

- Tool-style missions (mission pickers, admin consoles, debug launchers)
- Missions that need complex Python logic that would be awkward in MAST
- Rapid prototyping without the `.mast` parser

### Core pattern

```python
from sbs_utils.mast.label import label
from sbs_utils.mast.maststory import MastStory
from sbs_utils.mast.mast_node import MastDataObject
from sbs_utils.procedural.execution import AWAIT, jump, get_shared_variable, set_shared_variable
from sbs_utils.procedural.timers import timeout
from sbs_utils.procedural.cosmos import sim_create, sim_resume

@label()
def main_gui():
    sim_create()
    sim_resume()
    # Build GUI here
    yield AWAIT(gui({"start": start}))

@label()
def start():
    mission = get_shared_variable("mission")
    if mission is not None:
        sbs.run_next_mission(mission)
    yield AWAIT(gui({"back": main_gui}, timeout=timeout(10)))
    yield jump(main_gui)

class SimpleAiPage(StoryPage):
    story = MastStory()       # empty story — no .mast file
    main_server = main_gui    # server entry point
    main_client = main_gui    # client entry point

Gui.server_start_page_class(SimpleAiPage)
Gui.client_start_page_class(SimpleAiPage)
```

### MAST → PyMAST translation

| MAST | PyMAST |
|---|---|
| `await gui(...)` | `yield AWAIT(gui(...))` |
| `jump label_name` | `yield jump(label_fn)` |
| `await delay_sim(5)` | `yield AWAIT(delay_sim(5))` |
| `shared x = val` | `set_shared_variable("x", val)` |
| Read shared var | `get_shared_variable("x")` |

### GUI callbacks

Unlike MAST's `on gui_message`, PyMAST uses Python callbacks registered with `gui_message_callback`:

```python
lb = gui_list_box(items, "", item_template=render_item, select=True)

def on_select(event, sender):
    idx = lb.get_selected_index()
    if idx is not None:
        set_shared_variable("selected", items[idx])

gui_message_callback(lb, on_select)
yield AWAIT(gui({"confirm": confirm_fn}))
```

### Launching a different mission

```python
sbs.run_next_mission(mission_folder_name)
```

### `MastDataObject` — dict-to-attribute adapter

`MastDataObject(dict)` wraps a plain dict so its keys are accessible as attributes (`.name`, `.category`). Use `obj.get("key", default)` to read safely:

```python
from sbs_utils.mast.mast_node import MastDataObject

item = MastDataObject({"name": "Scout", "hp": 100})
item.name   # "Scout"
item.get("hp", 50)   # 100
```

### `story.json` for PyMAST missions

PyMAST missions typically only need the sbslib — no mastlibs required since there are no `.mast` labels to wire up:

```json
{
    "sbslib": ["artemis-sbs.sbs_utils.v1.3.0.sbslib"]
}
```
