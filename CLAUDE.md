# CLAUDE.md — Q·Flow (Qigong PWA)

## Purpose & Identity

**Q·Flow** is a personal morning Qigong practice app for Chris, built as a sibling to the **Morning Flow / B·Restore** strength-and-mobility app (`cwinter1/workout`). Where B·Restore undoes what desk work does to the body through strength, posture work, and yoga, Q·Flow is a quieter counterpart: a general flowing Qigong + breathwork practice — not tied to a single named tradition (not Baduanjin, not a specific animal-frolics set) — built around grounding, continuous movement, and stillness.

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
- Hardcoded YouTube video IDs — there was no way to verify real tutorial videos for this content, so movement guidance is text-only (`EX_INFO[name].desc`) rather than risk a broken or mismatched embed during practice

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
  → renderPreview()    — movement preview (description + intent) before each item
  → renderSession()    — active timer + movement display (dispatches to renderMeditation for the closing phase)
  → renderMeditation() — box-breathing visualization
  → renderDone()       — post-practice completion screen
  → renderProgram()    — 4-week program overview
```

No `renderMeasurements()` / no `renderRest()` — this app has neither body-measurement tracking nor inter-exercise rest periods (Qigong flows continuously; there's nothing to recover from between movements the way there is between strength sets).

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

`PROGRAM.days[0/1/2]` — 3 day archetypes, each repeated across 4 weeks:

| Index | Tag | Title | Kicker |
|-------|-----|-------|--------|
| 0 | ROOT | Root · Grounding Practice | Find the ground before you move |
| 1 | FLOW | Flow · Moving Qi | Continuous, wave-like movement |
| 2 | STILL | Still · Breath & Release | Slow down until it's just breath |

Each day has `phases` (`settle` → `open` → `flow` → `release` → `meditation`) → each phase has `exercises` → each exercise has `variants[0..3]` (one per week).

`SESSION_MIN = 20`. `PROGRAM.phaseShare` gives each non-meditation phase as a fraction of `SESSION_MIN` (settle 2/20, open 3/20, flow 9/20, release 3/20 — sums to 17/20); meditation is hardcoded to 180s regardless of total, same pattern as the workout app. `phaseSeconds(totalMin)` / `buildTimeline(dayIdx, week, totalMin)` mirror the workout app's functions, minus the `rest`-item insertion (Qigong's `flow` phase has no inter-exercise rests).

The `meditation` phase always contains a single shared exercise, **Dantian Breath**, reused across all three day types (same as the workout app reuses "Box Breath"). Its 4-number variant string (`'4·4·4·4'` → `'6·6·6·6'`) drives `getMedState()` exactly like the original box-breathing implementation — inhale/hold/exhale/hold, unchanged math.

---

## Key Constants

| Constant | Description |
|----------|-------------|
| `T` | Palette + font refs, identical to B·Restore |
| `PROGRAM` | Full 3-day × 4-week program data |
| `EX_INFO` | ~43 movements → `{ desc }`, a 1–2 sentence plain-language cue. No `gif`/video field. |
| `HOME_PHRASES` | 10 home-screen quotes, Qigong-flavored |
| `SESSION_MIN` | `20` |

`getExInfo(name)` falls back to a generic cue for any name not in `EX_INFO`.

---

## Key Functions

Session lifecycle (`startSession`, `pickDay`, `startTimer`, `updateTimerDisplay`, `skipExercise`, `abortSession`, `finishSession`) mirrors the workout app's implementation as closely as the domain allows — same wake-lock/no-sleep-video handling, same beep/vibrate on transition, same 3-2-1 countdown before a timer starts.

`getStreak()` and `pickMessage(streak, week, dayTag, total)` are ported with Qigong-specific copy (see source for the full message pool — milestones, week+tag combos, general rotation).

`shareSession(streak, total, duration)` is a deliberately simple replacement for the workout app's Canvas share-card: it builds a one-line text summary and uses `navigator.share` (falling back to clipboard). No Canvas, no image generation — kept lightweight since it wasn't a specifically requested feature.

---

## What NOT to Do

- Don't add a framework or build step
- Don't split into multiple files
- Don't add emojis
- Don't push directly to main
- Don't add colors outside the `T` object or fonts beyond the four loaded
- Don't hardcode YouTube video IDs without a way to verify them
- Don't reuse `mf.*` localStorage keys — this app shares an origin with `cwinter1/workout`
- Don't port over Garmin/measurements/photo-journal features unless specifically asked — they don't map onto a breathwork practice
