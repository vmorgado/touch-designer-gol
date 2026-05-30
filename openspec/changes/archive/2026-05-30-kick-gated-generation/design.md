## Context

Previously `gol_compute` ran GoL rules on every frame. The kick channel from `audioAnalysis` is a sustained envelope (0→1 as the kick sustains), not a single-frame pulse. A Logic CHOP in "pulse on rising edge" mode converts it to a guaranteed single-frame `1.0` signal, which is then used as the `uAdvance` shader uniform.

## Goals / Non-Goals

**Goals:**
- GoL frozen (state passed through unchanged) when `uAdvance < 0.5`
- GoL advances exactly one generation when `uAdvance ≥ 0.5`
- Seed texture still applied every frame (snare/rythm seeds persist in frozen state)
- No topological change to the feedback loop

**Non-Goals:**
- Multi-generation advance per kick (always exactly one)
- BPM-derived automatic tick rate (kick-only)

## Decisions

### uAdvance uniform via choptoTOP
`kick_gate` (Null CHOP) feeds a `kick_top` choptoTOP (1×1 pixel). `gol_compute` reads `uAdvance` from this TOP via a CHOP uniform binding. When the kick pulse fires (`kick_pulse` Logic CHOP rising-edge output), `uAdvance = 1.0` for that frame; otherwise `0.0`.

### Seed texture applied regardless of uAdvance
The GoL pixel shader always executes `next = max(next, seed)` even when frozen. This is intentional — snare and rythm can inject alive cells into a frozen grid, and those cells will be present when the next kick advances the simulation.

### Dead-grid recovery removed
Between kicks the grid is intentionally paused — a frozen grid at any density is not dead. If the grid dies (all cells dead), the next kick's pattern injection will restart it. Removing the recovery timer simplifies the script and eliminates spurious injections during quiet musical passages.

## Risks / Trade-offs

- If kicks are very infrequent (slow BPM or sparse percussion), the grid can appear completely static for long periods. This is a deliberate trade-off for the kick-visual synchrony.
- The Logic CHOP `kick_pulse` must be set to "Pulse on Rising Edge" mode (`preop = rise`). Other Logic modes will not produce a single-frame pulse from the sustained kick envelope.
