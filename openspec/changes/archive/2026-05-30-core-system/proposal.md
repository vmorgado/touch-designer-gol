## Why

Build a live-performance visual system in TouchDesigner where a 32×32 grid of 3D cubes simulates Conway's Game of Life, seeded and perturbed by real-time audio analysis. The system must run at stable 60 FPS with a single fullscreen output for live VJ / performance use.

## What Changes

- Introduce a 6-COMP TouchDesigner network (`/audio`, `/gol`, `/render`, `/underlayer`, `/composite`, `/output`)
- Audio input (microphone / line-in) is analyzed for frequency bands (bass / mid / treble) and beat detection
- A 32×32 GLSL pixel-shader computes Conway's GoL with a GPU feedback loop
- Band energy seeds random alive cells into the GoL each frame; beat detection triggers pattern injection
- 1024 cubes are rendered via geometry instancing; alive cubes show an underlayer texture on their faces; dead cubes are invisible
- Underlayer is switchable between video playback and a generative Perlin noise shader
- Output is a single Window COMP for fullscreen display

## Capabilities

### New Capabilities

- `audio-analysis`: Microphone/line-in capture, per-frame frequency band extraction (bass/mid/treble) and beat pulse detection
- `gol-simulation`: GPU-based Conway's Game of Life on a 32×32 RGBA32Float texture with feedback loop
- `audio-gol-seeding`: Maps audio band energy to random cell seeding and beat pulses to pattern injection
- `instanced-rendering`: 32×32 cube grid via geometry instancing with GLSL MAT driving per-cell height and underlayer colour
- `underlayer`: Switchable background source (video or generative GLSL shader) embedded in cube material
- `smooth-state`: Per-frame lerp feedback loop that smoothly animates GoL cell state transitions
- `camera-shake`: `fmsd`-driven X-axis jitter on the render camera proportional to audio transient energy

### Modified Capabilities

<!-- None — all capabilities are new -->

## Impact

- Requires TouchDesigner build 2023+ with GLSL compute and Python scripting enabled
- GPU with GLSL 4.5+ support required for GLSL TOP pixel shaders
- Microphone or line-in audio device required for live audio input
- Single output display; no multi-output support in scope
