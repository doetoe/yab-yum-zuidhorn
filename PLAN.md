# Yab Yum: After Hours — Delivery Plan

**Status:** v1.0 approved
**Approved:** 2026-11-16
**Original draft:** 2026-11-13
**Companion document:** `GDD.md` (rules, content, art and audio direction)

This document covers how the game gets built: scope, technology, architecture, data formats, milestones, testing and risks.

---

## 1. Confirmed decisions

| Area | Decision | Consequence |
|---|---|---|
| Direction | Modern reimagining | Original mechanics and levels are inspiration, not a specification. Design freedom, less ROM archaeology. |
| Core mechanic | Conga-line escort | A distinctive hook that must be prototyped early (Milestone 0). |
| Setting | Neon nightclub | Cohesive art and audio direction; keeps the original's spirit. |
| Content | Suggestive, non-explicit | Target rating roughly PEGI 12 / ESRB T. No nudity, no sexual content. |
| Platform | Desktop, `pygame-ce` | Linux, Windows and macOS builds via PyInstaller. Web is a later option. |
| Art | New original pixel art, swappable packs | The asset layer is an explicit interface so a future MSX-style pack (including privately supplied original assets) needs no code changes. |
| Process | Plan documents first | No source code until `PLAN.md` and `GDD.md` are reviewed. |

---

## 2. Product summary

A single-screen, room-based arcade puzzler. The player gathers partygoers into a conga line, dodges bouncers and rivals, solves each room's gimmick, and escorts the line to the exit before "last call".

- **Session length:** 20–90 seconds per room, 8–12 minutes per zone, 50–75 minutes for a first full run.
- **Structure:** 5 zones × 4 rooms, one bonus room per zone, one final boss.
- **Audience:** players who enjoy arcade puzzlers (Eggerland/Lolo, Pac-Man, Snake) and 80s/90s European microcomputer games.
- **Business model:** none. Free, non-commercial fan project.

---

## 3. v1 scope

### In scope

- One complete campaign: 20 standard rooms, 5 bonus rooms, 1 boss.
- Three bouncer behaviours, one rival type, one environmental hazard (spotlight), one floor hazard (spill).
- Keys, doors, pushable crates, charm shot, charisma meter.
- Timer, lives, scoring, combo multiplier, zone checkpoints.
- Title screen, options (volumes, scale, key rebinding), pause, high-score table, save/continue.
- English text, externalised into a string table for future localisation.
- One complete asset pack (`neon`) and one gamepad/keyboard control scheme.
- Linux, Windows and macOS builds.

### Out of scope for v1

Multiplayer, online leaderboards, level sharing, mobile, browser build, Steam integration, achievements, additional languages, 3D, voice acting, level editor for players (an internal editor is in scope).

---

## 4. Technology

| Choice | Version / notes | Rationale |
|---|---|---|
| Python | 3.12, pinned with `uv` | Modern typing, `tomllib` in stdlib, good wheel coverage. `uv` is already installed on the dev machine. |
| `pygame-ce` | ≥ 2.5 | Mature 2D API, fast blitting, gamepad support, simple packaging, community edition is actively maintained. |
| `platformdirs` | latest | Correct save/config locations on each OS. |
| `pytest` + `hypothesis` | latest | Unit and property-based testing. |
| `ruff` | latest | Lint and format. |
| `mypy` | latest, optional strictness | Type checking on `core/`, `world/` and `rules/`. |
| `pyinstaller` | latest | One-folder builds for the three desktop targets. |

Runtime dependencies stay at exactly two (`pygame-ce`, `platformdirs`). Audio synthesis and asset baking are build-time scripts using only the standard library (`wave`, `array`, `math`).

Rejected alternatives:

- **arcade / pyglet** — heavier, less predictable packaging, no benefit at this scale.
- **pygame (upstream)** — slower development, fewer wheels; `pygame-ce` is a drop-in.
- **Kivy / Godot** — wrong tool for a fixed-resolution pixel game; Godot would abandon the Python requirement.

---

## 5. Architecture

