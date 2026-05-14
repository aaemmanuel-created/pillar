# PVA App — Bug Hunt & Code Quality Audit

Use this prompt to systematically find and fix bugs, logic errors, and structural issues in the PVA app.

---

## Prompt

You are auditing a single-file React app (8,058 lines of JSX compiled by Babel Standalone in-browser). The app is a gamified Christian daily discipline tracker called PVA (Person, Vehicle, Assignment).

**Critical technical constraints you MUST respect:**
- React 18 loaded via CDN (UMD globals) — NOT ES module imports
- Must use `const { useState, useEffect, useCallback, useRef, useMemo } = React;` — never `import`
- Must use `function PPPPApp()` — never `export default`
- Babel Standalone compiles in-browser via `<script type="text/babel">`
- Font constant: `const FONT = "'JetBrains Mono', 'Fira Code', monospace";`
- Theme object `th` with properties: text, dim, mid, ghost, glass, glassBorder, surface, blue, blueDim, red, gold, bg, grad, check
- Glass style helpers: `glassStyle(th)`, `glassStyleElevated(th)`
- All state persisted through `loadState()`/`saveState()` using localStorage

**The source file is at:** `pppp-app/src/pppp-app.jsx`
**Build command:** `cd pppp-app && bash build.sh`
**Deploy copy goes to:** `pva-deploy/index.html`

---

### PHASE 1 — JSX Structural Integrity

Scan the ENTIRE file for:

1. **Mismatched JSX tags** — Every `<div>`, `<span>`, `<button>`, `<svg>`, fragment `<>` must have a matching close tag. Pay special attention to:
   - Ternary expressions (`condition ? <A/> : <B/>`) where one branch has different nesting depth
   - `.map()` callbacks that return JSX — check every opening tag has a close
   - IIFE patterns `{(() => { ... })()}` that return JSX — verify complete tag closure
   - Fragments `<>...</>` — count opens vs closes across the whole file

2. **Extra closing braces `}}`** — Search for patterns like `"--"}}` or `"text"}}` where a double brace after a string literal closes both a JSX expression AND a parent scope incorrectly

3. **Unclosed style objects** — Every `style={{ ... }}` must have matching `}}>`. Search for `style={{` without a corresponding `}}>`

4. **Self-closing tag issues** — Components like `<SectionDivider />`, `<Bar />`, `<HealthBar />` must be self-closed if they have no children

### PHASE 2 — State & Data Flow Bugs

5. **State initialisation mismatches** — Compare the fresh state object (returned when localStorage is empty, around line 645) with the day-rollover state (around line 628). Every field in one must exist in the other. Check for:
   - `loginBonusDay`, `loginBonusXp`, `prestige`, `prestigeTitle`, `prestigeBadges`
   - `sabbathGratitude`, `sabbathReflections`
   - Any field added by recent features that may be missing from one or both init paths

6. **Undefined access** — Search for array accesses like `ARRAY[index]` where `index` could be out of bounds:
   - `LOGIN_BONUS_CHAIN[currentDay]` when `currentDay` could be 7 (array only has indices 0-6)
   - `PRESTIGE_TITLES[prestige]` when prestige exceeds array length
   - `LEVELS[idx]` edge cases
   - `state.history[state.history.length - 1]` when history could be empty

7. **Stale closure bugs** — Look for `setState` callbacks that reference outer variables that may be stale:
   - `setTimeout`/`setInterval` callbacks referencing state
   - Event handlers defined outside `useCallback` that capture state values

8. **Save-after-setState race** — The pattern `setState(p => { ... }); saveState(state)` is buggy because `saveState` runs with the OLD state before React applies the update. Every `saveState()` call should use the value from the setState callback, not the current state variable

### PHASE 3 — Logic & Calculation Errors

9. **XP multiplier stacking** — Verify that the prestige multiplier (`1 + prestige * 0.1`) is applied correctly and consistently:
   - Is it applied when toggling tasks?
   - Is it applied to daily challenge XP?
   - Is it applied to boss battle XP?
   - Is it applied to login bonus XP?
   - Is it double-applied anywhere?

10. **Login bonus day tracking** — Verify the day counter:
    - Does it correctly cycle 0→1→2→3→4→5→6→0→1...?
    - What happens on the first ever load (no previous state)?
    - Is the XP actually added to `totalXp` in the day-rollover logic?

