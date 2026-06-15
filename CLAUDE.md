# Bloom — build rules (read this first)

You are building **Bloom**, a WebGPU reaction-diffusion RTS. Spec is in `GAMEPLAY.md`, `TECHNOLOGY.md`, `PROCESS.md`. This file is the non-negotiable cheat-sheet so you don't repeat known mistakes.

## Golden rules

1. **Build incrementally, verify each milestone in the browser before the next.** Follow `PROCESS.md`. Do **not** write the whole app then debug — that path failed repeatedly in the sibling project. Each milestone ends with a concrete in-browser check.
2. **The GPU sim is the gameplay.** Keep CPU work to: sparse brush writes (as *uniforms*, not buffer writes), and an infrequent score readback. **Never** read the field back to the CPU per frame.
3. **Single self-contained `index.html`**, no build step, exact import map below.
4. **WebGPU is required for the sim.** Detect it; if absent, show a clear "needs WebGPU" message rather than half-running.

## Exact import map (do not change versions or paths)

```html
<script type="importmap">
{
  "imports": {
    "three": "https://unpkg.com/three@0.184.0/build/three.webgpu.js",
    "three/webgpu": "https://unpkg.com/three@0.184.0/build/three.webgpu.js",
    "three/tsl": "https://unpkg.com/three@0.184.0/build/three.tsl.js",
    "three/addons/": "https://unpkg.com/three@0.184.0/examples/jsm/"
  }
}
</script>
```
`import * as THREE from 'three';` → the WebGPU build. `import { Fn, instanceIndex, storage, uniform, float, vec2, vec4, If, ... } from 'three/tsl';`.

## VERIFIED patterns (proven on hardware — copy these shapes)

- **Renderer init is async:** `const renderer = new THREE.WebGPURenderer({ antialias: true }); await renderer.init();` (top-level `await` works in a module). Backend check: `renderer.backend.isWebGPUBackend === true`.
- **Loop:** `renderer.setAnimationLoop(animate)`. Inside: `renderer.compute(node)` (queues a compute pass, ordered before render) then `renderer.render(scene, camera)`.
- **Storage buffer:** `const buf = new THREE.StorageBufferAttribute(count, itemSize);` — `.array` is the typed array; after CPU writes set `buf.needsUpdate = true`. Wrap for TSL: `const s = storage(buf, 'vec4', count);` then `s.element(indexNode)` to read/`.assign()` to write.
- **Compute kernel:** `const k = Fn(() => { const i = instanceIndex; /* read s.element(i), neighbours, write out.element(i).assign(...) */ }); const node = k().compute(count); renderer.compute(node);`. Use `.toVar()` for mutable locals, `If(cond, () => {...})` for branches.
- **Score readback (the ONLY readback):** `const ab = await renderer.getArrayBufferAsync(scoreBuf); const arr = new Float32Array(ab);` — throttle to ~3×/sec, guard with a `pending` flag, wrap in try/catch.
- **GPU timing (optional HUD):** `renderer.backend.trackTimestamp = true;` then `await renderer.resolveTimestampsAsync(THREE.TimestampQuery.RENDER)` → ms. Guard it; **ignore absurd values** (e.g. > 2000 ms) — they appear in throttled/backgrounded tabs.
- **Texture-array sampling (if you use atlases):** `texture(arrayTex, uv).depth(layerNode)` — `.depth()` selects the array layer; compiles on both WGSL and GLSL.

## VERIFY FIRST (standard but unproven here — confirm in milestone 1)

- **Reading a storage buffer in a *fragment* node** (for display). Likely fine (read-only storage in fragment is standard WGSL). If it misbehaves, fall back to a **storage texture** the compute writes (`textureStore`) and sample it normally — milestone 1 must settle this.
- **Atomics** (`atomicAdd`) — NOT verified. Do **not** rely on them for scoring. Use the non-atomic tiled reduction in `TECHNOLOGY.md` (each thread owns its output slot; CPU finishes the sum from a tiny buffer).

## Hard-won gotchas (these cost real time before)

- **`renderer.render()` only *submits* GL/GPU commands — it does not wait for the GPU.** Timing the call gives a meaningless ~0.3 ms. To measure real GPU cost, use the timestamp API or `gl.finish()` (WebGL). **FPS is vsync-capped** — report **GPU ms**, not FPS, for perf.
- **A backgrounded/inactive browser tab pauses `setAnimationLoop`/rAF AND makes GPU timing unreliable.** When you verify, the tab must be focused. Screenshots of a background tab can be stale.
- **Ping-pong with TSL nodes:** a compute node bakes in the specific storage nodes at creation. Create both directions (A→B and B→A) and run an **even** number of sim steps per frame so the result always lands back in buffer A — then display + brush always target A. (Details in `TECHNOLOGY.md`.)
- **Don't allocate per frame** (no `new` in the animate loop) — reuse scratch objects/uniforms.
- Reaction-diffusion is **numerically twitchy**: wrong feed/kill/dt/Laplacian weights give a dead field (all uniform) or a blown-up field (NaN). Use the exact constants in `TECHNOLOGY.md` first, get a known pattern, *then* expose knobs.

## Definition of done for the first playable

A focused browser tab shows a living reaction-diffusion field running entirely on the GPU; left-drag seeds your faction's pattern (it spreads); a second faction's pattern spreads and contests boundaries; the HUD shows a territory % per faction (from the GPU reduction) and GPU ms; it holds the refresh rate at 1080p. Everything else in `GAMEPLAY.md` is built on top of that.
