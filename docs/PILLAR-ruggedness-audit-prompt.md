# PILLAR React Native (Expo) — Ruggedness & Vibe-Code Audit Prompt

> **Role:** You are a senior production engineer who has shipped apps to millions of users and has reviewed hundreds of AI-generated codebases. Your speciality is finding the bugs that AI *doesn't know it introduced* — the silent failures, the happy-path-only logic, the missing edge cases, and the compliance gaps that get apps rejected from the App Store. Your job is to make the PILLAR app (`pillar-app/`) bulletproof and store-ready.

> **Important context:** This entire codebase was AI-generated (vibe-coded). Research shows AI-generated code produces 1.75x more logic errors, 1.64x more maintainability errors, 1.57x more security findings, and 1.42x more performance issues than human-written code. 60% of AI code faults are "silent failures" — code that compiles, runs, and appears correct but produces wrong results in edge cases. Treat every file as untrusted until proven otherwise.

---

## AUDIT 1 — HAPPY PATH BIAS SWEEP

AI models generate code that works under ideal conditions but neglects the real world. For EVERY function and component in the codebase, ask: "What happens when this goes wrong?"

### 1.1 Network & Connectivity Failures

Go through every Firestore call in `src/utils/cloudSync.ts` and every Firebase Auth call in `src/context/AuthContext.tsx` and verify:

- [ ] What happens when the device has no internet? Does the app crash, hang, or handle it gracefully?
- [ ] What happens when a Firestore read/write times out mid-operation? Is the local state left in a half-updated state?
- [ ] What happens when Firebase Auth returns an unexpected error code (not just "wrong password" but network timeout, too-many-requests, user-disabled, user-not-found)?
- [ ] What happens when `loadFromCloud()` succeeds but returns a document with missing fields?
- [ ] What happens when `saveToCloud()` fails silently — does the user think their data is backed up when it isn't?

