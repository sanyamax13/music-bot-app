# Stack Research — VDS Music (Telegram Mini App, single-file, no-build)

**Domain:** Telegram Mini App audio player, one `index.html`, CDN-only, GitHub Pages
**Researched:** 2026-04-15
**Confidence:** MEDIUM-HIGH overall (HIGH on architecture choices, MEDIUM on exact patch versions — tool sandbox blocked live npm/registry lookups during this research pass; versions below reflect state known as of May 2025 and should be bumped to the current `@latest` at implementation time via the pinned CDN URLs listed)

---

## TL;DR — The Prescribed Stack

> One sentence: **Vanilla JS + Web Components for structure, HTMLAudioElement (dual `<audio>` already in the code) for playback, MediaSession API for lock-screen, SortableJS for queue reorder, Lucide inline SVG sprite for icons, inline `<style>` with CSS custom properties + `@layer` for CSS, Telegram WebApp SDK v7+ for platform integration. No framework. No bundler. No npm.**

The existing `index.html` already uses the right primitives (dual `<audio>` buffers, inline CSS variables, tab views as plain DOM). The redesign does **not** need a framework — it needs discipline, a component boundary via custom elements, and a handful of tiny CDN libraries for things the platform doesn't give us for free.

---

## 1. UI Runtime — Framework Decision

### Recommended: **Vanilla JS + Web Components (Custom Elements v1)**, no framework

**Why this, not Preact/Alpine/Lit:**

| Option | Verdict | Reason |
|---|---|---|
| **Vanilla + Custom Elements** | RECOMMENDED | Zero runtime cost, zero CDN risk, native to every Telegram WebView since 2019. At 1–3k LOC a player is not big enough to justify a reactive layer. The pain points of vanilla (DOM diffing, re-renders) don't exist in a player — the player has ~6 views that swap with `display:none`, a persistent mini-player whose state you `innerText =` in place, and a queue list you rebuild on mutation. This is exactly what the current file already does correctly. |
| htm + Preact (via esm.sh) | Acceptable alternative | ~11 KB gzip. Good DX (JSX-like). But: (a) ESM imports from `esm.sh`/`unpkg` inside Telegram WebView on iOS have had sporadic cache/MIME issues; (b) re-introduces virtual DOM overhead on low-end Android where the budget is tight; (c) import-map indirection adds one more thing that can break on Pages. Use ONLY if the team strongly prefers a JSX-like authoring model. |
| Alpine.js 3.x | Rejected for this project | Great for sprinkles on server-rendered HTML; awkward for a stateful audio app with global player state, queue mutations, and MediaSession side effects. `x-data` scoping fights you when the mini-player needs to read/write state owned by three different views. |
| Lit 3.x | Rejected | Lit shines when you have a *design system* of many components. Here you have a player with maybe 8–10 custom elements. Lit's 6–8 KB + its decorator/reactive-property model is overhead for what vanilla `HTMLElement` + `attributeChangedCallback` does in 20 lines. |
| petite-vue | Rejected | Smaller Alpine; same scoping problems, smaller ecosystem, essentially dead upstream. |
| Vue/React/Svelte | Rejected | Require a build step. Hard constraint violation. |

**Pattern to use instead of a framework:**

```js
// One global store, event-driven
const Player = new EventTarget();
const state = { queue: [], index: -1, playing: false, track: null };
function setState(patch) { Object.assign(state, patch); Player.dispatchEvent(new Event('change')); }

// Custom element subscribes
class MiniPlayer extends HTMLElement {
  connectedCallback() { Player.addEventListener('change', this._r = () => this.render()); this.render(); }
  disconnectedCallback() { Player.removeEventListener('change', this._r); }
  render() { /* imperative DOM updates, no diffing */ }
}
customElements.define('mini-player', MiniPlayer);
```

This is the "Spotify Web Player" pattern (they use React, but the mental model is identical: single store, event bus, component subscribers). At 1–3k LOC you pay nothing and get full control of every pixel and every animation frame — critical when you're competing with native Spotify on an Android budget phone.

**Confidence:** HIGH — this is the only choice that fully satisfies the one-file, no-build, low-device constraint without importing a runtime.

---

## 2. Audio Playback Engine

### Recommended: **HTMLAudioElement × 2 (dual-buffer, already in code)** + MediaSession API

