# PILLAR — Converting to iOS & Android with Capacitor

## What is Capacitor?

Capacitor (by the Ionic team) wraps your existing web app in a native shell. Your `index.html` runs inside a WebView, but you get access to native APIs (push notifications, camera, biometrics, etc.) and can submit to the App Store and Google Play.

**Why Capacitor for PILLAR:**
- Your entire app is a single `index.html` — Capacitor works perfectly with this
- No rewrite needed — your React 18 UMD/Babel setup stays exactly as-is
- Firebase Auth and Firestore continue to work inside the WebView
- You keep one codebase for web + iOS + Android

---

## Prerequisites

### For Both Platforms
- **Node.js** (v18+): https://nodejs.org
- **npm** (comes with Node)

### For Android
- **Android Studio**: https://developer.android.com/studio
- **Java 17+** (bundled with Android Studio)
- **Google Play Developer Account**: $25 one-time fee → https://play.google.com/console

### For iOS
- **macOS** (required — you cannot build iOS apps on Windows/Linux)
- **Xcode** (free from Mac App Store)
- **Apple Developer Program**: $99/year → https://developer.apple.com/programs
- **CocoaPods**: `sudo gem install cocoapods`

---

## Step-by-Step Setup

### 1. Initialize the Project

Create a new folder for the Capacitor project (separate from your GitHub Pages repo):

```bash
mkdir pillar-native
cd pillar-native
npm init -y
```

### 2. Install Capacitor

```bash
npm install @capacitor/core @capacitor/cli
npx cap init
```

When prompted:
- **App name:** PILLAR
- **App Package ID:** `app.pillarbypva.pillar` (reverse domain notation)
- **Web asset directory:** `www`

This creates a `capacitor.config.ts` file.

### 3. Copy Your Web App

```bash
mkdir www
cp /path/to/pva-deploy/index.html www/
cp /path/to/pva-deploy/manifest.json www/
cp /path/to/pva-deploy/icon-192.png www/
cp /path/to/pva-deploy/icon-512.png www/
```

**Important:** You do NOT need `sw.js` for the native app (service workers don't apply inside Capacitor's WebView).

### 4. Update capacitor.config.ts

```typescript
import type { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'app.pillarbypva.pillar',
  appName: 'PILLAR',
  webDir: 'www',
  server: {
    // Use the bundled web assets (not a remote URL)
    androidScheme: 'https',
  },
  plugins: {
    SplashScreen: {
      launchShowDuration: 2000,
      backgroundColor: '#060810',
      showSpinner: false,
    },
    StatusBar: {
      style: 'dark',
    },
  },
};

export default config;
```

### 5. Add Platforms

```bash
# Android
npm install @capacitor/android
npx cap add android

# iOS
npm install @capacitor/ios
npx cap add ios
```

This creates `android/` and `ios/` folders with native project files.

### 6. Sync Web Assets to Native Projects

Run this every time you update `index.html`:

```bash
npx cap sync
```

This copies `www/` into both native projects and updates native dependencies.

---

## Small Code Changes Needed in index.html

### 1. Remove Service Worker Registration

Inside `index.html`, find and comment out (or wrap in a check) the service worker registration:

```javascript
// Before:
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
}

// After:
if ('serviceWorker' in navigator && !window.Capacitor) {
  navigator.serviceWorker.register('/sw.js');
}
```

### 2. Fix Firebase Auth for Capacitor

Google Sign-In via redirect doesn't work inside a WebView. You'll need to either:

**Option A — Use the Capacitor Firebase Auth plugin (recommended):**

```bash
npm install @capacitor-firebase/authentication
```

Then configure it to handle native Google Sign-In. See: https://github.com/capawesome-team/capacitor-firebase/tree/main/packages/authentication

**Option B — Use email/password auth only in native:**

Your app already supports email/password auth, so you could simply hide the Google Sign-In button when running inside Capacitor:

```javascript
// Detect Capacitor
const isNative = !!window.Capacitor;

// In your sign-in UI, conditionally show Google button
if (!isNative) {
  // Show Google Sign-In button
}
```

### 3. Handle Safe Areas (notch/dynamic island on iOS)

Add this CSS to your global `<style>` block:

```css
body {
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}
```

### 4. Detect Native vs Web

```javascript
// Add near top of your script
const IS_NATIVE = typeof window !== 'undefined' && !!window.Capacitor;
const IS_IOS = IS_NATIVE && window.Capacitor.getPlatform() === 'ios';
const IS_ANDROID = IS_NATIVE && window.Capacitor.getPlatform() === 'android';
```

