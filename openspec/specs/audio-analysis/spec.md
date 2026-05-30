# audio-analysis Specification

## Purpose
Capture live audio from a physical interface, analyse it into per-frame frequency band energy, onset/trigger pulses, and spectral descriptors, and expose everything as a single Null CHOP consumed by the rest of the network.

## Requirements

### Requirement: Audio capture via AudioDeviceIn CHOP
`audio_in` (AudioDeviceIn CHOP) inside `/audio` captures from a microphone or line-in device. No loopback or virtual audio required.

### Requirement: Analysis via audioAnalysis container
The `audioAnalysis` container COMP (Xavier Tremblay / Matthew Ragan / Greg Hermanovic, v6) is wired from `audio_in` and performs all analysis. The hand-built filter/analyze/beat chain is not used. Only three operators exist at the `/audio` level: `audio_in`, `audioAnalysis`, `out_audio_vals`.

### Requirement: Frequency band channels
Three channels measure RMS energy of filtered frequency ranges, normalised 0–1.

| Channel | Range | Filter | What it measures |
|---------|-------|--------|-----------------|
| `low` | 0–180 Hz | Lowpass @ 180 Hz | Sub-bass and bass energy — kick drums, bass guitars, low rumble |
| `mid` | ~180–3500 Hz | Bandpass @ 800 Hz | Midrange energy — vocals, snare body, guitars, keys |
| `high` | 3500 Hz+ | Highpass @ 3500 Hz | Treble energy — hi-hats, cymbals, sibilance, air |

Gain, smoothing, and threshold controls live inside `audioAnalysis` per band.

#### Scenario: Bass-heavy signal
- **WHEN** a loud low-frequency sound is present
- **THEN** `low` approaches 1.0 while `mid` and `high` remain low

### Requirement: Onset / trigger channels
Three channels fire as 0→1 pulses when a percussive event is detected. Thresholds are set inside `audioAnalysis`.

| Channel | What it detects | Signal nature |
|---------|----------------|---------------|
| `kick` | Kick drum — low-frequency transient onset | 0/1 pulse (rising edge = beat) |
| `snare` | Snare hits — mid-frequency transient onset | 0/1 pulse (rising edge = beat) |
| `rythm` | Broader rhythmic events not captured by kick/snare | 0/1 pulse (rising edge = beat) |

#### Scenario: Kick detected
- **WHEN** a kick onset crosses the configured threshold in `audioAnalysis`
- **THEN** `kick` fires a 1.0 value on that frame; `kick_pulse` Logic CHOP in `/gol` converts it to a guaranteed single-frame rising-edge pulse

### Requirement: Spectral descriptor channels
Three channels describe the shape and behaviour of the spectrum over time.

| Channel | Full name | What it measures | Typical range |
|---------|-----------|-----------------|---------------|
| `fmsd` | Fast Moving Standard Deviation | Short-window variance in overall energy — spikes on sudden transients and attacks | ~0 (silence) to ~0.1+ (loud transients) |
| `smsd` | Slow Moving Standard Deviation | Long-window variance in overall energy — reflects broad dynamics and build-ups | Negative to positive (differential-style signal) |
| `spectralCentroid` | Spectral Centroid | "Centre of mass" of the frequency spectrum — low = bass-heavy, high = bright/trebly | 0–1 normalised |

#### Scenario: Sharp transient (loud attack)
- **WHEN** a sudden loud onset occurs
- **THEN** `fmsd` spikes sharply; `smsd` rises more slowly reflecting the broader energy shift

### Requirement: Unified output CHOP
All channels are exported from `/audio` via a single `out_audio_vals` Null CHOP. This is the sole output consumed by `/gol/sel_audio`. All channels are single-sample (one value per frame, not multi-sample histories).

#### Scenario: Downstream read
- **WHEN** `/gol/sel_audio` reads `/audio/out_audio_vals`
- **THEN** all 9 channels (`low`, `mid`, `high`, `kick`, `snare`, `rythm`, `fmsd`, `smsd`, `spectralCentroid`) are available per frame

### Requirement: Threshold and gain controls inside audioAnalysis
Per-band gain, smoothing, and threshold sliders live inside the `audioAnalysis` container. If `kick`, `snare`, or `rythm` fire too often or rarely, adjust the relevant threshold inside the container — not at the `/audio` network level.

### Requirement: fmsd and smsd require abs() before use
Both `fmsd` and `smsd` can be negative (variance-derived signals). Downstream CHOP chains must apply `abs()` before using them as magnitude values. `spectralCentroid` is already 0–1 and needs no abs().

### Requirement: Current project channel mappings

| Channel | Mapped to | Effect |
|---------|-----------|--------|
| `low` | `/gol/seed_top_callbacks` | Seeds up to 20 random alive cells per kick proportional to bass energy |
| `mid` | `/gol/seed_top_callbacks` | Seeds up to 12 random alive cells per kick proportional to mid energy |
| `high` | `/gol/seed_top_callbacks` | Seeds up to 8 random alive cells per kick proportional to treble energy |
| `kick` | `/gol/seed_top_callbacks` + `/gol/gol_compute` | Rising edge: advances one GoL generation + injects category-rotating pattern + band seeding |
| `snare` | `/gol/seed_top_callbacks` | Rising edge: injects one oscillator-category pattern at random position |
| `rythm` | `/gol/seed_top_callbacks` | Rising edge: densely seeds a random 16×16 quadrant (~50% density) |
| `fmsd` | `/render/cam1` tx expression | Camera X-axis shake — abs(fmsd) × 0.2, clamped 0–0.5, × `shake_noise` (20 Hz Noise CHOP) |
| `smsd` | `/underlayer/gen_src` (`uAnimSpeed`) | Underlayer colour rotation speed — abs(smsd) × 0.15, clamped 0.02–0.3 |
| `spectralCentroid` | `/underlayer/gen_src` (`uHueShift`) | Underlayer hue offset — centroid × 0.5 |
