# Yab Yum: After Hours

A neon-noir arcade puzzler inspired by the 1985 MSX game *Yab Yum*. Gather partygoers into a conga line, dodge the staff, and escort the party to the exit before last call.

**Status:** design approved · next milestone: **M0 — prototype the conga mechanic**

## Start here

| Document | What it covers |
|---|---|
| [`PLAN.md`](PLAN.md) | scope, technology, architecture, asset-pack contract, level format, milestones, risks, licensing |
| [`GDD.md`](GDD.md) | rules, content structure, art and audio direction, accessibility |
| [`docs/prototype-notes.md`](docs/prototype-notes.md) | locked prototype decisions and the M0 checklist |

New to the project? Read `PLAN.md` §1–3, then `GDD.md` §6–7.

## At a glance

- **Genre:** single-screen arcade puzzler
- **Core hook:** the conga line — every dancer makes you stronger and more vulnerable
- **Platform:** desktop (Linux, Windows, macOS)
- **Stack:** Python 3.12, `pygame-ce`
- **Resolution:** 320×240 internal, integer-scaled
- **Rating target:** suggestive, non-explicit (roughly PEGI 12)
- **Campaign:** 5 zones, 26 screens, 50–75 minutes

## Project layout

The repository currently contains design documents only. The source layout is specified in `PLAN.md` §5 and will be created during Milestones 0–1.

## Licensing

- **Code and documentation:** MIT, see [`LICENSE`](LICENSE).
- **Art and audio packs:** CC BY 4.0, as declared in each pack's `pack.toml`. No pack artwork exists yet.
- **Original MSX assets** are not distributed with this project. The asset-pack interface supports privately supplied packs; see `PLAN.md` §6.

## Disclaimer

This is a non-commercial fan project. It is not affiliated with Tom Gerritsen, the original *Yab Yum*, or the Amsterdam venue of the same name. "Yab Yum" is used as a tribute reference; the title is provisional.
