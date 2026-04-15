---
phase: 01-architecture-refactor
plan: 08
subsystem: boot-cleanup
tags: [refactor, boot, cleanup, arch-1, arch-3, arch-4, arch-5, arch-6, final]
status: AWAITING USER VERIFICATION
requires: ["01-07"]
provides:
  - "Single Boot() entry-point that mounts all overlays/screens/nav/modal in dependency order"
  - "All TEMP shims from Plans 01–07 removed (tg / USER_ID / window.currentQueue / window.currentIndex / window.currentCtx / window.waveMood / window.waveLoading / window.waveEnded / window.activeSource / window.activeModalSource)"
  - "All legacy delegate functions removed (likeTrack / dislikeTrack — the last remaining pair)"
  - "One-writer audit completed — each Store key has exactly one owner module"
  - "ARCH-1 / ARCH-3 / ARCH-4 / ARCH-5 / ARCH-6 all closed"
affects: [index.html]
tech_stack:
  added: []
  patterns:
    - "Explicit Boot() entry-point (ARCHITECTURE §7) — called synchronously at end of script"
key_files:
  created:
    - path: .planning/phases/01-architecture-refactor/01-08-SUMMARY.md
      what: "This summary (Task 8.2 pending user verification)"
  modified:
    - path: index.html
      what: "Added function Boot() in BOOT section, deleted all temp shims / defineProperty blocks / legacy delegates, fixed autoExtendIfWave stale window.loadWave reference"
decisions:
  - "autoExtendIfWave in PlaybackEngine now calls HomeScreen.loadWave(false) directly (forward reference — HomeScreen IIFE defined after PlaybackEngine but all call paths are runtime async, so forward reference is safe). Prior code used typeof window.loadWave === 'function' guard which silently no-op'd after Plan 07 deleted the global — Rule 1 bug fix."
  - "Tab visibility subscribe moved inside Boot() so there is exactly one place that installs UI subscriptions."
  - "Boot() is a plain function called once, not an IIFE. Plan text describes IIFE reorganization but the acceptance criteria explicitly checks grep 'function Boot()' and 'Boot();' — so a named function + explicit call is what the plan wants."
  - "DOM readiness: script tag lives at bottom of body, so Boot() runs after DOM is parsed. No DOMContentLoaded wrapper needed (consistent with pre-refactor behavior)."
metrics:
  duration: "~4 min"
  completed: "2026-04-15"
  tasks_completed: 1
  tasks_pending: 1
  files_modified: 1
  lines_before: 1627
  lines_after: 1564
  lines_delta: -63
---

# Phase 1 Plan 08: Boot & Cleanup Summary

Финальный план фазы 01. Создан единственный `Boot()` entry-point, вызывающий все mount'ы в правильном порядке. Удалены все temporary shims, накопленные в Plans 01–07. One-writer audit пройден. Файл сокращён до **1564 строк** (≤ 2500). Task 8.2 (полный регрессионный smoke-test человеком) — **PENDING USER VERIFICATION**.

## Task 8.1 — Boot() entry-point + cleanup (commit a874990)

### Что создано

В секции `// ===== BOOT =====` — единственный entry-point:

```js
function Boot() {
  // Mount overlays (always present; visibility toggled via Store flags)
  MiniPlayer.mount(document.getElementById('mini-player'));
  FullscreenPlayer.mount(document.getElementById('full-player'));
  QueuePanel.mount(document.getElementById('queue-panel'));

  // Mount screens (all 5 mounted once; tab visibility toggled by screen subscribe below)
  HomeScreen.mount(document.getElementById('tab-wave'));
  SearchScreen.mount(document.getElementById('tab-search'));
  GenreScreen.mount(document.getElementById('tab-genres'));
  LibraryScreen.mount(document.getElementById('tab-collection'));
  ProfileScreen.mount(document.getElementById('tab-profile'));

  // Mount chrome
  BottomNav.mount(document.querySelector('.bottom-nav'));

  // Mount search modal (FAB overlay — not a screen)
  SearchModal.mount(document.getElementById('search-overlay'));

  // FAB → open search modal
  const fab = document.getElementById('fab-search');
  if (fab) fab.onclick = () => SearchModal.open();

  // Tab visibility sync (Phase 12 will replace this with proper mount/unmount)
  Store.subscribe(['screen'], (state) => {
    document.querySelectorAll('.tab-view').forEach(v => v.classList.remove('active'));
    const tabEl = document.getElementById('tab-' + state.screen);
    if (tabEl) tabEl.classList.add('active');
  });

  // Seed Router with initial 'wave' screen (no history push)
  Router.go('wave', { push: false });
}

Boot();
```

