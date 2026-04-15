---
phase: 01-architecture-refactor
plan: 03
subsystem: api
tags: [refactor, api, dedup, error-envelope, fetch-migration]
requires: ["01-02"]
provides:
  - "Api IIFE — единственный мост к https://api.vdsmusic.ru:8443 (ARCHITECTURE §6)"
  - "11 методов (wave, waveMoods, search, genre, likes, stats, taste, tasteReset, like, dislike, feedback) + streamUrl(t) builder"
  - "Inflight-дедуп одинаковых запросов через Map<key, Promise>"
  - "Error envelope: req() бросает Error(`HTTP {status} on {method} {path}`) при non-2xx"
affects: [index.html]
tech_stack:
  added: []
  patterns:
    - "Revealing IIFE с приватными base/inflight/req/key"
    - "In-flight deduplication через Map<string, Promise> с auto-cleanup через .finally"
    - "URL builder (streamUrl) как отдельная ветка API — не fetch, а сборка <audio>.src"
key_files:
  created:
    - path: .planning/phases/01-architecture-refactor/01-03-SUMMARY.md
      what: "This summary"
  modified:
    - path: index.html
      what: "Заполнена секция // ===== API ===== — Api IIFE. Все 12 fetch-сайтов переведены на Api.*. window.{searchTabQueue,searchModalQueue,genreQueue,collectionQueue} заменены на Store.set/get с ключами *Results. Удалён const API = Config.API из TEMP shims."
decisions:
  - "Api IIFE использует Config.API и Tg.USER_ID напрямую (а не legacy API/USER_ID шимы) — Api сразу пишется в чистом стиле, чтобы последующие планы не меняли его тело при чистке shim-блока."
  - "Retry/backoff сознательно НЕ добавлен в Plan 03 (это AUDIO-6 в Phase 2). Dedup через inflight уже снижает дубли при быстрых повторных кликах."
  - "Feedback queue НЕ добавлена — это минимальная заглушка в Plan 04 (полный модуль — Phase 2). sendFeedback просто зовёт Api.feedback(...).catch(() => {}) и полагается на ошибки в console."
  - "streamUrl оставлен в объекте Api как метод, а не вынесен в отдельный модуль — это URL builder семантически связан с backend-схемой /stream/:id, и держать его рядом с остальной картой эндпоинтов облегчает единовременные изменения при смене API."
  - "window.*Queue → Store.{searchTabResults,searchModalResults,genreResults,collectionResults}Results. Writer (Store.set) в doTabSearch/doModalSearch/loadGenre/loadCollection, readers (Store.get) в playFromContext. В рендер-секции после fetch локальная переменная tracks не пишется в window — только в Store."
metrics:
  duration: "~5 min"
  completed: "2026-04-15"
  tasks_completed: 2
  files_modified: 1
  lines_before: 1197
  lines_after: 1230
---

# Phase 1 Plan 03: Api IIFE Summary

Создан `Api` IIFE с 11 backend-методами, inflight-дедупом и error envelope; все 12 прежних `fetch(...)` сайтов в index.html переведены на `Api.*`; ad-hoc `window.*Queue` глобалы заменены на `Store.{...}Results` ключи.

## What Was Built

### Task 3.1 — Api IIFE (commit d5a55b8)

В секцию `// ===== API =====` (после Plan 02 — placeholder "filled in Plan 03") вставлен verbatim код из `<interfaces>` плана:

- **Приватное состояние**: `base = Config.API`, `inflight = new Map()`.
- **`key(method, path, body)`**: стабильный ключ для дедупа (JSON-сериализация body).
- **`req(path, opts)`**: метод-оркестр:
  1. Вычисляет ключ;
  2. Если в `inflight` уже есть Promise с этим ключом — возвращает его (dedup);
  3. Иначе запускает `fetch(base + path, ...)`, парсит JSON, бросает Error при `!r.ok`;
  4. Регистрирует Promise в `inflight`, по `finally` удаляет.
- **`streamUrl(t)`**: строит URL для `<audio>.src`. НЕ делает fetch — это чистый URL builder для `${base}/stream/${id}?source=&url=&title=&artist=`.
- **Публичный API** (11 методов + streamUrl):

| Метод | Эндпоинт | HTTP | Body |
|---|---|---|---|
| `wave(mood)` | `/wave?user_id=&mood=` | GET | — |
| `waveMoods()` | `/wave/moods?user_id=` | GET | — |
| `search(q, src)` | `/search?q=&sources=&user_id=` | GET | — |
| `genre(name)` | `/genre?q=&user_id=` | GET | — |
| `likes()` | `/library/likes?user_id=` | GET | — |
| `stats()` | `/library/stats?user_id=` | GET | — |
| `taste()` | `/taste/:user_id` | GET | — |
| `tasteReset()` | `/taste/reset?user_id=` | POST | — |
| `like(t)` | `/library/like` | POST | `{user_id, track_id, track}` |
| `dislike(t)` | `/library/dislike` | POST | `{user_id, track_id}` |
| `feedback(ev)` | `/wave/feedback` | POST | raw event |
| `streamUrl(t)` | `/stream/:id?source=&url=&title=&artist=` | — | (URL builder) |

