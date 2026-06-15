# Bloom — Technology & build reference

The authoritative description of *how Bloom works under the hood*. Read with `CLAUDE.md` (rules) and `PROCESS.md` (milestones). Pseudocode is TSL-flavoured; confirm exact method names against three r0.184.

---

## 1. The mental model

Everything that scales lives **on the GPU**:

- The **field** = an N×N grid of cells in a GPU **storage buffer**. Each cell holds two reacting chemicals + an owner.
- A **compute shader** advances the whole field every frame (reaction-diffusion), in parallel, reading each cell's neighbours.
- A second compute pass **scores** the field (territory per faction) via a tiled reduction.
- The **display** is one full-screen pass that reads the field and tints it per owner.

The CPU only ever:
- sets a handful of **uniforms** (brush position/radius/owner, per-faction recipes, dt, time),
- reads back a **tiny** score buffer ~3×/sec,
- runs the thin game layer (nutrient economy, UI, win check).

No per-cell CPU work. No per-frame readback.

---

## 2. Data layout

### Field (ping-pong storage buffers)
Two buffers, `bufA` and `bufB`, each `StorageBufferAttribute(N*N, 4)` — one **vec4 per cell**:

| component | meaning |
|---|---|
| `.x` = **A** | chemical A concentration (0..1), starts at 1 |
| `.y` = **B** | chemical B concentration (0..1), starts at 0 |
| `.z` = **owner** | faction id as a float: 0 = neutral, 1..K = players |
| `.w` = aux | reserved (age / catalyst strength — for later) |

Wrap as TSL storage nodes: `const fA = storage(bufA, 'vec4', N*N); const fB = storage(bufB, 'vec4', N*N);`

Cell index for (x, z): `i = z*N + x`. Neighbours with wrap-around (toroidal field): `(x±1)`, `(z±1)` mod N.

### Per-faction recipes (uniform)
A small uniform array, K+1 entries (index 0 = neutral). Each: `vec4(feed, kill, dA?, tintPacked)` — or split into separate `uniformArray`s for `feed`, `kill`, and a `color`. Keep it simple: a JS array of `{feed, kill, color:THREE.Color}` mirrored into uniforms the compute + display read by `owner`.

Two classic Gray-Scott regimes to start with (pick distinct ones per faction so they look/behave differently):

| regime | feed | kill | character |
|---|---|---|---|
| "mitosis" (dividing blobs) | 0.0367 | 0.0649 | blobs that split and fill space — aggressive |
| "coral" / "worms" | 0.0545 | 0.0620 | branching maze-like fronts — territorial |
| "spots/holes" | 0.030 | 0.062 | stable dots — defensive |

(There are many; expose feed/kill later as the per-faction "species".)

### Brush (uniform)
`uBrush = uniform(vec4)` → (x, z, radius, owner). Plus `uBrushActive = uniform(float)` (0/1) and `uBrushMode` (0 = seed, 1 = catalyst, 2 = inhibitor — modes 1/2 are *later*).

### Sim params (uniforms)
`uN` (float, grid size), `uDt` (~1.0), `uDa` (~1.0), `uDb` (~0.5), `uTime`.

---

## 3. Reaction-diffusion math (Gray-Scott)

For each cell, with Laplacian `L(·)` over neighbours:

```
A, B, owner = cell
recipe = recipes[owner]            // feed, kill  (use neutral's feed/kill where owner==0, e.g. feed=0)
La = L(A) ; Lb = L(B)
reaction = A * B * B
A' = A + (uDa*La - reaction + recipe.feed*(1 - A)) * uDt
B' = B + (uDb*Lb + reaction - (recipe.kill + recipe.feed)*B) * uDt
A' = clamp(A', 0, 1) ; B' = clamp(B', 0, 1)
```

**Laplacian** — 9-point kernel (sum of weighted neighbours minus centre):
```
L(v) = -1.0*center
     + 0.20*(left + right + up + down)
     + 0.05*(ul + ur + dl + dr)
```
Read each neighbour's `.x` (for A) / `.y` (for B) from the *input* buffer.

**Owner update (claim & combat)** — after computing A',B':
- If `B' > CLAIM_THRESHOLD` (e.g. 0.30) and the cell was neutral (`owner==0`), set `owner = ` the dominant neighbour's owner (or keep the seeding owner — see seeding).
- If `B'` drops below a small epsilon, decay `owner → 0` (territory lost).
- **Boundary combat:** when neighbours have different non-zero owners, the cell's owner follows whichever neighbouring owner has the **higher local B** (mass wins). This makes fronts push against each other. Keep the rule cheap (sample the 4 orthogonal neighbours' B and owner).

Write `vec4(A', B', owner', aux)` to the **output** buffer.

> Numerical note: initialise A=1, B=0 everywhere. Seed a few B=1 spots or the field stays dead. Run **several iterations per displayed frame** (e.g. 8–16) so patterns evolve at a watchable speed. Keep `uDt ≤ 1.0`. If you see NaNs, your Laplacian weights or dt are wrong — revert to the constants above.

---

## 4. The frame pipeline (ping-pong, ends on bufA)

