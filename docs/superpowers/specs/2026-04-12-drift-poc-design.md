# Drift — POC Design Spec
**Date:** 2026-04-12
**Status:** Approved

---

## Overview

Drift is a one-tap lofi rhythm platformer for Amazon Fire Tablet (Android). The POC is a browser-based canvas game targeting desktop Chrome/Firefox. Its purpose is to validate the core gravity flip feel before porting to Godot 4.

**One sentence:** Tap flips gravity. Thread the gap. Feel the groove.

---

## Scope

Build exactly this. Nothing else.

- 60fps canvas game loop
- Gravity flip on tap
- Auto-scrolling player (section-locked speed)
- Beat-synced obstacles pre-computed from level JSON
- Gap proximity detection: 3 bands
- Flow meter with 3 states and miss-based decay
- Section markers: start/end gates with payoff feedback on clear
- World 1 — The Record visual identity
- Synthesized lofi audio via Web Audio API (no external files)
- Haptics via Web Vibration API
- TDD throughout: Vitest, one test file per logic module

---

## Design Principles (from CLAUDE.md)

All 8 principles apply. The ones most load-bearing for the POC:

1. **One-tap only** — tap flips gravity. No other input.
2. **Music as gameplay** — obstacles are beat-synced. The groove defines the level.
5. **Mastery loop** — death to retry ≤800ms. Coyote time: 8 frames. Input buffer: 10 frames.
8. **Ergonomic satisfaction** — the feel system must make skilled play feel physically different.

---

## Architecture

### Pattern: Pure functions + single state object

All game state lives in one plain JS object. Each module exports pure functions that accept the state and return updated state. `main.js` orchestrates the loop — calling each module's update in sequence, then passing state to the renderer.

No event bus. No shared mutable state. No cross-module imports. The renderer is the only module that never returns state — it only reads.

```
main.js
  → audio.updateBeatClock(state)
  → level.resolveSection(state)
  → player.update(state, dt)
  → obstacles.update(state)
  → flow.update(state)
  → renderer.draw(state)
```

### Game State Shape

```js
{
  // Player
  player: {
    x: Number,          // fixed X position (world scrolls, player stays)
    y: Number,          // current Y position in px
    vy: Number,         // vertical velocity in px/frame
    gravityDir: 1 | -1, // 1 = down, -1 = up
    trail: [],          // array of {x, y, age} for trail rendering
    inputBuffer: 0,     // frames remaining in input buffer window
    coyoteFrames: 0,    // frames remaining in coyote time window
  },

  // World
  scrollX: Number,      // how far the world has scrolled in px

  // Obstacles (pre-computed at level load)
  obstacles: [
    {
      x: Number,        // world-space X position
      y: Number,        // world-space Y (0 = top for ceiling, CANVAS_HEIGHT for floor)
      width: Number,    // bar width in px
      height: Number,   // bar height in px (extends inward from edge)
      origin: 'floor' | 'ceiling',
      active: Boolean,  // has entered the viewport
      beaten: Boolean,  // has been passed by player
      proximityBand: 'safe' | 'close' | 'pixel' | null,
    }
  ],

  // Beat clock
  beat: {
    current: Number,    // fractional beat number (1.0 = first downbeat)
    startTime: Number,  // AudioContext.currentTime at level start
    bpm: Number,        // 78 for World 1
    interval: Number,   // seconds per beat (60 / bpm)
  },

  // Flow
  flow: {
    value: Number,      // 0–100
    state: 'low' | 'mid' | 'high',
  },

  // Level
  level: Object,        // parsed level JSON
  currentSection: {
    startBeat: Number,
    endBeat: Number,
    label: String,
    speed: Number,      // pixels per beat
  },

  // Game status
  status: 'playing' | 'dead' | 'section_clear',
  deathTime: Number | null,  // AudioContext.currentTime of death
  inputEnabled: Boolean,
}
```

---

## Module Specifications

### `constants.js`

All magic numbers. No constant may appear in any other file.

Key constants (initial values — tuned during feel testing):

