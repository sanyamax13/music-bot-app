# Phase 4: Mini-Player Polish - Research

**Researched:** 2026-04-16
**Domain:** Persistent mini-player UI enhancements (heart-button, buffering indicator)
**Confidence:** HIGH

## Summary

Phase 4 scope is narrower than originally planned: Phase 3 already implemented persistence, hairline progress (rAF), swipe gestures (up/left/right), and tap-to-expand. What remains is adding two features: (1) a heart-button for quick-like with optimistic UI, and (2) a buffering indicator that replaces the play/pause icon with a spinner during `waiting` events.

The existing MiniPlayer IIFE (line 1013-1111) is well-structured with `templateTrack()`, `patchPlayIcon()`, `tickProgress()` via rAF, and `hydrate()` for touch events. The codebase already has a like pattern in FullscreenPlayer (line 1239-1244): optimistic emoji swap `'🤍'` -> `'❤️'`, POST `Api.like(t)`, rollback on error. This pattern is directly reusable.

For buffering, `PlaybackEngine._onWaiting()` and `_onStalled()` are currently stubs (lines 752-759) with comments saying "Phase 4 concern". The cleanest approach is to add a Store key `isBuffering` set by PlaybackEngine, then subscribe in MiniPlayer to swap the play-icon for a CSS spinner.

**Primary recommendation:** Add `isBuffering` Store key driven by PlaybackEngine events, add heart-button to MiniPlayer template, CSS spinner for buffering -- all following established IIFE/Store/surgical-patch patterns.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Heart-button placement: thumb -> info -> heart -> play (Spotify pattern)
- **D-02:** Optimistic UI: heart fills immediately, POST `/library/like` in background, rollback on error. Extended in Phase 8 (FB-1), base mechanics here.
- **D-03:** Heart icon: emoji `🤍` (empty) / `❤️` (filled), matching FullscreenPlayer (line 1130)
- **D-04:** Buffering replaces play-icon content in `#mp-play-btn` with CSS spinner on `waiting`, restores play/pause on `canplay`. No extra DOM element.
- **D-05:** CSS spinner: small circular border-based, ~20px, color `var(--accent)`
- **D-06:** Only adding missing features (heart, buffering). Visual restyle deferred to Phase 13.
- **D-07:** On screens < 340px, hide heart-button via media query to prevent overflow.

### Claude's Discretion
- CSS animation for spinner appear/disappear (fade vs instant)
- Touch-target size for heart-button (>= 44px per WCAG)
- Whether to use a new Store key `isLiked` or check via existing mechanism

### Deferred Ideas (OUT OF SCOPE)
None
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| MINI-1 | Persistent, visible when currentTrack exists, on all screens, above BottomNav | Already implemented in Phase 3 (MiniPlayer IIFE line 1013). Verify no regression. |
| MINI-2 | Tap = expand FP; swipe-up = expand FP | Already implemented (line 1072, 1085-1092). Verify no regression after template change. |
| MINI-3 | Swipe-left/right = prev/next | Already implemented (line 1077-1081). Verify no regression. |
| MINI-4 | Hairline progress ~2px top, rAF-driven | Already implemented (tickProgress line 1042-1055, CSS line 263-269). Verify no regression. |
| MINI-5 | Heart-button for quick like without expanding FP | New: add to template, wire click handler, optimistic UI pattern from FP |
| MINI-6 | Buffering indicator: play-icon -> spinner on `waiting`, back on `canplay` | New: add `isBuffering` Store key, CSS spinner, subscribe in MiniPlayer |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Heart-button UI | Browser / Client (MiniPlayer IIFE) | -- | Pure UI: emoji swap, click handler, Store subscription |
| Like API call | Browser / Client (Api module) | API / Backend | POST `/like` already exists; frontend does optimistic update |
| Liked state tracking | Browser / Client (Store) | -- | `likedIds` Set already in Store (line 564), needs population |
| Buffering detection | Browser / Client (PlaybackEngine) | -- | `waiting`/`canplay`/`playing` events on HTMLAudioElement |
| Buffering UI | Browser / Client (MiniPlayer IIFE) | -- | CSS spinner replaces play icon via Store subscription |

## Standard Stack

No new libraries needed. This phase works entirely within existing patterns:

### Core (already in project)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Vanilla JS IIFE | -- | Component pattern | ARCH-5: no bundler, no framework |
| Store (pub/sub) | -- | State management | ARCH-1: key-scoped subscribe |
| PlaybackEngine | -- | Audio control + events | ARCH-2: single audio owner |
| CSS custom properties | -- | Theming | `--accent`, `--surface-2` etc. |

### Alternatives Considered
None. Phase scope is pure enhancement of existing components -- no new dependencies.

## Architecture Patterns

### System Flow: Heart-Button

