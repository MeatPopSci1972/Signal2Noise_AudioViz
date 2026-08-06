# HANDOFF — Signal2Noise AudioViz

> **First action for any fresh session (human or AI): read this file top to bottom.**
> This is a static GitHub Pages repo — no dev server, so there is no `GET /HANDOFF.md`
> or `GET /status`; this file *is* the status endpoint.

---

## 1. Current state

- **Current version:** `signal2noise-audioviz.html` (repo root) — v9 step 9
  - step 1 renderer extraction: ✅ committed, checkpoint passed
  - step 2 spiral view: ✅ checkpoint passed (feedback engine, bass→rotation, treble/mic→zoom)
  - step 3 tool tray: gate passed, ⬜ browser checkpoint pending
  - step 4 channel strip + per-channel intensity: ✅ checkpoint passed
  - step 4.1 fire ambient-gating fix: gate passed, ⬜ browser checkpoint pending.
    v8 latent behavior: ambient spectrum→heat injected unconditionally, so the
    fire pill looked permanently on. Now ambient heat only lands inside
    fire-enabled channels' x-regions (chanVfx added to shared read-only inputs);
    mic heat requires ≥1 fire-enabled channel. Deliberate v8-behavior change.
  - step 5 floating tools: superseded by step 6 same-session (user wanted
    docked, not floating). Kept: mesh-gradient grip, opacity slider,
    FH 220→440, ✕ / `tools` button / `h`.
  - step 6 docked tray: gate passed, ⬜ browser checkpoint pending.
    Bottom-docked drawer, full-bleed app (viz fills viewport behind it).
    Grip drag = ns-resize (clamped grip-height…92vh); drag-collapse preserves
    the restore height (bug caught by gate: trayH was being clobbered);
    dblclick grip = quick collapse/expand; `auto` button = collapse on
    pointerleave, expand on pointerenter. Canvas now CSS-stretched to fill
    (non-uniform scale — Cthugha-appropriate; revisit if it bothers).
    Perf watch stands: fire automaton at 2× cells.
- **Frozen baseline:** `v8/signal2noise_v8.html` — do not modify; it exists so `git diff` tells the refactor story
- **Behavior contract for step 1:** visually and audibly identical to v8
- **Validation done:** headless Node.js simulation (stubbed DOM/Canvas/WebAudio) — load+init, dispose→init lifecycle, dispatch→bus→onTrigger→tick full frame, all 6 VFX paths incl. balloon-pop over 120 frames, and exactly **1** trigger listener on the bus after repeated view switches (no leak)
- **Checkpoint status:** step 3 (tray) pending; steps 1–2 passed in browser

### Known issues

- **Mic "arm mic" permission error** (observed after v9 step 1 checkpoint). Sampler
  code is a verbatim move from v8, so this is almost certainly environmental —
  browser permission previously denied for the origin, or OS mic privacy settings.
  Not a current priority. Revisit before any sampler work.


### Steps 6-8 (this session)

- **step 6 docked tray** — bottom drawer; grip drag = ns-resize; dblclick collapse/expand; `auto` = collapse on pointerleave. Render height FH 220 to 440.
- **step 7 gravity view (SUPERSEDED — see step 9)** — was Barnes-Hut n-body over 700 particles + 4 merging cores. That renderer NO LONGER EXISTS; it was replaced wholesale in step 9. Kept here only because the debugging lessons below outlived the code.
  MEASURED: 1.0% mean / 4.8% worst force error vs direct O(N^2); 3.0 ms/frame vs 9.5 ms direct; 5.6x headroom at 60fps.
  FMM DECLINED: its own Table 1 shows direct and adaptive FMM tie at N=100 and FMM needs N~1600 for 10x. accelAt() is the documented swap seam.
