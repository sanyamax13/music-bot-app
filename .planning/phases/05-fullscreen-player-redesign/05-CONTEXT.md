# Phase 5: Fullscreen Player Redesign - Context

**Gathered:** 2026-04-16
**Status:** Ready for planning

<domain>
## Phase Boundary

Доработка fullscreen-плеера до уровня Spotify/Яндекс: увеличить обложку, добавить double-tap like, queue-peek, tap по артисту → поиск. Dominant-color фон, heart-button, swipe-down/left/right, скраббер — уже реализованы.

</domain>

<decisions>
## Implementation Decisions

### Обложка (FP-1)
- **D-01:** Обложка 70vw без max-width ограничения (текущие 60vw/280px → 70vw). На широких экранах (планшет/десктоп) ограничить через `max-width: 340px` чтобы не растягивалась до абсурда.
- **D-02:** `aspect-ratio: 1/1` сохраняется. border-radius 24px сохраняется.

### Double-tap like (FP-7)
- **D-03:** Double-tap на обложке = like. Анимация: emoji ❤️ появляется поверх обложки (scale 0 → 1.2 → 1 → fade out за 600мс). Паттерн Instagram/Spotify.
- **D-04:** Реализация: отслеживать два тапа в пределах 300мс на `#fp-cover`. Использовать setTimeout для различения single-tap (нет действия на обложке) от double-tap.
- **D-05:** Haptic feedback (`tg.HapticFeedback.impactOccurred('light')`) на double-tap если доступен.

### Queue-peek (FP-8)
- **D-06:** Одна строка под feedback-кнопками: "Далее: {title}" — показывает `queue[queueIndex + 1].title`. Если следующего трека нет — скрыть.
- **D-07:** Стиль: font-size 12px, color `var(--text-muted)`, text-overflow ellipsis, max-width 80%.
- **D-08:** Подписка на `['queue', 'queueIndex']` через Store для обновления.

### Tap по артисту (FP-9)
- **D-09:** Tap по `#fp-artist` → переключить на вкладку Search с prefilled query = artist name. Очистить предыдущие результаты, запустить поиск автоматически.
- **D-10:** Закрыть fullscreen-player перед переключением на Search.

### Скраббер (FP-3)
- **D-11:** Touch-target скраббера ≥ 44px (текущий `height: 20px` для input range). Увеличить padding вокруг thumb. Визуальный thumb 14px сохраняется, но tap-область расширяется.

### Claude's Discretion
- Точная CSS-анимация для double-tap heart overlay (keyframes, z-index)
- Нужна ли debounce для double-tap vs single-tap или достаточно setTimeout
- Структура DOM для queue-peek элемента (div внутри fp-controls или отдельный блок)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` §4 — FP-1..9 acceptance criteria
- `.planning/REQUIREMENTS.md` §10 — GEST-1 (gesture module), FB-3 (haptic feedback)

### Research
- `.planning/research/FEATURES.md` §2 — Fullscreen Player feature analysis
- `.planning/research/FEATURES.md` §10 — Gesture requirements (double-tap, swipe on cover)
- `.planning/research/PITFALLS.md` §5 — 100vh layout jump, positioning
- `.planning/research/PITFALLS.md` §3 — Canvas CORS (dominant-color fallback)

### Prior Phases
- `.planning/phases/04-mini-player-polish/04-CONTEXT.md` — likedIds pattern, optimistic like, emoji hearts

### Existing Code
- `index.html:1178-1420` — текущий FullscreenPlayer IIFE
- `index.html:299-393` — текущие CSS-стили full-player
- `index.html:1220-1263` — dominant-color extraction (уже работает)
- `index.html:1330-1356` — swipe жесты (уже работают)
- `index.html:1284-1327` — hydrate() onclick handler (like, toggle, prev, next, dislike)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **FullscreenPlayer IIFE** (line 1178): полный mount/destroy/subscriptions, dominant-color, swipe-жесты
- **PlaybackEngine**: getPosition(), getDuration(), next(), prev(), toggle() — полный API
- **Store**: likedIds (Set), queue, queueIndex, currentTrack, isPlaying, duration
- **Api.like()**: уже используется в MiniPlayer и FullscreenPlayer
- **Router**: openFullscreen(), closeFullscreen(), navigate(screen) — переключение экранов
- **hashColor()** (line 1239): fallback-палитра для CORS-blocked обложек

### Established Patterns
- **IIFE component**: mount(el) → subscribe → destroy() clears subscriptions
- **Surgical patching**: patchPlayIcon(), patchDuration() — без перерисовки innerHTML
- **Touch handling**: inline в hydrate(), touchstart/touchend с direction detection
- **Optimistic like**: emoji swap → Api.like() → .catch() rollback

### Integration Points
- **Store key `queue`**: нужен для queue-peek (read `queue[queueIndex + 1]`)
- **Router.navigate('search')**: для перехода на Search при tap по артисту
- **SearchScreen**: нужен метод или прямой доступ для prefill query и запуска поиска
- **Double-tap**: overlay элемент поверх обложки, абсолютное позиционирование

</code_context>

<specifics>
## Specific Ideas

- Double-tap heart overlay — как в Instagram: большое сердце ❤️ появляется по центру обложки, scale-анимация, затем fade out
- Queue-peek обновляется при смене трека и при reorder очереди
- Закрытие FP перед переходом на поиск артиста — чтобы не оставлять FP открытым поверх Search

</specifics>

<deferred>
## Deferred Ideas

- Todo "Spotify поиск + метаданные + YouTube стрим" (score 0.6) — это про источники данных, не про UI плеера. Остаётся в backlog.

</deferred>

---

*Phase: 05-fullscreen-player-redesign*
*Context gathered: 2026-04-16*