---

## App Icons & Splash Screens

### Android

Place icons in `android/app/src/main/res/`:
- `mipmap-mdpi/ic_launcher.png` (48x48)
- `mipmap-hdpi/ic_launcher.png` (72x72)
- `mipmap-xhdpi/ic_launcher.png` (96x96)
- `mipmap-xxhdpi/ic_launcher.png` (144x144)
- `mipmap-xxxhdpi/ic_launcher.png` (192x192)

Or use the **@capacitor/assets** tool to auto-generate all sizes:

```bash
npm install -D @capacitor/assets
npx capacitor-assets generate --iconBackgroundColor '#060810' --splashBackgroundColor '#060810'
```

Place your source icon as `assets/icon-only.png` (1024x1024) and splash as `assets/splash.png` (2732x2732) before running.

### iOS

Same tool generates iOS assets, or manually place a 1024x1024 icon in `ios/App/App/Assets.xcassets/AppIcon.appiconset/`.

---

## Building & Running

### Android

```bash
# Open in Android Studio
npx cap open android

# Or build from command line
cd android
./gradlew assembleDebug    # Debug APK
./gradlew assembleRelease  # Release APK (needs signing)
```

The APK will be at `android/app/build/outputs/apk/debug/app-debug.apk`.

To test on a physical device: enable USB debugging on your phone, connect via USB, and click Run in Android Studio.

### iOS

```bash
# Open in Xcode
npx cap open ios
```

In Xcode:
1. Select your development team (Apple Developer account)
2. Select your device or simulator
3. Click the Run button (▶)

---

## Publishing to Stores

### Google Play Store

1. **Generate a signed APK/AAB:**
   ```bash
   cd android
   ./gradlew bundleRelease
   ```
   You'll need to create a keystore first:
   ```bash
   keytool -genkey -v -keystore pillar-release.keystore -alias pillar -keyalg RSA -keysize 2048 -validity 10000
   ```

2. **Go to Google Play Console:** https://play.google.com/console
3. Create a new app → fill in listing details (screenshots, description, etc.)
4. Upload the AAB file from `android/app/build/outputs/bundle/release/`
5. Submit for review (usually takes a few hours to a few days)

### Apple App Store

1. In Xcode: **Product → Archive**
2. In the Organizer window: **Distribute App → App Store Connect**
3. Go to **App Store Connect:** https://appstoreconnect.apple.com
4. Create a new app → fill in metadata, screenshots, description
5. Select the build you uploaded → Submit for review (takes 1–3 days typically)

**Required for App Store:**
- Privacy policy URL (can be a simple page on pillarbypva.app)
- App screenshots for various device sizes
- App description, keywords, category (Lifestyle or Productivity)

---

## Useful Capacitor Plugins

```bash
# Status bar control (colour, style)
npm install @capacitor/status-bar

# Haptic feedback (for task completion taps)
npm install @capacitor/haptics

# Local notifications (daily reminders)
npm install @capacitor/local-notifications

# Push notifications
npm install @capacitor/push-notifications

# App badge (show incomplete task count)
npm install @capacitor/badge

# Share (share progress/testimony)
npm install @capacitor/share

# Keep screen awake (Focus mode — replaces Wake Lock API)
npm install @capacitor/keep-awake
```

---

## Development Workflow

```
Edit index.html (in pva-deploy/)
        │
        ├──→ Push to GitHub ──→ Live on pillarbypva.app (web/PWA)
        │
        └──→ Copy to www/ ──→ npx cap sync ──→ Build native app
```

You maintain ONE codebase (`index.html`) that deploys everywhere.

---

## Cost Summary

| Item | Cost |
|------|------|
| Google Play Developer | $25 one-time |
| Apple Developer Program | $99/year |
| Domain (pillarbypva.app) | ~$15/year |
| Firebase (Spark plan) | Free |
| **Total Year 1** | **~$139** |
| **Total subsequent years** | **~$114/year** |

---

## Next Steps

1. Set up the Capacitor project locally
2. Get Android running first (easier, cheaper)
3. Test Firebase auth flows in the WebView
4. Generate app icons from your existing SVG
5. Create store listing assets (screenshots, description)
6. Submit to Google Play
7. Then tackle iOS (requires Mac + Apple Developer account)