```js
CANVAS_WIDTH = 800
CANVAS_HEIGHT = 450
PLAYER_X = 150             // fixed player X (world scrolls past)
PLAYER_RADIUS = 16

GRAVITY = 0.5              // px/frame²
MAX_FALL_SPEED = 12        // px/frame terminal velocity

COYOTE_FRAMES = 8
INPUT_BUFFER_FRAMES = 10
DEATH_RETRY_MS = 800       // max ms from death to retry available

PROXIMITY_PIXEL = 4        // px threshold for pixel-perfect band
PROXIMITY_CLOSE = 15       // px threshold for close band
PROXIMITY_SAFE = 30        // px threshold for safe band (above = safe miss)

FLOW_GAIN_PIXEL = 15
FLOW_GAIN_CLOSE = 8
FLOW_DRAIN_SAFE = 5        // drained per safe-band obstacle passed
FLOW_GAIN_SECTION_CLEAR = 30
FLOW_STATE_MID = 40
FLOW_STATE_HIGH = 75

TRAIL_MAX_LENGTH = 20      // max trail positions stored
TRAIL_FADE_FRAMES = 10

SECTION_CLEAR_DISPLAY_MS = 600
```

---

### `player.js`

**Owns:** gravity flip physics, trail accumulation, coyote time, input buffer, death detection.

**Exports:**
- `updatePlayer(state, dt)` → state
- `handleTap(state)` → state

**Physics rules:**
1. Each frame: `vy += GRAVITY * gravityDir` (capped at `MAX_FALL_SPEED`)
2. Each frame: `y += vy`
3. On `handleTap`: if input buffer active or coyote time active → execute flip immediately. Otherwise: `vy = 0`, `gravityDir *= -1`
4. Input buffer: tap registered up to `INPUT_BUFFER_FRAMES` early. If a flip cannot execute this frame (e.g. mid-death transition), buffer the input and execute on the next valid frame.
5. Coyote time: grants a flip window for `COYOTE_FRAMES` frames after the player passes through the vertical midpoint of a gap. This forgives taps that arrive fractionally late — the player intended to flip at the gap centre but tapped just after.

**Death detection** (checked each frame in `updatePlayer`):
- `y - PLAYER_RADIUS < 0` → off top edge
- `y + PLAYER_RADIUS > CANVAS_HEIGHT` → off bottom edge
- Overlap with any active obstacle rect → sets `state.status = 'dead'`, `state.deathTime = now`

**Trail:** Each frame append `{x: PLAYER_X, y, age: 0}` to `player.trail`. Increment all ages. Remove entries where `age > TRAIL_MAX_LENGTH`.

---

### `obstacles.js`

**Owns:** pre-computation of obstacle world positions, activation, proximity detection.

**Exports:**
- `precomputeObstacles(level)` → obstacles array (called once at load)
- `updateObstacles(state)` → `{ state, proximityEvents }` where `proximityEvents` is an array of band strings (`'pixel'`, `'close'`, `'safe'`, `'near'`) for obstacles beaten this frame

**Pre-computation:**
- For each obstacle entry in `level.obstacles`, resolve which section it falls in by beat number
- Section pixel offset = `sum over all preceding sections of (section.endBeat - section.startBeat) * section.speed`
- `worldX = sectionOffset + (beat - section.startBeat) * section.speed`
- Obstacle height is authored in level JSON or defaults to `OBSTACLE_HEIGHT_DEFAULT` from `constants.js`
- Floor obstacles: `y = CANVAS_HEIGHT`, height extends upward. Ceiling obstacles: `y = 0`, height extends downward.

**Activation:** Each frame, activate any obstacle whose `worldX - scrollX` has entered the right-edge spawn window.

**Proximity detection:** When an obstacle transitions from `active` to `beaten` (player's X passes obstacle's X), measure the gap between player's Y position and the nearest obstacle edge. Set `proximityBand` and emit the result to `flow.js` via a return value (not an event).

---

### `flow.js`

**Owns:** flow meter value, state derivation, feedback trigger flags.

**Exports:**
- `updateFlow(state, proximityEvents)` → state

**Rules:**
- `proximityEvents` is an array of band strings produced by `obstacles.js` this frame
- For each event: `'pixel'` → `+FLOW_GAIN_PIXEL`, `'close'` → `+FLOW_GAIN_CLOSE`, `'near'` → no change, `'safe'` → `-FLOW_DRAIN_SAFE`
- On `state.status === 'dead'`: `flow.value = 0`
- On `state.status === 'section_clear'`: `flow.value = min(100, flow.value + FLOW_GAIN_SECTION_CLEAR)`
- `flow.state` derived each frame: `< FLOW_STATE_MID` → `'low'`, `< FLOW_STATE_HIGH` → `'mid'`, else `'high'`
- `flow.value` clamped to `[0, 100]`