- **step 7.1/7.2 containment (HISTORICAL — code removed in step 9)** — the merged remnant drifted off screen. Causes: (a) CORE TUNNELING — an interaction check at a fixed radius is defeated by a body that travels further than that radius in one frame; (b) FRAME CHASING A RUNAWAY — a plain centroid anchor lets one flung body drag the view and push everything else off the far edge; (c) NO OUTER BOUND. The lessons generalise; the code does not exist any more.
- **step 7.3 mass sink + buffer overrun (HISTORICAL — code removed in step 9)** — mass was monotonic with no sink, so the view collapsed to one screen-filling sun; a ceiling only stalls that, a decay/shed path cures it. *** THE LOAD-BEARING LESSON: the visible symptom was drift, the actual fault was NaN. *** Packed body arrays were sized by the STARTING core count, not the maximum; fragmentation pushed the index past the end; Float32Array DISCARDS out-of-range writes SILENTLY, so slots read back undefined, quadtree bounds went NaN, and every body corrupted one frame later with nothing thrown. The onscreen check read NaN as offscreen, so two rounds of parameter tuning chased a bug no parameter could reach. This lesson is why the checklist below exists.
  *** THE REAL BUG: NaN, not drift. *** Packed body arrays were sized NP+NCORE. NCORE is the STARTING core count, not the max, so fragmentation pushed bn past the end. Float32Array DISCARDS out-of-range writes SILENTLY; slots read back undefined, quadtree bounds went NaN, every body corrupted one frame later with nothing thrown. The onscreen check read NaN as "offscreen", which is why two rounds of halo tuning did nothing. Fixed with MAXCORE=8 bounding every core-indexed allocation.
  MEASURED (idle/moderate/hammered/brutal, 6k-12k frames): NaN frames 0 in all four; cores onscreen 100% in all four; equilibrium mass 3192/3581/8099/12148, bounded.
- **step 8 bounded viz** — opacity slider REMOVED. Tray is a flex sibling occupying real layout space, so the viz renders strictly ABOVE the beat area and tray height / viz height trade off directly. VIZ_MIN=140 floor.
- **LYRIC VIEW** — built in a PARALLEL SESSION. Scrolling kinetic typography, per-letter bass envelope with fast attack / slow release, focus line at 40% width, travelling wave, hue cycling. Text from the `lyricIn` transport input via the app-level `lyricText` global.


### Step 9 (parallel session, commits 757d343 and bec85ec)

- **panel fills the window width** (`757d343`). Introduced `:root` custom properties as the
  hook for the rest of the panel reorganisation: `--step-h`, `--step-gap`, `--name-w`,
  `--name-fs`, `--pad-x`, all `clamp()`-based so the grid scales with viewport width.
- **gravity rewritten: two bodies and a moon per channel** (`bec85ec`). Barnes-Hut, the
  700-particle cloud, core merging, fragmentation, mass decay, the halo and the COM anchoring
  are ALL DELETED. The new model: a primary at centre, a secondary on a fixed circular orbit,
  and at most one moon per channel orbiting the SECONDARY. A channel's first trigger spawns its
  moon; every moon's orbit decays continuously (`MOON_DECAY`) and every trigger on that
  channel shoves it back out (`MOON_KICK`). Orbits are advanced PARAMETRICALLY, so the
  secondary's path is exactly circular and cannot drift — the entire class of containment bugs
  from 7.1/7.2 is designed out rather than defended against. Kepler's third law sets angular
  rate (w proportional to r^-1.5), which is what makes an infalling moon visibly wind up.
  WHY: N is two plus at most sixteen. Even direct evaluation would have been free, so a tree was
  paying for a scale that never arrived.
  EMERGENT PROPERTY worth preserving: nothing tracks is the music playing. A busy channel holds
  its moon out against the decay, a sparse one spirals in and strikes, and silence returns the
  system to two bodies — that reading falls out of decay versus trigger rate on its own. Do not
  replace it with a state flag.
  VERIFIED by review (headless, against the code not the comments): lone moon falls in 359 frames
  = 6.0s, matching the stated ~6s; 16 seeded moons all gone 6.4s after triggers stop and sparks
  clear; busy channel (every 8 frames) holds its moon at 0.93 rOut while a sparse one (every 4s)
  sits at 0.36 rOut. Benign: MOON_KICK*(0.5+vol*viz) reaches 1.19 at max intensity so a kick can
  overshoot rOut, but measured peak is 1.002 because decay claws it back each frame.
  Confirmed no dead references left from the removal (accelAt, buildTree, MAXCORE, haloPull,
  recenterCOM, qBodies, fragment all zero); all four views cycle, one bus listener.
### MULTI-SESSION MERGE HAZARD (near-miss, 2026-08-05)

The lyric view was built in one thread while gravity was built in another. The second
session generated a patch against its own copy, whose hash matched the repo exactly --
because the UPLOAD matched, not because the sessions agreed. Applying it would have
silently DELETED the lyric renderer, its transport input, and its state global.
RULE: when more than one session touches this single-file app, ALWAYS diff against the
live repo file and read the diff for DELETIONS before writing. A matching hash proves
your base is current; it does NOT prove your changes are additive. Merge onto the repo
file rather than replacing it. Verify with: git diff -U0 | findstr "^-"

### Particle-system checklist (scar tissue from the gravity view)

Check any view where things MOVE, COMBINE or ACCUMULATE -- Sand and Organic especially.

