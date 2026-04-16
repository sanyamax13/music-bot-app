---
phase: 04-mini-player-polish
plan: 01
subsystem: mini-player, fullscreen-player, playback-engine
tags: [heart-button, buffering-spinner, likedIds, optimistic-ui]
dependency_graph:
  requires: [phase-3-gesture-foundation]
  provides: [mini-player-heart, buffering-indicator, likedIds-population, heart-sync]
  affects: [index.html]
tech_stack:
  added: []
  patterns: [optimistic-ui-with-rollback, store-subscription-patch, css-spinner-animation]
key_files:
  created: []
  modified: [index.html]
decisions:
  - "Used emoji hearts (U+2764 / U+1F90D) matching existing FP pattern instead of SVG icons"
  - "Non-blocking Api.likes() in Boot() - does not delay initial render"
  - "likedIds synced via Store Set replacement (new Set) to trigger subscriber notifications"
metrics:
  duration: "2m 47s"
  completed: "2026-04-16T10:07:21Z"
  tasks_completed: 3
  tasks_total: 3
  files_changed: 1
---

# Phase 4 Plan 1: Mini-Player Heart-Button & Buffering Spinner Summary

Heart-button for quick-like with optimistic UI and rollback, buffering spinner replacing play icon during waiting events, likedIds population from API at boot, and cross-component heart sync between MiniPlayer and FullscreenPlayer.

## Task Completion

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | isBuffering Store key + PlaybackEngine buffering events + CSS | 076cdcd | index.html (Store, PlaybackEngine, CSS) |
| 2 | MiniPlayer heart-button + buffering UI + click handler | 81d6d31 | index.html (MiniPlayer IIFE) |
| 3 | Populate likedIds at boot + sync FullscreenPlayer heart state | b380ced | index.html (Boot, FullscreenPlayer IIFE) |

## Changes Made

### Task 1: Store + PlaybackEngine + CSS
- Added `isBuffering: false` to Store initial state
- Implemented `_onStalled()` and `_onWaiting()` to set `isBuffering: true`
- Updated `_onPlaying()` to reset `isBuffering: false` before syncing isPlaying
- Added `isBuffering` reset in `playAt()` to prevent stuck spinner on track skip
- Added `#mp-heart-btn` CSS: 44px touch target, hidden below 340px, scale animation on active
- Added `.mp-spinner` CSS: 20px border-based spinner with `var(--accent)` color

### Task 2: MiniPlayer IIFE
- Added heart-button (`#mp-heart-btn`) between `.mp-info` and `#mp-play-btn` in templateTrack
- Heart checks `Store.get('likedIds').has(t.id)` for initial state
- `patchPlayIcon()` shows `<span class="mp-spinner">` during isBuffering
- Added `patchHeartIcon()` for surgical heart state updates
- Optimistic like handler: immediate red heart + scale animation, `likedIds.add()`, `Store.set()`, rollback on API failure
- `e.stopPropagation()` on like click prevents opening fullscreen player
- Added `unsubBuffering` and `unsubLiked` subscriptions in mount(), cleanup in destroy()

### Task 3: Boot + FullscreenPlayer sync
- Non-blocking `Api.likes().then()` in Boot populates `likedIds` Set before Router.go
- FP `templateTrack` checks `likedIds.has(t.id)` for correct initial heart emoji
- FP like handler now updates `likedIds` Set (add + Store.set, delete + Store.set on failure)
- FP mount subscribes to `['likedIds']` to patch heart when liked from MiniPlayer
- Added `unsubLiked` to FP let declaration and destroy() cleanup

## Deviations from Plan

None - plan executed exactly as written.

## Known Stubs

None - all functionality is fully wired.

## Verification Checklist

- [x] Heart button visible in mini-player between track info and play button
- [x] Tapping heart fills it red immediately without opening fullscreen player
- [x] Heart reverts to white if API call fails (catch block)
- [x] Play icon replaced by spinner when audio is buffering (isBuffering subscription)
- [x] Spinner disappears when audio resumes (_onPlaying resets isBuffering)
- [x] Heart state consistent between mini-player and fullscreen player (likedIds Store sync)
- [x] Mini-player does not overflow on 320px screens (heart hidden below 340px)
- [x] Existing gestures (tap, swipe up/left/right) still work (hydrate unchanged for touch listeners)
- [x] likedIds populated from /library/likes at boot (non-blocking)
