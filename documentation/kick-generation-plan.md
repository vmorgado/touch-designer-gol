# Kick-Gated GoL Generation Plan

**Date**: 2026-05-26 — **Status: Executed**  
**Goal**: Each kick hit advances the GoL simulation by exactly one generation. Between kicks the grid is frozen. Cell seeding from audio bands also only fires on kick frames.

---

## 1. Current Behaviour

- `gol_compute` (GLSL TOP) runs its pixel shader **every frame** — one new GoL generation per frame at ~60 fps.
- `seed_top` (Script TOP) runs every frame: band seeding (random cells from `low/mid/high`) fires unconditionally; pattern injection is already gated behind `beat_rising`.
- Dead-grid recovery fires after 2 s of silence to inject a methuselah pattern.

---

## 2. Target Behaviour

- **GoL advances once per kick hit** — the pixel shader freezes (outputs current state) unless `uAdvance = 1.0` for that frame.
- **Seeding only fires on kick** — both band seeding and pattern injection run on the same kick rising-edge frame.
- **Dead-grid recovery removed** — between kicks the grid is intentionally frozen, not dead. If the grid dies during a live session a subsequent kick will inject a pattern via the existing beat-triggered injection.

---

## 3. Approach

Two parallel changes work together:

| Layer | Mechanism | What it controls |
|---|---|---|
| CHOP network | Rising-edge pulse from `kick` → `uAdvance` uniform | Tells the GLSL shader whether to advance this frame |
| Script TOP | `beat_rising` guard on all seeding | Only generates new alive cells on kick frames |

No Switch TOP or Feedback TOP restructuring needed — the feedback loop topology stays identical.

---

## 4. Network Changes

### 4.1 New CHOP chain in `/project1/gol`

```mermaid
flowchart LR
    OAV[/project1/audio/out_audio_vals] -->|kick channel| KS[kick_sel\nSelect CHOP]
    KS --> KP[kick_pulse\nLogic CHOP\nrising edge mode]
    KP --> KG[kick_gate\nNull CHOP]
    KG -->|uAdvance uniform| GC[gol_compute\nGLSL TOP]
```

| Operator | Type | Configuration |
|---|---|---|
| `kick_sel` | Select CHOP | CHOP = `/project1/audio/out_audio_vals`, Channel = `kick` |
| `kick_pulse` | Logic CHOP | Function = **Pulse on Rising Edge** (converts sustained kick envelope to a single-frame 1.0 per hit) |
| `kick_gate` | Null CHOP | Clean endpoint for uniform reference |

### 4.2 `gol_compute` GLSL TOP — add `uAdvance` uniform

In `gol_compute`, add a new **CHOP uniform**:

| Field | Value |
|---|---|
| Name | `uAdvance` |
| CHOP | `kick_gate` |
| Channel | `chan1` |

---

## 5. Shader Change — `gol_compute_pixel`

Add `uniform float uAdvance;` and wrap the GoL rules in a branch. When `uAdvance < 0.5` the shader outputs the current state unchanged (freeze). When `uAdvance >= 0.5` it runs the full GoL rules and applies the seed texture.

**Before (simplified):**
```glsl
// Conway rules always run
float next = 0.0;
if (current > 0.5) {
    next = (neighbours == 2 || neighbours == 3) ? 1.0 : 0.0;
} else {
    next = (neighbours == 3) ? 1.0 : 0.0;
}
float seed = texelFetch(sTD2DInputs[1], coord, 0).r;
next = max(next, seed);
fragColor = TDOutputSwizzle(vec4(next, next, next, 1.0));
```

**After:**
```glsl
uniform float uAdvance;

// ...neighbour count stays the same...

float next;
if (uAdvance >= 0.5) {
    // Advance one generation
    if (current > 0.5) {
        next = (neighbours == 2 || neighbours == 3) ? 1.0 : 0.0;
    } else {
        next = (neighbours == 3) ? 1.0 : 0.0;
    }
    float seed = texelFetch(sTD2DInputs[1], coord, 0).r;
    next = max(next, seed);
} else {
    // Frozen — pass current state through unchanged
    next = current;
}
fragColor = TDOutputSwizzle(vec4(next, next, next, 1.0));
```

---

## 6. Script Change — `seed_top_callbacks`

Two changes:

**A — Gate band seeding behind `beat_rising`**

Currently the per-frame band seeding runs unconditionally. Move it inside the existing `if beat_rising:` block so it only fires on kick frames.

**Before:**
```python
n_low  = int(low  * 20)
n_mid  = int(mid  * 12)
n_high = int(high * 8)

for _ in range(n_low + n_mid + n_high):
    x = random.randint(0, 31)
    y = random.randint(0, 31)
    seed[y, x, 0] = 1.0
    seed[y, x, 3] = 1.0

if beat_rising:
    # pattern injection ...
```

**After:**
```python
if beat_rising:
    # Band seeding
    n_low  = int(low  * 20)
    n_mid  = int(mid  * 12)
    n_high = int(high * 8)

    for _ in range(n_low + n_mid + n_high):
        x = random.randint(0, 31)
        y = random.randint(0, 31)
        seed[y, x, 0] = 1.0
        seed[y, x, 3] = 1.0

    # Pattern injection (already here, unchanged)
    # ...
```

**B — Remove dead-grid recovery**

Delete the entire dead-grid recovery block (the `total_audio < 0.15` / `dead_timer` logic). The grid is intentionally paused between kicks — a frozen grid is not a dead grid. Remove `dead_timer` from both the `scriptOp.fetch` and `scriptOp.store` calls.

---

## 7. Execution Steps

1. **Create** `kick_sel` Select CHOP in `/project1/gol` — pull `kick` from `/project1/audio/out_audio_vals`.
2. **Create** `kick_pulse` Logic CHOP — rising-edge pulse mode, wired from `kick_sel`.
3. **Create** `kick_gate` Null CHOP — wired from `kick_pulse`.
4. **Add** CHOP uniform `uAdvance` on `gol_compute`, referencing `kick_gate/chan1`.
5. **Edit** `gol_compute_pixel` shader — add `uniform float uAdvance;` and branch on it.
6. **Edit** `seed_top_callbacks` — move band seeding inside `if beat_rising:`.
7. **Edit** `seed_top_callbacks` — remove dead-grid recovery block and `dead_timer` state.
8. **Verify** — with no audio playing, the grid should be completely frozen. On a kick hit, it should advance one frame and freeze again.

---

## 8. What Is Not Changing

| Component | Status |
|---|---|
| Feedback TOP topology | Unchanged |
| `gol_compute` input wiring | Unchanged |
| Pattern injection logic | Unchanged (already behind `beat_rising`) |
| `pattern_ctrl` Script CHOP | Unchanged (monitoring only, no downstream connections) |
| `sel_audio` → `out_audio_vals` path | Unchanged |
