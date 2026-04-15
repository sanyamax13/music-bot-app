---
phase: 01-architecture-refactor
plan: 05
subsystem: router
tags: [refactor, router, navigation, back-button, overlay, arch-4, arch-1]
requires: ["01-04"]
provides:
  - "Router IIFE — state machine для экранов + sole writer ключей screen/fullscreenOpen/queuePanelOpen в Store"
  - "BackButton precedence: queuePanelOpen → fullscreenOpen → stack → tg.close()"
  - "Публичный API: go, back, updateBackButton, openFullscreen, closeFullscreen, openQueuePanel, closeQueuePanel"
  - "switchTab — тонкий делегат к Router.go(tab, { push:false })"
  - "openFullPlayer/closeFullPlayer — тонкие делегаты к Router.openFullscreen/closeFullscreen"
  - "Два временных Store.subscribe моста (screen visibility + fullscreenOpen DOM toggle) — будут заменены BottomNav/FullscreenPlayer компонентами"
  - "Initial Router.go('wave', { push:false }) вызов в конце скрипта для seed BottomNav active class"
affects: [index.html]
tech_stack:
  added: []
  patterns:
    - "Router IIFE как state-машина с внутренним stack[] — единственный writer Store.screen"
    - "Overlay mutators паттерн: Router.openFullscreen/closeFullscreen/openQueuePanel/closeQueuePanel — единственные writer'ы overlay-флагов, компоненты НЕ пишут напрямую в Store"
    - "BackButton precedence кодирован в Router.back() + Router.updateBackButton() — одна точка мутации tg.BackButton.show/hide"
    - "Thin delegate pattern для legacy inline onclick: switchTab/openFullPlayer/closeFullPlayer → Router"
    - "Temporary Store.subscribe bridges для DOM-sync до Plan 06/07"
key_files:
  created:
    - path: .planning/phases/01-architecture-refactor/01-05-SUMMARY.md
      what: "This summary"
  modified:
    - path: index.html
      what: "Секция // ===== ROUTER ===== заполнена Router IIFE. switchTab body заменён на Router.go делегат. openFullPlayer/closeFullPlayer стали делегатами к Router overlay-мутаторам. Добавлены два Store.subscribe bridge (screen/fullscreenOpen). Добавлен initial Router.go('wave', {push:false}) вызов в конце скрипта."
decisions:
  - "Router — единственный writer screen/fullscreenOpen/queuePanelOpen. Это enforce'ит ARCH-1 для overlay-флагов: даже компоненты (будущие MiniPlayer/FullscreenPlayer/QueuePanel) ОБЯЗАНЫ звать Router.openFullscreen()/Router.closeFullscreen(), а не Store.set напрямую. Это зафиксировано inline-комментарием в IIFE."
  - "switchTab оставлен как глобал (не удалён), потому что HTML markup всё ещё содержит inline onclick=\"switchTab('wave', this)\". Plan 07 (BottomNav компонент) удалит inline handlers и switchTab можно будет убрать полностью."
  - "Screen visibility sync сделан через Store.subscribe(['screen']) — дублирует логику pre-refactor switchTab (show tab-view, toggle nav-item.active, lazy load wave/collection/profile), но теперь screen-driven. Plan 07 заменит на component mount/unmount."
  - "Fullscreen overlay sync сделан отдельным Store.subscribe(['fullscreenOpen']) bridge — Plan 06 (FullscreenPlayer component) заменит его на component, реагирующий на тот же Store-ключ."
  - "Initial Router.go('wave', { push:false }) помещён в самом конце INIT секции ПОСЛЕ loadMoods/loadWave/renderGenres/setupSearchTab/setupSearchModal, чтобы все подписки были навешаны к моменту первого Store.set({ screen }). push:false — tabs на root-уровне, стек пустой → BackButton не появляется."
  - "Router.back() использует Tg.tg.close() (не window.Telegram.WebApp.close) — соблюдает Plan 01 Tg IIFE инкапсуляцию."
metrics:
  duration: "~3 min"
  completed: "2026-04-15"
  tasks_completed: 2
  files_modified: 1
  lines_before: 1325
  lines_after: 1392
---

# Phase 1 Plan 05: Router Summary

Внедрён Router IIFE как state-машина экранов и единственный writer ключей `screen`, `fullscreenOpen`, `queuePanelOpen` в Store. BackButton precedence (queuePanelOpen → fullscreenOpen → stack → tg.close) кодирован в одной точке. switchTab / openFullPlayer / closeFullPlayer превращены в тонкие делегаты к Router. Добавлены два временных Store.subscribe моста для DOM-синхронизации до появления компонентов в Plan 06/07.

## What Was Built

### Task 5.1 — Router IIFE (commit ce2f549)

В секцию `// ===== ROUTER =====` вставлен verbatim код Router IIFE из `<interfaces>` плана.

