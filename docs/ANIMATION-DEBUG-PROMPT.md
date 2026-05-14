# RENEW Animation Debug & Fix Prompt

## Context

RENEW is a React PWA (`src/Renew.jsx`, ~3140 lines) that uses HTML5 Canvas + `requestAnimationFrame` at 60fps. The animation was recently rewritten from a fluorescence-microscopy neural style to "Starfield Genesis" — a cosmic creation theme where neurons are rendered as stars, synapses as cosmic web filaments, and the background is a nebula with dust particles.

**The animation looks broken / "weird".** This prompt is a comprehensive audit of every known and likely bug in the current render pipeline, with exact fixes.

---

## BUG 1: Trail alpha too low — canvas never fully clears, creates smearing

**Location:** `// ─── RENDER (Concept D: Starfield Genesis) ───` section

**Problem:** `trailAlpha` is `0.025` when speaking and `0.08` when silent. At 0.025, the canvas takes ~40 frames (0.67s) to fade to 50% opacity. Old frames persist as visible ghosting. Every element leaves a trail of smeared copies behind it as it moves. This makes the entire canvas look muddy, washed out, and "wrong."

**Why it looks weird:** Stars appear to have huge blurry streaks. Particles leave persistent trails. Nebula clouds never disappear. The whole canvas has a dirty, smudgy appearance instead of clean deep space black.

**Fix:** Increase trail alpha significantly. For a starfield, the background should be clean and dark:
```js
// OLD:
const trailAlpha = isSessionScreen && spk ? 0.025 : 0.08;
// NEW:
const trailAlpha = isSessionScreen && spk ? 0.12 : 0.20;
```

---

## BUG 2: Background starfield flickers — only renders every 3rd frame

**Location:** `// ── Layer 1: Fixed background starfield ──`

**Problem:** The starfield is inside `if (shimmerTimerRef.current % 3 === 0)` — it only renders 1 out of every 3 frames. But because each frame draws a semi-transparent black rect over the canvas (the trail), the stars are being faded 2 out of 3 frames and only redrawn 1 out of 3. This creates a constant flicker where stars appear dim → dimmer → bright → dim → dimmer → bright.

**Fix:** Remove the `% 3` optimization gate. 80 `fillRect(x,y,1,1)` calls is trivial — no perf savings justify visible flicker:
```js
// OLD:
if (shimmerTimerRef.current % 3 === 0) {
  for (let i = 0; i < 80; i++) { ... }
}
// NEW:
for (let i = 0; i < 80; i++) { ... }
```

---

## BUG 3: Nebula clouds use wrong alpha format — appears invisible or garbled

**Location:** `// ── Layer 2: Nebula gas clouds ──`

**Problem:** The third nebula cloud uses a hardcoded partial RGBA string `rgba(120, 80, 180, ` — this is correct format for the `${nc.color}${alpha})` pattern. But the `spc.fogRGB` and `spc.deepRGB` strings already have the format `rgba(r, g, b, ` from the `getPillarCached()` function. So the first two clouds are correct. However, the `nebulaIntensity` values are very low: `0.006` in silence, and `0.015 + vol * 0.035` when speaking. At vol=0.5, that's only `0.0325`. Multiplied by 1.2 at the center stop, that's `0.039`. This is essentially invisible.

**Fix:** Increase nebula opacity so the nebula is actually visible:
```js
// OLD:
const nebulaIntensity = isSessionScreen && spk ? 0.015 + vol * 0.035 : 0.006;
// NEW:
const nebulaIntensity = isSessionScreen && spk ? 0.04 + vol * 0.08 : 0.015;
```

---

## BUG 4: Particle system still labeled "neural dust" — uses attraction behavior that doesn't fit cosmic theme

**Location:** Particle initialization (`// Initialize neural dust particle system`)

**Problem:** The particles still have `attracted: false` and `homeX/homeY` fields from the Concept E "neural dust" design. While the update loop was changed to use swirl physics, the initialization still creates 200 particles with very low opacity (`0.015 + random * 0.035` → max 0.05). At this opacity, with the depth scaling (`0.3 + depth * 0.7`), the farthest particles are at opacity `0.015 * 0.3 = 0.0045` — completely invisible. Most particles are invisible.

