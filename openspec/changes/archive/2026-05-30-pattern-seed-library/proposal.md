## Why

Beat-only random cell seeding (15% density) produces homogeneous, blob-like GoL patterns that lack visual variety. Injecting canonical GoL patterns (gliders, oscillators, methuselahs) produces richer long-lived dynamics — spaceships travel, oscillators anchor activity, methuselahs explode into complex structures.

## What Changes

- Replace `seed_injector` GLSL TOP with a `seed_top` Script TOP that handles seeding entirely in Python
- Add `pattern_library` Table DAT encoding 8 canonical GoL patterns (glider, LWSS, blinker, toad, beacon, pulsar, R-pentomino, acorn)
- Re-enable frequency-band seeding (bass → 20 cells, mid → 12 cells, treble → 8 cells per kick)
- Add kick-triggered category-rotating pattern injection (spaceship → oscillator → methuselah cycle)
- Add snare-triggered oscillator injection (independent of kick)
- Add rythm-triggered quadrant burst seeding (random 16×16 quadrant at ~50% density)
- Add dead-grid recovery: if total audio energy stays below 0.15 for ≥ 2 s, inject one methuselah
- Add `ctrl_seeds` Constant CHOP for runtime-tunable parameters
- Add `noise_trigger` Noise TOP to force `seed_top` to cook every frame

## Capabilities

### New Capabilities

<!-- None — modifies existing audio-gol-seeding capability -->

### Modified Capabilities

- `audio-gol-seeding`: Extends seeding from beat-only random scatter to multi-trigger pattern injection with frequency-band seeding and dead-grid recovery

## Impact

- `/gol/seed_injector` GLSL TOP is removed; replaced by `seed_top` Script TOP + `seed_null` Null TOP
- `gol_compute` input 1 rewired from `seed_injector` to `seed_null`
- New operators: `pattern_library` (Table DAT), `pattern_state` (Table DAT, retained as debug artifact), `ctrl_seeds` (Constant CHOP), `pattern_ctrl` (Script CHOP, retained as debug artifact), `noise_trigger` (Noise TOP), `seed_top`, `seed_null`
- `/audio/out_audio_vals` extended with additional `audioAnalysis` channels: `kick`, `snare`, `rythm`, `smsd`, `fmsd`, `spectralCentroid`
