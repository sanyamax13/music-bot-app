---
phase: 01-architecture-refactor
plan: 04
type: execute
wave: 4
depends_on: ["01-03"]
files_modified: [index.html]
autonomous: true
requirements: [ARCH-2, ARCH-1]
must_haves:
  truths:
    - "Существует один объект `PlaybackEngine` с публичным API: playContext, play, pause, toggle, next, prev, seek, getPosition, getDuration"
    - "PlaybackEngine — единственное место, где код пишет в audioA / audioB / activeAudio / nextAudio"
    - "PlaybackEngine — единственный writer ключей `currentTrack`, `isPlaying`, `duration`, `playbackCtx`, `queue`, `queueIndex`"
    - "MediaSession metadata + handlers зарегистрированы один раз и обновляются из PlaybackEngine"
    - "Wave/Search/Genre/Collection продолжают играть как раньше"
  artifacts:
    - path: "index.html"
      provides: "PlaybackEngine IIFE — единственный владелец audio-элементов и playback-state"
      contains: "const PlaybackEngine = (() =>"
  key_links:
    - from: "playFromContext / skipNext / playPrev / togglePlay / seekAudio / onTrackEnded"
      to: "PlaybackEngine.*"
      via: "method call"
      pattern: "PlaybackEngine\\.(play|pause|toggle|next|prev|seek|playContext)"
---

<objective>
Создать `PlaybackEngine` IIFE — единственного владельца обоих `<audio>` элементов, очереди воспроизведения, MediaSession и playback-state в Store. Перевести все существующие пути воспроизведения (`playTrack`, `skipNext`, `playPrev`, `togglePlay`, `seekAudio`, `onTrackEnded`, `preloadNext`, `setupMediaSession`) на методы PlaybackEngine.

Это самый рискованный план фазы — здесь рефакторится playback-логика. Acceptance: после плана 10 подряд переключений треков работают, lock-screen controls работают, лайк отправляется как раньше.

Purpose: ARCH-2 целиком + ARCH-1 «один владелец на ключ» для playback-блока state. Разблокирует Plans 06/07 (компоненты подписываются на `currentTrack`/`isPlaying` и вызывают `PlaybackEngine.*`).

Output: PlaybackEngine с полным API; legacy playTrack/skipNext/playPrev/togglePlay/seekAudio становятся тонкими обёртками-делегатами (или удалены, см. Task 4.3).
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/REQUIREMENTS.md
@.planning/research/ARCHITECTURE.md
@.planning/research/PITFALLS.md
@index.html
@.planning/phases/01-architecture-refactor/01-03-SUMMARY.md
</context>

<interfaces>
**Целевой PlaybackEngine API** (ARCHITECTURE §5):

