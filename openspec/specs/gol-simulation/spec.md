# gol-simulation Specification

## Purpose
TBD - created by archiving change core-system. Update Purpose after archive.
## Requirements
### Requirement: 32×32 grid stored as GPU texture
The Game of Life state is a 32×32 pixel RGBA32Float texture. Each pixel's R channel encodes cell state: `1.0` = alive, `0.0` = dead.

#### Scenario: Initial state
- **WHEN** the network starts or a reset is triggered
- **THEN** the grid initialises with a sparse random pattern

### Requirement: Conway's Game of Life rules applied via GLSL pixel shader
**Previous**: One generation computed per frame (~60 fps).
**New**: GoL advances exactly **one generation per kick hit**. Between kicks the grid is completely frozen (current state passed through unchanged).

The `gol_compute` pixel shader reads `uAdvance` uniform:
- `uAdvance ≥ 0.5` → apply Conway rules + seed injection for one generation
- `uAdvance < 0.5` → output current state unchanged (frozen)

`uAdvance` is driven by `kick_top` (1×1 choptoTOP from `kick_gate` Null CHOP).

#### Scenario: Kick fires
- **WHEN** `kick_pulse` Logic CHOP outputs 1.0 on a kick rising edge
- **THEN** `uAdvance = 1.0` for that frame; GoL advances one generation

#### Scenario: No kick
- **WHEN** `kick_pulse` is 0.0
- **THEN** `uAdvance = 0.0`; `gol_compute` outputs current state unchanged

### Requirement: GPU feedback loop
Current state feeds back into itself each generation via a `Feedback TOP`. The feedback reference is `out_state` (Null TOP output of `gol_compute`).

```
const_init → feedbackTOP → gol_compute → out_state
                   ↑                          |
                   └──── par.top = out_state ─┘
```

#### Scenario: Feedback active
- **WHEN** `gol_compute` produces a new frame
- **THEN** `feedbackTOP` passes it as input on the next frame

### Requirement: Seed texture injection
`gol_compute` accepts a 32×32 seed texture as input 1. Seed cells are OR-merged into the computed next state (`next = max(next, seed)`), regardless of GoL rules.

#### Scenario: Seed injection
- **WHEN** the seed texture has a 1.0 pixel at position (x, y)
- **THEN** cell (x, y) is forced alive in the output, overriding the GoL rule result

### Requirement: Manual grid reset
The `feedbackTOP`'s `resetpulse` parameter triggers a full grid reset, returning all cells to 0.

#### Scenario: Reset triggered
- **WHEN** `feedback.par.resetpulse.pulse()` is called
- **THEN** all cells return to dead state on the next frame

### Requirement: Kick rising-edge pulse generation
`kick_sel` (Select CHOP) reads the `kick` channel from `/audio/out_audio_vals`. `kick_pulse` (Logic CHOP, preop = rise) converts the sustained kick envelope into a guaranteed single-frame 1.0 pulse per hit. `kick_gate` (Null CHOP) is the clean endpoint referenced by the `uAdvance` uniform.

