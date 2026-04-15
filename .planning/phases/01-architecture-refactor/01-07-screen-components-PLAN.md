---
phase: 01-architecture-refactor
plan: 07
type: execute
wave: 7
depends_on: ["01-06"]
files_modified: [index.html]
autonomous: true
requirements: [ARCH-3]
must_haves:
  truths:
    - "9 экранных/детских компонентов реализованы как IIFE с mount/render/destroy: HomeScreen (Wave), SearchScreen, GenreScreen, LibraryScreen (collection), ProfileScreen, MoodPicker, SourceChips, TasteBars, BottomNav"
    - "TrackCard — pure render-функция (без mount/destroy, потому что это не stateful компонент)"
    - "Все load-функции (loadWave, doTabSearch, doModalSearch, loadGenre, loadCollection, loadProfile) живут внутри соответствующих компонентов"
    - "Все 5 экранов работают идентично pre-refactor"
  artifacts:
    - path: "index.html"
      provides: "HomeScreen + SearchScreen + GenreScreen + LibraryScreen + ProfileScreen + MoodPicker + SourceChips + TasteBars + BottomNav + TrackCard pure-fn"
      contains: "const HomeScreen = (() =>"
  key_links:
    - from: "Router.go (subscribe screen)"
      to: "Screen components mount/render"
      via: "screen-driven mount selection"
      pattern: "case 'wave':"
---

<objective>
Завершить ARCH-3 — реализовать оставшиеся 9 компонентов (5 экранов + 4 child-компонента) как IIFE с `mount/render/destroy`, перенеся в них существующую логику `loadWave`, `doTabSearch`, `doModalSearch`, `loadGenre`, `loadCollection`, `loadProfile`, `renderTaste`, `renderStats`, `renderSourceChips`, `renderGenres`, `loadMoods`, `selectMood`, `buildTrackHTML`, `playFromContext`. Удалить все inline `onclick=` из markup'а и заменить на event delegation.

Purpose: ARCH-3 закрыт целиком (12 компонентов).

Output: Все 5 табов рендерятся компонентами; markup в `<body>` сводится к skeleton-roots для компонентов; временная subscribe-логика в Plan 05 (`Store.subscribe(['screen'], ...)` с inline lazy-load'ами) заменена на компонент-driven mount/destroy.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/REQUIREMENTS.md
@.planning/research/ARCHITECTURE.md
@index.html
@.planning/phases/01-architecture-refactor/01-06-SUMMARY.md
</context>

<interfaces>
**Целевая структура:** screen components маунтятся в `#tab-wave`, `#tab-search`, `#tab-genres`, `#tab-collection`, `#tab-profile` — эти div'ы уже существуют в markup (строки 386–434 pre-refactor). Каждый компонент `mount`-ится **один раз** при загрузке (lazy mount необязателен для Phase 1; полный mount/unmount на screen change — оптимизация для Phase 12 «states» / Phase 13). Toggle видимости остаётся за `Router`-подпиской из Plan 05.

**TrackCard** — pure render-функция (не stateful):

```js
const TrackCard = (() => {
  function render(t, idx, ctx) {
    const src = (t.source || '').toLowerCase();
    const srcIcon = Config.SOURCE_ICONS[src] || '';
    return `
      <div class="track-item" data-idx="${idx}" data-ctx="${ctx}">
        <img src="${Utils.escapeHtml(t.thumb || '')}" class="track-thumb" onerror="this.style.opacity=0.3">
        <div class="track-info" data-action="play">
          <div class="track-title">${srcIcon ? `<span class="source-icon">${srcIcon}</span>` : ''}${Utils.escapeHtml(t.title || '')}</div>
          <div class="track-artist">${Utils.escapeHtml(t.artist || '')}</div>
        </div>
        <div class="track-actions">
          <div class="icon-btn" data-action="like">❤️</div>
          <div class="icon-btn" data-action="dislike">👎</div>
        </div>
      </div>
    `;
  }
  return { render };
})();
```

Делегирование событий — на уровне списка-родителя: каждый screen component навешивает один `click` listener на свой root, читает `e.target.closest('[data-action]')` + `e.target.closest('[data-idx]')`, и вызывает либо `PlaybackEngine.playContext(ctx, ...)`, либо `Api.like/dislike`.

**Helper для делегирования** (общий для HomeScreen / SearchScreen / GenreScreen / LibraryScreen):

```js
function attachTrackListDelegate(rootEl, getTracks, ctx) {
  rootEl.onclick = (e) => {
    const row = e.target.closest('[data-idx]');
    if (!row) return;
    const idx = parseInt(row.dataset.idx, 10);
    const tracks = getTracks();
    const t = tracks[idx];
    if (!t) return;
    const action = e.target.closest('[data-action]')?.dataset.action;
    if (action === 'like')    { Api.like(t).catch(() => {}); return; }
    if (action === 'dislike') { Api.dislike(t).catch(() => {}); return; }
    if (action === 'play') {
      if (ctx === 'wave') {
        PlaybackEngine.playAt(idx);
      } else if (ctx === 'search-modal') {
        PlaybackEngine.playContext(ctx, tracks, idx);
        Store.set({ /* SearchModal close handled separately */ });
      } else {
        PlaybackEngine.playContext(ctx, tracks, idx);
      }
    }
  };
}
```
</interfaces>

<tasks>

<task type="auto">
  <name>Task 7.1: Реализовать TrackCard + screen-компоненты Home/Search/Genre/Library/Profile</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (все load-функции и render-функции в скрипте; markup секции `#tab-wave`, `#tab-search`, `#tab-genres`, `#tab-collection`, `#tab-profile`)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§4 — компонент skeleton)
  </read_first>
  <action>
