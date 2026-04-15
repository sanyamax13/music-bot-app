---
phase: 01-architecture-refactor
plan: 02
type: execute
wave: 2
depends_on: ["01-01"]
files_modified: [index.html]
autonomous: true
requirements: [ARCH-1]
must_haves:
  truths:
    - "Существует ровно один объект `Store` с методами `get`, `set`, `subscribe`"
    - "Все ранее топ-уровневые мутабельные переменные состояния (currentQueue, currentIndex, currentCtx, waveMood, waveLoading, waveEnded, activeAudio, nextAudio, currentTrackStart, lastFeedbackSent, activeSource, activeModalSource) переехали в `state` blob внутри Store"
    - "Каждый ключ в state blob помечен комментарием `// owner: <module>`"
    - "Текущий функционал Wave/Search/Genre/Collection/Profile продолжает работать"
  artifacts:
    - path: "index.html"
      provides: "Store IIFE с pub/sub и blob state"
      contains: "const Store = (() =>"
  key_links:
    - from: "playTrack (legacy)"
      to: "Store.set"
      via: "compatibility shim"
      pattern: "Store\\.set\\("
---

<objective>
Внедрить `Store` IIFE по спецификации ARCHITECTURE §2: единый `state` blob, key-scoped `subscribe`, документированный владелец на каждый ключ. На этой стадии Store **не** становится единственным владельцем — он сосуществует со старыми глобальными переменными через legacy-shim. Полная миграция владения произойдёт в Plans 04–07. Цель этого плана — создать инфраструктуру и blob.

Purpose: ARCH-1 требование «один Store с pub/sub и одним владельцем на ключ». Без Store невозможно сделать PlaybackEngine (Plan 04) и компоненты (Plans 06–07).

Output: Работающий `Store` IIFE с blob-состоянием, документированным владельцем каждого ключа, и стартовыми shim-функциями для постепенной миграции.
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
@.planning/phases/01-architecture-refactor/01-01-SUMMARY.md
</context>

<interfaces>
Целевой Store API (verbatim из ARCHITECTURE.md §2):

```js
const Store = (() => {
  const state = {
    // ===== Playback (owner: PlaybackEngine — Plan 04) =====
    currentTrack: null,
    isPlaying: false,
    duration: 0,
    playbackCtx: 'wave',           // 'wave' | 'genre' | 'search-tab' | 'search-modal' | 'collection'

    // ===== Queue (owner: PlaybackEngine — Plan 04) =====
    queue: [],
    queueIndex: -1,
    userQueue: [],                 // explicit "play next" — wired in Phase 6
    history: [],                   // for prev — wired in Phase 6

    // ===== Wave-specific (owner: HomeScreen — Plan 07; read by PlaybackEngine for auto-extend) =====
    waveMood: '',
    waveLoading: false,
    waveEnded: false,

    // ===== UI (owner: Router — Plan 05) =====
    screen: 'wave',                // 'wave' | 'search' | 'genres' | 'collection' | 'profile'
    fullscreenOpen: false,
    queuePanelOpen: false,         // Phase 6 will use this

    // ===== Search (owner: SearchScreen — Plan 07) =====
    activeSource: 'all',
    activeModalSource: 'all',
    searchTabResults: [],
    searchModalResults: [],

    // ===== Genre (owner: HomeScreen/GenreScreen — Plan 07) =====
    genreResults: [],

    // ===== Collection (owner: LibraryScreen — Plan 07) =====
    collectionResults: [],

    // ===== Per-user (owner: Boot/Api — Plan 08) =====
    userId: 0,
    likedIds: new Set(),
    taste: null,
    stats: null,
  };

  const subs = new Map();          // key -> Set<fn>

  function get(key) { return state[key]; }

  function set(patch) {
    const changedKeys = [];
    for (const k of Object.keys(patch)) {
      if (state[k] !== patch[k]) {
        state[k] = patch[k];
        changedKeys.push(k);
      }
    }
    if (changedKeys.length) notify(changedKeys);
  }

  function subscribe(keys, fn) {
    for (const k of keys) {
      if (!subs.has(k)) subs.set(k, new Set());
      subs.get(k).add(fn);
    }
    // return an unsubscribe function
    return () => { for (const k of keys) subs.get(k)?.delete(fn); };
  }

  function notify(changedKeys) {
    const called = new Set();
    for (const k of changedKeys) {
      const set = subs.get(k);
      if (!set) continue;
      for (const fn of set) {
        if (called.has(fn)) continue;
        called.add(fn);
        try { fn(state); } catch (e) { console.error('[Store] subscriber error', e); }
      }
    }
  }

  return { get, set, subscribe, _state: state };  // _state exposed only for debugging
})();
```

