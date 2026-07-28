# PRODUCTION BUILD SPEC
### Immersive WebGL Portfolio — Active Theory-led, Utsubo-informed
Built from real captured data (Playwright, both sites, chrome channel). Confidence tagged. Gaps stated, not papered over.

---

## 0. WHAT THIS DOCUMENT IS GROUNDED IN

| Data point | Source | Confidence |
|---|---|---|
| Colors, tokens, network manifest, canvas type | Direct capture, both sites | [Certain] |
| Utsubo full type scale | Direct capture (`styles.json`) | [Certain] |
| Active Theory hero type size/weight | NOT captured — canvas-rendered text, not DOM | [Guessing] where used |
| Motion timing/easing curves | Inferred from screenshots + one exposed Utsubo variable | [Likely]/[Guessing] |
| Shader source, Hydra engine internals | Not accessible to anyone outside Active Theory | Not attempted |

Corrected finding from raw data: total page weight is **not** why Active Theory feels heavier on your iPhone — Utsubo is actually 57.4MB vs Active Theory's 43.7MB, measured. [Certain] The smoothness gap is architectural: Utsubo renders WebGL on an `OffscreenCanvas` worker thread (confirmed: `webgl2: "offscreen"` in capture); Active Theory renders on the main thread via its custom "Hydra" engine (confirmed: `webgl2: true` on main-thread canvas, plus `hydra-thread.js` in network log — [Guessing] this may mean *some* work is threaded, but the primary render context is not offscreen). This single architectural choice is the highest-leverage decision in this entire document.

---

## 1. TECHNICAL STACK DECISION (fixed — do not let Claude Code substitute)

```
Renderer:        Three.js r16x, run inside an OffscreenCanvas + Web Worker
                  (mirrors Utsubo's confirmed architecture — this is the
                  single biggest lever for your smoothness complaint)
Fallback:         Main-thread Three.js render for browsers without
                  OffscreenCanvas support (Safari < 16.4, some older iOS)
Build:            Vite
Animation:        GSAP 3 + ScrollTrigger, driven via postMessage from
                  main thread to worker (scroll/pointer events captured
                  on main thread, forwarded to worker for camera control)
Smooth scroll:    Lenis (main thread only — DOM scroll, separate from
                  canvas worker)
Texture compress: KTX2 / Basis Universal (confirmed both sites use this
                  for their optimized assets — basis_transcoder.wasm)
Geometry:         glTF + Draco compression (confirmed Utsubo pattern:
                  *_draco.glb)
Post-processing:  `postprocessing` (pmndrs) — runs inside the worker
                  alongside the render loop
Hosting:          Vercel or Cloudflare Pages, static + edge CDN
```

**Why OffscreenCanvas over copying Active Theory's main-thread approach**: you explicitly said smoothness matters more than raw visual fidelity. Active Theory can afford main-thread rendering because their target audience skews toward higher-end devices for portfolio browsing and their brand tolerance for jank is near-zero-risk. You're optimizing for a 60Hz iPhone 12 Pro Max as a baseline — the worker-thread architecture is the correct trade for your stated constraint, even though it means diverging from your favorite reference on this one point.

[Guessing] OffscreenCanvas + Web Worker adds real implementation complexity (event forwarding, no direct DOM access inside the worker, debugging is harder). Flag this to Claude Code explicitly so it doesn't silently fall back to main-thread and lose the whole point.

---

## 2. DESIGN TOKENS

Palette leans Active Theory (pure black, your stated preference); token discipline and warmth calibration borrowed from Utsubo's approach (single exposed accent, careful off-white).