В секцию `// ===== COMPONENTS =====` (после QueuePanel из Plan 06) добавить **по порядку**:

1. **TrackCard** — pure render fn, ровно как в `<interfaces>` выше.

2. **MoodPicker** (child of HomeScreen):
```js
const MoodPicker = (() => {
  let root, unsub;
  async function load() {
    try {
      const d = await Api.waveMoods();
      const moods = d.moods || [];
      root.innerHTML = `<div class="mood-card${Store.get('waveMood') === '' ? ' active' : ''}" data-mood="">✨ Все</div>` +
        moods.map(m => `<div class="mood-card${Store.get('waveMood') === (m.id || m.name) ? ' active' : ''}" data-mood="${Utils.escapeHtml(m.id || m.name)}"><span class="mood-emoji">${m.emoji || ''}</span>${Utils.escapeHtml(m.name)}</div>`).join('');
    } catch (e) { root.innerHTML = ''; }
  }
  function hydrate() {
    root.onclick = (e) => {
      const card = e.target.closest('[data-mood]');
      if (!card) return;
      Store.set({ waveMood: card.dataset.mood, waveEnded: false, queue: [] });
      root.querySelectorAll('.mood-card').forEach(c => c.classList.remove('active'));
      card.classList.add('active');
      // re-trigger HomeScreen wave reload via store change OR call HomeScreen.reload()
      HomeScreen.reload();
    };
  }
  function mount(el) { root = el; load().then(hydrate); }
  function destroy() { if (root) root.innerHTML = ''; }
  return { mount, destroy };
})();
```

