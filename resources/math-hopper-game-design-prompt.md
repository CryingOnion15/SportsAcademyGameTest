# Math Hopper — Game Design Prompt File

> Generated from the `game-from-idea` skill framework.
> Paste this file as the starting context whenever you open a new Claude session to build this game.

---

## §0.7 — Game Shape Decision Note

| Field | Value |
|---|---|
| **Slug** | `math-hopper` |
| **One-line pitch** | A Frogger-style arcade game where your frog's numeric value must match the math puzzle at the finish line — collect power-ups, dodge traffic, ride logs, and solve problems before the 20-minute session timer runs out. |
| **Grade band** | K–5 (configurable per topic row) |
| **Math topic** | Arithmetic operations — addition, subtraction, multiplication, division (maps to `problems.topic` ingest rows) |
| **Math-as-verb** | The player IS a number. Every pickup transforms the frog's value (+, −, ×, ÷). Reaching a goal zone only scores if `frog_value === goal_answer`. The math is the movement decision — the player must plan their route through drops to build the right value before crossing. |
| **Core mechanic / verb** | Navigate a scrolling grid: dodge cars/trucks on road lanes, ride logs on river lanes, collect numeric modifier drops, reach a numbered end-zone goal that matches a displayed equation. |
| **Win / lose condition** | No single-match win state. The session runs for **20 minutes**. Points accumulate across all successful crossings. Being hit by a vehicle or falling in water resets the frog to start (no lives lost, no points deducted). Session ends at timer zero → results screen. |
| **Spine (rising tension)** | Session timer + diminishing point bonus. Early crossings in the session are slow; the player gets faster and greedier as the clock ticks down. |
| **Match length** | 20-minute session; individual crossing attempts are ~15–60 seconds each. |
| **Problem count** | Unlimited within the session; new equations generated per crossing attempt. |
| **Solo vs H2H** | Solo (primary). vs-score H2H future stretch. |
| **Stateful?** | Session-stateful: cumulative score persists across crossings within the 20-minute window. Each crossing attempt is stateless (reset on death). |
| **Modes** | `solo` (core), `flat` (diagnostic — no timer pressure, fixed batch of N crossings), `vs-score` (stretch). |
| **Feel / reference** | Frogger (1981) meets Math Blaster. Snappy grid movement, juicy pickup sounds, satisfying "clunk" when a goal zone accepts your value. Should feel arcade-crisp, not slow or educational. |

---

## Part 1 — Game Mechanics (full spec)

### The Frog / Player

- Occupies **one grid cell** at all times.
- Movement: **4-directional** (up, down, left, right), one cell per key/tap press. No diagonal.
- Has a **current numeric value** (the "Frog Value"), displayed prominently on the frog sprite and in the HUD.
- Starts each crossing attempt with a **base value** (e.g., `5`) — this resets every time the frog dies or successfully crosses.
- **Cannot carry a value between crossing attempts.** Each attempt starts fresh.

### The Level Layout (top-to-bottom, classic Frogger structure)

```
[ START ZONE      ]  — safe row at bottom; frog spawns here
[ ROAD LANES ×4   ]  — cars and trucks moving left/right
[ MEDIAN STRIP    ]  — safe row in the middle
[ RIVER LANES ×4  ]  — logs moving left/right; water is fatal without a log
[ GOAL ZONES ×5   ]  — numbered alcoves at the top, each showing a math equation
```

- The grid is **13 columns × ~14 rows** (classic Frogger proportions), cell size tunable.
- The level **scrolls with the frog** if needed, or fits in a single viewport at 1280×720.

### Road Lanes

- **Cars**: 1 cell wide. Move at varying speeds left or right.
- **Trucks**: 3–5 cells wide. Move slower than cars.
- Contact with any vehicle = **instant death** → frog resets to start, value resets.
- Lane directions alternate (left-moving, right-moving, etc.).
- Vehicle density and speed ramp up gently across the session as a soft difficulty curve.

### River Lanes

- Cells with **no log** are water — frog falls in = **instant death** → reset.
- **Logs**: 2–4 cells wide; frog rides on top and drifts with the log's velocity.
- If a log carries the frog off-screen = **instant death** → reset.
- Log density ensures the river is always passable but requires timing.

### Numeric Modifier Drops

- Drops appear on **road medians, log surfaces, and goal-zone approaches** as floating pickups.
- Types (each visually distinct):
  - **`+N`** — adds N to frog value (e.g., +3)
  - **`−N`** — subtracts N
  - **`×N`** — multiplies frog value
  - **`÷N`** — divides frog value (result always integer; N is chosen so division is clean)
- Collecting a drop is automatic on contact (no button press).
- Drops are **randomly placed** each crossing attempt; layout seeded for fairness.
- The set of drops available on a given attempt is **designed so at least one path exists** that reaches the correct answer for at least one goal zone.