---

### `level.js`

**Owns:** JSON parsing, section resolution by beat, section transition detection.

**Exports:**
- `loadLevel(json)` → `{ level, obstacles, currentSection, beat }`
- `resolveSection(state)` → state (checks for section transitions each frame)

**Section transition:** When `beat.current >= currentSection.endBeat`, trigger `status = 'section_clear'`, advance to next section, update `currentSection`.

---

### `audio.js`

**Owns:** Web Audio API context, beat clock, synthesized lofi bed, SFX, flow-driven mixing.

**Exports:**
- `initAudio(state)` → `{ audioCtx, beat }`
- `updateBeatClock(state)` → state
- `playSFX(type)` — fire-and-forget: `'flip'`, `'death'`, `'section_clear'`, `'close_pass'`, `'pixel_pass'`

**Beat clock:**
```js
beat.current = (audioCtx.currentTime - beat.startTime) / beat.interval
```

**Synthesized lofi bed — 5 voices at 78 BPM:**

| Voice | Pattern | Synthesis |
|---|---|---|
| Kick | Beats 1 and 3 | Sine oscillator, pitch sweep 160→60Hz, 200ms decay |
| Snare/snap | Beats 2 and 4 | Filtered noise burst, tight 40ms decay, slight pitch |
| Hi-hat | Every half-beat | High-pass noise, 30ms decay |
| Bass note | Every 2 beats | Sawtooth, low-pass at 400Hz, 800ms decay |
| Chord stab | Every 4 beats | 3-oscillator (root + maj3 + 5th), fast attack/decay |

**Flow mixing:**
- `'mid'` state: percussion gain +3dB
- `'high'` state: chord stab fires on every `'close'` or `'pixel'` proximity event

**SFX:**
- `'flip'`: short click/tick (noise burst, 20ms)
- `'death'`: single soft piano note (sine, slow decay, low velocity)
- `'section_clear'`: ascending 2-note chord (two sines, staggered 80ms)
- `'close_pass'`: micro-chord hit
- `'pixel_pass'`: ascending 2-note phrase

---

### `haptics.js`

**Owns:** Web Vibration API patterns.

**Exports:**
- `vibrate(type)` — `'pixel_pass'`: `[0, 0, 200]` (single long buzz), `'section_clear'`: `[80, 60, 80]` (double tap)

No-ops silently if Vibration API unavailable.

---

### `renderer.js`

**Owns:** all canvas draw calls. No logic, no state mutation.

**Exports:**
- `draw(ctx, state)` → void

**Draw order (World 1 — The Record):**

1. **Background:** Fill `#f5efe0`. Draw faint concentric vinyl rings centred on canvas.
2. **Obstacles:** Fill `#993C1D` bars (burnt sienna). No border.
3. **Player trail:** Coral smear `#F0997B`. Opacity fades with age. At `flow.state === 'high'`: warm orange ribbon, wider.
4. **Player circle:** Fill `#D85A30`. Faint vinyl label motif drawn inside, rotates by accumulated `gravityDir` flips.
5. **Flow vignette:** Only at `'mid'`/`'high'`. Radial gradient from transparent to `#F0997B` at screen edges. Opacity scales with `flow.value`.
6. **HUD:** Flow meter bar (top-left). Section label (top-centre, fades after `SECTION_CLEAR_DISPLAY_MS`).
7. **Death overlay:** Fade to white, soft piano note triggered in `audio.js`. "Tap to retry" appears after `DEATH_RETRY_MS`.
8. **Section clear overlay:** World-colour flash, "Section cleared" text fades over `SECTION_CLEAR_DISPLAY_MS`.

---

### `main.js`

**Owns:** game loop only. No game logic.

```js
function gameLoop(timestamp) {
  const dt = timestamp - lastTimestamp;
  lastTimestamp = timestamp;

  state = audio.updateBeatClock(state);
  state = level.resolveSection(state);
  state = player.updatePlayer(state, dt);
  const { state: newState, proximityEvents } = obstacles.updateObstacles(state);
  state = flow.updateFlow(newState, proximityEvents);

  renderer.draw(ctx, state);
  requestAnimationFrame(gameLoop);
}
```

