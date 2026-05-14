# CLAUDE.md — PILLAR by PVA

> Project context for Claude Code. Read this first before making changes.

## ⚠️ Read the shared workflow first

Before doing anything, also read:

- **`../_workflow/PVA-WORKFLOW.md`** — the cross-app dispatch / build / permissions workflow. Defines the **Standard build loop** that this folder's `.claude/settings.json` enforces, and the parity rule (every change ships to web + iOS + Android).
- **`../_workflow/DISPATCH-PROMPTS/`** — drop-in prompts for bugfix, feature, OTA push, dependency bump, store-readiness, brand pass, cleanup, inbox drain, and parity check.
- **`../_workflow/INBOX.md`** — one-line dispatch requests Emmanuel drops in from his phone.

PILLAR has **three surfaces** that must stay at feature parity: web (`pva-deploy/` — single-file PWA on `pillarbypva.app`), iOS and Android (`pillar-app/` — Expo wrapper). End every change-the-app session with a parity status line per `PVA-WORKFLOW.md` §1.

## Project Overview

**PILLAR** is a gamified Christian daily discipline Progressive Web App (PWA). It is a single-file React 18 app (UMD/CDN, compiled by Babel Standalone in-browser) with Firebase Auth + Firestore for cloud sync. Deployed via GitHub Pages at the custom domain **pillarbypva.app**.

**Owner:** Emmanuel Anie-Akwetey (eanieakwetey.eaa@gmail.com)

## Tech Stack

- **React 18** via CDN UMD globals (no ES modules, no build step)
- **Babel Standalone** compiles JSX in-browser via `<script type="text/babel">`
- **Firebase Auth** (Google sign-in + email/password) + **Firestore** (project: `pvaapp-164bf`)
- **PWA**: `manifest.json`, service worker (`sw.js`) with versioned cache (currently **v34**)
- **GitHub Pages** custom domain: A records (185.199.108–111.153) + CNAME → `pillarbypva.app`
- **.app TLD** requires HTTPS (enforced by default)

## Project Structure

All deployable code lives in `pva-deploy/` (this is what GitHub Pages serves):

```
pva-deploy/
├── index.html       (~10,400+ lines — the entire app in one file)
├── sw.js            (service worker, cache version v34)
├── manifest.json    (start_url: "/", scope: "/")
├── icon-192.png     (homescreen icon, gradient arcs, no ghost ring)
├── icon-512.png     (splash icon)
└── CNAME            (pillarbypva.app)
```

Any other PNGs in the folder (`icon-a-*`, `icon-b-*`, `icon-c-*`, `icon-new-*`) are experiments — the canonical icons are `icon-192.png` and `icon-512.png`.

## Architecture of index.html

The entire app is in a single `index.html`. Key sections (approximate line numbers; they shift with edits):

### Global Constants & Utilities (~lines 1–400)
- `FONT = "'JetBrains Mono', 'Fira Code', monospace"` — all fonts in the app
- Global CSS: `*` selector sets `font-family: 'JetBrains Mono'` + `input,textarea,button,select { font-family: inherit }`
- Theme objects: `dark` and `light` with all colour tokens
- `LOGIN_BONUS_CHAIN`, `LEVELS`, `TESTIMONY_MILESTONES`, `DEFAULT_TASKS`
- `loadState()`, `saveState()`, `loadProfile()`, `saveProfile()`
- `loadRoutine()`, `saveRoutineData()` — routine persistence in localStorage key `pva_routine`
- `buildTasksFromRoutineAnswers(routine)` — converts questionnaire answers to 4-pillar task structure

### Routine Builders (~lines 3505–3845)
- `parseRoutineText(text)` — parses free-form text like "05:00-06:00 - Prayer" into task items
- `buildTasksFromRoutine(items)` — categorises parsed items into person/people/process/product
- `GuidedRoutineBuilder({ onFinish, th })` — step-by-step questionnaire, calls `onFinish(buildFromAnswers())`
- `FreeTextRoutineBuilder({ onFinish, th })` — paste-your-routine textarea, calls `onFinish(buildTasksFromRoutine(parsed))`

### ViewPicker (~lines 3849–3910)
- Shows after onboarding (new users) or on every app open (returning users)
- Displays PILLAR logo SVG + "PILLAR by PVA" wordmark + "Choose your view"
- Two options: **Classic** (full dashboard) and **Focus** (minimal, one task at a time)
- Returning users go directly here (skip the splash)

### FocusModeView (~lines 3912–4730)
- Phase 0: entry transition with breathing ring (repositioned above text) + "FOCUS MODE / Build your temple / X TASKS AHEAD"
- Phase 1: active task display, swipe to complete/skip
- All-done celebration state

### OpeningSplash (~lines 4734–4793)
- "Time is a currency, let's spend it wisely." with staggered word-by-word fade-in
- Word duration: 1.8s each, stagger: 0.3s between words
- 3-stage keyframe: `blur(12px)` → partial clear → crisp
- Total duration: ~9.2s (holds 7s, then 2.2s fade-out)
- **Only shows for brand-new users** after completing onboarding
- Returning users skip this and go straight to ViewPicker

### OnboardingScreen (~lines 4795–5267)
- Step 0: Animated intro — logo starts at vertical centre, slides up, "PILLAR by PVA" fades in, description box fades in with pulsing blue glow border
- Step 1: Sign-in (Google or email/password)
- Step 2: "What's your name?" input
- Step 3: "Build Your Routine" — choice of Guided, Paste Text, or Skip (default routine)
- `doFinish(customTasks)` — saves routine data + profile + calls `onComplete`
- `doSkip()` — saves `{ skipped: true }` as routine + profile + calls `onComplete`