```
User taps heart in MiniPlayer
  -> onclick handler checks data-action="like"
  -> Optimistic: set emoji to '❤️', add to Store.likedIds
  -> Api.like(track) POST in background
  -> On success: noop (already updated)
  -> On failure: rollback emoji to '🤍', remove from Store.likedIds
```

### System Flow: Buffering Indicator

```
Audio element fires 'waiting' event
  -> PlaybackEngine._onWaiting() sets Store.set({ isBuffering: true })
  -> MiniPlayer subscribes to ['isBuffering']
  -> patchPlayIcon() checks isBuffering:
     - true: show CSS spinner
     - false: show play/pause emoji

Audio element fires 'playing' or 'canplay' event
  -> PlaybackEngine sets Store.set({ isBuffering: false })
  -> MiniPlayer restores play/pause icon
```

### Pattern 1: Surgical Icon Patching (existing pattern)
**What:** Update a single DOM element without re-rendering the entire template [VERIFIED: index.html line 1037-1039]
**When to use:** Any time a single visual property changes (isPlaying, isBuffering, isLiked)
**Example:**
```javascript
// Source: existing patchPlayIcon pattern (line 1037)
function patchPlayIcon() {
    const btn = root.querySelector('#mp-play-btn');
    if (!btn) return;
    if (Store.get('isBuffering')) {
        btn.innerHTML = '<span class="mp-spinner"></span>';
    } else {
        btn.innerText = Store.get('isPlaying') ? '⏸' : '▶';
    }
}
```

### Pattern 2: Optimistic Like (existing pattern)
**What:** Immediately update UI, POST in background, rollback on error [VERIFIED: index.html line 1239-1244]
**When to use:** Like/unlike actions where latency would feel bad
**Example:**
```javascript
// Source: FullscreenPlayer like handler (line 1239-1244)
case 'like': {
    const t = Store.get('currentTrack');
    if (!t) break;
    const heart = root.querySelector('#mp-heart-btn');
    if (heart) {
        heart.innerText = '❤️';
        heart.style.transform = 'scale(1.3)';
        setTimeout(() => heart.style.transform = '', 200);
    }
    Api.like(t).catch(() => { if (heart) heart.innerText = '🤍'; });
    break;
}
```

### Pattern 3: Store Subscription for Derived State
**What:** Subscribe to a Store key, run a patch function on change [VERIFIED: index.html line 1097-1098]
**Example:**
```javascript
// In mount():
unsubBuffering = Store.subscribe(['isBuffering'], patchPlayIcon);
// isPlaying subscription already calls patchPlayIcon, so merge them
unsubPlaying = Store.subscribe(['isPlaying'], patchPlayIcon);
```

### Anti-Patterns to Avoid
- **Re-rendering full template on state change:** MiniPlayer calls `renderTrack()` (full innerHTML) only on track change. Play/pause/buffering/like MUST use surgical patching, not re-render. [VERIFIED: existing code follows this pattern]
- **Listening to audio element directly from components:** Components MUST go through Store, not `document.getElementById('audioA')`. PlaybackEngine is the sole audio owner (ARCH-2). [VERIFIED: REQUIREMENTS.md ARCH-2]
- **Adding touch listeners outside hydrate():** All touch/click handlers go in `hydrate()` which runs after innerHTML update. [VERIFIED: existing pattern line 1057-1093]

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| CSS spinner | JS-animated spinner, canvas, SVG with JS | Pure CSS border-based spinner | 5 lines of CSS, no JS, no perf cost [ASSUMED] |
| Like state sync | Custom tracking of liked tracks | Store.likedIds Set + Api.like() | Already exists in codebase (line 564) [VERIFIED: index.html] |
| Debounced like | setTimeout-based debounce on like | Direct POST with rollback | Like is idempotent on backend, no debounce needed [ASSUMED] |

## Common Pitfalls

### Pitfall 1: Hydrate() Event Listener Accumulation
**What goes wrong:** `hydrate()` is called inside `renderTrack()` which runs on every track change. Each call adds new event listeners without removing old ones.
**Why it happens:** `renderTrack()` does `root.innerHTML = templateTrack(t)` which destroys old DOM, but event listeners are on `root` (the container), not the inner elements.
**How to avoid:** Either (a) move event delegation to `mount()` instead of `hydrate()`, or (b) use `{ once: true }` where appropriate, or (c) remove old listeners before adding new.
**Warning signs:** Tap heart triggers multiple API calls. Play/pause fires twice.
[VERIFIED: index.html line 1029-1034 shows hydrate() called on every renderTrack()]

