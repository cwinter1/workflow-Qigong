# CLAUDE.md — Q·Flow (Qigong PWA)

## Purpose & Identity

**Q·Flow** is a personal morning Qigong practice app for Chris, built as a sibling to the **Morning Flow / B·Restore** strength-and-mobility app (`cwinter1/workout`). Where B·Restore undoes what desk work does to the body through strength, posture work, and yoga, Q·Flow is a quieter counterpart built around **Baduanjin (Eight Pieces of Brocade)** — the standardized, traditional 8-movement qigong set. The app went through one earlier iteration built around a general, non-standardized flowing-qigong routine (5 broad phases per session); it was replaced with the actual Baduanjin set once it became clear that's what was wanted.

The app is a 4-week program: 20 minutes, 3 mornings a week. The app title is in Hebrew: **Q·Flow · כריס** (כריס = Chris). Layout is `lang="he" dir="rtl"`, and — unlike the workout app, which keeps English body copy inside its RTL shell — **all in-app text and the voice readout are in Hebrew** (see "Localization" below). Same audience and constraints as B·Restore: iOS Safari, one person, Israel.

**Live URL**: https://cwinter1.github.io/workflow-Qigong/ (GitHub Pages, deploys from `main`).

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

`BADUANJIN_PHASES` — the 8 traditional pieces, each its own phase (`b1`..`b8`), each phase containing exactly one exercise (the movement itself). Movement/phase names are stored in Hebrew (`name`); each exercise also carries an English `nameEn` used only for the YouTube search link (see Localization):

| # | Phase name (he) | Movement (he / en) | Benefit (traditional framing) |
|---|------------|----------|--------------------------------|
| 1 | ראשון | שתי ידיים תומכות בשמיים / Two Hands Hold Up the Heavens | Regulates the Triple Burner; lengthens the spine |
| 2 | שני | מתיחת הקשת לירות בנץ / Drawing the Bow to Shoot the Hawk | Opens the chest; strengthens arms/shoulders/lungs |
| 3 | שלישי | הפרדת שמיים וארץ / Separate Heaven and Earth | Regulates the spleen and stomach |
| 4 | רביעי | הינשוף החכם מביט לאחור / The Wise Owl Gazes Backward | Relieves general strain and fatigue |
| 5 | חמישי | נדנוד הראש וניעור הזנב / Sway the Head and Shake the Tail | Clears heat/tension from chest and head |
| 6 | שישי | שתי ידיים אוחזות בכפות הרגליים / Two Hands Hold the Feet | Strengthens the kidneys and low back |
| 7 | שביעי | קמיצת אגרופים ומבט זועם / Clench the Fists and Glare Fiercely | Builds strength and qi |
| 8 | שמיני | קפיצות קלות על קצות האצבעות והעקבים / Bouncing on Toes and Heels | Shakes the body loose; closes the set |

`PROGRAM.days[0/1/2]` are **identical** — all three point at `BADUANJIN_PHASES` with the same title/kicker/tag (`"באדואנג'ין"`). This is deliberate: Baduanjin is one complete set practiced the same way each time, not three varying routines. Progression across the 4 weeks is via `variants[0..3]` on each movement (more reps / deeper stance), not different content per day. `renderProgram()` reflects this — it shows the 8-movement breakdown once, not three times, then the shared 4-week × 3-day progress grid (`renderProgressGrid()`, same component used on the home screen).

`SESSION_MIN = 20`. `PROGRAM.phaseShare` splits the session evenly across the 8 phases (`1/8` each = 150s ≈ 2:30 per movement). `phaseSeconds(totalMin)` computes this generically from whatever keys exist in `phaseShare` (no hardcoded phase names) and `buildTimeline(dayIdx, week, totalMin)` builds one timeline item per phase — no special-cased meditation branch, no `rest`-item insertion.

Phase `name` fields are short ordinals ("First".."Eighth") — that's what shows in the compact phase-bar strip at the top of preview/session screens (truncated to one word) and in the "Session breakdown" list on the home screen. The full traditional movement title lives on the exercise's `name` field and is what displays large on the preview/session screens.

---

## Key Constants

| Constant | Description |
|----------|-------------|
| `T` | Palette + font refs, identical to B·Restore |
| `BADUANJIN_PHASES` | The 8 phases/movements, shared by all 3 `PROGRAM.days` entries |
| `PROGRAM` | 3 identical day entries (see Program Data above) × 4-week variants |
| `EX_INFO` | Keyed by the Hebrew movement `name` → `{ desc, illo }` — `desc` is a plain-language how-to cue (Hebrew), `illo()` returns an inline SVG stick-figure diagram |
| `HOME_PHRASES` | 10 home-screen quotes, Qigong-flavored |
| `SESSION_MIN` | `20` |

`getExInfo(name)` falls back to a generic cue for any name not in `EX_INFO` (no `illo` fallback — `renderPreview` just skips the diagram box if `info.illo` is absent).

### Illustrations

`svgFigure(inner)` wraps a `viewBox="0 0 100 140"` stick figure. Shared helpers: `bone(points, color, animTo?, dur?)` (a `<polyline>` limb — 2+ points), `headC(cx, cy, color, animCx?, animCy?, dur?)` (head circle), `groundLine()`, and stance presets `LEGS_NARROW()` / `LEGS_HORSE()`. Each movement's `illo(animated)` composes these with hand-picked coordinates for that pose — there's no generic pose parameterization, just 8 hand-tuned diagrams. Left/right limbs always originate from distinct shoulder points (`x=40` / `x=60`, not a shared `x=50` point) — sharing one origin makes two opposite-reaching limbs visually merge into a single line. "Two Hands Hold the Feet" is drawn side-on rather than front-facing, since a forward fold collapses onto one vertical line in a frontal view.

