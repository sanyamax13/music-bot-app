# Phase 4: Mini-Player Polish - Context

**Gathered:** 2026-04-16
**Status:** Ready for planning

<domain>
## Phase Boundary

Доработка persistent mini-player до уровня MINI-1..6: добавить heart-button для быстрого лайка и buffering-индикатор. Существующие жесты (swipe up/left/right), hairline progress и persistence уже реализованы в Phase 3.

</domain>

<decisions>
## Implementation Decisions

### Heart Button
- **D-01:** Heart-button размещается между блоком info (title/artist) и кнопкой play/pause — порядок: thumb → info → heart → play. Это паттерн Spotify mini-player.
- **D-02:** Используется optimistic UI: heart заполняется сразу, POST `/library/like` уходит в фоне. Rollback при ошибке — будет расширено в Phase 8 (FB-1), но базовая механика ставится здесь.
- **D-03:** Иконка сердца: `🤍` (пустое) / `❤️` (заполненное) — эмодзи, как в FullscreenPlayer (строка 1130: `🤍`). Единообразие.

### Buffering Indicator
- **D-04:** При событии `waiting` на audio element кнопка play заменяется на CSS-спиннер. При `canplay` — возвращается иконка play/pause. Не нужен отдельный DOM-элемент, подмена содержимого `#mp-play-btn`.
- **D-05:** CSS-спиннер — маленький круговой (border-based), ~20px, цвет `var(--accent)`.

### Visual Polish Scope
- **D-06:** Только добавление недостающих фич (heart, buffering). Глубокий визуальный рефакторинг стилей — Phase 13.
- **D-07:** При добавлении heart-button проверить, что mini-player не переполняется на узких экранах (320px). Если переполняется — скрыть heart на ширине < 340px через media query.

### Claude's Discretion
- CSS-анимация появления/исчезновения спиннера (fade vs instant)
- Точный размер touch-target для heart-button (≥ 44px по WCAG)
- Нужна ли подписка на новый store-ключ `isLiked` или проверять через существующий механизм

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` §5 — MINI-1..7 acceptance criteria
- `.planning/REQUIREMENTS.md` §10 — FB-1 (optimistic heart fill)

### Research
- `.planning/research/FEATURES.md` §3 — Persistent Mini-Player feature analysis
- `.planning/research/FEATURES.md` §10 — Gesture requirements table (swipe on mini-player)
- `.planning/research/PITFALLS.md` §5 — `100vh` layout jump, mini-player positioning fix
- `.planning/research/PITFALLS.md` §6 — `tg.expand()` async race condition

### Architecture
- `.planning/research/ARCHITECTURE.md` — IIFE component pattern (mount/render/destroy)

### Existing Code
- `index.html:1013-1111` — текущий MiniPlayer IIFE
- `index.html:233-269` — текущие CSS-стили mini-player
- `index.html:427-434` — HTML-разметка mini-player
- `index.html:667-900` — PlaybackEngine (getPosition, getDuration, play, pause, next, prev, toggle)
- `index.html:1113-1142` — FullscreenPlayer template (heart-button pattern `🤍`)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **MiniPlayer IIFE** (line 1013): уже реализован mount/destroy/subscriptions, swipe-жесты, rAF progress
- **PlaybackEngine** (line 667): `getPosition()`, `getDuration()`, `play()`, `pause()`, `toggle()`, `next()`, `prev()` — полный API
- **Store**: subscribe по ключам `currentTrack`, `isPlaying`, `duration` — уже подписаны
- **FullscreenPlayer heart pattern** (line 1130): `🤍` emoji + `data-action="like"` — можно повторить в mini-player
- **CSS переменные**: `--accent`, `--surface-2`, `--text-muted`, `--bg` — используются повсюду

### Established Patterns
- **IIFE component**: `mount(el)` → subscribe to Store → `destroy()` clears subscriptions + innerHTML
- **Surgical patching**: play icon обновляется через `patchPlayIcon()` без перерисовки всего template
- **Store subscription**: `Store.subscribe(['key'], callback)` возвращает unsubscribe function
- **Touch handling**: inline в `hydrate()`, не через внешний gesture module

### Integration Points
- **Store key `isLiked`**: нужно либо добавить, либо проверять через `/library/likes` — решение за planner
- **PlaybackEngine events**: `waiting`/`canplay` нужно пробросить через Store или напрямую слушать audio element
- **HTML template**: добавить heart-button в `templateTrack()` (line 1016)
- **CSS**: добавить стили для heart-button и spinner в блок `#mini-player` (line 233)

</code_context>

<specifics>
## Specific Ideas

- Heart-button в mini-player должен быть визуально таким же как в FullscreenPlayer (`🤍`/`❤️`), но меньшего размера
- Спиннер буферизации заменяет play/pause иконку, а не добавляется рядом — экономия места
- Нет close (×) кнопки на mini-player (анти-требование из FEATURES.md §3: "Spotify/Yandex/Apple Music all do not have a close button")

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 04-mini-player-polish*
*Context gathered: 2026-04-16*
