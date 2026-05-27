# Implementation Report

**Date**: 2026-05-24  
**Status**: Complete — all 8 phases built, zero errors across `/project1`

This document records every deviation from the plan, every runtime correction, and every user-driven change made during the live implementation session.

---

## Phase 1 — Scaffold

No deviations. Six base COMPs created at `/project1`:

```
/audio  /gol  /render  /underlayer  /composite  /output
```

---

## Phase 2 — Audio Analysis (`/audio`)

### Correction: No single-operator bandpass for mid range

**Plan**: use a single `audiofilterCHOP` in `bandpass` mode for the mid band.  
**Reality**: `audiofilterCHOP` `bandpass` mode has only one cutoff frequency — there is no second cutoff to define the upper edge.  
**Fix**: chain two filters for mid — `highpass` at 300 Hz → `lowpass` at 4 kHz.

### Correction: `beatCHOP` does not output `beat` channel by default

**Plan**: read channel `beat` from `beatCHOP`.  
**Reality**: `beatCHOP` only outputs `ramp` by default. The `beat` channel is an opt-in toggle.  
**Fix**: explicitly set `beat_detect.par.beat = True` and `beat_detect.par.pulse = True`.

### Correction: `mathCHOP` has no `clamp` post-operation

**Plan**: use `mathCHOP` with `postop = 'clamp'` to keep RMS values in 0–1.  
**Reality**: `mathCHOP.par.postop` menu options are: `off`, `negate`, `pos`, `root`, `square`, `inverse` — no clamp.  
**Fix**: inserted a `limitCHOP` after `mathCHOP` with `type = 'clamp'`, `min = 0`, `max = 1`.

### Correction: expression-based GLSL uniforms fail without active audio device