### Goal Zones

- **5 alcoves** across the top row, each displaying a unique math equation (e.g., `3 + ? = 8`, `? × 4 = 20`).
- Each alcove has a **target answer** (the `?` value).
- The frog can only **enter an alcove** if `frog_value === alcove.targetAnswer`.
  - Attempting to enter with the wrong value: frog is **bounced back** one cell (no death, no penalty — try a different alcove or a different path).
  - Entering with the correct value: **crossing scored!**
- Each alcove's equation is **server-generated** via `fetchProblems()` (bridge-authoritative).
- After a successful crossing, all 5 alcoves refresh with new equations; frog resets to start.

### Scoring

| Event | Points |
|---|---|
| Successful crossing | Base **100 pts** minimum |
| Speed bonus | `+1 pt per second remaining` on a per-crossing countdown (60-second soft cap per attempt; if no countdown is shown to the player, track internally) |
| Maximum per crossing | ~160 pts (100 base + up to ~60 speed bonus) |
| Death / reset | No deduction |
| Session end (20 min) | Final score displayed; no bonus |

- Score is **cumulative** across all successful crossings in the session.
- The per-crossing speed bonus incentivises fast play without punishing slower players below 100.

### Session Timer & End Screen

- A **20:00 countdown** is always visible in the HUD.
- At `00:00`: gameplay freezes, **results overlay** appears.
  - Shows: **final score**, crossings completed, average time per crossing.
  - Two buttons: **Play Again** (resets everything, new 20-min session) | **Exit**.
- No partial crossing is counted at timer expiry.

### Difficulty Progression (within session)

- Minutes 0–5: lighter traffic, slower vehicles, generous log coverage.
- Minutes 5–12: more cars, faster speeds, narrower log windows.
- Minutes 12–20: trucks appear more frequently, logs move faster, drop layouts are denser (more operations needed to reach target).
- Math difficulty (equation complexity) scales in parallel: starts with simple addition/subtraction, introduces multiplication/division mid-session.

---

## Part 2 — Screens & States

### 1. Title / Start Screen
- Game logo "Math Hopper", tagline.
- **Play** button → solo session start.
- Optional: high score from previous session (local storage).

### 2. In-Session HUD (primary screen)
Layout anchored around the transparent game canvas:
- **Top-left**: Session timer `MM:SS` (large, always visible).
- **Top-right**: Cumulative score.
- **Frog value badge**: displayed ON the frog sprite (canvas layer) and mirrored in HUD corner.
- **Goal zone math**: rendered by **KaTeX** in DOM overlay above each alcove. 5 equation slots across the top.
- **Bottom strip**: current crossing attempt's soft timer (smaller; counts up from 0 for speed bonus transparency).

### 3. Correct Crossing Feedback
- Alcove flashes green, score popup `+NNN pts` animates up.
- Brief fanfare SFX.
- Frog animates into alcove, then teleports back to start for next attempt.
- Duration: ~1.5 seconds.

### 4. Wrong-value Bounce Feedback
- Alcove flashes red briefly, frog bounces back.
- Short "boing" SFX.
- No score change shown.

### 5. Death / Reset Feedback
- Frog squish/splash animation.
- Short "splat" or "splash" SFX.
- Frog reappears at start after ~0.8 seconds.

### 6. Session End / Results Screen
- Full overlay (not a new page).
- Shows: final score, number of crossings, best single crossing score.
- **Play Again** | **Exit** buttons.

---

## Part 3 — Art Direction Seeds

**Theme**: Bright, retro-arcade frog world. Not gritty, not minimalist — vibrant pixel-adjacent 2D with clean outlines.

**Palette seeds**:
- Lily-pad green (`#4CAF50`), warm amber road (`#F5A623`), deep river blue (`#1565C0`), creamy sand median (`#F5E6C8`), bright sky gradient top.
- Vehicle accent: fire-engine red cars, olive-drab trucks.
- Drops: glowing amber coins/orbs with operation symbol.
- Goal zones: soft gold alcoves with equation panel above.

**Style reference**: Frogger arcade original × Crossy Road (clean cel-shaded look) × bright educational game energy (think Math Blaster NES era).

**Character**: A friendly green frog with a number badge on its back. The badge dynamically shows the current value. Expressions change: neutral hop, celebrate (arms up), squish (death), splash (water death).

**UI feel**: Snappy, high-contrast. Correct = green burst. Wrong = quick red flash. No slow animations — response budget under 100 ms visual feedback from player input.

---

## Part 4 — Asset Manifest Outline