**Keep the existing `audioA`/`audioB` dual-`<audio>` pattern.** It is the correct pattern for a streaming player.

| Option | Verdict | Why |
|---|---|---|
| **HTMLAudioElement (dual buffer)** | RECOMMENDED | Native, streams via HTTP range requests, battery-efficient, backgrounded by the OS, driven by the platform's media pipeline → MediaSession/lock-screen/AirPods controls Just Work. Gapless-ish playback is achievable by preloading the next track into the second element and `.play()`-ing on `ended`. This is what the existing code already does. |
| howler.js 2.2.x | Rejected | Howler is designed for **game SFX** (short sounds, sprite sheets, Web Audio). For **music streaming**, Howler uses Web Audio by default, which means: (a) no native background playback on iOS Safari / iOS Telegram — audio stops when the WebView is backgrounded; (b) MediaSession integration is fragile because the audio isn't flowing through an `HTMLMediaElement`; (c) no HTTP range streaming — Howler fetches the whole file. **Do not use howler for a music player.** |
| Raw Web Audio API | Rejected for playback | Same problems as howler + more code. Use Web Audio **only** for a visualizer (see §9 — deferred). |
| hls.js / shaka / dash.js | Not needed | Backend serves direct audio URLs (not HLS/DASH). Adding these is dead weight. |

**True gapless playback:** Not achievable in the browser without Web Audio (and therefore losing background playback on iOS). Telegram Mini App users do not expect gapless — Spotify Web doesn't have it either. The dual-`<audio>` "preload the next, swap on `ended`" pattern gives a ~30–80 ms gap, which is fine and what the current code does.

**Preloading pattern for the queue:**
```js
function prepareNext() {
  const next = state.queue[state.index + 1];
  if (!next) return;
  nextAudio.src = next.stream_url;
  nextAudio.preload = 'auto';
  nextAudio.load(); // start buffering
}
activeAudio.addEventListener('ended', () => { swap(); prepareNext(); });
```

**Confidence:** HIGH. This is industry-standard for browser-based music players (Spotify Web, Yandex Music Web, SoundCloud Web all use `HTMLAudioElement`).

---

## 3. MediaSession API — Lock-screen / Notification Controls

### Recommended: Use it unconditionally, but feature-detect and degrade silently on iOS.

**Current support (as of 2025/2026):**

| Platform | Support | Notes |
|---|---|---|
| **Android Chrome / WebView** (Telegram on Android) | FULL | `navigator.mediaSession.metadata`, `setActionHandler` for play/pause/previoustrack/nexttrack/seekbackward/seekforward/seekto, `setPositionState`. Works on the lock screen and in the notification shade. This is the primary win. |
| **Desktop Chrome/Edge/Firefox** (Telegram Desktop) | FULL | Shows in system media controls (Windows SMTC, macOS Now Playing). |
| **iOS Safari 16.4+** | PARTIAL | `MediaSession.metadata` and basic action handlers work, but only when audio is driven by `HTMLAudioElement` (another reason to not use howler). `setPositionState` works. `nexttrack`/`previoustrack` show in the Control Center. |
| **iOS Telegram (WKWebView)** | PARTIAL, with gotchas | WKWebView inherits Safari's MediaSession support since iOS 16.4. **Gotcha:** audio only keeps playing in the background if Telegram itself is foregrounded OR the lock screen is shown with the Now Playing widget active. If the user switches to another app, iOS suspends the WebView and playback stops. This is an iOS platform limitation, not fixable from the web. Document it in the UX as "playback continues on lock screen, not when switching apps" — same behavior as Spotify/YM web on iOS. |

**Mandatory implementation checklist:**
```js
if ('mediaSession' in navigator) {
  navigator.mediaSession.metadata = new MediaMetadata({
    title: track.title,
    artist: track.artist,
    album: track.album || 'VDS Music',
    artwork: [
      { src: track.cover, sizes: '96x96',  type: 'image/jpeg' },
      { src: track.cover, sizes: '256x256', type: 'image/jpeg' },
      { src: track.cover, sizes: '512x512', type: 'image/jpeg' },
    ],
  });
  navigator.mediaSession.setActionHandler('play',           () => play());
  navigator.mediaSession.setActionHandler('pause',          () => pause());
  navigator.mediaSession.setActionHandler('previoustrack',  () => prev());
  navigator.mediaSession.setActionHandler('nexttrack',      () => next());
  navigator.mediaSession.setActionHandler('seekto',         (e) => { activeAudio.currentTime = e.seekTime; });
  // position updates (throttle to 1/s, NOT every timeupdate):
  setInterval(() => {
    if (!state.track || activeAudio.paused) return;
    navigator.mediaSession.setPositionState({
      duration: activeAudio.duration || 0,
      position: activeAudio.currentTime || 0,
      playbackRate: activeAudio.playbackRate,
    });
  }, 1000);
}
```

