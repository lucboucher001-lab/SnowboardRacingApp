# Old Geezers Downhill Championship — Software Design Document

> A browser-based snowboard and ski racing visualiser that animates competitors down parallel tracks based on user-entered distances, complete with procedurally generated music, SVG character art, and a luxury-branded winner ceremony — built for a group of friends racing in Obergurgl, Austria.

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture & Technical Design](#2-architecture--technical-design)
3. [Core Logic & Algorithms](#3-core-logic--algorithms)
4. [UI & Interaction Design](#4-ui--interaction-design)
5. [Data Model](#5-data-model)
6. [Design Decisions & Trade-offs](#6-design-decisions--trade-offs)
7. [Known Limitations & Technical Debt](#7-known-limitations--technical-debt)
8. [Future Enhancement Opportunities](#8-future-enhancement-opportunities)
9. [Lessons Learned & Patterns](#9-lessons-learned--patterns)

---

## 1. Project Overview

Old Geezers Downhill Championship is a playful, single-page web application that turns real-world snowboard and ski race results into an animated visual experience. Users enter competitor names and the distances they travelled, then watch an animated race where characters descend parallel tracks proportional to their actual results. The race culminates in a stylised winner ceremony with a podium, procedurally generated background music, and luxury ski-brand visual theming.

The application is built for a specific social group — friends who ski and snowboard together in Obergurgl, Austria — and serves as a fun way to celebrate and compare results after a day on the slopes. It runs entirely in the browser with no backend, no authentication, and no data persistence. The technology stack is React 18 (loaded from CDN, no build step), inline CSS styling, the Web Audio API for procedural music generation, and hand-crafted SVG components for character art. The entire application lives in a single `index.html` file of approximately 920 lines, deployed as a static site on GitHub Pages.

---

## 2. Architecture & Technical Design

### Overall Structure

The application is a single HTML file containing three logical regions: CSS animations and base styles (lines 10–30), JavaScript application code (lines 34–918), and the minimal HTML shell (the `<div id="root">` mount point). There is no build step, no bundler, and no module system — React and ReactDOM are loaded from the Cloudflare CDN, and the entire application is written in pre-transpiled ES5-compatible JavaScript (using manual `_slicedToArray`, `_toConsumableArray`, and `_defineProperty` helper functions rather than modern destructuring or spread syntax).

### Code Organisation

The JavaScript is organised into four functional sections:

**Utility helpers and constants (lines 37–55)** — The `_slicedToArray`, `_toConsumableArray`, and `_defineProperty` polyfill functions provide ES6+ syntax support without a transpiler. The `COLORS` array defines the six racer colour palette. `RACE_DURATION` (25,000ms) controls animation length. `isDave()` checks if a competitor's name is "dave" (case-insensitive) to assign them a skier character instead of a snowboarder. `easeInOut()` implements a quadratic ease-in-out curve for smooth race animation.

**Music engine (lines 57–194)** — A self-contained procedural music system built on the Web Audio API. The `useMusicEngine()` custom hook manages the `AudioContext` lifecycle, and a collection of functions (`pNote`, `pKick`, `pSnare`, `pHat`, `schedLoop`) synthesise a looping chiptune-style track at 168 BPM in real time.

**SVG character components (lines 196–322)** — Three React components render the visual characters: `Skier()` draws a ski-pole-wielding figure, `Boarder()` draws a snowboarder, and `Model()` draws a decorative fashion-model skier used in the winner ceremony's visual branding. Each is a pure SVG composition with no external assets.

**Application components (lines 324–918)** — `WinnerName()` renders a gold-gradient animated winner name with auto-sizing. `WinnerPanel()` builds the full podium ceremony view with athletics-style ranking (ties handled correctly). `App()` is the root component managing all application state and the three-phase user flow: setup → countdown → racing/finished.

### Dependencies

| Dependency | Source | Purpose |
|---|---|---|
| React 18.2.0 | Cloudflare CDN | UI rendering and state management |
| ReactDOM 18.2.0 | Cloudflare CDN | DOM mounting via `createRoot` |
| Google Fonts (Bebas Neue, Nunito) | Google Fonts CDN | Typography — Bebas Neue for headings, Nunito for body text |

There are no other dependencies. The music engine, animations, and all SVG art are hand-coded.

### Entry Point and Initialisation

The application initialises on line 917: `ReactDOM.createRoot(document.getElementById('root')).render(React.createElement(App, null))`. This mounts the `App` component, which renders in the `setup` phase by default. No data is fetched, no side effects fire on mount beyond a `resize` event listener that tracks the race track container height.

### Deployment and Hosting

The application is deployed as a static site on GitHub Pages. Since the entire application is a single `index.html` file with all dependencies loaded from CDNs, deployment requires nothing more than pushing the file to the repository's Pages-enabled branch. There is no build step, no CI/CD pipeline, and no environment-specific configuration. The application requires an internet connection at load time to fetch React and font assets from their respective CDNs, but once loaded it runs entirely offline.

---

## 3. Core Logic & Algorithms

### Race Animation

The race animation is driven by `requestAnimationFrame` inside the `begin()` function (line 603). When the user clicks "Start the Race," the function first triggers a 3-second countdown via `setInterval`, then switches to the `racing` phase and begins the animation loop. The `anim()` callback (line 614) computes a linear progress value `p` as the elapsed time divided by `RACE_DURATION` (25 seconds), clamped to `[0, 1]`. This raw progress is stored in the `prog` state variable and later transformed by `easeInOut()` (line 642) to produce `eased`, which controls the visual position of all racers.

Each racer's vertical position on the track is calculated as `r * eased * maxT`, where `r` is the racer's distance as a proportion of the maximum distance (`parseFloat(p.distance) / maxD`), `eased` is the eased progress, and `maxT` is the available track pixel height minus space for the character sprite and finish line (`Math.max(trackH - SH - FG - 4, 20)`). This means the racer with the greatest distance always reaches the bottom of the track at exactly `t = RACE_DURATION`, while others stop proportionally short — visually conveying the relative gap between competitors.

### Winner Determination

When `p` reaches 1.0, the animation terminates and `winners` is computed (line 622) by finding all racers whose `distance` equals `maxDist` — the maximum distance in the valid racer set. This correctly handles ties: if two racers share the highest distance, both appear as joint winners. The `winner` state is set and the phase transitions to `finished`.

### Podium Ranking (Athletics-Style Ties)

The `WinnerPanel` component (line 358) implements athletics-style ranking where ties consume positions. The `sorted` array is grouped by distance: racers with identical distances share a rank, and the next rank skips ahead by the number of tied racers. For example, a two-way tie for 1st means the next rank is 3rd (no 2nd place). The podium display is limited to ranks 1–3, arranged in the classic silver–gold–bronze visual order (line 397).

The grouping algorithm (lines 367–381) iterates through the distance-sorted racer array, creating a new group each time the distance changes. It uses string conversion (`String(parseFloat(...))`) to avoid floating-point comparison issues. The `nextRank` counter increments by the size of each group, producing the standard athletics ranking sequence.

### Easing Function

`easeInOut(t)` (line 53) implements a quadratic ease-in-out: for `t < 0.5` it returns `2 * t * t` (accelerating), and for `t >= 0.5` it returns `-1 + (4 - 2 * t) * t` (decelerating). This gives the race a natural feel — racers appear to accelerate from the start line and decelerate as they approach the finish.

### Music Synthesis

The music engine generates a looping two-bar pattern at 168 BPM using the Web Audio API. `schedLoop()` (line 98) is the core scheduling function. It layers four elements per loop iteration:

1. **Drums** — `pKick()`, `pSnare()`, and `pHat()` create a standard four-on-the-floor pattern with alternating open/closed hi-hats. The kick synthesises a sine wave with a rapid frequency sweep from 150Hz to 40Hz. The snare generates a white-noise buffer filtered through a 2200Hz bandpass. Hi-hats use high-pass filtered noise at 9kHz.

2. **Chord progression** — A five-chord sequence (E3–B3, G3–D4, A3–E4, D3–A3, E3–B3) played with distorted sawtooth and square waves through `pNote()`, using the `mkDist()` waveshaper for overdrive.

3. **Bass line** — An eight-note pattern on triangle waves at root frequencies (E2, A2, D3, B2), providing the harmonic foundation.

4. **Lead melody** — A nine-note phrase on square waves (E4–G4–A4–B4–A4–G4–E4–D4–E4) that sits above the chord voicing.

The `useMusicEngine()` hook (line 126) manages the `AudioContext` lifecycle. `unlock()` creates the context and plays a silent buffer to satisfy mobile autoplay restrictions. `play()` begins a scheduling loop that runs every 300ms, always scheduling notes 1.6 seconds ahead of the current time to prevent audio gaps. `stop()` fades the master gain to zero over 400ms, then closes the context after 500ms. `toggle()` mutes/unmutes by ramping the master gain node.

### The "Dave" Easter Egg

The `isDave()` function (line 50) checks whether a racer's name (trimmed, lowercased) equals "dave". If so, that racer is rendered as a `Skier` component instead of a `Boarder`, and receives a slightly different wobble animation (`sb` instead of `bw`) with tighter rotation parameters. This is a personalised touch for a specific member of the friend group who presumably skis rather than snowboards.

---

## 4. UI & Interaction Design

### User Workflow

The application follows a strict three-phase flow: **Setup → Countdown → Racing/Finished**.

**Setup phase** — The user sees the branded header ("Old Geezers Downhill Championship — Obergurgl") with falling snowflake animations and a mountain silhouette backdrop. Below this is a registration form where competitors are added. Each row has a colour-coded number badge, a name input, a distance input (in metres), and a remove button. The form starts with two empty rows. The "Add Competitor" button (dashed border, understated styling) allows adding up to 6 racers. The "Start the Race!" button is disabled and visually dimmed until at least two valid entries (name + positive distance) exist. Validation is inline — the `valid` array is recomputed on every render by filtering `parts` for entries where `name.trim()` is truthy and `parseFloat(distance) > 0`.

**Countdown phase** — A large animated countdown (3… 2… 1…) fills the screen, using the `cd` keyframe animation that scales from 1.8× to 1× with a fade-in. The text "Racers take your positions…" appears below. This lasts exactly 3 seconds.

**Racing phase** — Parallel vertical tracks appear, one per racer. Each track is a rounded, semi-transparent container with subtle grid lines at 20%, 40%, 60%, and 80%. A chequered finish line sits at the bottom. The racer's SVG character sprite descends from the top, leaving trail lines behind it. Characters wobble as they move (via `bw` or `sb` CSS animations). The procedural music plays throughout. A mute/unmute button is available in the header.

**Finished phase** — When the animation completes, the winner's track glows with a gold pulsing border (`gp` animation). The `WinnerPanel` slides in from below with a luxury ski-brand aesthetic — a dark card with gold accents, the text "Maison Obergurgl · Collection Hiver", a trophy emoji, and the winner's name rendered with a shimmering gold gradient (`sh` animation on the background clip). Below this, a three-position podium (silver–gold–bronze arrangement) shows ranked results with medal emojis and animated block heights. Joint winners are handled gracefully with "Joint Champions" text, "Les Rois de la Montagne" (or singular "Roi"), and tied position indicators (e.g. "1="). A "Race Again!" button resets all state.

### Visual Design

The interface uses a full-viewport gradient background transitioning from deep navy (#040f1f) through blue (#163d75) to a light sky (#e5f2fb). Fixed-position SVG mountain silhouettes and a snow-white foreground strip sit at the bottom. Twenty-two CSS-animated snowflake particles fall continuously across the viewport using the `sf` keyframe. Typography pairs Bebas Neue (a condensed display font) for all headings and UI labels with Nunito (a rounded sans-serif) for body text. The colour palette for racers cycles through six values: red (#E63946), orange (#F4A261), teal (#2A9D8F), blue (#5B7BE5), purple (#9B5DE5), and pink (#F15BB5).

### Responsive Behaviour

The layout is capped at 600px width (`maxWidth: 600`) and centred horizontally, making it naturally mobile-friendly. Font sizes use `clamp()` for fluid scaling — the title, for example, ranges from 32px to 70px. The `WinnerName` component (line 324) implements auto-sizing: it starts at 36px and decrements until the text fits within the container width, using a `requestAnimationFrame` callback that measures `scrollWidth` against `clientWidth`. The viewport meta tag disables user zoom (`maximum-scale=1.0`).

---

## 5. Data Model

### Input Data

Users enter data through the registration form. Each competitor requires two fields:

| Field | Type | Validation | Example |
|---|---|---|---|
| `name` | String | Must be non-empty after trimming | "Luc" |
| `distance` | String (numeric) | Must parse to a positive float via `parseFloat()` | "847" |

### Internal Representation

The `parts` state array holds the raw form data. Each entry is an object:

```
{ id: Number, name: String, distance: String }
```

The `id` field is an auto-incrementing integer (starting at 1, tracked by the `nid` state variable) used as the React key. The `distance` field remains a string throughout the form lifecycle; it is parsed to a float only when computing race positions.

The `valid` array is a derived subset of `parts`, filtered to entries where `name.trim()` is truthy and `parseFloat(distance) > 0`. The `pwc` ("parts with colours") array extends `valid` by adding a `color` property from the `COLORS` palette, cycling by index.

The `sorted` array is a copy of `pwc` sorted by `parseFloat(distance)` in descending order, used for podium ranking.

### Derived Values

| Value | Derivation | Purpose |
|---|---|---|
| `maxD` | `Math.max()` across all valid distances | Normalises racer positions to `[0, 1]` range |
| `eased` | `easeInOut(prog)` | Smooth animation progress |
| `r` | `parseFloat(p.distance) / maxD` | Individual racer's proportional position |
| `top` | `r * eased * maxT` | Pixel offset from top of track |
| `maxT` | `trackH - SH - FG - 4` (minimum 20) | Available vertical pixels for animation |

### State Management

All state is managed through React `useState` hooks in the `App` component:

| State | Type | Purpose |
|---|---|---|
| `phase` | `'setup' \| 'countdown' \| 'racing' \| 'finished'` | Current application phase |
| `parts` | `Array<{id, name, distance}>` | Registered competitors |
| `nid` | `Number` | Next available competitor ID |
| `prog` | `Number [0–1]` | Race animation progress |
| `winner` | `Array<Racer> \| null` | Winner(s) after race completes |
| `cd` | `Number \| null` | Countdown value (3, 2, 1, or null) |
| `trackH` | `Number` | Track container height in pixels |
| `muted` | `Boolean` | Music mute state (inside `useMusicEngine`) |

Refs (`useRef`) hold mutable values that don't trigger re-renders: `rafR` (animation frame ID), `t0R` (animation start timestamp), `trackR` (DOM reference for track height measurement), and audio context references inside the music hook.

### Output

The application produces no persistent output. All results are visual — the animated race and the podium display. No data is saved, exported, or transmitted.

---

## 6. Design Decisions & Trade-offs

**Single HTML file with no build step** — The entire application lives in one file with CDN-loaded dependencies. This eliminates tooling complexity entirely: there is no package.json, no webpack, no transpiler configuration. The trade-off is that the code uses verbose ES5 patterns (manual destructuring helpers, `React.createElement` instead of JSX) and cannot use modern import/export syntax. For a fun, single-purpose application of this size, this is a reasonable trade-off — anyone can fork the file and open it locally.

**Pre-transpiled ES5 JavaScript** — Rather than writing JSX and running Babel, the code is written directly in `React.createElement` calls with manual polyfill functions. This means zero build dependencies, but the code is harder to read than equivalent JSX would be. The verbose helper functions (`_slicedToArray`, `_toConsumableArray`, `_defineProperty`) suggest the code may have originally been written in modern syntax and transpiled once, with the output committed directly.

**Procedural music via Web Audio API** — Instead of loading an audio file, the application synthesises music in real time. This keeps the project zero-asset (no MP3 files to host or load), adds a distinctive retro-game aesthetic, and avoids licensing concerns. The trade-off is code complexity — the music engine is roughly 130 lines of low-level oscillator and buffer manipulation — and the fact that the sound quality is inherently limited to chiptune-style synthesis.

**CDN-loaded React** — Loading React from Cloudflare's CDN means the application requires an internet connection on first load. This is acceptable for the GitHub Pages deployment target, where users are already online. The benefit is zero local dependencies and instant deployability.

**Fixed 25-second race duration** — The `RACE_DURATION` constant (25,000ms) is hardcoded. Every race takes exactly the same time regardless of the distances entered. This simplifies the animation logic and ensures a consistent viewing experience, but means the animation doesn't reflect actual time differences between competitors — only distance differences.

**Maximum 6 competitors** — The `addP()` function caps the racer count at 6 (matching the length of the `COLORS` array). This keeps the UI manageable on mobile screens where tracks must sit side by side, but limits use for larger groups.

**The "Dave" special case** — The `isDave()` check is a deliberate personalisation for a specific group member. Rather than adding a discipline selector to the UI, the code uses name detection to assign the skier character. This is charming for the intended audience but would be confusing to anyone outside the friend group.

---

## 7. Known Limitations & Technical Debt

**No data persistence** — All competitor data and results are lost on page refresh. There is no localStorage, no URL parameter encoding, and no export functionality. Re-entering data for repeated use requires manual input each time.

**Distance input accepts any string** — The distance input has `type="number"` but no further validation beyond checking `parseFloat() > 0`. Extremely large values, negative values that happen to parse as positive, or values with many decimal places are all accepted without sanitisation. The input also accepts scientific notation (e.g. "1e5") which `parseFloat` handles but which may surprise users.

**Hardcoded competitor limit of 6** — The colour palette has exactly 6 entries and the `addP()` function enforces this ceiling. Adding a 7th racer would require extending `COLORS` and potentially rethinking the side-by-side track layout for narrow screens.

**Music engine has no graceful degradation** — If the Web Audio API is unavailable (older browsers, or browsers with restrictive autoplay policies that resist the unlock hack), the `console.warn` in `unlock()` fires but the application provides no user-facing indication that music won't play.

**No accessibility considerations** — The application has no ARIA labels, no keyboard navigation for the form (beyond default browser behaviour), no skip links, and no screen-reader-friendly race narration. The heavy reliance on colour to distinguish racers is also problematic for colour-blind users — there are no patterns, labels, or other distinguishing marks on the tracks themselves.

**CSS animations are purely decorative but always active** — The snowflake particles and mountain scene render continuously via CSS `@keyframes`, even during the race when they are behind other content. On lower-powered devices this could affect animation smoothness.

**The `winner` variable reference in `begin()` is potentially stale** — Inside the `anim()` closure, `winners` is computed from `pwc`, which is derived from the component's render scope at the time `begin()` was called. Since `valid` and `pwc` are recalculated on every render but the closure captures the value from the render where `begin()` ran, this works correctly in practice (the inputs don't change during racing), but the pattern is fragile.

---

## 8. Future Enhancement Opportunities

**Save and share results via URL** — Encoding racer names, distances, and the race outcome into URL query parameters or a hash fragment would allow users to share a specific race result as a link. This would be high-value for the social use case and straightforward to implement since the data model is small.

**Add a discipline selector per racer** — Rather than relying on the `isDave()` name check, each racer could have a toggle between skier and snowboarder. This would make the application usable for any group while preserving the character variety.

**Persist recent results in localStorage** — Storing the last few races would let users quickly replay or compare across sessions without re-entering data.

**Sound effects for countdown and finish** — Adding a countdown beep and a finish-line fanfare (both synthesisable with the existing Web Audio engine) would enhance the dramatic arc without adding any asset dependencies.

**Responsive track layout** — For more than 3–4 racers on a narrow screen, the side-by-side tracks become very thin. A two-row layout or a horizontal scrolling approach would improve readability.

**Accessibility pass** — Adding ARIA labels to the form, colour-blind-friendly racer indicators (patterns, shapes, or always-visible name labels on the tracks), and keyboard shortcuts for start/reset would broaden the audience.

**Configurable race duration** — Allowing users to adjust the `RACE_DURATION` (e.g. "quick race" vs. "dramatic finish") would add replay variety.

---

## 9. Lessons Learned & Patterns

**Zero-dependency deployment is powerful for fun projects** — By committing to a single HTML file with CDN dependencies, the project achieves the simplest possible deployment story: push one file to GitHub Pages and it works. There is no CI/CD to configure, no build to break, no node_modules to manage. For a project whose purpose is entertainment rather than enterprise software, this simplicity is a feature, not a limitation.

**Procedural audio adds character with surprising efficiency** — The music engine demonstrates that a compelling chiptune soundtrack can be built in roughly 130 lines of Web Audio API code. The approach of scheduling notes slightly ahead of real time (the 1.6-second lookahead in `play()`) is a well-established pattern for avoiding audio glitches without introducing perceptible latency.

**SVG-as-React-components scales well for custom art** — The `Skier`, `Boarder`, and `Model` components show that hand-coded SVG, while verbose, produces resolution-independent, colour-customisable character art with no asset loading. The pattern of passing a `color` prop that flows through to multiple SVG elements makes each character trivially recolourable.

**Athletics-style tie handling is worth getting right** — The podium ranking algorithm in `WinnerPanel` correctly handles edge cases like multi-way ties (including the scenario where a 4-way tie for 1st means no silver or bronze). Using string-keyed distance grouping to avoid float comparison issues is a pragmatic defensive choice.

**Easing makes animation feel physical** — The quadratic ease-in-out on the race progress gives racers a sense of acceleration and deceleration that makes the 25-second animation feel like a real descent rather than a mechanical slider. This small detail significantly improves the perceived quality of the experience.
