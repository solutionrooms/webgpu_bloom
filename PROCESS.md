# Bloom — build process (milestones with verify-gates)

Build in this order. **Each milestone ends with a concrete in-browser check on a FOCUSED tab. Do not start the next milestone until the current one is verified.** This discipline is the single biggest reason the sibling project succeeded after early all-at-once attempts failed. Commit after each green milestone.

> Reminder (see `CLAUDE.md`): a backgrounded tab pauses the loop and corrupts GPU timing — verify with the tab focused. Report **GPU ms**, not FPS.

---

## M0 — Scaffold & WebGPU boot
**Build:** `index.html` with the exact import map (`CLAUDE.md`), a `WebGPURenderer` (`await renderer.init()`), an orthographic camera + full-screen plane with a flat `MeshBasicNodeMaterial`, `setAnimationLoop`, and a minimal HUD (backend label). If WebGPU is unavailable, show a "needs WebGPU" message.
**Verify:** page loads in Chrome with **no console errors**; HUD shows `Backend: WebGPU`; the plane renders a solid colour.
**Commit:** "M0 scaffold + WebGPU boot".

## M1 — Field buffer + display (THE risk gate)
**Build:** allocate `bufA`/`bufB` (`StorageBufferAttribute(N*N, 4)`), fill bufA with A=1, B=0, owner=0, plus a handful of random B=1 seed spots. Wire the **display** colorNode to read `fA.element(cell)` and tint by B (per `TECHNOLOGY.md §5`). No simulation yet — just display the static seeded field.
**Verify:** the seed spots are visible as bright dots on a dark field, at the correct positions. **This proves the fragment-stage storage-buffer read works.** If it doesn't, switch display to the storage-texture fallback now (don't proceed on a broken display).
**Commit:** "M1 field buffer + display verified".

## M2 — Reaction-diffusion compute (the heart)
**Build:** `stepAB`/`stepBA` compute kernels implementing Gray-Scott + the 9-point Laplacian with the exact constants in `TECHNOLOGY.md §3` (single faction/recipe for now; ignore owner combat). Ping-pong an **even** ITER per frame ending in bufA.
**Verify:** the seeds **grow into living patterns** (with the "mitosis" recipe you should see dividing blobs; with "coral", branching fronts). The field is neither dead-uniform nor NaN-blown-out. Tune ITER so it evolves at a watchable pace.
**Commit:** "M2 reaction-diffusion sim".
**Pitfall:** if dead → no seeds / feed too low; if exploding → dt or Laplacian weights wrong. Use the given constants verbatim first.

## M3 — Seeding via brush (input)
**Build:** pointer → `uBrush`/`uBrushActive` uniforms; a `seedA` compute that stamps B=1/owner within the brush radius into bufA before the sim steps. Left-drag to paint. Add brush-size buttons.
**Verify:** left-drag **plants a bloom that then spreads** from where you painted; brush sizes change the stamp.
**Commit:** "M3 brush seeding".

## M4 — Factions, ownership & combat
**Build:** the owner channel + per-faction recipe uniforms (≥2 factions with distinct feed/kill + colour); the claim/decay + boundary-combat rule (`TECHNOLOGY.md §3`). Faction buttons to choose what you seed. Seed two factions.
**Verify:** two differently-coloured blooms grow with different characters and **visibly contest a moving boundary**; owners flip along the front; colours are correct.
**Commit:** "M4 factions + combat".

## M5 — Scoring (GPU reduction + tiny readback)
**Build:** `scoreBuf` + the tiled `scoreFromA` reduction (`TECHNOLOGY.md §6`); throttle `getArrayBufferAsync` to ~3/sec; CPU sums tiles → per-faction territory %. HUD territory bars.
**Verify:** territory bars move as blooms grow/shrink and **sum to ~100%**; numbers look right vs the visible field; **no per-frame stalls** (GPU ms steady — the readback is throttled).
**Commit:** "M5 GPU scoring".

## M6 — Game layer: nutrient, AI, win
**Build:** nutrient resource (regen ∝ territory), gating seeding; a simple scripted AI faction (seed into large neutral regions on a timer — can use the coarse per-tile score map); win condition (≥X% or timer); restart.
**Verify:** a full round is **playable end-to-end** — seed, spread, contest the AI, snowball or get upset, win/lose, restart.
**Commit:** "M6 first playable".

## M7 — Telemetry, polish, perf
**Build:** GPU-ms timer (timestamp API, guarded), grid-size + ITER knobs, contested-front shimmer (aux fade), HUD tidy. Confirm it holds the refresh rate at 1080p at the target grid; note the GPU-ms cost per grid/ITER.
**Verify:** smooth at target settings; perf scales sensibly with N and ITER; no console errors; AC-style pass of the gotchas in `CLAUDE.md`.
**Commit:** "M7 polish + perf".

---

## After the first playable
Hand back to the owner for gameplay tailoring (recipes = the meta, economy, win conditions). Then optionally: catalyst/inhibitor tools, smarter AI from the coarse map, post-processing bloom, audio, more modes. Each as its own milestone with its own verify-gate.

## Verification toolkit (how to actually check)
- **In-browser, focused tab:** console for errors, a screenshot for the field, HUD numbers for state.
- **A manual probe** for things the throttled-loop hides: expose a tiny `window.__step()` that runs one compute+render synchronously and (for scoring) `await renderer.getArrayBufferAsync(...)` so you can confirm reductions/edits without depending on the animation loop running. Remove before shipping. (This pattern — a synchronous on-demand step — is exactly how the sibling project verified compute culling in a throttled tab.)
