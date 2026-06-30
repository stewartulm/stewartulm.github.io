Create a complete web app implementing a 3D arena survival game called "Neon Drift" using Three.js. The game features neon cyberpunk aesthetics, third-person camera, momentum-based movement, particle effects, orb collection, and enemy pursuit.

## Core Gameplay

- **Arena**: A square arena (half-size 40 units) bounded by glowing coral walls. Ground is a custom shader with multi-scale grid lines (2m and 8m spacing), distance fade toward edges, pulse rings emanating from the player's position, wave-dependent color shift (cold cyan → warm amber), coral edge glow, and subtle scanlines.
- **Player**: A glowing cyan icosahedron with a white core, a transparent outer shell (emissive, metalness 0.4), a rotating ring torus, and a pulsing point light. Movement is camera-relative (WASD/arrows), momentum-based with acceleration 60 and friction 4.5. Max speed 22. Clamped to arena with bounce-back. Smooth turning toward velocity direction with visual tilt.
- **Trail**: A ribbon of 40 line segments (additive blending, vertex colors) following the player. Width narrows toward tail, colors fade cyan→transparent. Brighter during boost.
- **Orbs**: 4 active at a time. Amber icosahedron with white core, halo sphere (additive blending), point light. Bob up/down, rotate, pulse. Spawned randomly in arena (avoiding player). Collect at distance < 1.4: combo increments (max 8), score += 10×combo, particle burst, expanding ring, popup text, camera kick, immediate respawn.
- **Enemies**: Crimson octahedron body (dark with coral emissive, metalness 0.6), inner glowing pink sphere, 6 cone spikes radially arranged, coral point light. Spawn at arena edges. Home toward player at speed = 5 + wave×0.7 + random 0–1.2. Body rotates, inner rotates opposite, face player. Spawn animation: scale grows 0→1 in 0.33s, emissive pulses. Collision at distance < 1.4 (not invulnerable, spawn animation > 0.4s): lose life, knockback, combo reset, particle burst, ring effect, damage flash, camera shake, enemy destroyed. 0 lives → massive explosion → game over.
- **Enemy spawning**: Timer-based, interval = max(0.45, 2.6 - wave×0.25) seconds, randomized ±30%. Cap at 18 + wave×2 enemies.
- **Waves**: Increment every 20 seconds. On wave change: particle burst + ring at player. Wave affects enemy speed, spawn rate, ground color.
- **Particles**: Pool of 600. Custom ShaderMaterial with soft-circle fragment. Gravity, ground bouncing, color fade over life. Used for orb collect, hits, death, boost exhaust, enemy trail.
- **Expanding rings**: RingGeometry (inner 0.5, outer 0.7), additive blending, scale 1→9 over 0.8s, opacity fades. Used on orb collect and hits.

## Particle System (Detailed)

The particle system is a **pooled buffer** of 600 particles using a custom `ShaderMaterial` with additive blending, transparency, and no depth write.

### Particle Shader

- **Vertex shader**: Takes `aSize` (float attribute) and `color` (vec3 attribute) per particle. Passes `vColor` as varying. Computes `gl_PointSize = aSize * (300.0 / -mv.z) * uPixel`, where `uPixel` is the renderer's pixel ratio. This makes particles scale correctly on high-DPI displays and shrink with distance.
- **Fragment shader**: Renders a soft circular sprite. Computes `d = length(gl_PointCoord - 0.5)`. Discards if `d > 0.5`. Otherwise outputs `vColor * (1.0 + a)` with alpha `a = pow(1.0 - d*2.0, 2.0)`. This creates a bright center that fades smoothly to the edge — a glowing dot.

### Particle Pool

- 600 pre-allocated particle objects, each with `{pos, vel, col, life, maxLife, size, active}`.
- The `emit()` function accepts `{count, color, speed, life, size, spread, vel}`.
  - If `vel` is provided, particles copy that exact velocity (used for boost exhaust).
  - Otherwise, velocity is random: `x,z = (random*2-1)*speed*spread`, `y = random*speed*0.8`, plus extra randomness `±speed*0.5` in x and z.
  - Size is randomized: `size * (0.6 + random*0.6)`.
- Particles are recycled: the function iterates from index 0 and picks the first inactive particle per emission.

### Per-frame Update

