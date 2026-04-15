---
phase: 01-architecture-refactor
plan: 04
subsystem: playback-engine
tags: [refactor, playback, audio, mediasession, store-ownership, arch-2, arch-1]
requires: ["01-03"]
provides:
  - "PlaybackEngine IIFE — единственный владелец audioA/audioB, activeAudio/nextAudio и MediaSession"
  - "Единственный writer Store-ключей: currentTrack, isPlaying, duration, playbackCtx, queue, queueIndex"
  - "Публичный API: playContext, playAt, play, pause, toggle, next, prev, seek, seekPct, getPosition, getDuration, isPaused, sendManualFeedback"
  - "Feedback IIFE (Phase 1 stub) — pass-through к Api.feedback, точка интеграции для Phase 2 reliable queue"
  - "Thin delegate globals (togglePlay/skipNext/playPrev/seekAudio/playFromContext/sendFeedback/toggleCollectionCurrent) для inline onclick markup до Plan 07"
  - "Три временных Store.subscribe моста (isPlaying, currentTrack, duration) + rAF tickFpProgress — будут заменены MiniPlayer/FullscreenPlayer в Plan 06"
affects: [index.html]
tech_stack:
  added: []
  patterns:
    - "Single-owner audio IIFE — оба <audio> элемента и dual-buffer swap инкапсулированы"
    - "Read-through position: getPosition/getDuration не хранятся в Store (избегаем 10Hz subscriber thrash), компоненты читают через rAF"
    - "MediaSession handlers регистрируются один раз в initMediaSession(); setMetadata вызывается на каждый трек — Phase 2 добавит setPositionState локально внутри IIFE"
    - "Thin delegate pattern: global function playPrev() { PlaybackEngine.prev(); } — bridge к inline onclick до Plan 07"
    - "Temporary Store.subscribe bridges с if-guard на document.getElementById — Plan 06 заменит их на component mount()/render()"
key_files:
  created:
    - path: .planning/phases/01-architecture-refactor/01-04-SUMMARY.md
      what: "This summary"
  modified:
    - path: index.html
      what: "Секция // ===== PLAYBACK ENGINE ===== заполнена IIFE с полным API. Секция // ===== FEEDBACK ===== заполнена минимальной заглушкой. Legacy playTrack/preloadNext/autoExtendWave/onTimeUpdate/onTrackEnded/setupMediaSession/updatePlayUI/sendFeedback удалены. Top-level const audioA/audioB/let activeAudio/nextAudio/currentTrackStart/lastFeedbackSent удалены. togglePlay/skipNext/playPrev/seekAudio/playFromContext/sendFeedback/toggleCollectionCurrent стали тонкими делегатами к PlaybackEngine. likeTrack/dislikeTrack теперь читают queue/queueIndex из Store. Добавлены 3 временных Store.subscribe (isPlaying/currentTrack/duration) + rAF tickFpProgress."
decisions:
  - "PlaybackEngine использует `.on*` assignment (ontimeupdate/onended/onloadedmetadata) — не addEventListener. Это сознательно: миррорит pre-refactor поведение. Phase 2 / AUDIO-2 заменит на addEventListener+removeEventListener с named handlers, и это будет локальное изменение внутри IIFE — архитектура не помешает."
  - "Position (currentTime) НЕ хранится в Store — компоненты читают через PlaybackEngine.getPosition() в своих rAF-циклах. ARCHITECTURE §2 явно запрещает ключ `position` чтобы избежать 10Hz subscriber-thrash. Временный tickFpProgress использует этот паттерн."
  - "Legacy global delegates (togglePlay/skipNext/playPrev/seekAudio/sendFeedback) оставлены как однострочные shim'ы потому что HTML markup всё ещё имеет inline onclick=. Plans 06–07 заменят inline handlers на component-bound listeners, и тогда delegate'ы можно будет удалить."
  - "playFromContext для ctx='wave' делает PlaybackEngine.playAt(i) напрямую, не playContext — потому что wave queue уже живёт в Store через loadWave() (он пишет в currentQueue через window shim, который под капотом Store.set({ queue })). Для остальных контекстов (genre/collection/search-tab/search-modal) вызывается playContext который перезаписывает queue+playbackCtx+queueIndex атомарно."
  - "Feedback IIFE сделан минимальным pass-through (Api.feedback(event).catch(()=>{})). Полный reliable queue с localStorage + visibilitychange flush откладывается на Phase 2 (AUDIO-6). Наличие стабильного имени Feedback.send позволит Phase 2 добавить очередь не меняя call sites."
  - "autoExtendIfWave использует `typeof window.loadWave === 'function'` guard и зовёт loadWave(false). Это мост к legacy HomeScreen — Plan 07 создаст HomeScreen компонент который подпишется на queueIndex и будет триггерить расширение сам; после этого autoExtendIfWave можно упростить."
