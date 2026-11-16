# Yab Yum: After Hours — Game Design Document

**Status:** v1.0 approved
**Approved:** 2026-11-16
**Original draft:** 2026-11-13
**Companion document:** `PLAN.md` (technology, milestones, risks)

---

## 1. High concept

A neon-noir arcade puzzler set in a nightclub that rearranges itself after midnight. You gather partygoers into a conga line, dodge the staff, and escort the whole party to the exit before last call.

One sentence: **Snake meets Pac-Man in a sleazy Amsterdam nightclub.**

---

## 2. Design pillars

1. **The line is the game.** Every dancer you collect makes you stronger and more vulnerable. Progress and risk are the same resource.
2. **Read the room.** Each screen is a legible puzzle box: hazards, keys and exits are visible from the start. Failure should feel like a mistake, not an ambush.
3. **Style over explicitness.** The tone is flirtatious, absurd and cartoonish. The joke is in the situation, never in explicit imagery.
4. **Short rooms, fast retries.** A room lasts 20–90 seconds. Restart is instant and cheap.

---

## 3. Tone, rating and non-goals

- **Tone:** neon-noir comedy. Deadpan, a little seedy, never cruel. Think a 1980s Amsterdam club photographed by someone who loves both the glamour and the grime.
- **Target rating:** roughly PEGI 12 / ESRB T. Suggestive themes, innuendo, cartoon violence against furniture. No nudity, no sexual content, no explicit language beyond mild innuendo.
- **Non-goals:** titillation, shock humour, objectification as a mechanic, realistic depiction of the sex industry. Dancers are characters with agency, not pickups to be "scored".
- **Narrative frame:** light. The club is the antagonist. The player is a guest trying to get everyone out before sunrise.

---

## 4. Setting and story frame

Club Yab Yum, Amsterdam, 02:00 until sunrise. The building is older than it looks and does not want the night to end: corridors loop, rooms rearrange, and the staff are unusually persistent.

The player arrives as a mysterious guest and discovers that the only way out is to leave with the whole party. Five zones, five versions of the same bad idea:

| Zone | Room theme | Time of night | Narrative beat |
|---|---|---|---|
| 1 | Coat Check | 02:00 | Learn the building; the cloakroom attendant has your coat and your exit. |
| 2 | The Bar | 02:45 | Keys, tabs and a rival who wants your crowd. |
| 3 | Main Floor | 03:30 | The DJ controls the lights; the floor is crowded and slippery. |
| 4 | VIP Lounge | 04:15 | One-way doors, private rooms, too many bouncers. |
| 5 | Rooftop | 05:00 | Sunrise, the owner, and one last dance. |

The story is delivered through short room-intro captions (one or two lines), never cutscenes. No dialogue trees, no voice acting.

---

## 5. Characters

- **The Guest (player).** Silent, charismatic, dressed for a night that has gone on too long. The only character the player controls.
- **Dancers (collectibles with personality).** Each has a colour, a silhouette and a mood. They follow in a conga line. Losing them is the main failure state, so they should read as people, not tokens.
- **Bouncers (hazards).** Three behaviours: patrol, chase, guard. Big, slow, readable. They cut the line rather than damage the player directly.
- **The Rival.** A better-dressed guest who targets the tail of your line. Cannot be charmed; must be outmanoeuvred or blocked with crates.
- **The DJ.** Not a physical enemy. Controls the spotlight that sweeps the dance floor and announces bonus rooms.
- **The Owner (boss).** Final dance-off. Three phases, pattern-matching on the dance floor while the line grows.

---

## 6. Core loop

```
Enter room
  → read layout (dancers, hazards, keys, exit)
  → collect dancers (line grows, combo rises)
  → manage hazards (charm, crates, route planning)
  → solve the room gimmick (keys, doors, spills, spotlight)
  → reach the exit with the line intact
  → score screen
  → next room
```

Outer loop: zone intro → four rooms → bonus room → zone clear → next zone. Three lives per zone; game over returns to the start of the zone.

---

## 7. Rules

### 7.1 Movement

- Four directions only. No diagonals, no momentum.
- Grid-locked: the player commits to one tile per step; holding a direction repeats the step.
- Step duration: 100 ms, with input buffering of up to one step.
- Turning is only possible on tile centres; the character sprite faces the last movement direction.
- Collision uses the destination tile at the moment of commitment, not the interpolated position.

### 7.2 The conga line

- Each collected dancer occupies the tile the player occupied **N** steps ago, where N is the dancer's position in the line (1 = immediately behind).
- The line follows the exact path history; it does not pathfind and does not cut corners.
- Path history is capped by the maximum line length (8 dancers).
- **Solidity (prototype decision, see `docs/prototype-notes.md`):** the line is solid to enemies and projectiles, but not to the player. This prevents the player from trapping themselves while preserving the line as a shield and a liability.
- The line can pass through doors and over spills; it cannot pass through walls or crates, because it only ever occupies tiles the player has already traversed.
- The line is handed over at the room exit and starts empty in the next room. Rooms are self-contained; the line is not carried between them.
- If the line is cut, the hit dancer and everyone behind them scatter and respawn at their original positions after 2 seconds, with a visible "returning" state. No life is lost.