Tap handler calls `player.handleTap(state)` and `audio.playSFX('flip')`.

---

## Level JSON Schema

```json
{
  "world": 1,
  "bpm": 78,
  "sections": [
    {
      "startBeat": 1,
      "endBeat": 16,
      "label": "intro",
      "speed": 3
    },
    {
      "startBeat": 16,
      "endBeat": 32,
      "label": "build",
      "speed": 4
    }
  ],
  "obstacles": [
    { "beat": 2, "origin": "floor", "width": 24, "height": 120 },
    { "beat": 3, "origin": "ceiling", "width": 24, "height": 100 },
    { "beat": 4, "origin": "floor", "width": 24, "height": 140 }
  ]
}
```

`speed` = pixels per beat. `height` = bar length in px extending from the edge inward.

---

## Feel System

| Band | Threshold | Flow delta | Visual | Audio | Haptic |
|---|---|---|---|---|---|
| Pixel-perfect | gap < 4px | +15 | `DRIFT!` burst + ribbon trail | Ascending 2-note phrase | Single long buzz |
| Close | gap 4–14px | +8 | Amber trail + `Close!` flash | Micro-chord hit | None |
| Near | gap 15–30px | 0 | Normal trail | None | None |
| Safe | gap > 30px | -5 | Short trail | None | None |

Gap is measured as the minimum distance between the player circle edge and the nearest obstacle edge at the moment the player's X passes the obstacle's X.

**Flow states:**

| State | Range | Visuals | Audio |
|---|---|---|---|
| Low | 0–39 | Normal | Standard mix |
| Mid | 40–74 | Longer trail, obstacles pulse on beat | Percussion +3dB |
| High | 75–100 | World-colour vignette, ribbon trail | Melodic layer, chord on every close pass |

---

## Death & Retry

- Death condition: player Y exits canvas bounds OR player rect overlaps any obstacle rect
- On death: music cuts, trail vanishes, single soft piano note plays, silence holds
- Retry available: `DEATH_RETRY_MS` (800ms) after death timestamp
- Retry: reset player to `{ x: PLAYER_X, y: CANVAS_HEIGHT / 2, vy: 0, gravityDir: 1 }`, reset `scrollX = 0`, reset `flow.value = 0`, re-init beat clock

---

## Testing Strategy

**Runner:** Vitest (Node, no browser required)

**One test file per logic module:**

| File | Key test cases |
|---|---|
| `tests/player.test.js` | Flip zeroes vy, inverts gravityDir; gravity accumulates correctly; death detected at bounds; coyote time grants flip after leaving edge; input buffer catches early tap |
| `tests/obstacles.test.js` | World X pre-computed correctly from beat + section speed; activation triggers at correct scrollX; proximity band assigned correctly at each threshold |
| `tests/flow.test.js` | Pixel event adds correct delta; safe event drains; death resets to 0; section clear adds 30; state derived correctly at boundary values |
| `tests/level.test.js` | JSON parsed correctly; section resolved by beat; transition detected when beat crosses endBeat |
| `tests/audio.test.js` | Beat number computed correctly from elapsed time and bpm |

`renderer.js` and `haptics.js` have no tests.

---

## File Structure

```
drift/
├── index.html
├── src/
│   ├── main.js
│   ├── player.js
│   ├── obstacles.js
│   ├── flow.js
│   ├── level.js
│   ├── audio.js
│   ├── renderer.js
│   ├── haptics.js
│   └── constants.js
├── levels/
│   └── world1.json
├── tests/
│   ├── player.test.js
│   ├── obstacles.test.js
│   ├── flow.test.js
│   ├── level.test.js
│   └── audio.test.js
├── docs/
│   └── superpowers/specs/
│       └── 2026-04-12-drift-poc-design.md
└── package.json
```

---

## Out of Scope (POC)

Do not build:
- Worlds 2 and 3
- Level editor or share codes
- Cosmetics unlock system
- Godot port
- Real audio track (MP3)
- Mobile touch testing (Fire Tablet)
- Multiplayer or leaderboards