```
sander/
  PLAN.md
  GDD.md
  pyproject.toml
  README.md
  src/yabyum/
    __main__.py           # entry point: parse args, build App, run
    app.py                # window, fixed-timestep loop, state stack, shutdown
    settings.py           # defaults, config load/save, keybinds
    core/
      clock.py            # accumulator, frame budget, interpolation alpha
      events.py           # input mapping, buffered actions, gamepad
      rng.py              # seeded RNG service
      tween.py            # easing and tile-to-tile interpolation
      grid.py             # directions, vector helpers, BFS/line-of-sight
      assets.py           # pack discovery, loading, logical-ID lookup
      pack.py             # pack schema validation
      save.py             # save file, high scores, migrations
      logging.py          # structured logging, crash log
    states/
      title.py
      zone_select.py
      play.py
      pause.py
      options.py
      zone_clear.py
      game_over.py
      credits.py
    world/
      level.py            # parse and validate level JSON
      tilemap.py          # collision grid, tile rendering, dirty regions
      entities.py         # dataclasses: player, dancer, bouncer, rival, prop
      conga.py            # path memory and follower placement
      ai.py               # patrol, chase, guard, rival target selection
      rules.py            # objectives, timer, lives, scoring, combo
      collision.py        # grid collision, line contact, projectile hits
    ui/
      hud.py
      menu.py             # reusable menu widgets
      font.py             # bitmap font loader
      transitions.py      # fades, wipes, screen shake
    data/
      packs/neon/         # default asset pack
      levels/*.json
      strings/en.json
    tools/
      validate_levels.py  # schema + reachability checks, CI entry point
      pack_check.py       # asset pack conformance, CI entry point
      level_editor.py     # developer-only F1 editor
      bake_sfx.py         # generate WAV/OGG SFX from Python
  tests/
    unit/
    integration/
    fixtures/
```

### 5.1 Main loop and timing

- Fixed simulation step of 1/60 s with an accumulator; rendering once per display frame.
- Input sampled once per simulation step; actions buffered for one step to avoid dropped presses.
- Rendering uses the interpolation alpha between simulation states for smooth movement.
- A frame-budget guard logs when the simulation exceeds 8 ms or the frame exceeds 16.6 ms for five consecutive frames.

### 5.2 State stack

`App` owns a stack of states. Only the top state receives input and updates; states below may render (for translucent pause menus). Transitions are explicit (`push`, `pop`, `replace`) and tested.

### 5.3 Data flow

```
level JSON ──> Level ──> Tilemap + Entity list ──> systems (input, AI, collision, rules)
                                                    │
                                                    └──> render snapshot ──> renderer
```

Entities are plain dataclasses in a flat list. Systems are stateless functions over the world state. This keeps tests trivial and avoids an ECS that would be overkill at this scale.

### 5.4 Save data

- Location: `platformdirs.user_data_dir("yabyum-after-hours")`.
- Format: JSON, one slot plus a high-score table.
- Contents: schema version, zone/room reached, lives, total score, settings reference, statistics (deaths, best combo), checksum.
- Migration: unknown or older schema versions are upgraded or discarded with a user-visible message; never crash on corrupt saves.

### 5.5 Failure behaviour

- Corrupt asset pack: fail fast in release builds with a readable error; in development, fall back to a `missing` placeholder sprite and log the logical ID.
- Missing level: skip with a logged error and a placeholder room, so a bad data file cannot brick the game.
- Unhandled exception: write a crash log next to the save file, show a minimal error state, attempt a clean shutdown.

---

## 6. Asset pack contract

The renderer never references file paths. It requests **logical IDs**. This is the mechanism that makes a future MSX pack possible without code changes.

### 6.1 Pack layout

```
data/packs/neon/
  pack.toml
  atlas.png
  frames.json
  audio/
    sfx/*.ogg
    music/*.ogg
  font.png
  font.json
  palette.json        # optional
```

### 6.2 `pack.toml`