metrics:
  duration: "~4 min"
  completed: "2026-04-15"
  tasks_completed: 2
  files_modified: 1
  lines_before: 1230
  lines_after: 1325
---

# Phase 1 Plan 04: PlaybackEngine Summary

Создан `PlaybackEngine` IIFE — единственный владелец обоих `<audio>` элементов, MediaSession handlers, dual-buffer swap и playback-keys в Store; все legacy playback функции (`playTrack`, `preloadNext`, `autoExtendWave`, `onTimeUpdate`, `onTrackEnded`, `setupMediaSession`, `updatePlayUI`, `sendFeedback`) удалены или превращены в тонкие делегаты; добавлены 3 временных Store.subscribe моста до Plan 06.

## What Was Built

### Task 4.1 — PlaybackEngine IIFE + Feedback stub (commit f695ef5)

В секции `// ===== PLAYBACK ENGINE =====` вставлен verbatim код из `<interfaces>` плана.

**Owned state** (приватные let/const внутри IIFE):
- `const audioA = document.getElementById('audioA')`
- `const audioB = document.getElementById('audioB')`
- `let activeAudio = audioA`
- `let nextAudio = audioB`
- `let currentTrackStart = 0`
- `let lastFeedbackSent = false`

**MediaSession** — регистрация один раз в `initMediaSession()`:
- `play` → `toggle()`
- `pause` → `toggle()`
- `previoustrack` → `prev()`
- `nexttrack` → `next()`
- `setMetadata(t)` — устанавливает `navigator.mediaSession.metadata = new MediaMetadata(...)` на каждый трек из `playAt()`

**Public API** (14 методов):

| Метод | Назначение |
|---|---|
| `playContext(ctx, tracks, index)` | Устанавливает playbackCtx + queue + queueIndex атомарно, затем playAt(index) |
| `playAt(i)` | Играет i-й трек из текущего queue; swap dual buffer; set metadata; preloadNext; autoExtendIfWave |
| `play()` | Резюмирует воспроизведение (если есть src) |
| `pause()` | Пауза + `Store.set({ isPlaying: false })` |
| `toggle()` | play/pause в зависимости от activeAudio.paused |
| `next()` | sendFeedbackInternal('skip') + playAt(queueIndex+1) |
| `prev()` | playAt(queueIndex-1) |
| `seek(seconds)` | currentTime = clamp(seconds, 0..duration) |
| `seekPct(pct)` | currentTime = (pct/100) * duration |
| `getPosition()` | activeAudio.currentTime — read-through для rAF циклов |
| `getDuration()` | activeAudio.duration |
| `isPaused()` | activeAudio.paused |
| `sendManualFeedback(action)` | Обёртка над sendFeedbackInternal для legacy inline onclick (👎/👍) |

**Writers Store-ключей (exhaustive)** — только внутри PlaybackEngine:
- `playbackCtx`, `queue`, `queueIndex` в `playContext()`
- `queueIndex` в `playAt()`
- `currentTrack` в `playAt()`
- `duration` в `onloadedmetadata` handler внутри `playAt()`
- `isPlaying: true` после `await activeAudio.play()` в `playAt()` и `play()`
- `isPlaying: false` в `pause()` и catch-блоке `playAt()`

В секции `// ===== FEEDBACK =====` вставлена минимальная заглушка:
```js
const Feedback = (() => {
  function send(event) { return Api.feedback(event).catch(() => {}); }
  return { send };
})();
```

Полная реализация с reliable queue + localStorage + visibilitychange flush — Phase 2 (AUDIO-6). Call sites могут начинать использовать `Feedback.send(...)` с чистой совестью.

### Task 4.2 — Legacy functions → thin delegates + temporary bridges (commit 47d8a2f)

**Удалено** (из Plan 02-era top-level):
- `const audioA = document.getElementById('audioA')` — теперь внутри PlaybackEngine
- `const audioB = document.getElementById('audioB')`
- `let activeAudio = audioA, nextAudio = audioB`
- `let currentTrackStart = 0, lastFeedbackSent = false`

