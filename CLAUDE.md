# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Vanilla JavaScript Tetris (README is in Spanish). No dependencies, no build step, no package.json — just three files:

- `index.html` — DOM structure: `#board` canvas (300×600, 10×20 grid of 30px blocks), `#next-canvas` (120×120) for the next-piece preview, HUD spans (`#score`, `#lines`, `#level`), the energy bar (`#energy-fill`) and ability buttons (`#ability-buttons`), the hidden `#preview-section`/`#preview-canvas` used only while the 5-piece-preview ability is active, and the `#overlay` used for both pause and game-over states.
- `style.css` — dark/retro arcade styling (flexbox layout, backdrop-filter blur on the overlay).
- `game.js` — all game logic (~300 lines), described below.

## Running the game

No install/build required. Either open `index.html` directly in a browser, or serve it statically:

```bash
python3 -m http.server 8000
# or
npx serve .
```

There is no test suite, linter, or bundler configured in this repo.

## Architecture (`game.js`)

Everything is global, procedural state — no classes, no modules. Key mutable state lives in one destructured `let` block: `board, current, next, queue, score, lines, level, energy, slowRemaining, previewRemaining, paused, gameOver, lastTime, dropAccum, dropInterval, animId`.

- **Board model**: `board` is a `ROWS × COLS` matrix; each cell is `0` (empty) or a color index `1–8` identifying which piece locked there.
- **Pieces**: the 7 standard tetrominoes plus two special pieces, the **Nut** (`NUT = 8`) and the **Y** letter piece (type `9`), are defined as square matrices in `PIECES` (index 0 unused/null). `randomPiece()` deep-copies a shape and centers it at spawn, drawing uniformly from `PIECES.length - 1` types (so adding a new piece just means appending to `PIECES`/`COLORS` — no other constant needs updating). The Nut is a 3×3 ring (`[[8,8,8],[8,0,8],[8,8,8]]`) with a real `0` hole in the center — `collide()` treats it as empty (nothing blocks moving into it) and `merge()` leaves it `0` in `board`, so a row containing an unbroken Nut hole can never satisfy `clearLines()`'s `every(v => v !== 0)` check until the ring is broken from above. `draw()`/`drawNext()` render the hole as a stroked circle via `drawNutHole()`; `isNutHole(r, c)` detects a locked-in hole on the board by checking all 8 neighbors are `NUT`. The Y piece (`[[9,0,9],[0,9,0],[0,9,0]]`) has no enclosed hole, so it needs none of that special-cased logic — its `0` cells are just normal empty space.
- **Rotation**: `rotateCW(shape)` transposes + reverses rows. `tryRotate()` applies it then tries wall-kick offsets `[0, -1, 1, -2, 2]` columns, keeping the first that doesn't collide.
- **Collision**: `collide(shape, ox, oy)` checks bounds and overlap against `board`; used for movement, rotation, ghost-piece projection, and spawn (game-over check).
- **Piece queue**: `queue` (length kept at `QUEUE_SIZE` = 5 by `fillQueue()`) replaces the old single-piece lookahead. `spawn()` shifts the front of `queue` into `current`, refills it, and sets `next = queue[0]` (so the `#next-canvas` preview is unchanged — it's just `queue[0]`). The queue exists so the "see next 5" ability has something to reveal; pieces are pre-generated with their spawn `x`/`y` already computed, which is safe since that doesn't depend on board state.
- **Locking**: `lockPiece()` → `merge()` (writes piece into `board`) → `clearLines()` (scans bottom-up, splices full rows, unshifts empty ones, updates score/lines/level/`dropInterval`/energy) → `spawn()` (promotes the front of `queue` to `current`, refills the queue, checks game-over collision).
- **Game loop**: `loop(ts)` runs via `requestAnimationFrame`, accumulating `dt` into `dropAccum`; when it exceeds the *effective* drop interval the piece drops one row or locks. Calls `updateEffects(dt)` (ticks down ability timers) and `draw()` every frame.
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` indexed by lines cleared at once, multiplied by `level`. Hard drop adds 2 pts/row dropped; soft drop adds 1 pt/row.
- **Leveling/speed**: level = `floor(lines / 10) + 1`; `dropInterval = max(100, 1000 - (level - 1) * 90)` ms. The loop's *effective* interval is `dropInterval / SLOW_FACTOR` while the slow-fall ability is active (see below), otherwise just `dropInterval`.
- **Energy bar & abilities**: every `clearLines()` call that clears at least one row charges `energy` by `ENERGY_GAINS[cleared]` (mirrors `LINE_SCORES`'s shape — bigger simultaneous clears charge more), capped at `ENERGY_MAX` (100) via `addEnergy()`. `updateEnergyHUD()` reflects `energy` as `#energy-fill`'s width and reveals `#ability-buttons` (and the `full` pulse class) once charged. When full, `activateAbility('preview' | 'slow')` — triggered by clicking a button or pressing `Digit1`/`Digit2` — drains `energy` to 0 and starts one of two timed effects, both `SLOW_DURATION`/`PREVIEW_DURATION` = 10000ms and both ticked down via `dt` in `updateEffects()` (not wall-clock timestamps, so they correctly freeze while paused, same as `dropAccum`):
  - `'preview'` sets `previewRemaining`, unhides `#preview-section`, and calls `drawPreviewQueue()` (renders `queue[0..4]` stacked in `#preview-canvas`, re-drawn from `spawn()` on every piece change while active) until the timer expires and the section is re-hidden.
  - `'slow'` sets `slowRemaining`, which `loop()` reads to divide `dropInterval` by `SLOW_FACTOR` (0.6), i.e. pieces fall at 60% speed; `#slow-indicator` shows a live countdown.
  Both abilities require `energy >= ENERGY_MAX` and are no-ops while `paused` or `gameOver`.
- **Ghost piece**: `ghostY()` projects `current` straight down until collision; drawn at `globalAlpha = 0.2`.
- **Rendering**: `draw()` clears and redraws the whole board every frame (grid lines, locked blocks, ghost, current piece) — no dirty-rect optimization. `drawNext()` renders the preview canvas the same way; `drawPreviewQueue()` does the same for up to 5 stacked pieces in `#preview-canvas` while that ability is active.
- **Input**: single `keydown` listener switches on `e.code` (arrows, `KeyX` rotate, `Space` hard drop, `KeyP` pause, `Digit1`/`Digit2` abilities). Movement/rotation happen synchronously against `collide()`; `updateHUD()` runs after every input.
- **Pause/Game over**: both reuse the same `#overlay` element, swapping `overlayTitle`/`overlayScore` text. `togglePause()` cancels/restarts the animation frame loop. `restartBtn` re-runs `init()`.

### Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `LINE_SCORES`, initial `dropInterval`, `ENERGY_MAX`, `ENERGY_GAINS`, `QUEUE_SIZE`, `PREVIEW_DURATION`, `SLOW_DURATION`, `SLOW_FACTOR`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS × BLOCK`, `ROWS × BLOCK`).
