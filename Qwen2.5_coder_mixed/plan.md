# Gravity Shift Runner – Implementation Plan

Deliverable: ONE file `index.html` (inline `<style>` + `<script>`). No libraries, no images, no fonts. Vanilla JS only.
Build in the order of the steps below. Keep code compact. Do not add features not listed here.

## 0. Constants (put at top of script, use these exact values)

| Name | Value | Meaning |
|---|---|---|
| W, H | 960, 540 | canvas size (CSS-scale to fit window, keep aspect) |
| STRIP | 40 | thickness of floor and ceiling strips |
| FLOOR_Y | 500 | top edge of floor (H - STRIP) |
| CEIL_Y | 40 | bottom edge of ceiling |
| SIZE | 32 | player square side |
| PLAYER_SX | 200 | fixed screen x of player (world scrolls) |
| G | 2400 | gravity px/s² |
| VMAX | 1500 | max vertical speed |
| SPEED0 / SPEED_MAX / ACCEL | 340 / 520 / 4 | run speed px/s, +ACCEL per second |
| COOLDOWN | 0.30 | seconds between flips |
| FLIP_T | 0.60 | time to cross floor→ceiling (computed: sqrt(2*428/2400)) |
| SPIKE_W, SPIKE_H | 32, 28 | spike triangle size |

## 1. HTML/CSS layout
- `body`: background `#0b0d14`, margin 0, flex-center, no scroll.
- `#wrap` (position relative, holds canvas + overlays). `canvas#c` width=W height=H, `max-width:100vw; max-height:100vh`.
- Three overlay `div`s absolutely centered over canvas, dark translucent bg `rgba(11,13,20,.8)`, light text, accent color `#4fd1ff`:
  - `#start`: title "GRAVITY SHIFT RUNNER", text "Space / Click = flip gravity", text "Press Space or click to start".
  - `#over`: "GAME OVER", final score, best score, `<button id="restart">Restart</button>`.
  - HUD is drawn on the canvas (step 7), not DOM.
- Overlays toggled by `.hidden{display:none}`.
- Button style: rounded, accent bg, dark text, hover brighten.

## 2. State
```
state: 'start' | 'playing' | 'over'
player: {x (world), y, vy, dir (+1 = gravity down/floor, -1 = up/ceiling), grounded, flipCd (seconds left)}
speed, score(float distance/10), best (from localStorage 'gsr_best', wrapped in try/catch)
world: {pits[], spikes[], zones lastEnd, lastSurface, nextX}
trail[], particles[], shake (magnitude number)
```
`reset()` sets EVERYTHING to initial values: player x=0, y=FLOOR_Y-SIZE, vy=0, dir=1, grounded=true, flipCd=0, speed=SPEED0, score=0, empty arrays (pits, spikes, trail, particles), shake=0, `lastEnd=900`, `lastSurface=null`, then calls `generate()`. Restart button: `reset(); state='playing'`; hide overlays. NO page reload.

## 3. Input
- `keydown` code `Space`: `preventDefault()`, ignore `e.repeat`. In `start` → `reset()` + play. In `playing` → `flip()`. In `over` → ignore (button only).
- `mousedown` on canvas: same logic as Space.
- `flip()`: only if `flipCd <= 0`: `dir *= -1; vy = 0; grounded = false; flipCd = COOLDOWN`.

## 4. Main loop
`requestAnimationFrame(loop)` always running. `dt = min((now-last)/1000, 1/30)`. If `playing`: `update(dt)`. Always `draw()`.

`update(dt)`:
1. `speed = min(SPEED_MAX, speed + ACCEL*dt)`; `player.x += speed*dt`; `score = player.x/10`.
2. `flipCd = max(0, flipCd - dt)`.
3. Physics: `vy += G*dir*dt`, clamp to ±VMAX, `prevY = y`, `y += vy*dt`.
4. Surface landing (see §6). Set `wasGrounded` before, detect air→ground transition → landing particles.
5. Death checks (§6). On death: `die()`.
6. `camX = player.x - PLAYER_SX`. Call `generate()` and cull items with x < camX - 100.
7. Push `{x: player.x, y: player.y}` to `trail`; keep only last 14.
8. Update particles/shake (§8).

## 5. Level generation (the solvability guarantee)
World is: full floor strip + full ceiling strip, MINUS **pits** (gaps in the strip; the strip pieces between pits are the platforms), PLUS **spikes** on strips. Everything is a **zone**: `{surface:'floor'|'ceil', x0, x1, type:'spike'|'pit'}`. Player must be on the OPPOSITE surface while crossing a zone.

Zones are generated strictly left→right, never overlapping, so only the previous zone matters.