- Gravity: `vel.y -= 9 * dt` (Earth-like).
- Air drag: `vel *= (1 - dt*1.5)`.
- Ground bounce: if `pos.y < 0.1`, clamp to 0.1, reflect `vel.y *= -0.4`, damp horizontal `vel *= 0.7`.
- Color fades linearly: `col.rgb * (life / maxLife)`.
- Size fades: `size * (0.3 + t*0.7)` where `t = life/maxLife`.
- Draw range updated to active count; position/color/size attributes flagged `needsUpdate = true`.

### Emission Patterns per Event

| Event             | Count | Color            | Speed | Life | Size | Spread | Notes                                                                                                                  |
| ----------------- | ----- | ---------------- | ----- | ---- | ---- | ------ | ---------------------------------------------------------------------------------------------------------------------- |
| **Orb collect**   | 30    | Amber (0xffb627) | 7     | 1.0  | 3    | 1.3    | + 10 white particles (speed 4, life 0.6, size 2, spread 0.8)                                                           |
| **Player hit**    | 40    | Pink (0xff006e)  | 9     | 0.9  | 3    | 1.4    | + 20 coral particles (speed 6, life 0.7, size 2.5, spread 1)                                                           |
| **Player death**  | 80    | Cyan (0x00f5d4)  | 12    | 1.5  | 4    | 2      | + 40 white particles (speed 8, life 1.2, size 3, spread 1.5)                                                           |
| **Wave change**   | 40    | Cyan (0x00f5d4)  | 8     | 1.2  | 3    | 1.5    | Burst at player position                                                                                               |
| **Boost exhaust** | 2     | Cyan (0x00f5d4)  | 5     | 0.5  | 2    | 0.6    | Emitted 80% of frames while boosting; uses explicit `vel` vector pointing backward from player angle, magnitude 6, y=2 |
| **Enemy trail**   | 1     | Coral (0xff3864) | 1.5   | 0.6  | 1.8  | 0.5    | Emitted 15% chance per enemy per frame                                                                                 |

## Lighting (Detailed)

### Hemisphere Light

- `HemisphereLight(0x88e0ff, 0x081014, 0.35)` — sky color is light blue, ground color is near-black, intensity 0.35. Provides subtle ambient fill.

### Key Directional Light

- `DirectionalLight(0xb0f5e8, 0.9)` — pale cyan-white, intensity 0.9.
- Position: `(20, 30, 18)` — high and to the right-front of the arena.
- Casts shadows: `shadow.mapSize = (2048, 2048)`, `PCFSoftShadowMap`.
- Shadow camera bounds: `left/right = -40..40`, `top/bottom = -40..40`, `near=1, far=80`, `bias=-0.0008`.

### Rim Directional Light

- `DirectionalLight(0xff3864, 0.25)` — coral color, low intensity 0.25.
- Position: `(-15, 12, -10)` — behind-left of the arena, creating dramatic backlit rim lighting on objects.

### Ground Point Light

- `PointLight(0x00f5d4, 0.6, 60, 2)` — cyan, intensity 0.6, range 60, decay exponent 2.
- Starts at `(0, 2, 0)`.
- **Follows player**: each frame, `position.set(player.x, 2, player.z)`.
- **Intensity varies with speed**: `0.5 + speedFactor*0.6`, where `speedFactor = min(1, speed/22)`.

### Player Point Light

- `PointLight(0x00f5d4, 2.4, 18, 2)` — cyan, intensity 2.4, range 18, decay 2.
- Positioned at `(0, 1.2, 0)` relative to player group.
- **Pulses**: `intensity = 2.2 + sin(time*4)*0.4` — oscillates between 1.8 and 2.6 at ~4Hz.

### Per-Orb Point Light

- `PointLight(0xffb627, 1.6, 10, 2)` — amber, intensity 1.6, range 10, decay 2.
- Positioned at y=0 within the orb group.
- **Pulses**: `intensity = 1.4 + sin(bob*3)*0.4`, where `bob` accumulates at `dt*2` per frame.

### Per-Enemy Point Light

- `PointLight(0xff3864, 1.2, 8, 2)` — coral, intensity 1.2, range 8, decay 2.
- Positioned at the group origin within the enemy group.

## Screen Shake (Detailed)