### Pitfall 2: isBuffering Not Cleared on Track Change
**What goes wrong:** If user skips track while buffering, `isBuffering` stays `true` because `_onWaiting` was fired but `_onPlaying` never fires for the old track.
**Why it happens:** `_wireEvents()` removes old listeners and adds to new `activeAudio`, but no explicit reset of `isBuffering`.
**How to avoid:** Reset `isBuffering` to `false` in `playAt()` when starting a new track.
**Warning signs:** Spinner stuck on mini-player after skip.
[VERIFIED: index.html line 810-848 playAt() does not reset any buffering state]

### Pitfall 3: Heart State Not Synced Between Mini and Fullscreen Player
**What goes wrong:** User likes in MiniPlayer, opens FullscreenPlayer -- heart still shows `🤍`.
**Why it happens:** Each component reads its own DOM state, not a shared Store key.
**How to avoid:** Use `Store.likedIds` Set as source of truth. Both components subscribe to it. When like succeeds, add track.id to `likedIds`. On render, check `likedIds.has(track.id)`.
**Warning signs:** Heart state differs between mini and fullscreen.
[VERIFIED: Store has `likedIds: new Set()` at line 564, but FullscreenPlayer does not use it -- line 1130 always renders '🤍']

### Pitfall 4: Click-Through on Heart Expands Fullscreen
**What goes wrong:** Tapping heart also triggers the `Router.openFullscreen()` catch-all click handler.
**Why it happens:** Current `root.onclick` (line 1085) opens fullscreen for any click not on `[data-action="toggle"]`.
**How to avoid:** Add `data-action="like"` to heart button, check for it in the onclick handler before the fallback.
**Warning signs:** Tap heart -> FP opens unexpectedly.
[VERIFIED: index.html line 1085-1092]

### Pitfall 5: Mini-Player Overflow on Narrow Screens
**What goes wrong:** Adding heart-button makes content too wide for 320px screens.
**Why it happens:** Current layout: thumb(42px) + margin(12px) + info(flex) + play-btn(~46px) = fits. Adding heart-btn(44px) may overflow.
**How to avoid:** D-07 mandates hiding heart on < 340px via media query. Use `@media (max-width: 339px) { #mp-heart-btn { display: none; } }`.
**Warning signs:** Text truncation breaks or elements wrap.
[VERIFIED: CSS line 233-269 shows flex layout with no explicit width constraints]

## Code Examples

### Heart Button Template Addition
```javascript
// Source: pattern from FullscreenPlayer (line 1130) adapted for MiniPlayer
function templateTrack(t) {
    if (!t) return '';
    const liked = Store.get('likedIds').has(t.id);
    return `
      <img id="mp-thumb" src="${Utils.escapeHtml(t.thumb || '')}">
      <div class="mp-info">
        <div id="mp-title">${Utils.escapeHtml(t.title || '')}</div>
        <div id="mp-artist">${Utils.escapeHtml(t.artist || '')}</div>
      </div>
      <div id="mp-heart-btn" data-action="like" style="font-size:20px;padding:6px;">${liked ? '❤️' : '🤍'}</div>
      <div id="mp-play-btn" data-action="toggle">${Store.get('isPlaying') ? '⏸' : '▶'}</div>
      <div id="mp-progress"></div>
    `;
}
```

### CSS Spinner for Buffering
```css
/* Source: standard CSS-only spinner pattern [ASSUMED] */
.mp-spinner {
    display: inline-block;
    width: 20px; height: 20px;
    border: 2px solid rgba(255,255,255,0.2);
    border-top-color: var(--accent);
    border-radius: 50%;
    animation: mp-spin 0.6s linear infinite;
}
@keyframes mp-spin {
    to { transform: rotate(360deg); }
}
```

### PlaybackEngine Buffering State
```javascript
// Added to PlaybackEngine (modify existing stubs at line 752-759)
function _onWaiting() {
    Store.set({ isBuffering: true });
}
function _onPlaying() {
    Store.set({ isBuffering: false });
    if (!Store.get('isPlaying')) Store.set({ isPlaying: true });
}
// Also in _onStalled — optionally set isBuffering: true
```