**Приватное состояние:**
- `const stack = []` — стек предыдущих screen'ов для `back()`

**Публичный API (7 методов):**

| Метод | Назначение |
|---|---|
| `go(screen, opts)` | Переключает screen; если `opts.push !== false` и prev !== screen, пушит prev в stack; обновляет BackButton |
| `back()` | Precedence: closes queuePanel → closes fullscreen → pops stack → tg.close() |
| `updateBackButton()` | Показывает tg.BackButton если `fullscreenOpen || queuePanelOpen || stack.length`, иначе скрывает |
| `openFullscreen()` | `Store.set({ fullscreenOpen: true })` |
| `closeFullscreen()` | `Store.set({ fullscreenOpen: false })` |
| `openQueuePanel()` | `Store.set({ queuePanelOpen: true })` |
| `closeQueuePanel()` | `Store.set({ queuePanelOpen: false })` |

**Wiring (выполняется один раз при конструировании IIFE):**
- `Tg.tg.BackButton.onClick(back)` — BackButton глобально привязан к `Router.back`
- `Store.subscribe(['fullscreenOpen', 'queuePanelOpen'], updateBackButton)` — любое изменение overlay-флагов (например, через закрытие FP из кода или ▾ handle) автоматически обновляет видимость BackButton

### Task 5.2 — switchTab/openFullPlayer/closeFullPlayer delegates + bridges (commit 174e791)

**switchTab** упрощён до одной строки:
```js
function switchTab(tab, el) {
    Router.go(tab, { push: false });
}
```

**openFullPlayer/closeFullPlayer** превращены в тонкие делегаты:
```js
function openFullPlayer()  { Router.openFullscreen();  }
function closeFullPlayer() { Router.closeFullscreen(); }
```

Оба остаются как глобалы до Plan 07, потому что HTML markup содержит inline onclick'и (`<div id="mini-player" onclick="openFullPlayer()">`, `<div class="fp-handle" onclick="closeFullPlayer()">▾</div>`, `<div class="nav-item" ... onclick="switchTab('wave', this)">`).

**Два Store.subscribe bridge'а** добавлены перед `// ============ INIT ============`:

1. **Screen visibility sync** (`Store.subscribe(['screen'], ...)`):
   - Снимает `.active` со всех `.tab-view`, добавляет на `#tab-{screen}`
   - Снимает `.active` со всех `.nav-item`, ставит на те где `dataset.tab === screen`
   - Lazy-load триггеры сохранены: wave (loadMoods + loadWave(true) если wave-list пуст), collection (loadCollection), profile (loadProfile)

2. **Fullscreen overlay sync** (`Store.subscribe(['fullscreenOpen'], ...)`):
   - Тогглит `.open` класс на `#full-player` на основе `state.fullscreenOpen`

**Initial bootstrap call** добавлен в самом конце `// ============ INIT ============`:
```js
Router.go('wave', { push: false });
```
Это вызывает screen-subscribe один раз и навешивает active класс на Wave nav-item при первой загрузке. push:false — tabs это root-уровень, стек пустой, BackButton не появляется.

## Acceptance Criteria — все пройдены

| Критерий | Результат |
|---|---|
| `grep -c "const Router = (() =>" index.html` → 1 | 1 ✓ |
| `grep -c "Tg.tg.BackButton.onClick(back)" index.html` → 1 | 1 ✓ |
| `grep -cE "Store\.subscribe\(\['fullscreenOpen', ?'queuePanelOpen'\]" index.html` → 1 | 1 ✓ |
| `grep -c "function openFullscreen()" index.html` → 1 | 1 ✓ |
| `grep -c "function closeFullscreen()" index.html` → 1 | 1 ✓ |
| `grep -c "function openQueuePanel()" index.html` → 1 | 1 ✓ |
| `grep -c "function closeQueuePanel()" index.html` → 1 | 1 ✓ |
| `grep -c "Router.go(tab" index.html` → 1 | 1 ✓ |
| `grep -cE "function openFullPlayer\(\) +\{ +Router\.openFullscreen" index.html` → 1 | 1 ✓ |
| `grep -cE "function closeFullPlayer\(\) +\{ +Router\.closeFullscreen" index.html` → 1 | 1 ✓ |
| `grep -cE "Store\.subscribe\(\['screen'\]" index.html` → 1 | 1 ✓ |
| `grep -cE "Store\.subscribe\(\['fullscreenOpen'\]" index.html` → 1 | 1 ✓ |
| Script extract → `node --check` | SYNTAX_OK ✓ |
| `wc -l index.html` | 1392 (было 1325) |

## One-Writer Audit — screen / fullscreenOpen / queuePanelOpen

`grep -nE "Store\.set\(\{ ?(screen\|fullscreenOpen\|queuePanelOpen)" index.html`:

