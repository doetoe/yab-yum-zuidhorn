# Prototype notes

Decision log for Milestone 0. Each entry answers a question from `GDD.md` §13 and records how confident we are, why, and what would make us revisit it.

## Locked for the vertical slice

| ID | Question | Decision | Confidence | Revisit |
|---|---|---|---|---|
| D1 | Is the conga line solid? | Solid to enemies and projectiles; passable by the player. | Medium (~0.7) | After the M1 playtest |
| D2 | Does the line persist between rooms? | No. It is handed over at the exit and starts empty in the next room. | Medium-high (~0.8) | If rooms feel disconnected in M4 |
| D3 | What does a hit cost? | The hit dancer and everyone behind them scatter; no life is lost. Only the timer costs a life. | Medium (~0.65) | After the M1 playtest |
| D4 | When does the exit open? | Immediately when the room objective is met. It is a fixed door the player must reach; there is no second objective or scripted return trip. | High (~0.85) | If rooms end too abruptly |

### D1 — line solidity

**Why:** making the line solid to the player turns every long line into a potential self-trap, which is frustrating in a game built on fast retries. Making it non-solid removes the line's value as a shield and a spatial constraint. Solid-to-enemies only keeps both properties.

**What would change our mind:** playtesters repeatedly body-blocking enemies with the line and ignoring routing, or repeatedly complaining that the line "doesn't do anything". In the first case, make bouncers push through the line after a delay; in the second, give the line a small defensive benefit (for example, absorbing one hit at the cost of the rearmost dancer).

### D2 — line persistence

**Why:** each room has 1–6 dancers and its own objective. Carrying a growing line between rooms would make later rooms exponentially harder, break the "read the room" pillar, and complicate scoring. Resetting keeps rooms self-contained and short.

**What would change our mind:** if the handover feels like losing progress. In that case, award a persistent "party size" bonus at each exit instead of carrying the actual line.

### D3 — cut penalty

**Why:** a cut should hurt positionally, not erase a life. Losing the tail costs time and combo, which is enough to make the player care without making the room restart.

**What would change our mind:** if cuts feel trivial at low line lengths, or devastating at high ones. Difficulty and assist options already plan to vary this (see `GDD.md` §7.9).

### D4 — exit behaviour

**Why:** unlocking immediately keeps the goal legible; walking a fixed door avoids a second, ambiguous objective type. It also gives the level designer a predictable endpoint for par times.

## M0 checklist

- [x] `PLAN.md` and `GDD.md` approved
- [x] Conga rules recorded (D1–D4)
- [ ] ASCII paper prototype of D1–D3 in one room, with a written result
- [ ] `docs/asset-list.md` with logical IDs and frame counts
- [ ] Pack schema v0 with a minimal example pack
- [ ] Level schema v0 with two example rooms
- [ ] One-paragraph theme brief per zone
- [ ] Milestone 1 scope signed off

## Prototype log

Add one entry per playtest or paper run:

```
### YYYY-MM-DD — <prototype name>
- Setup:
- What we tried:
- What happened:
- Decision / change:
- Next question:
```