```css
:root {
  /* Color */
  --bg-void:        #000000;   /* AT: confirmed pure black background */
  --bg-raised:      #0a0a0c;   /* panels/cards, barely lifted off void */
  --ink-primary:    #FFFFFF;   /* AT: confirmed pure white text */
  --ink-muted:      #C6C6C6;   /* AT: confirmed nav/secondary text color */
  --accent:         #00FFFF;   /* AT: confirmed cyan, seen once as interactive accent */
  --accent-warm:    #f9b639;   /* Utsubo accent — optional 2nd accent for
                                   loader/progress only, keeps hero pure
                                   black/white/cyan */

  /* Type — AT's actual typeface (nbarchitekt) is proprietary/custom-cut.
     Do not attempt to source or clone it by name. Use a geometric
     technical-feeling alternative instead. */
  --font-display: 'Neue Machina', 'Space Grotesk', monospace;
  --font-mono:    'JetBrains Mono', 'IBM Plex Mono', monospace;
                  /* AT's UI chrome is confirmed monospace at 12-14px —
                     replicate that texture, not the exact font */

  /* Type scale — fluid clamp(), AT hero size is a GUESS since their
     headline renders in-canvas, not DOM. Utsubo's confirmed 96px/w100
     hero used as the calibration reference for scale relationship. */
  --step-label:   0.75rem;                          /* 12px, AT confirmed */
  --step-body:    1rem;
  --step-section: clamp(1.75rem, 1.4rem + 2vw, 2.75rem);
  --step-hero:    clamp(3rem, 2rem + 9vw, 7rem);     /* [Guessing] scale,
                     verify by measuring AT screenshot yourself against
                     3840px canvas width before finalizing */
  --weight-hero:  400;   /* AT's chrome text is weight 400/700 only —
                            NOT the ultra-thin 100 Utsubo uses. Keep AT's
                            weight character since you prefer their look. */

  /* Motion — one primary easing curve, confirmed pattern from both sites
     (each site exposed exactly one dominant curve) */
  --ease-primary: cubic-bezier(0.25, 1, 0.5, 1);   /* Utsubo's confirmed
                     --mainEasing, reused here as the single global curve */
  --dur-fast:   200ms;
  --dur-base:   700ms;
  --dur-slow:   1400ms;

  /* Spacing */
  --sp-1: 0.5rem; --sp-2: 1rem; --sp-3: 2rem;
  --sp-4: 4rem;   --sp-5: 8rem; --sp-6: 12rem;
}
```

**Rule for Claude Code**: no hardcoded hex/px/ms anywhere outside this block.

---

## 3. SECTION-BY-SECTION STRUCTURE

Reconstructed from actual captured screenshots, not invented. Active Theory's site is effectively a single full-viewport canvas experience (confirmed: page height ≈ viewport height, "scroll" drives the 3D camera, not DOM position) — that's the structural pattern to replicate, not a traditional long-scroll page.

### SECTION: Loader (your #1 stated complaint — new design required)
Both reference loaders are explicitly rejected. Do not reference either visually. Requirements:
```
- Max 1.8s perceived duration. Real progress tied to
  THREE.LoadingManager.onProgress + document.fonts.ready — never a fake timer.
- Concept: the loader geometry becomes part of the hero scene rather than
  being a gate that disappears. [This is the signature element — design
  it yourself, don't default to a spinner or progress bar.]
- Exit and hero entrance overlap by ~300ms, never sequential.
- Worker-thread canvas init happens DURING the loader, not after —
  the loader IS the "warming up the worker" period, made visible
  rather than hidden.
```

### SECTION: Hero (confirmed structure from screenshots)
```
CONFIRMED ELEMENTS (from capture):
  - Full-viewport WebGL canvas, pure black background
  - Central circular emblem/logo element, cyan-to-white gradient stroke
  - Radiating curved line elements around the circle (organic, not geometric)
  - Ambient particle field, low density, blue + warm-white mixed
  - "SCROLL DOWN" label, monospace, small, centered below emblem
  - Nav appears on scroll: "WORK — CONTACT" top-right, monospace, pill-shaped
    background at ~90% opacity (confirmed: --baropacity: 0.9)

ON LOAD:
  - Canvas opacity 0→1, --dur-slow, --ease-primary
  - Emblem: scale 0.9→1, opacity 0→1, --dur-base, delay 200ms
  - Particle field: fades in over --dur-slow, staggered spawn (not
    all-at-once) to avoid a "popping in" look
  - "SCROLL DOWN" label: opacity 0→1, delay 600ms

ON SCROLL (drives 3D camera, not page position — ScrollTrigger with
  scrub, mapped to worker-side camera transform via postMessage):
  - Camera moves through/past the emblem
  - Particle field parallax, opacity fades 1→0.3
  - Nav bar fades in at ~15% scroll progress

ON POINTER MOVE:
  - Subtle camera rotation toward pointer (forward pointer coords to
    worker via postMessage, lerp factor ~0.03-0.05/frame in worker)

MOBILE OVERRIDE:
  - No pointer-follow (no meaningful cursor on touch)
  - Particle count ÷4
  - Camera scroll-scrub replaced with simpler opacity/scale transitions
```

