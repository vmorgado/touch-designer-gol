# TouchDesigner — Audio-Reactive Game of Life

## Project Overview

A live-performance visual system built in TouchDesigner. A 32×32 grid of 3D cubes simulates Conway's Game of Life, driven by real-time audio analysis. Alive cells rise and reveal a switchable underlayer (video or generative visuals); dead cells sink and become semi-transparent.

## Repository Structure

```
touch-designer-claude/
├── CLAUDE.md                  # This file
├── documentation/
│   └── requirements.md        # Full project requirements
└── (TouchDesigner .toe files will live here)
```

## Key Design Decisions

- **Grid**: 32×32 instanced geometry (SOPs rendered via Instancing or GLSL-based approach)
- **Audio source**: Microphone / line-in via AudioDeviceIn CHOP
- **Audio → GoL mapping**:
  - Frequency bands (bass/mid/treble) randomly seed cells proportional to band energy
  - Beat/transient detection reseeds or perturbs the grid state
- **Cell visuals**:
  - Alive → cube at full height, opaque, masks underlayer
  - Dead → cube sunk/flat, semi-transparent
- **Underlayer**: switchable between video/media playback and generative GLSL shader
- **Output**: real-time, single display, live VJ / performance context

## Network Architecture (planned)

| Component | Operator type | Role |
|---|---|---|
| Audio analysis | CHOP network | FFT, band split, beat detect |
| GoL simulation | TOP (pixel = cell state) | GLSL compute shader for GoL rules |
| Audio → seed | CHOP → DAT/Script | Map band energy to cell seeds |
| Grid geometry | SOP + Instancing | 32×32 cube instances |
| Height mapping | CHOP → Instance | Drive Y-scale and Y-position per instance |
| Underlayer | Movie File In TOP / GLSL TOP | Switchable source |
| Masking | TOP composite | Alive cells punch through to underlayer |
| Output | Window COMP | Fullscreen output |

## Working with Claude

- Always use Mermaid (` ```mermaid `) for any diagrams or flowcharts — never ASCII art
- Always read `documentation/requirements.md` before starting any new network component
- Prefer GLSL-based compute for the Game of Life grid (TOP pixel-per-cell pattern)
- Use instancing for the 32×32 cube grid — avoid 1024 separate geometry COMPs
- Keep audio analysis in a dedicated `/audio` base COMP
- Keep GoL logic in a dedicated `/gol` base COMP
- Keep rendering in `/render` base COMP
