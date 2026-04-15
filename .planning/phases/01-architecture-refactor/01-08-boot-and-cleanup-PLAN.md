---
phase: 01-architecture-refactor
plan: 08
type: execute
wave: 8
depends_on: ["01-07"]
files_modified: [index.html]
autonomous: false
requirements: [ARCH-1, ARCH-3, ARCH-4, ARCH-5, ARCH-6]
must_haves:
  truths:
    - "Существует один Boot() entry-point, вызывающий все необходимые mount'ы в правильном порядке"
    - "Все temporary shim'ы из Plans 01–07 удалены (currentQueue / currentIndex / waveMood / activeSource / fmtTime / escapeHtml / SOURCES / GENRES / tg / USER_ID / openFullPlayer / closeFullPlayer / likeTrack / dislikeTrack / sendFeedback / playFromContext / togglePlay / skipNext / playPrev / seekAudio / toggleCollectionCurrent / switchTab)"
    - "Полный регрессионный smoke-тест пройден: Wave plays, Search works, Genre works, Library works, Profile works, Like/Dislike POST, Skip 10 in a row, lock-screen controls"
    - "Файл `index.html` ≤ 2500 строк"
    - "Все window.* state-аксессоры удалены"
  artifacts:
    - path: "index.html"
      provides: "Boot entry + чистый файл без legacy shim'ов"
      contains: "function Boot()"
  key_links:
    - from: "Boot()"
      to: "All component mount methods"
      via: "sequential mount calls in dependency order"
      pattern: "Boot\\(\\)"
---

<objective>
Финальный план фазы. Создать единый `Boot()` entry-point, который вызывает все mount'ы в правильном порядке. Удалить все temporary shim'ы, накопленные в Plans 01–07. Прогнать полный регрессионный smoke-тест и checkpoint у пользователя.

Purpose: Закрыть ARCH-1 (один writer per key — final audit), ARCH-3 (все mount'ы вызваны), ARCH-4 (Router инициализирован), ARCH-5 (один файл, никаких сборок), ARCH-6 (≤2500 строк, секции видны в outline).

Output: Чистый `index.html` без legacy shim'ов, с явным `Boot()` entry-point и пройденным полным регрессионным тестом.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/REQUIREMENTS.md
@.planning/research/ARCHITECTURE.md
@index.html
@.planning/phases/01-architecture-refactor/01-07-SUMMARY.md
</context>

<tasks>

<task type="auto">
  <name>Task 8.1: Создать Boot() entry-point и удалить temporary shim'ы</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (целиком — это последний план; нужно знать, где живут все shim'ы из Plans 01/02/04/07)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§1 «Module list» — порядок секций; §7 «Build order» — Boot последний)
  </read_first>
  <action>
1. **В секцию `// ===== BOOT =====`** вставить:

```js
function Boot() {
  // Mount overlays (always present; visible/hidden via Store flags)
  MiniPlayer.mount(document.getElementById('mini-player'));
  FullscreenPlayer.mount(document.getElementById('full-player'));
  QueuePanel.mount(document.getElementById('queue-panel'));

  // Mount screens (all 5 mounted once; visibility toggled by Router subscribe)
  HomeScreen.mount(document.getElementById('tab-wave'));
  SearchScreen.mount(document.getElementById('tab-search'));
  GenreScreen.mount(document.getElementById('tab-genres'));
  LibraryScreen.mount(document.getElementById('tab-collection'));
  ProfileScreen.mount(document.getElementById('tab-profile'));

  // Mount nav
  BottomNav.mount(document.querySelector('.bottom-nav'));

  // Mount search modal
  SearchModal.mount(document.getElementById('search-overlay'));

  // FAB → open search modal
  const fab = document.querySelector('.fab');
  if (fab) fab.onclick = () => SearchModal.open();

  // Tab visibility sync (until Plan 12 turns this into proper component mount/unmount)
  Store.subscribe(['screen'], (state) => {
    document.querySelectorAll('.tab-view').forEach(v => v.classList.remove('active'));
    const tabEl = document.getElementById('tab-' + state.screen);
    if (tabEl) tabEl.classList.add('active');
  });

  // Initial screen
  Router.go('wave', { push: false });
}

Boot();
```

