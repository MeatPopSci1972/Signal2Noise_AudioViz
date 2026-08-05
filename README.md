# Signal2Noise AudioViz

A browser-based audio visualizer + drum sequencer, built as a prototype-to-production teaching project.

Originally inspired by Cthugha (1993), a classic DOS fire-and-waveform visualizer by Kevin "Zaph" Burfitt. Cthugha's engine was feedback-based — pixel translation tables and palette rotation.

## Live demo

**[▶ Launch Signal2Noise AudioViz](https://meatpopsci1972.github.io/signal2noise-audioviz/signal2noise-audioviz.html)**

---

## What it does

- **16-channel drum sequencer** — synthesized drums via Web Audio API, swing, 16/32 step modes
- **View system (v9)** — tab bar of visualizer views built on a Renderer strategy contract
- **Alpha view** — the original fire engine: cellular automaton pixels, each channel burns its own frequency column, plus VFX and animorphic creatures
- **Spiral view** — Cthugha-lineage feedback engine: rotate/zoom echo with pixel ghosting, bass drives rotation, treble drives zoom, per-channel comets
- **6 VFX types per channel** — fire, strobe rings, balloons, sparkles, fireworks (googly-eyed), animorphic creatures
- **Channel strip** — click channel names to select; one tri-state control set (VFX pills, volume, ⚡int) applies to the whole selection
- **Per-channel intensity** — ⚡int flags which channels ride the intensity slider; excluded channels stay at baseline
- **Docked tool tray** — full-bleed visualizer with a bottom drawer: drag the mesh-gradient grip to resize, double-click to collapse, `auto` to auto-hide on pointer leave
- **Lyric view** — scrolling kinetic typography: per-letter sizing driven by a bass envelope, a focus line where glyphs swell and a travelling wave runs, text set live from the transport bar
- **Gravity view** — Barnes-Hut n-body: colliding galactic cores with self-gravitating clouds, momentum-conserving mergers, drums inject mass and velocity
- **Mic sampler + preset builder** — record audio, onset detection + spectral centroid classifier auto-builds a drum pattern
- **5 built-in presets** — boom bap, house, jungle, hip-hop, techno + random + clear

---

## Prototype history

| Version | What changed |
|---------|-------------|
| v1 | Basic Cthugha-style fire visualizer, keyboard input |
| v2 | Keyboard drum layout, frequency-mapped fire columns |
| v3 | Per-note VFX: rings, vignettes, beam columns, drum-typed visuals |
| v4 | 16-channel step sequencer, synthesized drums, 5 presets |
| v5 | Mic sampler + onset detection preset builder |
| v6 | **Bug fix:** preset switching — 5 scheduler/state bugs found and fixed via Node.js simulation |
| v7 | **Bug fix:** fire visualization — analyser stride bug, per-channel direct heat injection, spectral centroid classifier |
| v8 | 6 VFX types per channel (pills UI), animorphic creatures, mountain silhouette, fireworks with googly eyes. **Frozen as pre-refactor baseline.** |
| v9 step 1 | **Refactor:** Renderer contract + EventBus. `trigViz()` deleted; all visual state moved into `createAlphaRenderer()`; globals 21 → 12; view tab bar added. Validated headless via Node.js lifecycle simulation. Behavior identical to v8 by contract. |
| v9 step 2 | **Spiral view** — feedback engine (rotate/zoom echo + ghost fade), one factory + one `VIEWS` entry, proving the contract |
| v9 steps 3–4 | **Channel strip** — per-row controls replaced by selection + one tri-state control set; per-channel intensity participation (`e.viz` added to the trigger payload) |
| v9 step 4.1 | **Fix:** ambient fire heat gated to fire-enabled channels' regions — a latent v8 behavior exposed by the new per-channel toggle |
| v9 steps 5–6 | **Docked tool tray** — full-bleed viz (render height 220→440, renderers rescaled via `init(env)` untouched), resizable/auto-hiding bottom drawer with opacity |

---

### v9 refactor progress

| Finding | Status |
|---------|--------|
| Strategy — renderer side | ✅ steps 1–2: Alpha extracted, Spiral added as pure factory — contract proven by a second implementation |
| Observer / EventBus | ✅ step 1: scheduler emits `trigger`; `trigViz()` deleted |
| Strategy — synth side | ⬜ planned (FM synth track) |
| Template Method — unified VFX lifecycle | ◐ partially absorbed by the renderer contract; revisit inside Alpha |
| Command — preset actions + undo | ⬜ planned |
| Facade — injectable interfaces for tests | ◐ frame-object injection exists; formalize with the test pass |
| 22 missing tests | ⬜ Node simulation scaffolding exists; tests not yet written |
| State machines — sequencer + sampler | ⬜ planned |

### View roadmap

| View | Status | Core idea |
|------|--------|-----------|
| alpha | ✅ shipped (v9 step 1) | Cthugha-lineage fire + VFX + creatures (the v8 look) |
| spiral | ✅ shipped (v9 step 2) | Feedback buffer with pixel ghosting — no clear, translucent fade + rotate |
| lyric | ✅ shipped | Scrolling kinetic typography; per-letter bass-driven intensity, focus line, travelling wave |
| gravity | ✅ shipped (v9 step 7) | Barnes-Hut n-body; galactic cores collide and merge, triggers add mass and velocity |
| sand | next | Particle deposition; channels pour colored grains, the pile is the song's history |
| organic | planned | Metaballs; channel blobs grow on triggers and absorb each other |
| galactic | planned | Softened n-body; mass centers per channel group, triggers add mass and velocity |

All views consume the same inputs — per-channel trigger events + a per-frame spectrum — which is exactly what the Renderer contract formalizes.

### Sound roadmap

FM synthesis controls (separate synth panel): two-operator FM voices — modulator → gain (index) → carrier frequency — with ratio / index / decay dials. Requires the synth-side Strategy refactor first.

---

## Running it

Open `signal2noise-audioviz.html` directly in any modern browser.

Click the visualizer first to unlock the AudioContext, then hit **play** or click a preset. Spacebar toggles play; `h` toggles the tool tray.

The tools live in a bottom drawer: drag its gradient grip to resize, double-click it to collapse, `auto` to auto-hide; the viz takes whatever space the tray gives back. Click channel names to select them; the control strip applies to the selection.

For mic input / sampler: click **arm mic** and grant browser microphone permission. (Known issue: a permission error is under investigation — see HANDOFF.md.)

Clicking the active view tab re-runs the renderer's dispose→init cycle — a built-in lifecycle self-test.

---

## Tech stack

- Vanilla JS (no framework)
- Web Audio API — synthesis, scheduling, analysis
- Canvas 2D API — fire pixel buffer + VFX overlay
- MediaRecorder API — mic capture
- No build tools, no bundler, no dependencies
