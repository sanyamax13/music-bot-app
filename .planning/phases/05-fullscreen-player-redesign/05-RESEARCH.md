# Phase 5: Fullscreen Player Redesign - Research

**Researched:** 2026-04-16
**Domain:** Mobile-first fullscreen player UI (CSS, touch gestures, DOM manipulation)
**Confidence:** HIGH

## Summary

Phase 5 is a surgical enhancement of the existing FullscreenPlayer IIFE (index.html lines 1178-1430). The codebase already has dominant-color extraction, swipe gestures (down/left/right), heart-button with optimistic like, scrubber with drag tracking, and MediaSession metadata. The work is adding four new capabilities (double-tap like overlay, queue-peek, artist tap-to-search, scrubber touch-target expansion) and one CSS tweak (cover 60vw -> 70vw).

No new libraries or frameworks are needed. All changes are vanilla CSS + JS within the existing single-file IIFE architecture. The SearchScreen IIFE currently has no public method to prefill and trigger search, so a small API extension (`SearchScreen.prefillAndSearch(query)`) is required as an integration point.

**Primary recommendation:** Implement as 3 focused waves: (1) CSS-only changes (cover size, scrubber touch target), (2) new DOM elements and interactions (heart overlay, queue-peek, artist tap), (3) integration testing for gesture conflicts and BackButton sync.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Cover 70vw, max-width 340px on wide screens
- D-02: aspect-ratio 1/1, border-radius 24px preserved
- D-03: Double-tap on cover = like; heart emoji overlay with scale 0->1.2->1->fade 600ms (Instagram pattern)
- D-04: Two taps within 300ms on #fp-cover; setTimeout to distinguish single from double
- D-05: Haptic feedback tg.HapticFeedback.impactOccurred('light') on double-tap
- D-06: Queue-peek "Dalee: {title}" showing queue[queueIndex + 1].title; hide if no next track
- D-07: Queue-peek style: 12px, var(--text-muted), ellipsis, max-width 80%
- D-08: Queue-peek subscribes to ['queue', 'queueIndex'] via Store
- D-09: Tap #fp-artist -> switch to Search tab with prefilled query, auto-trigger search
- D-10: Close fullscreen player before navigating to Search
- D-11: Scrubber touch-target >= 44px; visual thumb 14px preserved

### Claude's Discretion
- Exact CSS animation keyframes for double-tap heart overlay (z-index, timing)
- Whether debounce vs setTimeout for double-tap detection
- DOM structure for queue-peek element (inside fp-controls or separate block)

### Deferred Ideas (OUT OF SCOPE)
- "Spotify search + metadata + YouTube stream" — data source concern, not UI. Stays in backlog.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FP-1 | Large cover ~70% screen width, square | CSS change on #fp-cover: width 60vw->70vw, max-width 280px->340px. Trivial. |
| FP-2 | Dominant-color background from Canvas, fallback hash palette | Already implemented (lines 1220-1263). No changes needed. |
| FP-3 | Progress scrubber, touch-target >= 44px, time display | Scrubber exists. Needs CSS-only touch-target expansion via transparent border on thumb. |
| FP-4 | Heart-button, large hit area, fill without modal | Already implemented (lines 1304-1318). No changes needed. |
| FP-5 | Swipe-down = collapse to mini-player | Already implemented (lines 1330-1356). No changes needed. |
| FP-6 | Swipe-left/right on cover = next/prev | Already implemented (lines 1347-1355). No changes needed. |
| FP-7 | Double-tap on cover = like | New: double-tap detection + heart overlay animation on #fp-cover |
| FP-8 | Queue-peek "Dalee: ..." at bottom of FP | New: DOM element + Store subscription for queue/queueIndex |
| FP-9 | Tap artist name -> prefilled search | New: click handler on #fp-artist + SearchScreen API extension |
| FP-10 | MediaSession metadata updates per track | Already implemented in PlaybackEngine (line 967 initMediaSession). No changes needed. |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Cover resize (FP-1) | Browser / Client (CSS) | -- | Pure CSS property change |
| Dominant-color bg (FP-2) | Browser / Client (Canvas) | -- | Already implemented; Canvas getImageData on loaded img |
| Scrubber touch-target (FP-3) | Browser / Client (CSS) | -- | CSS thumb sizing, no JS changes |
| Heart button (FP-4) | Browser / Client (JS) | API (POST /library/like) | Already implemented with optimistic UI |
| Swipe-down collapse (FP-5) | Browser / Client (JS) | -- | Already implemented |
| Swipe L/R on cover (FP-6) | Browser / Client (JS) | -- | Already implemented |
| Double-tap like (FP-7) | Browser / Client (JS) | API (POST /library/like) | New touch detection + existing like flow |
| Queue-peek (FP-8) | Browser / Client (JS+CSS) | -- | Read from Store.get('queue'), render text |
| Artist tap -> search (FP-9) | Browser / Client (JS) | API (GET /search) | Router navigation + SearchScreen prefill |
| MediaSession metadata (FP-10) | Browser / Client (JS) | -- | Already implemented |