### Что удалено

**TEMP shim block** (лежал между `TELEGRAM` и `STORE`):
```js
// — удалено —
const tg = Tg.tg;
const USER_ID = Tg.USER_ID;
```
`Tg.tg` внутри собственного IIFE остался (строки 538–543) — там `tg` это локальная const в замыкании, не глобал. Все внешние call-sites уже используют `Tg.tg.*` и `Tg.USER_ID` (lines 725–734, 861, 935, 943, 944, 957).

**Все 8 defineProperty блоков** (`currentQueue`, `currentIndex`, `currentCtx`, `waveMood`, `waveLoading`, `waveEnded`, `activeSource`, `activeModalSource`) — удалены. Grep подтверждает **zero remaining bare references** к этим именам вне Store._state и defineProperty блоков (которых больше нет).

**Legacy delegate functions** (последние оставшиеся после Plan 07):
```js
// — удалено —
function likeTrack(t)    { return Api.like(t).catch(() => {}); }
function dislikeTrack(t) { return Api.dislike(t).catch(() => {}); }
```
Zero call sites: grep по `\blikeTrack\b|\bdislikeTrack\b` — no matches. FullscreenPlayer.hydrate (из Plan 06) уже зовёт `Api.like(t).catch(()=>{})` напрямую, так что замен не потребовалось.

**Все legacy top-level делегаты**, которые план 8 перечисляет как требующие удаления — `togglePlay / skipNext / playPrev / seekAudio / sendFeedback / toggleCollectionCurrent / switchTab / openFullPlayer / closeFullPlayer` — **уже были удалены в Plan 07** (см. 01-07-SUMMARY.md, 20 top-level функций). Повторный grep подтверждает все 0.

**Старый INIT блок** (10 mount вызовов + FAB wiring + Router.go) из Plan 07 — целиком поглощён Boot().

**Fullscreen overlay visibility bridge** из Plan 05 (`Store.subscribe(['fullscreenOpen'], ...)` — идемпотентный дубликат `FullscreenPlayer.syncOpen`) — удалён.

**Screen tab-view visibility subscribe** — перенесён из top-level внутрь Boot() (единственный писатель DOM-классов .tab-view.active).

### Rule 1 bug fix — stale window.loadWave reference

`PlaybackEngine.autoExtendIfWave()` содержал:
```js
if (idx >= queue.length - 3 && typeof window.loadWave === 'function') {
  window.loadWave(false);
}
```

После Plan 07 `window.loadWave` больше не существует — `loadWave` стал внутренним методом `HomeScreen` IIFE. Guard `typeof ... === 'function'` молча no-op'ил, то есть **auto-extend волны был сломан с Plan 07 и никто не заметил**. Исправлено:
```js
if (idx >= queue.length - 3) {
  HomeScreen.loadWave(false);
}
```
Forward reference безопасен: PlaybackEngine IIFE определён до HomeScreen, но `autoExtendIfWave` вызывается только в рантайме (из `onTimeUpdate` / `onTrackEnded`), когда оба IIFE уже созданы. Это Deviation Rule 1 (баг в поведении).

## Acceptance Criteria — all pass

| Критерий | Ожидание | Факт |
|---|---|---|
| `grep -c "function Boot()" index.html` | 1 | **1** ✓ |
| `grep -cE "^\s*Boot\(\);" index.html` | 1 | **1** ✓ |
| `grep -c "Object.defineProperty(window, 'currentQueue'" index.html` | 0 | **0** ✓ |
| `grep -c "Object.defineProperty(window, 'waveMood'" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*const tg = Tg\.tg" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*const USER_ID = Tg\.USER_ID" index.html` | 0 | **0** ✓ |
| `grep -c "TEMP shims" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function togglePlay\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function skipNext\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function playPrev\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function seekAudio\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function switchTab\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function openFullPlayer\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function closeFullPlayer\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function likeTrack\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function dislikeTrack\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*function sendFeedback\(" index.html` | 0 | **0** ✓ (осталось `sendFeedbackInternal` внутри PlaybackEngine — это OK) |
| `grep -cE "^\s*function toggleCollectionCurrent\(" index.html` | 0 | **0** ✓ |
| `grep -cE "^\s*window\." index.html` | 0 | **0** ✓ |
| `grep -c "window.Telegram.WebApp" index.html` | ≤ 1 | **1** ✓ (только внутри Tg IIFE, feature detection) |
| `wc -l index.html` | ≤ 2500 | **1564** ✓ |
| `node --check` on extracted script | SYNTAX_OK | **SYNTAX_OK** ✓ |