```toml
[meta]
id = "neon"
name = "Neon (default)"
version = "0.1.0"
license = "CC-BY-4.0"

[render]
internal_width = 320
internal_height = 240
tile_size = 16
scale_mode = "integer"      # integer | fit
aspect = "4:3"
palette = "palette.json"    # optional colour LUT for quantization

[sprites]
atlas = "atlas.png"
frames = "frames.json"

[audio]
manifest = "audio.json"

[font]
image = "font.png"
metrics = "font.json"
```

A future MSX profile would set `internal_width = 256`, `internal_height = 192`, `aspect = "4:3"` and a 16-colour palette. The game logic is resolution-independent because all positions are in tile or sub-tile units.

### 6.3 Logical ID namespaces

| Namespace | Examples | Notes |
|---|---|---|
| `tile.*` | `tile.floor`, `tile.wall`, `tile.door.red` | Tiles are 16×16 in the default pack. |
| `player.*` | `player.idle.s`, `player.walk.e.0` | Four directions; 2-frame idle, 4-frame walk. |
| `dancer.*` | `dancer.idle`, `dancer.walk.s.0` | Variants selected by index, e.g. `dancer.2.idle`. |
| `bouncer.*` | `bouncer.patrol.idle`, `bouncer.stunned` | Behaviour is separate from appearance. |
| `rival.*` | `rival.idle` | |
| `prop.*` | `prop.crate`, `prop.key.red`, `prop.exit` | |
| `fx.*` | `fx.charm`, `fx.cut`, `fx.sparkle` | |
| `ui.*` | `ui.heart`, `ui.clock`, `ui.charisma.full` | |
| `sfx.*` | `sfx.pickup`, `sfx.line_cut`, `sfx.charm` | |
| `music.*` | `music.title`, `music.zone1`, `music.boss` | |

### 6.4 Frame manifest

```json
{
  "player.idle.s": [0, 0, 16, 16],
  "player.walk.s.0": [16, 0, 16, 16],
  "tile.floor": [0, 64, 16, 16]
}
```

Animations are declared separately:

```json
{
  "player.walk.s": {"frames": ["player.walk.s.0", "player.walk.s.1"], "fps": 8, "loop": true}
}
```

### 6.5 Pack selection and precedence

1. `--pack <id>` selects a built-in pack.
2. `--pack-dir <path>` overlays a directory (for development or a privately supplied MSX pack).
3. `YABYUM_PACK` environment variable as a fallback.

Overlays replace individual files, not whole packs. `tools/pack_check.py` verifies every logical ID required by the code and every level entity type is present, and is a required CI step.

Packs are **data only**. Loading never executes code, imports modules, or resolves arbitrary paths outside the pack directory.

### 6.6 Why this matters for the MSX-asset goal

A future pack can:

- use a 256×192 internal resolution with a 16-colour palette;
- supply original-style sprites under the same logical IDs;
- optionally provide a palette LUT and a 4:3 display profile.

No game logic, collision, or level data changes. If an MSX pack cannot supply a given animation frame, the pack can alias IDs (for example, all walk frames to a single sprite), which keeps the contract satisfiable.

---

## 7. Level format

Human-editable JSON with an ASCII tile grid. Designed for clean diffs and a simple in-repo validator.

```json
{
  "schema": 1,
  "id": "z2-03",
  "zone": 2,
  "name": "Closing Time",
  "size": [20, 15],
  "legend": {
    "#": "wall",
    ".": "floor",
    "~": "spill",
    "r": "door.red",
    "c": "crate"
  },
  "tiles": [
    "####################",
    "#........#.........#",
    "#..c.....#....r....#",
    "#........#.........#",
    "####################"
  ],
  "entities": [
    {"type": "player", "at": [2, 2]},
    {"type": "dancer", "at": [4, 2], "variant": 1},
    {"type": "bouncer", "at": [10, 6], "behaviour": "patrol",
     "path": [[10, 6], [10, 10]]},
    {"type": "key", "colour": "red", "at": [16, 2]},
    {"type": "exit", "at": [18, 2], "requires": "all_dancers"}
  ],
  "time_limit": 90,
  "par_time": 45,
  "music": "music.zone2",
  "tags": ["crusha", "keys"]
}
```