## Standard Stack

No new libraries needed. Everything is vanilla CSS/JS in a single `index.html`.

### Core (existing, no changes)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Vanilla JS | ES2020 | All logic | ARCH-5 mandates no bundler/frameworks |
| CSS Custom Properties | -- | Theming (--bg, --accent, --text-muted, --surface) | Existing design system |
| Telegram WebApp SDK | Latest via CDN | HapticFeedback, BackButton | Already loaded |

### Supporting (existing, no changes)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Lucide Icons | CDN | Icons in other components | Not used in FP (emoji icons) |
| SortableJS | CDN | Drag-to-reorder in Queue | Not relevant to this phase |

No `npm install` needed. No new CDN dependencies.

## Architecture Patterns

### System Flow: Double-Tap Like

```
User double-taps #fp-cover
    |
    v
[1st tap] -> setTimeout(300ms) registered
    |
[2nd tap within 300ms] -> clearTimeout -> fire like
    |
    +-> Show .fp-heart-overlay with .animate class
    +-> HapticFeedback.impactOccurred('light')
    +-> Optimistic: likedIds.add(t.id), Store.set({likedIds})
    +-> Api.like(t).catch(() -> rollback)
    +-> After 600ms: remove .animate class
```

### System Flow: Artist Tap -> Search

```
User taps #fp-artist
    |
    v
Router.closeFullscreen()     // close FP first
    |
    v
Router.go('search')          // navigate to search tab
    |
    v
SearchScreen.prefillAndSearch(artistName)  // NEW public method
    |
    +-> Set inputEl.value = artistName
    +-> Clear previous results
    +-> Call doSearch() immediately
```

### System Flow: Queue-Peek Updates

```
Store.subscribe(['queue', 'queueIndex'], updateQueuePeek)
    |
    v
queue = Store.get('queue')
idx   = Store.get('queueIndex')
next  = queue[idx + 1]
    |
    +-> next exists: el.textContent = "Dalee: " + next.title; el.style.display = ''
    +-> no next:     el.style.display = 'none'
```

### Recommended Template Changes

The `templateTrack()` function (line 1184) needs these additions:

1. **Cover wrapper** for heart overlay positioning:
```html
<div class="fp-cover-wrap">
  <img id="fp-cover" src="..." crossorigin="anonymous">
  <div class="fp-heart-overlay">❤️</div>
</div>
```

2. **Queue-peek element** after `.fp-feedback`:
```html
<div class="fp-queue-peek"></div>
```

3. **Artist name** becomes clickable (`data-action="search-artist"` on `#fp-artist`)

### Pattern: Double-Tap Detection (setTimeout approach per D-04)

```javascript
// Inside hydrate(), on the cover wrapper
let tapTimer = null;
const coverWrap = root.querySelector('.fp-cover-wrap');
coverWrap.addEventListener('click', (e) => {
  // Don't trigger on scrubber or buttons
  if (e.target.closest('input, [data-action]')) return;
  if (tapTimer) {
    // Second tap — double-tap detected
    clearTimeout(tapTimer);
    tapTimer = null;
    fireLikeWithOverlay();
  } else {
    tapTimer = setTimeout(() => { tapTimer = null; }, 300);
  }
});
```
[VERIFIED: codebase pattern matches D-04 decision]

### Pattern: SearchScreen API Extension

```javascript
// Add to SearchScreen IIFE return
function prefillAndSearch(query) {
  inputEl.value = query;
  clearTimeout(timer);
  doSearch();
}
return { mount, destroy, prefillAndSearch };
```
[VERIFIED: SearchScreen at line 1638-1668 currently returns { mount, destroy } only]

### Anti-Patterns to Avoid