```js
const PlaybackEngine = (() => {
  // Owned audio elements — нигде больше не читаются и не пишутся
  const audioA = document.getElementById('audioA');
  const audioB = document.getElementById('audioB');
  let activeAudio = audioA;
  let nextAudio = audioB;

  // Internal feedback bookkeeping (was top-level in pre-refactor)
  let currentTrackStart = 0;
  let lastFeedbackSent = false;

  // === MediaSession setup — once, on init ===
  function initMediaSession() {
    if (!('mediaSession' in navigator)) return;
    navigator.mediaSession.setActionHandler('play',          () => toggle());
    navigator.mediaSession.setActionHandler('pause',         () => toggle());
    navigator.mediaSession.setActionHandler('previoustrack', () => prev());
    navigator.mediaSession.setActionHandler('nexttrack',     () => next());
    // seekto / seekbackward / seekforward / setPositionState — это AUDIO-3/4 в Phase 2, НЕ здесь
  }

  function setMetadata(t) {
    if (!('mediaSession' in navigator)) return;
    navigator.mediaSession.metadata = new MediaMetadata({
      title:  t.title  || '',
      artist: t.artist || '',
      artwork: t.thumb ? [{ src: t.thumb, sizes: '512x512', type: 'image/jpeg' }] : [],
    });
  }

  // === Public: play a context (queue + index) ===
  function playContext(ctx, tracks, index) {
    Store.set({ playbackCtx: ctx, queue: tracks.slice(), queueIndex: index });
    return playAt(index);
  }

  // === Public: play track at index in current queue ===
  async function playAt(i) {
    const queue = Store.get('queue');
    if (i < 0 || i >= queue.length) return;
    Store.set({ queueIndex: i });
    const t = queue[i];
    Store.set({ currentTrack: t });

    currentTrackStart = Date.now();
    lastFeedbackSent = false;

    // Swap dual buffers
    const prev = activeAudio;
    activeAudio = (activeAudio === audioA) ? audioB : audioA;
    nextAudio   = (nextAudio   === audioA) ? audioB : audioA;
    prev.pause();

    const url = Api.streamUrl(t);
    if (activeAudio.src !== url) {
      activeAudio.src = url;
    }

    // Wire events on active buffer (use named handlers to enable removeEventListener
    // hardening in Phase 2 / AUDIO-2 — for now keep .on* assignment to mirror current behavior)
    activeAudio.ontimeupdate    = onTimeUpdate;
    activeAudio.onended         = onTrackEnded;
    activeAudio.onloadedmetadata = () => Store.set({ duration: activeAudio.duration || 0 });

    setMetadata(t);

    try {
      await activeAudio.play();
      Store.set({ isPlaying: true });
      preloadNext();
      autoExtendIfWave();
    } catch (e) {
      Store.set({ isPlaying: false });
      console.warn('[PlaybackEngine] play() failed', e);
    }
  }

  function preloadNext() {
    const queue = Store.get('queue');
    const idx = Store.get('queueIndex');
    const next = queue[idx + 1];
    if (!next) return;
    nextAudio.src = Api.streamUrl(next);
    nextAudio.preload = 'auto';
    try { nextAudio.load(); } catch (e) {}
  }

  function autoExtendIfWave() {
    if (Store.get('playbackCtx') !== 'wave') return;
    const queue = Store.get('queue');
    const idx = Store.get('queueIndex');
    if (idx >= queue.length - 3 && typeof window.loadWave === 'function') {
      window.loadWave(false);   // legacy entry — replaced by HomeScreen in Plan 07
    }
  }

  function onTimeUpdate() {
    // Position is read-through (NOT stored in Store) per ARCHITECTURE §2 to avoid 10Hz subscriber thrash.
    // UI components read PlaybackEngine.getPosition() in their own rAF loops.
  }

  function onTrackEnded() {
    sendFeedbackInternal('finish');
    next();
  }

  function sendFeedbackInternal(action) {
    if (lastFeedbackSent) return;
    lastFeedbackSent = true;
    const idx = Store.get('queueIndex');
    if (idx < 0) return;
    const t = Store.get('queue')[idx];
    if (!t) return;
    Api.feedback({
      user_id: Tg.USER_ID,
      track_id: t.id,
      action,
      elapsed: Math.round((Date.now() - currentTrackStart) / 1000),
    }).catch(() => {});
  }

  // === Public controls ===
  function play()  { if (activeAudio.src) { activeAudio.play(); Store.set({ isPlaying: true }); } }
  function pause() { activeAudio.pause(); Store.set({ isPlaying: false }); }
  function toggle() {
    if (!activeAudio.src) return;
    if (activeAudio.paused) play(); else pause();
  }
  function next() { sendFeedbackInternal('skip'); playAt(Store.get('queueIndex') + 1); }
  function prev() { playAt(Store.get('queueIndex') - 1); }
  function seek(seconds) {
    if (!activeAudio.duration) return;
    activeAudio.currentTime = Math.max(0, Math.min(seconds, activeAudio.duration));
  }
  function seekPct(pct) {
    if (!activeAudio.duration) return;
    activeAudio.currentTime = (pct / 100) * activeAudio.duration;
  }

  // === Read-through position (called from rAF loops in components) ===
  function getPosition() { return activeAudio.currentTime || 0; }
  function getDuration() { return activeAudio.duration || 0; }
  function isPaused()    { return activeAudio.paused; }

  // === Internal feedback exposure (until full Feedback module in Phase 2) ===
  function sendManualFeedback(action) { sendFeedbackInternal(action); }

  initMediaSession();

  return {
    playContext, playAt, play, pause, toggle, next, prev,
    seek, seekPct, getPosition, getDuration, isPaused,
    sendManualFeedback,
  };
})();
```
</interfaces>

<tasks>

