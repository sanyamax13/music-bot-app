# Phase 5: Fullscreen Player Redesign - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-16
**Phase:** 05-fullscreen-player-redesign
**Areas discussed:** Cover sizing, Double-tap interaction, Queue-peek design, Artist tap behavior, Scrubber touch target
**Mode:** --auto (all decisions auto-selected)

---

## Cover Sizing

| Option | Description | Selected |
|--------|-------------|----------|
| 70vw, max-width 340px | Matches FP-1 spec, capped for tablets | ✓ |
| 70vw, no cap | Unlimited width | |
| Keep 60vw/280px | No change | |

**User's choice:** [auto] 70vw with 340px cap (recommended default)

---

## Double-tap Like Animation

| Option | Description | Selected |
|--------|-------------|----------|
| Heart overlay scales in + fades | Instagram/Spotify pattern | ✓ |
| Heart emoji flashes at tap point | Position-aware | |
| No animation, just fill heart | Minimal | |

**User's choice:** [auto] Heart overlay with scale animation (recommended default)

---

## Queue-peek Design

| Option | Description | Selected |
|--------|-------------|----------|
| Single line "Далее: {title}" | Below controls, minimal | ✓ |
| Two-line with thumb | Next track card mini-view | |
| No queue-peek | Omit | |

**User's choice:** [auto] Single line (recommended default)

---

## Artist Tap Behavior

| Option | Description | Selected |
|--------|-------------|----------|
| Switch to Search + prefill + auto-search | Full navigation | ✓ |
| Open search overlay | In-place | |
| Copy artist name | Clipboard | |

**User's choice:** [auto] Switch to Search tab with prefilled query (recommended default)

---

## Claude's Discretion

- Double-tap animation CSS keyframes
- Queue-peek DOM structure
- Debounce strategy for double-tap detection

## Deferred Ideas

- Spotify search + metadata todo — backlog item, not UI scope
