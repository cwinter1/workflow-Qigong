# CLAUDE.md — Q·Flow (Qigong PWA)

## Purpose & Identity

**Q·Flow** is a personal morning Qigong practice app for Chris, built as a sibling to the **Morning Flow / B·Restore** strength-and-mobility app (`cwinter1/workout`). Where B·Restore undoes what desk work does to the body through strength, posture work, and yoga, Q·Flow is a quieter counterpart built around **Baduanjin (Eight Pieces of Brocade)** — the standardized, traditional 8-movement qigong set. The app went through one earlier iteration built around a general, non-standardized flowing-qigong routine (5 broad phases per session); it was replaced with the actual Baduanjin set once it became clear that's what was wanted.

The app is a 4-week program: 20 minutes, 3 mornings a week. The app title is in Hebrew: **Q·Flow · כריס** (כריס = Chris). Layout is `lang="he" dir="rtl"`. Same audience and constraints as B·Restore: iOS Safari, one person, Israel.

**Design system**: intentionally identical to B·Restore Dark — same palette, same four fonts, same `el()`-based render architecture, same RTL/Hebrew shell. See `T` object below; do not deviate from it.

**Tone**: dry delivery, genuine warmth, no hype — same voice as B·Restore, adapted for a breath practice rather than a training log.

---

## Relationship to `cwinter1/workout`

This is a **separate app**, not a mode inside the workout app. It intentionally shares:
- The `T` palette, fonts, and `el()`/render-dispatch architecture
- The session lifecycle shape (preview → timer → meditation → done)
- Progress/streak tracking and message-pool mechanics

It intentionally does **not** carry over:
- Garmin sleep/activity tracking, body measurements, progress photos, Canvas share cards, Google Sheets sync, or the `puter.ai` voice-coaching integration — none of those map onto a breathwork practice and weren't requested
- Video as the *primary* movement reference — ads and intros add real friction to a practice built around the same 8 movements every session, so the built-in stick-figure diagram (`EX_INFO[name].illo()`) is what shows by default. A small "Not clear? Watch on YouTube" text link (`youtubeSearchUrl(name)`) sits underneath the diagram as a fallback, deliberately understated so it doesn't compete with the instant, ad-free default.
- The box-breathing meditation screen (`renderMeditation`/`getMedState`) from the earlier iteration — removed when the program switched to Baduanjin, since breath is threaded through each of the 8 movements individually rather than bolted on as a separate closing exercise. Don't re-add it without re-introducing a matching `kind:'meditation'` timeline item.

**Important — shared origin, separate storage.** Both apps are GitHub Pages project sites under `cwinter1.github.io` (different paths, same origin), so `localStorage` is shared between them. Q·Flow uses the `qf.*` key prefix (`qf.progress`, `qf.sessions`) specifically to avoid colliding with the workout app's `mf.*` keys. Never rename these to `mf.*` or anything that could collide.

---

## Non-Negotiable Constraints

- Single `index.html` — no split files, no framework, no build step
- iOS Safari only
- `localStorage` only — no backend, no sync, no account
- No emojis anywhere
- No hardcoded/unverified video IDs — text-only movement guidance
- Never push directly to main

---

## Design System — B·Restore Dark (reused exactly)

```javascript
const T = {
  bg:       '#0e1116',
  fg:       '#e8e5dd',
  sub:      '#8a8478',
  card:     '#16191e',
  cardLine: 'rgba(255,255,255,0.06)',
  accent:   '#7aa676',
  accentT:  '#0e1116',
  hairline: 'rgba(255,255,255,0.12)',
  mono:     'rgba(232,229,221,0.65)',
  pill:     'rgba(255,255,255,0.06)',
  display:  '"Space Grotesk", system-ui, sans-serif',
  body:     '"Manrope", system-ui, sans-serif',
  mono_ff:  '"JetBrains Mono", ui-monospace, monospace',
};
```

Fonts (Google Fonts, same four as B·Restore): Space Grotesk (display), Manrope (body), JetBrains Mono (labels/chips), Instrument Serif italic (encouragement quotes).

Do not introduce colors outside the `T` object or fonts beyond these four.

---

## Architecture

Single `<div id="root">` rendered entirely by JS. `render()` clears `root.innerHTML` and dispatches on `state.view`:

```
render()
  → renderHome()       — home screen
  → renderPreview()    — movement preview (stick-figure diagram + description + intent) before each item
  → renderSession()    — active timer + movement display
  → renderDone()       — post-practice completion screen
  → renderProgram()    — program overview (the 8 movements, shown once) + 4-week progress grid
```

No `renderMeasurements()` / no `renderRest()` / no `renderMeditation()` — this app has no body-measurement tracking, no inter-exercise rest periods, and no separate breathing-meditation screen (see "Relationship to `cwinter1/workout`" above for why the meditation screen was removed).

---

## State

```javascript
let state = {
  view: 'home',
  week: 1, day: 0,       // day 0/1/2 → PROGRAM.days[0/1/2]
  timeline: [], idx: 0, left: 0, paused: false,
  progress: {},          // { 'w1d0': { done: true, at: ts } }
  lastDuration: 0,
  startedAt: null,
};
```

`state.progress` persists to `qf.progress`. `nextSession()` scans it and auto-advances to the first incomplete session; defaults to W4D3 once all 12 are done.

---

## Program Data

`BADUANJIN_PHASES` — the 8 traditional pieces, each its own phase (`b1`..`b8`), each phase containing exactly one exercise (the movement itself):

