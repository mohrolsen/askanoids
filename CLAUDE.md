# Askanoids — Project Log

> **Keep this file updated.** Every session that changes specs, mechanics, visuals, or deployment should add an entry at the bottom. This is the single source of truth for what the game is and why decisions were made.

---

## Project overview

Single-file HTML5 mobile game. Asteroids × Arkanoid hybrid. No framework, no build step, no external dependencies. Everything — layout, game engine, rendering — lives in `index.html` (also versioned as `2026-04-10 Askanoids.html` per folder naming convention).

**Repo:** https://github.com/mohrolsen/askanoids  
**Live:** https://askanoids.vercel.app  
**Vercel project:** martins-projects-7dcc8758/askanoids (connected to GitHub main branch — push to deploy)

---

## Architecture

### Layout
Three vertical panels, portrait-only:
- **20% top** — status bar: wave number, score, lives
- **60% middle** — SVG game field
- **20% bottom** — controls: rotation slider + 2 power-up slots

Landscape mode shows a full-screen "rotate device" overlay (CSS `@media (orientation: landscape)`) and auto-pauses the game loop via `window.matchMedia`.

### Rendering
- All game elements drawn in SVG — no sprites, no canvas
- `viewBox="0 0 400 400"`, `preserveAspectRatio="xMidYMid meet"`, fills game area container
- Center of play field: `(200, 200)` — turret anchor
- Enemy spawn radius: 272px from center (just outside the field)
- SVG `<defs>` filters: `#gc` (cyan glow), `#gm` (magenta), `#gg` (green), `#gy` (yellow) — all `feGaussianBlur` + `feMerge` neon glow pattern
- Scanline overlay: CSS `repeating-linear-gradient` on `#game-area::after`
- Enemy spin: CSS `@keyframes spin` with `transform-box: fill-box; transform-origin: center` on the inner `<rect>` — classes `spin-n` (1.5s), `spin-f` (0.55s), `spin-s` (3.2s), `spin-m` (1s), `spin-b` (4.5s)

### Game loop
`requestAnimationFrame` with delta time capped at 50ms (prevents physics explosion after tab switch). State machine: `title | playing | wave_clear | game_over | paused`.

### JS architecture (key functions)
- `buildTurret()` — tears down and recreates turret SVG group; called on power-up state change
- `syncTurretAngle()` — sets `rotate(angle, 200, 200)` transform on turret group
- `muzzlePos(angleOffset)` — calculates bullet spawn position and velocity from angle + muzzle offset (45px)
- `autoFire(dt)` — rate-based fire tied to `G.lastFire` accumulator
- `checkCollisions()` — iterates bullets backwards, AABB vs enemies then circle vs drops
- `buildWaveQueue(w)` — builds shuffled enemy type array for wave `w`
- `spawnEnemy(type, pos?)` — creates SVG group, appends to `#enemies-layer`, pushes to `G.enemies`
- `killEnemy(e, i)` — explosion, drop chance, splitter/boss split logic, score update
- `collectDrop(drop, di, bi)` — stores power-up in first empty inventory slot, removes both drop and bullet
- `activatePowerUp(slot)` — applies power-up effect, rebuilds turret if visual change needed
- `update(dt)` — main update dispatcher; skips most systems during `wave_clear`
- `$s(tag, attrs)` — `document.createElementNS` wrapper for SVG elements
- `setSVGHTML(el, html)` — safely sets innerHTML on SVG elements via a wrapper div

### State object (`G`)
```
phase, wave, score, lives, angle
bullets[], enemies[], drops[], effects[]
inventory[2]          — held power-ups (null or type string)
pw{ RAPID, MULTI, PIERCE }  — active timed power-up countdowns (seconds)
hasShield             — boolean, absorbed on next hit
waveTimer, spawnQueue[], spawnTimer, lastFire
keys{}                — currently held keyboard keys
highScore             — persisted to localStorage key 'ask-hs'
```

---

## Enemies