- **Full innerHTML rewrite on tick:** The existing pattern uses surgical patching (patchPlayIcon, patchDuration). Queue-peek and heart-overlay must also use surgical patching, not template re-renders.
- **Gesture conflict with scrubber:** Double-tap detection on cover wrapper must NOT interfere with scrubber input. The click handler must check `e.target.closest('input')` to bail.
- **Stale overlay reference:** The heart overlay DOM element is re-created on every `renderTrack()`. The double-tap handler must re-query or use event delegation on root, not cache a reference.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Touch-target expansion | Complex JS hit-testing | CSS transparent border on ::-webkit-slider-thumb | Pure CSS, no JS overhead, works across browsers [ASSUMED] |
| Debounce/throttle for double-tap | Custom timing logic | setTimeout/clearTimeout (D-04) | Simple, matches existing codebase pattern, 6 lines of code |
| Prefill search | Custom event bus | Direct method call SearchScreen.prefillAndSearch() | IIFE pattern already exposes public methods; event bus is overkill |

**Key insight:** This phase is entirely incremental CSS + JS additions to an existing working component. No abstractions, no new patterns — follow what's already there.

## Common Pitfalls

### Pitfall 1: Double-Tap Conflicts with Swipe Gestures
**What goes wrong:** Swipe detection on `touchend` and tap detection on `click` can fire in wrong order, causing a swipe to also register as a tap.
**Why it happens:** `click` fires after `touchend` on mobile. A quick horizontal swipe might be short enough to also trigger a click event.
**How to avoid:** The existing swipe handler (line 1330-1356) uses touchstart/touchend. Double-tap should use `click` events which only fire if touch didn't move significantly. The browser handles this distinction natively.
**Warning signs:** Swiping left/right on cover also triggers like animation.

### Pitfall 2: Heart Overlay Stale After Track Change
**What goes wrong:** renderTrack() rewrites innerHTML, destroying the overlay mid-animation. If a track changes during the 600ms animation, the overlay disappears abruptly.
**Why it happens:** renderTrack subscribes to currentTrack and calls innerHTML = templateTrack(...).
**How to avoid:** The 600ms animation is short enough that this is unlikely to matter. If needed, check if .animate class is present before renderTrack and let animation complete, or simply accept the cut (track change is user-initiated).
**Warning signs:** Heart overlay flickers on rapid track skipping.

### Pitfall 3: Queue-Peek Shows Stale Data
**What goes wrong:** Queue-peek shows wrong "next track" after queue reorder or wave refill.
**Why it happens:** Subscribing only to `queueIndex` but not `queue` array changes.
**How to avoid:** Subscribe to BOTH `['queue', 'queueIndex']` as specified in D-08. Store uses reference equality, so `Store.set({ queue: newQueue })` triggers subscribers.
**Warning signs:** "Dalee:" shows a track that already played.

### Pitfall 4: SearchScreen.prefillAndSearch Called Before Mount
**What goes wrong:** `inputEl` is null because SearchScreen.mount() hasn't been called yet.
**Why it happens:** On cold start, user opens FP before ever visiting Search tab.
**How to avoid:** SearchScreen.mount() is called during Boot (line 1836). Verify mount order: Boot mounts all screens unconditionally, so inputEl is always available. If defensive coding is needed, add a null check.
**Warning signs:** TypeError: Cannot set property 'value' of null.

### Pitfall 5: CSS range thumb transparent border breaks on Firefox/Samsung Browser
**What goes wrong:** The transparent-border trick for expanding touch targets may not work identically across all mobile browsers.
**Why it happens:** Range input styling is not fully standardized; ::-webkit-slider-thumb only targets WebKit/Blink.
**How to avoid:** Also add ::-moz-range-thumb styles. Alternatively, use the wrapper padding approach (wrap input in div with padding 12px 0) which is more reliable cross-browser.
**Warning signs:** Scrubber hard to tap on non-Chrome browsers.

## Code Examples

### Cover Resize (FP-1) — CSS only
```css
/* Source: D-01, D-02 */
#fp-cover {
    width: 70vw;
    max-width: 340px;
    /* aspect-ratio, border-radius, etc. unchanged */
}
```
[VERIFIED: current values at line 323-332 are 60vw / 280px]

### Scrubber Touch Target (FP-3) — CSS approach
```css
/* Source: UI-SPEC scrubber section + D-11 */
/* Option A: transparent border (WebKit) */
input[type=range]::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 14px; height: 14px;
    background: var(--accent);
    border-radius: 50%;
    margin-top: -5px;
    border: 15px solid transparent;
    background-clip: content-box;
    box-sizing: content-box;
    box-shadow: 0 0 10px rgba(255,51,102,0.6);
}
/* Option B (recommended, more reliable): wrapper padding */
.fp-progress-wrap {
    padding: 12px 0;  /* (14px thumb + 12px*2 padding) = 38px, close to 44px */
    margin-top: 2px;   /* adjust to compensate visual shift */
}
```
[VERIFIED: current thumb is 14px at line 352-358, track height 4px at line 349]