### Validator rules (`tools/validate_levels.py`)

- Schema version and required fields.
- Exactly one player and one exit.
- At least one dancer.
- Every entity sits on a walkable tile; no entity shares a tile with a wall.
- Key/door consistency: every door colour has a key, and every key is reachable.
- Base reachability: BFS from the player over walkable tiles, with key/door state as a bitmask, proving dancers and the exit are reachable in some order.
- No dancer or key inside a sealed region.
- `par_time < time_limit`, both positive and sane.
- Spawn safety: no enemy spawn within 3 tiles of the player start.
- Warning (not an error) for rooms with more than one entrance/exit if the schema says single-screen.

Conga-line solvability cannot be exhaustively proven (the line state is exponential). The validator checks base connectivity, and a randomized "monkey" simulation flags rooms that fail to complete in N trials. Those rooms are reviewed manually rather than rejected automatically.

---

## 8. Milestones

Estimates are focused working days for one developer.

| # | Milestone | Deliverable | Est. |
|---|---|---|---|
| 0 | Design lock | Approved docs, conga rules recorded, paper prototype, asset list, pack and level schemas frozen at v0 | 2–3 d |
| 1 | Vertical slice | One room, player + conga, one dancer, one bouncer, exit, timer, win/lose, HUD, 60 FPS scaling | 4–6 d |
| 2 | Core systems | Tilemap/collision, keys/doors, crates, charisma + charm shot, lives, scoring, pause | 4–5 d |
| 3 | AI and hazards | Three bouncer behaviours, rival, spotlight, line cutting, spawn safety | 3–4 d |
| 4 | Content | Level validator + internal editor, 20 rooms + 5 bonus rooms, difficulty curve | 5–7 d |
| 5 | Presentation | Final neon pack, animation, particles, lighting, SFX, zone/menu/boss music | 5–7 d |
| 6 | Meta | Menus, options, key rebinding, saves, high scores, credits | 2–3 d |
| 7 | Polish and ship | Playtests, balance, three-OS builds, README, fan-project disclaimer | 3–4 d |

**Total: 28–39 focused days**, roughly 4–6 weeks part-time. Milestone 1 is the go/no-go checkpoint.

### Acceptance criteria per milestone

**M0 — Design lock**
- `GDD.md` and `PLAN.md` approved (done, v1.0).
- Conga rules (solidity, persistence, cut penalty, exit behaviour) recorded in `docs/prototype-notes.md` (done for the prototype; playtests may revise).
- `docs/asset-list.md` lists every logical asset ID with frame counts.
- Pack schema and level schema have worked examples that the future validator can target.
- Zone themes are summarised in one paragraph each.
- Milestone 1 scope is signed off.

**M1 — Vertical slice**
- One room playable start to finish with keyboard and gamepad.
- 60 FPS at 1920×1080 with integer scaling; no tearing or blurring.
- Input latency at most two frames.
- Win, lose and restart paths work.
- `ruff` and `pytest` green; headless integration test covers the slice.

**M2 — Core systems**
- All room mechanics in the GDD are implemented behind data flags.
- Scoring, combo and time bonus match the GDD formulas.
- Save/load survives a restart and a corrupt file.

**M3 — AI and hazards**
- Each enemy behaviour is unit-tested for its characteristic movement.
- No enemy can spawn on or adjacent to the player.
- Line cutting is deterministic and replayable from a seed.

**M4 — Content**
- All rooms pass `validate_levels.py`.
- Every room has a par time and a completion path.
- Two external playtest rounds completed; difficulty curve adjusted.

**M5 — Presentation**
- Final art pack passes `pack_check.py`.
- Every entity and UI element has animation or a deliberate static treatment.
- Music and SFX have independent volume controls and no clipping.

**M6 — Meta**
- All options persist.
- Key rebinding covers keyboard and gamepad and detects conflicts.
- High-score table persists and sorts correctly.

**M7 — Ship**
- Builds launch on clean Linux, Windows and macOS machines.
- No known crash or soft-lock in a full playthrough.
- README covers controls, installation, asset-pack support and licensing.