**Gotchas:**
1. Artwork **must** be same-origin or CORS-enabled. Cross-origin cover images from YouTube/SoundCloud/Jamendo thumbnails may fail silently on Android. Mitigation: proxy covers through the backend, or accept blank artwork on the lock screen.
2. `setPositionState` must be called with non-NaN `duration`. If you call it before metadata loads, Android Chrome throws.
3. Do NOT re-assign `mediaSession.metadata` on every `timeupdate` — it trashes the notification. Only on track change.
4. `setActionHandler('stop', …)` exists but Android may not render it; don't rely on it.

**Confidence:** HIGH — all points above are in MDN + Chrome Platform Status and match observed behavior in 2024-2025 WebView builds.

---

## 4. Telegram WebApp SDK

### Recommended: **`https://telegram.org/js/telegram-web-app.js` (the evergreen script, always latest)**

The SDK is **not** semver-pinned by Telegram — the script URL is evergreen and `window.Telegram.WebApp.version` reports the feature level (currently up to `"8.0"` / `"9.0"` range on recent clients; features are feature-detected, not version-pinned).

**Features that matter for this app (use all of them):**

| Feature | API | Why for VDS Music |
|---|---|---|
| Expand to full height | `tg.expand()` | Already used. Mandatory for a player. |
| Ready signal | `tg.ready()` | Call after first paint — hides the Telegram loading splash, avoids flash. **Missing from current code, add it.** |
| Disable vertical swipes | `tg.disableVerticalSwipes()` (v7.7+) | Critical: prevents the Telegram sheet from being dragged down when the user swipes on the queue/cover. **Mandatory for a touch music player.** |
| Closing confirmation | `tg.enableClosingConfirmation()` | Only enable while music is actively playing, to prevent accidental dismissal. Disable on pause. |
| Theme params | `tg.themeParams`, `tg.onEvent('themeChanged', …)` | The current design is dark-only. Keep it dark-only (matches Spotify/YM), but read `tg.themeParams.bg_color` only for header color sync via `tg.setHeaderColor()` / `tg.setBackgroundColor()`. |
| Haptics | `tg.HapticFeedback.impactOccurred('light'/'medium'/'heavy')`, `selectionChanged()`, `notificationOccurred('success'/'warning'/'error')` | Use on: play/pause tap (`light`), like/dislike (`medium`), long-press to open queue (`heavy`), tab switch (`selectionChanged`), wave mood swap (`selectionChanged`), error toast (`notificationOccurred('error')`). **This is what makes it feel native.** |
| BackButton | `tg.BackButton.show()/.hide()/.onClick()` | Show when the full-screen player is open or inside a modal (search source picker). Hide on the main tab. |
| MainButton | `tg.MainButton` | Use sparingly. Could be used on the "Profile" tab for "Reset taste" as a destructive MainButton. Optional. |
| SettingsButton | `tg.SettingsButton.show()/.onClick()` (v6.1+) | Wire it to open the Profile tab. Nice polish, not mandatory. |
| `initDataUnsafe.user.id` | Already used | Fine for this milestone (explicit decision in PROJECT.md). |
| `initData` (raw) | Reserved for future HMAC validation milestone | Do not ship server validation in this milestone — already out of scope. |
| Viewport | `tg.viewportHeight`, `tg.viewportStableHeight`, `onEvent('viewportChanged')` | Use `viewportStableHeight` (not `innerHeight`) to size the scroll container — it accounts for the keyboard and the Telegram nav bar correctly on iOS. **Current code uses `position: fixed; top:0; bottom:0` which is fine on Android but can clip on iOS when the keyboard opens during search. Fix with a CSS var `--tg-viewport-stable-height` updated on `viewportChanged`.** |

**Confidence:** HIGH — all listed APIs are documented at https://core.telegram.org/bots/webapps.