2. **Удалить старый init-блок** в конце скрипта (bывшие строки 1015–1020):
```js
// renderGenres();  ← удалено в Plan 07
// setupSearchTab(); ← удалено в Plan 07
// setupSearchModal(); ← удалено в Plan 07
// loadMoods(); ← HomeScreen.mount() это делает
// loadWave(true); ← HomeScreen.mount() это делает
```
В этом месте теперь только пусто (или комментарий `// init handled by Boot()`).

3. **Удалить TEMP shims блок** (созданный в Plan 01 / Plan 02 / Plan 03):
   - `const tg = Tg.tg;` — найти все call sites `tg.` (если они не `Tg.tg.`) и заменить на `Tg.tg.`. После замены удалить shim.
   - `const USER_ID = Tg.USER_ID;` — найти все `USER_ID` и заменить на `Tg.USER_ID`. Удалить shim.
   - `Object.defineProperty(window, 'currentQueue', ...)` — найти все call sites `currentQueue` (вне Store IIFE) и заменить на `Store.get('queue')` / `Store.set({ queue: ... })`. Удалить defineProperty.
   - Аналогично для `currentIndex` → `Store.get('queueIndex')`, `currentCtx` → `Store.get('playbackCtx')`, `waveMood` → `Store.get('waveMood')`, `waveLoading` → `Store.get('waveLoading')`, `waveEnded` → `Store.get('waveEnded')`, `activeSource` → `Store.get('activeSource')`, `activeModalSource` → `Store.get('activeModalSource')`.
   
   ВАЖНО: после удаления Plan 07 эти legacy идентификаторы уже почти не используются (все load-функции удалены). Если какие-то остались — надо переписать.

4. **Удалить top-level functions, которые остались после Plan 07** как тонкие делегаты:
   - `function togglePlay()` — больше нет call sites (FullscreenPlayer.hydrate handler сам зовёт PlaybackEngine.toggle); удалить.
   - `function skipNext()`, `function playPrev()`, `function seekAudio()` — аналогично, удалить.
   - `function sendFeedback()` — удалить (FullscreenPlayer.hydrate сам зовёт PlaybackEngine.sendManualFeedback).
   - `function toggleCollectionCurrent()` — удалить (FullscreenPlayer.hydrate handler `case 'like'` сам делает `Api.like(currentTrack)`).
   - `function switchTab()` — удалить (BottomNav.mount сам зовёт Router.go).
   - `function openFullPlayer()`, `function closeFullPlayer()` — удалить (MiniPlayer и FullscreenPlayer.hydrate сами пишут в Store).
   - `function likeTrack()`, `function dislikeTrack()` — удалить, заменив все call sites на `Api.like(t).catch(() => {})` / `Api.dislike(t).catch(() => {})`. (В FullscreenPlayer.hydrate `case 'like'` — заменить `likeTrack(t)` на `Api.like(t).catch(() => {})`.)

