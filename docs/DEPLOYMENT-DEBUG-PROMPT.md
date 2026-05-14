# RENEW — Deployment Pipeline Debug & Fix Prompt

## The Problem

"When I open the app on my iPhone, changes aren't loading and fixes aren't being applied." Despite uploading new code via GitHub web UI, the live app at `https://aaemmanuel-created.github.io/renew/` appears stuck on an old version.

---

## Investigation Results (April 3, 2026)

### What I Checked and What I Found

#### 1. GitHub Commits — ARE Real (Not Empty)

Every recent commit contains actual file changes:

| Commit | Date | Files Changed | Diff | Build |
|--------|------|---------------|------|-------|
| `11f6202` | Apr 3, 13:28 | 1 file (Renew.jsx) | +11 / -4 | PASS (2/2) |
| `f6aca53` | Apr 3 | 1 file (Renew.jsx) | +262 / -343 | PASS (2/2) |
| `318ed22` | Apr 3 | 1 file (Renew.jsx) | +260 / -38 | PASS (2/2) |
| `11ab15d` | Apr 2 | 2 files | +509 / -73 | **FAIL (0/2)** |

The commits are NOT empty. The GitHub web upload process is working — files are being included.

#### 2. GitHub Actions — Builds Are Succeeding

53 total workflow runs. All recent runs complete in ~45-55 seconds with passing status (2/2), except commit `11ab15d` which failed (0/2). The deployment workflow ("Deploy to GitHub Pages") is running and succeeding on the latest commits.

#### 3. Live Site — IS Serving the Latest Build

The live site at `aaemmanuel-created.github.io/renew/` loads bundle `index-jlh1-7Jx.js`. A fresh fetch from the server (bypassing cache) also returns `index-jlh1-7Jx.js`. The server IS serving the latest code.

#### 4. Service Worker — Self-Destructed Successfully (on Desktop Chrome)

The self-destruct `sw.js` worked: `navigator.serviceWorker.getRegistrations()` returns an empty array — no active service workers. The self-destruct SW in `public/sw.js` does: install → skipWaiting → activate → clear caches → unregister → reload clients.

**However:** The `renew-v1` cache STILL EXISTS in the browser, containing old assets like `index-DnU3BpMJ.js`, `index-wr5TASbP.js`, `index-logbhfZz.js`, etc. The SW unregistered itself but the cache data persists as orphaned storage.

#### 5. main.jsx — Has Defenses, But They Have Gaps

The `main.jsx` file has two defenses:
- **SW Unregister:** Calls `navigator.serviceWorker.getRegistrations()` and unregisters all SWs
- **Cache-Bust Check:** After 2 seconds, fetches fresh `index.html` with `cache: 'no-store'`, compares bundle hash, and reloads if different