3. **HomeScreen** (Wave-таб):
```js
const HomeScreen = (() => {
  let root, listEl, loaderEl;

  function ensureMarkup() {
    // Reuse existing children: #mood-scroll, #wave-list, #wave-loader
    listEl   = root.querySelector('#wave-list');
    loaderEl = root.querySelector('#wave-loader');
  }

  async function loadWave(reset) {
    if (Store.get('waveLoading') || Store.get('waveEnded')) return;
    Store.set({ waveLoading: true });
    if (loaderEl) loaderEl.style.display = 'block';
    try {
      const d = await Api.wave(Store.get('waveMood'));
      const tracks = d.tracks || [];
      if (!tracks.length) Store.set({ waveEnded: true });
      const prevQueue = Store.get('queue');
      const newQueue = reset ? tracks.slice() : prevQueue.concat(tracks);
      Store.set({ queue: newQueue, playbackCtx: 'wave' });
      if (reset) listEl.innerHTML = '';
      const startIdx = reset ? 0 : (newQueue.length - tracks.length);
      listEl.insertAdjacentHTML('beforeend',
        tracks.map((t, i) => TrackCard.render(t, startIdx + i, 'wave')).join('')
      );
    } catch (e) {
      listEl.insertAdjacentHTML('beforeend', '<p class="empty">Не удалось загрузить волну</p>');
    }
    Store.set({ waveLoading: false });
    if (loaderEl) loaderEl.style.display = 'none';
  }

  function reload() {
    Store.set({ queue: [], waveEnded: false });
    if (listEl) listEl.innerHTML = '';
    loadWave(true);
  }

  function mount(el) {
    root = el;
    ensureMarkup();
    MoodPicker.mount(root.querySelector('#mood-scroll'));
    attachTrackListDelegate(listEl, () => Store.get('queue'), 'wave');
    // Infinite scroll (preserves current behavior — scoped to content-area)
    const ca = document.getElementById('content-area');
    ca.addEventListener('scroll', (e) => {
      if (Store.get('screen') !== 'wave') return;
      const el = e.target;
      if (el.scrollHeight - el.scrollTop - el.clientHeight < 500) loadWave(false);
    });
    loadWave(true);
  }

  function destroy() { /* HomeScreen is mounted once; destroy unused in Phase 1 */ }

  return { mount, destroy, reload, loadWave };
})();
```

4. **SourceChips** — общий child для Search таба и Search-модала:
```js
const SourceChips = (() => {
  function mount(rootEl, storeKey, onChange) {
    const active = Store.get(storeKey);
    rootEl.innerHTML = Config.SOURCES.map(s =>
      `<div class="chip${s.id === active ? ' active' : ''}" data-src="${s.id}">${Utils.escapeHtml(s.name)}</div>`
    ).join('');
    rootEl.onclick = (e) => {
      const chip = e.target.closest('[data-src]');
      if (!chip) return;
      rootEl.querySelectorAll('.chip').forEach(c => c.classList.remove('active'));
      chip.classList.add('active');
      Store.set({ [storeKey]: chip.dataset.src });
      onChange(chip.dataset.src);
    };
  }
  return { mount };
})();
```

5. **SearchScreen** (Search-таб):
```js
const SearchScreen = (() => {
  let root, inputEl, resultsEl, timer = null;

  async function doSearch() {
    const q = inputEl.value.trim();
    if (!q) { resultsEl.innerHTML = '<p class="empty">Начните вводить запрос</p>'; return; }
    resultsEl.innerHTML = '<p class="loading">Ищем…</p>';
    try {
      const d = await Api.search(q, Store.get('activeSource'));
      const tracks = d.tracks || [];
      Store.set({ searchTabResults: tracks });
      resultsEl.innerHTML = tracks.length
        ? tracks.map((t, i) => TrackCard.render(t, i, 'search-tab')).join('')
        : '<p class="empty">Ничего не найдено</p>';
    } catch (e) { resultsEl.innerHTML = '<p class="empty">Ошибка поиска</p>'; }
  }

  function mount(el) {
    root = el;
    inputEl   = root.querySelector('#search-tab-input');
    resultsEl = root.querySelector('#search-tab-results');
    SourceChips.mount(root.querySelector('#source-chips'), 'activeSource', () => doSearch());
    inputEl.addEventListener('input', () => {
      clearTimeout(timer);
      timer = setTimeout(doSearch, 500);   // 500ms preserves current behavior; 300ms is Phase 10 (SR-1)
    });
    attachTrackListDelegate(resultsEl, () => Store.get('searchTabResults') || [], 'search-tab');
  }
  function destroy() {}
  return { mount, destroy };
})();
```