Create these compute nodes once (they bake their storage nodes in):

- `seedA` — reads/writes **bufA**: applies the brush (if active) to bufA before stepping.
- `stepAB` — reads **bufA**, writes **bufB** (one RD iteration).
- `stepBA` — reads **bufB**, writes **bufA** (one RD iteration).
- `scoreFromA` — reads **bufA**, writes the small `scoreBuf` (tiled reduction).

Each animate frame:
```
1. update uniforms (uTime, uDt, brush from pointer, recipes if changed)
2. renderer.compute(seedA)                      // inject brush into bufA
3. for (s = 0; s < ITER; s++)                   // ITER must be EVEN (e.g. 8)
     renderer.compute(s % 2 === 0 ? stepAB : stepBA)
   // even ITER => final state is back in bufA
4. occasionally (throttle ~3/sec): renderer.compute(scoreFromA), then async getArrayBufferAsync(scoreBuf)
5. renderer.render(scene, camera)               // display reads bufA
```
Because `ITER` is even, the latest field is always in **bufA** at render time — so the **display material and the brush always target bufA** (no need to alternate them). This is the clean way to ping-pong with baked TSL nodes.

`ITER` is a quality/perf knob: higher = faster pattern growth + more GPU cost. Start at 8.

---

## 5. Display (full-screen, reads the field)

A full-screen quad (orthographic camera + a plane that fills the view, or a screen-space technique). Its `NodeMaterial.colorNode`:
```
uv01   = screen uv (0..1)
cell   = floor(uv01 * uN)
i      = cell.y * uN + cell.x
c      = fA.element(i)                 // vec4 (A,B,owner,_)
tint   = factionColor[ round(c.z) ]    // neutral = dark substrate colour
// shade by B (pattern density); maybe rim/age via aux
color  = mix(substrateColor, tint, smoothstep(0.1, 0.5, c.y))
```
**VERIFY FIRST (milestone 1):** reading the storage buffer `fA.element(i)` in a *fragment* node. If it doesn't work, switch the sim to also write an output **storage texture** (rgba) that display samples normally — same math, different display source.

The look: substrate = dark; each faction's living B-pattern glows in its colour; contested fronts shimmer. This is where Bloom earns its name.

---

## 6. Scoring (GPU reduction, tiny readback)

Goal: territory cell-count per faction, a few times a second, without atomics and without a big readback.

- `scoreBuf = StorageBufferAttribute(TILES * (K+1), 1)` (float). Choose `TILES` ≈ 1024.
- `scoreFromA` compute dispatches **TILES** threads. Thread `t` loops over its slice of cells `[t*span, (t+1)*span)`, counts how many have each `owner` (a small local array of K+1 counts), and writes them to `scoreBuf[t*(K+1) + owner]`. **Each thread owns its output slots → no atomics, no races.**
- CPU: `getArrayBufferAsync(scoreBuf)` (TILES×(K+1) floats — e.g. 1024×4 = 16 KB, throttled to ~3/sec → negligible), then sum across tiles to get per-faction totals → territory %. Drive the HUD and win check from this.

(Atomics would let you skip the CPU sum, but they're unverified here — start with this.)

---

## 7. Input

- Pointer on the canvas → set `uBrush` = (tileX, tileZ, radius, currentFactionOwner), `uBrushActive = 1` while dragging, else 0. Tile from screen: invert the display's uv→cell mapping (it's a flat full-screen field, so it's just `floor(pointerUV * N)` — no raycasting needed).
- The `seedA` compute reads these uniforms and stamps B=1/owner within the radius. **All seeding is uniform-driven; never write the field buffer from the CPU mid-ping-pong.**
- Spending nutrient: the CPU game layer gates whether `uBrushActive` is allowed (enough nutrient) and decrements it per stamped area.

---

## 8. Performance levers

- **Grid size N** (e.g. 512, 1024, 2048). Cost scales with N² × ITER. 1024² × 8 iters = ~8.4M cell-updates/frame — comfortable on a discrete GPU; verify with the GPU-ms readout.
- **ITER per frame** — pattern speed vs cost.
- **Score frequency** — keep at ~3/sec.
- Report **GPU ms** (timestamp API), not FPS (vsync-capped). See `CLAUDE.md`.

---

## 9. HUD / telemetry

Minimal overlay (HTML/CSS, `pointer-events:none` except buttons): backend (WebGPU/none), **GPU ms**, grid N, ITER, per-faction **territory %** (from §6), nutrient, current tool/faction, and faction recipe buttons. This mirrors the sibling project's HUD discipline.

---

## 10. What pushes WebGPU here (and why WebGL2 can't)

- **Compute shaders** advance millions of cells/frame in parallel (WebGL2 has none — you'd be stuck with fragment-ping-pong, which can't do the reduction/owner-logic cleanly and doesn't scale the same).
- **Storage buffers** hold structured per-cell state read/written arbitrarily by the compute pass.
- **GPU reduction** scores the field without dragging it back to the CPU.
- The result: the entire world + scoring is GPU-resident, the CPU just paints intent — exactly the architecture this whole project explored.
