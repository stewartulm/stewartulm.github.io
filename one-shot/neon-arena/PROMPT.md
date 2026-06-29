Create a complete single-file HTML page implementing a 3D arena survival game called "NEON DRIFT — Arena" using Three.js (loaded from CDN v0.150.1). The game features neon cyberpunk aesthetics, third-person camera, momentum-based movement, orb collection, and enemy pursuit.

## Core Gameplay
- **Arena**: A square arena (half-size 40 units) bounded by glowing coral walls. Ground is a custom shader with multi-scale grid lines (2m and 8m spacing), distance fade toward edges, pulse rings emanating from the player's position, wave-dependent color shift (cold cyan → warm amber), coral edge glow, and subtle scanlines.
- **Player**: A glowing cyan icosahedron with a white core, a transparent outer shell (emissive, metalness 0.4), a rotating ring torus, and a pulsing point light. Movement is camera-relative (WASD/arrows), momentum-based with acceleration 60 and friction 4.5. Max speed 22 (37.4 when boosting). Clamped to arena with bounce-back. Smooth turning toward velocity direction with visual tilt.
- **Boost**: Shift key drains a 0→1 meter (0.5/s drain, 0.25/s recharge). 1.7x speed, brighter glow, cyan exhaust particles from behind.
- **Trail**: A ribbon of 40 line segments (additive blending, vertex colors) following the player. Width narrows toward tail, colors fade cyan→transparent. Brighter during boost.
- **Orbs**: 4 active at a time. Amber icosahedron with white core, halo sphere (additive blending), point light. Bob up/down, rotate, pulse. Spawned randomly in arena (avoiding player). Collect at distance < 1.4: combo increments (max 8), score += 10×combo, particle burst, expanding ring, popup text, camera kick, immediate respawn.
- **Enemies**: Crimson octahedron body (dark with coral emissive, metalness 0.6), inner glowing pink sphere, 6 cone spikes radially arranged, coral point light. Spawn at arena edges. Home toward player at speed = 5 + wave×0.7 + random 0–1.2. Body rotates, inner rotates opposite, face player. Spawn animation: scale grows 0→1 in 0.33s, emissive pulses. Collision at distance < 1.4 (not invulnerable, spawn animation > 0.4s): lose life, knockback, combo reset, particle burst, ring effect, damage flash, camera shake, enemy destroyed. 0 lives → massive explosion → game over.
- **Enemy spawning**: Timer-based, interval = max(0.45, 2.6 - wave×0.25) seconds, randomized ±30%. Cap at 18 + wave×2 enemies.
- **Waves**: Increment every 20 seconds. On wave change: particle burst + ring at player. Wave affects enemy speed, spawn rate, ground color.
- **Particles**: Pool of 600. Custom ShaderMaterial with soft-circle fragment. Gravity, ground bouncing, color fade over life. Used for orb collect, hits, death, boost exhaust, enemy trail.
- **Expanding rings**: RingGeometry (inner 0.5, outer 0.7), additive blending, scale 1→9 over 0.8s, opacity fades. Used on orb collect and hits.

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
- Single HTML file, no external files (except Three.js CDN and Google Fonts).
- All CSS in a `<style>` block, all JS in an inline `<script>` after Three.js.
- Use ES5-compatible syntax (no modules, no import).
- Ensure all Three.js constants are accessed via `window.THREE`.
- Use `(() => { 'use strict'; ... })()` pattern.
- Dispose geometries/materials when removing objects.

Please generate the complete HTML file with all the above features.