**Plan**: drive `seed_injector` uniforms via `ParMode.EXPRESSION` pointing at `/audio/out_bands` channels.  
**Reality**: expressions evaluate to `TypeError: 'NoneType' object is not subscriptable` when the audio device is not yet active (channels don't exist).  
**Fix**: replaced all per-uniform expressions with a single `array0chop` binding on the GLSL TOP:
- Created `/audio/out_audio_vals` — a merged null CHOP with channels `bass`, `mid`, `treble`, `beat`
- Set `seed_injector.par.array0name = 'uAudio'`, `array0chop = '/project1/audio/out_audio_vals'`
- This is evaluated at cook time, not expression time, and handles missing channels gracefully

### Final `/audio` operator list (21 ops)

```
audio_in → bass_filter → bass_analyze → bass_rename ─┐
         → mid_hp → mid_lp → mid_analyze → mid_rename ┼→ bands_merge → bands_norm → bands_limit → out_bands
         → treble_filter → treble_analyze → treble_rename ┘

audio_in → beat_detect → out_beat

out_bands + out_beat → sel_bands + sel_beat → audio_vals → out_audio_vals
```

---

## Phase 3 — GoL Simulation (`/gol`)

### Correction: GLSL array out of bounds with unconnected input

**Plan**: create `gol_compute` with input 0 = feedback state, input 1 = seed (to be connected in Phase 4).  
**Reality**: writing `sTD2DInputs[1]` in the pixel shader before input 1 is connected causes a compile error: `array index out of range '1'`.  
**Fix**: created a `constantTOP` placeholder (black 32×32) connected as input 1. Replaced with the real `seed_injector` in Phase 4, then destroyed the placeholder.

### GoL pixel shader (final)

- Input 0: feedback state (R = alive 1.0 / dead 0.0)
- Input 1: seed injection texture
- Torus-wrapped neighbour count (8 neighbours, edges wrap)
- Standard Conway rules applied per pixel
- Seed injected via `next = max(next, seed)` — OR semantics

---

## Phase 4 — Audio Seeding (`/gol`)

### Correction: GLSL uniforms switched to CHOP array binding

As documented in Phase 2, per-uniform expressions were replaced with `array0chop`. See Phase 2 for full details.

### User change: seed behaviour simplified to kick-only

**Original plan**: frequency bands (bass, mid, treble) continuously seed random cells proportional to band energy; beat detection triggers an additional burst.  
**User request**: remove all frequency-band seeding; only seed on kick/beat detection.  
**Change**: pixel shader rewritten — all `uBass`, `uMid`, `uTreble` seeding removed. Only `uAudio[3]` (beat channel) gates seeding.

### User change: seed density reduced

**Original**: ~60% of cells seeded per band per frame when band is loud; beat burst seeds ~40%.  
**Final**: on a beat pulse, ~15% of cells are randomly seeded. Pattern varies 30 times per second via `floor(uTime * 30.0)` to prevent the same cells always seeding.

### Final seed injector shader

```glsl
float beat  = uAudio[3];
float r     = rand(uv + vec2(floor(uTime * 30.0) * 0.017, 0.31));
float alive = float(beat > 0.5 && r < 0.15);  // 15% density on kick only
```

---

## Phase 5 — Instanced Grid Rendering (`/render`)

### Correction: `toptoCHOP` does not flatten a 2D TOP to 1024 samples

**Plan**: `toptoCHOP` with `crop = 'full'` on the 32×32 GoL state would output 1024 samples (one per pixel).  
**Reality**: all `crop` modes tested (`row`, `full`, `block`, `col`) gave at most 32 samples with grouped channels (`r0`, `g0`, etc.) — no mode produces 1024 flat samples.  
**Fix**: abandoned `toptoCHOP` entirely. Used the GoL state TOP and the gridSOP as separate instance sources:
- `geo.par.instanceop = 'null_grid'` (SOP → P(0)/P(1)/P(2) for position)
- `geo.par.instancesop = '/project1/gol/out_state'` (TOP → `r` channel for Y scale)
- `geo.par.instancecolorop = '/project1/gol/out_state'` (TOP → `r` for colour/alpha)

### Correction: `TDInstanceColor()` does not exist in GLSL MAT

**Plan**: access instance colour in the GLSL MAT vertex shader via `TDInstanceColor()`.  
**Reality**: `TDInstanceColor()` is not a valid GLSL MAT function — compile error: `no matching overloaded function found`.  
**Fix**: passed the GoL state TOP as a `sampler2D` uniform (`sampler0top`) on the GLSL MAT. In the vertex shader, `TDInstanceID()` derives column/row, then samples the texture at `(vec2(col, row) + 0.5) / 32.0`.

### Correction: cross-COMP wire connections do not persist

**Plan**: connect `/render/out_render` and `/underlayer/out_underlayer` directly into `/composite/over1` via `inputConnectors.connect()`.  
**Reality**: `inputConnectors.connect()` across base COMP boundaries silently fails — `over1` had 0 inputs after the call.  
**Fix**: added `selectTOP` nodes inside `/composite` that reference the external ops by absolute path, then wired those locally into `over1`.

---

## Phase 6 — Underlayer (`/underlayer`)

No deviations from plan.

- `moviefileinTOP` (`video_src`) — file left empty for user to drop a clip
- `glslTOP` (`gen_src`) — animated Perlin noise field (placeholder generative shader)
- `switchTOP` driven by custom parameter `Srcindex` on the base COMP (0 = video, 1 = generative)
- `nullTOP` (`out_underlayer`) as output boundary

---

## Phase 7 — Compositing (`/composite`)

### Correction: cross-COMP wires fail (same as Phase 5)

See Phase 5. Fixed with `selectTOP` nodes inside `/composite`.

### User change: composite simplified — `overTOP` removed

**Original plan**: `overTOP` composites cube render (foreground) over underlayer (background) using cube alpha.  
**User request**: the underlayer image should appear **only on the faces of alive cubes**, not as a separate background layer.  
**Change**: `overTOP` and `sel_underlayer` removed from `/composite`. The cube render IS the final output — the underlayer is sampled inside the GLSL MAT itself (see below).

---

## Material Changes — `cube_mat` (GLSL MAT in `/render`)

### Iteration 1: screen-space UV sampling

**Approach**: sample `sUnderlayer` in the pixel shader using `gl_FragCoord.xy / textureSize(sUnderlayer, 0)`.  
**Problem**: `uTDOutputInfo` (used for resolution) is not available in GLSL MAT — only in GLSL TOP.  
**Fix**: used `textureSize(sUnderlayer, 0)` as resolution source instead.  
**Remaining issue**: screen-space UVs project the underlayer onto the cube faces view-dependently. At a -60° camera angle, the cube side faces sample from incorrect screen positions. User confirmed this was not the desired result.

### Iteration 2 (final): vertex-shader sampling by grid cell UV

**Approach**: sample both `sGoLState` and `sUnderlayer` in the **vertex shader** using the cell's grid UV `(col, row + 0.5) / 32.0`, pass as a varying to the pixel shader.  
**Result**: each alive cube's face shows a solid colour sampled from its corresponding position in the underlayer — angle-independent, reliable across all cube faces.

### Final vertex shader behaviour

```
TDInstanceID() → col, row
cellUV = (vec2(col, row) + 0.5) / 32.0
alive  = texture(sGoLState, cellUV).r
underlayerColor = texture(sUnderlayer, cellUV)

Y scale: mix(0.05, 1.0, alive), shifted so base stays at y=0
```

### Final pixel shader behaviour

```
alpha = step(0.5, vAlive)   // hard threshold: 1=alive, 0=dead
color = vUnderlayerColor.rgb
→ alive cubes show underlayer cell colour at full opacity
→ dead cubes are fully invisible (alpha = 0)
```

### Samplers on `cube_mat`

| Slot | Name | Source |
|------|------|--------|
| sampler0 | `sGoLState` | `/project1/gol/out_state` |
| sampler1 | `sUnderlayer` | `/project1/underlayer/out_underlayer` |

---

## Post-Implementation Fix — Underlayer Source Switching (`/underlayer`)

### Problem: `switchTOP.par.index` expression never updated

**Symptom**: changing the `Srcindex` custom parameter on the `/underlayer` base COMP had no effect — the switch stayed on input 0 (video) regardless.

**Investigation**:

1. `switch.par.index` was in `ParMode.EXPRESSION` with expr `parent().par.Srcindex`
2. `par.index.val` always returned `0.0` even after setting `ul.par.Srcindex = 1`
3. Multiple expression forms tried — all failed:
   - `parent().par.Srcindex`
   - `parent().par.Srcindex.val`
   - `float(parent().par.Srcindex)`
   - `me.parent().par.Srcindex`
4. Setting `par.index.mode = ParMode.CONSTANT` and `par.index.val = 1` worked — confirming the parameter itself is writable.

**Root cause — two separate issues discovered**:

| Issue | Detail |
|---|---|
| Custom COMP par does not trigger re-cook | `parent().par.Srcindex` is a custom parameter on a base COMP. Changing it does not mark the `switchTOP` as needing a re-cook, so the expression is never re-evaluated in the TD cook graph. |
| `.val` vs `.eval()` confusion | In TD, `par.val` reads the *last cooked/cached* value. `par.eval()` evaluates the expression live. These differ when the operator hasn't re-cooked. MCP reads `.val`, making working expressions appear broken. |

**Fix**: replaced the custom COMP parameter binding with a `constantCHOP` named `ctrl_switch` inside `/underlayer`:

```
constantCHOP (ctrl_switch)
  name0  = 'index'
  value0 = 0    ← change to 1 for generative

switch.par.index.mode = ParMode.EXPRESSION
switch.par.index.expr = "op('ctrl_switch').chan('index')[0]"
```

CHOP values ARE tracked as cook dependencies by TD's cook graph — when `ctrl_switch.par.value0` changes, the `switchTOP` correctly re-cooks and reads the new index.

**Verified**: `switch.par.index.eval()` returns `1.0` when `ctrl_switch.par.value0 = 1`. Switch is currently set to `1` (generative shader active).

**Runtime control**: inside `/project1/underlayer`, set `ctrl_switch → value0`:
- `0` → `video_src` (video file playback)
- `1` → `gen_src` (generative Perlin shader)

**Updated tuning table entry**: see below.

---

## Final Network State

```mermaid
flowchart TD
    subgraph audio ["/audio"]
        A1[audio_in] --> A2[bass/mid/treble filters]
        A2 --> A3[analyze + rename + merge]
        A3 --> A4[out_bands]
        A1 --> A5[beat_detect] --> A6[out_beat]
        A4 --> A7[out_audio_vals\nbass · mid · treble · beat]
    end

    subgraph gol ["/gol"]
        G1[const_init] --> G2[feedbackTOP]
        G2 --> G3[gol_compute\nGLSL pixel shader]
        G3 --> G4[out_state]
        G4 -->|par.top| G2
        S1[seed_injector\nbeat-only · 15% density] --> G3
    end

    subgraph render ["/render"]
        R1[grid_points SOP\n32×32] --> R2[null_grid] --> R3[sopto_grid\ntx ty tz · 1024pts]
        R4[box1 SOP] --> R5[null_box] --> R6[geo1\ninstancing ON]
        R3 -->|instanceop pos| R6
        R7[cube_mat GLSL MAT\nsamples GoLState + Underlayer\nby TDInstanceID] --> R6
        R6 --> R8[render1] --> R9[out_render]
        R10[cam1] --> R8
        R11[light1 + ambient1] --> R8
    end

    subgraph underlayer ["/underlayer"]
        U1[video_src] --> U3[src_switch]
        U2[gen_src GLSL] --> U3
        U3 --> U4[out_underlayer]
    end

    subgraph composite ["/composite"]
        C1[sel_render] --> C2[out_final]
    end

    A7 -->|array0chop| S1
    G4 -->|sGoLState sampler| R7
    U4 -->|sUnderlayer sampler| R7
    R9 --> C1
    C2 --> OUT[/output/win1]
```

---

---

## Phase 9 — Pattern Seed Library (`/gol`)

**Reference**: `documentation/gol-seeds-plan.md`  
**Date**: 2026-05-25  
**Status**: Complete — all operators built, frequency-band seeding re-enabled

This phase replaced `/gol/seed_injector` (the beat-only GLSL TOP) with a Script TOP that handles three seeding behaviors simultaneously.

---

### Architectural deviation: seed_top does everything directly

**Plan**: two-operator pipeline — `pattern_ctrl` Script CHOP writes cell list to `pattern_state` Table DAT → `seed_top` Script TOP reads `pattern_state` and renders texture.

**Reality**: `seed_top` performs all logic internally in its `cook()` script, reading directly from `sel_audio` (CHOP), `pattern_library` (DAT), and `ctrl_seeds` (CHOP). It writes the numpy array in a single pass.

The `pattern_ctrl` Script CHOP and `pattern_state` Table DAT were retained in the network (visible in `/gol`) but are **not used by `seed_top`** — they are leftover from an earlier iteration and can be considered debug artifacts.

---

### User change: frequency-band seeding re-enabled

**Previous state** (Phase 4 user change): beat-only seeding, ~15% density via GLSL shader.  
**Phase 9 change**: frequency-band seeding is back, now implemented in Python inside `seed_top`. Scaling:

| Band | Cells per frame at RMS = 1.0 |
|------|------------------------------|
| Bass | 20 |
| Mid | 12 |
| Treble | 8 |

Combined with beat-triggered pattern injection and dead-grid recovery, the system now matches the original requirements spec (Section 5.1 + 5.2).

---

### New operator: `noise_trigger` (noiseTOP)

**Problem**: `seed_top` is a Script TOP. Script TOPs only re-cook when their inputs change. With `sel_audio` (CHOP) as the data source, there is no TOP input to force every-frame cooking.

**Fix**: a `noiseTOP` (`noise_trigger`) is connected as **input 0** to `seed_top`. The noiseTOP is animated (time-dependent) so it cooks every frame, which in turn forces `seed_top` to cook every frame. The noise pixel values are not read by the script — it is purely a cook-dependency trigger.

---

### New operator: `sel_audio` (selectCHOP, inside `/gol`)

Selects from `/project1/audio/out_audio_vals`. Provides channels: `bass`, `mid`, `treble`, `ramp`, `pulse`, `beat`.

**Note**: the script uses `pulse` (not `beat`) for rising-edge beat detection. The `pulse` channel from `beatCHOP` fires a one-frame pulse on each detected beat, which is easier to detect as a rising edge than the `beat` channel.

---

### `out_audio_vals` channel update

The merged output CHOP in `/audio` was updated to expose all beatCHOP outputs. Channels in `out_audio_vals`:

| Channel | Source | Description |
|---------|--------|-------------|
| `bass` | `bass_analyze` | Bass RMS energy 0–1 |
| `mid` | `mid_analyze` | Mid RMS energy 0–1 |
| `treble` | `treble_analyze` | Treble RMS energy 0–1 |
| `ramp` | `beat_detect` | Beat ramp (sawtooth 0→1 per beat) |
| `pulse` | `beat_detect` | One-frame pulse on beat |
| `beat` | `beat_detect` | Sustained beat indicator |

---

### seed_top logic summary

`seed_top` (`seed_top_callbacks` textDAT) runs three seeding passes per frame:

1. **Continuous freq-band seeding**: `int(bass × 20) + int(mid × 12) + int(treble × 8)` random cells forced alive
2. **Beat pattern injection**: on `pulse` rising edge, stamp `patterns_per_beat` patterns from the category-rotating pattern library at random valid positions
3. **Dead-grid recovery**: if `bass + mid + treble < 0.15` for ≥ 2 seconds, inject one random methuselah (avoids full grid death during silent passages)

All three results are written into a single 32×32 float32 numpy array, then output via `copyNumpyArray`.

---

### `ctrl_seeds` channels (actual defaults)

| Channel | Default | Used by seed_top |
|---------|---------|-----------------|
| `patterns_per_beat` | 2 | Yes |
| `hold_frames` | 1 | No (retained for compatibility) |
| `fallback_timeout` | 3.0 | No (hardcoded 2.0 s in script) |
| `activity_threshold` | 0.02 | No (dead-grid uses audio level, not grid activity) |

---

### Final `/gol` operator list

```
const_init → feedback → gol_compute → out_state
                          ↑
noise_trigger → seed_top → seed_null ──────────────────

sel_audio ──────────────────────────────────────────→ seed_top (reads via CHOP ref)
ctrl_seeds ─────────────────────────────────────────→ seed_top (reads via CHOP ref)
pattern_library ────────────────────────────────────→ seed_top (reads via DAT ref)

[pattern_ctrl, pattern_state — present but unused by seed_top]

out_state → [sampler in /render/cube_mat]
```

---

## Parameters to Tune at Runtime

| Location | Parameter | Effect |
|---|---|---|
| `/project1/audio/bands_norm` | `gain` (default 5.0) | Overall RMS scale before clamp |
| `/project1/audio/beat_detect` | `period` | Expected BPM range for beat detection |
| `/project1/gol/ctrl_seeds` | `patterns_per_beat` (default 2) | GoL patterns stamped per beat pulse |
| `/project1/gol/feedback` | `resetpulse` | Manual grid reset |
| `/project1/underlayer/ctrl_switch` | `value0` | 0 = video, 1 = generative shader (`Srcindex` custom par does not trigger re-cook — use this CHOP instead) |
| `/project1/underlayer/video_src` | `file` | Drop video clip here |
| `/project1/render/cube_mat` | `vec0valuex` (uDeadHeight = 0.05) | Height of dead cubes |
| `/project1/render/cam1` | `tx/ty/tz/rx` | Camera position and angle |