### Heart Overlay Animation (FP-7) — from UI-SPEC
```css
/* Source: 05-UI-SPEC.md */
.fp-cover-wrap {
    position: relative;
    display: flex;
    justify-content: center;
}
.fp-heart-overlay {
    position: absolute;
    top: 50%; left: 50%;
    transform: translate(-50%, -50%) scale(0);
    font-size: 72px;
    pointer-events: none;
    z-index: 10;
    opacity: 0;
}
.fp-heart-overlay.animate {
    animation: fp-heart-pop 600ms cubic-bezier(0.2, 0.8, 0.2, 1) forwards;
}
@keyframes fp-heart-pop {
    0%   { transform: translate(-50%, -50%) scale(0); opacity: 1; }
    30%  { transform: translate(-50%, -50%) scale(1.2); opacity: 1; }
    50%  { transform: translate(-50%, -50%) scale(1); opacity: 1; }
    100% { transform: translate(-50%, -50%) scale(1); opacity: 0; }
}
```
[VERIFIED: matches UI-SPEC exactly]

### Double-Tap Detection (FP-7)
```javascript
// Source: D-03, D-04, D-05 — inside hydrate()
let doubleTapTimer = null;
const coverWrap = root.querySelector('.fp-cover-wrap');
if (coverWrap) {
  coverWrap.addEventListener('click', (e) => {
    if (e.target.closest('input, [data-action]')) return;
    if (doubleTapTimer) {
      clearTimeout(doubleTapTimer);
      doubleTapTimer = null;
      // Fire like
      const t = Store.get('currentTrack');
      if (!t) return;
      // Show overlay
      const overlay = root.querySelector('.fp-heart-overlay');
      if (overlay) {
        overlay.classList.remove('animate');
        void overlay.offsetWidth; // force reflow to restart animation
        overlay.classList.add('animate');
        setTimeout(() => overlay.classList.remove('animate'), 600);
      }
      // Haptic
      if (window.Telegram?.WebApp?.HapticFeedback) {
        Telegram.WebApp.HapticFeedback.impactOccurred('light');
      }
      // Optimistic like (reuse existing pattern from line 1304-1318)
      const liked = Store.get('likedIds');
      liked.add(t.id);
      Store.set({ likedIds: new Set(liked) });
      Api.like(t).catch(() => {
        liked.delete(t.id);
        Store.set({ likedIds: new Set(liked) });
      });
    } else {
      doubleTapTimer = setTimeout(() => { doubleTapTimer = null; }, 300);
    }
  });
}
```
[VERIFIED: matches existing like pattern at lines 1304-1318]

### Queue-Peek (FP-8)
```javascript
// Source: D-06, D-07, D-08 — inside mount(), after renderTrack subscription
const unsubQueuePeek = Store.subscribe(['queue', 'queueIndex'], () => {
  const el = root.querySelector('.fp-queue-peek');
  if (!el) return;
  const queue = Store.get('queue');
  const idx = Store.get('queueIndex');
  const next = queue[idx + 1];
  if (next) {
    el.textContent = '\u0414\u0430\u043B\u0435\u0435: ' + next.title;
    el.style.display = '';
  } else {
    el.style.display = 'none';
  }
});
```
[VERIFIED: Store keys queue/queueIndex exist per lines 564-565]

### Artist Tap -> Search (FP-9)
```javascript
// Source: D-09, D-10 — add case in hydrate() onclick handler
case 'search-artist': {
  const t = Store.get('currentTrack');
  if (!t || !t.artist) break;
  Router.closeFullscreen();
  Router.go('search');
  SearchScreen.prefillAndSearch(t.artist);
  break;
}
```
[VERIFIED: Router.go('search') and Router.closeFullscreen() both exist]

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Scrubber hit area via JS touch detection | CSS transparent border / padding wrapper | Widely adopted | Simpler, no JS overhead |
| Custom gesture library for double-tap | setTimeout approach inline | Standard for single-component use | No dependency needed for one gesture |

