---
phase: 01-architecture-refactor
plan: 02
subsystem: architecture
tags: [refactor, store, pubsub, state, shims]
requires: ["01-01"]
provides:
  - "Store IIFE с state blob (ARCHITECTURE §2) — get/set/subscribe/_state"
  - "Key-scoped pub/sub (subs: Map<key, Set<fn>>) с дедупликацией вызовов подписчиков при мульти-ключевых патчах"
  - "Legacy window.* property shims для currentQueue/currentIndex/currentCtx, waveMood/waveLoading/waveEnded, activeSource/activeModalSource — bridge старых call-sites на Store без их переписывания"
affects: [index.html]
tech_stack:
  added: []
  patterns:
    - "Revealing IIFE + blob-state"
    - "Object.defineProperty(window, ...) getter/setter shim (reassignment-only, не через in-place mutation)"
    - "Key-scoped pub/sub со snapshot-дедупликацией подписчиков"
key_files:
  created:
    - path: .planning/phases/01-architecture-refactor/01-02-SUMMARY.md
      what: "This summary"
  modified:
    - path: index.html
      what: "Заполнена секция // ===== STORE ===== — Store IIFE + 8 legacy window.* shims. Удалены топ-уровневые let'ы для 8 ключей состояния (currentQueue/Index/Ctx, waveMood/Loading/Ended, activeSource/ModalSource)."
decisions:
  - "Legacy shimming через Object.defineProperty(window, ...) — держит diff атомарным (не трогает ~30 call-sites `currentQueue = tracks`, `currentIndex++`, `waveMood`, `currentQueue.length`). Альтернатива — массовая замена всех read/write на Store.get/set — отвергнута: раздувает этот plan, размывает границы между Plan 02 (инфраструктура) и Plans 04–07 (реальная миграция владения)."
  - "Сознательно НЕ мигрированы: activeAudio/nextAudio (dual-buffer не state, а механика), currentTrackStart/lastFeedbackSent (feedback flow internals). Всё это переедет в PlaybackEngine IIFE в Plan 04 — не в Store, а внутрь модуля."
  - "window.searchTabQueue/searchModalQueue/genreQueue/collectionQueue оставлены как есть — рефакторим вместе с Api в Plan 03."
  - "В-place мутации (.push/.splice/arr[i]=) обходят setter → не нотифаят подписчиков. Текущий pre-refactor код использует только reassignment для всех мигрированных ключей, поэтому проблемы нет. Зафиксировано комментарием в коде, чтобы Plans 04–07 не ввели in-place mutation случайно."
metrics:
  duration: "~6 min"
  completed: "2026-04-15"
  tasks_completed: 2
  files_modified: 1
  lines_before: 1071
  lines_after: 1197
---

# Phase 1 Plan 02: Store IIFE Summary

Создан `Store` IIFE со state blob и key-scoped pub/sub по спецификации ARCHITECTURE §2; 8 мутабельных state-переменных перенесены в `state` blob через Object.defineProperty(window, ...) shims без изменения call-sites.

## What Was Built

### Task 2.1 — Store IIFE (commit d5ad2ce, вместе с шимами)

В секцию `// ===== STORE =====` (после Plan 01 — placeholder "filled in Plan 02") вставлен верcим код из `<interfaces>` плана:

- `state` blob с 23 ключами, сгруппированными по владельцу:
  - **Playback** (owner: PlaybackEngine — Plan 04): `currentTrack`, `isPlaying`, `duration`, `playbackCtx`
  - **Queue** (owner: PlaybackEngine — Plan 04): `queue`, `queueIndex`, `userQueue`, `history`
  - **Wave** (owner: HomeScreen — Plan 07): `waveMood`, `waveLoading`, `waveEnded`
  - **UI** (owner: Router — Plan 05): `screen`, `fullscreenOpen`, `queuePanelOpen`
  - **Search** (owner: SearchScreen — Plan 07): `activeSource`, `activeModalSource`, `searchTabResults`, `searchModalResults`
  - **Genre** (owner: HomeScreen/GenreScreen — Plan 07): `genreResults`
  - **Collection** (owner: LibraryScreen — Plan 07): `collectionResults`
  - **Per-user** (owner: Boot/Api — Plan 08): `userId`, `likedIds` (Set), `taste`, `stats`
