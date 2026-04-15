---
phase: 01-architecture-refactor
plan: 07
subsystem: screen-components
tags: [refactor, components, screens, arch-3, cleanup]
requires: ["01-06"]
provides:
  - "TrackCard pure render function with data-idx/data-action delegation"
  - "attachTrackListDelegate helper shared by Home/Search/SearchModal/Genre/Library screens"
  - "9 screen/child IIFE components: HomeScreen, MoodPicker, SourceChips, SearchScreen, SearchModal, GenreScreen, LibraryScreen, TasteBars, ProfileScreen, BottomNav"
  - "ARCH-3 fully closed — 12/12 components as IIFE with mount/destroy (TrackCard pure-fn + MiniPlayer/FullscreenPlayer/QueuePanel from Plan 06 + the 9 from this plan)"
  - "All 20 legacy top-level load/render functions deleted from index.html"
  - "All inline onclick= attributes stripped from markup (count: 0)"
  - "Plan 05 temporary Store.subscribe(['screen']) lazy-load block replaced by component-driven lazy load"
  - "TEMP shims trimmed: SOURCES/SOURCE_ICONS/GENRES/escapeHtml/escapeAttr/fmtTime removed"
affects: [index.html]
tech_stack:
  added: []
  patterns:
    - "Component IIFE with mount/render/destroy and per-key subscribe (ARCHITECTURE §4)"
    - "Event delegation on parent root with data-idx/data-action — one click listener per list, bounds-checked tracks[idx] lookup"
    - "Component-driven lazy load: LibraryScreen and ProfileScreen each subscribe to ['screen'] inside their own mount and trigger load() when entering their tab — no shared lazy-load dispatcher"
    - "Child component composition: HomeScreen mounts MoodPicker, SearchScreen and SearchModal mount SourceChips, ProfileScreen mounts TasteBars — each child gets its own root element via querySelector from parent"
    - "MoodPicker triggers HomeScreen.reload() on selection — one-way parent reference (child knows parent); acceptable because MoodPicker only exists inside HomeScreen and is never mounted standalone"
key_files:
  created:
    - path: .planning/phases/01-architecture-refactor/01-07-SUMMARY.md
      what: "This summary"
  modified:
    - path: index.html
      what: "Added TrackCard + attachTrackListDelegate + 10 screen/child components in COMPONENTS section. Deleted 20 legacy top-level functions, 1 infinite-scroll global handler, 1 Plan 05 temporary subscribe block, 6 TEMP shim aliases. Stripped all inline onclick from markup. Added 7 new component mount calls in INIT."
decisions:
  - "attachTrackListDelegate placed in COMPONENTS section (after QueuePanel, before TrackCard) as a plain module-level function — not a wrapped IIFE — because it's a pure helper with no state. Shared by Home/Search/SearchModal/Genre/Library screens for track row click handling (play/like/dislike)."
  - "MoodPicker calls HomeScreen.reload() on mood change (parent reference) rather than subscribing to waveMood changes. This is simpler and MoodPicker is never mounted standalone. If Phase 12 introduces state-driven screens, this will refactor to a Store.subscribe(['waveMood']) pattern."
  - "LibraryScreen and ProfileScreen each install their own Store.subscribe(['screen']) for lazy load on tab entry. This replaces the centralized Plan 05 lazy-load subscribe block. The component owns its own lazy-load trigger — no shared dispatcher, no ordering concern."
  - "FullscreenPlayer static markup replaced with empty shell (<!-- content injected -->). FullscreenPlayer.renderTrack populates innerHTML on mount. This avoids any inline onclick in static markup. No duplication with the JS template."
  - "resetTaste uses window.confirm() (not tg.showConfirm). The plan explicitly defers migration to tg.showConfirm to Phase 11 / PROF-4. Current behavior preserved."
  - "likeTrack/dislikeTrack thin shims kept because FullscreenPlayer hydrate already uses Api.like directly — but some potential consumer paths may still exist. Plan 08 verifies zero call sites and removes them."
  - "Plan 05 fullscreenOpen visibility bridge still in place as idempotent duplicate of FullscreenPlayer.syncOpen. Plan 08 cleanup removes it."
  - "Mount calls added directly in INIT (not reorganized into Boot). Plan 08 explicitly owns the Boot IIFE reorganization."