| # | Phase name | Movement | Benefit (traditional framing) |
|---|------------|----------|--------------------------------|
| 1 | First | Two Hands Hold Up the Heavens | Regulates the Triple Burner; lengthens the spine |
| 2 | Second | Drawing the Bow to Shoot the Hawk | Opens the chest; strengthens arms/shoulders/lungs |
| 3 | Third | Separate Heaven and Earth | Regulates the spleen and stomach |
| 4 | Fourth | The Wise Owl Gazes Backward | Relieves general strain and fatigue |
| 5 | Fifth | Sway the Head and Shake the Tail | Clears heat/tension from chest and head |
| 6 | Sixth | Two Hands Hold the Feet | Strengthens the kidneys and low back |
| 7 | Seventh | Clench the Fists and Glare Fiercely | Builds strength and qi |
| 8 | Eighth | Bouncing on Toes and Heels | Shakes the body loose; closes the set |

`PROGRAM.days[0/1/2]` are **identical** — all three point at `BADUANJIN_PHASES` with the same title/kicker/tag (`BADUANJIN`). This is deliberate: Baduanjin is one complete set practiced the same way each time, not three varying routines. Progression across the 4 weeks is via `variants[0..3]` on each movement (more reps / deeper stance), not different content per day. `renderProgram()` reflects this — it shows the 8-movement breakdown once, not three times, then the shared 4-week × 3-day progress grid (`renderProgressGrid()`, same component used on the home screen).

`SESSION_MIN = 20`. `PROGRAM.phaseShare` splits the session evenly across the 8 phases (`1/8` each = 150s ≈ 2:30 per movement). `phaseSeconds(totalMin)` computes this generically from whatever keys exist in `phaseShare` (no hardcoded phase names) and `buildTimeline(dayIdx, week, totalMin)` builds one timeline item per phase — no special-cased meditation branch, no `rest`-item insertion.

Phase `name` fields are short ordinals ("First".."Eighth") — that's what shows in the compact phase-bar strip at the top of preview/session screens (truncated to one word) and in the "Session breakdown" list on the home screen. The full traditional movement title lives on the exercise's `name` field and is what displays large on the preview/session screens.

---

## Key Constants

| Constant | Description |
|----------|-------------|
| `T` | Palette + font refs, identical to B·Restore |
| `BADUANJIN_PHASES` | The 8 phases/movements, shared by all 3 `PROGRAM.days` entries |
| `PROGRAM` | 3 identical day entries (see Program Data above) × 4-week variants |
| `EX_INFO` | The 8 movements → `{ desc, illo }` — `desc` is a plain-language how-to cue, `illo()` returns an inline SVG stick-figure diagram |
| `HOME_PHRASES` | 10 home-screen quotes, Qigong-flavored |
| `SESSION_MIN` | `20` |

`getExInfo(name)` falls back to a generic cue for any name not in `EX_INFO` (no `illo` fallback — `renderPreview` just skips the diagram box if `info.illo` is absent).

### Illustrations

`svgFigure(inner)` wraps a `viewBox="0 0 100 140"` stick figure. Shared helpers: `bone(points, color)` (a `<polyline>` limb — 2+ points), `headC(cx, cy, color)` (head circle), `groundLine()`, and stance presets `LEGS_NARROW()` / `LEGS_HORSE()`. Each movement's `illo()` composes these with hand-picked coordinates for that pose — there's no generic pose parameterization, just 8 hand-tuned diagrams. Left/right limbs always originate from distinct shoulder points (`x=40` / `x=60`, not a shared `x=50` point) — sharing one origin makes two opposite-reaching limbs visually merge into a single line. "Two Hands Hold the Feet" is drawn side-on rather than front-facing, since a forward fold collapses onto one vertical line in a frontal view.

### Voice readout

`speakText(text)` uses the browser's built-in `speechSynthesis` API (no external service, works offline once the page is loaded, native to iOS Safari) to read a movement's `desc` aloud. Wired to a small speaker-icon button (`iconSpeaker`) next to the "How to do it" header on the preview screen — tap it, it cancels any in-flight utterance and speaks `${exName}. ${desc}`.

---

## Key Functions

Session lifecycle (`startSession`, `pickDay`, `startTimer`, `updateTimerDisplay`, `skipExercise`, `abortSession`, `finishSession`) mirrors the workout app's implementation as closely as the domain allows — same wake-lock/no-sleep-video handling, same beep/vibrate on transition, same 3-2-1 countdown before a timer starts.

`getStreak()` and `pickMessage(streak, week, dayTag, total)` are ported with Qigong-specific copy (see source for the full message pool — milestones, week+tag combos, general rotation). Since `dayTag` is always `'BADUANJIN'` now, the week+tag combo dict only needs 4 entries (`1-BADUANJIN`..`4-BADUANJIN`), not 12.

`shareSession(streak, total, duration)` is a deliberately simple replacement for the workout app's Canvas share-card: it builds a one-line text summary and uses `navigator.share` (falling back to clipboard). No Canvas, no image generation — kept lightweight since it wasn't a specifically requested feature.

---

## What NOT to Do

- Don't add a framework or build step
- Don't split into multiple files
- Don't add emojis
- Don't push directly to main
- Don't add colors outside the `T` object or fonts beyond the four loaded
- Don't add video (hardcoded IDs are unverifiable; even a search link adds ad/loading friction to a same-movements-every-session practice) — use the built-in SVG illustrations instead
- Don't reuse `mf.*` localStorage keys — this app shares an origin with `cwinter1/workout`
- Don't port over Garmin/measurements/photo-journal features unless specifically asked — they don't map onto a breathwork practice
