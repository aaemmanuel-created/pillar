# PILLAR by PVA — Build & Ship to TestFlight Prompt

Paste this into a fresh Claude Code session inside `PVA/PILLAR by PVA/` when you're ready to ship a new build.

---

You are shipping a new TestFlight build of **PILLAR by PVA**, the Christian discipleship companion app. The codebase lives in `pillar-app/` (Expo SDK 54 + React Native + TypeScript). Your job is to take it from "code on disk" to "build available in TestFlight on Emmanuel's phone."

## What's already set up — don't redo

- **EAS account:** `aaemmanuel` (`eanieakwetey.eaa@gmail.com`). Should already be logged in. Check with `npx eas-cli whoami`.
- **EAS project:** `@aaemmanuel/pillar-by-pva`, ID `3c3ee6a2-b962-4f19-aff8-64afd514c551` — already wired into `pillar-app/app.config.ts`.
- **Bundle ID:** `app.pillarbypva.pillar` (iOS + Android). This is unique — no collision with RENEW (`com.pva.renew`) or BURDEN (`app.burdenbypva.burden`).
- **Git:** `pillar-app/` is already a git repo. EAS is happy with this.
- **`.env`:** present in `pillar-app/` with Firebase + Sentry public env vars (`EXPO_PUBLIC_*`).
- **App icons:** present in `pillar-app/assets/`.
- **Apple Developer + ASC API key:** Emmanuel has both from the BURDEN ship. Same `.p8` key works for PILLAR.

## What might NOT be set up — verify before building

1. **Apple Developer bundle ID registration.** PILLAR's bundle ID `app.pillarbypva.pillar` may or may not be registered yet at https://developer.apple.com/account/resources/identifiers. If this is the **first ever PILLAR build**, Emmanuel needs to register it (5 min, web UI):
   - "+" → App IDs → App
   - Description: `PILLAR by PVA`
   - Bundle ID (Explicit): `app.pillarbypva.pillar`
   - Capabilities: leave defaults
   - Continue → Register

2. **App Store Connect app entry.** PILLAR needs a dedicated entry in App Store Connect. **This is the step that bit BURDEN** — without it, `--auto-submit` routes the .ipa to whichever existing app entry happens to match, and the build appears under the wrong name in TestFlight. Emmanuel must do this in https://appstoreconnect.apple.com → My Apps → "+" → New App:
   - Platforms: iOS
   - Name: `PILLAR by PVA`
   - Primary language: English (UK)
   - Bundle ID: pick `app.pillarbypva.pillar` from the dropdown (only appears if step 1 was done)
   - SKU: `pillar-by-pva-001`
   - User Access: Full Access
   - Click Create

3. **EAS credentials for `app.pillarbypva.pillar`.** First build will trigger EAS to provision a distribution certificate + provisioning profile for this bundle ID. This is interactive (Apple ID + 2FA on Emmanuel's trusted device). Cannot be done from a non-interactive session.

**Stop and ask Emmanuel** before proceeding if you can't confirm steps 1 and 2 are done. Don't run the build until both exist — it'll either fail at submit or land in the wrong app.

## Pre-flight checks (you can do these without Emmanuel)

Run from `pillar-app/`:

```bash
npx eas-cli whoami                                  # confirm aaemmanuel
npx eas-cli project:info                            # confirm pillar-by-pva project loads
git status                                          # working tree must be clean — EAS uses git HEAD
npm run typecheck                                   # tsc --noEmit must pass
npm run security-check                              # PILLAR has its own security script
npm test                                            # jest suite must pass
```

If any of these fail, fix the underlying issue before building. Don't ship a broken or untyped build to TestFlight.

## The build

When pre-flight is green AND Emmanuel has confirmed the App Store Connect entry exists:

```bash
cd pillar-app
npx eas-cli build --platform ios --profile production --auto-submit
```

This is interactive on first run — Apple ID, 2FA, signing cert provisioning. After the first successful build, subsequent runs reuse credentials and don't prompt.

Build runs in EAS cloud (~10–25 min for PILLAR — it has more native deps than BURDEN: Reanimated, Sentry native, Secure Store, etc).

## Watch the auto-submit prompt carefully

When the build finishes and submit kicks off, EAS shows a confirmation: "Submitting build X to App Store Connect app Y." The app name should read **PILLAR by PVA**. If it says RENEW, BURDEN, or anything else, **abort immediately** (Ctrl+C before the upload completes — usually about 30 seconds of grace) and figure out why the bundle ID resolved to the wrong app.

## Verification — the build is live in PILLAR's TestFlight

1. App Store Connect → My Apps → **PILLAR by PVA** → TestFlight tab → Builds.
2. The new build appears in 5–10 min, status "Processing".
3. Processing takes 10–30 min. Emmanuel gets an email when it's ready.
4. To install on his phone: TestFlight tab → Internal Testing → his test group → Add Build → pick the new one. Then his TestFlight app on iPhone shows the install button within minutes.

## If the auto-submit fails (build succeeds, submit doesn't)

Common cause: ASC API key not configured for this app, or expired. Fix:

```bash
npx eas-cli submit --platform ios --latest
```

Pick PILLAR by PVA when prompted. The .ipa from the just-completed build is reused — no new build needed.

## What to report back

When done, tell Emmanuel:
- Build URL (printed by EAS at the end of the build)
- Submit URL (printed at end of submit)
- App Store Connect link to PILLAR's TestFlight builds page
- Build number (so he knows which one is fresh)

If the build failed before producing an .ipa, paste the last 30–40 lines of EAS output and stop — don't retry blindly.

---

**Coding standards reminder:** PILLAR is a real shipped app. Don't introduce backwards-compatibility shims, half-finished features, or "TODO: handle this later" comments in production code paths. If you find work that should be done but isn't in scope for this build, flag it as a follow-up — don't smuggle it into the ship build.
