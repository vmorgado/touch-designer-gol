## 1. Project Scaffold

- [x] 1.1 Create six base COMPs at `/project1`: `/audio`, `/gol`, `/render`, `/underlayer`, `/composite`, `/output`

## 2. Audio Analysis (`/audio`)

- [x] 2.1 Create `audio_in` AudioDeviceIn CHOP
- [x] 2.2 Build bass filter chain: `bass_filter (lowpass 180 Hz) → bass_analyze (RMS) → bass_rename`
- [x] 2.3 Build mid filter chain: `mid_hp (highpass 300 Hz) → mid_lp (lowpass 4 kHz) → mid_analyze → mid_rename`
- [x] 2.4 Build treble filter chain: `treble_filter (highpass 3500 Hz) → treble_analyze → treble_rename`
- [x] 2.5 Create `beat_detect` Beat CHOP; enable `beat` and `pulse` output channels
- [x] 2.6 Merge all channels into `out_audio_vals` Null CHOP via `audio_vals` Merge CHOP
- [x] 2.7 Replace per-uniform CHOP expressions with `array0chop` binding on seed injector

## 3. GoL Simulation (`/gol`)

- [x] 3.1 Create `const_init` Constant TOP (black 32×32 RGBA32Float)
- [x] 3.2 Create `feedback` Feedback TOP; set `par.top = 'out_state'`
- [x] 3.3 Create `gol_compute` GLSL TOP (pixel shader, 32×32, RGBA32Float); wire feedback as input 0
- [x] 3.4 Write GoL pixel shader: 8-neighbour count (torus-wrap), Conway rules, seed OR injection via `max(next, seed)`
- [x] 3.5 Create `out_state` Null TOP; complete feedback loop
- [x] 3.6 Connect `const_init` placeholder as input 1 (replaced by seed in Phase 4)

## 4. Audio Seeding (`/gol`)

- [x] 4.1 Create `seed_injector` GLSL TOP (32×32); bind `array0chop` to `/audio/out_audio_vals`
- [x] 4.2 Write seed shader: beat-only 15% density seeding via `floor(uTime * 30.0)` varied pattern
- [x] 4.3 Create `out_seed` Null TOP; rewire as `gol_compute` input 1

## 5. Instanced Grid Rendering (`/render`)

- [x] 5.1 Create `grid_points` gridSOP (32×32), `null_grid` NullSOP, `sopto_grid` SopToCHOP
- [x] 5.2 Create `box1` BoxSOP (0.9×1.0×0.9), `null_box` NullSOP
- [x] 5.3 Create `geo1` Geometry COMP; enable instancing; set `instanceop = null_grid`, `instancesop = /gol/out_state`
- [x] 5.4 Create `cube_mat` GLSL MAT; pass `sGoLState` and `sUnderlayer` as sampler uniforms
- [x] 5.5 Write vertex shader: derive `(col, row)` from `TDInstanceID()`; sample state + underlayer at cell UV; set Y scale via `mix(uDeadHeight, 1.0, alive)` with Y offset correction
- [x] 5.6 Write pixel shader: `alpha = step(0.5, vAlive)`, `color = vUnderlayerColor.rgb`
- [x] 5.7 Create `cam1` Camera COMP (tx=0, ty=40, tz=20, rx=-60), `light1`, `ambient1`
- [x] 5.8 Create `render1` Render TOP; create `out_render` Null TOP

## 6. Underlayer (`/underlayer`)

- [x] 6.1 Create `video_src` Movie File In TOP (file left empty for user to drop clip)
- [x] 6.2 Create `gen_src` GLSL TOP with animated Perlin noise colour field
- [x] 6.3 Create `ctrl_switch` Constant CHOP (channel `index`, value=0)
- [x] 6.4 Create `src_switch` Switch TOP; bind `par.index` expression to `op('ctrl_switch').chan('index')[0]`
- [x] 6.5 Create `out_underlayer` Null TOP

## 7. Compositing (`/composite`)

- [x] 7.1 Create `sel_render` Select TOP referencing `/render/out_render`
- [x] 7.2 Create `out_final` Null TOP (passes through render — underlayer is embedded in cube MAT)

## 8. Output (`/output`)

- [x] 8.1 Create `win1` Window COMP; set `par.top = '/composite/out_final'`; configure resolution

## 9. Smooth State Rendering (`/render`)

- [x] 9.1 Create `raw_state_ref` Select TOP referencing `/gol/out_state`
- [x] 9.2 Create `smooth_feedback` Feedback TOP
- [x] 9.3 Create `smooth_state` GLSL TOP; write lerp shader: `mix(current, target, uLerpSpeed)` with `uLerpSpeed = 0.15`
- [x] 9.4 Create `smooth_out` Null TOP; update `cube_mat` samplers to reference `smooth_out`
