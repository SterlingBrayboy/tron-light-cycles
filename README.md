# TRON · Light Cycles

A browser-based Tron Light Cycles game — zero dependencies, zero build step, single HTML file.

## Play

Just open `index.html` in any modern browser. No server required.

## Gameplay

Steer your light cycle around the grid, leaving a trail of light behind you. Force your opponent to crash into a wall, a trail, or themselves. First to 5 round wins takes the match. Speed increases every round.

### Controls

| Action | Cyan (P1) | Orange (P2 / CPU) |
|---|---|---|
| Move | `W A S D` | `↑ ← ↓ →` |
| Pause | `P` | — |
| Mute | `M` | — |
| Next round | `Space` | — |

### Modes

- **Play vs CPU** — single player against an AI opponent that uses flood-fill pathfinding to avoid traps
- **2 Players** — local multiplayer on the same keyboard

## Features

- **Procedural audio** — fully synthesized engine hum, turn clicks, crash effects, countdown beeps, fanfare, and a looping background music track. No audio files — everything is generated with the Web Audio API
- **Engine hum pitch-tracks game speed** — the hum rises in frequency as rounds get faster
- **Fixed-timestep simulation** — game logic runs at a consistent rate regardless of frame rate
- **Simultaneous collision resolution** — head-on crashes are detected correctly; neither rider gets an unfair advantage
- **Particle effects** — crash explosions with glowing particles

## Architecture

Everything lives in a single `<script>` tag inside `index.html` as one IIFE. Two main subsystems:

- **`Audio` module** — a revealing-module-pattern Web Audio engine with a lookahead music scheduler (~120 ms ahead of the rAF clock) and a lazy `AudioContext` that is only created on the first user gesture (satisfying browser autoplay policy)
- **Game core** — a `Uint8Array` grid (`120 × 90` cells at 6 px each) with a fixed-timestep loop, state machine (`menu → countdown → playing → roundover → paused`), and a flood-fill AI

## License

MIT
