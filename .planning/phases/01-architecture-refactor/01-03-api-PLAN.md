---
phase: 01-architecture-refactor
plan: 03
type: execute
wave: 3
depends_on: ["01-02"]
files_modified: [index.html]
autonomous: true
requirements: [ARCH-5]
must_haves:
  truths:
    - "Существует один объект `Api` с методами для каждого backend-эндпоинта"
    - "Все существующие fetch-вызовы в index.html идут через `Api.*`"
    - "Параллельные одинаковые запросы дедуплицируются (один inflight Promise)"
    - "При HTTP-ошибке метод бросает Error с понятным сообщением"
  artifacts:
    - path: "index.html"
      provides: "Api IIFE с дедупом и error envelope"
      contains: "const Api = (() =>"
  key_links:
    - from: "loadWave / doTabSearch / loadGenre / loadCollection / loadProfile / likeTrack / dislikeTrack / sendFeedback / resetTaste / playTrack"
      to: "Api.*"
      via: "method call"
      pattern: "Api\\.(wave|search|genre|likes|stats|taste|tasteReset|like|dislike|feedback|streamUrl|waveMoods)"
---

<objective>
Создать `Api` IIFE — единственный мост между фронтом и `https://api.vdsmusic.ru:8443`. Перевести все текущие `fetch(...)` в index.html на методы `Api.*`. Backend frozen — никаких новых эндпоинтов, никаких изменений payload'ов.

Purpose: ARCH-5 («один index.html, CDN-only») и подготовка к Plans 04 (PlaybackEngine использует `Api.streamUrl` и `Api.feedback`) и 07 (компоненты экранов используют `Api.search`, `Api.genre`, `Api.likes`, `Api.taste`, `Api.stats`).

Output: Все 11 backend-эндпоинтов обёрнуты в `Api.*`; все вызовы из index.html идут через них; дедуп через inflight-map.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/REQUIREMENTS.md
@.planning/research/ARCHITECTURE.md
@index.html
@.planning/phases/01-architecture-refactor/01-02-SUMMARY.md
</context>

<interfaces>
**Целевой Api API** — методы соответствуют ровно тем эндпоинтам, что уже работают в index.html (см. строки 572, 600, 663, 703, 730, 750, 765, 766, 809, 866, 894, 965, 979, 990):

```js
const Api = (() => {
  const base = Config.API;
  const inflight = new Map();

  function key(method, path, body) {
    return `${method} ${path} ${body ? JSON.stringify(body) : ''}`;
  }

  async function req(path, opts = {}) {
    const method = opts.method || 'GET';
    const k = key(method, path, opts.body);
    if (inflight.has(k)) return inflight.get(k);
    const p = (async () => {
      const r = await fetch(base + path, {
        method,
        headers: opts.body ? { 'Content-Type': 'application/json' } : undefined,
        body: opts.body ? JSON.stringify(opts.body) : undefined,
      });
      if (!r.ok) throw new Error(`HTTP ${r.status} on ${method} ${path}`);
      return await r.json();
    })();
    inflight.set(k, p);
    p.finally(() => inflight.delete(k));
    return p;
  }

  // streamUrl is NOT a fetch — it's URL construction for <audio>.src
  function streamUrl(t) {
    const id = encodeURIComponent(t.id || '');
    const source = encodeURIComponent(t.source || '');
    const url = encodeURIComponent(t.stream_url || '');
    const title = encodeURIComponent(t.title || '');
    const artist = encodeURIComponent(t.artist || '');
    return `${base}/stream/${id}?source=${source}&url=${url}&title=${title}&artist=${artist}`;
  }

  return {
    wave:       (mood)        => req(`/wave?user_id=${Tg.USER_ID}&mood=${encodeURIComponent(mood || '')}`),
    waveMoods:  ()            => req(`/wave/moods?user_id=${Tg.USER_ID}`),
    search:     (q, src)      => req(`/search?q=${encodeURIComponent(q)}${src && src !== 'all' ? `&sources=${src}` : ''}&user_id=${Tg.USER_ID}`),
    genre:      (name)        => req(`/genre?q=${encodeURIComponent(name)}&user_id=${Tg.USER_ID}`),
    likes:      ()            => req(`/library/likes?user_id=${Tg.USER_ID}`),
    stats:      ()            => req(`/library/stats?user_id=${Tg.USER_ID}`),
    taste:      ()            => req(`/taste/${Tg.USER_ID}`),
    tasteReset: ()            => req(`/taste/reset?user_id=${Tg.USER_ID}`, { method: 'POST' }),
    like:       (t)           => req(`/library/like`,    { method: 'POST', body: { user_id: Tg.USER_ID, track_id: t.id, track: t } }),
    dislike:    (t)           => req(`/library/dislike`, { method: 'POST', body: { user_id: Tg.USER_ID, track_id: t.id } }),
    feedback:   (ev)          => req(`/wave/feedback`,   { method: 'POST', body: ev }),
    streamUrl,
  };
})();
```

