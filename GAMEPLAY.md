# Bloom — Gameplay (v1)

A complete, buildable first design. The owner will tailor it; everything marked **[knob]** is meant to be tuned, and **[later]** is deliberately deferred. Build the **Core loop** first.

## Fantasy

You are a gardener of living chemistry. Your weapon is a self-spreading pattern — a "bloom" — grown from reagents on a shared field. You don't move units or place tiles; you decide **where to seed**, **where to feed**, and **where to poison**, and the chemistry does the rest. Two or more blooms on the same field inevitably collide, and their fronts fight for substrate. It's part RTS, part Go, part Petri-dish.

## The field

- A toroidal (wrap-around) grid of cells (start **1024×1024** [knob]). Every cell is substrate (food) hosting two reacting chemicals; living pattern (chemical B) consumes substrate as it grows.
- Substrate is finite-feeling: dense patterns starve their own interiors, so blooms naturally form fronts, rings, and branches rather than solid fills. This is what makes territory contestable instead of a paint-bucket race.

## Factions ("species")

- 2–4 factions [knob]. Each is a **Gray-Scott recipe** (a feed/kill pair) + a colour. Recipes give genuinely different behaviour:
  - **Mitosis** — dividing blobs that fill space fast; aggressive expansion, fragile fronts.
  - **Coral** — branching maze-fronts; slower but holds ground and walls off area.
  - **Spots** — stable dots; defensive, hard to dislodge, poor at expanding.
- Soft rock-paper-scissors emerges from the chemistry at the boundaries (mass/front-shape decides who pushes whom) — **emergent, not scripted**.

## Core loop (build this first)

1. The field starts mostly neutral substrate with a few seed spots per faction.
2. **Seed (left-drag):** paint your reagent into the field — within the brush you set B=1 and owner=you. Your pattern spreads from there according to your recipe. Costs **nutrient** per area stamped.
3. Patterns grow on the GPU every frame; **fronts collide**; the cell-by-cell combat rule (higher local B wins the cell) decides the moving border.
4. **Nutrient** regenerates over time, faster the more **territory** you own (territory % comes from the GPU score reduction). Snowbally — but a well-placed front or a poison can break a leader.
5. **Win:** own ≥ **60%** [knob] of the field, **or** hold the most territory when a **5-minute** [knob] timer ends.

That's a complete game: seed → spread → contest → snowball/upset → win. Everything below is layered on top.

## Actions & economy (v1)

| Action | Effect | Cost |
|---|---|---|
| **Seed** (left-drag) | Stamp your living pattern; it spreads | nutrient ∝ area |
| **Catalyst** [later] | Local feed boost — your nearby pattern surges | more nutrient |
| **Inhibitor** [later] | Local kill boost — dissolves pattern (incl. enemy) | most nutrient |
| **Brush size** | 1× / 4× / 16× cell radius | — |

- **Nutrient:** a single resource. Regen rate = `base + perTerritory × territory%`. A small bank, so you choose *where* to commit, not *whether*. [knob: regen curve]
- Seeding into **enemy** territory is wasteful (their pattern resists); seeding the **substrate just ahead of a front** is how you outflank — this is the skill.

## Opponents

- **v1:** a simple scripted AI per rival faction — pick the largest neutral region near its territory and seed there on a timer; occasionally reinforce a losing front. (No GPU readback needed for this if it seeds blindly toward map regions; if it needs to "see" territory, reuse the per-tile score buffer the scoring pass already produces — it's a coarse map, cheap.)
- **[later]:** hot-seat / online; smarter AI that reads the coarse territory map and targets weak fronts.

## Controls

- **Left-drag** — seed with the current tool/faction.
- **Faction/tool buttons** (HUD) — pick which recipe you're seeding; later: catalyst/inhibitor.
- **Brush size** buttons — 1 / 4 / 16.
- **Scroll** — zoom the view into the field [optional]; pan with drag-middle or arrows [optional].
- **R** — restart round.

## HUD

- **Territory bar** per faction (from the GPU reduction) — the scoreboard and the tension.
- Nutrient meter, current faction/tool, brush size, round timer.
- **GPU ms** + grid size + ITER (dev/perf; can hide for players).

## Feel / juice (cheap wins)

- Faction-coloured glow for living B; dark substrate; bright shimmering **contested fronts** (cells whose owner flipped recently — use the `aux` channel as a fade timer).
- A soft bloom/blur post pass makes the chemistry look alive [later].
- A pulse when you seed; a chord when a front breaks through. [later]

## Progression / modes (pick later)

- **Skirmish** (the core), **Survival** (waves of AI blooms), **Puzzle** (reach a target shape with limited nutrient), **Sandbox** (free recipes, no scoring) — Sandbox is also the best dev harness.

## Explicit tailoring knobs (for the owner)

Grid size; number of factions; the feed/kill recipes (this *is* the meta); claim/decay thresholds; combat rule (mass-wins vs recipe-counter table); nutrient regen curve; brush sizes/costs; win % and timer; ITER (pattern tempo). Start from `TECHNOLOGY.md`'s constants — they give known-good patterns — then tune.

## Out of scope for v1

Catalyst/inhibitor tools, online multiplayer, smart AI, audio, post-processing, save/load. Get **seed → spread → contest → score → win** solid first.
