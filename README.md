# Bloom

**A pattern-warfare RTS played on a living chemistry field.**

You and rival factions are gardeners on a giant **reaction-diffusion** canvas — a grid of millions of cells, each holding two reacting chemicals. You spend *nutrient* to seed self-spreading living patterns (spirals, dividing blobs, crawling fronts). They grow, eat the substrate, claim territory, and fight at their boundaries. You never place a single cell — you steer the **chemistry**: where to seed, where to drop catalysts (accelerate) or inhibitors (kill). Last bloom standing — or the most territory when the timer ends — wins.

It is, deliberately, a game that **cannot exist without GPU compute**: the entire world is simulated and scored on the graphics card, with the CPU only injecting sparse brush strokes and reading a tiny score back a few times a second.

## Why this design

This is built on a single principle (see `TECHNOLOGY.md` and `CLAUDE.md`):

> The massively-parallel GPU simulation **is** the gameplay. Player input is a few sparse writes per frame; the only CPU readbacks are infrequent (a score), computed on the GPU as a reduction.

That keeps the whole loop on the GPU and sidesteps the one thing the GPU is bad at — reading individual values back to the CPU for per-frame game logic.

## Stack (no build step)

- **three.js r0.184.0**, `WebGPURenderer` + **TSL** (Three Shading Language), loaded via an ES-module import map from a CDN. **No bundler, no build.** Serve the folder statically.
- WebGPU backend (compute shaders, storage buffers). Optional WebGL2 fallback for *display only* — the compute sim is WebGPU-only.

## Run

```bash
# any static server; WebGPU + ES modules need http(s), not file://
python3 -m http.server 8080
# open http://localhost:8080/  in Chrome/Edge (WebGPU on by default) or Firefox/Safari 18+
```

## Deploy

Static — push to a repo and enable **GitHub Pages** (branch `main`, root). The CDN import map works over HTTPS.

## Files

```
index.html            self-contained app (import map + HUD + module)
README.md             this file
GAMEPLAY.md           the game design (v1 — tailor later)
TECHNOLOGY.md         the GPU pipeline, data layout, RD math, scoring — the build reference
PROCESS.md            milestone build plan with a verify-gate at every stage
CLAUDE.md             critical rules / gotchas for the building agent (read first)
```

## Status

Spec only. Build per `PROCESS.md`, verifying each milestone in the browser before moving on.

## Provenance

The technical details here are **carried over from a sibling project** (`terraine_demo`) where the WebGPU + TSL + compute-cull + storage-buffer + readback + GPU-timing patterns were all built and verified on real WebGPU hardware in Chrome. Anything marked **VERIFIED** in `TECHNOLOGY.md` was proven there; anything marked **VERIFY FIRST** is a standard pattern that must be confirmed in milestone 1.