**Текущие fetch-сайты в index.html, которые нужно переписать (точные строки в pre-refactor состоянии):**

| Строка | Текущий вызов | Новый вызов |
|---|---|---|
| 572 | `fetch(\`${API}/wave/moods?user_id=${USER_ID}\`)` | `Api.waveMoods()` |
| 600 | `fetch(url)` где url=`${API}/wave?...` | `Api.wave(waveMood)` |
| 663 | `fetch(\`${API}/search?...\`)` (tab) | `Api.search(q, activeSource)` |
| 703 | `fetch(\`${API}/search?...\`)` (modal) | `Api.search(q, activeModalSource)` |
| 730 | `fetch(\`${API}/genre?...\`)` | `Api.genre(name)` |
| 750 | `fetch(\`${API}/library/likes?...\`)` | `Api.likes()` |
| 765–766 | `fetch(\`${API}/taste/${USER_ID}\`)`, `fetch(\`${API}/library/stats?...\`)` | `Api.taste()`, `Api.stats()` |
| 809 | `fetch(\`${API}/taste/reset?...\`, { method: 'POST' })` | `Api.tasteReset()` |
| 866 | `const streamUrl = \`${API}/stream/...\`` | `const streamUrl = Api.streamUrl(t)` |
| 894 | `const url = \`${API}/stream/...\`` (preloadNext) | `const url = Api.streamUrl(next)` |
| 965 | `fetch(\`${API}/wave/feedback\`, { method: 'POST', ... })` | `Api.feedback({ user_id: Tg.USER_ID, track_id: t.id, action, elapsed: ... })` |
| 979 | `fetch(\`${API}/library/like\`, ...)` | `Api.like(t)` |
| 990 | `fetch(\`${API}/library/dislike\`, ...)` | `Api.dislike(t)` |
</interfaces>

<tasks>

<task type="auto">
  <name>Task 3.1: Создать Api IIFE в секции `// ===== API =====`</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (секции `// ===== API =====` после Plan 02; все fetch-сайты — найти через grep `fetch(` в скрипте)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§6 «Data flow to backend» — verbatim Api IIFE и таблица эндпоинтов)
  </read_first>
  <action>