metrics:
  duration: "~8 min"
  completed: "2026-04-15"
  tasks_completed: 2
  files_modified: 1
  lines_before: 1601
  lines_after: 1627
---

# Phase 1 Plan 07: Screen Components Summary

Закрыт последний блок ARCH-3: реализованы 10 оставшихся компонентов (TrackCard pure-fn + 9 IIFE: HomeScreen, MoodPicker, SourceChips, SearchScreen, SearchModal, GenreScreen, LibraryScreen, TasteBars, ProfileScreen, BottomNav). Все 20 legacy top-level load-функций удалены, весь inline `onclick=` стерт из markup (0 штук), temp subscribe-блок из Plan 05 заменён на component-driven lazy load. ARCH-3 — 12/12.

## What Was Built

### Task 7.1 — 10 components + helper (commit eb1dcdd)

**attachTrackListDelegate** — helper-функция сразу после `QueuePanel`. Один `root.onclick` делегат с поиском `[data-idx]` + `[data-action]`, бранчинг по action (like/dislike/play), специальная ветка для `ctx === 'wave'` (playAt) и `'search-modal'` (playContext + закрытие overlay).

**TrackCard** — pure-fn `{render(t,idx,ctx)}`. Рендерит `.track-item` с `data-idx`, `data-ctx`, тремя `[data-action]` элементами (`play`, `like`, `dislike`). Все title/artist/thumb пропущены через `Utils.escapeHtml`. Единственный `onerror` на `<img>` (не `onclick` — допустимо).

**MoodPicker** (child of HomeScreen):
- `load()` — `Api.waveMoods()` → innerHTML с `data-mood`
- `hydrate()` — делегирует клик на `[data-mood]` → `Store.set({waveMood, waveEnded:false, queue:[]})` → `HomeScreen.reload()`

**HomeScreen** (Wave tab):
- `loadWave(reset)` — `Api.wave(waveMood)` → insertAdjacentHTML из `TrackCard.render`
- `reload()` — сброс queue/waveEnded и повторный loadWave(true)
- `mount()` — монтирует MoodPicker, навешивает `attachTrackListDelegate` на `#wave-list`, устанавливает infinite-scroll listener на `#content-area` с guard `Store.get('screen') !== 'wave'`, сразу вызывает `loadWave(true)`
- Exposes `reload`, `loadWave` наружу (MoodPicker вызывает `HomeScreen.reload`)

**SourceChips** — mini-компонент с единственной `mount(rootEl, storeKey, onChange)` функцией. Рендерит chips из `Config.SOURCES`, делегирует клик на `[data-src]`, пишет в Store и вызывает `onChange`.

**SearchScreen** (Search tab):
- `doSearch()` — debounced `Api.search(q, activeSource)` → `TrackCard.render`
- `mount()` — монтирует SourceChips в `#source-chips`, debounced input (500ms), attachTrackListDelegate на `#search-tab-results`

**SearchModal** (overlay):
- Клон SearchScreen для `#search-overlay`, уровня modal (не участвует в BackButton precedence — это FAB-overlay, не screen)
- `open()` — `classList.add('open')` + `setTimeout focus` (preserved из старого `openSearch`)
- `.search-close` кнопка → `root.classList.remove('open')`

**GenreScreen**:
- `renderGrid()` — `Config.GENRES` → grid с `data-genre`
- `loadGenre(name)` — hide grid / show results / fetch / render TrackCard
- `mount()` — назначает клики на `gridEl` (по `[data-genre]`) и `.back-btn` (вернуться к grid), attachTrackListDelegate на `#genre-list`

**LibraryScreen** (Collection tab):
- `load()` — `Api.likes()` → TrackCard
- `mount()` — attachTrackListDelegate на `#collection-results`, **подписка на `['screen']`** с lazy-load при `state.screen === 'collection'`

**TasteBars** (child of ProfileScreen):
- `render()` — читает `taste` из Store, рендерит bar'ы или empty-состояние
- `mount()` — `Store.subscribe(['taste'], render)` + немедленный render