<task type="auto">
  <name>Task 4.1: Создать PlaybackEngine IIFE в секции `// ===== PLAYBACK ENGINE =====`</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (текущие функции `playTrack` строки 844–889, `preloadNext` 891–898, `autoExtendWave` 900–905, `onTimeUpdate` 907–912, `onTrackEnded` 914–917, `skipNext` 919–922, `playPrev` 924, `togglePlay` 926–930, `updatePlayUI` 932–935, `seekAudio` 937–940, `setupMediaSession` 945–956, `sendFeedback` 959–976)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§5 «Audio + queue architecture» — verbatim PlaybackEngine скелет)
    - /root/music-bot-frontend/.planning/research/PITFALLS.md (для понимания, что в этом плане НЕ исправляется — iOS unlock, named handlers, setPositionState — это всё Phase 2, не Phase 1)
  </read_first>
  <action>
В секцию `// ===== PLAYBACK ENGINE =====` вставить ровно тот код PlaybackEngine IIFE, что приведён в `<interfaces>`.

Дополнительно в `// ===== FEEDBACK =====` секцию вставить **минимальную заглушку** — полная реализация Feedback module с queue + localStorage откладывается на Phase 2 (она нужна для `AUDIO-6 retry`):

```js
// ===== FEEDBACK =====
// Phase 1 stub. Real reliable queue (with localStorage backing + visibilitychange flush)
// is Phase 2 scope. For now Feedback is just a thin pass-through to Api so call sites
// (PlaybackEngine.sendFeedbackInternal, future Wave swipes) have a stable name to call.
const Feedback = (() => {
  function send(event) { return Api.feedback(event).catch(() => {}); }
  return { send };
})();
```

ВАЖНО:
- НЕ удалять старые `playTrack`, `preloadNext`, `autoExtendWave`, `togglePlay`, `playPrev`, `skipNext`, `setupMediaSession`, `sendFeedback`, `onTrackEnded`, `updatePlayUI`, `seekAudio` функции в этом таске. Они будут переведены в shim в Task 4.2.
- НЕ трогать `let activeAudio = audioA, nextAudio = audioB;` на топ-уровне — он останется до Task 4.2.
  </action>
  <verify>
    <automated>grep -c "const PlaybackEngine = (() =>" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "const PlaybackEngine = (() =>" index.html` → `1`
    - `grep -c "function playContext(ctx, tracks, index)" index.html` → `1`
    - `grep -c "function getPosition()" index.html` → `1`
    - `grep -c "const Feedback = (() =>" index.html` → `1`
    - `grep -c "navigator.mediaSession.setActionHandler" index.html` ≥ `4` (play, pause, previoustrack, nexttrack)
    - В DevTools console: `typeof PlaybackEngine.playContext === 'function'` → `true`
  </acceptance_criteria>
  <done>PlaybackEngine и Feedback IIFE существуют. Старая логика всё ещё работает (используется legacy playTrack — заменим в Task 4.2).</done>
</task>

<task type="auto">
  <name>Task 4.2: Заменить legacy playback функции на тонкие делегаты к PlaybackEngine</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (всё, что было создано в Task 4.1 + старые playTrack/skipNext/playPrev/togglePlay/seekAudio/onTrackEnded/preloadNext/autoExtendWave/setupMediaSession/sendFeedback/updatePlayUI)
    - call sites старых функций: `switchTab`, `playFromContext` (833), `selectMood` (585), `loadWave` (594), `closeFullPlayer`, обработчики `onclick=` в HTML markup (439–474): `mp-play-btn`, `fp-play-btn`, `playPrev()`, `skipNext()`, кнопки feedback `sendFeedback('dislike')`, `sendFeedback('like')`, `seekAudio(this.value)`, `toggleCollectionCurrent`
  </read_first>
  <action>
1. **Удалить** старые функции (тела заменяются на делегаты или вообще убираются):
   - `function playTrack(i)` (строки 844–889) → удалить полностью
   - `function preloadNext()` (891–898) → удалить (это теперь приватный метод PlaybackEngine)
   - `function autoExtendWave()` (900–905) → удалить (это `autoExtendIfWave` внутри PlaybackEngine)
   - `function onTimeUpdate()` (907–912) → удалить (UI обновления — Plan 06 через rAF)
   - `function onTrackEnded()` (914–917) → удалить (внутри PlaybackEngine)
   - `function setupMediaSession(t)` (945–956) → удалить (внутри PlaybackEngine.initMediaSession + setMetadata)
   - `function sendFeedback(action)` (959–976) → удалить
   - `let currentTrackStart = 0, lastFeedbackSent = false;` → удалить (теперь внутри PlaybackEngine)
   - `let activeAudio = audioA, nextAudio = audioB;` → удалить (теперь внутри PlaybackEngine)
   - `const audioA = document.getElementById('audioA');` → удалить (теперь внутри PlaybackEngine)
   - `const audioB = document.getElementById('audioB');` → удалить

2. **Заменить** оставшиеся top-level функции на тонкие делегаты (они нужны как глобалы потому что HTML markup использует inline `onclick=`; markup чистим в Plans 06–07, не здесь):

```js
function playFromContext(i, ctx) {
  // ctx maps directly to playbackCtx + source of queue
  let tracks;
  switch (ctx) {
    case 'wave':         tracks = Store.get('queue'); break;  // wave queue is already the mainline
    case 'genre':        tracks = Store.get('genreResults') || []; break;
    case 'collection':   tracks = Store.get('collectionResults') || []; break;
    case 'search-tab':   tracks = Store.get('searchTabResults') || []; break;
    case 'search-modal': tracks = Store.get('searchModalResults') || []; closeSearch(); break;
    default:             tracks = []; break;
  }
  if (ctx === 'wave') {
    PlaybackEngine.playAt(i);     // queue already lives in Store from loadWave()
  } else {
    PlaybackEngine.playContext(ctx, tracks, i);
  }
}

