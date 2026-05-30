# runtime-tuning Specification

## Purpose
Reference for all parameters that should be adjusted during live performance or rehearsal. These are the knobs that affect visual output without touching code or network topology.

## Requirements

### Requirement: Audio analysis thresholds
Inside the `audioAnalysis` container at `/project1/audio/audioAnalysis`, each band has its own gain, smoothing, and threshold sliders. These are the first things to adjust when kicks/snares/rythm fire too often or not enough.

| What to adjust | Where | Effect |
|----------------|-------|--------|
| Kick threshold | `audioAnalysis` → kick sub-container | Raise if false positives on bass notes; lower if kicks are missed |
| Snare threshold | `audioAnalysis` → snare sub-container | As above for snare |
| Rythm threshold | `audioAnalysis` → rythm sub-container | As above for broad rhythmic events |
| Per-band gain | `audioAnalysis` → low/mid/high containers | Scales RMS energy before normalisation |

### Requirement: GoL pattern injection rate
`/project1/gol/ctrl_seeds` (Constant CHOP), channel `patterns_per_beat` — default **2**.

Controls how many canonical GoL patterns are stamped per kick hit. Increase for denser seeding at low BPM; decrease for less visual chaos at high BPM.

#### Scenario: Grid goes quiet during set
- **WHEN** the grid looks sparse and patterns are not spreading
- **THEN** raise `patterns_per_beat` to 3–4 to inject more structure per kick

### Requirement: Manual grid reset
`/project1/gol/feedback`, parameter `resetpulse` — pulse to trigger.

Wipes all cells to dead state immediately. Use during performance to start a fresh evolution from the next kick.

#### Scenario: Grid reaches a dead still-life
- **WHEN** all cells are static (no alive neighbours at edge counts that change)
- **THEN** pulse `feedback.par.resetpulse` — next kick will re-seed from scratch

### Requirement: Underlayer source switching
`/project1/underlayer/ctrl_switch` (Constant CHOP), channel `index`:
- `0` → `video_src` (Movie File In TOP — video or image playback)
- `1` → `gen_src` (generative Perlin noise shader — default)

**Do not use the `Srcindex` custom parameter on the `/underlayer` base COMP** — custom COMP parameters do not register as cook dependencies on downstream TOPs and the switch will not update. Use `ctrl_switch.par.value0` instead.

#### Scenario: Switch to video during performance
- **WHEN** `ctrl_switch.par.value0` is set to 0
- **THEN** `src_switch` re-cooks and outputs the video source without interrupting GoL simulation

### Requirement: Video source file
`/project1/underlayer/video_src`, parameter `file` — drop a video clip or image path here. The clip plays and loops continuously. Can be hot-swapped during performance.

### Requirement: Dead cube height
`/project1/render/cube_mat`, parameter `vec0valuex` (uniform `uDeadHeight`) — default **0.05**.

Controls the Y scale of dead cubes. Lower = flatter grid, more visual contrast between alive and dead. Range 0.01–0.5 is practical.

#### Scenario: Grid reads as too flat
- **WHEN** dead cubes are hard to distinguish from the background
- **THEN** raise `uDeadHeight` to 0.1–0.2 to give them more visible presence

### Requirement: Cell transition smoothing speed
`/project1/render/smooth_state`, uniform `uLerpSpeed` — default **0.15**.

Controls how fast cells animate between alive and dead states. At 60 fps, 0.15 gives ~0.3 s transitions. Lower = slower, dreamier. Higher = snappier, approaching instant.

#### Scenario: Transitions feel laggy at high BPM
- **WHEN** kicks are frequent and cells haven't finished animating before the next hit
- **THEN** raise `uLerpSpeed` to 0.25–0.4 for faster transitions

### Requirement: Camera position
`/project1/render/cam1` parameters: `tx` (X), `ty` (Y), `tz` (Z), `rx` (X rotation).

Default: tx=0, ty=40, tz=20, rx=−60 (isometric-style view of the XZ grid).

Note: `cam1.par.tx` is driven by the camera shake expression. Setting `tx` directly while shake is active will conflict — adjust `fmsd_math` gain instead to reduce shake magnitude without changing rest position.

### Requirement: Camera shake magnitude
`/project1/render/fmsd_math` (Math CHOP), multiply value — default **0.2** (abs(fmsd) × 0.2).

`/project1/render/fmsd_limit` (Limit CHOP), max clamp — default **0.5**.

Lower the multiply or max clamp to reduce shake intensity. Set max clamp to 0 to disable shake entirely without breaking the expression.