Использует `Config.API` и `Tg.USER_ID` напрямую, не через legacy shims.

### Task 3.2 — Миграция fetch-сайтов + удаление window.*Queue (commit ee2ef57)

**12 fetch-сайтов переведены:**

| Ранее | Теперь | Функция |
|---|---|---|
| `fetch(${API}/wave/moods?...)` | `Api.waveMoods()` | `loadMoods` |
| `fetch(${API}/wave?...)` | `Api.wave(waveMood)` | `loadWave` |
| `fetch(${API}/search?...)` (tab) | `Api.search(q, activeSource)` | `doTabSearch` |
| `fetch(${API}/search?...)` (modal) | `Api.search(q, activeModalSource)` | `doModalSearch` |
| `fetch(${API}/genre?...)` | `Api.genre(name)` | `loadGenre` |
| `fetch(${API}/library/likes?...)` | `Api.likes()` | `loadCollection` |
| `fetch(${API}/taste/${USER_ID})` | `Api.taste()` | `loadProfile` |
| `fetch(${API}/library/stats?...)` | `Api.stats()` | `loadProfile` |
| `fetch(${API}/taste/reset?...)` | `Api.tasteReset()` | `resetTaste` |
| `${API}/stream/...` (inline) | `Api.streamUrl(t)` | `playTrack` |
| `${API}/stream/...` (inline) | `Api.streamUrl(next)` | `preloadNext` |
| `fetch(${API}/wave/feedback, ...)` | `Api.feedback({...})` | `sendFeedback` |
| `fetch(${API}/library/like, ...)` | `Api.like(t)` | `likeTrack` |
| `fetch(${API}/library/dislike, ...)` | `Api.dislike(t)` | `dislikeTrack` |

(14 замен — включая два вызова внутри `Promise.all` в `loadProfile` и два URL builder-сайта в плеере.)

**Ad-hoc window.* глобалы убраны:**

| Ранее (writer/reader) | Теперь |
|---|---|
| `window.searchTabQueue = d.tracks` | `Store.set({ searchTabResults: tracks })` |
| `window.searchModalQueue = d.tracks` | `Store.set({ searchModalResults: tracks })` |
| `window.genreQueue = d.tracks` | `Store.set({ genreResults: tracks })` |
| `window.collectionQueue = d.tracks` | `Store.set({ collectionResults: tracks })` |
| `currentQueue = window.searchTabQueue` | `currentQueue = Store.get('searchTabResults')` |
| `currentQueue = window.searchModalQueue` | `currentQueue = Store.get('searchModalResults')` |
| `currentQueue = window.genreQueue` | `currentQueue = Store.get('genreResults')` |
| `currentQueue = window.collectionQueue` | `currentQueue = Store.get('collectionResults')` |

Renders теперь используют локальную `const tracks = d.tracks || [];` вместо `window.*.length`, `window.*.map` — чище и независимее от глобала.

**Shim cleanup:**

- **Удалено** из TEMP shim-блока: `const API = Config.API;` — больше никакой код не ссылается на `API` как идентификатор.
- **Оставлено** (миграция в Plans 04/07/08): `SOURCES`, `SOURCE_ICONS`, `GENRES`, `tg`, `USER_ID`, `fmtTime`, `escapeHtml`, `escapeAttr`.

**Feedback упрощена:**

```js
function sendFeedback(action) {
  if (lastFeedbackSent || currentIndex < 0) return;
  lastFeedbackSent = true;
  const t = currentQueue[currentIndex];
  if (!t) return;
  Api.feedback({
    user_id: Tg.USER_ID,
    track_id: t.id,
    action: action,
    elapsed: Math.round((Date.now() - currentTrackStart) / 1000),
  }).catch(() => {});  // полноценная очередь — Phase 2
}
```

Убран try/catch — поскольку `Api.feedback` возвращает Promise, синхронных ошибок там не бывает, а async-ошибки ловятся через `.catch(() => {})`.

## Acceptance Criteria — все пройдены

