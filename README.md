# Fishes Schooling — Interactive Vicsek Simulation

A browser-based flocking simulation built as a single-file HTML/Canvas port of the
`FishesSchooling` sketch from [awmartin/spatialpixel](https://github.com/awmartin/spatialpixel),
extended with the Vicsek statistical-mechanics model so the toy doubles as a live
phase-transition demo.

**Live demo:** https://spatial-pixel-fish.pages.dev/

**Run locally:** open `index.html` in any modern browser. No build, no deps.

---

## Origin

`spatialpixel` is a computational-design toolkit written for Processing 3 in Python Mode.
Processing 4 dropped Python Mode support, so the original `.pyde` sketches no longer run
out of the box. Rather than install legacy Processing 3 + Python Mode, this version
re-implements the algorithm in vanilla JS so it runs instantly in a browser, with
extensions (Vicsek noise, order parameter, phase-transition sweep, underwater aesthetic)
that weren't in the original.

The port preserves the original algorithm in `behaviors.py` line-for-line; the rest is new.

---

## What it is

A self-contained schooling simulation with:

| Feature | What it does |
|---|---|
| **Boids-style steering** | Cohesion, separation, alignment — Reynolds (1987) flocking |
| **Vicsek noise η** | Random heading perturbation per step — drives phase transition |
| **Order parameter φ** | Live measurement of how aligned the school is, 0 (disorder) → 1 (perfect order) |
| **Phase-transition sweep** | Automated η sweep that plots φ(η) — visualizes the order-disorder transition |
| **Underwater aesthetic** | Gradient depth, drifting caustics, rising bubbles |
| **Motion trails** | Alpha-fade compositing for ghostly streaks |
| **Flex bodies** | Fish bezier curves bend opposite to their turn direction |

---

## Architecture

Three stacked HTML5 canvases + an order-parameter overlay:

```
┌─────────────────────────────────────────────────┐
│  #order  (320×180, top-right)                   │  ← sparkline / phase plot
├─────────────────────────────────────────────────┤
│                                                 │
│  #fx     (full viewport, top layer)             │  ← bubbles
│  #stage  (full viewport, mid layer)             │  ← fish + motion trails
│  #bg     (full viewport, bottom layer)          │  ← gradient + caustics
│                                                 │
└─────────────────────────────────────────────────┘
```

Each frame:
1. Repaint `#bg` (gradient + caustics, fully redrawn)
2. Alpha-fade `#stage` with `rgba(2,16,25, P.trailFade)` instead of clear → motion trails
3. Run `step()` — update all fish via O(n²) all-pairs interaction
4. Draw fish bezier bodies to `#stage`
5. Clear `#fx`, spawn/move/draw bubbles
6. Every 2 frames: compute φ, shift sparkline buffer, redraw `#order`

---

## Physics — Vicsek + boids hybrid

The simulation is a hybrid of two well-known models.

### Boids (Reynolds 1987)

Each fish *i* applies three local steering forces based on its neighbors:

- **Cohesion** — steer toward the centroid of neighbors within radius `cohesion`
- **Separation** — turn away from the single closest fish if within radius `separation`
- **Alignment** — match the average heading of neighbors within radius `alignR`

These are weighted (`cohesionWeight`, `separationWeight`, `alignmentWeight`) and summed
into a `turnrate` and `speed` delta. Turn rate is clamped to `turnRateLimit` (π/18 rad/step);
speed is clamped to `speedLimit`.

### Vicsek (1995)

Vicsek's minimal model: every agent has constant speed; the only update is heading.
A neighbor's mean direction sets your new heading, plus uniform random noise:

```
θᵢ(t+1) = ⟨θⱼ(t)⟩_{j ∈ N(i)} + Δθ,    Δθ ~ U(-η/2, η/2)
```

In this sim, the noise term `Δθ` is added directly to the boids turn rate, producing
a continuum: at **η = 0** the dynamics are pure boids; as **η → 2π** the alignment
signal is washed out and the school becomes a random swarm.

### Order parameter

The flocking order parameter is the magnitude of the mean velocity unit vector:

```
φ = | (1/N) Σᵢ v̂ᵢ |  ∈  [0, 1]
```

- **φ ≈ 1.0** — all fish point the same direction (ordered phase)
- **φ ≈ 0.0** — headings are uniformly distributed (disordered phase)

The original Vicsek paper (1995) showed this transition is *continuous* (second-order)
in 2D — φ rises smoothly from 0 as you decrease η below a critical value η_c that
depends on density and speed. You can reproduce this curve directly via the **Sweep**
button.

---

## Parameters

All sliders update live; no restart needed (except `Fish` count, which respawns).

| Slider | Range | What it does | Physics intuition |
|---|---|---|---|
| **Fish** | 20–600 | Number of agents | Higher density → lower critical noise η_c |
| **Alignment R** | 10–120 | Neighborhood radius for alignment | Larger R → easier to find a coherent heading → more order |
| **Cohesion** | 0–80 | Neighborhood radius for centroid attraction | Holds the school spatially compact |
| **Separation** | 0–40 | Personal-space radius | Prevents fish overlap; too high → school fragments |
| **Noise η** | 0–2π | Random heading perturbation per step | The phase-transition control parameter |
| **Speed** | 1.0–6.0 | Speed cap (px/frame) | Faster fish → less time to align → more disorder at same η |
| **Trail fade** | 0.03–1.0 | Alpha of the per-frame overlay | Lower = longer trails; 1.0 = no trails (full clear) |

Hidden internal constants (in `P` object): `cohesionWeight=20`, `separationWeight=18`,
`alignmentWeight=6`, `cohesionThreshold=25`, `turnRateLimit=π/18`.

---

## How to use it

### Reproduce the phase transition

1. **Reset** to randomize headings
2. Set **Noise η** to **0.1** — within ~10 seconds the φ sparkline climbs above 0.9.
   The whole school glides as a single coherent flow.
3. Slowly drag η up to **~3.0** — φ collapses to ~0.2. The school breaks into chaos.
4. Hit **Sweep η → φ curve** — the right panel switches to the full φ(η) curve, showing
   the sigmoid-like transition. The inflection point is the critical noise η_c for this
   density/speed configuration.

### Reproduce the original sketch

Set: Fish=100, Alignment R=50, Cohesion=50, Separation=15, Noise η=0, Speed=3,
Trail fade=1.0. This is the original `FishesSchooling.pyde` configuration.

### Aesthetic mode

Fish=300, Alignment R=70, Noise η=0.4, Trail fade=0.08. Long ghostly trails with
gentle wandering motion.

---

## Code structure (one file, ~330 lines)

```
fishes-schooling.html
├── <canvas> stack            — bg, stage, fx, order
├── <div class="panel">       — sliders, buttons
└── <script>
    ├── Canvas + resize       — fits viewport, multi-canvas refs
    ├── P (parameters)        — single source of truth, mutated by sliders
    ├── class Fish            — x, y, dir, speed, turn, color
    ├── bind(id, key, fmt)    — wires <input> → P[key] → DOM text
    ├── computeOrder()        — φ = |⟨v̂⟩|
    ├── drawOrder()           — sparkline + axes
    ├── step()                — O(n²) interaction loop + Vicsek noise
    ├── drawBackground()      — gradient + animated radial caustics
    ├── drawBubbles()         — particle FX
    ├── drawFish(f)           — bezier body, flexes with turn rate
    ├── frame()               — main RAF loop
    └── sweepBtn.onclick      — async η-sweep, replaces sparkline with φ(η) plot
```

---

## Performance notes

The simulation uses a **uniform-grid spatial hash** to scale to 10,000+ fish:

- **Cell size** = `max(alignR, cohesion, separation)` — guarantees every neighbor within
  the largest interaction radius lives in the current cell or one of its 8 neighbors.
- Each frame: rebuild the grid (~O(n)), then each fish only scans the ~9 cells
  near it (~O(1) per fish on average), giving **O(n) amortized** instead of O(n²).
- Real numbers on a 2024 M-series Mac in Chrome: **10,000 fish at ~50fps**.
  The same machine running the naïve O(n²) version chokes around 600 fish.

**LOD rendering:** above `lodThreshold` (default 1500), fish are drawn as batched
triangles (one fillStyle per color group) instead of two bezier curves + two fin
lines. The bezier path is the visual win for small flocks; the triangle path is
the only way to keep 60fps at 10k.

**FPS counter** in the panel header pill shows live framerate.

For 100k+: WebGL compute shader, fish state as RGBA texture, ping-pong rendering.

---

## Possible extensions

| Direction | What it adds |
|---|---|
| **Spatial hash** | Scale to 10k+ fish |
| **Predator + vision cone** | Click-to-place shark, fish can only see fish in their forward arc — produces panic waves, school splitting |
| **Obstacles** | Static rocks/plants the school must avoid |
| **Food / metabolism** | Fish accumulate hunger, follow food sources |
| **Multiple species** | Different parameter sets, mixed schools, inter-species behavior |
| **GA evolution** | Population of schools, fitness = survival under predation, evolve cohesion/separation/alignment weights over generations |
| **Vicsek lattice plot** | Heatmap of where fish spend time, gives a density field |
| **Save as GIF/MP4** | `gif.js` or MediaRecorder for capturing simulation runs |
| **3D mode** | Three.js port, true 3D flocking |

---

## References

- Reynolds, C. W. (1987). *Flocks, herds and schools: A distributed behavioral model.* SIGGRAPH '87.
- Vicsek, T., Czirók, A., Ben-Jacob, E., Cohen, I., & Shochet, O. (1995). *Novel type of phase transition in a system of self-driven particles.* Phys. Rev. Lett. 75(6), 1226.
- Original Processing sketch: [awmartin/spatialpixel/Sketches/FishesSchooling](https://github.com/awmartin/spatialpixel/tree/master/Sketches/FishesSchooling)

---

## Files

- `index.html` — the simulation (single file, ~330 lines)
- `README.md` — this document

## Deploy

Hosted on Cloudflare Pages. To redeploy after edits:

```bash
wrangler pages deploy . --project-name=spatial-pixel-fish --branch=main --commit-dirty=true
```

## License

MIT — same as the upstream [awmartin/spatialpixel](https://github.com/awmartin/spatialpixel).