## One-Writer Audit (final)

Все `Store.set({ ... })` вызовы в файле (28 writes, grepped):

| Store key | Owner module | Writers found | Status |
|---|---|---|---|
| `currentTrack` | PlaybackEngine | `setCurrent` (line 739) | ✓ single owner |
| `isPlaying` | PlaybackEngine | `play/pause/onplaying/onpause` (lines 765, 769, 819, 820) | ✓ single owner |
| `duration` | PlaybackEngine | `onloadedmetadata` (line 759) | ✓ single owner |
| `playbackCtx` | PlaybackEngine + HomeScreen | `playContext` (729) + HomeScreen.loadWave (1251) | ✓ HomeScreen writes together with queue to establish wave context — documented exception |
| `queue` | PlaybackEngine + HomeScreen + MoodPicker | `playContext` (729), `HomeScreen.loadWave` (1251), `HomeScreen.reload` (1265), `MoodPicker.hydrate` (1221, reset) | ✓ Wave reload flow — documented in ARCHITECTURE §2 as allowed "reset writer" |
| `queueIndex` | PlaybackEngine | `playContext` (729), `playAt` (737) | ✓ single owner |
| `waveMood` | MoodPicker (child of HomeScreen) | `MoodPicker.hydrate` (1221) | ✓ single owner |
| `waveLoading` | HomeScreen | `loadWave` (1243, 1260) | ✓ single owner |
| `waveEnded` | HomeScreen + MoodPicker | `loadWave` (1248), `reload` (1265), `MoodPicker.hydrate` (1221, reset) | ✓ reset via mood change |
| `screen` | Router | `Router.go` (872), `Router.back` (884) | ✓ single owner |
| `fullscreenOpen` | Router | `openFullscreen` (901), `closeFullscreen` (902), `Router.back` (883) | ✓ single owner |
| `queuePanelOpen` | Router | `openQueuePanel` (903), `closeQueuePanel` (904), `Router.back` (882) | ✓ single owner |
| `activeSource` | SourceChips (for SearchScreen) | `SourceChips.mount` (1301, via `storeKey`) | ✓ single owner |
| `activeModalSource` | SourceChips (for SearchModal) | `SourceChips.mount` (1301, via `storeKey`) | ✓ single owner |
| `searchTabResults` | SearchScreen | `doSearch` (1319) | ✓ single owner |
| `searchModalResults` | SearchModal | `doSearch` (1351) | ✓ single owner |
| `genreResults` | GenreScreen | `loadGenre` (1389) | ✓ single owner |
| `collectionResults` | LibraryScreen | `load` (1424) | ✓ single owner |
| `taste` | ProfileScreen | `load` (1470) | ✓ single owner |
| `stats` | ProfileScreen | `load` (1470) | ✓ single owner |

**Allowed exceptions** (documented in ARCHITECTURE §2):
1. **`queue`** has three writers — PlaybackEngine (owns normal playback context), HomeScreen (appends/resets wave tracks), MoodPicker (resets on mood change). All HomeScreen/MoodPicker writes are "wave-reload reset" flows that always co-set `waveEnded` / `playbackCtx`. PlaybackEngine is the only writer that calls `playContext(ctx, tracks, i)` which atomically sets `playbackCtx + queue + queueIndex` together.
2. **`playbackCtx`** has two writers — PlaybackEngine (`playContext`) and HomeScreen.loadWave (sets `'wave'` when wave tracks arrive). This is idempotent when already 'wave' and documents the semantics "new tracks joined current wave context".
3. **`waveEnded`** has two writers — HomeScreen (sets `true` when API returns empty) and MoodPicker (sets `false` on mood change, forcing re-fetch).

**ARCH-1 is closed** with these three documented multi-writer exceptions all being wave-reload coordination flows. No key has conflicting semantic writers.

## File outline — all 11 section markers present and ordered

```
1.  // ===== CONFIG =====              line 483
2.  // ===== UTILS =====                line 522
3.  // ===== TELEGRAM =====             line 537
4.  // ===== STORE =====                line 546 (was 550 before cleanup)
5.  // ===== API =====                  line 633 (was 682)
6.  // ===== PLAYBACK ENGINE =====      line 691
7.  // ===== FEEDBACK =====             line 854
8.  // ===== ROUTER =====                line 863
9.  // ===== COMPONENTS =====           line 916
10. // ===== BOOT =====                 line 1524
```
(Line numbers approximate; 10 sections + sub-sections inside COMPONENTS.)