### Sprites (Ludo)
- `frog_idle` — neutral stand, 4 directional frames
- `frog_hop` — hop animation (25 frames @ eagle, 384px frame)
- `frog_celebrate` — arms-up win animation (25 frames)
- `frog_death_squish` — vehicle hit (25 frames)
- `frog_death_splash` — water hit (25 frames)
- `car_red` — 1-cell car, left/right facing
- `truck_olive` — 3-cell truck sprite (tileable)
- `log_short` — 2-cell log
- `log_long` — 4-cell log
- `drop_add` — +N pickup orb (idle + collect animation)
- `drop_sub` — −N pickup orb
- `drop_mul` — ×N pickup orb
- `drop_div` — ÷N pickup orb

### Backgrounds / Tiles (Ludo)
- `road_tile` — asphalt lane tile, seamless horizontal
- `median_tile` — grassy median strip
- `river_tile` — animated flowing water (loop)
- `start_zone_tile` — lily pad safe zone
- `goal_zone_tile` — golden alcove backdrop (5 variants or 1 reusable)
- `sky_bg` — top-of-screen sky gradient

### FX (Ludo)
- `fx_correct` — green starburst particle
- `fx_wrong` — red shake flash
- `fx_pickup` — sparkle on drop collect
- `fx_splash` — water ripple on death

### SFX / Music (Ludo)
- `sfx_hop` — frog hop sound
- `sfx_splash` — water death
- `sfx_squish` — vehicle death
- `sfx_pickup` — drop collect chime
- `sfx_correct` — crossing success fanfare
- `sfx_wrong` — wrong-value bounce
- `sfx_tick` — session timer warning (last 60 seconds)
- `music_main` — upbeat looping arcade track, ~90 BPM

---

## Part 5 — Claude Design Brief (paste into "Claude design")

````markdown
You are designing the UI/UX for a browser-based educational arcade game.
Assume you have ZERO access to any codebase or prior conversation — this
brief stands alone. Read all of it before designing.

Deliverable: a SINGLE self-contained HTML file with inline CSS (no build
step, no external deps except a Google Font link if you want one). Semantic
class names. A `data-testid` attribute on every interactive element.

## Part 1 — The game (so you understand what you're styling)
- One-line pitch: A Frogger-style arcade game where your frog's numeric value must match the math puzzle at the finish line — collect power-ups, dodge traffic, ride logs, and solve problems to score points before a 20-minute timer expires.
- Theme / setting / fantasy: Retro arcade frog world. Bright, vibrant, snappy — think classic Frogger × Crossy Road with educational-game energy. The player is a clever frog carrying a number on its back.
- Audience: grades K–5, drilling arithmetic (addition, subtraction, multiplication, division).
- Math-as-verb: The frog IS a number. Collectible drops modify the frog's value (+N, −N, ×N, ÷N). The player must plan their route to build a value that matches one of the 5 goal-zone equations before crossing. The math is the navigation decision.
- Core loop: Frog spawns at bottom with a base value (e.g., 5). Player navigates up through traffic lanes (dodge cars/trucks) and river lanes (hop logs). Numeric modifier drops appear on safe surfaces — collecting them changes the frog's value. At the top are 5 goal alcoves, each showing a math equation with a target answer. The frog can only enter an alcove whose target matches its current value. CORRECT = points + reset to start + new equations. WRONG value = bounced back (no death). Vehicle hit or water = death + reset to start, value lost.
- On CORRECT: alcove flashes green, score popup `+NNN pts` floats up, fanfare SFX, frog animates into alcove, teleports back to start. New equations appear in the 5 alcoves.
- On WRONG value attempt: alcove flashes red briefly, frog bounces back one cell, "boing" SFX.
- On death: squish/splash animation, frog reappears at start after ~0.8s.
- Match structure: 20-minute session. No per-attempt limit — player can try as many crossings as they can in 20 minutes. Timer hits 00:00 → results screen.
- Modes (the UI must accommodate all three):
  - solo — one player vs the clock.
  - vs-score (H2H) — 2–4 players race for highest score in the same 20 min; show live opponent leaderboard.
  - flat (diagnostic) — fixed batch of N crossing attempts, NO failure state, no timer pressure.
- The spine (what makes it tense): The session timer + a per-crossing speed bonus. Every second saved on a crossing = +1 bonus point (up to ~60 extra per crossing). Players get greedier and faster as the 20 minutes winds down.
- The feel: Arcade-crisp, snappy, high-contrast. Feedback must be instant (<100 ms visual response). Celebratory pops on correct. Quick red flash on wrong. Fun, not stressful.