- `subs: Map<string, Set<fn>>` — key-scoped pub/sub.
- `get(key)` — прямое чтение.
- `set(patch)` — diff по идентичности (`state[k] !== patch[k]`), запоминает `changedKeys`, вызывает `notify`.
- `subscribe(keys, fn)` — регистрирует `fn` на каждый key из `keys`, возвращает unsubscribe.
- `notify(changedKeys)` — дедупликация подписчиков через `called: Set<fn>`: если один callback подписан на несколько изменённых в одном `set` ключей, он вызывается ровно один раз (ключевая семантика для будущего Router — Plan 05).
- Экспорт: `{ get, set, subscribe, _state }`. `_state` — только для отладки.

### Task 2.2 — Legacy shims + удаление top-level let'ов (шимы в d5ad2ce, let'ы удалены в a8fb76c)

**Шимы** (вставлены сразу под Store IIFE) — 8 штук, `Object.defineProperty(window, key, { get, set, configurable: true })`:

| window.* | → Store key |
|---|---|
| `currentQueue` | `queue` |
| `currentIndex` | `queueIndex` |
| `currentCtx` | `playbackCtx` |
| `waveMood` | `waveMood` |
| `waveLoading` | `waveLoading` |
| `waveEnded` | `waveEnded` |
| `activeSource` | `activeSource` |
| `activeModalSource` | `activeModalSource` |

**Удалено** со старых позиций (строки 607–613 до рефактора):

```js
let currentQueue = [], currentIndex = -1, currentCtx = 'wave';
let waveMood = '', waveLoading = false, waveEnded = false;
let activeSource = 'all';
let activeModalSource = 'all';
```

**Оставлено** (переедет в PlaybackEngine IIFE в Plan 04, НЕ в Store):

```js
let activeAudio = audioA, nextAudio = audioB;       // dual-buffer mechanics
let currentTrackStart = 0, lastFeedbackSent = false; // feedback flow internals
```

## Почему shimming через window

`let currentQueue` в скрипте — это _не_ свойство `window` (у `let`-переменных на глобальном скрипте нет биндинга к `window`). Но все call-sites (`currentQueue = tracks`, `currentQueue.length`, `currentIndex++`) обращаются к идентификатору без префикса, и JavaScript резолвит его по scope chain: сначала локальный scope, потом глобальный, а глобальный scope в браузере смотрит на `window`. После удаления `let currentQueue` верхнеуровневый идентификатор перестаёт существовать как локальный — и запрос падает на `window.currentQueue`, который мы определили через `Object.defineProperty` как getter/setter, маршрутизирующий в Store.

Результат: старые строки вроде `currentQueue = tracks` → читаются как `window.currentQueue = tracks` → вызывают setter → `Store.set({ queue: tracks })` → notify подписчикам. Аналогично `currentQueue.length` → getter → `Store.get('queue').length`.

**Ограничение:** срабатывает только reassignment. `currentQueue.push(x)` — это сначала чтение (getter → возвращает массив из `_state`), а потом in-place `.push` на возвращённый объект. Setter не срабатывает, подписчики Store не уведомляются. Pre-refactor код использует reassignment везде — проверил grep'ом `(currentQueue|currentIndex|waveMood|activeSource|activeModalSource)\.(push|splice|shift|unshift|pop)` — совпадений ноль. Комментарий в коде предупреждает будущие планы не вводить in-place mutation до миграции в Plan 04.

## Acceptance Criteria — все пройдены

| Критерий | Результат |
|---|---|
| `grep -c "const Store = (() =>" index.html` | 1 ✓ |
| `grep -c "function subscribe(keys, fn)" index.html` | 1 ✓ |
| `grep -c "owner: PlaybackEngine" index.html` ≥ 1 | 2 ✓ |
| `grep -c "owner: Router" index.html` ≥ 1 | 1 ✓ |
| `grep -c "owner: HomeScreen" index.html` ≥ 1 | 2 ✓ |
| `grep -cE "^\s*let currentQueue" index.html` | 0 ✓ |
| `grep -cE "^\s*let waveMood" index.html` | 0 ✓ |
| `grep -cE "^\s*let activeSource" index.html` | 0 ✓ |
| `grep -c "Object.defineProperty(window, 'currentQueue'" index.html` | 1 ✓ |
| `grep -c "Object.defineProperty(window, 'waveMood'" index.html` | 1 ✓ |
| `grep -c "let activeAudio" index.html` | 1 ✓ (остаётся до Plan 04) |
| `grep -c "let currentTrackStart" index.html` | 1 ✓ (остаётся до Plan 04) |
| `wc -l index.html` ≤ 2500 | 1197 ✓ |
| `node --check` на extract'е script блока | SYNTAX OK ✓ |

