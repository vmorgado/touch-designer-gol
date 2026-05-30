## Context

TouchDesigner requires GPU-friendly, operator-based data flow. Direct CPU scripting is viable for logic (Script CHOPs/TOPs) but heavy compute should stay on GPU via GLSL. The system must sustain 60 FPS while audio analysis, GoL compute, and instanced rendering all run every frame.

## Goals / Non-Goals

**Goals:**
- All GoL computation on GPU (GLSL pixel shader + feedback TOP)
- All geometry via instancing (one Geometry COMP, not 1024)
- Audio analysis and GoL logic cleanly separated into dedicated base COMPs
- Underlayer colour embedded in the cube GLSL MAT (no separate Over TOP composite)

**Non-Goals:**
- Multi-display output
- Audio loopback or virtual devices
- Per-alive-cube height modulation from audio (all alive cubes are uniform height 1.0)
- PBR lighting (effectively unlit GLSL MAT)

## Decisions

### GoL Compute: pixel shader on GLSL TOP (not compute shader)
Using a pixel shader (`glslTOP` in pixel-shader mode) with `texelFetch` for neighbour sampling. Simpler neighbour access than a compute shader and sufficient for 32×32. Each pixel = one cell; feedback loop: `const_init → feedbackTOP → gol_compute → out_state → feedback.par.top`.

### Grid positions: gridSOP not soptoCHOP for instances
`toptoCHOP` does not produce 1024 flat samples from a 32×32 TOP in any crop mode — it groups channels, not pixels. The geometry COMP uses `instanceop` (SOP → position) and `instancesop` (TOP → R channel for Y scale), keeping position data in SOP and state data in TOP.

### GoL state → cube height: sampled in vertex shader by TDInstanceID
`TDInstanceColor()` does not exist in GLSL MAT. Instead, the GoL state TOP (`sGoLState`) and underlayer TOP (`sUnderlayer`) are passed as `sampler2D` uniforms. The vertex shader derives `(col, row)` from `TDInstanceID()` and samples both textures at `(col + 0.5, row + 0.5) / 32.0`.

### Underlayer: embedded in cube MAT (not Over TOP)
The underlayer appears only on alive cube faces — the grid background is black. Dead cubes have alpha = 0. No `overTOP` or `Composite TOP` is needed; `/composite` passes through the render output.

### Underlayer source switching: constantCHOP (not custom COMP parameter)
Custom COMP parameters do not trigger downstream re-cooks. `switchTOP.par.index` must read from a CHOP (`ctrl_switch` Constant CHOP) whose value IS a cook dependency. Setting `ctrl_switch.par.value0` correctly re-cooks the switch.

### Cross-COMP wiring: selectTOP inside /composite
`inputConnectors.connect()` across base COMP boundaries silently fails. `selectTOP` nodes inside `/composite` reference external ops by absolute path and are wired locally.

### Mid-band filter chain: two AudioFilter CHOPs
`audiofilterCHOP` in `bandpass` mode has only one cutoff frequency. The mid band uses `highpass @ 300 Hz → lowpass @ 4 kHz` in series.

### beatCHOP outputs: explicit enable
`beatCHOP` only outputs `ramp` by default. `beat` and `pulse` channels require `beat_detect.par.beat = True` and `beat_detect.par.pulse = True`.

### mathCHOP clamp: use limitCHOP
`mathCHOP.par.postop` has no clamp option. A `limitCHOP` (type=clamp, min=0, max=1) is inserted after the math node.

### GLSL uniform binding: array0chop not per-uniform expressions
Per-uniform expressions (`ParMode.EXPRESSION`) fail with `TypeError: 'NoneType' object is not subscriptable` when the audio device is inactive (channels absent). `array0chop` binds a CHOP at cook time and handles missing channels gracefully.

## Risks / Trade-offs

- **Script TOP cook triggering**: Script TOPs only re-cook when their TOP inputs change. A `noiseTOP` (`noise_trigger`) is wired as input 0 to force every-frame cooking via its time-dependency. The noise values are not read — it is a cook-dependency trigger only.
- **Y-flip on copyNumpyArray**: `texelFetch` in GLSL uses y=0 at bottom (OpenGL convention); `copyNumpyArray` row 0 also maps to texture bottom. Pattern coordinates `(x, y)` map directly to `arr[y, x]` — no flip needed.
- **GoL state reads .val vs .eval()**: `par.val` returns the last cooked/cached value; `par.eval()` evaluates live. When an operator hasn't re-cooked, these differ. MCP inspection uses `.val`, which can make working expressions appear broken. Always use `par.eval()` to test live expression values.