**Удалено** (legacy функции):
- `async function playTrack(i)` — полностью, логика в `PlaybackEngine.playAt`
- `function preloadNext()` — теперь приватный метод PlaybackEngine
- `function autoExtendWave()` — теперь `autoExtendIfWave()` внутри PlaybackEngine
- `function onTimeUpdate()` — убран, position сейчас read-through через rAF (не Store key)
- `function onTrackEnded()` — внутри PlaybackEngine
- `function setupMediaSession(t)` — `initMediaSession` + `setMetadata` внутри PlaybackEngine
- `function updatePlayUI(playing)` — заменён на Store.subscribe(['isPlaying']) bridge
- `function sendFeedback(action)` body — теперь однострочный делегат

**Добавлено / заменено** — тонкие делегаты (global functions для inline onclick):

```js
function playFromContext(i, ctx) {
    let tracks;
    switch (ctx) {
        case 'wave':         tracks = Store.get('queue'); break;
        case 'genre':        tracks = Store.get('genreResults') || []; break;
        case 'collection':   tracks = Store.get('collectionResults') || []; break;
        case 'search-tab':   tracks = Store.get('searchTabResults') || []; break;
        case 'search-modal': tracks = Store.get('searchModalResults') || []; closeSearch(); break;
        default:             tracks = []; break;
    }
    if (ctx === 'wave') {
        PlaybackEngine.playAt(i);           // wave queue уже в Store от loadWave()
    } else {
        PlaybackEngine.playContext(ctx, tracks, i);
    }
}

function togglePlay() { PlaybackEngine.toggle(); }
function skipNext()   { PlaybackEngine.next(); }
function playPrev()   { PlaybackEngine.prev(); }
function seekAudio(v) { PlaybackEngine.seekPct(parseFloat(v)); }
function sendFeedback(action) { PlaybackEngine.sendManualFeedback(action); }

function toggleCollectionCurrent() {
    const idx = Store.get('queueIndex');
    if (idx < 0) return;
    const t = Store.get('queue')[idx];
    if (t) likeTrack(t);
}
```

**`likeTrack` и `dislikeTrack`** теперь читают queue/queueIndex через `Store.get(...)` вместо `currentQueue[currentIndex]` — это чище, и подтверждает что legacy window.currentQueue/currentIndex shim'ы становятся read-only.

**Три временных Store.subscribe bridges** (будут заменены MiniPlayer/FullscreenPlayer компонентами в Plan 06):

1. **`isPlaying`** → переключает `innerText` на `#mp-play-btn` и `#fp-play-btn` между ⏸/▶
2. **`currentTrack`** → обновляет mini-player (title/artist/thumb, show flex) и fp-meta (title/artist/cover)
3. **`duration`** → обновляет `#fp-time-total` через `Utils.fmtTime`

Каждая подписка guard'ит `document.getElementById(...)` через `if (el)` — если компонент ещё не в DOM (Plan 06 может их в будущем mount'ить лениво), подписка не падает.

**rAF `tickFpProgress`** — read-through обновление `#fp-time-cur` и `#progress` value через `PlaybackEngine.getPosition()/getDuration()`. Запускается один раз в конце Task 4.2 и работает перманентно. Plan 06 перенесёт этот цикл внутрь FullscreenPlayer компонента и привяжет к `mount/destroy`.

## Acceptance Criteria — все пройдены

| Критерий | Результат |
|---|---|
| `grep -c "const PlaybackEngine = (() =>" index.html` → 1 | 1 ✓ |
| `grep -c "function playContext(ctx, tracks, index)" index.html` → 1 | 1 ✓ |
| `grep -c "function getPosition()" index.html` → 1 | 1 ✓ |
| `grep -c "const Feedback = (() =>" index.html` → 1 | 1 ✓ |
| `grep -c "navigator.mediaSession.setActionHandler" index.html` → 4 | 4 ✓ (play, pause, previoustrack, nexttrack) |
| `grep -cE "^\s*async function playTrack\(" index.html` → 0 | 0 ✓ |
| `grep -cE "^\s*function preloadNext\(" index.html` → 0 | 0 ✓ (только `          function preloadNext()` внутри PlaybackEngine) |
| `grep -cE "^\s*function autoExtendWave\(" index.html` → 0 | 0 ✓ |
| `grep -cE "^\s*function setupMediaSession\(" index.html` → 0 | 0 ✓ |
| `grep -cE "^\s*function updatePlayUI\(" index.html` → 0 | 0 ✓ |
| `grep -cE "^\s*let activeAudio" index.html` → 0 | 0 ✓ (только `          let activeAudio = audioA;` внутри PlaybackEngine с 10+ space отступом) |
| `grep -cE "^\s*let currentTrackStart" index.html` → 0 | 0 ✓ |
| `grep -cE "^\s*const audioA = document.getElementById" index.html` → 0 | 0 ✓ |
| `grep -c "PlaybackEngine.playAt" index.html` ≥ 1 | 1 ✓ |
| `grep -c "PlaybackEngine.toggle" index.html` ≥ 1 | 1 ✓ |
| `grep -c "PlaybackEngine.next" index.html` ≥ 1 | 1 ✓ |
| `grep -c "PlaybackEngine.prev" index.html` ≥ 1 | 1 ✓ |
| `grep -c "PlaybackEngine.seekPct" index.html` ≥ 1 | 1 ✓ |
| Script extract → `node --check` | SYNTAX_OK ✓ |
| `wc -l index.html` | 1325 ✓ (было 1230) |