---

## 5. CSS at 1–3k Lines Without a Bundler

### Recommended: **Inline `<style>` + CSS Custom Properties + `@layer` + a naming convention (BEM-lite)**. No utility framework.

**Why no Tailwind / UnoCSS / Pico / etc.:**
- Tailwind CDN (`cdn.tailwindcss.com`) is explicitly marked "for development only" and ships the whole JIT compiler (~300 KB) on every page load. Unacceptable for Telegram WebView on 3G.
- UnoCSS runtime is similar, ~50 KB + runtime cost.
- Pico/Bulma/Bootstrap bring a visual style that fights against a Spotify/YM custom design.
- Utility classes at 1–3k LOC give you no real leverage over semantic CSS.

**Use these browser features to keep inline CSS maintainable at this scale:**

1. **CSS Custom Properties** (already in use — expand them). Promote every magic number to a variable. Theme via vars, not class swaps.
   ```css
   :root {
     /* color */
     --bg: #0f0f13;
     --surface-1: #1c1c24;
     --surface-2: #252530;
     --accent: #ff3366;
     --accent-glow: #ff336680;
     --text: #ffffff;
     --text-muted: #9aa0a6;
     /* geometry */
     --radius-sm: 8px;
     --radius-md: 16px;
     --radius-lg: 24px;
     --mini-player-h: 72px;
     --safe-bottom: env(safe-area-inset-bottom);
     /* motion */
     --ease-out: cubic-bezier(.2,.8,.2,1);
     --ease-spring: cubic-bezier(.5,1.6,.4,1);
     --dur-fast: 150ms;
     --dur-med: 280ms;
   }
   ```

2. **`@layer`** — group styles so later overrides don't fight specificity:
   ```css
   @layer reset, base, tokens, layout, components, utilities, state;
   @layer components { .track-item { ... } }
   @layer state { .track-item.is-playing { ... } }
   ```
   This is the single biggest maintainability win for inline CSS at this scale. Supported in all 2023+ browsers (Android Chrome 99+, iOS Safari 15.4+) — fine for Telegram WebView.

3. **Logical properties** — `padding-block`, `margin-inline` — future-proof for RTL if Russian UI ever needs Hebrew/Arabic.

4. **`color-mix()`** — derive tints/shades without hardcoding:
   ```css
   background: color-mix(in oklab, var(--accent) 20%, var(--surface-1));
   ```
   Supported in Android Chrome 111+, iOS Safari 16.2+. Safe.

5. **Container queries (`@container`)** — the full-screen player, mini-player, and queue panel can share components that self-adapt. Supported in Android Chrome 105+, iOS Safari 16+. Safe.

6. **Naming convention — BEM-lite:** `.player`, `.player__cover`, `.player__cover--expanded`, `.player.is-playing`. Do not mix with utility classes.

7. **Split the `<style>` block into comment-delimited sections** and keep them in the same order as the DOM (reset → tokens → layout → tab views → mini-player → full-screen player → queue → search → profile → utilities → animations). A senior dev should be able to Cmd-F for `/* === MINI PLAYER === */` and find everything.

**Fonts:** The current `@import` of Inter from Google Fonts works, but blocks first paint. Replace with a `<link rel="preconnect">` + `<link rel="stylesheet">` in `<head>` and preload just the one weight used above the fold (likely 600). Alternative: `font-display: swap` is already in Google's CSS — just stop using `@import` and use `<link>` so it parallelizes.

**Confidence:** HIGH on @layer / custom properties / color-mix / container queries (all in current browser baselines). HIGH on "no Tailwind CDN."

---

## 6. Icon System

### Recommended: **Lucide icons, copy-pasted as an inline SVG `<symbol>` sprite** in `index.html`. No CDN font.

