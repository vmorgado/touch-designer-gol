## ADDED Requirements

### Requirement: 32×32 cube grid via geometry instancing
1024 cubes are rendered using a single Geometry COMP (`geo1`) with instancing enabled. No separate COMP per cube. Grid is laid out on the XZ plane, cubes extend upward on Y.

#### Scenario: Network renders
- **WHEN** `render1` Render TOP cooks
- **THEN** exactly 1024 cube instances appear at their correct XZ positions

### Requirement: Per-cell height driven by GoL state
Each cube's Y scale is determined by its cell's GoL state. Height values are sampled from the smoothed state texture in the vertex shader.

| State | Y scale |
|-------|---------|
| Alive (1.0) | 1.0 (full height) |
| Dead (0.0) | `uDeadHeight` (default 0.05) |

Y position is offset so the cube base stays at Y=0 regardless of scale.

#### Scenario: Alive cell
- **WHEN** a cell's smoothed state is 1.0
- **THEN** its cube renders at full height, fully opaque

#### Scenario: Dead cell
- **WHEN** a cell's smoothed state is 0.0
- **THEN** its cube is flat (height = `uDeadHeight`) and fully invisible (alpha = 0)

### Requirement: Underlayer colour sampled per cell in vertex shader
Each cube samples the underlayer texture at its cell's grid UV `((col + 0.5) / 32, (row + 0.5) / 32)` in the vertex shader. This colour is passed as a varying to the fragment shader.

- Alive cubes render the sampled underlayer colour at full opacity
- Dead cubes have alpha = 0 (underlayer not visible through them)

#### Scenario: Underlayer colour applied
- **WHEN** a cell is alive
- **THEN** its cube face shows the underlayer colour at that grid cell position, independent of camera angle

### Requirement: GLSL MAT with two sampler uniforms
`cube_mat` is a GLSL MAT with:
- `sampler0` (`sGoLState`) → `/render/smooth_out` (smoothed state)
- `sampler1` (`sUnderlayer`) → `/underlayer/out_underlayer`

Cell coordinates are derived from `TDInstanceID()` as `col = id % 32`, `row = id / 32`.

### Requirement: Camera positioned for isometric-style view
`cam1` Camera COMP: tx=0, ty=40, tz=20, rx=-60. One directional light and one ambient light drive the render.