### Click Handler Update
```javascript
// Updated onclick in hydrate() — add 'like' action
root.onclick = (e) => {
    if (e.target.closest('[data-action="like"]')) {
        e.stopPropagation();
        const t = Store.get('currentTrack');
        if (!t) return;
        const heart = root.querySelector('#mp-heart-btn');
        if (heart) {
            heart.innerText = '❤️';
            heart.style.transform = 'scale(1.3)';
            setTimeout(() => heart.style.transform = '', 200);
        }
        Api.like(t).catch(() => { if (heart) heart.innerText = '🤍'; });
    } else if (e.target.closest('[data-action="toggle"]')) {
        e.stopPropagation();
        PlaybackEngine.toggle();
    } else {
        Router.openFullscreen();
    }
};
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Full re-render on state change | Surgical DOM patching | Phase 1 (ARCH-3) | 60fps maintained during playback |
| Direct audio element access | Store-mediated state | Phase 1-2 (ARCH-1, ARCH-2) | Components decoupled from audio |
| No like in mini-player | Heart-button with optimistic UI | This phase (MINI-5) | Quick like without expanding FP |
| No buffering indication | CSS spinner on waiting event | This phase (MINI-6) | User knows when stream is buffering |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Like API (`/like` POST) is idempotent -- multiple likes for same track don't cause errors | Don't Hand-Roll | If not idempotent, need client-side dedup. Low risk -- backend likely handles this. |
| A2 | CSS border-based spinner at 20px is sufficient visual feedback | Code Examples | If too small or unclear, may need SVG. Very low risk. |
| A3 | `Store.get('likedIds')` Set is populated at boot from `/library/likes` | Architecture Patterns | If not populated, heart state will always show empty on initial load. Needs verification in Phase 8 boot sequence, but safe to code against now. |

## Open Questions

1. **likedIds population timing**
   - What we know: `Store` has `likedIds: new Set()` (line 564) but no code currently populates it from `/library/likes`.
   - What's unclear: Should Phase 4 add the boot-time population of `likedIds`, or leave that to Phase 8 (Feedback UX)?
   - Recommendation: Add a minimal `likedIds` population in Boot or as a lazy-load on first MiniPlayer render. Without it, heart will always render empty until user likes in this session.

2. **FullscreenPlayer heart sync**
   - What we know: FP always renders `🤍` (line 1130), doesn't check `likedIds`. MiniPlayer will use `likedIds`.
   - What's unclear: Should Phase 4 also fix FP to use `likedIds`, or leave inconsistency for Phase 8?
   - Recommendation: Fix both in Phase 4 since the pattern is identical and prevents confusing UX.

3. **canplay vs playing for buffering-end**
   - What we know: `canplay` fires when enough data is buffered. `playing` fires when playback actually resumes.
   - What's unclear: Which is the better signal for removing spinner?
   - Recommendation: Use `playing` event (already wired at line 717) as the definitive "not buffering" signal. `canplay` can fire before playback resumes. The existing `_onPlaying` handler is the right place.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual testing (no automated test framework -- ARCH-5 single HTML file) |
| Config file | none |
| Quick run command | Open in Telegram WebApp or browser, verify interactions |
| Full suite command | Manual device testing per PERF-4 |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| MINI-1 | Persistent across screens | manual | Navigate Home/Library/Search/Profile with track playing | N/A |
| MINI-2 | Tap/swipe-up expand FP | manual | Tap mini-player, swipe up | N/A |
| MINI-3 | Swipe left/right prev/next | manual | Swipe on mini-player | N/A |
| MINI-4 | Hairline progress 60fps | manual | Play track, observe progress bar smoothness | N/A |
| MINI-5 | Heart-button quick like | manual | Tap heart, verify emoji changes, verify API call in network tab | N/A |
| MINI-6 | Buffering spinner | manual | Throttle network to Slow 3G, play track, observe spinner | N/A |

### Sampling Rate
- **Per task:** Open in browser, verify the specific change
- **Phase gate:** Full manual check of all 6 MINI requirements on throttled network

### Wave 0 Gaps
None -- no automated test infrastructure exists per ARCH-5 (single HTML file, no bundler). All validation is manual.

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | -- |
| V3 Session Management | no | -- |
| V4 Access Control | no | user_id from Telegram initData (TG-6 trade-off) |
| V5 Input Validation | yes | `Utils.escapeHtml()` on all track data in templates [VERIFIED: line 1019-1022] |
| V6 Cryptography | no | -- |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| XSS via track title/artist in template | Tampering | `Utils.escapeHtml()` already applied [VERIFIED] |
| Like spam (rapid taps) | Denial of Service | Backend should rate-limit; frontend can disable button briefly [ASSUMED] |

## Sources

### Primary (HIGH confidence)
- index.html lines 233-269, 427-434, 564, 658-659, 667-947, 1013-1111, 1113-1147, 1239-1244 -- current MiniPlayer, PlaybackEngine, FullscreenPlayer, Store, Api code
- .planning/REQUIREMENTS.md -- MINI-1..6 acceptance criteria
- .planning/phases/04-mini-player-polish/04-CONTEXT.md -- locked decisions D-01..D-07

### Secondary (MEDIUM confidence)
- HTMLAudioElement events (waiting, canplay, playing) -- standard Web API behavior [ASSUMED based on MDN knowledge]

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new libraries, all existing patterns
- Architecture: HIGH -- code fully read, patterns verified in codebase
- Pitfalls: HIGH -- derived from actual code analysis (event accumulation, state sync, click delegation)

**Research date:** 2026-04-16
**Valid until:** 2026-05-16 (stable -- no external dependencies)