**Fix:** Increase particle opacity range and count:
```js
// OLD:
opacity: 0.015 + Math.random() * 0.035,
// NEW:
opacity: 0.03 + Math.random() * 0.06,
```

---

## BUG 5: Neuron maturity ramp-up is painfully slow — stars appear invisible for ~80 seconds

**Location:** Physics update loop: `n.maturity = Math.min(1, n.maturity + 0.0002);`

**Problem:** New neurons start with `maturity = 0`. At `+0.0002` per frame (60fps), maturity reaches 0.5 at ~4167 frames = ~69 seconds. Every visual element is multiplied by `n.maturity` — the radius, the dendrite alpha, the diffraction spikes, the star core gradient. For the first 30+ seconds, a new star is essentially invisible. The user speaks, a new neuron spawns, but they see nothing because maturity is 0.01.

For a cosmic theme where a star should IGNITE dramatically, this slow fade-in is the opposite of the intended "stellar ignition" effect.

**Fix:** Make stars appear much faster — ignition should be dramatic:
```js
// OLD:
n.maturity = Math.min(1, n.maturity + 0.0002);
// NEW:
n.maturity = Math.min(1, n.maturity + 0.002);  // ~8 seconds to full brightness
```

---

## BUG 6: Diffraction spikes are too faint — the signature star effect is barely visible

**Location:** `// ── Diffraction spikes (4-pointed star cross) ──`

**Problem:** `spikeOp = (0.08 + fire * 0.2 + energy * 0.06) * n.maturity * lum`. With a resting star (fire=0, energy=0.15, maturity=1, lum=0.575): `(0.08 + 0 + 0.009) * 1 * 0.575 = 0.051`. The spike is drawn at 5% opacity with lineWidth 0.8 — this is invisible on most screens. The diffraction spikes are THE visual signature of a star, and they're too faint to see.

**Fix:** Boost spike opacity and width:
```js
// OLD:
const spikeOp = (0.08 + fire * 0.2 + energy * 0.06) * n.maturity * lum;
// ...
ctx.lineWidth = 0.8;
// NEW:
const spikeOp = (0.15 + fire * 0.35 + energy * 0.12) * n.maturity * lum;
// ...
ctx.lineWidth = 1.2;
```

Also boost diagonal spikes:
```js
// OLD:
const diagOp = spikeOp * 0.4;
ctx.lineWidth = 0.5;
// NEW:
const diagOp = spikeOp * 0.5;
ctx.lineWidth = 0.7;
```

---

## BUG 7: Stellar halo is massive but dim — looks like a fuzzy blob, not a star

**Location:** `// ── Wide stellar halo ──`

**Problem:** `haloR = (r + 30 + fire * 50 + energy * 25) * breathMod`. With r=7, fire=0, energy=0.15: `(7 + 30 + 0 + 3.75) * ~1.08 = ~44px radius`. But the alpha at the center is only `energy * 0.18 * lum = 0.15 * 0.18 * 0.575 = 0.0155`. A 44px radius circle at 1.5% opacity is an enormous, nearly invisible blob. It makes stars look like huge smudges.

**Fix:** Reduce halo radius, increase center brightness:
```js
// OLD:
const haloR = (r + 30 + fire * 50 + energy * 25) * breathMod;
// NEW:
const haloR = (r + 15 + fire * 30 + energy * 12) * breathMod;
```
And boost halo alpha:
```js
// OLD (resting):
halo.addColorStop(0, `${pc.brightRGB}${energy * 0.18 * lum})`);
// NEW:
halo.addColorStop(0, `${pc.brightRGB}${energy * 0.35 * lum})`);
```

---

## BUG 8: Dendrites (stellar corona filaments) are essentially invisible

**Location:** `// ── Stellar corona / gravitational arms ──`