### 7.3 Dancers and collection

- Each room contains 1–6 dancers, placed to create route decisions.
- Touching a dancer adds them to the tail instantly and increments the combo.
- Dancers have variants (colour, silhouette, idle animation) chosen by `variant` in the level file.
- A dancer cannot be collected twice in the same room attempt.
- Maximum line length is 8; additional dancers wait at their spawn until there is room.

### 7.4 Hazards

| Hazard | Behaviour | Counter |
|---|---|---|
| Bouncer — patrol | Moves along a fixed path, pauses at endpoints | Route around, block with crates, charm |
| Bouncer — chase | Follows the player at one tile per two steps, gives up after 6 tiles | Break line of sight, use doors |
| Bouncer — guard | Stationary, rotates to face the player, lunges one tile when aligned | Time the crossing, charm |
| Rival | Targets the tail dancer, avoids the player, immune to charm | Block with crates, use the line as bait |
| Spotlight | Sweeps a row or column; standing in it slows the player and alerts bouncers | Move with the sweep, use cover |
| Spill | Slippery tile; the player continues one extra tile in the same direction | Avoid, or use to cross gaps quickly |

Contact with a bouncer or rival cuts the line at the contact point: that dancer and everyone behind them scatter. The player is not damaged directly.

### 7.5 Charm shot

- Fires in the facing direction, range 6 tiles, stops at the first wall or crate.
- Stuns the first bouncer hit for 3 seconds; the rival is immune.
- Costs 1 charisma pip from a maximum of 3; pips refill from pickups and on room clear.
- Cooldown 0.5 seconds; the shot is visible for 0.2 seconds.
- Charming a bouncer in the spotlight stuns it for 5 seconds instead (environmental synergy).

### 7.6 Room objectives

Every room has an explicit objective shown on entry:

- `all_dancers` — collect every dancer, then reach the exit (default).
- `key_and_exit` — collect the key, open the door, reach the exit; dancers optional but scored.
- `escort` — the exit only opens while the line has at least N dancers.
- `survive` — stay alive for the timer duration; dancers are bonus score.
- `bonus` — no hazards; a short timing challenge.

The exit is inert until the objective is met. Its state is telegraphed by colour and animation. Unlocking is immediate; the remaining task is to walk the line to the fixed exit. There is no second objective or scripted escort back to the spawn point.

### 7.7 Timer, lives and failure

- Each room has a time limit (typically 90 seconds) displayed as "time until last call".
- Timer reaching zero costs a life and restarts the room. There is no separate "death" state.
- A line cut does **not** cost a life; it costs position, combo and time. This keeps the failure state proportional.
- Three lives per zone. Losing the last life restarts the zone from its first room with a fresh score for that zone (total score is preserved).
- Time-extension pickups add 10 seconds; they are placed off the critical path.

### 7.8 Scoring

| Event | Points |
|---|---|
| Dancer collected | `100 × combo_multiplier` |
| Combo multiplier | `1 + 0.5 × (pickups_since_last_cut - 1)`, capped at ×4 |
| Room cleared | `time_left × 10` |
| No-hit room | `+500` |
| Exit line multiplier | `1 + 0.1 × line_length`, capped at ×2, applied to room score |
| Zone cleared | `+2000` |
| Bonus room | `+1000` plus a charisma refill or extra life |

High-score table stores the top 10 runs with zone reached, score and date. Scores are compared only within the same game version and difficulty.

### 7.9 Difficulty and assist options

- Three difficulties: **Matinee**, **Prime Time**, **After Hours**.
- Differences are timer length, enemy speed, and how many dancers a cut removes (one vs the whole tail).
- Assist mode (independent toggles): longer timer, slower enemies, reduced screen shake, high-contrast palette, colourblind outlines on dancers and keys.
- Hints appear after three failed attempts at a room: a one-line text hint, never an automatic solution.

---

## 8. Content structure

- 5 zones × 4 standard rooms = 20 rooms.
- 1 bonus room per zone = 5 rooms.
- 1 boss room.
- Total: 26 screens, 50–75 minutes for a first completion.

Room archetypes, used as a grammar for the level designer:

| Archetype | Purpose | Example |
|---|---|---|
| Collect | Teach a mechanic; straightforward route | Zone 1, room 1 |
| Escort | Route planning with a long line | Zone 2, room 3 |
| Key hunt | Exploration and back-tracking | Zone 2, room 2 |
| Crate puzzle | Spatial manipulation, blocking enemies | Zone 3, room 2 |
| Gauntlet | Hazard density and timing | Zone 4, room 3 |
| Survival | Timer pressure, crowd control | Zone 4, room 4 |
| Bonus | Timing mini-game, no fail state | After each zone |