function togglePlay() { PlaybackEngine.toggle(); }
function skipNext()   { PlaybackEngine.next(); }
function playPrev()   { PlaybackEngine.prev(); }
function seekAudio(v) { PlaybackEngine.seekPct(parseFloat(v)); }

function sendFeedback(action) {     // legacy глобал нужен для inline onclick'ов в FP markup (471–472)
  PlaybackEngine.sendManualFeedback(action);
}

function toggleCollectionCurrent() {
  const idx = Store.get('queueIndex');
  if (idx < 0) return;
  const t = Store.get('queue')[idx];
  if (t) likeTrack(t);
}
```

3. **Удалить** `function updatePlayUI(playing)` (932–935) — DOM-обновления mini-player и fp-play-btn теперь делает подписка на `isPlaying`. Но компонентов ещё нет (они в Plans 06–07), поэтому **временно** добавить subscribe сразу после Boot (или в конце скрипта перед `// ===== INIT =====`):

```js
// === Temporary play/pause UI sync until MiniPlayer/FullscreenPlayer components land in Plan 06 ===
Store.subscribe(['isPlaying'], (state) => {
  const txt = state.isPlaying ? '⏸' : '▶';
  const mp = document.getElementById('mp-play-btn'); if (mp) mp.innerText = txt;
  const fp = document.getElementById('fp-play-btn'); if (fp) fp.innerText = txt;
});

// === Temporary currentTrack UI sync (mini-player + fp meta) until Plan 06 ===
Store.subscribe(['currentTrack'], (state) => {
  const t = state.currentTrack;
  if (!t) return;
  const mini = document.getElementById('mini-player');     if (mini) mini.style.display = 'flex';
  const mpT  = document.getElementById('mp-title');        if (mpT)  mpT.innerText = t.title || '';
  const mpA  = document.getElementById('mp-artist');       if (mpA)  mpA.innerText = t.artist || '';
  const mpTh = document.getElementById('mp-thumb');        if (mpTh) mpTh.src = t.thumb || '';
  const fpT  = document.getElementById('fp-title');        if (fpT)  fpT.innerText = t.title || '';
  const fpA  = document.getElementById('fp-artist');       if (fpA)  fpA.innerText = t.artist || '';
  const fpC  = document.getElementById('fp-cover');        if (fpC)  fpC.src = t.thumb || '';
});

// === Temporary duration → fp-time-total sync ===
Store.subscribe(['duration'], (state) => {
  const el = document.getElementById('fp-time-total');
  if (el) el.innerText = Utils.fmtTime(state.duration);
});

// === Temporary fp-time-cur and progress bar via rAF (read-through, not Store-driven) ===
(function tickFpProgress() {
  const cur = document.getElementById('fp-time-cur');
  const prog = document.getElementById('progress');
  if (cur)  cur.innerText = Utils.fmtTime(PlaybackEngine.getPosition());
  if (prog) {
    const dur = PlaybackEngine.getDuration();
    if (dur) prog.value = (PlaybackEngine.getPosition() / dur) * 100;
  }
  requestAnimationFrame(tickFpProgress);
})();
```

