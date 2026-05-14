# PILLAR React Native (Expo) — Security Audit, Bug Fix & Hardening Prompt

> **Role:** You are a senior cybersecurity engineer and full-stack mobile developer with 20+ years of experience in React Native, Firebase, and app-store-grade production hardening. Your job is to audit, fix, and harden the PILLAR Expo app located in `pillar-app/`. Do NOT skip any item. Do NOT leave TODOs. Implement every fix fully.

---

## CONTEXT

PILLAR is a gamified Christian daily-discipline app built with Expo SDK 54, React Native 0.81, Firebase Auth (email/password + future Google Sign-In), Firestore cloud sync, and AsyncStorage for local persistence. The scaffold was generated from a single-file PWA (index.html) and needs a thorough security pass before it can ship to the App Store and Google Play.

**Firebase project:** `pvaapp-164bf`
**Package ID target:** `app.pillarbypva.pillar`
**Tech stack:** Expo 54 (managed workflow), React Navigation 7, Firebase JS SDK 12, AsyncStorage, Reanimated 4, Gesture Handler, expo-haptics, expo-keep-awake, expo-linear-gradient, react-native-svg.

---

## PHASE 1 — CRITICAL SECURITY FIXES

### 1.1 Firebase Config Leaked in Source (`src/utils/firebase.ts`)

**Problem:** The Firebase API key, project ID, auth domain, sender ID, and storage bucket are hardcoded in plain text. The `appId` is a placeholder (`1:861398023812:web:0000000000000000000000`) which will cause Firebase init to silently fail or produce cryptic errors.