| Строка | Контекст | Внутри Router IIFE? |
|---|---|---|
| 951 | `Router.go()` — `Store.set({ screen })` | ✓ Router |
| 963 | `Router.back()` — `Store.set({ screen: stack.pop() })` | ✓ Router |
| 962 | `Router.back()` — `Store.set({ fullscreenOpen: false })` (precedence step 2) | ✓ Router |
| 980 | `Router.openFullscreen()` — `Store.set({ fullscreenOpen: true })` | ✓ Router |
| 981 | `Router.closeFullscreen()` — `Store.set({ fullscreenOpen: false })` | ✓ Router |
| 961 | `Router.back()` — `Store.set({ queuePanelOpen: false })` (precedence step 1) | ✓ Router |
| 982 | `Router.openQueuePanel()` — `Store.set({ queuePanelOpen: true })` | ✓ Router |
| 983 | `Router.closeQueuePanel()` — `Store.set({ queuePanelOpen: false })` | ✓ Router |

**Вывод:** все writer'ы screen/fullscreenOpen/queuePanelOpen — внутри `const Router = (() => { ... })();`. ARCH-1 (one-writer-per-key) для routing/overlay блока закрыт. ARCH-4 закрыт.

## BackButton Precedence — ручная верификация (pending browser)

Ожидаемый flow (из acceptance criteria):

1. Загрузка → Wave → BackButton СКРЫТ (stack=[], нет overlay'ев)
2. Tap Поиск → Router.go('search', {push:false}) → stack по-прежнему [] → BackButton СКРЫТ (tabs не пушатся в стек)
3. Tap Wave → играть трек → tap mini-player → openFullPlayer → Router.openFullscreen → Store.set({fullscreenOpen:true}) → subscribe trigger → FP DOM открывается, updateBackButton вызывается (через подписку на overlay flags) → BackButton ПОКАЗАН
4. Tap tg.BackButton → Router.back() → queuePanelOpen=false пропускаем, fullscreenOpen=true → Store.set({fullscreenOpen:false}) → FP закрывается, updateBackButton → BackButton СКРЫТ
5. Повторить open FP → переключить таб с открытым FP → fullscreenOpen не меняется (Router.go не трогает overlay) → FP остаётся открытым → play продолжается (PlaybackEngine не знает про Router)
6. Закрыть FP через ▾ handle (`onclick="closeFullPlayer()"`) → Router.closeFullscreen → Store.set({fullscreenOpen:false}) → DOM sync + BackButton hide

Все шаги проходят по коду. Browser smoke test помечается **pending user verification** — executor работает без браузера.

## Deviations from Plan

Нет. План исполнен точно как написан (verbatim Router IIFE, два subscribe-моста как указано, initial Router.go в конце INIT).

## Known Stubs

- **Temporary Store.subscribe bridges (screen visibility + fullscreenOpen DOM toggle)** — не stub, а сознательные bridges. Plan 06 заменит fullscreenOpen bridge на FullscreenPlayer компонент (с mount/render/destroy). Plan 07 заменит screen bridge на BottomNav + component-mounted screens.
- **switchTab / openFullPlayer / closeFullPlayer глобалы** — живут как тонкие делегаты до Plan 07 (markup cleanup удалит inline onclick'и).

## Threat Flags

Нет нового surface. Router инкапсулирует уже существовавшие tg.BackButton вызовы и DOM-мутации без добавления endpoint'ов, trust boundaries или persistent state. Threat model плана зафиксировал T-01-08 (BackButton race) как accept — подтверждено: подписка идемпотентна, tg.BackButton.show/hide сами по себе идемпотентны.

## Requirements Progress

- **ARCH-4** («Router IIFE как state-машина экранов + BackButton precedence») — **закрыт** ✓
- **ARCH-1** («один владелец на key в Store») — **укреплён для routing блока**: screen, fullscreenOpen, queuePanelOpen теперь имеют ровно одного writer (Router).

## Commits

| Hash | Type | Description |
|---|---|---|
| ce2f549 | feat(01-05) | add Router IIFE state machine with BackButton precedence |
| 174e791 | refactor(01-05) | migrate switchTab and full-player toggles to Router |

## Self-Check: PASSED

- FOUND: `.planning/phases/01-architecture-refactor/01-05-SUMMARY.md` (this file)
- FOUND: commit ce2f549 (`feat(01-05): add Router IIFE state machine with BackButton precedence`)
- FOUND: commit 174e791 (`refactor(01-05): migrate switchTab and full-player toggles to Router`)
- FOUND: `const Router = (() =>` in index.html
- FOUND: `Router.go(tab, { push: false })` inside switchTab
- FOUND: `Router.openFullscreen()` inside openFullPlayer, `Router.closeFullscreen()` inside closeFullPlayer
- FOUND: `Store.subscribe(['screen'], ...)` bridge
- FOUND: `Store.subscribe(['fullscreenOpen'], ...)` bridge
- FOUND: initial `Router.go('wave', { push: false })` call
- Script block passes `node --check` (SYNTAX_OK)