**Problem:** `baseAlpha = (0.06 + energy * 0.12 + fire * 0.1) * n.maturity`. With energy=0.15, fire=0, maturity=1: `0.06 + 0.018 + 0 = 0.078`. Then the filament glow is drawn at `baseAlpha * 0.1 = 0.0078` opacity. The main filament is drawn at `rgba(140, 160, 220, 0.078)`. On a black background, these are completely invisible. The dendrites add NO visible structure around the star.

**Fix:** Boost dendrite visibility:
```js
// OLD:
const baseAlpha = (0.06 + energy * 0.12 + fire * 0.1) * n.maturity;
// NEW:
const baseAlpha = (0.12 + energy * 0.25 + fire * 0.2) * n.maturity;
```
And boost the glow halo:
```js
// OLD:
ctx.strokeStyle = `rgba(120, 140, 200, ${baseAlpha * 0.1 * axonBoost})`;
// NEW:
ctx.strokeStyle = `rgba(120, 140, 200, ${baseAlpha * 0.2 * axonBoost})`;
```

---

## BUG 9: Star core radius is tiny — stars look like dim dots

**Location:** `// ── Star core — bright white-hot center ──`

**Problem:** The star core is drawn with radius `r * 1.2`. With a typical `r` of 7 (from `radius=7, maturity=1, pulse=0.94`): the core circle is only 8.4px radius. On a 390px wide iPhone screen at 3x DPR, this is about 2.8 logical pixels. Combined with the maturity ramp and low alpha values, the star is barely a speck.

**Fix:** Increase the core rendering radius:
```js
// OLD:
ctx.beginPath(); ctx.arc(n.x, n.y, r * 1.2, 0, Math.PI * 2); ctx.fill();
// NEW:
ctx.beginPath(); ctx.arc(n.x, n.y, r * 1.8, 0, Math.PI * 2); ctx.fill();
```
Also update the gradient to match:
```js
// OLD:
const starGrad = ctx.createRadialGradient(n.x, n.y, 0, n.x, n.y, r * 1.2);
// NEW:
const starGrad = ctx.createRadialGradient(n.x, n.y, 0, n.x, n.y, r * 1.8);
```

---

## BUG 10: Shooting stars use a linear gradient as strokeStyle — doesn't work on all browsers

**Location:** `// ── Layer 8: Occasional shooting stars ──`

**Problem:** `ctx.strokeStyle = ssGrad` where `ssGrad` is a `createLinearGradient`. While this technically works in most browsers, it can appear as a solid color or invisible line on some mobile Safari versions. The gradient direction and stroke direction must match perfectly.

**Fix:** Draw the shooting star as a filled shape instead of a stroked line, or use a simple opacity fade:
```js
// Simpler approach: just use a solid white line that fades with lineWidth
ctx.globalAlpha = 0.3;
ctx.strokeStyle = "rgba(220, 230, 255, 1)";
ctx.lineWidth = 1;
ctx.lineCap = "round";
ctx.beginPath();
ctx.moveTo(ssX, ssY);
ctx.lineTo(ssX + Math.cos(ssAngle) * ssLen, ssY + Math.sin(ssAngle) * ssLen);
ctx.stroke();
ctx.globalAlpha = 1;
```

---

## Summary: Root cause of "looks weird"

The animation appears broken primarily because of **Bug 1** (trail alpha too low = smearing) and the cumulative effect of **Bugs 3-9** (everything is too dim). The cosmic theme needs:

1. **Clean dark background** — higher trail alpha for crisp deep space
2. **Visible nebula** — higher nebula intensity values
3. **Bright stars** — faster maturity, larger cores, brighter halos
4. **Visible spikes** — thicker, more opaque diffraction crosses
5. **Visible structure** — brighter dendrite filaments

The previous neural microscopy theme worked with low opacities because it used multiple overlapping glows on a black background. The cosmic theme needs more contrast: bright stars against clean dark space.

---

## How to use this prompt

Paste this to Claude and say:

> "Apply all 10 fixes from this debug audit to Renew.jsx. The changes are all in the render loop section (approximately lines 1620-1960). Only change the specific values identified — do not restructure the code."
