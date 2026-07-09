# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Asteroids clone built with plain HTML5 Canvas and vanilla ES6+ JavaScript — no build step, no bundler, no dependencies, no package.json. The entire game logic lives in a single file, `game.js`.

## Running the game

Open `index.html` directly in a browser, or serve it locally:

```bash
npx serve .
```

Then visit `http://localhost:3000`.

There is no build, lint, or test tooling in this repo — changes to `game.js` take effect on a browser refresh. Verify changes by loading `index.html` in a browser and playing the game (controls: `←`/`→` rotate, `↑` thrust, `Space` shoot).

## Architecture

`game.js` is organized as a set of sections (marked by `// ── Section ──` banner comments) in dependency order:

1. **Input** — `keys`/`justPressed` maps populated by `keydown`/`keyup` listeners. `pressed(code)` consumes a "just pressed" edge (used for single-fire actions like shooting or restarting), while `keys[code]` gives continuous hold state (used for rotation/thrust).
2. **Utils** — `wrap(v, max)` implements toroidal space (used by every entity's `update` for both x and y against `W`/`H`), plus `dist`, `rand`, `randInt`.
3. **Entity classes** — `Bullet`, `Asteroid`, `Ship`, `Particle`. Each follows the same shape: constructor sets initial state, `update(dt)` advances physics and sets `this.dead = true` when expired/invalid, `draw()` renders via `ctx`. There is no shared base class — the convention is duck-typed (`dead`, `update`, `draw`, `x`, `y`, `radius`).
4. **Game state** (module-level `let` variables: `ship`, `bullets`, `asteroids`, `particles`, `score`, `lives`, `level`, `state`, `deadTimer`) plus `initGame()`, `nextLevel()`, `explode()`, `killShip()`.
5. **Update** — `update(dt)` branches on `state` (`'playing' | 'dead' | 'gameover'`) and, in the `'playing'` branch, does: input → per-entity `update` → dead-filtering → bullet/asteroid collision (splits asteroids via `Asteroid.split()`, awards `POINTS[size]`, spawns explosion particles) → ship/asteroid collision (respects `ship.invincible`) → level-complete check (`asteroids.length === 0` → `nextLevel()`).
6. **Draw** — `draw()` clears the canvas and renders particles → asteroids → bullets → ship → HUD, then any state overlay (only `gameover` has one currently; `drawOverlay` is generic and reusable).
7. **Main loop** — `requestAnimationFrame`-driven `loop(ts)` computing clamped `dt` (max 0.05s) and calling `update(dt)` then `draw()`.

Key invariants to preserve when editing:
- All moving entities wrap position through `wrap()` against `W`/`H` (800×600) — don't use raw modulo or clamping instead.
- Entities signal removal via `this.dead = true`; arrays are pruned each frame with `.filter(e => !e.dead)`, never spliced in place mid-iteration.
- Asteroid size scale is 3 (large) → 1 (small); `RADII`, `SPEEDS`, `POINTS` arrays are indexed by size and index 0 is unused padding.
- `state` machine values (`'playing'`, `'dead'`, `'dead'`→timer→`'playing'` on respawn, `'gameover'`) gate what `update`/`draw` do each frame — new game states should follow this same branch-early pattern.

Note: the README describes power-ups and a special "shooting star" asteroid type that are not present in the current `game.js` — treat those as aspirational/future features, not existing behavior, unless you add them.