Текущие топ-уровневые переменные в `index.html`, которые этот план переносит в state blob:

```js
// строки 509–513 (после Plan 01)
let activeAudio = audioA, nextAudio = audioB;       // → НЕ в Store, остаётся в PlaybackEngine (Plan 04)
let currentQueue = [], currentIndex = -1, currentCtx = 'wave';   // → state.queue, state.queueIndex, state.playbackCtx
let waveMood = '', waveLoading = false, waveEnded = false;       // → state.waveMood, state.waveLoading, state.waveEnded
let currentTrackStart = 0, lastFeedbackSent = false;             // → НЕ в Store (внутреннее состояние PlaybackEngine, Plan 04)
let activeSource = 'all';                                        // → state.activeSource
let activeModalSource = 'all';                                   // → state.activeModalSource
```
</interfaces>

<tasks>

<task type="auto">
  <name>Task 2.1: Создать Store IIFE с blob и pub/sub</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (особенно секция `// ===== STORE =====` после Plan 01 и текущие top-level let'ы вокруг строки 509–513)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§2 «State management» — vebatim эталон Store API)
  </read_first>
  <action>
В секцию `// ===== STORE =====` вставить ровно тот код, что приведён в `<interfaces>` выше. Никаких сокращений, всех ключей оставить с комментариями `owner:`. Это эталон, на который опираются все следующие плеи.
  </action>
  <verify>
    <automated>grep -c "const Store = (() =>" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "const Store = (() =>" index.html` → `1`
    - `grep -c "function subscribe(keys, fn)" index.html` → `1`
    - `grep -c "owner: PlaybackEngine" index.html` ≥ `1`
    - `grep -c "owner: Router" index.html` ≥ `1`
    - `grep -c "owner: HomeScreen" index.html` ≥ `1`
    - В DevTools console: `Store.get('queue')` возвращает `[]`, `Store.get('queueIndex')` возвращает `-1`
  </acceptance_criteria>
  <done>Store IIFE присутствует, доступен в console, blob содержит все перечисленные ключи с комментариями владельцев.</done>
</task>

<task type="auto">
  <name>Task 2.2: Подключить legacy-переменные к Store через геттеры/сеттеры (не ломая поведение)</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (текущие top-level let'ы и все их read/write-сайты: `playFromContext`, `playTrack`, `loadWave`, `selectMood`, `doTabSearch`, `doModalSearch`, `loadGenre`, `loadCollection`, `autoExtendWave`)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§2 — правило «exactly one module writes each key»)
  </read_first>
  <action>
**Цель:** старый код (`currentQueue.length`, `currentQueue = tracks`, `waveMood`, `currentIndex++` и т.п.) должен продолжать работать БЕЗ переписывания call sites, но фактическим хранилищем стал `Store._state`.

Сразу под Store IIFE добавить блок legacy-аксессоров:

```js
// === LEGACY shims for Plans 02→04 — these will be removed in Plan 04 (PlaybackEngine takeover) ===
// Each shim is a getter/setter pair on a property of an anonymous object that bridges
// the old `let varName` style with Store. Old code keeps reading/writing `currentQueue` etc.
// and the values flow through Store under the hood.
Object.defineProperty(window, 'currentQueue', {
  get: () => Store.get('queue'),
  set: (v) => Store.set({ queue: v }),
  configurable: true,
});
Object.defineProperty(window, 'currentIndex', {
  get: () => Store.get('queueIndex'),
  set: (v) => Store.set({ queueIndex: v }),
  configurable: true,
});
Object.defineProperty(window, 'currentCtx', {
  get: () => Store.get('playbackCtx'),
  set: (v) => Store.set({ playbackCtx: v }),
  configurable: true,
});
Object.defineProperty(window, 'waveMood', {
  get: () => Store.get('waveMood'),
  set: (v) => Store.set({ waveMood: v }),
  configurable: true,
});
Object.defineProperty(window, 'waveLoading', {
  get: () => Store.get('waveLoading'),
  set: (v) => Store.set({ waveLoading: v }),
  configurable: true,
});
Object.defineProperty(window, 'waveEnded', {
  get: () => Store.get('waveEnded'),
  set: (v) => Store.set({ waveEnded: v }),
  configurable: true,
});
Object.defineProperty(window, 'activeSource', {
  get: () => Store.get('activeSource'),
  set: (v) => Store.set({ activeSource: v }),
  configurable: true,
});
Object.defineProperty(window, 'activeModalSource', {
  get: () => Store.get('activeModalSource'),
  set: (v) => Store.set({ activeModalSource: v }),
  configurable: true,
});
```

Затем **удалить** соответствующие топ-уровневые объявления из index.html:
- `let currentQueue = [], currentIndex = -1, currentCtx = 'wave';`
- `let waveMood = '', waveLoading = false, waveEnded = false;`
- `let activeSource = 'all';`
- `let activeModalSource = 'all';`

ОСТАВИТЬ:
- `let activeAudio = audioA, nextAudio = audioB;` (не state, а внутреннее dual-buffer; уйдёт в PlaybackEngine в Plan 04)
- `let currentTrackStart = 0, lastFeedbackSent = false;` (внутреннее состояние feedback flow, уйдёт в Plan 04)

ВАЖНО: `window.searchTabQueue`, `window.searchModalQueue`, `window.genreQueue`, `window.collectionQueue` (текущие ad-hoc глобалы из строк 665, 705, 732, 752) ОСТАВИТЬ как есть в этом плане — их рефакторим в Plan 03 одновременно с Api.
  </action>
  <verify>
    <automated>grep -cE "^\s*let currentQueue" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -cE "^\s*let currentQueue" index.html` → `0`
    - `grep -cE "^\s*let waveMood" index.html` → `0`
    - `grep -cE "^\s*let activeSource" index.html` → `0`
    - `grep -c "Object.defineProperty(window, 'currentQueue'" index.html` → `1`
    - `grep -c "Object.defineProperty(window, 'waveMood'" index.html` → `1`
    - Browser smoke: открыть index.html, тапнуть Wave → треки загружаются (значит `currentQueue = tracks` через shim работает), тапнуть на трек → плеер запускается, переключиться на Поиск → ввести «pop» → результаты видны, лайк трека → POST в Network log; в DevTools console `Store.get('queue').length > 0`.
    - `grep -c "let activeAudio" index.html` → `1` (всё ещё есть, уйдёт в Plan 04)
  </acceptance_criteria>
  <done>Старый код не трогает топ-уровневых state-переменных напрямую — он проходит через Store благодаря property-аксессорам. Wave/Search/Genre/Collection/Profile работают идентично pre-refactor.</done>
</task>

</tasks>

<threat_model>
| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-03 | Tampering | Store global | accept | Single-file vanilla — нет границы доверия между модулями внутри одного скрипта |
</threat_model>

<verification>
- Wave / Search / Genre / Collection / Profile — все 5 табов работают как до рефактора
- В DevTools console: `Store.get('queue')`, `Store.get('queueIndex')`, `Store.get('waveMood')` отдают актуальные значения во время использования
- `wc -l index.html` ≤ 2500
</verification>

<success_criteria>
- ARCH-1 половина закрыта: Store существует, есть subscribe API, blob документирован
- Полное «один владелец на ключ» докручивается в Plans 04–07
</success_criteria>

<output>
После завершения создать `.planning/phases/01-architecture-refactor/01-02-SUMMARY.md` с перечнем мигрированных в Store ключей и оставшихся legacy-let'ов (activeAudio, nextAudio, currentTrackStart, lastFeedbackSent).
</output>