These work IF the new main.jsx code is actually running. But if the browser is serving the OLD main.jsx from cache (which didn't have these defenses), they never execute.

---

## ROOT CAUSE: iOS Safari PWA Standalone Mode Caching

The `index.html` contains:
```html
<meta name="apple-mobile-web-app-capable" content="yes" />
```

This means when Emmanuel adds RENEW to his iPhone home screen and opens it, it runs in **standalone PWA mode**. iOS Safari's PWA mode has fundamentally different caching behavior from regular browser tabs:

### How iOS PWA Caching Breaks Updates

1. **iOS caches the entire app shell aggressively.** When a PWA is launched from the home screen, iOS loads the cached HTML, CSS, and JS from its own internal cache — NOT the service worker cache, and NOT the HTTP cache. This is a separate, iOS-managed cache.

2. **iOS only checks for PWA updates under specific conditions:**
   - The user must close the app completely (swipe it away from the app switcher)
   - Then wait (iOS checks periodically, but the interval is unpredictable — could be hours or days)
   - Then reopen the app
   - Even then, iOS may serve the cached version first and download updates in the background for the NEXT launch

3. **The service worker self-destruct can't run if the old code is cached.** If iOS is serving the old `index.html` which loaded the old `main.jsx` which registered the OLD caching service worker, the self-destruct `sw.js` never gets fetched because the old SW intercepts all requests.

4. **The cache-bust check in main.jsx can't run if main.jsx itself is cached.** It's a chicken-and-egg problem: the code that checks for updates is IN the update that isn't loading.

### The Vicious Cycle on iPhone

```
Old cached index.html loads
  → Old cached main.jsx runs (no cache-bust code)
    → Old cached Renew.jsx renders (old animation)
      → Old service worker intercepts all fetches
        → User sees stale app
          → Uploads new code to GitHub
            → GitHub builds & deploys new code
              → iPhone still loads old cached version
```

---

## FIXES

### Fix 1: Nuclear Cache Clear (Immediate — Do This on Your iPhone Right Now)

**For Safari (if using the app in the browser):**
1. Go to Settings → Safari → Advanced → Website Data
2. Search for "github.io"
3. Swipe left and delete ALL data for `aaemmanuel-created.github.io`
4. Go back to Safari and open `https://aaemmanuel-created.github.io/renew/`

**For Home Screen PWA (if you added RENEW to your home screen):**
1. Delete the RENEW icon from your home screen (long press → Remove App)
2. Open Safari and go to Settings → Safari → Clear History and Website Data → Clear History and Data
3. Open `https://aaemmanuel-created.github.io/renew/` in Safari
4. Verify the new version is working
5. THEN re-add to home screen (Share → Add to Home Screen)

### Fix 2: Add Cache-Busting to index.html Itself (Permanent Fix)

The `index.html` is the first file iOS loads. If we add cache-busting logic HERE (before any JS bundle loads), it can break the cycle. Add this inline script to `index.html`, BEFORE the module script:

```html
<body>
  <div id="root"></div>

  <!-- Cache-buster: runs before React loads -->
  <script>
    // Force-unregister any service workers
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.getRegistrations().then(function(regs) {
        regs.forEach(function(r) { r.unregister(); });
      });
      // Also delete all caches
      if (window.caches) {
        caches.keys().then(function(names) {
          names.forEach(function(name) { caches.delete(name); });
        });
      }
    }
  </script>

  <script type="module" src="/src/main.jsx"></script>
</body>
```

This ensures that even if iOS serves a cached `index.html`, the inline script will kill any lingering service workers and caches before the main bundle loads.

### Fix 3: Remove PWA Standalone Mode (Prevents Future iOS Cache Issues)

Remove or change these lines in `index.html`:

```html
<!-- REMOVE THIS LINE to prevent iOS standalone PWA caching: -->
<!-- <meta name="apple-mobile-web-app-capable" content="yes" /> -->

<!-- KEEP the manifest for installability prompt, but the app will open in Safari -->
<link rel="manifest" href="/renew/manifest.json" />
```

Without `apple-mobile-web-app-capable`, tapping the home screen icon opens the app in a regular Safari tab instead of standalone mode. Safari tabs have much more predictable caching behavior and respect HTTP cache headers properly.

**Trade-off:** The app won't be fullscreen anymore — it'll have the Safari URL bar. But updates will work reliably.

### Fix 4: Add a Version Indicator (Debug Aid)

Add a visible version string so you can instantly tell which version is running. In `Renew.jsx`, add to the landing/splash screen:

```jsx
// At the bottom of the splash screen, tiny gray text:
<div style={{
  position: 'fixed', bottom: 4, right: 8,
  fontSize: 9, color: '#333', fontFamily: 'monospace'
}}>
  v2026.04.03a
</div>
```

Update this string every time you upload. If you open the app and see the old version string, you know caching is the problem — not your code.

### Fix 5: Add No-Cache Headers via GitHub Pages (Belt-and-Suspenders)

Create a file `public/_headers` (some static hosts respect this):
```
/renew/index.html
  Cache-Control: no-cache, no-store, must-revalidate
  Pragma: no-cache
```

Note: GitHub Pages does NOT support custom `_headers` files natively. But you can add cache-busting via the Vite build config instead. In `vite.config.js`:

```js
export default defineConfig({
  base: '/renew/',
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        // Ensure unique filenames for every build
        entryFileNames: `assets/[name]-[hash].js`,
        chunkFileNames: `assets/[name]-[hash].js`,
        assetFileNames: `assets/[name]-[hash].[ext]`
      }
    }
  }
});
```

This is likely already the default with Vite, but worth confirming.

### Fix 6: Stop Using the Self-Destruct SW Pattern

The current approach of having `public/sw.js` as a self-destruct is clever but fragile. Instead, simply DON'T have a service worker at all. The `sw.js` in `public/` can be replaced with an empty file or removed entirely. The unregistration code in `main.jsx` is sufficient.

But if you DO want to keep the self-destruct as a safety net, move the cache-clearing into `main.jsx` as well:

```js
// In main.jsx — add cache clearing alongside SW unregister
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.getRegistrations().then(registrations => {
    registrations.forEach(r => r.unregister());
  });
}
// Clear orphaned caches too
if (window.caches) {
  caches.keys().then(names => {
    names.forEach(name => caches.delete(name));
  });
}
```

---

## Priority Order of Fixes

1. **Fix 1 (Nuclear Clear)** — Do this RIGHT NOW on your iPhone to see the latest version immediately
2. **Fix 2 (Inline cache-buster in index.html)** — Apply this to the code and upload. This is the most important permanent fix.
3. **Fix 4 (Version indicator)** — Add this so you can always tell which version is running
4. **Fix 6 (Clear caches in main.jsx)** — Quick code addition
5. **Fix 3 (Remove standalone PWA)** — Consider this if caching continues to be a problem
6. **Fix 5 (Vite config)** — Verify hash filenames are working (they likely already are)

---

## How to Use This Prompt

Paste this to Claude and say:

> "Apply Fixes 2, 4, and 6 from this deployment debug audit. Update index.html to add the inline cache-buster script, add a version indicator to the Renew.jsx splash screen, and add cache-clearing to main.jsx. Then give me the updated files to upload."

Then do Fix 1 manually on your iPhone to clear the old cache.