Эти три подписки + rAF — временные мостики до Plan 06, который превратит их в `MiniPlayer` и `FullscreenPlayer` компоненты с `mount/render/destroy`.

4. **Удалить** из shim-блока (Plan 01): больше ничего не удалять в этом плане.
  </action>
  <verify>
    <automated>grep -cE "^\s*async function playTrack\(" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -cE "^\s*async function playTrack\(" index.html` → `0` (legacy playTrack удалён)
    - `grep -cE "^\s*function preloadNext\(" index.html` → `0`
    - `grep -cE "^\s*function autoExtendWave\(" index.html` → `0`
    - `grep -cE "^\s*function setupMediaSession\(" index.html` → `0`
    - `grep -cE "^\s*function updatePlayUI\(" index.html` → `0`
    - `grep -cE "^\s*let activeAudio" index.html` → `0`
    - `grep -cE "^\s*let currentTrackStart" index.html` → `0`
    - `grep -cE "^\s*const audioA = document.getElementById" index.html` → `0` (только внутри PlaybackEngine)
    - `grep -c "PlaybackEngine.playAt" index.html` ≥ `1`
    - `grep -c "PlaybackEngine.toggle" index.html` ≥ `1`
    - `grep -c "PlaybackEngine.next" index.html` ≥ `1`
    - `grep -c "PlaybackEngine.prev" index.html` ≥ `1`
    - `grep -c "PlaybackEngine.seekPct" index.html` ≥ `1`
    - **One-writer audit:** `grep -cE "Store\.set\(\{ ?(currentTrack|queueIndex|queue|isPlaying|playbackCtx|duration)" index.html` — все совпадения должны быть **только внутри PlaybackEngine IIFE** (визуально проверить через `grep -n` что соответствующие строки находятся между `const PlaybackEngine = (() =>` и его закрывающим `})();`)
    - Browser smoke test (КРИТИЧЕСКИ ВАЖНЫЙ): открыть index.html, тапнуть Wave-трек → играет → тап ⏸ → пауза → тап ⏵ → играет → тап ⏭ → следующий трек → 10 раз подряд тап ⏭ → плеер прошёл 10 треков, не залип → открыть полноэкранный плеер (тап на mini-player) → progress движется → seek scrubber работает → 👎/👍 в FP отправляют POST /wave/feedback (Network log) → закрыть FP → лайк трека из списка → POST /library/like → переключиться на Поиск → ввести «pop» → играет → переключить на Коллекция → играет лайкнутый.
    - Lock-screen / шторка Android (если доступно): ▶ ⏸ ⏭ ⏮ работают.
  </acceptance_criteria>
  <done>PlaybackEngine — единственный, кто трогает audio-элементы и пишет playback-state. Legacy playback функции либо удалены, либо превращены в однострочные делегаты. UI продолжает синхронизироваться через временные подписки (заменятся компонентами в Plan 06).</done>
</task>

</tasks>

<threat_model>
| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-06 | Tampering | PlaybackEngine internal state | accept | Single-file vanilla — нет реальной границы доверия |
| T-01-07 | Denial of Service | iOS WKWebView autoplay | accept | Pitfalls AUDIO-1 явно scope для Phase 2 — не должен покрываться этим планом, но архитектура (sync `play()` в одной функции) не препятствует фиксу |
</threat_model>

<verification>
- Browser smoke test (см. Task 4.2 acceptance) полностью пройден
- DevTools Console: нет ошибок при 10 подряд переключениях треков
- Network log: каждое переключение трека отправляет POST /wave/feedback (либо `skip`, либо `finish`)
- One-writer audit для playback-ключей (currentTrack/queueIndex/queue/isPlaying/playbackCtx/duration) — все Store.set этих ключей внутри PlaybackEngine
</verification>

<success_criteria>
- ARCH-2 закрыт (PlaybackEngine — единственный владелец audio + MediaSession)
- ARCH-1 укреплён для playback-ключей
- Voice-control / lock-screen / Bluetooth headphones — управление работает (через MediaSession handlers)
- Фундамент для Phase 2 готов: добавление iOS unlock, named-event handlers, setPositionState — это локальные изменения внутри PlaybackEngine, не требующие повторного рефактора
</success_criteria>

<output>
После завершения создать `.planning/phases/01-architecture-refactor/01-04-SUMMARY.md` со списком: что владеет PlaybackEngine, какие подписки временные (на удаление в Plan 06), и подтверждение 10-skip ручного теста.
</output>
