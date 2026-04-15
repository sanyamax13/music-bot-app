---
phase: 01-architecture-refactor
plan: 06
subsystem: overlay-components
tags: [refactor, components, mini-player, fullscreen-player, queue-panel, arch-3, arch-1]
requires: ["01-05"]
provides:
  - "MiniPlayer IIFE component with mount/destroy — separate subscriptions on ['currentTrack'] (full render) and ['isPlaying'] (surgical play-icon patch)"
  - "FullscreenPlayer IIFE component with mount/destroy — four separate subscriptions ['currentTrack']/['isPlaying']/['duration']/['fullscreenOpen']; full render only on track change, play-icon/duration patched surgically, rAF tickProgress with dragging flag"
  - "QueuePanel IIFE skeleton with mount/render/destroy — subscribes to ['queue','queueIndex'] for render and ['queuePanelOpen'] for visibility toggle (full bottom-sheet UI deferred to Phase 6)"
  - "3 of 12 ARCH-3 components implemented"
  - "Plan 04 temporary subscribe bridges (isPlaying/currentTrack/duration) + tickFpProgress rAF removed — components own these subscriptions now"
  - "Router remains sole writer of fullscreenOpen: MiniPlayer hydrate calls Router.openFullscreen(), FullscreenPlayer close action calls Router.closeFullscreen() — no direct Store.set on fullscreenOpen outside Router IIFE"
affects: [index.html]
tech_stack:
  added: []
  patterns:
    - "Component IIFE with mount/render/destroy and per-key subscribe (ARCHITECTURE §4)"
    - "Surgical patch pattern: rare events (currentTrack) trigger full innerHTML re-render; frequent events (isPlaying, duration, rAF tick) patch specific nodes — keeps #progress range drag state intact, avoids flicker on pause/play"
    - "Dragging flag on FullscreenPlayer scrubber: pointerdown/touchstart set dragging=true, pointerup/touchend clear it; tickProgress skips prog.value write while dragging — rAF loop doesn't fight user's finger"
    - "Overlay mutators are sole writers: components never Store.set({ fullscreenOpen }) — they call Router.openFullscreen/closeFullscreen (enforces ARCH-1 for routing keys)"
    - "Data-action delegation inside FullscreenPlayer.hydrate: single root.onclick with switch(action) instead of inline onclick per button"
key_files:
  created:
    - path: .planning/phases/01-architecture-refactor/01-06-SUMMARY.md
      what: "This summary"
  modified:
    - path: index.html
      what: "COMPONENTS section filled with MiniPlayer, FullscreenPlayer, QueuePanel IIFE. #queue-panel div added after #full-player. Plan 04 temp subscribe blocks (isPlaying/currentTrack/duration) + tickFpProgress rAF deleted. Three mount calls added at start of INIT section."
decisions:
  - "SEPARATE subscriptions per Store key (not a single combined ['currentTrack','isPlaying','duration'] subscription): currentTrack → renderTrack (full innerHTML rewrite), isPlaying → patchPlayIcon (only #mp-play-btn/#fp-play-btn .innerText), duration → patchDuration (#fp-time-total .innerText). This prevents pause/play from rewriting the whole innerHTML and losing the #progress range's focus/drag state. This is the critical fix called out in the plan review."
  - "Dragging flag in FullscreenPlayer blocks rAF tickProgress from overwriting #progress.value while user is scrubbing. Both pointer events (desktop + modern mobile) and touch events (iOS Telegram WebView safety) are wired."
  - "FullscreenPlayer.hydrate uses delegated click on root with data-action switch, not inline onclick in template. This keeps all behavior in JS (testable) and means Plan 08 markup cleanup already applies to FP content — the static markup only matters until mount replaces innerHTML."
  - "MiniPlayer and FullscreenPlayer call Router.openFullscreen() / Router.closeFullscreen() — never Store.set({ fullscreenOpen }) directly. This enforces ARCH-1 (Router is the sole writer of overlay-state keys) even though the raw Store API would allow a direct write."
  - "QueuePanel is a Phase 1 skeleton only: render produces a flat list of queue rows with is-current marker. Full bottom-sheet overlay with History / Now Playing / Up Next sections, SortableJS reorder, swipe-left remove, long-press action sheet comes in Phase 6. The skeleton exists now to (a) close 3/12 ARCH-3 components, (b) let Router.openQueuePanel toggle something visible for BackButton precedence testing, (c) establish the mount point #queue-panel."
  - "Plan 05 fullscreenOpen Store.subscribe bridge (toggles .open class on #full-player) is intentionally LEFT IN PLACE. It duplicates FullscreenPlayer.syncOpen idempotently — safe. Plan 06 scope text explicitly says to remove Plan 04 temp subs; Plan 05 bridge is Plan 08 cleanup scope."
  - "openFullPlayer / closeFullPlayer thin delegates at lines 1299-1300 are LEFT UNCHANGED — inline onclick in markup still references them. Plan 08 removes both the delegates and the inline onclicks together."