6. **SearchModal** (overlay search): пере-инкапсулировать существующий `#search-overlay` (markup строки 487–496) как mini-компонент с тем же подходом. Полный код аналогичен SearchScreen, но с `searchModalResults` и `activeModalSource`. Закрытие модала — `document.getElementById('search-overlay').classList.remove('open')` (или новый Store-флаг — но пока inline ok, это не overlay в смысле BackButton precedence).

```js
const SearchModal = (() => {
  let root, inputEl, resultsEl, timer = null;
  async function doSearch() {
    const q = inputEl.value.trim();
    if (!q) { resultsEl.innerHTML = '<p class="empty">Введите запрос для поиска</p>'; return; }
    resultsEl.innerHTML = '<p class="loading">Ищем…</p>';
    try {
      const d = await Api.search(q, Store.get('activeModalSource'));
      const tracks = d.tracks || [];
      Store.set({ searchModalResults: tracks });
      resultsEl.innerHTML = tracks.length
        ? tracks.map((t, i) => TrackCard.render(t, i, 'search-modal')).join('')
        : '<p class="empty">Ничего не найдено</p>';
    } catch (e) { resultsEl.innerHTML = '<p class="empty">Ошибка поиска</p>'; }
  }
  function mount(el) {
    root = el;
    inputEl   = root.querySelector('#search-input');
    resultsEl = root.querySelector('#search-results-list');
    SourceChips.mount(root.querySelector('#modal-source-chips'), 'activeModalSource', () => doSearch());
    inputEl.addEventListener('input', () => { clearTimeout(timer); timer = setTimeout(doSearch, 500); });
    attachTrackListDelegate(resultsEl, () => Store.get('searchModalResults') || [], 'search-modal');
    root.querySelector('.search-close').onclick = () => root.classList.remove('open');
  }
  function destroy() {}
  function open() { root.classList.add('open'); setTimeout(() => inputEl.focus(), 200); }
  return { mount, destroy, open };
})();
```

7. **GenreScreen** (Жанры-таб):
```js
const GenreScreen = (() => {
  let root, gridEl, resultsEl, listEl, titleEl;
  function renderGrid() {
    gridEl.innerHTML = Config.GENRES.map(g => `
      <div class="genre-card" style="background:${g.grad}" data-genre="${Utils.escapeHtml(g.name)}">
        <div class="g-emoji">${g.emoji}</div>
        <div class="g-name">${Utils.escapeHtml(g.name)}</div>
      </div>`).join('');
  }
  async function loadGenre(name) {
    gridEl.style.display = 'none';
    resultsEl.style.display = 'block';
    titleEl.innerText = name;
    listEl.innerHTML = '<p class="loading">Загружаем…</p>';
    try {
      const d = await Api.genre(name);
      const tracks = d.tracks || [];
      Store.set({ genreResults: tracks });
      listEl.innerHTML = tracks.length
        ? tracks.map((t, i) => TrackCard.render(t, i, 'genre')).join('')
        : '<p class="empty">Пусто</p>';
    } catch (e) { listEl.innerHTML = '<p class="empty">Ошибка загрузки</p>'; }
  }
  function mount(el) {
    root = el;
    gridEl    = root.querySelector('#genre-grid');
    resultsEl = root.querySelector('#genre-results');
    listEl    = root.querySelector('#genre-list');
    titleEl   = root.querySelector('#genre-results-title');
    renderGrid();
    gridEl.onclick = (e) => {
      const card = e.target.closest('[data-genre]');
      if (card) loadGenre(card.dataset.genre);
    };
    root.querySelector('.back-btn').onclick = () => {
      gridEl.style.display = 'grid';
      resultsEl.style.display = 'none';
    };
    attachTrackListDelegate(listEl, () => Store.get('genreResults') || [], 'genre');
  }
  function destroy() {}
  return { mount, destroy };
})();
```

