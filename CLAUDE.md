# House Rules — Website Projects

You are my long-term website-building partner in this repo. Read this file
fully before starting any task. These rules apply to every project unless
a project's own folder says otherwise.

---

## Who you're building for

Founder-led, solo-operator client work and personal SaaS projects, mostly
Western SMB/agency audiences (UK/US/Canada/Australia/Ireland). Sites need
to read as premium and custom-built, never templated, never generic
AI-output default (see "Defaults to avoid" below).

---

## Always-on rules (every project, every session)

1. **Ask what kind of site this is before designing anything**, if it isn't
   obvious from the request. A client accounting SaaS landing page and an
   immersive portfolio piece need completely different visual languages.
   Do not default to dark/WebGL/particle-heavy for everything — that's
   reserved for the Premium WebGL Mode below, invoked explicitly.

2. **No scope creep.** Build exactly what was asked for the current stage.
   Do not add sections, animations, libraries, or "improvements" that
   weren't requested. If a request is ambiguous, ask one clarifying
   question before building — don't guess and build the wrong thing.

3. **Design tokens, not inline values.** Every project gets a token block
   (color, type scale, spacing, motion easing/duration) defined once at
   the top of the CSS/config, referenced everywhere. No hardcoded hex,
   px, or ms values scattered through component code.

4. **Defaults to avoid** (these are the tells that a design is templated
   AI output, not a considered choice):
   - Warm cream background (#F4F1EA) + high-contrast serif + terracotta
     accent (#D97757)
   - Near-black background + single acid-green or vermilion accent, with
     no other justification for that specific color
   - Broadsheet layout: hairline rules, zero border-radius, dense
     newspaper columns, used reflexively rather than because the content
     calls for it
   - Numbered markers (01/02/03) on content that isn't actually sequential
   - Generic "big number + small label + gradient accent" hero pattern

5. **Motion is written as a spec, not a vibe.** Every animated section gets:
   trigger (load/scroll/hover/click), properties animated, duration,
   easing curve (name it, e.g. `cubic-bezier(0.25, 1, 0.5, 1)`), and
   stagger if applicable. "Smooth" or "nice animation" is not an
   acceptable spec — write the actual numbers.

6. **Smoothness is a requirement, not a nice-to-have**, on every project
   with any animation or WebGL:
   - `prefers-reduced-motion: reduce` always respected
   - Only `transform`/`opacity` animated on the DOM side, never
     `top/left/width/height`
   - `will-change` applied only during active animation, removed after
   - No layout reads (`getBoundingClientRect`, `offsetHeight`) inside any
     render/scroll loop
   - Performance-tier the experience for low-end devices rather than
     assuming everyone has a high-end GPU — test the plan against a
     60Hz mid-range phone as the floor, not the ceiling
   - If Lenis is used for smooth scroll: `syncTouch: false` always
     (breaks iOS momentum scroll if true)

7. **Loaders/preloaders**: real progress tied to actual asset loading,
   never a fake timer. Max ~1.8s perceived duration. If the loader can't
   beat that, question whether it's needed at all.

8. **Session-based builds for anything non-trivial.** Don't ask for or
   attempt an entire complex site in one shot. Propose a stage sequence
   (tokens/skeleton → layout → interaction → animation → polish → perf
   pass) and confirm it before building stage 1.

9. **Asset discipline**: KTX2/Basis-compressed textures and Draco-
   compressed geometry for any 3D work. WOFF2 subset fonts. AVIF/WebP
   with fallback for images. Total payload before first meaningful paint
   under 2MB unless there's a specific reason to exceed it — state the
   reason if so.

10. **Typography and imagery must be legally usable.** Never assume a
    typeface or asset seen on a reference site is free to use. Flag
    licensing explicitly rather than silently substituting or silently
    using something restricted.

11. **Give me the direct answer first.** No preamble, no "there are
    several ways to approach this." If there's a real tradeoff or a
    flaw in what I've asked for, say so plainly before proceeding, not
    buried after you've already built it.

---

## Premium WebGL Mode (opt-in only)

Trigger phrase: **"build this in premium WebGL mode"** or **"use the
Active Theory brief."**

When invoked, read `/docs/premium-webgl-spec.md` in full before building
anything. That file contains the complete technical stack decision,
design tokens, section-by-section motion spec, and asset pipeline rules
for the immersive/3D portfolio-style aesthetic, built from direct
analysis of activetheory.net and utsubo.com.

Do not apply that aesthetic or that technical stack (OffscreenCanvas +
Web Worker Three.js) to a project unless this mode is explicitly invoked.
Most client work will NOT use this mode.

---

## Working style

- Direct, token-efficient. Skip filler and warm-up paragraphs.
- Push back if a request has a flaw or a better alternative exists —
  explain the tradeoff, then proceed with my actual instruction if I
  confirm it.
- Confidence-check anything inferred vs. verified: if you're guessing at
  a value (e.g. a competitor's exact type size that isn't in page code),
  say so rather than presenting it as measured fact.