| Type     | Size | HP | Speed (base) | Color    | Score | Drop % | Notes |
|----------|------|----|--------------|----------|-------|--------|-------|
| basic    | 10px | 1  | 42 px/s      | #00ffff  | 10    | 12%    | |
| scout    | 8px  | 1  | 96 px/s      | #ff00ff  | 20    | 15%    | Fast |
| tank     | 18px | 3  | 28 px/s      | #ffff00  | 50    | 45%    | HP pips shown |
| splitter | 12px | 2  | 58 px/s      | #00ff41  | 100   | 25%    | Splits into 2 basics on death |
| boss     | 32px | 8  | 18 px/s      | #ff6600  | 200   | 100%   | Every 5 waves; splits into 2 tanks |

Speed scales: `baseSpeed * (1 + (wave - 1) * 0.045)` — 4.5% faster per wave.

Enemy movement: straight toward center `(200, 200)` with a small random `drift` angle (±0.25 rad) applied at spawn.

### Wave composition (wave `w`)
- Basics: `3 + w * 2`
- Scouts: from wave 3: `w - 2`
- Tanks: from wave 5: `floor((w - 4) / 2)`
- Splitters: from wave 6: `floor((w - 5) / 2)`
- Boss: if `w % 5 === 0` (waves 5, 10, 15…)
- Queue is Fisher-Yates shuffled before spawning
- Spawn interval: 0.42s between enemies
- Wave clear → 3.2s countdown → next wave

---

## Turret

- Centered at `(200, 200)`, rotates via `rotate(angle, 200, 200)` SVG transform
- Barrel points "up" at angle 0°; muzzle 45px from center
- Autofire — no player button needed
- Base fire rate: 2 shots/sec
- MULTI adds side barrels at ±25° offsets (3 shots per fire tick at ±15° spread)
- RAPID adds a pulsing magenta ring animation; changes stroke to magenta
- PIERCE tints barrel/stroke green

---

## Power-ups

Dropped by killed enemies (chance varies per type). Float toward center at 20 px/s. **Player must shoot the drop to collect it** (bullet-circle collision, drop radius 13px). Stored in inventory (max 2 slots). Activated with Z/X (keyboard) or slot buttons.

| ID     | Color   | Effect                          | Duration  |
|--------|---------|---------------------------------|-----------|
| RAPID  | #00ffff | 3× fire rate (6/s)              | 10s       |
| MULTI  | #ff00ff | Triple spread shot (±15°)       | 10s       |
| PIERCE | #00ff41 | Bullets pierce up to 3 enemies  | 10s       |
| SHIELD | #ffff00 | Absorbs 1 enemy hit             | Until hit |
| NOVA   | #ff00ff | Clears all enemies on screen    | One-time  |

Timed power-ups (RAPID/MULTI/PIERCE) accumulate: activating the same type while active adds 10s.

---

## Controls

### Mobile
- `<input type="range">` slider, min=0 max=359, absolute angle mapping (0°=up, 90°=right)
- Slot buttons: tap to activate held power-up
- Slider styled: gradient track `#00ffff88 → #ff00ff88`, glowing cyan thumb

### Keyboard
- `←` / `→` — rotate turret (190°/sec)
- `Z` — activate slot 0
- `X` — activate slot 1
- `P` / `Escape` — pause/resume

---

## Visual aesthetic

```
Background:  #05050f
Grid lines:  #0c1e2e (40px pattern, 0.6px stroke)
Cyan:        #00ffff  (turret, basic enemies, bullets, UI)
Magenta:     #ff00ff  (scouts, power-ups, title)
Green:       #00ff41  (splitter, PIERCE state)
Yellow:      #ffff00  (tanks, SHIELD)
Orange:      #ff6600  (boss)
Red flash:   #ff2244  (game over, damage)
```

Font: `'Courier New', Courier, monospace` — uppercase throughout, letter-spacing on all labels.

---

## Session log

### 2026-04-10 — Initial build
- Designed and implemented complete game from scratch in one session
- Single HTML file, ~1,100 lines, ~40KB
- Deployed to GitHub (mohrolsen/askanoids) and Vercel (askanoids.vercel.app)
- Decisions made:
  - SVG-only rendering (no canvas) — works well for this scale, glow filters are cheap
  - Absolute slider (not velocity/joystick) — user preference
  - No sound for now — can add Web Audio API oscillators later without restructuring
  - `index.html` copy maintained alongside dated file for Vercel root serving
