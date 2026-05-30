## 1. Pattern Library DAT

- [x] 1.1 Create `pattern_library` Table DAT in `/gol` with columns: `name`, `category`, `cells`
- [x] 1.2 Populate 8 patterns: glider, LWSS (spaceship); blinker, toad, beacon, pulsar (oscillator); R-pentomino, acorn (methuselah)
- [x] 1.3 Encode `cells` column as JSON arrays of `[x, y]` offsets

## 2. Control Parameters

- [x] 2.1 Create `ctrl_seeds` Constant CHOP with channels: `patterns_per_beat=2`, `hold_frames=1`, `fallback_timeout=3.0`, `activity_threshold=0.02`

## 3. Cook-Dependency Trigger

- [x] 3.1 Create `noise_trigger` Noise TOP (time-dependent, 32×32); wire as input 0 to `seed_top`

## 4. Seed TOP Script

- [x] 4.1 Create `seed_top` Script TOP (32×32, r32f format)
- [x] 4.2 Write `seed_top_callbacks` cook() — three seeding passes:
  - Freq-band seeding gated on kick rising edge
  - Category-rotating pattern injection on kick rising edge
  - Dead-grid recovery (methuselah) after 2 s silence
- [x] 4.3 Verify pattern cell coordinates map to `arr[y, x]` without Y-flip

## 5. Rewire and Cleanup

- [x] 5.1 Create `seed_null` Null TOP; wire from `seed_top`
- [x] 5.2 Rewire `gol_compute` input 1 from `seed_injector` to `seed_null`
- [x] 5.3 Destroy `seed_injector` GLSL TOP
- [x] 5.4 Extend `/audio/out_audio_vals` with `snare`, `rythm`, `smsd`, `fmsd`, `spectralCentroid` channels from `audioAnalysis`

## 6. Underlayer Audio Modulation

- [x] 6.1 Create `centroid_sel` → `centroid_math` (× 0.5) → `centroid_top` choptoTOP chain; wire as input 1 to `gen_src`
- [x] 6.2 Create `smsd_sel` → `smsd_math` (abs, × 0.15) → `smsd_limit` (clamp 0.02–0.3) → `smsd_top` chain; wire as input 2 to `gen_src`
- [x] 6.3 Update `gen_src` shader to read `uHueShift` from input 1 and `uAnimSpeed` from input 2