11. **Streak calculation** — Check the streak logic in `loadState()`:
    - Does Sabbath day correctly protect the streak?
    - Does grace day correctly protect the streak?
    - Does anchor habit correctly protect the streak?
    - Edge case: what if the user opens the app after 2+ missed days?

12. **HP / Strength system** — Now reframed as "Strength":
    - Verify all display labels say "STRENGTH" or "STR" not "HP"
    - Check that the Season of Renewal overlay triggers at hp <= 0
    - Verify completing all 4 spirit tasks correctly restores HP to 100
    - Check the warm amber color scheme is applied (not red)

13. **Sabbath mode** — When `state.isSabbath` is true:
    - Verify the Four Pillars section is hidden
    - Verify the Daily Routine section is hidden
    - Verify the SabbathExperience component renders
    - Check that gratitude/reflection inputs save to state
    - Verify the weekly review stats calculate correctly from `state.history`

14. **Prestige system** — When user reaches Overcomer:
    - Does the prestige screen trigger?
    - Does "BEGIN ANEW" correctly reset `totalXp` and `xp` to 0?
    - Does it correctly increment `prestige` count?
    - Does it persist the title and badge?
    - Can the user prestige multiple times?

### PHASE 4 — UI & UX Issues

15. **Dark mode compatibility** — Search for any hardcoded colours:
    - `color: "#000"` or `color: "black"` — should be `color: th.text`
    - `background: "#fff"` or `background: "white"` — should use theme
    - `color: "#000"` on prestige button text — is this intentional (gold background)?

16. **Missing `fontFamily: FONT`** — Every text element should use the FONT constant. Search for any `<div>`, `<span>`, `<button>`, `<input>`, `<textarea>` that renders text but lacks `fontFamily: FONT` in its style

17. **Overflow & scroll issues** — Check for:
    - Content that could overflow on small mobile screens (< 375px width)
    - Horizontal scroll where it shouldn't exist
    - The pillar tiles scroll container — does it snap correctly?

18. **Touch targets** — Interactive elements should be at least 44x44px for mobile. Check:
    - Task checkboxes
    - The FAB (floating action button)
    - Navigation buttons
    - Login bonus day circles

19. **Z-index conflicts** — Multiple overlays exist (RedemptionMode, PrestigeScreen, modals). Verify z-index stacking:
    - PrestigeScreen: z-index 1001
    - RedemptionMode: z-index 1000
    - Other modals — what are their z-indexes?
    - Could two overlays appear simultaneously?

### PHASE 5 — Performance

20. **Unnecessary re-renders** — The main `PPPPApp` component is massive. Look for:
    - Inline arrow functions in JSX that create new references every render
    - Object/array literals in style props that aren't memoized
    - Heavy computation in render body that should be in `useMemo`

21. **localStorage spam** — Does `saveState()` get called too frequently?
    - Is it called on every task toggle?
    - Is it called on every input keystroke in Sabbath mode?
    - Could this cause jank on low-end devices?

22. **Memory leaks** — Check for:
    - `setInterval`/`setTimeout` without cleanup in `useEffect`
    - Event listeners added without removal
    - `toastTimerRef` — is the timeout properly cleared?

### PHASE 6 — Edge Cases

23. **First-time user flow** — Clear localStorage and walk through:
    - Does onboarding show correctly?
    - Can the user skip without crashing?
    - Does the home page render without errors after skip?
    - Is `loginBonusDay` correctly initialised?

24. **Date boundary** — What happens if the user opens the app at 11:59 PM and keeps it open past midnight?
    - `TODAY` is computed once at module load — it won't update
    - This could cause state corruption if tasks are toggled after midnight

25. **Firebase offline** — When Firebase is unavailable:
    - Does the app still load?
    - Do all features work with localStorage only?
    - Are there any uncaught promise rejections?

26. **Empty data** — Test with:
    - No tasks configured (all sections empty)
    - No history data
    - No profile set
    - No routine data

---

### Output Format

For each bug found, provide:
1. **Bug ID** (e.g., BUG-001)
2. **Severity** (Critical / High / Medium / Low)
3. **Location** (file + line number)
4. **Description** of the issue
5. **Root cause** analysis
6. **Fix** — the exact code change needed

After listing all bugs, apply ALL fixes to the source file, then run `cd pppp-app && bash build.sh` and copy the result to `pva-deploy/index.html`.