| Критерий | Результат |
|---|---|
| `grep -c "const Api = (() =>" index.html` → 1 | 1 ✓ |
| `grep -c "function streamUrl(t)" index.html` → 1 | 1 ✓ |
| `grep -c "inflight.set(k, p)" index.html` → 1 | 1 ✓ |
| `grep -cE "fetch\(.*API.*\)" index.html` → 0 | 0 ✓ |
| `grep -c "Api.wave(" index.html` ≥ 1 | 1 ✓ |
| `grep -c "Api.search(" index.html` ≥ 2 | 2 ✓ (tab + modal) |
| `grep -c "Api.streamUrl(" index.html` ≥ 2 | 2 ✓ (playTrack + preloadNext) |
| `grep -c "Api.like(" index.html` ≥ 1 | 1 ✓ |
| `grep -c "Api.dislike(" index.html` ≥ 1 | 1 ✓ |
| `grep -c "Api.feedback(" index.html` → 1 | 1 ✓ |
| `grep -c "window.searchTabQueue" index.html` → 0 | 0 ✓ |
| `grep -c "window.searchModalQueue" index.html` → 0 | 0 ✓ |
| `grep -c "window.genreQueue" index.html` → 0 | 0 ✓ |
| `grep -c "window.collectionQueue" index.html` → 0 | 0 ✓ |
| `grep -c "Store.get('searchTabResults')" index.html` ≥ 1 | 1 ✓ |
| `grep -cE "^\s*const API = Config\.API" index.html` → 0 | 0 ✓ |
| Единственный оставшийся `fetch(` — внутри `Api.req` | 1 (строка 730, `fetch(base + path, ...)`) ✓ |
| `node --check` на extract'е script блока | SYNTAX_OK ✓ |
| `wc -l index.html` | 1230 ✓ |

## Syntax & Safety Checks

- **Script extract → `node --check`:** PASS. Никаких TDZ/redeclaration конфликтов, Api IIFE корректно замыкает `Config.API` и `Tg.USER_ID`.
- **Единственный `fetch(`:** после рефактора во всём файле остался один вызов `fetch(` — внутри `Api.req` (строка 730). Это подтверждает, что Api стал единственным мостом к backend (ARCH-5).
- **Dedup behavior:** при двух быстрых подряд вызовах `Api.waveMoods()` первый кладёт Promise в `inflight.set(k, p)`, второй увидит тот же ключ и получит тот же Promise. По `.finally` Promise снимается — следующий вызов после resolve запустит новый fetch.
- **Error envelope:** `await Api.wave('rock')` при HTTP 500 бросит `Error("HTTP 500 on GET /wave?user_id=...&mood=rock")` — catch-ы во всех loader'ах (loadWave/doTabSearch/...) ловят это и показывают empty state.

## Deviations from Plan

Нет. План исполнен точно как написан. Небольшие нюансы, не являющиеся отклонениями:

- **В render-функциях** (doTabSearch/doModalSearch/loadGenre/loadCollection) используется локальная `const tracks = d.tracks || [];` вместо повторного чтения `Store.get(...)` после `Store.set(...)`. Это чище и полностью эквивалентно — Store.set сохранил ту же ссылку на массив.
- **`loadProfile`** теперь вызывает `await Promise.all([Api.taste(), Api.stats()])` и сразу деструктурирует результаты — не нужны промежуточные `tasteR/statsR` + `.json()`, поскольку Api.req уже возвращает распарсенный JSON.

## Known Stubs

Нет. `sendFeedback` без полноценной очереди — это не stub, а минимальная реализация по плану (полноценный Feedback module — Plan 04 / Phase 2). Документировано inline-комментарием `// полноценная очередь — Phase 2`.

## Threat Flags

Нет нового surface. Api IIFE обёртывает уже существовавшие fetch-сайты без добавления новых endpoint'ов, новых trust boundaries или новых потоков данных. `T-01-04` (Information Disclosure через error envelope) и `T-01-05` (DoS через retry) — зафиксированы в threat model плана как accept/mitigate согласно Phase 1 scope.

## Requirements Progress

- **ARCH-5** («один index.html, CDN-only, централизованный доступ к backend») — **укреплён**: после этого плана существует ровно один `fetch(` в файле (внутри `Api.req`), и все компоненты взаимодействуют с backend через `Api.*`. Формальная галочка для ARCH-5 — после Plan 08 (когда cleanup TEMP shims завершён и у файла нет никаких legacy-идентификаторов).

## Commits

| Hash | Type | Description |
|---|---|---|
| d5a55b8 | feat(01-03) | add Api IIFE with dedup and error envelope |
| ee2ef57 | refactor(01-03) | migrate all fetch calls to Api.* and drop window.*Queue |

## Self-Check: PASSED

- FOUND: `.planning/phases/01-architecture-refactor/01-03-SUMMARY.md`
- FOUND: commit d5a55b8 (`feat(01-03): add Api IIFE with dedup and error envelope`)
- FOUND: commit ee2ef57 (`refactor(01-03): migrate all fetch calls to Api.* and drop window.*Queue`)
- FOUND: `const Api = (() =>` in index.html
- FOUND: `function streamUrl(t)` in index.html
- FOUND: `inflight.set(k, p)` in index.html
- MISSING: `fetch\(.*API.*\)` (correct — all migrated)
- MISSING: `window.searchTabQueue|searchModalQueue|genreQueue|collectionQueue` (correct — removed)
- MISSING: `^\s*const API = Config\.API` (correct — shim removed)
- Script block passes `node --check`