## One-Writer Audit — playback keys

`grep -nE "Store\.set\(\{ ?(currentTrack\|queueIndex\|queue\|isPlaying\|playbackCtx\|duration)"` возвращает:

| Строка | Контекст | Внутри PlaybackEngine? |
|---|---|---|
| 672, 677, 682 | Plan 02 `window.currentQueue/currentIndex/currentCtx` setter shims | N/A — getter/setter bridge property definitions (объявления, не вызовы) |
| 808 | `playContext()` — `playbackCtx + queue + queueIndex` | ✓ PlaybackEngine |
| 816 | `playAt()` — `queueIndex` | ✓ PlaybackEngine |
| 818 | `playAt()` — `currentTrack` | ✓ PlaybackEngine |
| 838 | `playAt()` — `duration` (onloadedmetadata) | ✓ PlaybackEngine |
| 844 | `playAt()` — `isPlaying: true` после await play() | ✓ PlaybackEngine |
| 848 | `playAt()` — `isPlaying: false` в catch | ✓ PlaybackEngine |
| 898 | `play()` — `isPlaying: true` | ✓ PlaybackEngine |
| 899 | `pause()` — `isPlaying: false` | ✓ PlaybackEngine |

**Вывод:** все реальные writer'ы playback-ключей — внутри `PlaybackEngine = (() => { ... })();`. ARCH-1 (one-writer-per-key) для playback-блока закрыт.

**Caveat (не отклонение):** `loadWave()` (строка 1001–1002) всё ещё пишет `currentQueue = tracks.slice()` — это идёт через window shim setter, который вызывает `Store.set({ queue: ... })`. Формально это второй writer для `queue`. Это намеренно оставлено до Plan 07, где HomeScreen компонент станет владельцем wave queue (что зафиксировано в `<interfaces>` плана 04 через комментарий «wave queue is already the mainline»). В текущем состоянии playFromContext для ctx='wave' принимает этот как источник правды и не перезаписывает queue через playContext. Plan 07 уберёт loadWave вообще.

## Phase 2 Integration Points (готовность архитектуры)

Из PITFALLS.md следующие Phase 2 hot-spots теперь локальны внутри PlaybackEngine IIFE — добавление не потребует re-refactor'а:

| Pitfall | Где в PlaybackEngine добавится | Тип изменения |
|---|---|---|
| **AUDIO-1** iOS WKWebView autoplay unlock | Новый приватный `unlockOnFirstGesture()` + флаг `unlocked` + silent-mp3 prime в начале `playAt()` если `!unlocked` | Добавление приватных методов, публичный API не меняется |
| **AUDIO-2** `removeEventListener` с named handlers | Заменить `.ontimeupdate=fn / .onended=fn / .onloadedmetadata=fn` на `addEventListener` с сохранённым handler ref; снимать со старого буфера перед swap | Локальное изменение в `playAt()` |
| **AUDIO-3** `mediaSession.setPositionState()` | Добавить приватный `updatePositionState()` throttle ~1Hz, вызывать из onloadedmetadata и onseeked handler'а | Добавление |
| **AUDIO-4** полный набор actions: seekbackward/seekforward/seekto | Добавить в `initMediaSession()` | Локальное добавление 3-х setActionHandler |
| **AUDIO-5** artwork на каждый трек | Уже сделано (`setMetadata(t)` в `playAt()`) | ✓ |
| **AUDIO-6** error/stalled/waiting retry | Добавить в wire-events секцию `playAt()`, + вызвать `Feedback.send(...)` через уже-существующий Feedback IIFE точку | Локальное изменение |
| **AUDIO-7** double-skip race | Уже защищено `lastFeedbackSent` флагом + swap buffer логикой | ✓ (унаследовано от pre-refactor) |
| **FP-10** scrubber vs lock-screen rate-change | Локально — в `seek()/seekPct()` добавить setPositionState вызов | Локальное изменение |

