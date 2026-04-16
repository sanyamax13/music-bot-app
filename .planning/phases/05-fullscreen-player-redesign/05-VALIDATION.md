---
phase: 5
slug: fullscreen-player-redesign
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-16
---

# Phase 5 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual browser testing (no test framework — single index.html project) |
| **Config file** | none |
| **Quick run command** | `Open index.html in Telegram WebView or browser` |
| **Full suite command** | `Manual acceptance checklist below` |
| **Estimated runtime** | ~10 minutes manual |

---

## Sampling Rate

- **After every task commit:** Visual inspection of fullscreen player in browser
- **After each wave:** Full acceptance checklist run

---

## Acceptance Checklist

| ID | Check | How to verify |
|----|-------|---------------|
| FP-1 | Cover ~70% width | Open FP, measure cover visually vs viewport |
| FP-2 | Dominant-color bg changes on track switch | Skip tracks, observe gradient change ≤300ms |
| FP-3 | Scrubber touch-target ≥ 44px | Tap on scrubber area, verify responsive |
| FP-4 | Heart button toggles like | Tap heart, verify POST in Network |
| FP-5 | Swipe-down collapses FP | Swipe down on FP |
| FP-6 | Swipe left/right on cover = next/prev | Swipe on album art |
| FP-7 | Double-tap on cover = like with heart overlay | Double-tap cover, verify animation |
| FP-8 | Queue-peek shows next track | Check "Далее: ..." text below controls |
| FP-9 | Tap artist → Search with prefilled query | Tap artist name, verify search screen |
| FP-10 | MediaSession metadata updates | Check lock screen controls |

---

## Regression Checks

| Check | How to verify |
|-------|---------------|
| Mini-player still works | Navigate with track playing |
| Swipe gestures no conflict | Swipe in various directions |
| BackButton precedence | Press back from FP, verify close |
| Heart sync MP ↔ FP | Like in FP, check MP heart |