8. **LibraryScreen** (Коллекция-таб):
```js
const LibraryScreen = (() => {
  let root, resultsEl;
  async function load() {
    resultsEl.innerHTML = '<p class="loading">Загружаем лайки…</p>';
    try {
      const d = await Api.likes();
      const tracks = d.tracks || [];
      Store.set({ collectionResults: tracks });
      resultsEl.innerHTML = tracks.length
        ? tracks.map((t, i) => TrackCard.render(t, i, 'collection')).join('')
        : '<p class="empty">Пока нет лайков</p>';
    } catch (e) { resultsEl.innerHTML = '<p class="empty">Ошибка загрузки</p>'; }
  }
  function mount(el) {
    root = el;
    resultsEl = root.querySelector('#collection-results');
    attachTrackListDelegate(resultsEl, () => Store.get('collectionResults') || [], 'collection');
    // Lazy load on screen entry
    Store.subscribe(['screen'], (state) => { if (state.screen === 'collection') load(); });
  }
  function destroy() {}
  return { mount, destroy, reload: load };
})();
```

9. **TasteBars** (child of ProfileScreen):
```js
const TasteBars = (() => {
  let root, unsub;
  function render() {
    const taste = Store.get('taste');
    if (!taste || !taste.length) {
      root.innerHTML = '<p style="color:var(--text-muted); font-size:13px;">Слушайте больше, чтобы появился профиль</p>';
      return;
    }
    const max = Math.max(...taste.map(t => t.weight || t.score || 0), 1);
    root.innerHTML = taste.slice(0, 8).map(t => {
      const w = t.weight || t.score || 0;
      const pct = Math.round((w / max) * 100);
      return `<div class="taste-row">
        <div class="taste-label"><span>${Utils.escapeHtml(t.name || t.tag || '')}</span><span>${pct}%</span></div>
        <div class="taste-bar"><div class="taste-fill" style="width:${pct}%"></div></div>
      </div>`;
    }).join('');
  }
  function mount(el) { root = el; unsub = Store.subscribe(['taste'], render); render(); }
  function destroy() { if (unsub) unsub(); }
  return { mount, destroy };
})();
```

10. **ProfileScreen** (Профиль-таб):
```js
const ProfileScreen = (() => {
  let root, statsEl, resetBtn;
  async function load() {
    try {
      const [taste, stats] = await Promise.all([Api.taste(), Api.stats()]);
      Store.set({ taste: taste.tags || [], stats });
      renderStats(stats);
    } catch (e) { /* TasteBars render handles empty */ }
  }
  function renderStats(s) {
    const items = [
      { v: s.tracks_count || s.tracks || 0, l: 'Треков' },
      { v: s.likes_count  || s.likes  || 0, l: 'Лайков' },
      { v: s.listens_count|| s.listens|| 0, l: 'Прослушиваний' },
      { v: s.artists_count|| s.artists|| 0, l: 'Артистов' },
    ];
    statsEl.innerHTML = items.map(i =>
      `<div class="stat-box"><div class="stat-value">${i.v}</div><div class="stat-label">${Utils.escapeHtml(i.l)}</div></div>`
    ).join('');
  }
  async function reset() {
    // Per ARCH-4 / TG-4: tg.showConfirm — это требование Phase 11 (PROF-4). Phase 1 сохраняет current behavior с window.confirm.
    if (!confirm('Сбросить вкусовой профиль?')) return;
    try { await Api.tasteReset(); load(); } catch (e) {}
  }
  function mount(el) {
    root = el;
    statsEl  = root.querySelector('#stat-grid');
    resetBtn = root.querySelector('.btn-danger');
    TasteBars.mount(root.querySelector('#taste-bars'));
    if (resetBtn) resetBtn.onclick = reset;
    Store.subscribe(['screen'], (state) => { if (state.screen === 'profile') load(); });
  }
  function destroy() {}
  return { mount, destroy };
})();
```