## Part 2 — Screens & states to design (every one)
1. **Title / Start screen** — Game logo "Math Hopper", tagline, Play button, optional local high score display.
2. **Lobby / matchmaking (vs-score)** — Players joining, ready state, countdown to session start.
3. **In-match HUD** — The primary screen. Laid AROUND a transparent central gameplay viewport (leave the middle empty for the Phaser canvas). Must include:
   - **Session timer** `MM:SS` — top-left, large, always dominant. Turns red in final 60 seconds.
   - **Cumulative score** — top-right.
   - **Frog value display** — mirrored in HUD (the sprite also shows it, but the HUD shows it for clarity), near bottom-center.
   - **Goal zone equation panels** — 5 slots laid across the TOP edge of the viewport. Each slot shows one KaTeX-rendered equation (e.g., `3 + ? = 8`). These update after each successful crossing.
   - **Speed bonus tracker** — small, bottom strip — shows the current crossing's elapsed time for transparency.
   - **Opponent leaderboard** — right panel (vs-score mode only), hidden in solo.
4. **Correct-answer feedback state** — green flash on the matching alcove, animated `+NNN pts` score pop floating upward, brief full-screen green tint overlay lasting ~0.5s.
5. **Wrong-answer feedback state** — red flash on the alcove, quick screen shake, "WRONG VALUE" micro-text near the frog.
6. **Death / reset feedback state** — quick overlay darkening (~0.3s) then snap back as frog respawns.
7. **Session end / results screen** — full overlay (not a new page). Shows: final score (large, centered), crossings completed, best single-crossing score. Two buttons: **Play Again** | **Exit**.

## Part 3 — Visual design direction
- Style: Bright retro-arcade. Vibrant greens, ambers, blues. High contrast. Chunky rounded UI panels. Think "educational arcade cabinet" — not flat/minimal, not dark/gritty.
- Palette anchors: lily-pad green (#4CAF50), warm amber (#F5A623), river blue (#1565C0), creamy sand (#F5E6C8). Goal zones are gold-tinted. Vehicles: red cars, olive trucks. Drops glow amber with operation symbols.
- Typography: a friendly rounded display font for headings (Google Fonts — suggest "Fredoka One" or "Nunito ExtraBold"), monospace or tabular figures for the timer and score.
- The score and timer must be readable at a glance from a distance. Prioritize legibility.
- KaTeX equations in the goal zone panels must have enough room for 1–2 lines without reflowing the layout.
- Juice: green starburst particles on correct, red vignette flash on wrong, sparkle on pickup. Correct scoring number should animate (scale up, fade out upward).
- The session timer in the final 60 seconds should visually escalate: pulse animation, color shift to red/orange.

## Part 4 — Hard technical constraints
- Base canvas 1280×720, scaled to fit; design at that ratio.
- The HUD OVERLAYS a game canvas — the center must be transparent / empty so the Phaser canvas art shows through. Anchor UI to edges/corners and the very top strip (for goal equations).
- KaTeX renders the equation stems; give each of the 5 goal-zone containers enough width for short equations without overflow.
- `data-testid` on every interactive element: `btn-play`, `btn-play-again`, `btn-exit`, `input-answer` (if typed input exists), `goal-zone-0` through `goal-zone-4`, `session-timer`, `score-display`, `frog-value-display`.
- Support both tap-able and typed answer modes (the frog navigates into an alcove physically — but a typed-answer panel is needed for `flat` diagnostic mode where physical navigation is bypassed).

## Part 5 — Out of scope (do NOT do this)
- Do NOT design or draw sprites, characters, backgrounds, particles, or any in-canvas art. Those are generated separately. Design ONLY the DOM chrome that frames and overlays the canvas.
````

---

## Part 6 — Bridge & Server Integration Notes

- **Problem fetching**: `fetchProblems({ topic, grade })` returns equation objects with `stem`, `answer`, `distractors`. The 5 goal-zone equations are drawn from this queue each crossing.
- **Grading**: When the frog enters a goal zone, call `submitAttempt({ problemId, answer: frog_value })`. Apply points **only from the returned verdict** — never compare client-side. The server returns `{ correct: bool, score: number }`.
- **Flat mode**: `resetForFlatPhase()` on session start; `onBatchDone()` when N crossings complete. No session timer — remove it from HUD.
- **vs-score mode**: `createMatchController`; show live `sortLeaderboardEntries()` in the right panel. Each player's frog value and position are broadcast; render opponent frogs as ghost sprites on the canvas.

---

## Part 7 — Open Questions (resolve before Phase 3)

1. **Grade selector**: Should the title screen let the player choose grade/topic, or is it operator-configured?
2. **Base frog value**: Always `5`? Or randomized per attempt? Randomized adds variety; fixed is more learnable.
3. **Drop density**: How many drops max per crossing attempt? Recommend 4–6 to keep the path planning tractable.
4. **vs-score**: Is this in scope for the initial build, or `flat` + `solo` only?
5. **Leaderboard persistence**: Local session only, or submitted to the platform leaderboard on session end?
6. **Mobile / touch**: Is touch input required for the initial build? (affects grid cell size and movement controls.)