- **Speed vs radius.** Cap speed strictly BELOW the smallest interaction radius or fast
  bodies tunnel through the check. Make it impossible by construction, not by tuning.
- **Scale capture radius** with whatever makes a body dominant, or big bodies whip past
  instead of absorbing.
- **Sources need sinks.** Any quantity with a source and no sink grows monotonically and
  ends the show. A ceiling only STALLS it; add decay/shed/evaporate. Make the sink visible.
- **Anchor on the LUMINOUS mass**, not the statistical mean; weight superlinearly so one
  runaway cannot hijack the frame.
- **Shape containment like the CANVAS.** A circle on a 640x440 pane fits the SHORT axis
  and leaves the long axis unguarded. Make the restoring force superlinear.
- **Size buffers by the MAXIMUM count, never the INITIAL count.** A later feature WILL
  exceed the starting number, and typed arrays discard overflow SILENTLY.
- **Gate the OBSERVABLE that was reported**, not a statistic that correlates with it.
  Scan for non-finite values every frame -- a corrupted sim and a wandering sim look
  IDENTICAL through a boolean position check.
- **When a fix has NO EFFECT, stop tuning and trace the state.**

REUSE NOTE (updated at step 9): there is NO force-evaluation code left in this repo — step 9 replaced Barnes-Hut with parametric orbits. If a future view ever needs real mutual forces, the thresholds still hold: direct summation below roughly N~150, a tree only above it. Sand remains a CONTACT problem (short-range neighbours), not long-range 1/r^2 — a uniform grid or heightmap beats a tree there. Organic's blobs (~16) want direct summation. Do NOT reintroduce a tree without an N that justifies it.
more than direct summation. Alpha's creatures (16), Spiral's comets (~120) and Organic's
blobs (~16) should all use direct summation. Sand is a CONTACT problem (short-range
neighbours), not long-range 1/r^2 -- a uniform grid or heightmap beats a tree there. Do
NOT extract a shared physics module until a second view genuinely needs long-range
forces; the MERGE MECHANICS (capture radius, mass-conserving absorption, cooldown,
respawn) are the part that will actually transfer, and Organic is the likely first caller.
## 2. File map

```
README.md                        project front door; history table; roadmaps
HANDOFF.md                       this file — state, contract, workflow, backlog
signal2noise-audioviz.html       the app — current working file (single-file, zero-dependency)
v8/signal2noise_v8.html          frozen pre-refactor baseline
architecture/architecture_review.html   v8 rubric: GoF / SOLID / 22 missing tests
```
Branch: `main` (master deleted). Live demo is served by GitHub Pages from main
at /signal2noise-audioviz.html — there is no index.html, so the bare Pages URL
404s by design; README links the full path.

Everything ships as single-file HTML. No build step. proto2prod rules apply:
validate before optimize, surgical edits, visible errors over graceful degradation.

## 3. Architecture contract (v9)

### Renderer contract — every view implements exactly this

```
init(env)       env = {ctx, vx, FW, FH}. Build ALL internal state fresh.
onTrigger(e)    e = {ci, cx, xFrac, width, intensity, vol, col, vfxSet, viz, t}
                One sequencer hit, timing already compensated. React or ignore.
                e.viz = per-channel effective intensity: vizIntensity if the
                channel participates (chanIntensity[ci]), else 1.0. Use e.viz
                for trigger-born visuals; global vizIntensity stays for
                ambient/whole-frame effects (Alpha's spectrum heat, Spiral's
                spectrum arm, creature render scale).
tick(frame)     frame = {seqFreq, micLevel, t}. Draw exactly one frame.
dispose()       Release everything. Manager may init a different renderer
                on the same canvases immediately after.
```

### Load-bearing boundaries

- **Renderers NEVER touch WebAudio nodes.** The app loop reads analysers once
  per frame and passes data in via `frame`. This is the wall all views live behind.
- **The scheduler knows no VFX by name.** It calls `dispatchTrigger(ci, delay)`,
  which bumps the vizBar and emits `trigger` on the EventBus. `trigViz()` is
  deleted; do not reintroduce direct scheduler→visual calls.
- **App-level (shared by all views):** transport, presets, grid, vizBar,
  sampler/mic, `vizIntensity`, `CH_HEAT`, `CHANNELS` (read-only to renderers).