В секцию `// ===== API =====` вставить ровно тот код Api IIFE, что приведён в `<interfaces>` выше. ВАЖНО:
1. Использовать `Config.API` и `Tg.USER_ID` (а не legacy shim'ы `API` / `USER_ID`) — Api сразу пишется в чистом стиле.
2. Не добавлять retry-loop (это Plan для Phase 2 / AUDIO-6). В Plan 03 — только дедуп и error envelope.
3. Не добавлять Feedback queue — это будет минимальная заглушка в Plan 04 (полная реализация Feedback module — Phase 2).
  </action>
  <verify>
    <automated>grep -c "const Api = (() =>" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "const Api = (() =>" index.html` → `1`
    - `grep -c "function streamUrl(t)" index.html` → `1`
    - `grep -c "inflight.set(k, p)" index.html` → `1`
    - В DevTools console: `await Api.waveMoods()` возвращает объект с `moods`
    - В DevTools console: `Api.streamUrl({id:'x',source:'youtube',stream_url:'',title:'t',artist:'a'})` возвращает строку начинающуюся с `https://api.vdsmusic.ru:8443/stream/x?source=youtube`
  </acceptance_criteria>
  <done>Api IIFE доступен; все 11 методов и `streamUrl` присутствуют; дедуп работает.</done>
</task>

<task type="auto">
  <name>Task 3.2: Заменить все fetch-вызовы на Api.* и убрать window.* очереди</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (все строки с `fetch(`, `window.searchTabQueue`, `window.searchModalQueue`, `window.genreQueue`, `window.collectionQueue`)
    - Таблица замен в `<interfaces>` выше
  </read_first>
  <action>
1. **Заменить fetch-вызовы** согласно таблице в `<interfaces>` (12 сайтов). После замены `loadWave` должен выглядеть примерно так:

```js
async function loadWave(reset) {
  if (waveLoading || waveEnded) return;
  waveLoading = true;
  document.getElementById('wave-loader').style.display = 'block';
  try {
    const d = await Api.wave(waveMood);
    const tracks = d.tracks || [];
    if (!tracks.length) { waveEnded = true; }
    if (reset) { currentQueue = tracks.slice(); }
    else { currentQueue = currentQueue.concat(tracks); }
    const list = document.getElementById('wave-list');
    list.insertAdjacentHTML('beforeend', tracks.map((t, i) => {
      const idx = reset ? i : (currentQueue.length - tracks.length + i);
      return buildTrackHTML(t, idx, 'wave');
    }).join(''));
  } catch (e) {
    document.getElementById('wave-list').insertAdjacentHTML('beforeend',
      '<p class="empty">Не удалось загрузить волну</p>');
  }
  waveLoading = false;
  document.getElementById('wave-loader').style.display = 'none';
}
```

Аналогично переписать `loadMoods`, `doTabSearch`, `doModalSearch`, `loadGenre`, `loadCollection`, `loadProfile`, `resetTaste`, `playTrack` (только URL-сайт), `preloadNext`, `sendFeedback`, `likeTrack`, `dislikeTrack`.

2. **Убрать `window.*Queue` глобалы.** Заменить:
   - `window.searchTabQueue = d.tracks || [];` → `Store.set({ searchTabResults: d.tracks || [] });`
   - `window.searchModalQueue = d.tracks || [];` → `Store.set({ searchModalResults: d.tracks || [] });`
   - `window.genreQueue = d.tracks || [];` → `Store.set({ genreResults: d.tracks || [] });`
   - `window.collectionQueue = d.tracks || [];` → `Store.set({ collectionResults: d.tracks || [] });`
   - И в `playFromContext` (строки 833–841) поменять чтение `window.searchTabQueue`/`window.genreQueue`/`window.collectionQueue`/`window.searchModalQueue` на `Store.get('searchTabResults')`/`Store.get('genreResults')`/`Store.get('collectionResults')`/`Store.get('searchModalResults')`.

3. **`sendFeedback`** перевести на `Api.feedback`:
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

4. **Удалить из shim-блока (Plan 01):**
   - `const API = Config.API;` — больше не нужен, никакой код напрямую `API` не использует
   - **Оставить** `const fmtTime = Utils.fmtTime;`, `const escapeHtml = Utils.escapeHtml;`, `const escapeAttr = Utils.escapeAttr;` — пока на них опираются `buildTrackHTML`, `selectMood`, `loadMoods`, `loadGenre`. Уйдут в Plan 07.
   - **Оставить** `const tg = Tg.tg;`, `const USER_ID = Tg.USER_ID;` — могут ещё попасться. Уйдут в Plan 08.
   - **Оставить** `const SOURCES = Config.SOURCES;`, `const SOURCE_ICONS = Config.SOURCE_ICONS;`, `const GENRES = Config.GENRES;` — на них опираются `renderSourceChips` и `renderGenres`. Уйдут в Plan 07.
  </action>
  <verify>
    <automated>grep -cE "fetch\(\`?\\\$?\{API\}" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -cE "fetch\(.*API.*\)" index.html` → `0` (все fetch-вызовы к backend переведены через Api)
    - `grep -c "Api.wave(" index.html` ≥ `1`
    - `grep -c "Api.search(" index.html` ≥ `2` (tab + modal)
    - `grep -c "Api.streamUrl(" index.html` ≥ `2` (playTrack + preloadNext)
    - `grep -c "Api.like(" index.html` ≥ `1`
    - `grep -c "Api.dislike(" index.html` ≥ `1`
    - `grep -c "Api.feedback(" index.html` → `1`
    - `grep -c "window.searchTabQueue" index.html` → `0`
    - `grep -c "window.searchModalQueue" index.html` → `0`
    - `grep -c "window.genreQueue" index.html` → `0`
    - `grep -c "window.collectionQueue" index.html` → `0`
    - `grep -c "Store.get('searchTabResults')" index.html` ≥ `1`
    - `grep -cE "^\s*const API = Config\.API" index.html` → `0` (удалён из shim)
    - Browser smoke: открыть в браузере, Wave-таб подгружает треки → переключить на Поиск → ввести «pop» → результаты появляются → тапнуть один результат → плеер играет; Жанры → тап «Pop» → треки появляются и играют; Коллекция → лайки видны; Профиль → taste bars. В Network log все запросы идут на api.vdsmusic.ru:8443.
  </acceptance_criteria>
  <done>Все backend-вызовы и stream-URL построения проходят через Api; ad-hoc window.*Queue глобалы убраны; функционал идентичен до рефактора.</done>
</task>

</tasks>

<threat_model>
| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-04 | Information Disclosure | `Api.req` error envelope | accept | Phase 1 рефактор не добавляет логирования; ошибки только в console |
| T-01-05 | Denial of Service | `Api.req` retry | mitigate | Retry осознанно НЕ добавлен в Plan 03 (это AUDIO-6 в Phase 2); inflight-дедуп уже снижает шанс параллельных дублей |
</threat_model>

<verification>
- Все 5 табов работают
- Network log показывает запросы только на api.vdsmusic.ru:8443
- Дублирующиеся запросы (например, два быстрых клика на тот же mood) дедуплицируются — в Network panel виден один request
</verification>

<success_criteria>
- ARCH-5 укреплён: API-доступ централизован
- Готова почва для PlaybackEngine (использует `Api.streamUrl` и `Api.feedback`)
</success_criteria>

<output>
После завершения создать `.planning/phases/01-architecture-refactor/01-03-SUMMARY.md` со списком переписанных fetch-сайтов и удалённых window.* глобалов.
</output>
