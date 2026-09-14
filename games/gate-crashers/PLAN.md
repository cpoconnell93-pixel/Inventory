# Gate Crashers — build plan

A playable, single-file HTML5 clone of the "run your squad through +N / ×N gates and
fight the horde" lane-runner genre (the Top War / Count Masters ad loop), tuned to be
genuinely challenging. Runs inside the Claude mobile app as an Artifact, portrait,
touch-first, no external libraries.

## Roles

| Stage | Model | Job |
|---|---|---|
| 1. Build | Opus | Design the architecture and write the complete game in `index.html`. |
| 2. Playtest & tune | Sonnet | Drive the game headlessly with Playwright, log bugs, tune difficulty and feel, fix. |
| 3. Ship | Lead | Publish as an Artifact, commit, push. |

## Core loop (what the ad shows)

1. Your **blue squad** (a cluster of small soldiers) auto-runs up a long road.
2. **Drag left/right** to steer. The road has a left half and a right half.
3. Every few seconds a **gate pair** spans the road: one gate per half. Each gate is
   either a **buff** (`+N`, `×N`) or a **debuff** (`−N`, `÷N`). You pass through
   exactly one. Gate values are shown as big numbers.
4. **Enemy hordes** (red clusters) sit in the road. Your soldiers auto-fire tracer
   bullets forward and kill enemies at range. Anything that reaches you fights melee:
   soldiers and enemies trade 1-for-1.
5. Every level ends with a **boss**: a big red brute with a large HP number. He is
   beaten by the number of soldiers you bring. Not enough soldiers = you lose.
6. Win the boss → next level, faster and meaner. Lose → restart the level.

## Difficulty spec ("decently challenging")

- The **optimal-path** count going into the boss should sit roughly 15–30% above the
  boss HP. A player who takes ~2 wrong or lazy gate choices per level should lose.
- Gate pairs are **not** always one-good / one-bad. Mix in: good vs better
  (`+20` vs `×2` — which is better depends on your current count), bad vs worse,
  and gates that are **sandwiched with an enemy horde** in front of the better one.
- Hordes scale with the level. Bullets have a fire rate proportional to squad size
  (bigger squad = more DPS), but hordes also chip at you on contact, so the
  "run into everything" strategy fails.
- Squad cap so a runaway `×3` chain can't trivialise things (cap ~ 400, shown to the
  player as "MAX").
- Boss HP per level roughly: 30, 60, 110, 180, 280, 400, then +40% per level.
- Movement: steering is a lerp with a small delay so last-second gate switches are
  risky — no instant teleporting between halves.

## Presentation

- Portrait canvas that fills the viewport (`100dvh`), `devicePixelRatio`-aware.
- Pseudo-3D road: perspective projection (objects farther up the road are smaller and
  converge). No real 3D. Road is a grey-blue slab with gate rails on the sides,
  sky/ground gradient beyond.
- Soldiers are small capsule sprites drawn on canvas (head + body), packed in a
  circle formation whose radius grows with sqrt(count). Draw at most ~150 sprites and
  scale the rest into the count label.
- Big count label floats above the squad; enemy hordes and bosses show their number.
- Gate colours: buff = blue, debuff = red/orange, value text in heavy white.
- Tracer bullets: yellow streaks, muzzle flash puffs on hit.
- HUD: level, current squad count, best level. Start screen, game-over screen with
  "Retry level", level-clear screen with "Next level". All buttons ≥ 48px tall.
- Single deliberate visual world (arcade-ish, saturated primaries), single-theme, but
  every colour painted explicitly.
- Respect `prefers-reduced-motion` by dropping screen shake / particle counts.

## Controls

- Pointer events: drag anywhere on the canvas moves the squad horizontally (relative
  drag, not absolute tap-to-position). Tap to start / retry.
- Keyboard fallback: ← → / A D.
- Prevent page scroll and pinch on the canvas (`touch-action: none`).

## Technical constraints

- One file: `games/gate-crashers/index.html`. Vanilla JS, Canvas 2D, no CDN deps.
- The Artifact host wraps the file in its own document skeleton, so the file must NOT
  contain `<!DOCTYPE>`, `<html>`, `<head>`, or `<body>` tags — start with `<title>`,
  then `<style>`, then the markup, then `<script>`.
- Must also work as a plain local file opened in a browser (for Playwright tests).
- `requestAnimationFrame` loop with delta-time clamped to 50 ms.
- Persist best level in `localStorage` inside try/catch.
- Expose `window.__game` with `{ state, level, count, setInputX(x), tick(dt) }` so the
  playtest agent can drive it headlessly. Keep this as a small debug surface.
- Register `window.claude?.hot?.snapshot` / `ready` boot pattern if present, so an
  artifact republish keeps the current run.

## Acceptance checklist

- [ ] Loads with zero console errors; runs at 60 fps on a mid phone (no per-frame allocations in hot paths).
- [ ] Level 1 is winnable by a competent player; a random-gate bot loses level 1 most of the time.
- [ ] Levels 3+ require thinking about `+N` vs `×N` relative to current count.
- [ ] Touch drag steers; no accidental scroll; works at 390×844 and 360×780.
- [ ] Boss fight resolves clearly; win and lose screens are obvious.
