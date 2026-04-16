# Phase 4: Mini-Player Polish - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-16
**Phase:** 04-mini-player-polish
**Areas discussed:** Heart button placement, Buffering indicator, Visual polish scope
**Mode:** --auto (all decisions auto-selected)

---

## Heart Button Placement

| Option | Description | Selected |
|--------|-------------|----------|
| Between info and play | thumb → info → heart → play (Spotify pattern) | ✓ |
| Right of play button | thumb → info → play → heart | |
| Below title/artist | Stacked inside info block | |

**User's choice:** [auto] Between info and play button (recommended default)
**Notes:** Consistent with FullscreenPlayer emoji pattern (🤍/❤️)

---

## Buffering Indicator

| Option | Description | Selected |
|--------|-------------|----------|
| Replace play icon with CSS spinner | Swap content of play btn on waiting/canplay | ✓ |
| Separate spinner element | Add dedicated spinner next to play button | |
| Opacity pulse on play button | Animate existing icon opacity | |

**User's choice:** [auto] Replace play icon with CSS spinner (recommended default)
**Notes:** No extra DOM needed, clear user feedback

---

## Visual Polish Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Add missing features only | Heart + buffering, minimal style changes | ✓ |
| Light visual refresh | Small tweaks to spacing, shadows, etc. | |
| Full redesign | Complete visual overhaul of mini-player | |

**User's choice:** [auto] Add missing features only (recommended default)
**Notes:** Visual polish deferred to Phase 13 per ROADMAP.md principles

---

## Claude's Discretion

- CSS animation for spinner fade
- Heart button touch target size
- Store key strategy for liked state

## Deferred Ideas

None
