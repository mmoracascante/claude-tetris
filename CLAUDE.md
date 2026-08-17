# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Vanilla-JS Tetris. No build, no dependencies, no `package.json`, no bundler/transpiler. Three files: `index.html`, `style.css`, `game.js`. Canvas 2D rendering. UI text is Spanish; code identifiers are English.

## Run

Open directly or serve statically:

```bash
open index.html                 # macOS — no server needed
python3 -m http.server 8000     # then http://localhost:8000
```

No test suite, no lint config. Verification is manual in-browser.

## Architecture (`game.js`, ~300 lines, single global scope)

`'use strict'` module with module-level mutable state (`board, current, next, score, lines, level, paused, gameOver, dropInterval, dropAccum, animId`). No classes. Understanding these coupled pieces matters:

- **Data model**: `board` is a `ROWS×COLS` matrix; each cell holds `0` (empty) or a **color index 1–7**. That same index keys both `COLORS` and `PIECES` — the two arrays are index-aligned and their leading `null` slot exists so index 1 = I-piece. Changing one requires changing the other.
- **Pieces are mutated in place**: `current.shape` is reassigned on rotation (`tryRotate` → `rotateCW`, transpose + reverse rows). `randomPiece()` deep-copies from `PIECES` so the templates stay pristine.
- **Collision is the single gate**: `collide(shape, x, y)` is reused for movement, rotation wall-kicks, ghost projection, soft/hard drop, and spawn-time game-over detection. Any new piece behavior should route through it rather than re-check bounds.
- **Wall kicks**: `tryRotate` tries horizontal offsets `[0,-1,1,-2,2]` after rotating; first non-colliding offset wins.
- **Game loop**: `loop(ts)` is `requestAnimationFrame`-driven; accumulates `dt` into `dropAccum` and steps the piece down once `dropAccum >= dropInterval`. Pausing cancels the frame (`cancelAnimationFrame(animId)`) and resuming restarts the loop. `loop` also guards on `gameOver || paused` at entry and re-checks `gameOver` before rescheduling: `cancelAnimationFrame` is a no-op when `endGame()` fires from inside the currently-executing frame (`loop` → `lockPiece` → `spawn` → `endGame`), so without those guards the loop would resurrect itself and keep spawning pieces after game over.
- **Lock flow**: `lockPiece()` = `merge()` (stamp shape into `board`) → `clearLines()` → `spawn()`. `spawn()` promotes `next` to `current`, draws the new preview, and calls `endGame()` if the fresh piece already collides.
- **Scoring/level coupling**: `clearLines` uses `LINE_SCORES` (`[0,100,300,500,800]`) × `level`; `level = floor(lines/10)+1`; `dropInterval = max(100, 1000-(level-1)*90)`. Soft drop +1/row, hard drop +2/cell.
- **Rendering**: `draw()` redraws everything each frame — grid, board, ghost (`ghostY()` projected, `alpha 0.2`), then current piece. `drawBlock()` is shared by board and the separate `next-canvas` preview (different block size arg).

## Constants ↔ HTML dependency

`COLS`, `ROWS`, `BLOCK` (top of `game.js`) define geometry. If you change them, you **must** update `<canvas id="board">` `width`/`height` in `index.html` to `COLS*BLOCK` × `ROWS*BLOCK`, or rendering desyncs from logic. DOM element IDs are looked up once at the top of `game.js`; renaming an ID in HTML breaks the corresponding lookup.