- **View-level (private to a renderer's closure):** every pixel buffer,
  particle pool, and animation state. Alpha holds the fire field, VFX pool,
  balloons, creatures, mountain.
- **`vfxSet` in the trigger payload is Alpha-specific** (fire/strobe/balloon/
  sparkle/firework/creature pills). Future views may ignore it. If a second
  view ever needs pills, generalize then — not before.
- **Adding a view = one factory + one `VIEWS` entry.** Nothing else changes.
- **Lifecycle self-test:** clicking the ACTIVE view tab re-runs dispose→init.
  Use it after any renderer change.

### Decisions on record

- FM synthesis will live in a **separate synth panel** (new tonal channels
  alongside drums), not per-drum-channel dials. Requires synth-side Strategy first.
- vizBar stays app-level so every view keeps the per-channel meters.

## 4. Workflow / gate

1. Make the change in `v9/signal2noise_v9.html` (or a new `vN/` when a version freezes).
2. **Gate:** syntax check + headless Node simulation of the touched seams
   (lifecycle, bus wiring, full-frame tick). No browser-only "looks fine" ships.
3. Human checkpoint in the browser — side-by-side with the previous state
   when the contract is "behavior identical."
4. Commit with a message referencing the architecture-review finding it closes,
   e.g. `refactor: Strategy + EventBus — renderer extraction, view tab bar (v9 step 1)`.
5. Never check in broken code — a broken baseline makes the diff story unreadable.
6. When a version is complete: update the README history table and this file's
   §1/§5, then tag and release: `git tag <tag> && git push --tags`, GitHub →
   Releases → attach `signal2noise-audioviz.html` → publish. README links
   /releases/latest, so no README edit is needed per release.

## 5. Backlog (ordered)

1. ✅ Checkpoint v9 step 1 — passed (boom matched side by side)
2. ✅ Commit step 1; `index.html` redirect → v9 — done
3. ✅ **Spiral view** — shipped + checkpoint passed
4. ✅ Tool tray (step 3) — superseded by step 5's floating tray
5. ✅ **Channel strip** (step 4) — checkpoint passed. Click channel NAMES to
   select; tri-state pills + volume + ⚡int apply to selection; excluded
   channels dim and pin at e.viz=1.0.
5b. ⬜ **Step 4.1 + 5 checkpoint** — fire pill actually kills fire per-channel;
   tray drags/opacity/toggles; viz at 2× height holds framerate.
6. ⬜ Sand view — particle deposition, per-channel pour points, heightmap + angle of repose
7. ⬜ Organic view — metaballs via radial gradients + threshold pass
8. ⬜ Galactic view — softened n-body, triggers add mass/velocity
9. ⬜ Synth-side Strategy — `synth()` switch → voice map (arch review finding)
10. ⬜ FM voices + synth panel — 2-op FM (mod → gain(index) → carrier.freq),
    dials: ratio / index / decay
11. ⬜ Write the 22 missing tests (arch review) on the Node-simulation scaffolding
12. ⬜ Command pattern — preset actions + undo stack
13. ⬜ State machines — sequencer + sampler

### 5a. Channel strip — decision record (resolved)

Selection is the editing TOOL (transient); it sets two INDEPENDENT per-channel
flags, which may overlap freely:
- chanVfx[ci] — which elements a hit adds to the visualization (Alpha pills)
- chanIntensity[ci] — whether the channel rides the global intensity slider
Exception gets the marker: intensity-excluded names render dimmed; the
all-on default stays visually quiet. No arm-and-fire mechanics (proto2prod).

## 6. Development discipline (distilled from the v1–v9 history)

The full prompt-by-prompt narrative lived in TEACHING_NOTES.md (see git history,
pre-v9-step-1). The load-bearing lessons:

- **Validate before you ship.** Simulate the logic in Node.js with synthetic
  data before trusting the UI. This caught the ZCR denominator bug, the hi-hat
  double-assign, dropped consecutive onsets (v5); five scheduler/state bugs
  from one structured report (v6); the `b+=3` analyser stride bug (v7); and
  three pre-baseline bugs including `Object.freeze` on a Float32Array (v8).
- **Expected / Actual / Wanted.** Structured bug reports find root causes in
  batches, not one at a time.
- **Debug the data, not the renderer.** The visualizer is a symptom; the
  spectrum is the disease.
- **Checkpoint before refactor.** The architecture review is the rubric;
  a refactor that doesn't close identified gaps is just a rewrite with new
  unknown gaps.
- **Commit before refactor.** The before-state gives the after-state meaning.
  `git diff v8-baseline..v9-step1` is the teaching material.
- **IP due diligence before publishing.** Renamed from the Cthugha lineage
  before the first public commit — no history to explain.
- **Questions about correctness come before questions about completeness.**
  Build, validate, audit, document, commit.

---

*Update §1 and §5 at the end of every working session. If §1 disagrees with
the repo, the repo wins — then fix this file.*
