# PILLAR by PVA

A gamified Christian daily discipline Progressive Web App. Live at **[pillarbypva.app](https://pillarbypva.app)**.

This folder is the development root. The deployable site is in `pva-deploy/` — that's the folder GitHub Pages serves.

## Quick Start (Claude Code)

From a terminal, in this folder:

```bash
cd "PILLAR by PVA"
claude
```

Claude Code will pick up `CLAUDE.md` automatically and load the project context.

## Run Locally

The app has no build step — it's a single-file React app compiled in the browser by Babel Standalone.

```bash
cd pva-deploy
python3 -m http.server 8000
```

Then open <http://localhost:8000> in a browser.

## Deploy

1. Make changes inside `pva-deploy/`.
2. **Bump the cache version** in `pva-deploy/sw.js` (e.g. `pva-cache-v34` → `v35`). The service worker won't ship updates otherwise.
3. Commit and push to the GitHub Pages branch.
4. GitHub Pages rebuilds; `pillarbypva.app` updates within a minute or two.

## Structure

```
PILLAR by PVA/
├── CLAUDE.md           Project context for Claude Code — read this first
├── README.md           This file
├── .gitignore
└── pva-deploy/         The actual deployable site
    ├── index.html      ~10,400-line single-file React app
    ├── sw.js           Service worker (bump cache version on every change!)
    ├── manifest.json   PWA manifest
    ├── icon-192.png    Homescreen icon
    ├── icon-512.png    Splash icon
    └── CNAME           pillarbypva.app
```

## Key Constraints

- No build step, no npm, no bundlers. React via CDN UMD, JSX via Babel Standalone.
- All app code is in one `index.html`. Do not split into modules.
- Bump `pva-cache-v<N>` in `sw.js` whenever you change `index.html`.
- `.app` TLD requires HTTPS (already enforced by GitHub Pages).

See `CLAUDE.md` for architecture, known bug-fix regressions to avoid, and file-section line references.