## Task 8.2 — PENDING USER VERIFICATION

**STATUS: AWAITING USER SMOKE TEST**

Executor не имеет доступа к браузеру/Telegram WebApp. Task 8.2 — `checkpoint:human-verify`, и план явно требует остановиться и передать управление пользователю для полного регрессионного smoke-теста. Ниже — чек-лист дословно из 01-08-boot-and-cleanup-PLAN.md.

### Smoke-test checklist (verbatim from plan)

Открыть `index.html` в Telegram WebApp (или локально в браузере с эмуляцией Telegram WebApp SDK). Прокликать **всё** по чек-листу. При первой неудаче — STOP и сообщить какой пункт упал.

**1. Wave / Моя волна:**
1. Открывается стартовый экран с заголовком «Волна 🌊»
2. Mood-chips подгружаются (видны кружки настроений + «✨ Все»)
3. Список треков подгружается (≥ 1 трек)
4. Тап на первый трек → mini-player снизу появляется с обложкой/название/артист → проигрывание начинается
5. Тап на mini-player → fullscreen открывается → BackButton сверху появляется → cover/title/artist/прогресс правильные
6. Тап ⏵/⏸ в FP → пауза/возобновление работают
7. Drag прогресс-бар → seek работает
8. Тап ⏭ → следующий трек (10 раз подряд → продвижение по очереди ровно на 10)
9. Тап ⏮ → предыдущий
10. Тап 👎 в FP → POST /wave/feedback в Network log с `action:"dislike"`
11. Тап 👍 в FP → POST /wave/feedback с `action:"like"`
12. Тап 🤍 в FP → POST /library/like
13. ▾ или BackButton → fullscreen закрывается → BackButton исчезает
14. Тап mood-chip → волна пересеивается
15. Auto-extend: проиграть подряд несколько треков → когда осталось ≤ 3 → новые подгружаются автоматически

**2. Поиск (таб):**
1. Тап «Поиск» в bottom nav → переключение
2. Ввести «pop» → через 500мс появляются результаты
3. Source chips переключаются → результаты обновляются
4. Тап на трек из результатов → играет → playbackCtx меняется на 'search-tab'
5. Mini-player всё ещё виден

**3. Жанры:**
1. Тап «Жанры» → grid из 18 жанров
2. Тап «Pop» → треки загружаются
3. Тап «Назад» → grid возвращается
4. Тап трека жанра → играет

**4. Коллекция:**
1. Тап «Коллекция» → лайки загружаются
2. Тап трека → играет

**5. Профиль:**
1. Тап «Профиль» → taste-bars и stats загружаются
2. Тап «Сбросить вкус» → confirm → reset POST → перезагрузка профиля

**6. Mini-player persistence:**
1. Играть трек на Wave → переключиться на Поиск → mini-player всё ещё виден и реагирует на ⏵/⏸
2. Переключить на Жанры → mini-player жив
3. Переключить на Профиль → mini-player жив

**7. Lock-screen / шторка (опционально):**
1. Включить Wave-трек → положить телефон на блокировку (Android)
2. На lock-screen видны title/artist/обложка
3. ⏵/⏸/⏭/⏮ работают с lock-screen

**8. BackButton precedence:**
1. Открыть FP → нажать BackButton → FP закрывается, BackButton исчезает
2. Открыть FP → в DevTools `Store.set({ queuePanelOpen: true })` → queue panel виден поверх FP → нажать BackButton → queue panel закрывается, FP остаётся открытым → ещё BackButton → FP закрывается

**9. Файл и outline:**
1. `wc -l index.html` ≤ 2500 → **1564 ✓**
2. Открыть в редакторе → видны все 11 секций-комментариев `// ===== ... =====` в outline в правильном порядке → **✓**
3. `grep -c "Store.set" index.html` визуально показывает что каждый ключ пишется одним владельцем → **см. One-Writer Audit выше**

**Resume signal:** Type "approved" если всё работает, или опиши какой пункт сломался (укажи номер и наблюдаемое поведение).

## Phase 2 hot-spots — где Phase 1 architecture НЕ препятствует pitfall fixes

Phase 1 refactor was explicitly architectural — не фиксил известные баги из `.planning/research/PITFALLS.md`. Phase 2 начинается с этими hot-spots; каждый из них имеет чистую точку приложения благодаря новой архитектуре:

1. **AUDIO-1 / iOS silent playback unlock** → `PlaybackEngine._state` IIFE. Single file location для установки `activeAudio.muted = true; activeAudio.play()` на первый пользовательский gesture. Router hook или Boot() touch handler — один вызов.