11. **BottomNav**:
```js
const BottomNav = (() => {
  let root, unsub;
  function render() {
    const s = Store.get('screen');
    root.querySelectorAll('.nav-item').forEach(n => n.classList.toggle('active', n.dataset.tab === s));
  }
  function mount(el) {
    root = el;
    root.onclick = (e) => {
      const item = e.target.closest('[data-tab]');
      if (!item) return;
      Router.go(item.dataset.tab, { push: false });
    };
    unsub = Store.subscribe(['screen'], render);
    render();
  }
  function destroy() { if (unsub) unsub(); }
  return { mount, destroy };
})();
```

12. **Привязать helper** `attachTrackListDelegate` — поместить ровно над секцией компонентов или сразу под секцией Utils. Это helper, не компонент.
  </action>
  <verify>
    <automated>grep -c "const TrackCard = (() =>" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "const TrackCard = (() =>" index.html` → `1`
    - `grep -c "const MoodPicker = (() =>" index.html` → `1`
    - `grep -c "const HomeScreen = (() =>" index.html` → `1`
    - `grep -c "const SourceChips = (() =>" index.html` → `1`
    - `grep -c "const SearchScreen = (() =>" index.html` → `1`
    - `grep -c "const SearchModal = (() =>" index.html` → `1`
    - `grep -c "const GenreScreen = (() =>" index.html` → `1`
    - `grep -c "const LibraryScreen = (() =>" index.html` → `1`
    - `grep -c "const TasteBars = (() =>" index.html` → `1`
    - `grep -c "const ProfileScreen = (() =>" index.html` → `1`
    - `grep -c "const BottomNav = (() =>" index.html` → `1`
    - `grep -c "function attachTrackListDelegate" index.html` → `1`
    - **Все 12 компонентов из ARCH-3** существуют в файле (TrackCard считается одним; MoodPicker, SourceChips, TasteBars, BottomNav — тоже компоненты по списку требований)
  </acceptance_criteria>
  <done>Все 9 экранных + детских компонентов + TrackCard pure-fn определены в `// ===== COMPONENTS =====`. Mount-вызовы будут добавлены в Plan 08 (Boot).</done>
</task>

<task type="auto">
  <name>Task 7.2: Удалить legacy load-функции и temporary subscribe-блоки</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (после Task 7.1 — надо удалить старые функции, дублирующие логику внутри компонентов)
  </read_first>
  <action>
**Удалить целиком** следующие top-level функции и блоки (всё это мигрировало в компоненты в Task 7.1):