### Boss: "Last Dance"

- Three phases, one per song section.
- The Owner mirrors the player's inputs on a four-tile dance floor, then adds a pattern; the player must repeat it while the line grows.
- Bouncers patrol the edge of the floor; the spotlight marks the next tile in the pattern.
- Mistakes cut the line; completing a phase restores charisma and clears the floor.
- No explicit content: the "dance-off" is literal dancing, with comedic animations.

---

## 9. Level design principles

1. **One new idea per room.** Introduce, combine, then twist.
2. **Everything is visible.** No hidden traps; no off-screen threats in a single-screen game.
3. **The line creates the puzzle.** Good rooms force a choice about *when* to collect the last dancer.
4. **The exit is always readable.** Distinct silhouette and colour, visible from the spawn tile.
5. **Failure is cheap.** Rooms are short; restart is instant; the first 10 seconds are never a cutscene.
6. **No soft-locks.** If the player can reach an unrecoverable state, the room design is wrong. The validator's monkey test is the safety net.
7. **Par times are honest.** `par_time` is the time a competent first-time player needs, not a speedrun time.

---

## 10. Art direction

- **Perspective:** top-down, slightly stylised, 16×16 tiles.
- **Internal resolution:** 320×240, integer scaling, nearest-neighbour.
- **Palette:** limited neon-noir set. Core colours: deep navy `#0B0B1A`, magenta `#FF3FA4`, cyan `#35E1E8`, amber `#FFB03A`, off-white `#F4E9D8`, plus a 32-step ramp for shading.
- **Readability first:** every gameplay-relevant object has a unique silhouette and a 1-pixel outline. Effects never obscure the player or the line.
- **Animation:** 4-direction walk cycles at 8 fps, 2-frame idle, 1-frame telegraphs for lunges and line cuts. Dancers get a subtle idle sway; bouncers get weight and anticipation.
- **Line:** a visible ribbon or hand-holding chain between dancers, colour-coded by line position, so length is readable at a glance.
- **Lighting:** cheap additive glow around neon signs and the spotlight; no per-pixel lighting.
- **UI:** chunky bitmap font, high contrast, minimal chrome. Timer, dancers collected, charisma pips, lives, score, combo.

Sprite budget (default pack): player 24 frames, dancers 6 variants × 10 frames, bouncers 3 × 12 frames, rival 10, props 20, tiles 40, UI 20, FX 16. Roughly 300 frames, comfortably inside one atlas.

---

## 11. Audio direction

- **Style:** synthwave and chiptune hybrid; warm bass, glassy pads, dry drum machine. Period-appropriate without pastiche.
- **Music:** title theme, five zone tracks, boss theme, bonus-room sting. One track per zone, 90–120 seconds, loopable.
- **SFX:** step, pickup, line join, line cut, charm, stun, door, crate push, key, exit open, timer warning, UI move/confirm, combo milestone.
- **Mix:** music and SFX have separate volume sliders; no sound is required to play; visual telegraphs duplicate every audio cue.
- **Licensing:** bespoke or CC0 only. No melodies from the original game.

---

## 12. Accessibility

- Full keyboard and gamepad remapping with conflict detection.
- Remappable action set: move, charm, pause, confirm, cancel.
- Screen-shake toggle, flash reduction, high-contrast palette.
- Colourblind-safe outlines and shapes on dancers, keys and doors.
- Assist toggles listed in section 7.9.
- No audio-only information; no timed text; no rapid mashing.
- Large-window and fullscreen modes with integer scaling.

---

## 13. Prototype questions

The conga rules that block implementation are locked in `docs/prototype-notes.md`. The remaining questions are tuning, not blockers, and are answered through playtests during M1–M4:

1. **Room length:** is 90 seconds the right default, or does 60 feel tighter?
2. **Charm economy:** are three pips and a 0.5 s cooldown enough to feel tactical rather than spammy?
3. **Boss:** dance-off pattern-matching vs pursuit vs multi-room chase.
4. **Line length cap:** is 8 the right maximum, or does 6 read better on screen?
5. **Difficulty split:** which parameters belong to difficulty levels versus assist mode?

Collection order is decided: the exit opens immediately when the objective is met and is a fixed door the player must reach.

Each answer gets a short note with a recommendation, recorded in `docs/prototype-notes.md` or the relevant milestone's playtest log.

---

## 14. Future ideas (not v1)

- Local co-op: second player controls the line's tail.
- Daily challenge rooms with fixed seeds.
- Level sharing via plain JSON files.
- An MSX-style asset pack and a 256×192 render profile.
- An "original mode" that reproduces the 1985 room layouts, if a clean-room reconstruction is possible.
- Speedrun timer and ghost replay.