**Fix:**
- Create a `.env` file at project root (add it to `.gitignore` immediately).
- Install `react-native-dotenv` (or use Expo's built-in `expo-constants` + `app.config.js` approach — the Expo approach is preferred for managed workflow).
- Convert `app.json` → `app.config.ts` so you can read `process.env.*` values.
- Move ALL Firebase config values into environment variables:
  ```
  EXPO_PUBLIC_FIREBASE_API_KEY=AIzaSyDmt9Q3Bvy3VV33L0gJSe-eBEX_2x2W-AY
  EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN=pvaapp-164bf.firebaseapp.com
  EXPO_PUBLIC_FIREBASE_PROJECT_ID=pvaapp-164bf
  EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET=pvaapp-164bf.appspot.com
  EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=861398023812
  EXPO_PUBLIC_FIREBASE_APP_ID=<REAL_APP_ID_HERE>
  ```
- Update `src/utils/firebase.ts` to read from `process.env.EXPO_PUBLIC_*`.
- Add a startup guard: if any env var is missing, throw a clear error in dev and fail gracefully in production.
- Update `.gitignore` to include `.env`, `.env.local`, `.env.*.local`.
- **CRITICAL:** Replace the placeholder `appId` with the REAL Firebase Web App ID from the Firebase console (the user must supply this — add a clear `// TODO: REQUIRED` comment with instructions if you cannot find it).

### 1.2 No Input Validation or Sanitisation (`src/screens/onboarding/SignInScreen.tsx`)

**Problem:** The email and password fields are sent directly to Firebase with only a `trim()` on email. No validation of email format, no minimum password length, no protection against injection payloads in the error display.

**Fix:**
- Add email regex validation before submission (reject malformed emails client-side).
- Enforce a minimum password length of 8 characters with a user-visible error.
- Sanitise the `error` string before rendering — Firebase error messages can contain HTML-like content. Strip or escape any `<` `>` `&` characters.
- Add a submission debounce (disable the button for 2 seconds after each attempt) to prevent rapid-fire auth attempts.
- Apply the same validation to `NameScreen.tsx` — the name field should reject empty strings, strings over 50 characters, and strings containing `<script>` or HTML tags.

### 1.3 No Firestore Security Rules Enforcement (Cloud Sync)

**Problem:** `src/utils/cloudSync.ts` writes to `users/{uid}` with `setDoc(..., { merge: true })`. If your Firestore rules are open (the default for new projects), ANY authenticated user can read/write ANY other user's data by guessing UIDs.

**Fix:**
- Create a `firestore.rules` file at project root with these rules:
  ```
  rules_version = '2';
  service cloud.firestore {
    match /databases/{database}/documents {
      match /users/{userId} {
        allow read, write: if request.auth != null && request.auth.uid == userId;
      }
      match /{document=**} {
        allow read, write: if false;
      }
    }
  }
  ```
- Add a `firebase.json` at project root referencing the rules file.
- Add a README note: "Deploy rules with `firebase deploy --only firestore:rules`".

### 1.4 No Data Validation on Cloud Restore (`src/utils/cloudSync.ts`)

**Problem:** `restoreLocalData()` takes whatever comes from Firestore and writes it directly into AsyncStorage with zero validation. A corrupted or tampered cloud document could inject arbitrary JSON into every storage key, crash the app, or corrupt all local state.

**Fix:**
- Create a `src/utils/validators.ts` module.
- Write a `validateAppState(data: unknown): AppState | null` function that type-checks every field of the AppState interface (check types, ranges, required fields). Return `null` if invalid.
- Write a `validateProfile(data: unknown): Profile | null` function with the same approach.
- In `restoreLocalData()`: parse each cloud key's JSON, run it through the appropriate validator, and only write to AsyncStorage if validation passes. Log warnings for rejected keys.
- In `shouldUseCloud()`: wrap both `JSON.parse()` calls in proper try/catch and validate the parsed objects before comparing `totalXp`.

### 1.5 Auth State Not Gated Properly (`src/utils/cloudSync.ts`, `src/hooks/useCloudSync.ts`)

**Problem:** `saveToCloud(uid, data)` only checks `if (!uid) return` — it doesn't verify that the current Firebase user's UID matches the `uid` parameter. A bug elsewhere could cause data to be written to the wrong user's document.

**Fix:**
- In `saveToCloud()`, add: `if (auth.currentUser?.uid !== uid) return;` (import `auth` from firebase.ts).
- In `loadFromCloud()`, add the same check.
- In `useCloudSync`, verify `user.uid` is defined and non-empty before calling any sync function.

---

## PHASE 2 — BUG FIXES

### 2.1 Firebase `auth` Typed as `any` (`src/utils/firebase.ts`)

**Problem:** `let auth: any;` discards all type safety. The `try/catch` pattern for `initializeAuth` vs `getAuth` is correct but the `any` type hides potential issues.

**Fix:**
- Type it as `import { Auth } from 'firebase/auth'` and use `let auth: Auth;`.
- The `@ts-ignore` on `getReactNativePersistence` can be replaced with a proper type assertion or declaration merge.

### 2.2 Silent Error Swallowing Throughout Codebase

**Problem:** There are 20+ empty `catch {}` or `catch () {}` blocks across `storage.ts`, `cloudSync.ts`, `AuthContext.tsx`, `AppStateContext.tsx`, `App.tsx`, and `firebase.ts`. These silently swallow errors, making debugging impossible.

**Fix:**
- Create a `src/utils/logger.ts` module with a `log.warn(tag, message, error?)` and `log.error(tag, message, error?)` function. In dev mode, these log to console. In production, they write to an in-memory ring buffer (last 50 errors) that can be shown in a debug screen in Settings.
- Replace every empty `catch {}` with `catch (e) { log.warn('TagName', 'description', e); }`.
- The most critical ones to fix:
  - `storage.ts` `setJSON()` — "storage may be full" comment but no logging means the user silently loses data.
  - `cloudSync.ts` `restoreLocalData()` — failed writes are invisible.
  - `AuthContext.tsx` `signOut()` — failed sign-out leaves the app in a broken auth state.
  - `App.tsx` font loading — failed font load is fine to swallow but should still log.

### 2.3 Race Condition in `useCloudSync.ts`

**Problem:** The `useEffect` for debounced push fires on every `state` change. But the `useEffect` for initial pull (on sign-in) also modifies state via `restoreLocalData() → reload()`. This means:
1. User signs in → pull effect fires → restores cloud data → triggers state change → push effect fires → immediately overwrites cloud with the just-restored data (a no-op at best, a data loss vector at worst if the restore was partial).

**Fix:**
- Add a `syncingRef = useRef(false)` flag.
- Set `syncingRef.current = true` at the start of the pull effect, set it back to `false` when done.
- In the push effect, check `if (syncingRef.current) return;` before scheduling the debounced push.
- Also: the eslint-disable comment on the pull `useEffect` deps array means `reload` is missing from deps. Add it properly and wrap the async function in a stable ref if needed.

### 2.4 `app.json` / `app.config` Misconfigurations

**Problem:** Multiple misconfigurations:
- `splash.backgroundColor` is `#ffffff` (white) but the app's dark theme background is `#060810`. Users see a jarring white flash on launch.
- `userInterfaceStyle` is `"light"` but the app defaults to dark mode.
- No `bundleIdentifier` for iOS.
- No `package` for Android.
- `slug` is generic `"pillar-app"` — should match the brand.

**Fix (in the new `app.config.ts`):**
```typescript
splash: {
  image: "./assets/splash-icon.png",
  resizeMode: "contain",
  backgroundColor: "#060810",
},
userInterfaceStyle: "automatic",
ios: {
  supportsTablet: true,
  bundleIdentifier: "app.pillarbypva.pillar",
  infoPlist: {
    ITSAppUsesNonExemptEncryption: false, // Required for App Store — PILLAR uses HTTPS only via Firebase, which is exempt
  },
},
android: {
  adaptiveIcon: {
    foregroundImage: "./assets/adaptive-icon.png",
    backgroundColor: "#060810",
  },
  package: "app.pillarbypva.pillar",
  edgeToEdgeEnabled: true,
},
slug: "pillar-by-pva",
```

### 2.5 `rollOver()` Timezone Vulnerability (`src/utils/dailyReset.ts`)

**Problem:** `TODAY()` in `storage.ts` uses `new Date()` which depends on device local time. A user can change their phone's clock to the past, trigger a rollover, change it back, and trigger another rollover — effectively farming XP and streak resets.

**Fix:**
- Add a `lastKnownDate` field to AppState.
- In `rollOver()`, compare the new date against `lastKnownDate`. If the new date is EARLIER than `lastKnownDate`, skip the rollover and keep the existing state (log a warning).
- Store `lastKnownDate = today` on every successful rollover.
- This doesn't fully prevent clock manipulation but blocks the most obvious exploit.

### 2.6 XP Integer Overflow / Negative XP (`src/context/AppStateContext.tsx`)

**Problem:** `toggleTask` and `incCounter` can produce negative `xp` and `totalXp` values through rapid toggling. The refund logic subtracts from `p.xp` without flooring at 0.

**Fix:**
- After every XP mutation in `toggleTask`, `incCounter`, and `toggleChallenge`, floor the values:
  ```typescript
  xp: Math.max(0, p.xp + (was ? -dx : dx)),
  totalXp: Math.max(0, p.totalXp + (was ? -dx : dx)),
  ```
- `totalXp` should NEVER go below 0. The decay logic in `rollOver()` already does `Math.max(0, ...)` which is correct — apply the same pattern everywhere.

### 2.7 Missing React Error Boundary

**Problem:** There is no error boundary anywhere in the app. Any unhandled JS error in any component crashes the entire app with a white screen or the Expo error overlay.

**Fix:**
- Create `src/components/ErrorBoundary.tsx` — a class component that catches errors and renders a "Something went wrong" screen with a "Restart" button that calls `reload()` from AppStateContext (or `Updates.reloadAsync()` from expo-updates).
- Wrap `<RootNavigator />` in the ErrorBoundary inside `App.tsx`.
- Log the caught error using the new `logger.ts` module.

### 2.8 `useAppState` Stale Closure in Callbacks

**Problem:** `toggleTask` depends on `allTasks` and `totalMult` via `useCallback` deps. But `allTasks` is derived from `useMemo(() => flattenTasks(tasks), [tasks])` and `tasks` comes from `state.customTasks || DEFAULT_TASKS`. If `state.customTasks` changes between renders, the memoised `allTasks` used in the closure may be stale.

**Fix:**
- Move `allTasks.find(...)` inside the `setState` updater function so it reads from the latest state:
  ```typescript
  setState((p) => {
    const currentTasks = p.customTasks || DEFAULT_TASKS;
    const flat = flattenTasks(currentTasks);
    const t = flat.find((x) => x.id === taskId);
    // ... rest of the logic
  });
  ```
- Remove `allTasks` and `totalMult` from the useCallback deps since they're now read inside the updater.

---

## PHASE 3 — HARDENING & PRODUCTION READINESS

### 3.1 Add Network Status Handling

- Install `@react-native-community/netinfo`.
- Create a `useNetworkStatus()` hook.
- In `useCloudSync`, skip sync attempts when offline (don't set status to 'error' — set it to 'offline').
- Show a subtle "offline" indicator in the UI when disconnected.

### 3.2 Add Secure Storage for Auth Tokens

- Install `expo-secure-store`.
- While Firebase Auth handles its own token persistence via AsyncStorage (configured in firebase.ts), consider migrating the auth persistence to SecureStore for sensitive token storage on-device. At minimum, ensure no raw tokens or credentials are stored in plain AsyncStorage.

### 3.3 Add Deep Link / URL Scheme Protection

- In `app.config.ts`, define a `scheme` for the app: `scheme: "pillar"`.
- In the NavigationContainer, add a `linking` config that only accepts known routes.
- Reject malformed or unknown deep links to prevent navigation injection.

### 3.4 Add Certificate Pinning (Optional but Recommended)

- For Firebase connections, consider adding SSL certificate pinning via a custom Expo config plugin. This prevents MITM attacks on the Firebase API endpoints.
- At minimum, document this as a future hardening step.

### 3.5 Secure the Build Pipeline

- Ensure `.env` files are in `.gitignore` (check this — the current `.gitignore` has `.env*.local` but NOT `.env` itself).
- Add `google-services.json` and `GoogleService-Info.plist` to `.gitignore` (these will contain sensitive keys after prebuild).
- Add `*.keystore`, `*.jks`, `*.p12`, `*.p8`, `*.key`, `*.mobileprovision` to `.gitignore` (most are already there — verify).

### 3.6 Prevent Screen Capture of Sensitive Screens (Optional)

- On the Settings screen where the user might see their email or account info, consider using `expo-screen-capture` to prevent screenshots/screen recordings if needed.

### 3.7 Add Proper TypeScript Strict Checks

- `tsconfig.json` has `"strict": true` which is good.
- But there are `@ts-ignore` comments and `any` types throughout. Fix these:
  - `firebase.ts`: `auth` typed as `any` → type as `Auth`.
  - `firebase.ts`: `@ts-ignore` on `getReactNativePersistence` → add a proper type declaration.
  - `AuthContext.tsx`: `catch (e: any)` → use `catch (e: unknown)` and narrow with `e instanceof Error`.
  - `types.ts`: `soulCheckIns: any[]` → define a proper `SoulCheckIn` type.
  - `storage.ts`: `loadNotes` returns `Promise<any>` → define a `Notes` type.

---

## PHASE 4 — VERIFICATION CHECKLIST

After completing all fixes, verify:

1. **`npx expo start` runs without errors** — the app boots on iOS Simulator and Android Emulator.
2. **TypeScript compiles clean** — run `npx tsc --noEmit` with zero errors.
3. **No hardcoded secrets in source files** — grep the entire `src/` folder for API keys, passwords, tokens. Nothing should be hardcoded.
4. **`.gitignore` covers all sensitive files** — `.env`, `google-services.json`, `GoogleService-Info.plist`, keystores, etc.
5. **Cloud sync round-trip works** — sign in, make changes, sign out, sign back in, verify data restores correctly.
6. **Offline mode doesn't crash** — enable airplane mode, use the app, verify no unhandled errors.
7. **Rapid task toggling doesn't produce negative XP** — tap a task 20 times quickly, verify XP never goes below 0.
8. **Error boundary catches crashes** — intentionally throw in a component, verify the fallback screen appears.
9. **Empty/malformed cloud data doesn't corrupt local state** — the validators reject bad data gracefully.
10. **Firestore rules file exists** and is documented for deployment.

---

## FILE MANIFEST (files to create or modify)

### New files to create:
- `app.config.ts` (replaces `app.json`)
- `.env` + `.env.example`
- `firestore.rules`
- `firebase.json`
- `src/utils/validators.ts`
- `src/utils/logger.ts`
- `src/components/ErrorBoundary.tsx`
- `src/hooks/useNetworkStatus.ts`

### Files to modify:
- `src/utils/firebase.ts` — env vars, proper typing, remove hardcoded config
- `src/utils/cloudSync.ts` — data validation, auth gating, error logging
- `src/utils/storage.ts` — error logging, notes typing
- `src/utils/dailyReset.ts` — timezone exploit guard
- `src/utils/xp.ts` — no changes needed (already solid)
- `src/context/AppStateContext.tsx` — XP flooring, stale closure fix, error logging
- `src/context/AuthContext.tsx` — error typing, sign-out error handling
- `src/hooks/useCloudSync.ts` — race condition fix, offline handling
- `src/screens/onboarding/SignInScreen.tsx` — input validation, debounce
- `src/screens/onboarding/NameScreen.tsx` — input validation
- `src/navigation/RootNavigator.tsx` — wrap in ErrorBoundary
- `src/types.ts` — add SoulCheckIn type, Notes type, lastKnownDate field
- `App.tsx` — ErrorBoundary wrapper, font error logging
- `.gitignore` — add `.env`, Firebase service files
- `package.json` — add `@react-native-community/netinfo`, `expo-secure-store`, `expo-screen-capture` (optional)

### Files to delete:
- `app.json` (replaced by `app.config.ts`)

---

## EXECUTION ORDER

1. Phase 1 (security) first — these are ship-blockers
2. Phase 2 (bugs) second — these cause data loss or crashes
3. Phase 3 (hardening) third — these raise the quality bar
4. Phase 4 (verification) last — confirm everything works

Do NOT leave placeholder comments like `// TODO` or `// FIXME`. Every fix must be fully implemented. If you need information from the user (like the real Firebase App ID), add a clear runtime guard that throws an informative error in dev mode.

---

## PHASE 5 — VIBE-CODING SECURITY REMEDIATION

> This app was scaffolded using AI (vibe coding). Research shows 45% of AI-generated code contains OWASP Top-10 vulnerabilities (Veracode GenAI Code Security Report 2025), and vulnerabilities increase by 37% after 5+ iteration rounds. This phase specifically targets the patterns AI code generators get wrong.

### 5.1 Dependency Audit & Supply Chain Security

**Problem:** AI-generated scaffolds frequently pull in outdated or vulnerable packages without checking. The current `package.json` has 20+ dependencies that were chosen by the AI and have never been audited.

**Fix:**
- Run `npm audit` and fix ALL vulnerabilities flagged as high or critical.
- Run `npx expo-doctor` to check for Expo SDK compatibility issues.
- Review every dependency in `package.json` — confirm each one is actually used in the source code. Remove any unused packages.
- Add an `npm audit` step to any CI pipeline or pre-commit hook.
- Pin dependency versions (remove `^` prefixes) for all security-sensitive packages: `firebase`, `@react-native-async-storage/async-storage`, `react-native-screens`.
- Check that no dependency was published fewer than 6 months ago with fewer than 1,000 weekly downloads — these are supply chain attack vectors.

### 5.2 Migrate Sensitive Storage from AsyncStorage to expo-secure-store

**Problem:** AsyncStorage stores data as unencrypted plain text on disk. On a rooted/jailbroken device, any app can read it. Firebase Auth tokens persisted via `getReactNativePersistence(AsyncStorage)` are exposed. This is a **critical** mobile security violation.

**Fix:**
- Install `expo-secure-store`.
- Create a `src/utils/secureStorage.ts` module that wraps `expo-secure-store` with the same `getItem/setItem/removeItem` API as AsyncStorage.
- In `src/utils/firebase.ts`, replace `getReactNativePersistence(AsyncStorage)` with a custom persistence adapter that uses SecureStore for auth tokens. Firebase JS SDK allows custom persistence implementations — create one backed by SecureStore.
- Non-sensitive data (task state, history, routine) can stay in AsyncStorage. Only auth tokens and profile credentials need SecureStore.
- Document which keys go where so future developers don't accidentally store secrets in AsyncStorage.

### 5.3 Jailbreak / Root Detection

**Problem:** On rooted Android or jailbroken iOS devices, attackers can inspect AsyncStorage, intercept network traffic, modify app memory, and bypass security controls. The app currently has zero device integrity checks.

**Fix:**
- Install `expo-device` (already in Expo SDK) and a jailbreak detection library compatible with Expo managed workflow.
- Create a `src/utils/deviceIntegrity.ts` module that checks:
  - Whether the device is rooted/jailbroken (using common detection heuristics).
  - Whether the app is running in a debugger.
  - Whether the app binary has been tampered with.
- On detection: do NOT crash the app or block it entirely (this frustrates legitimate users with custom ROMs). Instead:
  - Log a warning.
  - Disable cloud sync (don't send data to/from Firestore on compromised devices).
  - Show a subtle banner: "Device security check failed — cloud sync disabled."
- This is a proportionate response for a productivity app — save the hard blocks for banking apps.

### 5.4 Binary Obfuscation & ProGuard/R8

**Problem:** AI-generated code ships with no obfuscation. Anyone can decompile the APK/IPA and read the full source, extract API keys, understand the XP/gamification logic, and craft exploits.

**Fix:**
- Install `expo-build-properties`.
- In `app.config.ts`, enable ProGuard/R8 for Android release builds:
  ```typescript
  plugins: [
    ["expo-build-properties", {
      android: {
        enableProguardInReleaseBuilds: true,
        enableShrinkResourcesInReleaseBuilds: true,
      },
      ios: {
        // iOS uses Bitcode/Swift obfuscation via Xcode settings
      },
    }],
  ],
  ```
- For JavaScript bundle protection, enable Hermes (already default in Expo SDK 54) — Hermes bytecode is significantly harder to reverse-engineer than plain JS bundles.
- Verify Hermes is enabled: in `app.config.ts`, confirm `jsEngine: "hermes"` (or that it's not explicitly set to "jsc").

### 5.5 OWASP Mobile Top 10 Sweep

Run through each OWASP Mobile Top 10 (2024) category against the codebase and fix anything found:

| # | Category | PILLAR Status | Action Required |
|---|----------|--------------|-----------------|
| M1 | Improper Credential Usage | Firebase API key in source | Fixed in Phase 1.1 (env vars) |
| M2 | Inadequate Supply Chain Security | Unaudited deps | Fixed in Phase 5.1 |
| M3 | Insecure Authentication | No input validation, no rate limiting | Fixed in Phase 1.2 |
| M4 | Insufficient Input/Output Validation | Cloud data written without validation | Fixed in Phase 1.4 |
| M5 | Insecure Communication | HTTPS via Firebase (OK), but no cert pinning | Add cert pinning per Phase 3.4 |
| M6 | Inadequate Privacy Controls | User email visible in SignInScreen success state | Mask email: show first 2 chars + `***@domain` |
| M7 | Insufficient Binary Protections | No obfuscation | Fixed in Phase 5.4 |
| M8 | Security Misconfiguration | app.json wrong values, no bundle ID | Fixed in Phase 2.4 |
| M9 | Insecure Data Storage | Auth tokens in AsyncStorage | Fixed in Phase 5.2 |
| M10 | Insufficient Cryptography | N/A (Firebase handles encryption) | No action needed |

For **M6 specifically** — in `SignInScreen.tsx`, the success state displays `user.displayName || user.email` in full. Mask the email address:
```typescript
const maskedEmail = user.email
  ? user.email.substring(0, 2) + '***@' + user.email.split('@')[1]
  : '';
```

### 5.6 Automated Security Scanning Setup

**Problem:** Without automated scanning, every future code change could reintroduce vulnerabilities. AI-assisted development makes this worse because each iteration can introduce new flaws.

**Fix:**
- Create a `scripts/security-check.sh` script that runs:
  ```bash
  #!/bin/bash
  set -e
  echo "=== Dependency Audit ==="
  npm audit --audit-level=high
  echo "=== TypeScript Strict Check ==="
  npx tsc --noEmit
  echo "=== Secrets Scan ==="
  grep -rn "AIzaSy\|firebase.*apiKey\|-----BEGIN\|password.*=.*['\"]" src/ --include="*.ts" --include="*.tsx" && echo "WARNING: Possible hardcoded secrets found!" && exit 1 || echo "No hardcoded secrets found."
  echo "=== All checks passed ==="
  ```
- Add this to `package.json` scripts: `"security-check": "bash scripts/security-check.sh"`
- Add a note in the README: run `npm run security-check` before every commit.

### 5.7 Rate Limiting & Brute Force Protection

**Problem:** The SignInScreen has no rate limiting. An attacker could automate thousands of sign-in attempts. While Firebase has its own rate limiting, the client should also protect itself.

**Fix:**
- In `SignInScreen.tsx`, add a failed-attempt counter using `useRef`.
- After 5 consecutive failed attempts, disable the form for 30 seconds with a countdown timer displayed to the user.
- After 10 consecutive failed attempts, disable for 5 minutes.
- Reset the counter on successful sign-in.
- This protects the UX even if Firebase's server-side rate limiting hasn't kicked in yet.

---

## UPDATED EXECUTION ORDER

1. **Phase 1** (critical security) — ship-blockers
2. **Phase 5** (vibe-coding remediation) — because this code was AI-generated, treat it as untrusted third-party code
3. **Phase 2** (bugs) — data loss and crash prevention
4. **Phase 3** (hardening) — production quality bar
5. **Phase 4** (verification) — confirm everything works

## UPDATED FILE MANIFEST — Additional new files:

- `src/utils/secureStorage.ts` (SecureStore wrapper)
- `src/utils/deviceIntegrity.ts` (jailbreak/root detection)
- `scripts/security-check.sh` (automated scanning)

## UPDATED FILES TO MODIFY — Additional:

- `src/utils/firebase.ts` — SecureStore-backed auth persistence
- `src/screens/onboarding/SignInScreen.tsx` — email masking, rate limiting
- `app.config.ts` — ProGuard, expo-build-properties plugin
- `package.json` — add `expo-secure-store`, `expo-build-properties`, `expo-device`; add `security-check` script