1. `function loadMoods()` — заменено `MoodPicker.load()`
2. `function selectMood(mood, el)` — заменено `MoodPicker.hydrate()`
3. `function loadWave(reset)` — заменено `HomeScreen.loadWave()`
4. Глобальный `document.getElementById('content-area').addEventListener('scroll', ...)` infinite-scroll handler — перенесён внутрь `HomeScreen.mount()`
5. `function renderSourceChips(...)` — заменено `SourceChips.mount()`
6. `let searchTimer = null; function setupSearchTab() {...}` — внутри `SearchScreen.mount()`
7. `async function doTabSearch()` — внутри `SearchScreen.doSearch()`
8. `function openSearch() {...}` — заменено `SearchModal.open()`
9. `function closeSearch() {...}` — больше не нужен
10. `let modalSearchTimer = null; function setupSearchModal() {...}` — внутри `SearchModal.mount()`
11. `async function doModalSearch() {...}` — внутри `SearchModal.doSearch()`
12. `function renderGenres()` — внутри `GenreScreen.renderGrid()`
13. `async function loadGenre(name)` — внутри `GenreScreen.loadGenre()`
14. `function showGenreGrid()` — внутри обработчика back-btn в `GenreScreen.mount()`
15. `async function loadCollection()` — внутри `LibraryScreen.load()`
16. `async function loadProfile()` — внутри `ProfileScreen.load()`
17. `function renderTaste(tags)` — внутри `TasteBars.render()`
18. `function renderStats(s)` — внутри `ProfileScreen.renderStats()`
19. `async function resetTaste()` — внутри `ProfileScreen.reset()`
20. `function buildTrackHTML(t, i, ctx)` — заменено `TrackCard.render()`
21. `function playFromContext(i, ctx)` — заменено `attachTrackListDelegate` + `PlaybackEngine.playContext`
22. `function likeTrack(t)` и `function dislikeTrack(t)` — больше не вызываются (FullscreenPlayer и attachTrackListDelegate напрямую вызывают `Api.like/dislike`). НО: `likeTrack` всё ещё вызывается из `FullscreenPlayer` Plan 06 hydrate (`case 'like'`) и из `toggleCollectionCurrent`. Поэтому **оставить** thin shim:
```js
function likeTrack(t)    { return Api.like(t).catch(() => {}); }
function dislikeTrack(t) { return Api.dislike(t).catch(() => {}); }
```
Эти shim'ы чистятся в Plan 08 (Boot — только если все call sites переведены на Api напрямую).

23. **Удалить временный subscribe из Plan 05:**
```js
Store.subscribe(['screen'], (state) => { /* lazy load loadWave/loadCollection/loadProfile */ });
```
Заменён компонент-driven lazy load: `LibraryScreen` и `ProfileScreen` сами подписываются на `screen`. Wave грузится из `HomeScreen.mount()` (в Plan 08) безусловно (потому что Wave — стартовый экран). Тогглинг видимости табов всё ещё нужен — он переходит в Plan 08 (Boot).

24. **Очистить inline `onclick=` из markup'а** (строки 437–496 + 480–484):
   - Убрать `onclick="openSearch()"` с FAB → заменить на `id="fab-search"`, после mount компонентов навесить `document.getElementById('fab-search').onclick = () => SearchModal.open()` в Boot.
   - Убрать `onclick="openFullPlayer()"` с `#mini-player` — обработчик уже есть в `MiniPlayer.hydrate()`.
   - Убрать `onclick="closeFullPlayer()"`, `onclick="toggleCollectionCurrent()"`, `onclick="seekAudio(this.value)"`, `onclick="togglePlay()"`, `onclick="playPrev()"`, `onclick="skipNext()"`, `onclick="sendFeedback(...)"` со всех элементов внутри `#full-player` — все обработчики уже навешаны в `FullscreenPlayer.hydrate()`.
   - Убрать `onclick="switchTab(...)"` с `.nav-item` — обработчик в `BottomNav.mount()`.
   - Убрать `onclick="resetTaste()"` с кнопки сброса — обработчик в `ProfileScreen.mount()`.
   - Убрать `onclick="closeSearch()"` с `.search-close` — навесим в `SearchModal.mount()`.