**Fix pattern:** Every async operation that touches the network must have:
1. A timeout (don't let the user wait forever)
2. A specific error handler (not just `catch {}`)
3. A user-visible feedback mechanism (loading spinner, error toast, retry button)
4. A safe fallback (local state preserved if cloud fails)

### 1.2 Empty, Null, and Undefined Inputs

For every function that accepts parameters, test with:
- `null`, `undefined`, `""` (empty string), `0`, `-1`, `NaN`, `Infinity`
- Empty arrays `[]` and empty objects `{}`
- Strings containing only whitespace `"   "`

**Specific files to check:**

- `src/utils/xp.ts` → `getLevel(NaN)`, `getLevel(-1)`, `getLevel(Infinity)`, `getMult(0)`, `getMult(-5)`, `xpWithVariance(0)`, `xpWithVariance(-10)`, `levelProgress(NaN)`
- `src/utils/dailyReset.ts` → `rollOver(null, null)`, `rollOver(stateWithMissingFields, null)`, `computeDayPct(state, [])`, `computeDayPct(state, null)`
- `src/utils/storage.ts` → `getJSON("nonexistent_key")`, `setJSON("key", undefined)`, `setJSON("key", circular_reference)`, `TODAY()` on a device with a corrupted system clock
- `src/utils/cloudSync.ts` → `saveToCloud("", data)`, `saveToCloud(uid, null)`, `loadFromCloud("")`, `shouldUseCloud(null)`, `shouldUseCloud({})`, `shouldUseCloud({ pppp_state: "not_valid_json" })`, `tsToMs("not-a-date")`, `tsToMs(null)`, `tsToMs({ seconds: -1 })`
- `src/utils/routineBuilder.ts` and `routineParser.ts` → Empty input, malformed input, input with HTML/script tags, extremely long input (10,000+ characters)

**Fix pattern:** Every function must either:
1. Validate inputs at the top and return a safe default, OR
2. Throw a typed error that the caller handles

### 1.3 Boundary Values & Overflow

- [ ] What happens when `totalXp` reaches `Number.MAX_SAFE_INTEGER`? (unlikely but must not crash)
- [ ] What happens when `streak` is `0`? Does `getMult(0)` return the right value?
- [ ] What happens when `history` array has 10,000 entries? (the trim logic caps at 30, but what if the cloud restores a corrupted document with thousands?)
- [ ] What happens when a task's `target` is `0`? Does `incCounter` divide by zero or hit an infinite loop?
- [ ] What happens when `loginBonusDay` exceeds the length of `LOGIN_BONUS_CHAIN`? Is there an array bounds check?
- [ ] What happens when `prestige` is a very large number? Does `prestigeMult = 1 + prestige * 0.1` overflow or create absurd XP values?

**Fix pattern:** Add explicit bounds checks and clamp values to sane ranges. Document what "sane" means for each field in `src/types.ts` as comments.

### 1.4 Concurrent Access & Race Conditions

- [ ] What happens when the user taps two different tasks at the exact same time? Do both `setState` calls in `toggleTask` race?
- [ ] What happens when `useCloudSync` fires a push while a pull is still in progress?
- [ ] What happens when the app is backgrounded mid-sync and then foregrounded — does it retry, double-sync, or corrupt?
- [ ] What happens when `rollOver()` runs while a `saveState()` is still writing to AsyncStorage?
- [ ] What happens when the user kills the app mid-write to AsyncStorage? Is the stored JSON truncated/corrupted?

**Fix pattern:** 
- Use a mutex/lock for cloud sync operations (only one sync at a time).
- For AsyncStorage writes, consider writing to a temp key first, then renaming (atomic swap pattern).
- For `setState` batching, ensure the updater function pattern is used everywhere (it already is in most places — verify ALL).

---

## AUDIT 2 — SILENT FAILURE DETECTION

These are the bugs that won't show up until production. The app will appear to work but data will be wrong.

### 2.1 State Persistence Integrity

- [ ] After every `saveState()` call, can `loadStateRaw()` correctly round-trip the data? JSON.stringify/parse can lose: `undefined` values (silently dropped), `Date` objects (become strings), `NaN`/`Infinity` (become `null`), functions, symbols, circular references.
- [ ] Does `freshState()` produce a state that passes TypeScript's `AppState` type at runtime, not just compile time? Add a runtime validator.
- [ ] When a new field is added to `AppState` in an update, do existing users' saved states gracefully handle the missing field? The `rollOver()` function backfills some fields — verify it covers ALL optional fields in the type.
- [ ] Does `markLocalUpdate()` always fire after `setJSON()`? Check that the call order is correct and that a failed `markLocalUpdate` doesn't leave sync timestamps stale.

### 2.2 XP Calculation Correctness

This is the core of the gamification loop. If XP math is wrong, users either feel cheated or exploit the system.

- [ ] Verify that toggling a task ON then immediately OFF results in exactly net-zero XP change (not off-by-one due to variance).
  - Currently `toggleTask` awards `xpWithVariance(base)` on completion but refunds using `p.xpAwards[taskId]` (the stored award). This is correct IF `xpAwards` is always populated. Verify it is.
- [ ] Verify that `totalXp` can never go negative — trace every code path that subtracts from it: `toggleTask` untoggle, `incCounter` decrement, `rollOver` decay, `toggleChallenge` untoggle.
- [ ] Verify that `streak` can never go below 1 (a fresh user starts at 1, not 0). Trace every code path.
- [ ] Verify that `rollOver()` correctly handles the case where the user opens the app after 3 days away — does it roll over once (correct) or three times (wrong)?
- [ ] Verify that `computeDayPct()` returns 0 when there are no tasks, not `NaN` from `0/0`.

### 2.3 Date & Time Bugs

Date handling is the #1 source of silent failures in AI-generated code.

- [ ] `TODAY()` uses `new Date()` which is local device time. If a user travels across time zones mid-day, does the rollover fire twice or not at all?
- [ ] What happens at exactly midnight? If the user has the app open at 23:59:59 and a state mutation fires at 00:00:01, does the rollover trigger mid-session and wipe their in-progress tasks?
- [ ] Are date comparisons always string-based (`"2026-05-08" === "2026-05-08"`)? What happens if one side is `"2026-5-8"` (no zero-padding)? `TODAY()` zero-pads correctly — verify nothing else constructs dates differently.
- [ ] `history` entries use date strings. If the device clock jumps backward (DST, manual change), could two history entries share the same date, or could a future-dated entry appear?

### 2.4 AsyncStorage Corruption Resilience

- [ ] What happens if AsyncStorage returns corrupted JSON (truncated write from a crash)? Does `getJSON()` handle `JSON.parse` throwing?
- [ ] What happens if AsyncStorage is full? The `setJSON()` function swallows the error — the user's progress is silently lost.
- [ ] What happens if a value stored as a string is expected as an object, or vice versa?
- [ ] Is there any mechanism to detect and recover from a corrupted state? Consider adding a state checksum or version number.

---

## AUDIT 3 — APP STORE COMPLIANCE (REJECTION BLOCKERS)

These will get the app rejected. Fix them before submitting.

### 3.1 Account Deletion (Apple Guideline 5.1.1(v)) — MANDATORY

Apple requires that any app supporting account creation must allow users to delete their account from within the app. This has been enforced since June 30, 2022.

**Required implementation:**
- Add a "Delete Account" button in the Settings screen.
- The flow must:
  1. Show a confirmation dialog explaining what will be deleted.
  2. Re-authenticate the user (Firebase requires recent auth for account deletion).
  3. Delete the user's Firestore document (`users/{uid}`).
  4. Delete ALL subcollections/related data.
  5. Call `firebase.auth().currentUser.delete()`.
  6. Clear all AsyncStorage keys.
  7. Navigate to the Welcome/Sign-In screen.
- The deletion must be REAL, not just hiding the account.

### 3.2 Privacy Policy (Apple Guideline 5.1.1) — MANDATORY

- [ ] Create a privacy policy page hosted at `https://pillarbypva.app/privacy`.
- [ ] The policy must describe: what data is collected (email, name, usage data, task completion data), where it's stored (Firebase/Google Cloud, US servers), how long it's retained, how users can request deletion, and whether data is shared with third parties (it's not).
- [ ] Add a "Privacy Policy" link in the Settings screen that opens the URL.
- [ ] Add the privacy policy URL in `app.config.ts` under `expo.ios.infoPlist` or reference it in App Store Connect metadata.

### 3.3 Terms of Service — RECOMMENDED

- [ ] Create a terms of service page at `https://pillarbypva.app/terms`.
- [ ] Link it from the Settings screen.
- [ ] Apple and Google both recommend (and sometimes require) this for apps with user accounts.

### 3.4 App Store Metadata Requirements

Verify these are configured in `app.config.ts`:
- [ ] `ios.bundleIdentifier` is set (it is: `app.pillarbypva.pillar`).
- [ ] `android.package` is set (it is: `app.pillarbypva.pillar`).
- [ ] `ios.infoPlist.ITSAppUsesNonExemptEncryption` is set to `false` (it is).
- [ ] `ios.infoPlist.NSCameraUsageDescription` — NOT needed unless you add camera features.
- [ ] `version` follows semver and matches what you'll enter in App Store Connect.
- [ ] App icons exist at all required sizes (check `assets/icon.png` is 1024x1024).
- [ ] Splash screen exists and matches the app's dark theme (verify `assets/splash-icon.png`).

### 3.5 Minimum Functionality (Apple Guideline 4.2)

Apple rejects apps that feel like "glorified websites." Since PILLAR was ported from a PWA running in a WebView, verify:
- [ ] The app uses native navigation (React Navigation), not a web-style single-page approach.
- [ ] The app uses native components (React Native Views, TextInputs), not WebViews.
- [ ] The app uses native features (Haptics, Keep Awake) that justify being a native app.
- [ ] The app doesn't just load a remote URL in a WebView (it doesn't — but verify no WebView components exist).

---

## AUDIT 4 — ERROR HANDLING OVERHAUL

AI-generated code has 2x the error handling gaps of human-written code. This audit fixes that.

### 4.1 Replace Every Silent Catch

Search the entire codebase for `catch` blocks. For every one found:

```bash
grep -rn "catch" src/ --include="*.ts" --include="*.tsx"
```

For each result, classify it:
- **CRITICAL** (data loss possible): `storage.ts`, `cloudSync.ts`, `AppStateContext.tsx` — these MUST log errors and show user feedback.
- **IMPORTANT** (degraded experience): `AuthContext.tsx`, `useCloudSync.ts` — these MUST log errors.
- **LOW** (cosmetic): `App.tsx` font loading, Haptics calls — logging is sufficient.

Replace every empty `catch {}` and `catch () {}` with proper error handling using the logger module from the security audit prompt.

### 4.2 Add User-Facing Error States

For every screen that loads data, verify there is:
- [ ] A loading state (spinner or skeleton).
- [ ] An error state (message + retry button).
- [ ] An empty state (meaningful message when there's no data).

Screens to check:
- `HomeScreen.tsx` — what shows when state fails to load?
- `FocusModeScreen.tsx` — what shows when there are zero tasks?
- `CategoryScreen.tsx` — what shows when the pillar has no sections?
- `WeeklyScreen.tsx` — what shows when history is empty?
- `SeasonScreen.tsx` — what shows when there's no data?
- `SettingsScreen.tsx` — what shows when profile is null?
- All feature screens (Guild, Notes, Prayer Journal, etc.)

### 4.3 Add a Global Unhandled Error Catcher

Beyond the ErrorBoundary component (from the security audit), add:

```typescript
// In App.tsx or index.ts
import { ErrorUtils } from 'react-native';

const defaultHandler = ErrorUtils.getGlobalHandler();
ErrorUtils.setGlobalHandler((error, isFatal) => {
  // Log to your logger
  logger.error('GLOBAL', `${isFatal ? 'FATAL' : 'NON-FATAL'}: ${error.message}`, error);
  // Call default handler
  defaultHandler(error, isFatal);
});
```

Also handle unhandled promise rejections:

```typescript
// Polyfill for tracking unhandled promise rejections
if (typeof global?.addEventListener === 'function') {
  global.addEventListener('unhandledrejection', (event) => {
    logger.error('PROMISE', 'Unhandled rejection', event.reason);
  });
}
```

---

## AUDIT 5 — DATA INTEGRITY & MIGRATION

### 5.1 Schema Versioning

- [ ] Add a `schemaVersion: number` field to `AppState`.
- [ ] Set the current version to `1`.
- [ ] In `loadStateRaw()` (or `rollOver()`), check the schema version. If it's older than current, run a migration function that adds new fields with safe defaults.
- [ ] This prevents crashes when you ship an update that adds new fields to the state — existing users' saved data won't have them.

### 5.2 Data Backup Before Destructive Operations

- [ ] Before `resetState()`, save a backup: `AsyncStorage.setItem('pppp_state_backup', currentState)`.
- [ ] Before cloud restore (`restoreLocalData()`), save a backup of local state.
- [ ] Add a "Restore Backup" option in Settings (hidden behind a long-press or developer menu).

### 5.3 Cloud Data Conflict Logging

- [ ] When `shouldUseCloud()` decides which version to keep, log the decision and the timestamps/XP values that were compared.
- [ ] If the cloud version is chosen, log what local data was overwritten.
- [ ] This creates an audit trail for debugging "my progress disappeared" complaints.

---

## AUDIT 6 — PERFORMANCE & MEMORY

### 6.1 Re-render Prevention

- [ ] Check that `AppStateProvider` doesn't cause the entire app to re-render on every state change. The `value` prop of `Ctx.Provider` creates a new object reference every render. Memoize it with `useMemo`.
- [ ] Check that `toggleTask`, `incCounter`, `setTaskNote`, and `toggleChallenge` are properly memoized with `useCallback` and stable deps.
- [ ] Check that child components consuming context are wrapped in `React.memo` where appropriate.

### 6.2 AsyncStorage Batching

- [ ] The current implementation writes to AsyncStorage on EVERY state change (`useEffect` in `AppStateProvider`). If the user toggles 5 tasks in 2 seconds, that's 5 separate writes. Debounce the persist to 500ms.
- [ ] For cloud sync, the 3-second debounce in `useCloudSync` is good — verify it actually works by adding a log.

### 6.3 Memory Leaks

- [ ] Check every `useEffect` returns a cleanup function for subscriptions and timers.
- [ ] Check that the `onAuthStateChanged` listener in `AuthContext` is properly cleaned up.
- [ ] Check that the debounce timer in `useCloudSync` is properly cleared.
- [ ] Check for any event listeners that aren't removed on unmount.

---

## AUDIT 7 — ACCESSIBILITY

### 7.1 Screen Reader Support

- [ ] Every interactive element (`Pressable`, `TouchableOpacity`) must have an `accessibilityLabel` and `accessibilityRole`.
- [ ] Every icon-only button must have an `accessibilityLabel` describing its action.
- [ ] Task checkboxes must announce their state: `accessibilityState={{ checked: isCompleted }}`.
- [ ] Progress indicators (XP bar, day percentage) must have `accessibilityValue={{ now: current, min: 0, max: max }}`.

### 7.2 Dynamic Font Scaling

- [ ] Verify the app doesn't break when the user sets their system font to the largest size.
- [ ] Use `allowFontScaling={false}` ONLY on elements where scaling would break layout (e.g., small badges). Everything else should scale.

### 7.3 Colour Contrast

- [ ] Verify the `dim` colour token has at least 4.5:1 contrast ratio against the `bg` colour (WCAG AA).
- [ ] Verify the `ghost` colour isn't used for important text (it's likely too faint).

---

## EXECUTION INSTRUCTIONS

1. **Read every file** in `src/` before making changes. Understand the full codebase first.
2. **Run the app** with `npx expo start` before making changes. Note any existing errors or warnings.
3. **Work through audits 1–7 in order.** For each checklist item:
   - Test the scenario (manually or with a quick script).
   - If it fails, fix it immediately.
   - If it passes, move on.
4. **Do not skip any checklist item.** Mark each as verified or fixed.
5. **After all audits**, run `npx tsc --noEmit` to verify zero TypeScript errors.
6. **Create a `RUGGEDNESS-REPORT.md`** at project root listing every issue found and every fix applied, with file paths and line numbers.

## PRIORITY ORDER (if time-constrained)

1. **Audit 3** (App Store compliance) — these are hard blockers, the app WILL be rejected without them
2. **Audit 2** (silent failures) — these cause data loss and user trust destruction
3. **Audit 1** (happy path bias) — these cause crashes in the real world
4. **Audit 4** (error handling) — these make debugging possible
5. **Audit 5** (data integrity) — these prevent future update disasters
6. **Audit 6** (performance) — these affect day-to-day experience
7. **Audit 7** (accessibility) — these affect reach and App Store perception
