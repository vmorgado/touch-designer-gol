# Audio Parameter Mapping

**Date**: 2026-05-26 (last updated 2026-05-26 — all suggestions implemented)  
**Source**: `/project1/audio/audioAnalysis` → `out_audio_vals` Null CHOP  
**Consumer**: `/project1/gol/sel_audio` → GoL simulation scripts

---

## Channel Reference

### Frequency Bands

These three channels measure the RMS (root mean square) energy of filtered frequency ranges. Values are normalised 0–1 with gain, smoothing, and threshold controls inside `audioAnalysis`.

| Channel | Frequency Range | Filter Type | What It Measures |
|---|---|---|---|
| `low` | 0 – 180 Hz | Lowpass @ 180 Hz | Sub-bass and bass energy — kick drums, bass guitars, low rumble |
| `mid` | ~180 – 3500 Hz | Bandpass @ 800 Hz | Midrange energy — vocals, snare body, guitars, keys |
| `high` | 3500 Hz+ | Highpass @ 3500 Hz | Treble energy — hi-hats, cymbals, sibilance, air |

---

### Onset / Trigger Channels

These channels fire as 0 → 1 pulses when a specific percussive event is detected. Detection is threshold-based; the threshold slider lives inside the corresponding `audioAnalysis` sub-container.

| Channel | What It Detects | Signal Nature |
|---|---|---|
| `kick` | Kick drum hits — low-frequency transient onset | 0/1 pulse (rising edge = beat) |
| `snare` | Snare hits — mid-frequency transient onset | 0/1 pulse (rising edge = beat) |
| `rythm` | Broader rhythmic events not captured by kick/snare — general percussive onset | 0/1 pulse (rising edge = beat) |

---

### Spectral Descriptor Channels

These channels describe the *shape and behaviour* of the spectrum over time rather than energy in a fixed band.

| Channel | Full Name | What It Measures | Typical Range |
|---|---|---|---|
| `fmsd` | Fast Moving Standard Deviation | Short-window variance in overall energy — spikes on sudden transients and attacks | ~0 (silence) to ~0.1+ (loud transients) |
| `smsd` | Slow Moving Standard Deviation | Long-window variance in overall energy — reflects broad dynamics and build-ups over time | Negative to positive (differential-style signal) |
| `spectralCentroid` | Spectral Centroid | "Centre of mass" of the frequency spectrum — low = bass-heavy sound, high = bright/treble-heavy | 0–1 normalised |

---

## Current Project Mappings

| Channel | Mapped To | Effect |
|---|---|---|
| `low` | `/gol/seed_top_callbacks` | Seeds up to **20 random alive cells per kick** proportional to bass energy |
| `mid` | `/gol/seed_top_callbacks` | Seeds up to **12 random alive cells per kick** proportional to mid energy |
| `high` | `/gol/seed_top_callbacks` | Seeds up to **8 random alive cells per kick** proportional to treble energy |
| `kick` | `/gol/seed_top_callbacks` + `/gol/gol_compute` pixel shader | Rising edge: advances one GoL generation + injects category-rotating pattern (spaceship → oscillator → methuselah) + band seeding |
| `snare` | `/gol/seed_top_callbacks` | Rising edge: injects one **oscillator** category pattern at a random grid position — independent of kick |
| `rythm` | `/gol/seed_top_callbacks` | Rising edge: densely seeds a random **16×16 quadrant** (~50% density) — independent of kick |
| `fmsd` | `/render/cam1` tx expression | Drives camera **X-axis shake** — magnitude = abs(fmsd) × 0.2, clamped 0–0.5, multiplied by fast noise for random direction |
| `smsd` | `/underlayer/gen_src` shader (`uAnimSpeed`) | Drives underlayer **colour rotation speed** — abs(smsd) × 0.15, clamped 0.02–0.3 |
| `spectralCentroid` | `/underlayer/gen_src` shader (`uHueShift`) | Drives underlayer **hue offset** — centroid × 0.5, shifts the base hue of the Perlin noise colour field |

---

## Implementation Notes

### GoL seed architecture

`snare` and `rythm` seeds apply **independently of kick** — they inject alive cells directly into the seed texture even when the GoL is frozen. The GoL pixel shader always applies the seed layer (`max(next, seed)`) regardless of the `uAdvance` flag. Seeds from snare/rythm persist in the frozen state and evolve when the next kick fires.

### Underlayer uniforms

`spectralCentroid` and `smsd` are converted to 1×1 textures via `choptoTOP` and wired as inputs 1 and 2 to `gen_src` (GLSL TOP). The shader reads them with `texelFetch(sTD2DInputs[n], ivec2(0,0), 0).r`.

CHOP chain for each:
- `spectralCentroid` → `centroid_sel` → `centroid_math` (× 0.5) → `centroid_top`
- `smsd` → `smsd_sel` → `smsd_math` (abs, × 0.15) → `smsd_limit` (clamp 0.02–0.3) → `smsd_top`

### Camera shake

`fmsd` → `fmsd_sel` → `fmsd_math` (abs, × 0.2) → `fmsd_limit` (clamp 0–0.5) → drives `cam1.par.tx` expression, multiplied by `shake_noise` (Noise CHOP at 20 Hz period) for random direction.

---

## Notes

- All channels are sampled once per frame (time-sliced) — they are single-sample CHOPs, not multi-sample histories.
- `kick`, `snare`, and `rythm` thresholds can be tuned directly inside `audioAnalysis` via the threshold sliders on each sub-container. If triggers fire too often or too rarely, adjust there first.
- `fmsd` and `smsd` values can be negative (they are variance-derived) — both are abs'd before use.
- `spectralCentroid` is already normalised 0–1 and used directly (scaled by 0.5).