25. **Удалить из shim-блока** (Plan 01) — больше не нужны:
   - `const SOURCES = Config.SOURCES;`
   - `const SOURCE_ICONS = Config.SOURCE_ICONS;`
   - `const GENRES = Config.GENRES;`
   - `const escapeHtml = Utils.escapeHtml;`
   - `const escapeAttr = Utils.escapeAttr;`
   - `const fmtTime = Utils.fmtTime;`
   - **Оставить** `const tg = Tg.tg;` и `const USER_ID = Tg.USER_ID;` — могут ещё быть call sites; чистка — Plan 08.
   - Оставить `function likeTrack`/`function dislikeTrack` shims (см. п.22).
   - Оставить shim'ы для `currentQueue`/`currentIndex`/`currentCtx`/`waveMood`/`waveLoading`/`waveEnded`/`activeSource`/`activeModalSource` — все load-функции, которые их использовали, удалены, но shim'ы дешёвые. Чистка — Plan 08.
  </action>
  <verify>
    <automated>grep -cE "^\s*function loadWave\(" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -cE "^\s*function loadWave\(" index.html` → `0`
    - `grep -cE "^\s*function loadMoods\(" index.html` → `0`
    - `grep -cE "^\s*function selectMood\(" index.html` → `0`
    - `grep -cE "^\s*function doTabSearch\(" index.html` → `0`
    - `grep -cE "^\s*function doModalSearch\(" index.html` → `0`
    - `grep -cE "^\s*function loadGenre\(" index.html` → `0`
    - `grep -cE "^\s*function loadCollection\(" index.html` → `0`
    - `grep -cE "^\s*function loadProfile\(" index.html` → `0`
    - `grep -cE "^\s*function buildTrackHTML\(" index.html` → `0`
    - `grep -cE "^\s*function playFromContext\(" index.html` → `0`
    - `grep -cE "^\s*function renderTaste\(" index.html` → `0`
    - `grep -cE "^\s*function renderStats\(" index.html` → `0`
    - `grep -cE "^\s*function renderGenres\(" index.html` → `0`
    - `grep -cE "^\s*function renderSourceChips\(" index.html` → `0`
    - `grep -cE "^\s*function setupSearchTab\(" index.html` → `0`
    - `grep -cE "^\s*function setupSearchModal\(" index.html` → `0`
    - `grep -cE "^\s*function openSearch\(" index.html` → `0`
    - `grep -cE "^\s*function closeSearch\(" index.html` → `0`
    - `grep -cE "^\s*function showGenreGrid\(" index.html` → `0`
    - `grep -cE "^\s*function resetTaste\(" index.html` → `0`
    - **Inline onclick audit** в markup: `grep -c "onclick=" index.html` ≤ `2` (допустимо `onerror="this.style.opacity=0.3"` на `<img>` — это onerror, не onclick; идеально 0 onclick'ов)
    - Browser smoke (полный): открыть → Wave загружается через `HomeScreen` → треки играют через `attachTrackListDelegate` + `PlaybackEngine.playContext('wave')` → переключить на Поиск → ввести «pop» → результаты появляются через `SearchScreen` → тап на трек → играет → переключить на Жанры → тап «Pop» → треки появляются через `GenreScreen` → играют → переключить на Коллекция → лайки видны через `LibraryScreen` → играют → Профиль → taste bars и stats через `ProfileScreen`/`TasteBars` → нажать Сбросить → confirm dialog → вкус сброшен → mini-player всё это время виден → fullscreen работает → BackButton precedence работает.
  </acceptance_criteria>
  <done>Все legacy load-функции удалены, inline onclick'и зачищены, ARCH-3 (12 компонентов) полностью реализован.</done>
</task>

</tasks>

<threat_model>
| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-10 | Information Disclosure | TrackCard innerHTML | mitigate | Все API-данные проходят через `Utils.escapeHtml` перед template literal interpolation |
| T-01-11 | Tampering | data-idx integer parsing | mitigate | `parseInt(row.dataset.idx, 10)` с явным radix; bounds-check `tracks[idx]` перед использованием |
</threat_model>

<verification>
- Все 5 экранов работают
- Все 12 компонентов ARCH-3 присутствуют как IIFE
- Все load-функции удалены
- inline onclick почти полностью убран
</verification>

<success_criteria>
- ARCH-3 закрыт целиком
- Готово к Boot-фазе (Plan 08), которая будет навешивать mount-вызовы и финальную smoke-проверку
</success_criteria>

<output>
После завершения создать `.planning/phases/01-architecture-refactor/01-07-SUMMARY.md` со списком всех 12 реализованных компонентов и подтверждением что все 5 экранов проходят smoke-тест.
</output>
