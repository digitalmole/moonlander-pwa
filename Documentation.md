# Design and purpose of the game

This game was coded with Mistal Vibe and a Gemma-4-26b-a4b-nvfp4 model on a 32GB RAM machine using MLX.

Claude chat was asked to write a prompt to write a HTML PWA moonlander game. That prompt was then used to generate the first draft. I tried several AI models in nvfp4. Gemma-4-26b-a4b gave the best first result from the claude prompt. I tried Qwen3.6 27b, Qwen3.6-35b-a3b, Qwen3-coder-30b-a3b and Gemma-4-31b. They all generated html that gave black screens without anything working.

Gemma-4 did quite wel. The game was playable though sound and more details were missing. Corrective prompts were given to add missing implementations details and fix bugs. Next additional features were asked. At no point directive code change instructions were given. Only more general instructions like 'stop thruster sound when landed' too see how far the llm would bring me without getting involved as a programmer. 

The Claude prompt that started this was:

You are an expert game developer. Generate a single self-contained HTML file (index.html)
that implements a complete Moonlander PWA game. Do not split into multiple files.
Do not use external libraries or CDN links. Everything — CSS, JavaScript, assets (drawn
via Canvas API), PWA manifest, and service worker (inline via Blob URL) — must live
in one HTML file.

---

## TARGET PLATFORMS
- iPadOS (Safari, touch-first, pointer events)
- macOS (Safari/Chrome, keyboard + mouse)
- Progressive Web App: installable, works offline, fullscreen

---

## PWA REQUIREMENTS

Include a <meta name="apple-mobile-web-app-capable" content="yes"> and all required
apple-touch meta tags. Embed the Web App Manifest as a JSON blob URL in a <link> tag.

Manifest fields:
  name: "Moonlander"
  short_name: "Moonlander"
  display: "fullscreen"
  orientation: "landscape"
  background_color: "#000010"
  theme_color: "#000010"
  A single generated icon (192x192 and 512x512) drawn on an OffscreenCanvas:
  black background, a white pixel-art lander shape, crescent moon, star dots.

Register a Service Worker via a Blob URL. The SW must:
  - Cache the page itself on install (cache name: "moonlander-v1")
  - Serve from cache on fetch (cache-first strategy)
  - This makes the app fully offline-capable after first load

---

## VISUAL STYLE — "RETRO NASA TERMINAL"

