# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Vanilla JavaScript Tetris (README is in Spanish). No dependencies, no build step, no package.json — just three files:

- `index.html` — DOM structure: `#board` canvas (300×600, 10×20 grid of 30px blocks), `#next-canvas` (120×120) for the next-piece preview, HUD spans (`#score`, `#lines`, `#level`), and the `#overlay` used for both pause and game-over states.
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

Everything is global, procedural state — no classes, no modules. Key mutable state lives in one destructured `let` block: `board, current, next, score, lines, level, paused, gameOver, lastTime, dropAccum, dropInterval, animId`.

- **Board model**: `board` is a `ROWS × COLS` matrix; each cell is `0` (empty) or a color index `1–8` identifying which piece locked there.
- **Pieces**: the 7 standard tetrominoes plus an 8th special piece, the **Nut** (`NUT = 8`), are defined as square matrices in `PIECES` (index 0 unused/null). `randomPiece()` deep-copies a shape and centers it at spawn, drawing uniformly from all 8 types. The Nut is a 3×3 ring (`[[8,8,8],[8,0,8],[8,8,8]]`) with a real `0` hole in the center — `collide()` treats it as empty (nothing blocks moving into it) and `merge()` leaves it `0` in `board`, so a row containing an unbroken Nut hole can never satisfy `clearLines()`'s `every(v => v !== 0)` check until the ring is broken from above. `draw()`/`drawNext()` render the hole as a stroked circle via `drawNutHole()`; `isNutHole(r, c)` detects a locked-in hole on the board by checking all 8 neighbors are `NUT`.
- **Rotation**: `rotateCW(shape)` transposes + reverses rows. `tryRotate()` applies it then tries wall-kick offsets `[0, -1, 1, -2, 2]` columns, keeping the first that doesn't collide.
- **Collision**: `collide(shape, ox, oy)` checks bounds and overlap against `board`; used for movement, rotation, ghost-piece projection, and spawn (game-over check).
- **Locking**: `lockPiece()` → `merge()` (writes piece into `board`) → `clearLines()` (scans bottom-up, splices full rows, unshifts empty ones, updates score/lines/level/`dropInterval`) → `spawn()` (promotes `next` to `current`, generates a new `next`, checks game-over collision).
- **Game loop**: `loop(ts)` runs via `requestAnimationFrame`, accumulating `dt` into `dropAccum`; when it exceeds `dropInterval` the piece drops one row or locks. Calls `draw()` every frame.
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` indexed by lines cleared at once, multiplied by `level`. Hard drop adds 2 pts/row dropped; soft drop adds 1 pt/row.
- **Leveling/speed**: level = `floor(lines / 10) + 1`; `dropInterval = max(100, 1000 - (level - 1) * 90)` ms.
- **Ghost piece**: `ghostY()` projects `current` straight down until collision; drawn at `globalAlpha = 0.2`.
- **Rendering**: `draw()` clears and redraws the whole board every frame (grid lines, locked blocks, ghost, current piece) — no dirty-rect optimization. `drawNext()` renders the preview canvas the same way.
- **Input**: single `keydown` listener switches on `e.code` (arrows, `KeyX` rotate, `Space` hard drop, `KeyP` pause). Movement/rotation happen synchronously against `collide()`; `updateHUD()` runs after every input.
- **Pause/Game over**: both reuse the same `#overlay` element, swapping `overlayTitle`/`overlayScore` text. `togglePause()` cancels/restarts the animation frame loop. `restartBtn` re-runs `init()`.

### Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS × BLOCK`, `ROWS × BLOCK`).
