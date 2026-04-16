---
phase: 4
slug: mini-player-polish
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-16
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual browser testing (no test framework — single index.html project) |
| **Config file** | none |
| **Quick run command** | `Open index.html in Telegram WebView or browser` |
| **Full suite command** | `Manual acceptance checklist below` |
| **Estimated runtime** | ~5 minutes manual |

---

## Sampling Rate

- **After every task commit:** Visual inspection of mini-player in browser
- **After each wave:** Full acceptance checklist run

---

## Acceptance Checklist

| ID | Check | How to verify |
|----|-------|---------------|
| MINI-1 | Mini-player visible on all screens when track playing | Navigate Wave→Search→Library→Profile with track playing |
| MINI-2 | Tap and swipe-up both open fullscreen | Tap mini-player, swipe up on mini-player |
| MINI-3 | Swipe left/right = next/prev | Swipe left, swipe right on mini-player |
| MINI-4 | Hairline progress animates smoothly | Watch progress bar during playback, check 60fps |
| MINI-5 | Heart button toggles like state | Tap heart, verify emoji changes, check network for POST |
| MINI-6 | Buffering spinner shows during loading | Throttle network, observe spinner replacing play icon |

---

## Regression Checks

| Check | How to verify |
|-------|---------------|
| Play/pause still works | Tap play button during playback |
| Swipe gestures no conflict | Swipe in various directions, verify correct action |
| Fullscreen player heart syncs | Like in mini-player, open fullscreen, verify heart filled |
| No event listener accumulation | Skip 10 tracks rapidly, check no duplicate handlers in console |