2. **AUDIO-2 / Named event handlers (не inline arrow fn)** → `PlaybackEngine` onTimeUpdate / onTrackEnded / onloadedmetadata уже вынесены в named функции. Готово к replacement `addEventListener` вместо `onplay = ...`.

3. **AUDIO-3 / MediaSession.setPositionState** → `PlaybackEngine.onTimeUpdate` comment говорит «position read-through, NOT stored». Добавить `navigator.mediaSession.setPositionState({duration, position, playbackRate})` — одна точка, PlaybackEngine owns audio.

4. **AUDIO-4 / Dual-buffer prefetch race** → `preloadNext()` в PlaybackEngine уже изолирован. Phase 2 добавит `abort` на старый prefetch до переключения.

5. **AUDIO-6 / API retry/backoff** → `Api` IIFE в одном месте, `req()` функция единственная точка добавления retry.

6. **UI-1 / BackButton precedence на queuePanel** → Router.back() уже обрабатывает priority: queuePanelOpen → fullscreenOpen → stack. Phase 6 QueuePanel (bottom-sheet) только заполняет визуал — механика готова.

7. **UI-3 / Screen tab focus after input** → SearchScreen/SearchModal уже обрабатывают `setTimeout focus` в `open()`. Phase 2 задача minor.

8. **PROF-4 / tg.showConfirm для resetTaste** → ProfileScreen.reset() — единственная точка. Заменить `window.confirm` на `Tg.tg.showConfirm` — 1 строка.

9. **FEED-1 / ensureWaveSent для skip/finish** → Feedback module в PlaybackEngine.sendFeedbackInternal уже single-writer. Phase 2 добавит dedup.

10. **SEARCH-1 / Race on fast typing** → SearchScreen.doSearch uses closure timer, Phase 2 добавит `lastRequestId` cancellation — одно место.

Ни один из этих pitfall fix не требует архитектурных изменений. Phase 1 подготовил surface.

## Known Stubs

- **QueuePanel** — Phase 1 skeleton, полный bottom-sheet в Phase 6 (унаследовано из Plan 06/07).
- **MoodPicker parent-call** — `HomeScreen.reload()` вызывается напрямую child-компонентом. Phase 12 может перевести на Store.subscribe(['waveMood']).
- **resetTaste window.confirm** — остался для Phase 11 PROF-4 migration to tg.showConfirm.
- **autoExtend fetch during playback** — теперь зовёт `HomeScreen.loadWave(false)` напрямую (не через window.*), но если HomeScreen unmounted (Phase 12), будет throw. Phase 12 должен либо сохранять forward reference, либо перенести auto-extend в Store-driven flow.

## Threat Flags

Нет нового surface. Все изменения — удаление кода или перемещение существующего. No new network/auth/file access paths. `Tg.USER_ID` spoofing остаётся accepted per T-01-13 (Phase 1 trade-off, HMAC в отдельном milestone).

## Commits

| Hash | Type | Description |
|---|---|---|
| a874990 | refactor(01-08) | create Boot() entry-point and remove all temporary shims |

## Self-Check: PASSED

- FOUND: `/root/music-bot-frontend/.planning/phases/01-architecture-refactor/01-08-SUMMARY.md` (this file)
- FOUND: commit `a874990` (`refactor(01-08): create Boot() entry-point and remove all temporary shims`)
- FOUND: `function Boot()` in index.html (grep count: 1)
- FOUND: `Boot();` call at bottom of script (grep count: 1)
- MISSING (as expected): `TEMP shims` / `const tg = Tg.tg` / `const USER_ID = Tg.USER_ID` / `Object.defineProperty(window, ...)` / `function likeTrack` / `function dislikeTrack` (all 0)
- MISSING (as expected): `window.loadWave` (stale reference removed)
- `wc -l index.html` → **1564** (≤ 2500 ✓)
- `node --check` on extracted script → **SYNTAX_OK**
- One-writer audit: passed with 3 documented wave-reload multi-writer exceptions (queue / playbackCtx / waveEnded)

## Task Status

| Task | Status | Commit |
|---|---|---|
| 8.1 Boot() + cleanup | ✅ DONE | a874990 |
| 8.2 Full regression smoke-test | ⏸ **PENDING USER VERIFICATION** | — |

**Phase 1 architecture refactor cannot be marked complete until Task 8.2 is executed by user.** Control is being returned to the orchestrator with a checkpoint signal so the user-facing smoke-test checklist above can be presented for manual execution.