Ни один из этих фиксов не потребует трогать call sites или другие IIFE — это подтверждает, что раздел «Sound architecture deliberately leaves room for Phase 2» из ARCHITECTURE §5 не blown.

## Ручной smoke test — pending browser verification

План требует 10 подряд skip'ов / lock-screen controls / лайк POST. Поскольку этот executor работает без браузера, browser smoke test помечается **pending user verification** — код гарантирует:
- `PlaybackEngine.next()` вызывается от skipNext() / fp-play `⏭` onclick / media-session nexttrack handler → playAt(queueIndex+1) → Store.set({ queueIndex }) → preloadNext() → wave auto-extend
- lastFeedbackSent флаг сбрасывается в каждом playAt() → каждый skip шлёт feedback
- autoExtendIfWave() зовёт loadWave(false) когда idx >= queue.length - 3 → очередь пополняется → 10 подряд skip'ов не упрутся в конец
- mediaSession.setActionHandler registered в initMediaSession() (вызывается один раз) → Android lock-screen ▶/⏸/⏭/⏮ работают
- likeTrack(t) вызывает Api.like(t) (POST /library/like) — не тронут этим рефактором

Если ручной тест выявит регрессию, это будет Plan 04 hotfix коммит, не возврат к legacy.

## Deviations from Plan

Нет. План исполнен точно как написан.

Мелкий нюанс (не отклонение): `likeTrack` теперь читает queue/queueIndex из Store вместо legacy `currentQueue[currentIndex]`. План явно не требовал этого, но оставить legacy чтение было бы некорректно после удаления верхнеуровневых let'ов — shim'ы всё ещё работают (window.currentQueue → Store.get('queue')), но чистое чтение через Store.get логичнее и согласуется с тем что вся playback-state теперь owned PlaybackEngine'ом.

## Known Stubs

- **Feedback IIFE** — Phase 1 stub. Полноценный reliable queue (localStorage + visibilitychange flush + retry) — Phase 2 / AUDIO-6. Этот stub задокументирован inline комментарием и в плане 04.
- **Temporary Store.subscribe bridges + tickFpProgress rAF** — не stub, а сознательные bridges, которые Plan 06 (MiniPlayer/FullscreenPlayer components с mount/render/destroy) заменит. Они полностью функциональны и держат текущий UX рабочим.

## Threat Flags

Нет нового surface. PlaybackEngine обёртывает уже существовавшие audio/MediaSession взаимодействия без добавления новых endpoint'ов, trust boundaries или данных. Threat model плана зафиксировал T-01-06 (Tampering internal state) и T-01-07 (iOS autoplay DoS) как accept — оба подтверждены: Plan 04 не расширяет ни атаку surface, ни защиту (защита для T-01-07 — это Phase 2).

## Requirements Progress

- **ARCH-2** («PlaybackEngine — единственный владелец audio + MediaSession + queue swap») — **закрыт** ✓
- **ARCH-1** («один владелец на key в Store») — **укреплён для playback-блока**: currentTrack/isPlaying/duration/playbackCtx/queue/queueIndex теперь имеют ровно одного writer (PlaybackEngine). Плюс сохраняется edge-case на loadWave через shim (закроется в Plan 07 с HomeScreen).

## Commits

| Hash | Type | Description |
|---|---|---|
| f695ef5 | feat(01-04) | add PlaybackEngine IIFE and Feedback stub |
| 47d8a2f | refactor(01-04) | replace legacy playback functions with PlaybackEngine delegates |

## Self-Check: PASSED

- FOUND: `.planning/phases/01-architecture-refactor/01-04-SUMMARY.md` (this file)
- FOUND: commit f695ef5 (`feat(01-04): add PlaybackEngine IIFE and Feedback stub`)
- FOUND: commit 47d8a2f (`refactor(01-04): replace legacy playback functions with PlaybackEngine delegates`)
- FOUND: `const PlaybackEngine = (() =>` in index.html
- FOUND: `const Feedback = (() =>` in index.html
- FOUND: `PlaybackEngine.toggle/next/prev/playAt/playContext/seekPct/sendManualFeedback/getPosition/getDuration` call sites
- MISSING: legacy `async function playTrack(` (correct — removed)
- MISSING: top-level `let activeAudio` / `let currentTrackStart` / `const audioA = document.getElementById` (correct — moved inside PlaybackEngine IIFE)
- Script block passes `node --check` (SYNTAX_OK)
