# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file browser game: **Tron Light Cycles**. Everything — HTML, CSS, and all
JavaScript — lives inline in [`index.html`](index.html). There is no build step, no
package manager, no dependencies, and no network/fetch calls. The game runs by opening
`index.html` directly (`file://`) in any modern browser.

## Running & verifying

- **Run it:** open `index.html` in a browser. A static server is *not* required
  (the file is fully self-contained), but `.claude/serve.py` serves the folder on
  port 8131 (`python3 .claude/serve.py`) if you want one. `.claude/launch.json`
  wires the same up for the preview panel.
- **Sandbox note:** the in-harness preview server and `python3 -m http.server`
  fail here with `Operation not permitted` (the launcher's sandbox can't open
  files / call `getcwd`). The Launch preview panel still renders the file directly.
- **No browser? Verify headlessly with JavaScriptCore via `osascript`** — the only
  JS runtime available in this environment (no node/deno/bun). Three useful layers:
  1. **Syntax check:** read the file, split out the `<script>` body, and wrap it in
     `new Function(code)` — throws on syntax errors without executing.
  2. **Init/render check:** stub `window`, `document.getElementById` (return a fake
     element with `addEventListener`/`classList`/`style`/`getContext`), `performance`,
     `requestAnimationFrame`, `setTimeout`/`setInterval`, then `eval(code)`. Confirms
     boot + one `draw()` frame don't throw. `AudioContext` is intentionally left
     undefined so the audio engine no-ops.
  3. **Gameplay check:** in the stub, capture the `keydown` handler and the `btnSolo`
     click handler, make `performance.now()` return a mutable clock you advance, and
     have `requestAnimationFrame` store the callback so you can pump frames manually.
     Driving an uncontrolled CPU match to completion proves collision + round scoring.

## Architecture

All code is inside one IIFE in the `<script>` tag. Nothing is exported to the global
scope — to drive the game from a test harness you must capture the event handlers it
registers (see "Gameplay check" above), not call internal functions.

Two cooperating subsystems:

- **`Audio` module** (top of the script): a fully **procedural Web Audio engine** —
  no audio files. Builds a master → music/sfx gain graph lazily. Key constraint:
  the `AudioContext` is created and `resume()`d only inside `unlock()`, which must be
  called from a **user gesture** (button click / keypress) to satisfy browser autoplay
  policy — every entry point (`begin()`, `keydown`, audio toggle) calls `unlock()`
  first. Sounds: continuous engine `hum` (pitch tracks game speed), one-shot `turn`/
  `crash`/`countdownTick`/`fanfare`, and a **lookahead-scheduled** background music
  loop (`startMusic`/`scheduler` schedule notes ~120ms ahead off the Web Audio clock,
  *not* the rAF loop). Mute flips the master gain.

- **Game core**: a fixed-cell grid (`CELL` px → `COLS`×`ROWS`) backed by a flat
  `Uint8Array occupied` (0 = empty, 1/2 = owner). Riders hold `{x,y,dir,next,trail}`;
  the `next`-vs-`dir` split with the `OPP` map prevents 180° reversals.

### Two clocks — keep them separate

1. A **fixed-timestep simulation** inside `loop()`: an accumulator advances the game
   by discrete `step()` moves at `baseSpeed` moves/sec (which ramps each round),
   decoupled from the variable-rate `requestAnimationFrame` render.
2. The **Web Audio clock** drives music scheduling independently. Don't try to drive
   audio timing from rAF or the sim accumulator.

### Collision resolution (the subtle part)

`step()` resolves all riders **simultaneously**: it computes every rider's next cell,
tallies target cells to detect **head-on** collisions (two riders entering the same
cell), then checks wall / trail hits — and only *after* all deaths are decided does it
commit the survivors' positions into `occupied` and their trails. Editing this needs
care: marking a rider dead and writing its cell in the same pass would corrupt the
head-on and trail checks for the other rider.

### State machine

`state` ∈ `menu | countdown | playing | roundover | paused`, driven from `loop()` and
the input handler. `roundover`→next round and the match-end overlay are deferred via
`setTimeout`, but score mutation (`scores[winner]++`, `updateHUD()`) happens
synchronously inside `endRound()` — relevant when verifying headlessly with a no-op
`setTimeout`.

## Conventions

Keep the single-file, zero-dependency, zero-build constraint intact — it's the point
of the project. Match the existing style: the `Audio` revealing-module pattern, the
`el.*` cache of DOM references, grid coords (`idx()`/`inBounds()`) vs. pixel coords
(multiply by `CELL`), and the `COL`/`DIRS`/`OPP` lookup tables.