### Main App (PPPPApp) (~lines 8274–end)
- All `useState`, `useRef`, `useEffect`, `useMemo`, `useCallback` hooks declared first
- **CRITICAL**: The `if (!profile)` onboarding check is placed AFTER all hooks (around line 8813) to avoid a React hooks-ordering violation. This was a major bug fix — previously hooks were called after an early return, causing the transition from onboarding to splash to silently fail.
- `if (launchFlow)` check renders either OpeningSplash or ViewPicker
- Main home view: pillar tiles, task list, stats, streak, XP, daily challenge
- Focus button: top-right corner, only visible on home (Classic) view
- Login bonus popup: **removed** (code still exists but rendering disabled)

## Key Bug Fixes Applied (Do Not Regress)

### 1. React Hooks Violation (Onboarding → Splash Transition)
**Problem:** After completing onboarding routine, the "let's spend it wisely" splash page wouldn't load without a manual refresh.

**Root cause:** In `PPPPApp`, there was an early `return <OnboardingScreen.../>` when `!profile`. But 13 React hooks (`useRef`, `useEffect`, `useMemo`, `useCallback`) were called AFTER that early return. When `profile` transitioned from null → truthy, React saw a different hook count, silently breaking state updates.

**Fix:**
- Moved the `if (!profile)` check to AFTER all hook declarations (~line 8813)
- Changed `const userName = profile.name` to `const userName = profile ? profile.name : ""`
- All hooks now run on every render regardless of profile state

### 2. Missing saveRoutineData in doFinish
**Problem:** After completing the guided or text routine builder, `loadRoutine()` returned null because the routine was never saved to localStorage.

**Fix:** Added `saveRoutineData({ directTasks: customTasks, ... })` call in `doFinish` before `saveProfile`.

### 3. Focus Mode Breathing Ring Overlap
**Problem:** Circle appeared on top of text in Focus mode entry screen.

**Fix:** Changed the breathing ring from `position: absolute` to a normal flow element in a flex column, positioned above the text with `marginBottom: 32`.

### 4. CSS Keyframes Scoping
**Problem:** Onboarding animations didn't work because keyframes were defined in the home page's `<style>` block, not the onboarding component's.

**Fix:** Added keyframes directly into the OnboardingScreen's own `<style>` tag.

## Current State of Changes

### Animations
- **Onboarding step 0:** Logo at vertical centre → slides up (1.2s) → "PILLAR by PVA" fades in → description box fades in with pulsing blue glow border (3-layer `box-shadow` + border-color animation)
- **Splash ("Time is a currency"):** Slow, cinematic word-by-word fade with blur, 1.8s per word, 0.3s stagger, holds 7s, 2.2s fade-out
- **Home page:** Gentle 0.9s fade-in via `pvaHomePageIn` keyframe

### Visual Design
- **Font:** JetBrains Mono everywhere (global CSS `*` selector + `font-family: inherit` on form elements)
- **Logo arcs:** strokeWidth 4.5 (thinner than original 6)
- **Ghost ring:** Removed from both in-app SVG and icon PNGs
- **Icon PNGs:** Regenerated via ImageMagick from SVG source with thinner strokes, no ghost ring

### User Flow
- **New users:** Onboarding steps 0→1→2→3 → OpeningSplash → ViewPicker → Home
- **Returning users:** ViewPicker (skip splash) → Home
- **Returning user via Firebase auth:** Cloud data restore → ViewPicker → Home

### What's Disabled
- Daily login bonus popup (rendering removed, XP logic still runs in background)
- Ghost ring circle in logo

## Service Worker Cache

Current version: **v34** (`pva-cache-v34` in `sw.js`)

> **IMPORTANT:** Bump the cache version every time you change `index.html` — the service worker caches aggressively and users won't see updates without a version bump.

## Domain & Deployment

- **Domain:** pillarbypva.app (registered on Namecheap)
- **DNS:** A records pointing to GitHub Pages IPs (185.199.108.153, .109, .110, .111) + CNAME
- **GitHub Pages:** Custom domain configured, HTTPS enforced
- **Firebase:** Project `pvaapp-164bf`, authorized domain includes `pillarbypva.app`

## Key Constants

```javascript
const FONT = "'JetBrains Mono', 'Fira Code', monospace";
const APP_VERSION = "..."; // check current value in file
const PROFILE_KEY = "pva_profile";
const ROUTINE_STORAGE_KEY = "pva_routine";
const FIREBASE_PROJECT = "pvaapp-164bf";
```

## Working Conventions

- **No build step.** Do not introduce bundlers, npm packages, or ES modules. The app must run from a static file host with nothing but the files in `pva-deploy/`.
- **Single-file constraint.** All React components live in `index.html` inside one `<script type="text/babel">` block. Do not split into separate files.
- **When editing `index.html`:** bump `pva-cache-v<N>` in `sw.js` as part of the same change.
- **Test locally** by serving `pva-deploy/` over any static server (`python3 -m http.server` works) before pushing.
- **Hook rules:** Never place an early return before React hooks in `PPPPApp`. Every hook must run on every render.

## Owner Preferences

- **Font for documents:** Trebuchet MS (Commonwealth/PhD work), JetBrains Mono (PILLAR app)
- **Sources:** Never use Wikipedia. Use primary/official sources.
- **File formats:** Office suite preferred for documents (not relevant to this code project, but applies to any docs produced).