---

## 9. Testing and quality

| Level | Tool | Coverage |
|---|---|---|
| Unit | `pytest` | Grid movement, collision, conga path memory, scoring, save/load, level parsing |
| Property | `hypothesis` | Random levels never crash the parser or validator; scoring is monotonic; save round-trips |
| Integration | `pytest` + `SDL_VIDEODRIVER=dummy` | Scripted inputs drive state transitions; full room completion; pause/resume |
| Data | `tools/validate_levels.py`, `tools/pack_check.py` | CI gates for level and asset validity |
| Visual | Optional golden-image snapshots | HUD layout, title screen, palette regressions |
| Manual | Playtest checklist | Feel, difficulty, readability, accessibility |

Determinism: every simulation runs from an explicit seed. A failing run can be replayed from the seed plus the recorded input stream.

Performance budget: simulation ≤ 8 ms per step, full frame ≤ 16.6 ms at 1080p on integrated graphics, memory < 250 MB. Measured by a `--benchmark` mode in CI on Linux.

CI: `ruff check`, `ruff format --check`, `mypy` (core modules), `pytest`, `pack_check`, `validate_levels`, plus a headless smoke run.

---

## 10. Risk register

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Conga mechanic frustrates or feels unfair | High | Medium | Paper prototype in M0; keep line solidity and failure rules configurable; playtest at M1 |
| Art scope balloons | Medium | High | Logical-ID packs, programmer art first, hard sprite budget per entity |
| 25 rooms is too much content for the schedule | Medium | Medium | Room grammar and editor; cut to 15 rooms if needed, keep 5 zones by reusing archetypes |
| Music licensing | Medium | Low | Bespoke or CC0 tracks only; no original melodies |
| Name/trademark conflict | Medium | Low | Non-commercial disclaimer; fallback subtitle/rename option |
| Packaging issues on macOS/Windows | Medium | Medium | Build early in M1, not only at M7 |
| Scope creep (multiplayer, web, editor) | High | High | Explicit out-of-scope list; changes require a written amendment |
| pygame-ce performance on low-end hardware | Low | Low | Fixed internal resolution, dirty-rect rendering if needed |

---

## 11. Legal and licensing

- Fan project, non-commercial. No original ROM, graphics, music or code is distributed.
- The default `neon` pack is original work and will carry an explicit licence.
- Users may point the game at a privately supplied pack (`--pack-dir`). The project does not ship, link to, or download original assets.
- "Yab Yum" is used as a tribute reference. The name is associated with a real venue and trademarks in other sectors, so the title remains provisional; a rename is a one-line change in the string table.
- A disclaimer appears in the README and credits: not affiliated with the original author or the venue.

This section records intent, not legal advice.

---

## 12. Open questions

### Resolved for the prototype

1. Conga solidity — solid to enemies and projectiles, passable by the player.
2. Line persistence — resets at each room exit; the line is not carried between rooms.
3. Cut penalty — contact removes the hit dancer and everyone behind; no life is lost. Only the timer costs a life.
4. Exit behaviour — unlocks immediately when the objective is met; a fixed door the player must reach.

Rationale, confidence and revisit triggers are in `docs/prototype-notes.md`.

### Open

5. Final title: keep "Yab Yum: After Hours" or use a fully original name?
6. Boss design: dance-off, chase, or multi-phase room?
7. Art pipeline: hand-authored, generated with manual cleanup, or commissioned?
8. Minimum hardware target, which sets the final performance budget.
9. Assist mode: disable line cutting entirely, or only slow enemies?

---

## 13. Immediate next actions

1. ~~Review and approve `PLAN.md` and `GDD.md`.~~ Done, v1.0 (2026-11-16).
2. ~~Resolve conga solidity, persistence and failure rules.~~ Done; see `docs/prototype-notes.md`.
3. Build the ASCII paper prototype of the conga rules and record the result in `docs/prototype-notes.md`.
4. Produce `docs/asset-list.md`, the v0 pack example and two example level files.
5. Sign off Milestone 1 scope, then scaffold the repository per §5.