metrics:
  duration: "~5 min"
  completed: "2026-04-15"
  tasks_completed: 2
  files_modified: 1
  lines_before: 1392
  lines_after: 1601
---

# Phase 1 Plan 06: Overlay Components Summary

Три персистентных overlay-элемента (mini-player, fullscreen player, queue panel) превращены в полноценные IIFE-компоненты с mount/destroy по ARCHITECTURE §4. Критический patch: MiniPlayer и FullscreenPlayer используют раздельные подписки на ключи Store (currentTrack → полный render, isPlaying → хирургический patch иконки play, duration → patch #fp-time-total) — пауза/воспроизведение больше не перерисовывает весь innerHTML и не сбивает drag-состояние scrubber'а. FullscreenPlayer scrubber защищён флагом `dragging` от rAF-цикла.

## What Was Built

### Task 6.1 — MiniPlayer + FullscreenPlayer IIFE (commit 503bc44)

**MiniPlayer** (lines 996–1055 в index.html, внутри секции `// ===== COMPONENTS =====`):

- `templateTrack(t)` — вся разметка mini-player (обложка, тайтл, артист, play-button с data-action)
- `renderTrack()` — полная перерисовка innerHTML + hydrate; вызывается ТОЛЬКО при смене `currentTrack` (редкое событие)
- `patchPlayIcon()` — меняет `.innerText` у `#mp-play-btn` на ⏸/▶; вызывается на каждое изменение `isPlaying` и НЕ перерисовывает innerHTML
- `hydrate()` — навешивает `root.onclick` с делегированной проверкой `data-action="toggle"`: тап по кнопке → `PlaybackEngine.toggle()`, тап в любое другое место → `Router.openFullscreen()` (не `Store.set({fullscreenOpen:true})`)
- `mount(el)` — 2 подписки: `['currentTrack'] → renderTrack`, `['isPlaying'] → patchPlayIcon`
- `destroy()` — оба `unsub()` + очистка innerHTML + скрытие root

**FullscreenPlayer** (lines 1057–1181):

- `templateTrack(t)` — полная разметка FP с data-action атрибутами на всех интерактивных элементах (`close`, `like`, `toggle`, `prev`, `next`, `dislike`, `like-feedback`)
- `renderTrack()` — полный rewrite innerHTML; ТОЛЬКО на смене currentTrack
- `patchPlayIcon()` — `#fp-play-btn .innerText = ⏸/▶`; подписан на `['isPlaying']`
- `patchDuration()` — `#fp-time-total .innerText = Utils.fmtTime(...)`; подписан на `['duration']`
- `hydrate()` — один делегированный `root.onclick` со `switch(action)`; отдельно обрабатывает `#progress range` с `oninput = seekPct` и пропагандой `dragging` флага через pointerdown/pointerup + touchstart/touchend
- `tickProgress()` — rAF-цикл: останавливается если `fullscreenOpen=false`, патчит `#fp-time-cur`, патчит `#progress.value` **только** если `!dragging`
- `syncOpen()` — toggle `.open` класс + старт tickProgress при открытии
- `mount(el)` — 4 подписки: `['currentTrack'] → renderTrack`, `['isPlaying'] → patchPlayIcon`, `['duration'] → patchDuration`, `['fullscreenOpen'] → syncOpen`
- `destroy()` — cancel rAF + все 4 unsub + очистка innerHTML

**Удалены** (lines 1322-1358 из Plan 04):
- `Store.subscribe(['isPlaying'], …)` mini+fp play-icon sync block
- `Store.subscribe(['currentTrack'], …)` meta sync block
- `Store.subscribe(['duration'], …)` time-total sync block
- IIFE `(function tickFpProgress() { … })()` rAF loop

**Добавлены mount-вызовы** в начале `// ============ INIT ============`:
```js
MiniPlayer.mount(document.getElementById('mini-player'));
FullscreenPlayer.mount(document.getElementById('full-player'));
```

**openFullPlayer/closeFullPlayer** делегаты (lines 1299-1300) **НЕ тронуты** — они нужны для legacy inline onclick в markup и уйдут в Plan 08 вместе с самими inline-атрибутами.

### Task 6.2 — QueuePanel skeleton (commit b8fd7b3)

**#queue-panel div** добавлен в markup сразу после `</div>` закрывающего `#full-player`:
```html
<div id="queue-panel" style="display:none"></div>
```

**QueuePanel IIFE** (lines 1184–1236):

- `template(state)` — Phase 1 stub: если `queue.length === 0` → `<p class="empty">Очередь пуста</p>`; иначе flat-list `.qp-row` с `.is-current` на `state.queueIndex`
- `render()` — читает `queue` + `queueIndex` из Store, пишет innerHTML
- `syncOpen()` — `root.style.display = open ? 'block' : 'none'`
- `mount(el)` — 2 подписки: `['queue','queueIndex'] → render`, `['queuePanelOpen'] → syncOpen`
- `destroy()` — оба `unsub()` + очистка innerHTML

**Добавлен mount-вызов** после FullscreenPlayer.mount:
```js
QueuePanel.mount(document.getElementById('queue-panel'));
```

Полный bottom-sheet UI (History / Now Playing / Up Next секции, SortableJS reorder, swipe-left remove, long-press action sheet) реализуется в **Phase 6**. Сейчас компонент существует чтобы:
1. Закрыть 3/12 компонентов ARCH-3
2. Дать Router.openQueuePanel/closeQueuePanel видимый эффект (для ручного BackButton precedence теста)
3. Зафиксировать mount-point `#queue-panel`

## Acceptance Criteria — all pass

| Критерий | Ожидание | Результат |
|---|---|---|
| `grep -c "const MiniPlayer = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const FullscreenPlayer = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const QueuePanel = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c 'id="queue-panel"' index.html` | 1 | 1 ✓ |
| `grep -c "MiniPlayer.mount(document.getElementById('mini-player'))" index.html` | 1 | 1 ✓ |
| `grep -c "FullscreenPlayer.mount(document.getElementById('full-player'))" index.html` | 1 | 1 ✓ |
| `grep -c "QueuePanel.mount(document.getElementById('queue-panel'))" index.html` | 1 | 1 ✓ |
| `grep -c "PlaybackEngine.getPosition" index.html` | ≥ 1 | 2 ✓ |
| `grep -c "Router.openFullscreen()" index.html` | ≥ 1 | 3 ✓ |
| `grep -c "Router.closeFullscreen()" index.html` | ≥ 1 | 2 ✓ |
| `grep -c "patchPlayIcon" index.html` | ≥ 2 | 4 ✓ (2 defs + 2 subscribe call-sites) |
| `grep -cE "Store\.subscribe\(\['isPlaying'\]" index.html` | 2 | 2 ✓ |
| `grep -cE "Store\.subscribe\(\['currentTrack', ?'isPlaying'" index.html` | 0 | 0 ✓ |
| `grep -c "let dragging" index.html` | 1 | 1 ✓ |
| `grep -c "dragging = true" index.html` | ≥ 2 | 2 ✓ (pointerdown + touchstart) |
| `grep -c "!dragging" index.html` | 1 | 1 ✓ |
| `grep -cE "function tickFpProgress" index.html` | 0 | 0 ✓ (Plan 04 rAF deleted) |
| Script extract → `node --check` | SYNTAX_OK | SYNTAX_OK ✓ |
| `wc -l index.html` | — | 1601 (was 1392) |

## One-Writer Audit — `fullscreenOpen` (ARCH-1)

`grep -nE "Store\.set\(\{ ?fullscreenOpen" index.html`:

| Line | Context | Inside Router IIFE? |
|---|---|---|
| 963 | `Router.back()` precedence step 2 — `Store.set({ fullscreenOpen: false })` | ✓ Router |
| 979 | Comment explaining the one-writer invariant | n/a (comment) |
| 981 | `Router.openFullscreen()` — `Store.set({ fullscreenOpen: true })` | ✓ Router |
| 982 | `Router.closeFullscreen()` — `Store.set({ fullscreenOpen: false })` | ✓ Router |

**MiniPlayer and FullscreenPlayer contain zero direct writes** to `fullscreenOpen`. They route all mutations through `Router.openFullscreen()` / `Router.closeFullscreen()`. ARCH-1 invariant for routing/overlay keys remains intact.

## Subscription Topology — isPlaying audit

Expected: two separate `['isPlaying']` subscriptions — one in MiniPlayer (→ patchPlayIcon), one in FullscreenPlayer (→ patchPlayIcon). No combined multi-key subscription.

`grep -nE "Store\.subscribe\(\['isPlaying'\]" index.html`:
- Line ~1044 (MiniPlayer.mount) → `unsubPlaying = Store.subscribe(['isPlaying'], patchPlayIcon);`
- Line ~1151 (FullscreenPlayer.mount) → `unsubPlaying = Store.subscribe(['isPlaying'], patchPlayIcon);`

Both call **patchPlayIcon**, not renderTrack. Pause/play flips only the play-button's `.innerText`; the rest of mini-player and full-player markup (including `#progress range` element and its drag state) is untouched.

## Dragging-flag topology — scrubber drag preservation

Inside FullscreenPlayer IIFE:
- `let dragging = false;` — module-level closure state
- `range.onpointerdown = () => { dragging = true; };`
- `range.ontouchstart  = () => { dragging = true; };`
- `range.onpointerup   = () => { dragging = false; };`
- `range.ontouchend    = () => { dragging = false; };`
- Inside `tickProgress()`: `if (prog && dur && !dragging) prog.value = (pos / dur) * 100;`

The rAF loop reads position from PlaybackEngine on every frame and updates `#fp-time-cur.innerText` unconditionally, but only writes `#progress.value` when the user is NOT currently dragging. This is the critical fix that prevents the scrubber from "snapping back" to playback position while the user holds the thumb.

## Browser smoke test — pending user verification

Executor runs without a browser. The following flows must be manually verified (все уже проходят по коду, но UI проверка помечена **pending**):

1. Открыть страницу, включить трек → mini-player виден с обложкой / тайтлом / артистом
2. Тап на mini-player → fullscreen открывается с правильным треком (Router.openFullscreen path)
3. Seek-bar движется сам (rAF tickProgress)
4. **Удерживать палец на scrubber'е, двигать** → значение НЕ прыгает обратно к позиции трека (dragging flag работает)
5. **Пауза на полноэкранном плеере**: ⏸ → иконка меняется на ▶ **без перерисовки всего блока** — в DevTools → Elements видно мигание только `#fp-play-btn .innerText`, остальной DOM не перерисовывается
6. ⏭/⏮ работают (data-action=next/prev → PlaybackEngine)
7. 👎/👍 отправляют POST `/wave/feedback` (sendManualFeedback)
8. 🤍 отправляет POST `/library/like`
9. ▾ закрывает FP (data-action=close → Router.closeFullscreen)
10. Переключить таб с играющим треком → mini-player по-прежнему виден и реагирует на ⏸/⏵
11. BackButton precedence: открыть FP → BackButton показан; `Store.set({ queuePanelOpen: true })` в DevTools → #queue-panel виден, BackButton всё ещё показан; tg.BackButton тап → queue-panel закрывается (Router.back precedence step 1), FP остаётся открытым; ещё раз BackButton → FP закрывается

## Deviations from Plan

Нет. План исполнен точно как написан — код MiniPlayer / FullscreenPlayer / QueuePanel взят verbatim из `<interfaces>` секции плана (с полным набором пост-ревью фиксов: раздельные подписки, patch-функции, dragging flag, Router-делегация). Удалены именно те 3 subscribe-блока + tickFpProgress IIFE, что указаны в action-шагах. Mount-вызовы добавлены в начало INIT, чтобы гарантировать наличие DOM + всех определённых выше IIFE (Utils, Store, Api, PlaybackEngine, Router).

Plan 05 bridge `Store.subscribe(['fullscreenOpen'], …)` (toggle `.open` class) — оставлен без изменений: он дублирует `FullscreenPlayer.syncOpen` идемпотентно. Явная чистка Plan 05 моста — вне scope Plan 06.

## Known Stubs

- **QueuePanel render — Phase 1 stub.** Сейчас рендерит flat-list `.qp-row` без секций History/Now Playing/Up Next, без drag-to-reorder, без swipe-left remove, без long-press action sheet. Это **запланированный stub**: полная реализация в Phase 6 (Q-1..5 + GEST-2). Плана Phase 6 ARCHITECTURE.md §4 явно упоминает QueuePanel как отдельный компонент.
- **QueuePanel без CSS.** `#queue-panel` — просто `display: none/block`. Анимация slide-up bottom-sheet, backdrop, z-index stacking — Phase 6.
- **Plan 05 fullscreenOpen bridge** (строки 1340-1345) — не stub, а идемпотентный дубликат `FullscreenPlayer.syncOpen`. Оставлен как есть; Plan 08 markup cleanup его удалит.
- **Plan 05 screen visibility bridge** (Store.subscribe(['screen'])) — сознательный мост до Plan 07, который заменит его на BottomNav + component-mounted screens.
- **openFullPlayer / closeFullPlayer / switchTab / togglePlay / skipNext / playPrev / seekAudio / sendFeedback / toggleCollectionCurrent глобалы** — legacy делегаты для inline onclick в markup. Plan 08 удалит их вместе с inline handlers.

## Threat Flags

Нет нового surface. Все title/artist/thumb значения из API проходят через `Utils.escapeHtml` перед interpolation в template literal (T-01-09 mitigation сохранён из плана). Никаких новых endpoint'ов, persistent state, trust boundaries.

## Requirements Progress

- **ARCH-3** («12 компонентов с mount/render/destroy») — **3/12 закрыто**: MiniPlayer, FullscreenPlayer, QueuePanel. Оставшиеся 9 — в Plan 07 (WaveScreen, SearchScreen, GenresScreen, CollectionScreen, ProfileScreen, BottomNav, SearchModal, MoodScroll, TrackCard).
- **ARCH-1** («один владелец на key в Store») — **подтверждён для fullscreenOpen**: audit показывает 0 прямых Store.set({ fullscreenOpen }) вне Router IIFE. MiniPlayer/FullscreenPlayer используют Router.openFullscreen/closeFullscreen.

## Commits

| Hash | Type | Description |
|---|---|---|
| 503bc44 | feat(01-06) | add MiniPlayer and FullscreenPlayer IIFE components |
| b8fd7b3 | feat(01-06) | add QueuePanel IIFE skeleton with mount/render/destroy |

## Self-Check: PASSED

- FOUND: `/root/music-bot-frontend/.planning/phases/01-architecture-refactor/01-06-SUMMARY.md` (this file, written via Write tool)
- FOUND: commit `503bc44` (`feat(01-06): add MiniPlayer and FullscreenPlayer IIFE components`)
- FOUND: commit `b8fd7b3` (`feat(01-06): add QueuePanel IIFE skeleton with mount/render/destroy`)
- FOUND: `const MiniPlayer = (() =>` in index.html
- FOUND: `const FullscreenPlayer = (() =>` in index.html
- FOUND: `const QueuePanel = (() =>` in index.html
- FOUND: `<div id="queue-panel" style="display:none"></div>` in markup
- FOUND: `MiniPlayer.mount(document.getElementById('mini-player'))` in INIT
- FOUND: `FullscreenPlayer.mount(document.getElementById('full-player'))` in INIT
- FOUND: `QueuePanel.mount(document.getElementById('queue-panel'))` in INIT
- FOUND: `Router.openFullscreen()` inside MiniPlayer.hydrate (else branch)
- FOUND: `Router.closeFullscreen()` inside FullscreenPlayer.hydrate (case 'close')
- FOUND: `let dragging = false` inside FullscreenPlayer IIFE
- FOUND: `!dragging` guard inside tickProgress
- MISSING: `function tickFpProgress` (correctly deleted)
- MISSING: Plan 04 temporary `Store.subscribe(['isPlaying']/['currentTrack']/['duration'])` blocks (correctly deleted — only component-scoped subscriptions remain)
- Script block passes `node --check` (SYNTAX_OK)
