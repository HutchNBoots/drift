# Drift

A one-tap lofi rhythm platformer for Amazon Fire Tablet (Android).
Core mechanic: tap flips gravity. The world responds. You drift through it.

## What Claude must do first

Read this file completely before writing any code.
Follow the Superpowers workflow: brainstorm → plan → implement. Never skip ahead.

---

## The 8 design principles

Constraints, not suggestions. Flag any implementation that violates one.

1. **One-tap only** — tap flips gravity. No other input exists.
2. **Music as gameplay** — obstacles spawn and move on the beat. The song defines the level.
3. **Readable visuals** — every element readable at arm's length on a tablet at any flow state.
4. **Short sessions** — one run = one track = 60–90 seconds. Hard cap: 3 minutes.
5. **Mastery loop** — death to retry under 800ms. Coyote time: 8 frames. Input buffer: 10 frames.
6. **Unlockable identity** — cosmetics only. Every unlock earnable without payment.
7. **Community levels** — levels shareable via 6-digit code.
8. **Ergonomic satisfaction** — skilled play feels different from lucky play. See Feel System.

---

## Core mechanic — gravity flip

Tap does not jump. Tap inverts gravity for the player character.
- Player drifts continuously forward (auto-scroll).
- Obstacles extend from floor AND ceiling — no surface is safe.
- Flip timing relative to the beat determines proximity band (see Feel System).
- Off-beat flips are mechanically valid but feel clumsy — natural skill expression.

---

## Feel system (Principle 8)

Gap proximity is measured in px at the moment of passing an obstacle.

| Band | Threshold | Visual | Audio | Haptic |
|---|---|---|---|---|
| Safe | >30px | Short trail | None | None |
| Close | <15px | Amber trail + "Close!" flash | Micro-chord hit | None |
| Pixel-perfect | <4px | "DRIFT!" burst + ribbon trail | Ascending 2-note phrase | Single long buzz |

Flow meter:
- 0–40%: normal visuals, standard audio mix
- 40–75%: longer trail, percussion layer added, obstacles pulse on beat
- 75–100%: world-colour vignette at screen edges, melodic layer added, every gap triggers a chord hit

Death: music cuts, trail vanishes, single soft piano note. Silence is the punishment.
Section clear: world-colour screen flash, ascending 2-note chord, double haptic tap, "Section cleared" fades over 600ms, flow +30%.

---

## The three worlds

Each world is a self-contained visual identity. Same mechanic, different atmosphere.
Progression: World 1 → 2 → 3 maps to increasing BPM and obstacle density.

### World 1 — The Record
**Mood:** Cream and vinyl. Dusty record sleeve. Sunday afternoon. Warm, graphic, unique.
**BPM:** 78
**Palette:** Background #f5efe0 (cream) · Obstacles #993C1D (burnt sienna bars) · Player #D85A30 (coral circle) · Flow accent #F0997B · UI text #2a1f0a
**Obstacles:** Bold vertical bars — like record groove lines. Extend from floor and ceiling.
**Background:** Cream. Faint concentric vinyl record rings. Light and airy — no darkness.
**Player trail:** Warm coral smear. At full flow becomes a ribbon of warm orange.
**Flow vignette:** Coral bleeds into background. Record rings animate.
**Signature detail:** Player circle carries a faint vinyl label motif that rotates on flip.

### World 2 — The Study
**Mood:** Warm amber, desk lamp, hand-drawn ink. Late night, intimate, pencil on paper.
**BPM:** 85
**Palette:** Background #1a1208 (deep warm black) · Obstacles #3a2c14 (ink strokes) · Player #FAC775 (amber circle) · Flow accent #EF9F27 · UI text #c8a882
**Obstacles:** Vertical ink strokes — like pencil marks on graph paper. Organic edges.
**Background:** Deep warm near-black. Distant bookshelf silhouette. Desk lamp cone of light.
**Player trail:** Amber glow. Extends and warms at higher flow.
**Flow vignette:** Background lamp brightens subtly. Ink strokes gain faint halos.
**Signature detail:** Faint graph-paper grid visible in background at low flow, fades at full flow.

### World 3 — The Rooftop
**Mood:** Cool indigo, 3am, rain on concrete. City lights blur in puddles. Cinematic, wide.
**BPM:** 95
**Palette:** Background #0d0f1a (deep indigo) · Obstacles #1D9E75 (teal bars) · Player #5DCAA5 (teal circle) · Flow accent #534AB7 (purple) · UI text #9FE1CB
**Obstacles:** Vertical rain streaks that solidify into barriers. Thin and sharp.
**Background:** Deep indigo. City building silhouettes. Rain lines at 1px opacity. Floor plane shows blurred light reflections.
**Player trail:** Horizontal motion-blur smear. At full flow angles with apparent speed.
**Flow vignette:** Purple blooms at screen edges. Rain streaks angle forward.
**Signature detail:** Faint city window lights flicker in background buildings on the beat.

---

## POC scope — build exactly this, nothing else

- 60fps canvas game loop
- Gravity flip on tap (not jump)
- Player drifts forward automatically
- Obstacles from floor and ceiling, beat-synced from level JSON
- Gap proximity detection: 3 bands as defined in Feel System
- Flow meter with 3 states
- Section markers: start/end gates, payoff feedback on clear
- One complete level using World 1 — The Record visual identity
- Audio: Web Audio API, BPM-locked beat clock at 78 BPM
- Haptics: Web Vibration API

---

## Module map

One concern per module. No cross-module logic.

| File | Owns |
|---|---|
| `main.js` | Game loop only |
| `player.js` | Gravity flip physics, trail, proximity detection |
| `obstacles.js` | Spawning, movement, beat sync |
| `flow.js` | Flow meter state, feedback triggers |
| `level.js` | JSON loader, section marker logic |
| `audio.js` | Web Audio API, beat clock, SFX |
| `renderer.js` | All canvas draw calls |
| `haptics.js` | Vibration patterns |
| `constants.js` | All magic numbers — nowhere else |

---

## Level JSON schema

```json
{
  "world": 1,
  "bpm": 78,
  "track": "assets/audio/world1.mp3",
  "beats": [
    { "beat": 1, "type": "obstacle", "origin": "floor" },
    { "beat": 3, "type": "obstacle", "origin": "ceiling" }
  ],
  "sections": [
    { "startBeat": 8, "endBeat": 16, "label": "gap run" }
  ]
}
```

---

## Code rules

- **TDD always:** write a failing test first, then minimum code to pass it.
- **YAGNI:** if it is not in POC scope, do not build it.
- **No magic numbers:** every constant goes in `constants.js`.
- **Tests:** `tests/` folder, one file per module.

---

## Post-POC — do not build yet

After family playtesting validates the gravity flip feel:
- Port to Godot 4 for native Android APK
- Add Worlds 2 and 3
- Level editor and share codes
- Cosmetics unlock system

Document all feel-critical values in `constants.js` so they survive the port.   
