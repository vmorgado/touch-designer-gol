## ADDED Requirements

### Requirement: 32×32 grid stored as GPU texture
The Game of Life state is a 32×32 pixel RGBA32Float texture. Each pixel's R channel encodes cell state: `1.0` = alive, `0.0` = dead.

#### Scenario: Initial state
- **WHEN** the network starts or a reset is triggered
- **THEN** the grid initialises with a sparse random pattern

### Requirement: Conway's Game of Life rules applied via GLSL pixel shader
A `GLSL TOP` (`gol_compute`) applies standard Conway rules each generation using a pixel shader. One pixel = one cell. Neighbours are counted with torus-wrapping at grid edges.

Rules:
1. Alive cell with 2 or 3 alive neighbours → survives
2. Dead cell with exactly 3 alive neighbours → becomes alive
3. All other cells → die or remain dead

#### Scenario: Stable pattern
- **WHEN** the grid contains a 2×2 block (still life)
- **THEN** the block remains unchanged across generations

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
