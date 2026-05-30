## ADDED Requirements

### Requirement: Lerp feedback loop for smooth cell transitions
A `smooth_state` GLSL TOP in `/render` interpolates between the previous smoothed state and the current raw GoL state each frame using `mix(current, target, uLerpSpeed)` with `uLerpSpeed = 0.15`.

```
raw_state_ref (Select TOP → /gol/out_state)
    ↓ (input 1 — target)
smooth_feedback (Feedback TOP) → smooth_state (GLSL TOP) → smooth_out (Null TOP)
    ↑                               mix(current, target, 0.15)
    └───────────────────────────────────────────────────────┘
```

#### Scenario: Cell becomes alive
- **WHEN** a dead cell transitions to alive in the raw GoL state
- **THEN** `smooth_out` ramps from 0.0 to 1.0 over approximately 0.3 seconds (at 60 fps)

#### Scenario: Cell dies
- **WHEN** an alive cell transitions to dead in the raw GoL state
- **THEN** `smooth_out` ramps from 1.0 to 0.0 over approximately 0.3 seconds

### Requirement: Raw GoL feedback loop is unaffected
`raw_state_ref` always reads the raw binary `/gol/out_state` directly. The smooth lerp loop is entirely separate — the GoL compute loop operates on binary state only.

### Requirement: Cube material reads smoothed state
`cube_mat.sampler0top` and `gol_state_ref` both reference `smooth_out`, not `out_state`. The cube height and alpha are driven by the smoothed value.

### Requirement: Lerp speed is tunable
`uLerpSpeed` is a vec uniform on `smooth_state`. Default 0.15 produces ~0.3 s transitions at 60 fps. Values closer to 1.0 make transitions instant; values closer to 0.0 make them slower.