**Every `illo` takes an `animated` boolean:**
- `illo()` / `illo(false)` (used by `renderPreview`) → static single pose, exactly as before.
- `illo(true)` (used by `renderSession`, the timer screen) → the moving limbs loop neutral→pose→neutral via native SVG SMIL `<animate>` on the `points`/`cx`/`cy` attributes — no JS/CSS keyframes, and it works in Safari. `movingBone(neutralPts, activePts, color, animated, dur?)` / `movingHead(nCx, nCy, aCx, aCy, color, animated, dur?)` are the two helpers that pick static-vs-animated output; `NEUTRAL_ARM_L`/`NEUTRAL_ARM_R` are the shared "arms hanging at sides" baseline reused by several movements. Default loop is `2.6s`; "Bouncing on Toes and Heels" uses `1.8s` since a bounce reads better faster.

This exists specifically so there's something to move with in real time on the timer screen, as an alternative to embedding actual video — real per-movement tutorial videos were investigated (see git history around the Hebrew-translation-era commits) but rejected: YouTube blocks this project's fetch tooling so candidate video IDs can't be verified as live/embeddable, and the best candidates found via search came from 8 different, inconsistent creators. The animated diagram is the reliable, self-contained answer to "give me something to copy along with while the timer counts."

### Voice readout

`toggleSpeech(text, btn)` uses the browser's built-in `speechSynthesis` API (no external service, works offline once the page is loaded, native to iOS Safari) to read a movement's `desc` aloud. Wired to a small speaker-icon button (`iconSpeaker`/`iconStop`) next to the "How to do it" header on the preview screen. It's a **toggle, not a fire-and-forget call** — tapping it while it's already reading stops playback immediately (swapping to `iconStop` while active) rather than restarting or running to completion regardless of further taps. `_speakingBtn` (module-level) tracks which button is currently speaking; `render()` also calls `speechSynthesis.cancel()` on every screen transition so navigating away (e.g. starting the timer) always stops it too. `utter.lang = 'he-IL'` is set explicitly so iOS picks a Hebrew voice rather than defaulting to whatever the system locale implies.

### Localization

All in-app text is Hebrew — there's no i18n layer or language switch, since this is a single-language app for one Hebrew-speaking user. Translated content lives directly in the data (`BADUANJIN_PHASES`, `EX_INFO`, `HOME_PHRASES`, `pickMessage`'s message pools) and inline in each `renderX()` function's UI strings.

One deliberate exception: each exercise carries both `name` (Hebrew, displayed and spoken) and `nameEn` (English, used only by `youtubeSearchUrl(nameEn)`). Baduanjin tutorials are overwhelmingly in English/Chinese, so an English search query surfaces far better results than a Hebrew one would for this Chinese-origin content — don't switch that link to the Hebrew name.

`W{n}`/`D{n}` week/day labels are kept as Latin-letter mono abbreviations throughout (headers, chips, progress grid) rather than translated — this matches the sibling workout app's established convention of treating them as compact technical labels, not prose.

---

## Key Functions

Session lifecycle (`startSession`, `pickDay`, `startTimer`, `updateTimerDisplay`, `skipExercise`, `abortSession`, `finishSession`) mirrors the workout app's implementation as closely as the domain allows — same wake-lock/no-sleep-video handling, same beep/vibrate on transition, same 3-2-1 countdown before a timer starts.

`getStreak()` and `pickMessage(streak, week, total)` are ported with Qigong-specific copy (see source for the full message pool — milestones, week combos, general rotation). The signature dropped a `dayTag` parameter it used to take: since there's only ever one day type (Baduanjin), a `${week}-${dayTag}` combo key was redundant — the combo dict is now just keyed by `week` (1-4) directly.

`shareSession(streak, total, duration)` is a deliberately simple replacement for the workout app's Canvas share-card: it builds a one-line text summary and uses `navigator.share` (falling back to clipboard). No Canvas, no image generation — kept lightweight since it wasn't a specifically requested feature.

---

## What NOT to Do

- Don't add a framework or build step
- Don't split into multiple files
- Don't add emojis
- Don't push directly to main
- Don't add colors outside the `T` object or fonts beyond the four loaded
- Don't make video the primary movement reference (hardcoded IDs are unverifiable, and ads/intros add friction to a same-movements-every-session practice) — the built-in SVG illustration is the default; the YouTube search link is a small, secondary fallback underneath it
- Don't reuse `mf.*` localStorage keys — this app shares an origin with `cwinter1/workout`
- Don't switch the YouTube search link to the Hebrew movement name, or drop the `nameEn` field — English search terms surface real tutorials for this content far better than Hebrew ones do
- Don't port over Garmin/measurements/photo-journal features unless specifically asked — they don't map onto a breathwork practice
- Don't add a rotating pool of "ancient Chinese proverbs" or invented historical color. Discussed and explicitly declined (2026-07-06): a lot of what circulates online attributed to Confucius/Laozi is fabricated or mistranslated, and a large AI-generated "wisdom" pool risks presenting invented quotes as authentic. If cultural/historical depth is wanted later, keep it narrow and verifiable — e.g. one factual paragraph on Baduanjin's actual history (roughly Song Dynasty origin, the Yue Fei attribution being legend rather than confirmed fact) on the Program screen, or a couple of properly sourced classical lines — not a large invented pool.