5. После всех удалений — проверить `wc -l index.html` ≤ 2500.
  </action>
  <verify>
    <automated>grep -c "function Boot()" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "function Boot()" index.html` → `1`
    - `grep -cE "^\s*Boot\(\);" index.html` → `1`
    - **All shims gone:**
      - `grep -c "Object.defineProperty(window, 'currentQueue'" index.html` → `0`
      - `grep -c "Object.defineProperty(window, 'waveMood'" index.html` → `0`
      - `grep -cE "^\s*const tg = Tg\.tg" index.html` → `0`
      - `grep -cE "^\s*const USER_ID = Tg\.USER_ID" index.html` → `0`
      - `grep -c "TEMP shims" index.html` → `0`
    - **All legacy delegate-functions gone:**
      - `grep -cE "^\s*function togglePlay\(" index.html` → `0`
      - `grep -cE "^\s*function skipNext\(" index.html` → `0`
      - `grep -cE "^\s*function playPrev\(" index.html` → `0`
      - `grep -cE "^\s*function seekAudio\(" index.html` → `0`
      - `grep -cE "^\s*function switchTab\(" index.html` → `0`
      - `grep -cE "^\s*function openFullPlayer\(" index.html` → `0`
      - `grep -cE "^\s*function closeFullPlayer\(" index.html` → `0`
      - `grep -cE "^\s*function likeTrack\(" index.html` → `0`
      - `grep -cE "^\s*function dislikeTrack\(" index.html` → `0`
      - `grep -cE "^\s*function sendFeedback\(" index.html` → `0`
      - `grep -cE "^\s*function toggleCollectionCurrent\(" index.html` → `0`
    - **One-writer audit (final):**
      - `grep -n "Store.set" index.html | grep -v "// " | wc -l` ≤ 30 (sanity check)
      - Каждый ключ blob проверить визуально через `grep -n "Store.set({ <key>"`:
        - `currentTrack`, `isPlaying`, `duration`, `playbackCtx`, `queue`, `queueIndex` → только PlaybackEngine
        - `waveMood`, `waveLoading`, `waveEnded` → только HomeScreen / MoodPicker (HomeScreen children)
        - `screen` → только Router
        - `fullscreenOpen`, `queuePanelOpen` → только Router (через подписку), MiniPlayer (открытие через таб) и FullscreenPlayer (закрытие)
        - `activeSource` → только SearchScreen / SourceChips (children of SearchScreen)
        - `activeModalSource` → только SearchModal / SourceChips
        - `searchTabResults` → только SearchScreen
        - `searchModalResults` → только SearchModal
        - `genreResults` → только GenreScreen
        - `collectionResults` → только LibraryScreen
        - `taste`, `stats` → только ProfileScreen
    - `wc -l index.html` ≤ 2500
    - `grep -cE "^\s*window\." index.html` → `0` (никаких state-bearing window.* присваиваний; допустимо `window.Telegram.WebApp` в Tg IIFE для feature detection)
    - `grep -c "window.Telegram.WebApp" index.html` ≤ `1` (только внутри Tg IIFE)
  </acceptance_criteria>
  <done>Boot() — единственная entry-point; все shim'ы и legacy функции удалены; one-writer audit пройден; файл ≤ 2500 строк.</done>
</task>

<task type="checkpoint:human-verify" gate="blocking">
  <name>Task 8.2: Полный регрессионный smoke-тест (human verify)</name>
  <what-built>
    Phase 1 рефактор завершён: модульная архитектура (Config / Utils / Tg / Store / Api / PlaybackEngine / Feedback / Router / 12 компонентов / Boot), один владелец на каждый ключ Store, BackButton precedence, файл ≤ 2500 строк. Все pre-refactor функции работают.
  </what-built>
  <how-to-verify>
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
    1. `wc -l index.html` ≤ 2500
    2. Открыть в редакторе → видны все 11 секций-комментариев `// ===== ... =====` в outline в правильном порядке
    3. `grep -c "Store.set" index.html` визуально показывает что каждый ключ пишется одним владельцем
  </how-to-verify>
  <resume-signal>Type "approved" если всё работает, или опиши какой пункт сломался (укажи номер и наблюдаемое поведение)</resume-signal>
</task>

</tasks>

<threat_model>
| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-12 | Tampering | Boot() race condition | accept | Boot вызывается синхронно после определения всех модулей; DOM готов (скрипт в конце body) |
| T-01-13 | Spoofing | Tg.USER_ID | accept | Известный trade-off из PROJECT.md; не меняется в Phase 1; HMAC — отдельный milestone |
</threat_model>

<verification>
- Полный smoke-тест Task 8.2 пройден
- One-writer audit Task 8.1 пройден
- `wc -l index.html` ≤ 2500
</verification>

<success_criteria>
- ARCH-1 закрыт: Store + один владелец на ключ
- ARCH-2 закрыт: PlaybackEngine — единственный владелец audio
- ARCH-3 закрыт: 12 компонентов с mount/render/destroy
- ARCH-4 закрыт: Router + BackButton precedence
- ARCH-5 закрыт: один index.html, CDN-only, no build
- ARCH-6 закрыт: ≤ 2500 строк, секционная разбивка видна
- Полный регрессионный smoke-тест пройден человеком
</success_criteria>

<output>
После завершения создать `.planning/phases/01-architecture-refactor/01-08-SUMMARY.md` со списком:
- Финальный wc -l index.html
- Owner per key audit (key → owner module)
- Что временно осталось как «делегат для inline onclick» (если что-то осталось)
- Подтверждение что smoke-test пройден
- Список потенциальных Phase 2 hot-spots, где Phase 1 архитектура **не** препятствует pitfall fixes (iOS unlock, named handlers, setPositionState)
</output>