### SECTION: Content/Scene transitions (confirmed via network: "tree_room" scene)
```
CONFIRMED: a PBR-lit 3D environment ("tree_room") using compressed KTX2
textures for rock/soil/cable/structure materials, driven by a Draco/raw
geometry buffer (structure.bin).

[Guessing exact trigger] Likely a scroll- or click-triggered transition
into a fuller environment — consistent with "case study reveal" patterns
common to this class of site. Verify against the live site since we only
captured a snapshot, not the full interaction sequence.

MOTION (pattern to follow):
  - Cross-fade or camera-fly transition between hero and scene,
    --dur-slow, --ease-primary
  - Scene elements reveal with stagger (not simultaneous pop-in)
  - Exit: reverse of entry, not a hard cut
```

### SECTION: Footer/Contact
[Not captured in this scroll depth — page height ≈ viewport, meaning contact is likely reached via nav click, not scroll, on Active Theory's actual site.] Structure your contact as a nav-triggered overlay/route, not a scroll-to-bottom section, to match the confirmed single-viewport pattern.

---

## 4. THE SMOOTHNESS FIX (your core requirement — mandatory, not optional)

### 4.1 Worker architecture (primary fix)
```js
// main.js
const canvas = document.querySelector('#scene').transferControlToOffscreen();
const worker = new Worker('./render-worker.js', { type: 'module' });
worker.postMessage({ type: 'init', canvas, width: innerWidth, height: innerHeight }, [canvas]);

// forward only the events the worker needs, throttled to rAF rate
let pending = null;
window.addEventListener('pointermove', e => {
  pending = { x: e.clientX / innerWidth, y: e.clientY / innerHeight };
});
function tick() {
  if (pending) { worker.postMessage({ type: 'pointer', ...pending }); pending = null; }
  requestAnimationFrame(tick);
}
tick();
```
```js
// render-worker.js — Three.js scene lives entirely here
// receives canvas via transferControlToOffscreen, runs its own rAF loop,
// completely isolated from main-thread GSAP/Lenis/DOM work
```

### 4.2 Fallback detection (required — not all iOS versions support this)
```js
const supportsOffscreen = 'OffscreenCanvas' in window &&
  HTMLCanvasElement.prototype.transferControlToOffscreen;
// iOS Safari supports OffscreenCanvas from 16.4+. iPhone 12 Pro Max
// on an older iOS version needs the main-thread fallback path.
```

### 4.3 Scroll (Lenis config, iOS-specific)
```js
const lenis = new Lenis({
  duration: 1.1,
  easing: t => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
  smoothWheel: true,
  syncTouch: false,   // CRITICAL — true breaks iOS momentum scroll
  touchMultiplier: 1.5,
});
gsap.ticker.add(t => lenis.raf(t * 1000));
gsap.ticker.lagSmoothing(0);
```

### 4.4 GPU/texture budget
```js
// applies inside the worker
renderer.setPixelRatio(Math.min(self.devicePixelRatio, 2)); // iPhone DPR
                                                              // is 3; capping
                                                              // to 2 is ~44%
                                                              // fewer pixels,
                                                              // visually
                                                              // indistinguishable
renderer.powerPreference = 'high-performance';
```
- All textures KTX2/Basis compressed (both reference sites confirm this is standard practice for their optimized assets — Active Theory's non-KTX2 PNGs, up to 2MB each, are the exception to avoid, not the rule to follow)
- Texture budget under 128MB total (iOS Safari kills WebGL contexts above ~256MB)
- No duplicate asset loads — audit against Active Theory's confirmed bug (their `reel.mp4` loads twice, wasting 18.5MB)

### 4.5 Performance tiers (mandatory, auto-demoting)
```
TIER A: OffscreenCanvas supported + desktop/high-end mobile GPU
        → full particle count, full post-processing
TIER B: OffscreenCanvas supported + mid-range mobile
        → particles 40%, bloom only, no extra passes
TIER C: OffscreenCanvas unsupported → main-thread fallback
        → particles 15%, no post-processing
TIER D: prefers-reduced-motion OR sustained <45fps for 2s
        → static image/CSS only, zero WebGL
```
Detect via: `OffscreenCanvas` feature check, `navigator.hardwareConcurrency`, `navigator.deviceMemory`, a 1-second FPS probe on load. Auto-demote on sustained low FPS; never auto-promote back up mid-session.

### 4.6 Jank killers
- Only `transform`/`opacity` animated on the DOM side — never `top/left/width/height`
- `will-change` applied only during active animation, removed after
- No layout reads (`getBoundingClientRect`) inside any render loop — cache on resize, debounced 150ms

---

## 5. ASSET PIPELINE

| Asset | Format | Rule | Source confirmation |
|---|---|---|---|
| 3D models | `.glb`, Draco-compressed | <2MB each | Confirmed pattern: both sites use `_draco.glb` naming |
| Textures | KTX2/Basis | POT dimensions, 2048² max hero | Confirmed: `basis_transcoder.wasm` on both |
| Video | Avoid, or AV1→WebM→MP4 chain, <3MB | `muted playsinline preload="metadata"` | AT's 18.5MB duplicate-loaded video is the anti-pattern to avoid |
| Fonts | WOFF2, subset | `font-display: swap` | AT confirmed: `nbarchitekt` WOFF-equivalent at ~19KB — good practice to match |
| Images (non-3D) | AVIF→WebP→JPEG, srcset | lazy below fold | — |

**Total budget before first meaningful paint: under 2MB.** Everything else streams progressively after.

---

## 6. BACKEND / ARCHITECTURE

- **Hosting**: Vercel or Cloudflare Pages, static output, edge CDN
- **Asset CDN**: serve KTX2/GLB/fonts from a dedicated CDN path (both reference sites do this — Utsubo uses `cdn.utsubo.io`) for cache-header control independent of your app deploys
- **Contact**: serverless function (Vercel function or Resend API), no database needed
- **No CMS/database** unless case studies need frequent updates — MDX files are sufficient for a portfolio scale
- **Caching**: immutable hashed filenames, `Cache-Control: max-age=31536000, immutable` on all binary assets
- **Worker script**: served from same origin (required — cross-origin workers need CORS handling you don't want to debug)

---

## 7. QUALITY FLOOR (non-negotiable)

- Real semantic HTML behind the canvas — readable with JS disabled
- Keyboard-navigable, visible focus states, skip-to-content link
- `prefers-reduced-motion: reduce` → Tier D behavior
- WebGL context-loss handler → demotes to Tier D, never white-screens
- No console errors, CLS under 0.1, LCP under 2.5s on 4G
- Test the OffscreenCanvas fallback path explicitly — don't assume it works untested

---

## 8. CLAUDE CODE SESSION SEQUENCE

Do not request the whole build in one prompt.

1. **Tokens + semantic HTML skeleton.** No animation, no WebGL. Verify content reads correctly as a plain page.
2. **Lenis + GSAP scroll behavior only.** No 3D yet. Test smoothness on your iPhone here, before any GPU load exists in the picture.
3. **Main-thread Three.js hero scene, static.** Get the visual right before adding the worker complexity.
4. **Migrate to OffscreenCanvas + worker.** This is its own dedicated session — don't bundle it with new features. Test the fallback path immediately after.
5. **Wire scroll/pointer to worker via postMessage.** Camera control, parallax.
6. **Loader**, built as its own session, tied to real `LoadingManager` progress.
7. **Performance tiering system + auto-demotion logic.**
8. **Post-processing, polish, micro-interactions.**
9. **Full performance pass** — profile on your actual iPhone, cut what doesn't hold up, retest.

Test on your iPhone at the end of every session, not just at the end of the project.

---

## 9. PROMPT TEMPLATE

```
Build [SECTION] for a WebGL portfolio site.

STACK (fixed): Vite + vanilla JS. Three.js r16x running inside an
OffscreenCanvas Web Worker (main-thread fallback required for browsers
without OffscreenCanvas support). GSAP 3 + ScrollTrigger on main thread,
forwarding scroll/pointer state to the worker via postMessage. Lenis for
smooth scroll. postprocessing (pmndrs) inside the worker.

TOKENS: [paste Section 2 block]

SECTION SPEC: [paste the relevant Section 3 block]

CONSTRAINTS:
- All render work in the worker; main thread only handles DOM/scroll/input
- setPixelRatio capped at 2
- Performance tiers A-D per [paste Section 4.5]
- transform/opacity only for DOM animation
- prefers-reduced-motion → Tier D
- No hardcoded values outside the token block
- No duplicate asset loads — check network tab before calling this done

DO NOT: add sections not specified, add libraries not listed, fall back
to main-thread rendering without flagging it explicitly, or "improve"
the motion spec unprompted. If the spec is ambiguous, ask before building.
```

---

## 10. OPEN ITEMS ONLY YOU CAN CLOSE

1. Measure Active Theory's actual hero text size from the screenshot — the `--step-hero` value above is a placeholder pending that.
2. Confirm whether you want the AT-style weight-400/700 chrome text feel, or Utsubo's weight-100 extreme-thin hero — the token block currently keeps AT's character since you said you prefer their look overall, but this is your call to finalize.
3. Typeface licensing: neither `nbarchitekt` nor `PPMori` is available to you legally. Pick and license a replacement, or confirm you're fine with a free equivalent (Space Grotesk / Neue Machina Alt).