## Syntax & Safety Checks

- **Script extract → `node --check`:** PASS на обеих точках (после шимов и после удаления let'ов). Никаких TDZ/redeclaration конфликтов.
- **Identifier resolution:** после удаления `let currentQueue` в скрипте, все ссылки на `currentQueue` в коде ниже резолвятся в `window.currentQueue` через scope chain — это штатное поведение браузера для неквалифицированных идентификаторов, падающих до глобального scope. Проверено ментально на всех ~15 call-sites (`loadWave`, `playFromContext`, `playTrack`, `autoExtendWave`, `doTabSearch`, `doModalSearch`, `loadGenre`, `loadCollection`, `selectMood`).
- **pub/sub deduplication:** в `notify`, `called: Set<fn>` гарантирует что `fn` с множественной подпиской (например, Router в Plan 05 подписан сразу на `screen` + `fullscreenOpen`) вызывается один раз при патче, затрагивающем оба ключа — это необходимо для будущих планов.

## Deviations from Plan

Нет. План исполнен точно как написан, код Store IIFE — verbatim из `<interfaces>`. Единственный мелкий нюанс:

- **Структура коммитов:** план не указывает границу между коммитами внутри Task 2.2, поэтому я положил добавление шимов в коммит Task 2.1 (d5ad2ce), а удаление `let`-деклараций — в отдельный коммит Task 2.2 (a8fb76c). Логически это корректно: шимы должны существовать до того, как исчезнут `let`'ы, иначе между двумя коммитами была бы краткая стадия, где call-sites ломаются. Атомарность операций (добавление → удаление) таким образом сохраняется.

## Smoke Test — deferred to Plan 08

Физический smoke-test (Telegram Mini App → все 5 табов) невозможен в этой среде. Статические гарантии:

- `node --check` extract'а проходит после обоих коммитов.
- Все identifier-references целы: grep на `currentQueue|currentIndex|currentCtx|waveMood|waveLoading|waveEnded|activeSource|activeModalSource` показывает consistent usage (read/write/length/и т.д. не изменялись).
- Shim semantic verified в голове: `Store.set` идентично reassignment семантике старых `let`'ов в каждом site.

Полный manual smoke-test — gate для Plan 08 cleanup.

## Known Stubs

Нет. Legacy shims — это осознанно временный compatibility layer, документированный в коде комментарием `LEGACY shims for Plans 02→04 — these will be removed in Plan 04`.

## Requirements Progress

- **ARCH-1** (один Store с pub/sub и одним владельцем на ключ) — **половина закрыта**: Store существует, subscribe API есть, blob документирован с owner-комментариями. Полное «один владелец на ключ» закрывается в Plans 04–07, когда PlaybackEngine/Router/Screens станут _единственными_ writer'ами соответствующих ключей (сейчас writer всё ещё legacy-код через shims).

## Commits

| Hash | Type | Description |
|---|---|---|
| d5ad2ce | feat(01-02) | add Store IIFE with state blob and pub/sub (+ window shims) |
| a8fb76c | refactor(01-02) | remove legacy let declarations now owned by Store |

## Self-Check: PASSED

- FOUND: `.planning/phases/01-architecture-refactor/01-02-SUMMARY.md`
- FOUND: commit d5ad2ce (`feat(01-02): add Store IIFE with state blob and pub/sub`)
- FOUND: commit a8fb76c (`refactor(01-02): remove legacy let declarations now owned by Store`)
- FOUND: `const Store = (() =>` in index.html
- FOUND: `function subscribe(keys, fn)` in index.html
- FOUND: 8 `Object.defineProperty(window, '...'` shim lines in index.html
- MISSING: `^\s*let currentQueue` (correct — removed)
- MISSING: `^\s*let waveMood` (correct — removed)
- MISSING: `^\s*let activeSource` (correct — removed)
- Script block passes `node --check`