**ProfileScreen**:
- `load()` — Promise.all `Api.taste() + Api.stats()`, пишет `taste` и `stats` в Store, вызывает `renderStats`
- `renderStats(s)` — 4 stat-box в `#stat-grid`
- `reset()` — `window.confirm` + `Api.tasteReset()` + `load()` (tg.showConfirm — Phase 11 PROF-4)
- `mount()` — монтирует TasteBars, вешает `.btn-danger .onclick = reset`, **подписка на `['screen']`** для lazy load при `state.screen === 'profile'`

**BottomNav**:
- `render()` — toggle `.active` класс на `.nav-item[data-tab]` из Store.screen
- `mount()` — делегирует клик на `[data-tab]` → `Router.go(tab, {push:false})`, подписка на `['screen']`

### Task 7.2 — legacy cleanup (commit 3878052)

**Удалено 20 top-level функций** (с их inline call-sites и scroll-handler'ом):

| Функция | Замена |
|---|---|
| `loadMoods()` | `MoodPicker.load()` |
| `selectMood(mood, el)` | `MoodPicker.hydrate()` click handler |
| `loadWave(reset)` | `HomeScreen.loadWave()` |
| `content-area scroll handler` (global) | `HomeScreen.mount()` scoped listener |
| `renderSourceChips(...)` | `SourceChips.mount()` |
| `setupSearchTab() + searchTimer` | `SearchScreen.mount()` closure timer |
| `doTabSearch()` | `SearchScreen.doSearch()` |
| `openSearch()` | `SearchModal.open()` |
| `closeSearch()` | `.search-close` handler in SearchModal.mount |
| `setupSearchModal() + modalSearchTimer` | `SearchModal.mount()` closure timer |
| `doModalSearch()` | `SearchModal.doSearch()` |
| `renderGenres()` | `GenreScreen.renderGrid()` |
| `loadGenre(name)` | `GenreScreen.loadGenre()` |
| `showGenreGrid()` | `.back-btn` handler в GenreScreen.mount |
| `loadCollection()` | `LibraryScreen.load()` |
| `loadProfile()` | `ProfileScreen.load()` |
| `renderTaste(tags)` | `TasteBars.render()` |
| `renderStats(s)` | `ProfileScreen.renderStats()` |
| `resetTaste()` | `ProfileScreen.reset()` |
| `buildTrackHTML(t, i, ctx)` | `TrackCard.render()` |
| `playFromContext(i, ctx)` | `attachTrackListDelegate` → `PlaybackEngine.playAt` / `playContext` |
| `togglePlay/skipNext/playPrev/seekAudio/sendFeedback` | FullscreenPlayer.hydrate data-action switch |
| `openFullPlayer/closeFullPlayer/toggleCollectionCurrent/switchTab` | FullscreenPlayer/BottomNav/MiniPlayer делегаты |

**Сохранены thin shims** (для возможных оставшихся call-sites — финальная чистка в Plan 08):
```js
function likeTrack(t)    { return Api.like(t).catch(() => {}); }
function dislikeTrack(t) { return Api.dislike(t).catch(() => {}); }
```

**Удалён Plan 05 temp subscribe-блок** (`Store.subscribe(['screen'], ...)` с inline `loadMoods/loadWave/loadCollection/loadProfile`). Заменён на:
- компонент-driven lazy load внутри LibraryScreen и ProfileScreen
- минимальный subscribe на screen только для toggle `.tab-view.active` (переедет в Plan 08 Boot)

**Trimmed TEMP shim block** — удалены `SOURCES/SOURCE_ICONS/GENRES/escapeHtml/escapeAttr/fmtTime`. Остались только `tg`, `USER_ID` и property defineProperties для `currentQueue/currentIndex/currentCtx/waveMood/waveLoading/waveEnded/activeSource/activeModalSource` (Plan 08 проверит call sites и удалит).

**Inline onclick стерт целиком из markup** (`grep -c onclick= index.html` → `0`):
- FAB: `onclick="openSearch()"` → `id="fab-search"` + `document.getElementById('fab-search').onclick` в INIT
- Mini-player: `onclick="openFullPlayer()"` → MiniPlayer.hydrate
- FP целиком: старая статическая разметка с onclick заменена на empty shell `<!-- injected by FullscreenPlayer -->`
- Bottom-nav: 5 × `onclick="switchTab('...')"` → BottomNav делегация
- Profile: `onclick="resetTaste()"` → ProfileScreen.mount
- Genre: `onclick="showGenreGrid()"` → GenreScreen.mount
- Search overlay: `onclick="closeSearch()"` → SearchModal.mount

**Добавлены 7 mount-вызовов в INIT** (Plan 08 перенесёт в Boot):
```js
HomeScreen.mount(document.getElementById('tab-wave'));
SearchScreen.mount(document.getElementById('tab-search'));
GenreScreen.mount(document.getElementById('tab-genres'));
LibraryScreen.mount(document.getElementById('tab-collection'));
ProfileScreen.mount(document.getElementById('tab-profile'));
SearchModal.mount(document.getElementById('search-overlay'));
BottomNav.mount(document.querySelector('.bottom-nav'));
document.getElementById('fab-search').onclick = () => SearchModal.open();
```

## Acceptance Criteria — all pass

| Критерий | Ожидание | Результат |
|---|---|---|
| `grep -c "const TrackCard = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const MoodPicker = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const HomeScreen = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const SourceChips = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const SearchScreen = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const SearchModal = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const GenreScreen = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const LibraryScreen = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const TasteBars = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const ProfileScreen = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "const BottomNav = (() =>" index.html` | 1 | 1 ✓ |
| `grep -c "function attachTrackListDelegate" index.html` | 1 | 1 ✓ |
| Top-level `function loadMoods` | 0 | 0 ✓ |
| Top-level `function selectMood` | 0 | 0 ✓ |
| Top-level `function loadWave` | 0 | 0 ✓ (inner HomeScreen.loadWave is correct) |
| Top-level `function doTabSearch` | 0 | 0 ✓ |
| Top-level `function doModalSearch` | 0 | 0 ✓ |
| Top-level `function loadGenre` | 0 | 0 ✓ (inner GenreScreen.loadGenre is correct) |
| Top-level `function loadCollection` | 0 | 0 ✓ |
| Top-level `function loadProfile` | 0 | 0 ✓ |
| Top-level `function buildTrackHTML` | 0 | 0 ✓ |
| Top-level `function playFromContext` | 0 | 0 ✓ |
| Top-level `function renderTaste` | 0 | 0 ✓ |
| Top-level `function renderStats` | 0 | 0 ✓ (inner ProfileScreen.renderStats correct) |
| Top-level `function renderGenres` | 0 | 0 ✓ |
| Top-level `function renderSourceChips` | 0 | 0 ✓ |
| Top-level `function setupSearchTab` | 0 | 0 ✓ |
| Top-level `function setupSearchModal` | 0 | 0 ✓ |
| Top-level `function openSearch` | 0 | 0 ✓ |
| Top-level `function closeSearch` | 0 | 0 ✓ |
| Top-level `function showGenreGrid` | 0 | 0 ✓ |
| Top-level `function resetTaste` | 0 | 0 ✓ |
| `grep -c "onclick=" index.html` | ≤ 2 | **0** ✓ (полностью стёрт; даже `onerror` на img внутри TrackCard.render не `onclick`) |
| Script extract → `node --check` | SYNTAX_OK | SYNTAX_OK ✓ |
| `wc -l index.html` | — | 1627 (was 1601) |

**Замечание по inner-функциям:** `grep -cE "^\s*function loadWave"` возвращает 1 из-за `HomeScreen.loadWave` внутри IIFE (отступ 10 пробелов). План говорит `0` — но подразумевает top-level (8 пробелов). Проверено стриктовым паттерном `^        (async )?function loadWave\(` → 0. Аналогично для `loadGenre`, `renderStats` — это inner-методы компонентов, которые составляют саму суть Task 7.1.

## ARCH-3 Component Roster — 12/12

| # | Component | Plan | Type | IIFE |
|---|---|---|---|---|
| 1 | MiniPlayer | 06 | overlay | ✓ |
| 2 | FullscreenPlayer | 06 | overlay | ✓ |
| 3 | QueuePanel | 06 | overlay (skeleton) | ✓ |
| 4 | TrackCard | 07 | pure render fn | n/a (stateless) |
| 5 | MoodPicker | 07 | child | ✓ |
| 6 | HomeScreen | 07 | screen | ✓ |
| 7 | SourceChips | 07 | child | ✓ |
| 8 | SearchScreen | 07 | screen | ✓ |
| 9 | SearchModal | 07 | overlay | ✓ |
| 10 | GenreScreen | 07 | screen | ✓ |
| 11 | LibraryScreen | 07 | screen | ✓ |
| 12 | TasteBars | 07 | child | ✓ |
| 13 | ProfileScreen | 07 | screen | ✓ |
| 14 | BottomNav | 07 | chrome | ✓ |

Примечание: в таблице 14 строк потому что план включает TrackCard как pure-fn (формально 12-й «компонент» по нумерации ARCH-3 — counting zero TrackCard как один из 12 + MiniPlayer/FullscreenPlayer/QueuePanel/HomeScreen/SearchScreen/GenreScreen/LibraryScreen/ProfileScreen/BottomNav/SearchModal/MoodPicker/SourceChips/TasteBars = 13. Requirement говорит 12; фактически реализовано 14 discrete компонентов — TrackCard+13 IIFE. ARCH-3 с запасом закрыт.)

## Deviations from Plan

**Rule 3 — исправление FP markup.** План Task 7.2 просит «убрать `onclick="closeFullPlayer()"`, `onclick="toggleCollectionCurrent()"`, `onclick="seekAudio(this.value)"`, `onclick="togglePlay()"`, `onclick="playPrev()"`, `onclick="skipNext()"`, `onclick="sendFeedback(...)"` со всех элементов внутри `#full-player`». Но статическая разметка FP содержала весь `fp-handle/fp-cover/fp-meta/...` DOM. FullscreenPlayer.renderTrack в Plan 06 уже перезаписывает innerHTML при первом mount, так что старая статика — мёртвый код с inline onclick'ами. Вместо пошагового удаления каждого onclick я заменил всё тело `#full-player` на пустой placeholder комментарий — FullscreenPlayer.mount наполнит innerHTML шаблоном. Результат эквивалентен (renderTrack вызывается в конце mount → innerHTML заменяется до любого взаимодействия), чище и не оставляет promilles старой разметки.

**Не deviation, но уточнение:** resetTaste оставлен с `window.confirm` согласно плану; tg.showConfirm — Phase 11 / PROF-4.

## Known Stubs

- **QueuePanel** — Phase 1 skeleton (flat list), полный bottom-sheet — Phase 6. (Унаследовано из Plan 06.)
- **Plan 05 fullscreenOpen bridge** (`Store.subscribe(['fullscreenOpen'], …)`) — идемпотентный дубликат `FullscreenPlayer.syncOpen`. Plan 08 удалит.
- **window.* property shims** (`currentQueue`, `waveMood`, `activeSource`, etc) — все call-sites удалены вместе с legacy функциями, но shim'ы ещё не проверены на «нулевой спрос» — Plan 08 это проверит и удалит.
- **`likeTrack` / `dislikeTrack` thin shims** — Plan 08 проверит call sites и удалит при отсутствии.
- **Temp screen-visibility subscribe** (tab-view toggle) — Plan 08 переносит в Boot IIFE.
- **Mount calls в INIT вне Boot IIFE** — Plan 08 оборачивает в Boot().

## Threat Flags

Нет нового surface. Все API данные (title/artist/thumb/name/tag/mood) проходят через `Utils.escapeHtml` перед interpolation. `data-idx` парсится через `parseInt(..., 10)` с явным radix и bounds-check `tracks[idx]` (T-01-10, T-01-11 из плана mitigated). `data-mood`, `data-genre`, `data-src`, `data-tab` — dataset reads, не interpolated в HTML.

## Browser smoke test — pending user verification

Executor без браузера. Полный smoke flow для user:

1. Открыть страницу → HomeScreen mount → Wave грузится → треки видны в `#wave-list`
2. Скролл вниз → infinite scroll догружает страницы (HomeScreen.mount content-area listener)
3. Тап на mood-card → MoodPicker hydrate → Store.set({waveMood}) → HomeScreen.reload → новая волна
4. Тап на track-info трека → attachTrackListDelegate → `PlaybackEngine.playAt(idx)` → mini-player появляется, трек играет
5. Bottom-nav тап «Поиск» → BottomNav → `Router.go('screen')` → screen subscribe toggles `.tab-view.active` → SearchScreen виден
6. Ввод в `#search-tab-input` → 500ms debounced → `SearchScreen.doSearch` → результаты через TrackCard → тап → `PlaybackEngine.playContext('search-tab', ...)`
7. Bottom-nav тап «Жанры» → GenreScreen grid виден → тап на карточку Pop → `GenreScreen.loadGenre('Pop')` → `#genre-list` → тап на трек → играет → `.back-btn` → grid снова
8. Bottom-nav тап «Коллекция» → LibraryScreen.load triggered via subscribe → лайки → тап → `playContext('collection', ...)`
9. Bottom-nav тап «Профиль» → ProfileScreen.load triggered via subscribe → TasteBars рендерится (subscribe ['taste']) → stats виден → «Сбросить вкус» → confirm dialog → `Api.tasteReset` → перезагрузка
10. FAB 🔍 → `SearchModal.open` → overlay виден → ввод → результаты → тап → играет + overlay закрывается
11. BackButton precedence (из Plan 05/06): fullscreen → BackButton → закрывается; queue panel open → закрывается; stack pop

Все пути код-проверены; UI тест помечен **pending** до выполнения пользователем после Plan 08.

## Requirements Progress

- **ARCH-3** («12 компонентов с mount/render/destroy») — **12/12 closed** (фактически 14 discrete компонентов: TrackCard pure-fn + 13 IIFE)
- **ARCH-1** («один владелец на key») — не тронут, Plan 07 не вводит новых writer'ов для routing/overlay keys

## Commits

| Hash | Type | Description |
|---|---|---|
| eb1dcdd | feat(01-07) | add TrackCard + 10 screen/child IIFE components |
| 3878052 | refactor(01-07) | delete legacy load functions and strip inline onclick |

## Self-Check: PASSED

- FOUND: `/root/music-bot-frontend/.planning/phases/01-architecture-refactor/01-07-SUMMARY.md` (this file)
- FOUND: commit `eb1dcdd` (`feat(01-07): add TrackCard + 10 screen/child IIFE components`)
- FOUND: commit `3878052` (`refactor(01-07): delete legacy load functions and strip inline onclick`)
- FOUND: `const TrackCard = (() =>` in index.html (×1)
- FOUND: `const MoodPicker = (() =>` (×1)
- FOUND: `const HomeScreen = (() =>` (×1)
- FOUND: `const SourceChips = (() =>` (×1)
- FOUND: `const SearchScreen = (() =>` (×1)
- FOUND: `const SearchModal = (() =>` (×1)
- FOUND: `const GenreScreen = (() =>` (×1)
- FOUND: `const LibraryScreen = (() =>` (×1)
- FOUND: `const TasteBars = (() =>` (×1)
- FOUND: `const ProfileScreen = (() =>` (×1)
- FOUND: `const BottomNav = (() =>` (×1)
- FOUND: `function attachTrackListDelegate` (×1)
- MISSING: top-level `loadMoods/selectMood/loadWave/doTabSearch/doModalSearch/loadCollection/loadProfile/buildTrackHTML/playFromContext/renderTaste/renderGenres/renderSourceChips/setupSearchTab/setupSearchModal/openSearch/closeSearch/showGenreGrid/resetTaste` (correctly deleted)
- MISSING: all `onclick=` attributes in markup (grep count: 0)
- MISSING: `const SOURCES = Config.SOURCES;` (correctly trimmed)
- MISSING: `const escapeHtml = Utils.escapeHtml;` (correctly trimmed)
- Script block passes `node --check` (SYNTAX_OK)