`generate()`: while `lastEnd < player.x + 2000`:
1. `surface` = random 'floor'/'ceil'. `type` = spike 70% / pit 30%.
2. Width: spike → `32 * randInt(1,3)`; pit → `randInt(64,160)`.
3. `MIN_GAP = (speed + 60) * 0.9`  (= flip time 0.6 s + 0.3 s reaction, in px; the +60 covers speed growth).
4. `gap = (surface != lastSurface) ? MIN_GAP + rand(0,200) : 96 + rand(0,200)`.
5. `x0 = lastEnd + gap; x1 = x0 + width`. Add to `pits` (type pit) or `spikes` (type spike: one spike every 32 px from x0).
6. `lastEnd = x1; lastSurface = surface`.

Why it is always passable: first zone is ≥900 px from start; when consecutive zones are on different surfaces the gap ≥ MIN_GAP, which is enough distance to flip (cooldown 0.3 < gap time) and finish crossing (0.6 s) before the next zone starts. Same-surface neighbours need no flip. Pit zones need the player on the other strip, which is always solid because zones never overlap.

## 6. Collision rules
- Player AABB shrunk by 3 px on each side.
- **Support:** surface at `dir` is solid at player if the player's CENTER x is not inside a pit of that surface.
- **Landing:** if `dir==1` and `y+SIZE >= FLOOR_Y` and `prevY+SIZE <= FLOOR_Y+4` and supported → `y = FLOOR_Y-SIZE; vy=0; grounded=true`. Mirror for `dir==-1` with `y <= CEIL_Y` and `prevY >= CEIL_Y-4` → `y = CEIL_Y; vy=0; grounded=true`. Otherwise `grounded=false` (player falls through the pit).
- **Fall death:** `y > H + 60` or `y < -60 - SIZE`.
- **Spike death:** floor spike hitbox = rect centered on spike, width 20, height 22, bottom at FLOOR_Y. Ceiling spike: width 20, height 22, top at CEIL_Y. Die if player AABB overlaps any spike rect (only check spikes within camX..camX+W).
- `die()`: `state='over'`, `shake=14`, burst of 30 particles at player, update `best` if `floor(score)>best` and save to localStorage, fill `#over` text, show `#over`. Keep drawing the frozen scene with shake decaying (draw continues in `over`; run particle/shake update there too).

## 7. Rendering (all on canvas, order matters)
1. Clear with `#0b0d14`. `ctx.save(); translate(random(-shake,shake), random(-shake,shake))`.
2. Faint background parallax: a few vertical grid lines at `x = -(camX*0.3) % 80` spacing 80, color `rgba(255,255,255,.04)`.
3. Floor and ceiling strips: color `#1b2033`, accent edge line `#4fd1ff` 2 px on the playfield side. Draw them as segments between pits (cut out pit ranges from `camX` to `camX+W`).
4. Spikes: triangles, color `#ff4d6d`, base on the strip, apex pointing into the playfield. Screen x = `worldX - camX`.
5. Trail: for each entry i (older → newer) draw a square of SIZE*(i/len) centered on its position, `rgba(79,209,255, 0.35*i/len)`.
6. Player: square `#4fd1ff` with 2 px lighter border, at screen x `PLAYER_SX`. Hide when `state=='over'`.
7. Particles: small squares, color alpha = life.
8. `ctx.restore()`. HUD (no shake): top-left `Score: N` (floor), top-right `Best: N`, top-center flip meter: 160×10 rounded bar, background `#222`, fill width = `160 * (1 - flipCd/COOLDOWN)`, fill color accent when full, dimmer while charging.

## 8. Juice
- **Landing particles** (air→ground transition only): 10 particles at the contact point (player center x, surface y); velocity: x random ±120, y away from surface random 60–260; gravity on them `dir`-independent: `vy += 600*dt` toward the surface; life 0.5 s, size 2–4 px.
- **Shake:** on `die()` set `shake=14`; each frame `shake *= 0.9`; below 0.3 → 0. Also tiny shake 2 on landing.
- **Trail:** §4 step 7 / §7 step 5. Clear `trail` in `reset()`.
- Particle update: move, decrement life by dt, remove when life ≤ 0.

## 9. Flow summary
Page load → state `start` (start overlay shown, scene idle). Space/click → `reset()`, hide overlay, `playing`. Death → `over`, show overlay with Restart. Restart click → `reset()`, hide overlay, `playing`. Best score persisted via `localStorage` key `gsr_best`.

## 10. Self-check before finishing
- [ ] Opens by double-click, no console errors, no external requests.
- [ ] Space and click both flip; holding Space does not repeat-flip; page doesn't scroll on Space.
- [ ] Meter empties on flip and refills in 0.3 s; flips ignored while charging.
- [ ] Spikes/pits appear on both floor and ceiling; first hazard is far from start.
- [ ] Death → shake + overlay; Restart resets score, speed, level, trail, particles with no reload.
- [ ] Best score survives page refresh.
