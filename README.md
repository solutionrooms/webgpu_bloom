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

## Play

You are **Coral** (green). Two AI factions — **Bloom** (magenta) and **Spore** (amber) — contest the same field.

- **Left-drag** on the field to seed your living pattern; it spreads and claims territory. Seeding costs **nutrient** (regenerates faster the more territory you hold).
- **Brush S / M / L** — bigger brush, bigger stamp, more nutrient.
- **Win** by holding the most territory when the timer ends (or ≥55% at any point).
- **R** — restart the round.

Tip: seed into open neutral ground ahead of the fronts — that's how you out-claim the AI.

## Run

```bash
# any static server; WebGPU + ES modules need http(s), not file://
python3 -m http.server 8080
# open http://localhost:8080/  in Chrome/Edge (WebGPU on by default) or Firefox/Safari 18+
```

Grid size is a URL knob: `?n=256` / `512` (default) / `1024` / `2048`.

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

**Playable.** Built and verified milestone-by-milestone in Chrome (M0–M7 in `PROCESS.md`): WebGPU boot → field buffer + fragment-stage storage read → Gray-Scott reaction-diffusion compute → brush seeding → factions + ownership/combat → GPU territory reduction → nutrient economy + AI + win/restart → aspect-correct display, GPU-ms telemetry, grid knob, contested-front shimmer. The entire world is simulated and scored on the GPU; the CPU only injects brush strokes and reads a tiny score buffer a few times a second.

Recipes are the meta — tune the per-faction feed/kill in `FACTIONS` (and the `[knob]` constants) in `index.html`.

## Provenance

The technical details here are **carried over from a sibling project** (`terraine_demo`) where the WebGPU + TSL + compute-cull + storage-buffer + readback + GPU-timing patterns were all built and verified on real WebGPU hardware in Chrome. Anything marked **VERIFIED** in `TECHNOLOGY.md` was proven there; anything marked **VERIFY FIRST** is a standard pattern that must be confirmed in milestone 1.