- State variable `shake` is a float (0 = no shake, positive = shaking).
- **Set to 0.7** on `damageFlash()` (player hit).
- **Set to 0.9** on `hitPlayer()` (enemy collision).
- **Set to max(shake, 0.15)** on `collectOrb()` (orb collection kick).
- **Decays**: `shake -= dt * 2.5` per frame, clamped to ≥ 0.
- **Applied to camera**: each frame, if `shake > 0`:
  - `camPos.x += (random*2-1) * s * 0.6`
  - `camPos.y += (random*2-1) * s * 0.4`
  - `camPos.z += (random*2-1) * s * 0.6`
    where `s = max(0, shake)`.
  - Note: shake is _added to the lerped camera position_, not multiplied — it accumulates offset until decay brings it back.

## Additional Visual Effects (Detailed)

### Damage Flash

- A fullscreen overlay div (`#damage`) with `radial-gradient(ellipse at center, transparent 30%, rgba(255,0,110,.55) 100%)`.
- Normally `opacity: 0` with `transition: opacity 0.12s ease-out`.
- On hit: adds class `.hit` → `opacity: 1` with `transition: opacity 0.05s`.
- Removes `.hit` after 80ms via `setTimeout`.
- The fast transition in (0.05s) and slow transition out (0.12s) creates a brief, sharp flash.

### Vignette

- A persistent fullscreen overlay (`#vignette`) with `radial-gradient(ellipse at center, transparent 40%, rgba(0,0,0,.55) 100%)`.
- Darkens arena edges, enhancing the neon glow in the center.

### Expanding Ring Effect

- `RingGeometry(inner=0.5, outer=0.7, segments=48)`, `MeshBasicMaterial` with color, transparent, `opacity: 1`, `side: DoubleSide`, additive blending, `depthWrite: false`.
- Rotated flat (`rotation.x = -PI/2`), positioned at `pos.y = 0.1`.
- Over 0.8s: scale grows linearly from `1` to `9`, opacity fades from `0.9` to `0`.
- On completion: `scene.remove()`, `geometry.dispose()`, `material.dispose()`.
- Used on: orb collect (amber), player hit (pink), player death (cyan + pink simultaneously), wave change (cyan).

### Score Popups (HTML overlay)

- Created dynamically as `<div class="pop">` appended to `#hud`.
- World position projected to 2D: `worldPos.clone().project(camera)`, then mapped to pixel coords.
- CSS animation `pop` (1s duration):
  - 0%: opacity 0, scale 0.6, centered on position.
  - 20%: opacity 1, scale 1.1, shifted 20% upward.
  - 100%: opacity 0, scale 0.9, shifted 180% upward.
- Color set to the event color (amber for orb collect `+N`, pink for hit `-1 LIFE`).
- Font: Rajdhani 800, 22px, with `text-shadow: 0 0 14px currentColor`.
- Removed after 1s via `setTimeout`.

### Player Blink During Invulnerability

- When `invuln > 0`: shell, core, and ring visibility toggle at 14Hz:
  `blink = floor(invuln * 14) % 2 === 0` → visible on even.
- When `invuln <= 0`: all three are always visible.
- Creates a rapid flashing effect (7 full on-off cycles per second).

### Wave Change Burst

- On wave increment: `emit(player.position + y=1, 40 cyan particles, speed 8, life 1.2, size 3, spread 1.5)` + `spawnRing(player.position + y=0.1, cyan)`.
- Creates a celebratory burst around the player.

### Orb Visual Animation

- **Bob**: `pos.y = 1 + sin(bob)*0.25` where bob accumulates at `dt*2`.
- **Shell rotation**: `rot.y = rot`, `rot.x = rot*0.7` where rot accumulates at `dt*1.5`.
- **Halo pulse**: `halo.scale = 1 + sin(bob*2)*0.15`, `halo.opacity = 0.15 + sin(bob*2)*0.08`.
- **Light pulse**: `light.intensity = 1.4 + sin(bob*3)*0.4`.

### Enemy Spawn Telegraph

- During first 0.5s of life (`spawnT < 0.5`): `body.emissiveIntensity = 2.5 - spawnT*2` — starts very bright (2.5) and fades to normal (1.4) over the 0.5s window.
- After 0.5s: normal pulse `1.4 + sin(time*5 + bob)*0.3`.
- Combined with scale animation `scale = min(1, spawnT*3)` (grows from 0→1 in 0.33s).
- This warns the player that an enemy just appeared.