**Deprecated/outdated:**
- None relevant. The vanilla JS/CSS approach used in this project is stable and not subject to framework churn.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | CSS transparent border on range thumb works in Telegram WebView (WebKit-based) | Scrubber Touch Target | LOW -- fallback is wrapper padding approach, which is reliable everywhere |
| A2 | `void overlay.offsetWidth` reflow trick restarts CSS animation reliably on all targets | Heart Overlay | LOW -- well-known pattern, widely used |
| A3 | SearchScreen.mount() is called before any FP interaction can trigger prefillAndSearch | Artist Tap | MEDIUM -- verify mount order in Boot section |

## Open Questions

1. **SearchScreen mount timing**
   - What we know: Boot section (line 1836) calls `SearchScreen.mount(...)` unconditionally
   - What's unclear: Is there a race condition if user opens FP immediately on cold start before Boot completes?
   - Recommendation: Add defensive null check in prefillAndSearch. Risk is minimal since Boot is synchronous.

2. **Cover wrapper impact on existing swipe gestures**
   - What we know: Swipe detection (line 1330-1356) checks `cover.getBoundingClientRect()` for Y bounds
   - What's unclear: Wrapping img in `.fp-cover-wrap` changes DOM structure; swipe handler queries `#fp-cover` directly
   - Recommendation: Swipe handler queries `#fp-cover` by ID which will still work inside the wrapper. Verify after implementation.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual testing (no automated test framework in project) |
| Config file | none -- single index.html, ARCH-5 mandates no build tools |
| Quick run command | Open in Telegram WebApp, interact with FP |
| Full suite command | Manual device matrix test |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FP-1 | Cover 70vw / max 340px | manual | DevTools responsive check | N/A |
| FP-2 | Dominant-color bg <= 300ms | manual | Performance panel timeline | N/A |
| FP-3 | Scrubber touch >= 44px | manual | DevTools element inspect + finger test | N/A |
| FP-4 | Heart-button optimistic like | manual | Tap heart, check network + UI | N/A |
| FP-5 | Swipe-down collapses | manual | Swipe gesture on device | N/A |
| FP-6 | Swipe L/R = next/prev | manual | Swipe on cover area | N/A |
| FP-7 | Double-tap = like with overlay | manual | Double-tap cover, observe animation | N/A |
| FP-8 | Queue-peek shows next track | manual | Play track, verify "Dalee: ..." text | N/A |
| FP-9 | Artist tap -> search | manual | Tap artist in FP, verify search tab opens with query | N/A |
| FP-10 | MediaSession metadata | manual | Check lock screen / notification shade | N/A |

### Sampling Rate
- **Per task commit:** Open in browser, verify changed behavior
- **Per wave merge:** Full manual walkthrough of all FP interactions
- **Phase gate:** Test on real Android device in Telegram WebApp

### Wave 0 Gaps
None -- no automated test infrastructure exists or is expected (ARCH-5: no bundler/npm).

## Security Domain

This phase is client-only UI changes with no new API endpoints, no new data flows, and no authentication changes. Existing `Api.like()` endpoint is already wired.

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | N/A (no auth changes) |
| V3 Session Management | no | N/A |
| V4 Access Control | no | N/A |
| V5 Input Validation | no | Artist name from Store (server-provided), not user input; escapeHtml already applied |
| V6 Cryptography | no | N/A |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| XSS via artist name in search prefill | Tampering | SearchScreen sets inputEl.value (DOM property, not innerHTML) -- safe by default |
| Cover image CORS bypass | Information Disclosure | Already handled: crossorigin="anonymous" + Canvas catch block (line 1222-1236) |

## Sources

### Primary (HIGH confidence)
- Codebase: index.html lines 1178-1430 (FullscreenPlayer IIFE) -- verified all existing features
- Codebase: index.html lines 1637-1668 (SearchScreen IIFE) -- verified public API
- Codebase: index.html lines 991-1037 (Router IIFE) -- verified navigation methods
- 05-CONTEXT.md -- all locked decisions D-01 through D-11
- 05-UI-SPEC.md -- CSS specs, animation keyframes, DOM structure

### Secondary (MEDIUM confidence)
- REQUIREMENTS.md section 4 -- FP-1 through FP-10 acceptance criteria

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - no new libraries, purely extending existing code
- Architecture: HIGH - all patterns verified in codebase, integration points identified
- Pitfalls: HIGH - based on direct code analysis, not hypothetical

**Research date:** 2026-04-16
**Valid until:** 2026-05-16 (stable -- no framework dependencies to age)
