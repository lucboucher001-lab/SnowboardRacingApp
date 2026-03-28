# Old Geezers Downhill Championship — Summary

## Purpose

A browser-based animated race visualiser built for a group of friends skiing and snowboarding in Obergurgl, Austria. Users enter competitor names and distances, then watch SVG characters race down parallel tracks to a procedurally generated chiptune soundtrack, culminating in a luxury-branded podium ceremony with athletics-style tie handling.

## Technology Stack

Single-file HTML application (~920 lines) using React 18 loaded from Cloudflare CDN, the Web Audio API for procedural music synthesis, hand-crafted SVG components for character art, and inline CSS with `@keyframes` animations. No build step, no bundler, no server. Deployed on GitHub Pages. Typography via Google Fonts (Bebas Neue and Nunito).

## Architecture Overview

The entire application lives in one `index.html` file organised into four logical sections: ES5 utility helpers and constants, a self-contained Web Audio music engine (`useMusicEngine` hook with `schedLoop`, `pNote`, `pKick`, `pSnare`, `pHat`), SVG character components (`Skier`, `Boarder`, `Model`), and the main `App` component which manages a three-phase flow (setup → countdown → racing/finished). All state is held in React `useState` hooks; animation runs via `requestAnimationFrame` over a fixed 25-second duration with quadratic easing. A `WinnerPanel` component renders the podium with correct athletics-style ranking.

## Key Design Decisions

- **Single HTML file with CDN dependencies** — eliminates all build tooling; anyone can fork and deploy by pushing one file to GitHub Pages
- **Procedural music via Web Audio API** — zero audio assets to host, distinctive retro aesthetic, avoids licensing concerns
- **Pre-transpiled ES5 with `React.createElement`** — no JSX, no Babel, no transpiler dependency; the trade-off is more verbose but fully self-contained code
- **Fixed 25-second race duration** — consistent viewing experience regardless of input distances; racers are positioned proportionally rather than by time
- **Name-based character assignment** — the `isDave()` check assigns a skier sprite to a specific friend; everyone else gets a snowboarder

## Current Limitations

- No data persistence — all inputs and results are lost on refresh
- Maximum 6 competitors, hardcoded to match the colour palette
- No accessibility features (ARIA labels, keyboard navigation, colour-blind support)
- Music engine fails silently if Web Audio API is unavailable
- Distance input lacks robust validation beyond `parseFloat() > 0`

## Status & Next Steps

The application is fully functional and deployed on GitHub Pages. The highest-value enhancements would be URL-based result sharing (encoding race data in query parameters for social sharing), localStorage persistence for recent races, a per-racer discipline selector to replace the `isDave()` easter egg, and an accessibility pass covering ARIA labels and colour-blind-friendly racer indicators.