### Enemy Visual Animation

- **Bob**: `pos.y = 0.9 + sin(time*4 + bob)*0.1`.
- **Body rotation**: `rot.y += rotSpeed*dt`, `rot.x += rotSpeed*0.6*dt` where rotSpeed is random ±2 rad/s.
- **Inner core rotation**: opposite direction `rot.y -= rotSpeed*dt`.
- **Inner core pulse**: `scale = 1 + sin(time*6 + bob)*0.2`.
- **Face player**: `group.rotation.y = atan2(direction.x, direction.z)`.

### Player Visual Feedback (Tilt & Glow)

- **Tilt**: `tiltAmount = min(1, speed/22) * 0.35`. Shell rotation:
  - `shell.rot.x = sin(time*6)*0.15 + tiltAmount * (-playerVel.z)*0.02`
  - `shell.rot.z = sin(time*5)*0.15 + tiltAmount * playerVel.x*0.02`
- **Core spin**: `core.rot.y = time*2`, `core.rot.x = time*1.5`.
- **Ring spin**: `ring.rot.z = time*1.2`.
- **Boost glow**: `emissiveIntensity = 1.6 + (boosting ? 1.5 : 0) + sin(time*4)*0.2`.
- **Boost opacity**: `shell.opacity = 0.85 + (boosting ? 0.1 : 0)`.

## Rendering & Lighting

- WebGLRenderer with antialias, high-performance, PCFSoftShadowMap, sRGBEncoding, ACESFilmic tone mapping (exposure 1.15). Pixel ratio capped at 2.
- Scene with FogExp2 (0.022 density, color #050a0e).
- HemisphereLight (blue sky, dark ground). DirectionalLight key (white-cyan, 2048² shadows). Rim directional light (coral, behind-left). PointLight (cyan, follows player at ground level).
- Per-orb point light (amber). Per-enemy point light (coral).

## Camera

- Third-person follow behind player relative to facing angle. Distance 14 + speedFactor×3, height 11 + speedFactor×1.5. Lerp smoothing (3× dt). Looks ahead of player (velocity×0.25, lerp 5× dt). Shake on damage/collection. Menu/over: idle orbit at radius 18, height 12, 0.15 rad/s.

## HUD (HTML/CSS overlays)

- Rajdhani font (Google Fonts). Dark background (#050a0e), light text (#e8f4f8).
- Score: top-left, 42px gradient (white→cyan), animated lerp, 4-digit padding.
- Lives: top-right, 3 star shapes (clip-path polygon), lost state: scale 0.6, rotate 20°, opacity 0.12, grayscale.
- Combo: centered below, shows when > 1, gradient (white→amber).
- Wave timer: bottom center, thin 3px bar with gradient fill (cyan→amber→coral), updates over 20s wave duration.
- Vignette: radial gradient dark overlay. Damage flash: radial pink overlay (80ms).
- Score popups: project 3D→2D, animated upward 1s, additive blending.
- Responsive: at ≤600px, reduce title/score sizes and panel padding.

## Screens

- **Start screen**: Title "NEON DRIFT" (88px gradient, white→cyan→teal), subtitle "Arena Survival", instruction grid (WASD, Shift, Touch, Avoid with cyan key badges), "Begin" button (cyan gradient, hover lift/shadow).
- **Game Over screen**: "Drift Complete" subtitle, final score (120px amber gradient), 3 stats (Orbs, Wave, Survived time), pulsing "Press R to Restart" button.
- Start: Space/click. Restart: R/click. Resets all state, clears enemies, respawns orbs, clears trail.

## Other Details

- Arena has low glowing coral walls (1.2m tall) and 4 corner pillars (cyan emissive, 6m tall with glowing caps).
- Player blinking during invulnerability (toggle visibility at 14Hz).
- Ground light follows player, intensity varies with speed.
- All Three.js objects: scene.add, scene.remove with proper cleanup.
- Window resize updates camera aspect, renderer size, particle pixel ratio.
- Use strict mode, IIFE, no global pollution.

## Technical Constraints

- Dispose geometries/materials when removing objects.