Aesthetic: monochrome phosphor green on deep space black, with amber accent highlights.
  - Background: #000010 (near-black deep blue)
  - Primary color (terrain, UI text, lander): #00FF88 (phosphor green)
  - Thrust flame: amber/orange gradient (#FF8800 → #FFFF00)
  - Danger/explosion: red (#FF2200)
  - Landing pad: bright cyan (#00FFFF) with blinking lights
  - HUD font: monospace, uppercase, letter-spacing 0.1em
  - Stars: randomly placed white/dim dots with subtle twinkle animation (CSS keyframes
    or canvas alpha oscillation)
  - CRT scanline overlay: a semi-transparent repeating-linear-gradient of
    rgba(0,0,0,0.08) stripes every 3px layered on top of the canvas via a
    CSS ::after pseudo-element on a wrapper div
  - A subtle vignette (radial gradient from transparent center to rgba(0,0,0,0.5)
    edges) on the overlay

---

## GAME CANVAS SETUP

Use a single <canvas id="game"> that fills 100vw × 100vh.
On resize (and on init), set canvas.width = window.innerWidth,
canvas.height = window.innerHeight.
Maintain a VIRTUAL coordinate space of 1600 × 900 units. Scale all game coordinates
by (canvas.width / 1600) horizontally and (canvas.height / 900) vertically before
drawing. This ensures the game looks correct on any screen ratio.
Use requestAnimationFrame for the game loop. Pass a delta time (dt in seconds, capped
at 0.05) to all update functions.

---

## GAME PHYSICS ENGINE

Constants (all in virtual units per second):
  GRAVITY = 80          // downward acceleration, vu/s²
  THRUST_FORCE = 180    // upward acceleration when thrusting, vu/s²
  ROTATE_SPEED = 120    // degrees per second when rotating
  MAX_SAFE_VERT_SPEED = 40   // max vertical speed at landing (vu/s)
  MAX_SAFE_HORIZ_SPEED = 30  // max horizontal speed at landing (vu/s)
  MAX_SAFE_ANGLE = 12        // max tilt at landing (degrees from vertical)
  FUEL_BURN_RATE = 25        // fuel units per second of thrust

Lander state object:
  x, y          // position (center of lander), virtual units
  vx, vy        // velocity, virtual units per second
  angle         // degrees, 0 = upright, positive = clockwise
  fuel          // starts at 600 units
  thrusting     // boolean
  rotating      // "left" | "right" | null
  alive         // boolean
  landed        // boolean

Physics update (called each frame with dt):
  1. If thrusting and fuel > 0:
       ax = -sin(angle_rad) * THRUST_FORCE
       ay = -cos(angle_rad) * THRUST_FORCE   // negative = upward in screen space
       vx += ax * dt
       vy += ay * dt
       fuel -= FUEL_BURN_RATE * dt
  2. Apply gravity: vy += GRAVITY * dt
  3. If rotating left: angle -= ROTATE_SPEED * dt
     If rotating right: angle += ROTATE_SPEED * dt
  4. Integrate position: x += vx * dt, y += vy * dt
  5. Clamp x to [20, 1580] (screen wrap not used; bounce off walls with vx *= -0.3)
  6. If y < 0: vy = Math.abs(vy) * 0.5 (soft ceiling bounce)

---

## TERRAIN GENERATION

Generate a random terrain per level using a midpoint displacement algorithm:
  - Start with points at (0, 700) and (1600, 700)
  - Recursively subdivide with random vertical displacement ±(segment_length * roughness)
  - roughness = 0.55 (adjustable per level, 0.45–0.7)
  - Min terrain height: 600 (y value), max: 820
  - After generation, smooth the array with one pass of a 3-point moving average
  - Place 1–3 landing pads: flat segments, width 80 + (3 - level) * 20 virtual units
    (narrower pads on higher levels). Each pad is worth different points (see scoring).
  - Store terrain as an array of {x, y} points spaced ~8 virtual units apart.

Pad scoring multiplier by width:
  Width > 100 → ×1  (easy, 100 pts base)
  Width 60–100 → ×2  (200 pts base)
  Width < 60 → ×3  (300 pts base)

Collision detection:
  For each pair of adjacent terrain points, check if the lander's bounding circle
  (radius 14 virtual units) intersects the line segment.
  If intersection found:
    - If the segment is a landing pad AND lander.alive:
        Check landing conditions (speed, angle).
        If safe → LANDED. If not → CRASHED.
    - Else → CRASHED.

---

## LANDER DRAWING

Draw the lander on the canvas using ctx.save() / ctx.restore() and
ctx.translate(sx, sy) / ctx.rotate(angle_rad) where sx,sy are scaled screen coords.

Lander shape (all coords relative to center, in virtual units, then scaled):
  Body: a downward-pointing triangle with rounded corners
    Points: (-12, -10), (12, -10), (0, 14)
    Fill: none. Stroke: #00FF88, lineWidth 2 (scaled)
  Legs: two lines from bottom corners outward/downward
    Left: from (-10, 10) to (-18, 20)
    Right: from (10, 10) to (18, 20)
    Stroke: #00FF88, lineWidth 1.5 (scaled)
  Foot pads: small horizontal lines at leg ends, width 8
  Cockpit window: small filled circle at (0, -2), radius 4, fill #003322
  Engine nozzle: small trapezoid at the bottom center,
    points: (-5, 12), (5, 12), (3, 16), (-3, 16), fill #005544

Thrust flame (draw only when thrusting and fuel > 0):
  Draw a flickering triangle below the nozzle:
    Base: (-4+rand*2, 16) to (4+rand*2, 16)
    Tip: (rand*3, 16 + 14 + rand*10)  // random flicker
    Use a canvas gradient from #FF8800 (top) to rgba(255,255,0,0) (tip)
    Redraw each frame for flicker effect

---

## TERRAIN DRAWING

Draw terrain as a filled polygon:
  ctx.beginPath()
  ctx.moveTo(0, canvas.height)
  For each terrain point: ctx.lineTo(scaledX, scaledY)
  ctx.lineTo(canvas.width, canvas.height)
  ctx.closePath()
  ctx.fillStyle = "#001A0A"  // very dark green fill
  ctx.fill()
  ctx.strokeStyle = "#00FF88"
  ctx.lineWidth = 2
  ctx.stroke()  // only the top surface line

For landing pads:
  Draw a bright cyan (#00FFFF) flat line on top of the pad segment, lineWidth 3.
  Draw 3 evenly spaced "lights": small circles radius 3, fill #00FFFF.
  Animate lights: blink at 1Hz using Date.now() % 1000 < 500 toggle.

---

## HUD (Heads-Up Display)

Draw HUD elements directly on the canvas (not DOM) in virtual space, scaled to screen.
Font: "16px monospace" (scale font size by canvas.height / 900).

Top-left panel (semi-transparent black bg, rounded rect):
  FUEL: [bar graph, 120px wide] [NNN]
  VERT: [±NN.N m/s] — red if > MAX_SAFE_VERT_SPEED
  HORIZ: [±NN.N m/s] — red if > MAX_SAFE_HORIZ_SPEED
  ALT: [NNN m]  // y distance from lander to terrain below

Top-right:
  SCORE: NNNNNN
  LEVEL: N
  LIVES: ★★★ (filled stars = remaining lives, max 3)

Fuel bar:
  Outline rect, filled portion #00FF88, turns amber at 30%, red at 15%.
  Animate a subtle pulse when fuel < 15%.

Altitude: raycast straight down from lander center to nearest terrain point
  (linear interpolation between terrain array points).

---

## PARTICLE SYSTEM

Implement a simple particle array. Each particle: {x, y, vx, vy, life, maxLife, color, size}

Thrust particles (emitted when thrusting, 4 per frame):
  Spawn at engine nozzle (rotated correctly), velocity: rotated downward ±spread,
  speed 60–120 vu/s, life 0.3–0.6s, color cycling #FF8800→#FFFF00→transparent, size 2–4.

Explosion particles (emitted on crash, 80 particles):
  Radial burst, speed 80–300 vu/s in all directions, life 0.8–2.0s,
  colors: mix of #FF2200, #FF8800, #FFFF00, #00FF88, size 2–6.
  Add gravity to explosion particles (vy += GRAVITY * 0.3 * dt).

Landing sparkles (on successful land, 20 particles):
  Upward burst from pad, speed 40–100 vu/s, color #00FFFF, life 0.4–0.8s.

Draw particles as filled circles, alpha = life/maxLife.

---

## GAME STATES

Implement a state machine with these states:

  "TITLE"
    Full-screen title screen drawn on canvas.
    Large text: "MOONLANDER" in #00FF88, with a CRT flicker animation.
    Subtitle: "MISTRAL VIBE EDITION" smaller, #00FFFF.
    Blinking prompt: "TAP TO BEGIN" or "PRESS SPACE" depending on input detected.
    Show a small animated demo lander drifting slowly across screen with random
    gentle thrusts (just visual, not physics-driven — simple sine wave path).
    High score displayed if > 0.

  "PLAYING"
    Normal gameplay loop.

  "PAUSED"
    Overlay: semi-transparent black rect over canvas.
    Text: "PAUSED — TAP/PRESS P TO RESUME"
    Show current score and fuel.

  "LANDED"
    Freeze lander. Play sparkle particles.
    Overlay message box:
      "LANDING SUCCESSFUL"
      "PAD BONUS: ×N"
      "FUEL BONUS: +NNN"
      "TOTAL: NNNNNN"
    After 2.5s or tap: load next level, keep score, add fuel bonus,
    generate new terrain.

  "CRASHED"
    Play explosion. Lose a life.
    Overlay: "CRASHED" in red, pulsing.
    After 2.0s or tap: if lives > 0, restart same level (new lander, same terrain,
    same score). If lives = 0 → GAMEOVER.

  "GAMEOVER"
    Full canvas overlay.
    "GAME OVER" large red text.
    Final score, high score (localStorage), level reached.
    "TAP TO RESTART" prompt.
    Save high score to localStorage key "moonlander_hiscore".

---

## SCORING

Base landing score: 100 × pad_multiplier
Fuel bonus: Math.floor(lander.fuel) × 2 points
Level bonus: level × 50
Perfect landing bonus (angle < 3°, speed < 10 vu/s in both axes): +500 pts

Score multiplier increases with level (×1 at level 1, ×1.5 at level 3, ×2 at level 5+).

Display score change as floating "+NNN" text rising from the pad on landing,
fading out over 1.5s (implement as a simple timed text object in an array).

---

## LEVEL PROGRESSION

Level 1: fuel=600, gravity=80, 3 pads, roughness=0.45, wide pads
Level 2: fuel=550, gravity=85, 2 pads, roughness=0.50
Level 3: fuel=500, gravity=90, 2 pads, roughness=0.55, narrower pads
Level 4+: fuel=480, gravity=95, 1–2 pads, roughness=0.60–0.70, narrow pads

Cap level at 10 for difficulty. After level 10, repeat level 10 params with new terrain.

Lander spawns at x=800, y=80 (virtual), vx=random ±20, vy=0, angle=0.

---

## KEYBOARD CONTROLS (macOS + iPadOS with keyboard)

  UP / W / Space → Thrust
  LEFT / A      → Rotate left (counter-clockwise)
  RIGHT / D     → Rotate right (clockwise)
  P / Escape    → Pause / Resume
  R             → Restart (from GAMEOVER or CRASHED state only)
  Enter         → Confirm / advance from TITLE, LANDED, GAMEOVER screens

Use keydown/keyup events. Store key state in a Set<string>.
Check set each physics frame (not in the keydown handler directly) for smooth input.

---

## TOUCH CONTROLS (iPadOS primary, also works on macOS trackpad)

Do not use on-screen buttons unless specified below.
Use a split-screen touch zone approach:

  Left third of screen  → Rotate LEFT (touch held)
  Right third of screen → Rotate RIGHT (touch held)
  Center third of screen → THRUST (touch held)

Draw subtle semi-transparent touch zone indicators at bottom of screen
(only when no keyboard has been detected):
  Three pill shapes, labeled with icons:
    "◄" (left), "▲" (center/thrust), "►" (right)
  Style: rgba(0,255,136,0.08) fill, rgba(0,255,136,0.3) border,
  Height: 80px from bottom, each zone full height of that region.
  Fade these out during active gameplay if no touch events in last 5 seconds
  (they remain but at 30% opacity), restore on touch.

Support multi-touch: track up to 3 simultaneous touches.
A single touch can cover multiple zones if it spans them (use touch.clientX
to determine zone, re-evaluate on touchmove).

Use touchstart, touchend, touchmove, touchcancel events.
On TITLE/GAMEOVER/LANDED/CRASHED screens: any tap advances state.

Keyboard detection: set a flag hasKeyboard=true on first keydown event.
If hasKeyboard, hide touch zone indicators entirely.

---

## AUDIO (Web Audio API — no external files)

Generate all sounds procedurally using AudioContext + OscillatorNode.

  Thrust sound:
    Low rumble: OscillatorNode type="sawtooth", freq=55Hz,
    gain ramped to 0.15 on thrust start, ramped to 0 on stop.
    Add a second oscillator at 80Hz with slight detuning for texture.
    GainNode connected to destination.

  Explosion sound:
    White noise buffer (create 1s buffer, fill with Math.random()*2-1),
    connect through BiquadFilterNode (type="lowpass", freq=400),
    gain envelope: attack 0ms, sustain 0.4s at 0.5, release 0.6s.

  Landing success sound:
    Three ascending tones in sequence (C4→E4→G4), each 0.12s,
    OscillatorNode type="sine", gain 0.2, with short release.

  UI click sound (on any state transition):
    Short 880Hz sine, 0.05s, gain 0.1.

  Low fuel warning (when fuel < 15%):
    Repeating beep: 1200Hz, 0.05s on / 0.45s off cycle, while condition true.
    Stop immediately when fuel >= 15% or thrusting stops.

Create AudioContext on first user interaction (touchstart or keydown) to comply
with browser autoplay policies. Resume context if suspended.

---

## PERFORMANCE REQUIREMENTS

- Target 60fps on iPad Air (A14) and MacBook Air (M1).
- Reuse particle arrays (pool pattern): pre-allocate 200 particle slots,
  reset and reuse dead particles rather than garbage-collecting.
- Terrain point array should have ~200 points max (space ~8 virtual units apart).
- Do not create new objects in the hot game loop path where avoidable.
- Clear only dirty regions if performance allows; otherwise full canvas clear is fine.

---

## COMPLETE CODE STRUCTURE

Organize the JavaScript inside a single <script> tag as follows (comments required):

  // === CONFIG ===
  // === STATE ===
  // === AUDIO ===
  // === TERRAIN ===
  // === PARTICLES ===
  // === LANDER ===
  // === INPUT ===
  // === DRAWING ===
  //   drawStars(), drawTerrain(), drawLander(), drawParticles(), drawHUD(),
  //   drawTouchZones(), drawOverlay()
  // === PHYSICS ===
  // === COLLISION ===
  // === GAME LOOP ===
  // === STATE MACHINE ===
  // === INIT ===
  // === PWA ===

---

## FINAL OUTPUT REQUIREMENTS

- Single index.html file, no external dependencies whatsoever.
- Valid HTML5 document with <!DOCTYPE html>.
- Mobile viewport meta: width=device-width, initial-scale=1, viewport-fit=cover.
- No framework (no React, no Vue, no Phaser). Vanilla JS only.
- The game must be fully playable without any server — openable directly from
  the filesystem (file:// protocol) or served from any static host.
- Code must be clean, well-commented, and complete. Do not use placeholder
  comments like "// add explosion here". Implement everything.
- Do not truncate the output. Emit the complete file from <!DOCTYPE html>
  to </html> without interruption.