**Why:**
- **No network fetch** — sprite lives in the HTML, zero FOUC, works offline, works on first paint.
- **Themable via `currentColor`** — icons inherit `color`, so theming is free.
- **~20 icons × ~300 bytes each = ~6 KB** added to `index.html` — cheaper than any icon font's first-load cost.
- **Lucide (https://lucide.dev)** is the modern fork of Feather, actively maintained, 1500+ icons, MIT, visually consistent with Spotify/YM icon language (thin strokes, rounded, 24×24 grid).

**Pattern:**
```html
<svg width="0" height="0" style="position:absolute" aria-hidden="true">
  <symbol id="i-play" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
    <polygon points="6 3 20 12 6 21 6 3"></polygon>
  </symbol>
  <!-- pause, skip-back, skip-forward, heart, heart-off, shuffle, repeat, list, search, user, chevron-down, chevron-right, more-vertical, x, settings, check, volume-2, grip-vertical (drag handle) -->
</svg>

<!-- usage -->
<button class="btn-icon"><svg class="icon"><use href="#i-play"/></svg></button>
```

**Icons needed (concrete list for this player):**
`play, pause, skip-back, skip-forward, shuffle, repeat, repeat-1, heart, heart-filled, thumbs-down, list (queue), search, user, home, radio (wave), library, chevron-down, chevron-right, more-vertical, x, settings, check, grip-vertical, plus, minus, refresh-cw (reset taste).`

**Rejected alternatives:**
- **Font Awesome / Material Icons font from CDN** — extra network request, ligature-based (fragile), bigger first paint, theming quirks.
- **Emoji (as in current code)** — inconsistent rendering across Android/iOS, different sizes, different colors, can't recolor. Drop them from UI chrome; keep only where they're content (genre cards, moods).
- **Iconify via CDN runtime** — adds a JS runtime that fetches SVGs over network. No.

**Confidence:** HIGH.

---

## 7. Drag-and-Drop for Queue Reorder (touch)

### Recommended: **SortableJS 1.15.x** from jsDelivr CDN.

```html
<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.2/Sortable.min.js"></script>
```
(pin to `@1.15.2` or move to `@1` for minor updates; latest minor as of training-data cutoff is 1.15.x — **verify the current patch at implementation time by opening the jsDelivr URL in a browser**).

**Why SortableJS:**
- **~45 KB min / ~13 KB gzip** — tiny.
- **Touch-first** — works natively on mobile without DnD API polyfills.
- **Handles autoscroll** on long lists (critical for a 50+ track queue).
- **`handle: '.drag-handle'` option** lets you put a grip icon on each row, so a tap still plays the track but a drag from the grip reorders. This is how Spotify/YM do it.
- **`onEnd` callback** gives you `oldIndex`/`newIndex` — just `queue.splice` and you're done.

**Why not alternatives:**
- **HTML5 native DnD API** — notoriously broken on mobile; requires polyfills; not worth fighting.
- **Pointer Events from scratch** — doable but ~200 LOC you'll re-debug. SortableJS has solved autoscroll, multi-touch, haptic triggers, and the long-press delay already.
- **dragula** — unmaintained since 2020. Avoid.
- **@dnd-kit / react-dnd** — require a framework. Out.

**Telegram haptic on drag:** Hook `onStart` → `tg.HapticFeedback.impactOccurred('medium')` and `onEnd` → `selectionChanged()`. Makes reorder feel native.

**Confidence:** HIGH. SortableJS is the de-facto standard for touch reorder since ~2016.

---

## 8. Cover-Art → Dynamic Player Background

### Recommended: **ColorThief 2.4.x via CDN** (or replace with a 30-line canvas-based extractor).

**Use case:** Yandex.Music and Spotify fade the full-screen player's background to the dominant color of the cover art. This is the single biggest visual payoff for minimal code.

**Option A — ColorThief (easiest):**
```html
<script src="https://cdn.jsdelivr.net/npm/colorthief@2.4.0/dist/color-thief.umd.js"></script>
```
```js
const ct = new ColorThief();
img.addEventListener('load', () => {
  const [r, g, b] = ct.getColor(img);
  document.documentElement.style.setProperty('--cover-tint', `rgb(${r},${g},${b})`);
});
```
Then in CSS:
```css
.player-full {
  background: linear-gradient(180deg,
    color-mix(in oklab, var(--cover-tint) 60%, var(--bg)) 0%,
    var(--bg) 60%);
  transition: background var(--dur-med) var(--ease-out);
}
```

**Gotchas:**
- Image **must be CORS-enabled** (`crossorigin="anonymous"` on the `<img>` AND `Access-Control-Allow-Origin` from the image host). YouTube/SoundCloud/Jamendo thumbs generally don't set CORS → ColorThief throws. **Mitigation:** proxy cover images through `api.vdsmusic.ru:8443` (one small backend change — but out of scope this milestone) OR fall back to a default gradient when extraction fails.
- Since backend changes are out of scope, **implement with a graceful fallback**: wrap `getColor` in try/catch and keep the current `--surface` background when it fails.

**Option B — inline vanilla extractor** (~40 LOC using `canvas.getContext('2d').getImageData` and averaging pixels). Same CORS constraint. Use if you want zero CDN deps.

**Recommendation:** Ship ColorThief behind a `try/catch`. The moment covers are proxied, it starts working for free. This is the single biggest UX upgrade in the redesign per line-of-code spent.

**Confidence:** MEDIUM — CORS constraint is real and may force Option B or defer the feature to when backend can proxy covers.

---

## 9. Other Libraries — Keep, Defer, or Reject

| Library | Verdict | Reason |
|---|---|---|
| **wavesurfer.js** | REJECT for this milestone | Beautiful waveforms, but: (a) requires decoding the full audio via Web Audio, which means losing background playback on iOS (same howler problem); (b) streaming waveforms need a pre-computed peaks file from the backend — backend changes out of scope. Defer to a future milestone. |
| **Hammer.js / ZingTouch** (gestures) | REJECT | Pointer Events are native and all you need for swipe-left/right/down. ~30 LOC inline. Hammer is unmaintained. |
| **Swiper.js** | REJECT | Heavy (~40 KB). The Wave "horizontal mood scroller" and genre carousels are trivially doable with `overflow-x: auto; scroll-snap-type: x mandatory; scroll-snap-align: start;` — which is what the current code does correctly. Keep it. |
| **GSAP** | REJECT | Fantastic animation library, but CSS transitions + Web Animations API + `transform`/`opacity` cover 100% of what a music player needs. GSAP is ~70 KB. Not justified. |
| **Motion One** (`@motionone/dom`) | Optional alternative | If animation composition gets painful, Motion One is 3.8 KB gzip and has a clean declarative API. Only pull in if `element.animate()` gets unwieldy. |
| **IntersectionObserver polyfill** | REJECT | Native everywhere Telegram runs. Use IO directly for lazy-loading cover images in long lists. |
| **date-fns / dayjs** | REJECT | You only format durations (`mm:ss`). One helper function, five lines. |
| **fetch polyfill / axios** | REJECT | `fetch` is native. Wrap it in a 20-line helper with `AbortController`, JSON parsing, and error toast hooks. |
| **Zod / Yup** | REJECT | No form validation needed beyond a search box. |
| **localForage** | REJECT | `localStorage` + JSON is fine for the data this app persists (last queue, volume, repeat/shuffle state, last wave mood). Budget is a few KB. |

---

## CDN URL Patterns (Prescriptive)

All served from jsDelivr (primary) with unpkg as mental fallback. jsDelivr has better global POPs for Russia/CIS and served CORS headers correctly for Telegram WebViews in 2024–2025.

```html
<!-- Telegram WebApp SDK (evergreen, do NOT pin) -->
<script src="https://telegram.org/js/telegram-web-app.js"></script>

<!-- SortableJS (pin minor, allow patch) -->
<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15/Sortable.min.js"></script>

<!-- ColorThief (pin minor) -->
<script src="https://cdn.jsdelivr.net/npm/colorthief@2.4/dist/color-thief.umd.js"></script>

<!-- Inter font (non-blocking) -->
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap">
```

**Version pinning strategy:**
- Telegram SDK → evergreen URL, always.
- Third-party libs → pin the **minor** (`@1.15`, `@2.4`), allow patches. Major-version pinning is too loose (breaking changes), patch-pinning is too tight (misses fixes).
- **Before shipping, verify each URL resolves to the expected version** by hitting it in a browser — jsDelivr will show the resolved version in the `X-Served-By` / response.
- Consider using **SRI hashes** (`integrity="sha384-..."`) for security on GitHub Pages. jsDelivr provides them in its UI. This is a small hardening win with zero perf cost.

---

## Installation

**Not applicable.** No npm, no bundler. Installation = pasting `<script>` and `<link>` tags into `<head>` of `index.html` and committing the file.

---

## Alternatives Considered

| Recommended | Alternative | When the alternative is better |
|---|---|---|
| Vanilla + Web Components | htm + Preact via esm.sh | When the codebase will clearly exceed ~3–4k LOC and the team needs JSX ergonomics more than raw control. Not the case here (redesign, not rewrite; existing code already vanilla). |
| HTMLAudioElement × 2 | Single `<audio>` + swap `.src` | Simpler, but causes a visible gap and re-buffering on every track change. Dual-buffer is a 30-line upgrade and worth it. |
| HTMLAudioElement | Web Audio API / howler | Only if you need gapless, visualizers, or effects — AND you can live without iOS background playback. Not this project. |
| Inline CSS + `@layer` | Tailwind Play CDN | Never for production. Use only for prototypes. |
| Lucide inline sprite | Iconify on-demand | Only if you need hundreds of icons across many pages. A player needs ~25. |
| SortableJS | Hand-rolled Pointer Events | If you can afford ~200 LOC and want zero deps. Viable, but SortableJS is 13 KB gzip of solved problems. |
| ColorThief | Inline canvas extractor | If you want zero CDN deps. ~40 LOC, same CORS caveats. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|---|---|---|
| **Tailwind CDN** (`cdn.tailwindcss.com`) | Ships the JIT compiler (~300 KB) on every load; marked "dev only" by maintainers. | Inline `<style>` + custom properties + `@layer`. |
| **howler.js** | Uses Web Audio → loses iOS background playback; no HTTP range streaming; no native MediaSession pipeline. | `HTMLAudioElement` × 2. |
| **wavesurfer.js** (this milestone) | Requires full decode via Web Audio; same iOS background issue; needs precomputed peaks. | Defer. Use `<input type="range">` as the seek bar. |
| **Swiper.js** | ~40 KB for a horizontal carousel that CSS scroll-snap already does. | `overflow-x:auto; scroll-snap-type:x mandatory`. |
| **GSAP** | ~70 KB; CSS transitions + `element.animate()` cover 100% of needs. | Native CSS/WAAPI. |
| **Alpine.js** | Scoping model fights global player state; framework overhead for no reactive wins in this app shape. | Vanilla + custom elements. |
| **Lit** | Authoring overhead not justified at ~10 components. | Vanilla `HTMLElement`. |
| **Hammer.js** | Unmaintained; Pointer Events are native. | Raw Pointer Events in ~30 LOC. |
| **Emoji as UI icons** | Inconsistent cross-platform rendering, uncolorable, sizing mismatches. | Lucide SVG sprite. Keep emoji only where it's *content* (genre cards). |
| **`@import` Google Fonts** (current code) | Blocks first paint. | `<link rel="preconnect">` + `<link rel="stylesheet">`. |
| **dragula** | Unmaintained. | SortableJS. |
| **axios / fetch polyfill** | Not needed; `fetch` is universal in Telegram WebView. | Native `fetch` + `AbortController` wrapper. |
| **Moment.js** | Huge; only formatting `mm:ss` is needed. | Two-line helper. |
| **jQuery** | Obvious. | DOM APIs. |

---

## Stack Patterns by Variant

**If the team wants JSX-style authoring more than raw performance:**
- Replace §1 with `import { html, render } from "https://esm.sh/htm/preact/standalone"`.
- Everything else in this doc stays identical.
- Accept: +11 KB gzip runtime, one import-map point of failure, slightly slower first paint on low-end Android.

**If covers can be proxied through `api.vdsmusic.ru`:**
- Add `crossorigin="anonymous"` to `<img class="cover">` elements.
- ColorThief dynamic backgrounds become the default, not the fallback.
- (Requires backend CORS header — out of scope this milestone, but note it as a future unlock.)

**If iOS background playback becomes a blocker for users:**
- Nothing to do on the web side — it's a WKWebView platform limit.
- Document it; consider a future native wrapper if the product ever leaves Telegram.

---

## Version Compatibility & Platform Baselines

| Feature | Min Android Chrome | Min iOS Safari / WKWebView | Notes |
|---|---|---|---|
| MediaSession basic | 73 | 15.0 | Ubiquitous in Telegram in 2025. |
| MediaSession `setPositionState` | 81 | 15.4 | Required for lock-screen scrubbing. |
| `@layer` | 99 | 15.4 | Safe. |
| `color-mix()` | 111 | 16.2 | Safe for 2025 WebViews. Fall back with a static second variable if you want to support iOS 15. |
| Container queries | 105 | 16.0 | Safe. |
| Custom elements v1 | 54 | 10.1 | Universal. |
| Pointer Events | 55 | 13.0 | Universal. |
| `element.animate()` (WAAPI) | 61 | 13.1 | Universal. |
| `IntersectionObserver` | 58 | 12.2 | Universal. |
| CSS `scroll-snap` | 69 | 11 | Universal. |
| `env(safe-area-inset-*)` | 69 | 11.2 | Required — iPhones with notch need `padding-bottom: env(safe-area-inset-bottom)` on the mini-player. |

Telegram WebView on iOS follows the system Safari engine; on Android, WebView is Chromium and auto-updates. In practice, **anything available in iOS Safari ≥ 16 and Android Chrome ≥ 110 is safe** — that's the effective 2025/2026 baseline for Telegram Mini Apps.

---

## Roadmap Implications (for the orchestrator)

1. **Phase ordering should start with a CSS/design-token foundation pass** before redesigning any view — otherwise you'll refactor tokens three times. Establish `:root` vars, `@layer` order, type scale, spacing scale first.
2. **The mini-player + full-screen player is the single biggest risk item** — persistent bottom element that expands to full-screen with shared-element animation is the trickiest piece of the whole redesign. It deserves its own phase.
3. **MediaSession should be wired in the same phase as the player refactor**, not later — they share the same state events.
4. **Queue + SortableJS + haptics** is a clean, self-contained phase.
5. **ColorThief dynamic backgrounds** should be the *last* polish phase and should be shippable-with-fallback (do not block the milestone on CORS/backend cover proxying).
6. **Wave homepage redesign** (Yandex-like) and **library/search redesign** (Spotify-like) are parallel phases that only share the token foundation from step 1.
7. **Do not introduce a framework mid-milestone** — if vanilla becomes painful at ~1500 LOC, stop and reconsider at a phase boundary, not mid-phase.

---

## Confidence Assessment per Section

| Section | Confidence | Why |
|---|---|---|
| §1 Framework choice | HIGH | Architectural fit is clear given constraints; widely validated pattern. |
| §2 Audio engine | HIGH | MDN-documented; matches every major web music player. |
| §3 MediaSession | HIGH on Android, MEDIUM on iOS background-app edge case (real platform limit, well known). |
| §4 Telegram SDK | HIGH | All listed APIs are in official docs. |
| §5 CSS approach | HIGH | Browser features are in current baselines. |
| §6 Icons | HIGH | Lucide inline sprite is a common, proven pattern. |
| §7 SortableJS | HIGH | De-facto standard. |
| §8 ColorThief | MEDIUM | Blocked by CORS on third-party thumbs — fallback plan required. |
| §9 Rejected libs | HIGH | All rationales derive from known platform behaviors. |
| Exact patch versions | MEDIUM | Live registry lookups were blocked by the research sandbox. The CDN URL patterns above pin minor versions, which is the correct strategy regardless — verify resolved patches before ship. |

---

## Sources

- MDN Web Docs — Media Session API, HTMLMediaElement, Custom Elements, CSS `@layer`, `color-mix()`, container queries, Pointer Events, Intersection Observer (HIGH)
- Telegram Bot API — WebApps reference: https://core.telegram.org/bots/webapps (HIGH)
- Lucide Icons — https://lucide.dev (HIGH)
- SortableJS — https://github.com/SortableJS/Sortable (HIGH)
- ColorThief — https://github.com/lokesh/color-thief (HIGH)
- jsDelivr CDN — https://www.jsdelivr.com (HIGH)
- Existing `index.html` (lines 1–60, 499–580) — confirmed current architecture already uses dual `<audio>`, CSS variables, tab views, Telegram SDK basics (HIGH, direct inspection)
- Training-data knowledge of Chrome/Safari platform status for listed features through May 2025 (HIGH for listed baselines; patch versions of libraries MEDIUM — verify at implementation time)

*Note:* Live registry lookups (`registry.npmjs.org`) and web search were unavailable in this research environment, so exact patch versions for SortableJS, ColorThief, Preact, Alpine, etc. should be confirmed by opening the pinned jsDelivr URLs listed in §CDN URL Patterns before merging the redesign. The minor-version pinning strategy (`@1.15`, `@2.4`) makes this low risk.

---
*Stack research for: Telegram Mini App audio player, single-file no-build*
*Researched: 2026-04-15*
