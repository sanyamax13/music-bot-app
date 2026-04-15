---
phase: 01-architecture-refactor
plan: 06
type: execute
wave: 6
depends_on: ["01-05"]
files_modified: [index.html]
autonomous: true
requirements: [ARCH-3]
must_haves:
  truths:
    - "MiniPlayer — IIFE-компонент с mount/destroy, подписан отдельно на ['currentTrack'] (full render) и ['isPlaying'] (surgical patch play-icon)"
    - "FullscreenPlayer — IIFE-компонент с mount/destroy, подписан отдельно на ['currentTrack'] (full render), ['isPlaying'] (patch play-icon), ['duration'] (patch total-time), ['fullscreenOpen'] (toggle class + start/stop rAF)"
    - "Ни MiniPlayer, ни FullscreenPlayer НЕ перерисовывают весь innerHTML на isPlaying/duration — эти события патчат конкретные узлы точечно. Полный re-render только на currentTrack."
    - "MiniPlayer и FullscreenPlayer НЕ пишут `fullscreenOpen` напрямую — зовут `Router.openFullscreen()` / `Router.closeFullscreen()`. Router — единственный writer этого ключа."
    - "FullscreenPlayer scrubber не теряет drag-состояние: флаг `dragging` блокирует rAF-обновление `prog.value` пока юзер держит палец"
    - "QueuePanel — IIFE-скелет с mount/destroy (полная функциональность — Phase 6, тут только заготовка)"
    - "Все временные подписки `Store.subscribe(['currentTrack'/'isPlaying'/'duration'])` из Plan 04 удалены — их заменили компоненты"
    - "Wave/Search/Genre/Collection/Profile продолжают играть треки и обновлять UI"
  artifacts:
    - path: "index.html"
      provides: "MiniPlayer, FullscreenPlayer, QueuePanel IIFE-компоненты"
      contains: "const MiniPlayer = (() =>"
  key_links:
    - from: "MiniPlayer.tickProgress"
      to: "PlaybackEngine.getPosition"
      via: "rAF loop while mounted"
      pattern: "PlaybackEngine\\.getPosition"
---

<objective>
Превратить три персистентных overlay-элемента (mini-player, fullscreen player, queue panel) в полноценные IIFE-компоненты со скелетом `mount(root) / render() / destroy()` по ARCHITECTURE §4. На этой стадии:
- MiniPlayer и FullscreenPlayer **полностью** перенимают рендер из временных подписок Plan 04
- QueuePanel — только пустой скелет с mount/render/destroy и subscribe (полная реализация — Phase 6)
- Внешний вид остаётся идентичным pre-refactor (визуальные апгрейды — Phases 5/13)

Purpose: ARCH-3 (12 компонентов с mount/render/destroy) — закрывает 3 из 12. Остальные 9 — в Plan 07.

Output: MiniPlayer, FullscreenPlayer, QueuePanel живут в секции `// ===== COMPONENTS =====`; временные subscribe-блоки из Plan 04 удалены.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/REQUIREMENTS.md
@.planning/research/ARCHITECTURE.md
@index.html
@.planning/phases/01-architecture-refactor/01-05-SUMMARY.md
</context>

<interfaces>
**Component skeleton** (verbatim из ARCHITECTURE §4):

```js
const ComponentName = (() => {
  let root, rafId, unsub;

  function template(state) { return `<div>...</div>`; }

  function render() {
    root.innerHTML = template(/* state args */);
    hydrate();
  }

  function hydrate() { /* attach event handlers via delegation */ }

  function mount(el) {
    root = el;
    unsub = Store.subscribe(['key1','key2'], render);
    render();
    // optional: rAF loop, started here
  }

  function destroy() {
    cancelAnimationFrame(rafId);
    if (unsub) unsub();
    if (root) root.innerHTML = '';
  }

  return { mount, render, destroy };
})();
```

**MiniPlayer** маунтится в `#mini-player` (уже существует в markup строки 439–446). Сейчас он рендерится подписками из Plan 04 — этот план переносит логику в компонент.

**FullscreenPlayer** маунтится в `#full-player` (markup строки 448–474). Сейчас открытие/закрытие управляется `Store.set({ fullscreenOpen })` через `openFullPlayer()/closeFullPlayer()`. Компонент подписывается на `fullscreenOpen` для toggle класса `.open`.

**QueuePanel** mount-точка пока не существует в markup. **Создать** скрытый div `<div id="queue-panel-root" style="display:none"></div>` рядом с `<div id="full-player">` (после строки 474 в markup), и в этот div маунтится QueuePanel-скелет.
</interfaces>

<tasks>

<task type="auto">
  <name>Task 6.1: Реализовать MiniPlayer и FullscreenPlayer как IIFE-компоненты</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (markup mini-player строки 439–446, full-player 448–474; временные subscribe-блоки из Plan 04)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§4 «Component boundaries» — verbatim component skeleton с rAF loop)
  </read_first>
  <action>
1. **MiniPlayer** — добавить в секцию `// ===== COMPONENTS =====`:

```js
const MiniPlayer = (() => {
  let root, unsubTrack, unsubPlaying;

  function templateTrack(t) {
    if (!t) return '';
    return `
      <img id="mp-thumb" src="${Utils.escapeHtml(t.thumb || '')}">
      <div class="mp-info">
        <div id="mp-title">${Utils.escapeHtml(t.title || '')}</div>
        <div id="mp-artist">${Utils.escapeHtml(t.artist || '')}</div>
      </div>
      <div id="mp-play-btn" data-action="toggle">${Store.get('isPlaying') ? '⏸' : '▶'}</div>
    `;
  }

  // Full re-render only on currentTrack change (rare event).
  function renderTrack() {
    const t = Store.get('currentTrack');
    root.style.display = t ? 'flex' : 'none';
    root.innerHTML = templateTrack(t);
    hydrate();
  }

  // Surgical patch on isPlaying change — do NOT rewrite innerHTML.
  function patchPlayIcon() {
    const btn = root.querySelector('#mp-play-btn');
    if (btn) btn.innerText = Store.get('isPlaying') ? '⏸' : '▶';
  }

  function hydrate() {
    // Single delegated click on root: tap on play btn → toggle, tap elsewhere → expand FP.
    // IMPORTANT: MiniPlayer does NOT write fullscreenOpen directly. It calls Router.openFullscreen()
    // to keep Router as the single writer of that key (ARCH-1).
    root.onclick = (e) => {
      if (e.target.closest('[data-action="toggle"]')) {
        e.stopPropagation();
        PlaybackEngine.toggle();
      } else {
        Router.openFullscreen();
      }
    };
  }

  function mount(el) {
    root = el;
    unsubTrack   = Store.subscribe(['currentTrack'], renderTrack);
    unsubPlaying = Store.subscribe(['isPlaying'],    patchPlayIcon);
    renderTrack();
    // Hairline progress is a Phase 4 (MINI-4) concern — not implemented here.
  }

  function destroy() {
    if (unsubTrack)   unsubTrack();
    if (unsubPlaying) unsubPlaying();
    if (root) { root.innerHTML = ''; root.style.display = 'none'; }
  }

  return { mount, destroy };
})();
```

2. **FullscreenPlayer** — добавить в секцию `// ===== COMPONENTS =====`:

```js
const FullscreenPlayer = (() => {
  let root, rafId, unsubTrack, unsubDuration, unsubPlaying, unsubOpen;
  let dragging = false;   // set while user is scrubbing #progress — rAF tick must not overwrite

  // Full template is used ONLY on currentTrack change (rare).
  // isPlaying and position updates NEVER rewrite innerHTML — they patch nodes surgically.
  function templateTrack(t) {
    t = t || {};
    return `
      <div class="fp-handle" data-action="close">▾</div>
      <img id="fp-cover" src="${Utils.escapeHtml(t.thumb || '')}">
      <div class="fp-meta">
        <div class="track-info">
          <h2 id="fp-title">${Utils.escapeHtml(t.title || '')}</h2>
          <p id="fp-artist">${Utils.escapeHtml(t.artist || '')}</p>
        </div>
        <div id="fp-fav-btn" class="icon-btn" style="font-size:26px;" data-action="like">🤍</div>
      </div>
      <div class="fp-progress-wrap">
        <input type="range" id="progress" value="0" min="0" max="100" step="0.1">
        <div class="fp-time">
          <span id="fp-time-cur">0:00</span>
          <span id="fp-time-total">${Utils.fmtTime(Store.get('duration'))}</span>
        </div>
      </div>
      <div class="fp-controls">
        <div data-action="prev">⏮</div>
        <div id="fp-play-btn" data-action="toggle">${Store.get('isPlaying') ? '⏸' : '▶'}</div>
        <div data-action="next">⏭</div>
      </div>
      <div class="fp-feedback">
        <div data-action="dislike">👎</div>
        <div data-action="like-feedback">👍</div>
      </div>
    `;
  }

  function renderTrack() {
    root.innerHTML = templateTrack(Store.get('currentTrack'));
    hydrate();
  }

  // Surgical patches — do NOT rewrite innerHTML. This keeps #progress range
  // focus/drag state, avoids flicker on pause/play, and is cheap on every
  // isPlaying/duration/tick event.
  function patchPlayIcon() {
    const btn = root.querySelector('#fp-play-btn');
    if (btn) btn.innerText = Store.get('isPlaying') ? '⏸' : '▶';
  }
  function patchDuration() {
    const el = root.querySelector('#fp-time-total');
    if (el) el.innerText = Utils.fmtTime(Store.get('duration'));
  }

  function hydrate() {
    root.onclick = (e) => {
      const action = e.target.closest('[data-action]')?.dataset.action;
      if (!action) return;
      switch (action) {
        case 'close':         Router.closeFullscreen();             break;   // Router owns fullscreenOpen
        case 'toggle':        PlaybackEngine.toggle();              break;
        case 'prev':          PlaybackEngine.prev();                break;
        case 'next':          PlaybackEngine.next();                break;
        case 'dislike':       PlaybackEngine.sendManualFeedback('dislike'); break;
        case 'like-feedback': PlaybackEngine.sendManualFeedback('like');    break;
        case 'like':          {
          const t = Store.get('currentTrack');
          if (t) Api.like(t).catch(() => {});
          break;
        }
      }
    };
    // Range scrub — track drag state so rAF tick doesn't fight the user's finger.
    const range = root.querySelector('#progress');
    if (range) {
      range.oninput      = () => PlaybackEngine.seekPct(parseFloat(range.value));
      range.onpointerdown = () => { dragging = true; };
      range.onpointerup   = () => { dragging = false; };
      range.ontouchstart  = () => { dragging = true; };
      range.ontouchend    = () => { dragging = false; };
    }
  }

  function tickProgress() {
    if (!Store.get('fullscreenOpen')) {
      cancelAnimationFrame(rafId);
      rafId = null;
      return;
    }
    const cur  = root.querySelector('#fp-time-cur');
    const prog = root.querySelector('#progress');
    const pos  = PlaybackEngine.getPosition();
    const dur  = PlaybackEngine.getDuration();
    if (cur)  cur.innerText = Utils.fmtTime(pos);
    if (prog && dur && !dragging) prog.value = (pos / dur) * 100;
    rafId = requestAnimationFrame(tickProgress);
  }

  function syncOpen() {
    const open = Store.get('fullscreenOpen');
    root.classList.toggle('open', open);
    if (open && !rafId) tickProgress();
  }

  function mount(el) {
    root = el;
    // Full re-render only on track change. isPlaying / duration patch in place.
    unsubTrack    = Store.subscribe(['currentTrack'],    renderTrack);
    unsubPlaying  = Store.subscribe(['isPlaying'],       patchPlayIcon);
    unsubDuration = Store.subscribe(['duration'],        patchDuration);
    unsubOpen     = Store.subscribe(['fullscreenOpen'],  syncOpen);
    renderTrack();
    syncOpen();
  }

  function destroy() {
    cancelAnimationFrame(rafId);
    if (unsubTrack)    unsubTrack();
    if (unsubPlaying)  unsubPlaying();
    if (unsubDuration) unsubDuration();
    if (unsubOpen)     unsubOpen();
    if (root) root.innerHTML = '';
  }

  return { mount, destroy };
})();
```

3. **Удалить** временные subscribe-блоки из Plan 04 (которые сейчас живут где-то в конце скрипта):
   - `Store.subscribe(['isPlaying'], ...)` блок
   - `Store.subscribe(['currentTrack'], ...)` блок
   - `Store.subscribe(['duration'], ...)` блок
   - rAF-функцию `tickFpProgress` (она встроена в FullscreenPlayer)
   - Функции `openFullPlayer()`/`closeFullPlayer()` остаются как тонкие делегаты до Plan 08 (нужны для inline `onclick` в markup, который чистится только в Plan 07). Они уже переписаны в Plan 05 на `Router.openFullscreen()` / `Router.closeFullscreen()` — менять их в этом плане НЕ надо:
   ```js
   // Phase 05 уже сделал эти делегаты. Здесь они остаются без изменений:
   function openFullPlayer()  { Router.openFullscreen();  }
   function closeFullPlayer() { Router.closeFullscreen(); }
   ```
   Удалятся в Plan 08 после полной чистки markup'а.

4. В секции `// ===== BOOT =====` (или временно — в самом конце скрипта перед `</script>`) добавить mount-вызовы:
```js
MiniPlayer.mount(document.getElementById('mini-player'));
FullscreenPlayer.mount(document.getElementById('full-player'));
```
Поместить эти строки **после** того, как загружены DOM (скрипт в конце body — DOM уже готов) и **после** определения `Router`.
  </action>
  <verify>
    <automated>grep -c "const MiniPlayer = (() =>" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "const MiniPlayer = (() =>" index.html` → `1`
    - `grep -c "const FullscreenPlayer = (() =>" index.html` → `1`
    - `grep -c "MiniPlayer.mount(document.getElementById('mini-player'))" index.html` → `1`
    - `grep -c "FullscreenPlayer.mount(document.getElementById('full-player'))" index.html` → `1`
    - `grep -c "PlaybackEngine.getPosition" index.html` ≥ `1` (внутри FullscreenPlayer.tickProgress)
    - **Router owns overlay flags (ARCH-1):**
      - `grep -c "Router.openFullscreen()" index.html` ≥ `1` (внутри MiniPlayer hydrate)
      - `grep -c "Router.closeFullscreen()" index.html` ≥ `1` (внутри FullscreenPlayer hydrate case 'close')
      - `grep -cE "Store\.set\(\{ ?fullscreenOpen" index.html` — все совпадения должны быть **только** внутри Router IIFE. MiniPlayer и FullscreenPlayer hydrate НЕ должны содержать `Store.set({ fullscreenOpen`.
    - **isPlaying patch вместо innerHTML rerender (предотвращает flicker/drag-loss):**
      - `grep -c "patchPlayIcon" index.html` ≥ `2` (один в MiniPlayer, один в FullscreenPlayer)
      - `grep -cE "Store\.subscribe\(\['isPlaying'\]" index.html` → `2` (по одной подписке в MiniPlayer и FullscreenPlayer, каждая зовёт patchPlayIcon — НЕ renderTrack)
      - В коде FullscreenPlayer: подписка на `['currentTrack']` вызывает renderTrack, подписка на `['isPlaying']` — patchPlayIcon, подписка на `['duration']` — patchDuration. **Единой подписки `['currentTrack', 'isPlaying', 'duration']` быть не должно.**
      - `grep -cE "Store\.subscribe\(\['currentTrack', ?'isPlaying'" index.html` → `0`
    - **Drag-preservation на scrubber:**
      - `grep -c "let dragging" index.html` → `1` (внутри FullscreenPlayer)
      - `grep -c "dragging = true" index.html` ≥ `2` (pointerdown + touchstart)
      - `grep -c "!dragging" index.html` → `1` (внутри tickProgress — не переписывает `prog.value` пока юзер драгает)
    - **Удалены временные подписки Plan 04:**
      - `grep -cE "function tickFpProgress" index.html` → `0`
      - Временные блоки `Store.subscribe(['isPlaying'], ...)`, `Store.subscribe(['currentTrack'], ...)`, `Store.subscribe(['duration'], ...)` из Plan 04 (которые жили в конце скрипта перед Boot'ом) удалены. Оставшиеся subscribe-вызовы с этими ключами должны быть **только** внутри MiniPlayer IIFE и FullscreenPlayer IIFE.
    - Browser smoke (критически важный для fix'а):
      1. Открыть страницу, включить трек → mini-player виден с обложкой/заголовком/артистом
      2. Тап на mini-player → fullscreen открывается с правильным треком
      3. Seek-bar движется сам (rAF tick)
      4. Seek: **удерживать палец на scrubber'е, двигать** → во время drag'а значение НЕ прыгает обратно к позиции трека (dragging flag работает)
      5. **Пауза на полноэкранном плеере**: ⏸ → иконка меняется на ▶ **без перерисовки всего блока** (в DevTools → Elements: во время тапа только `#fp-play-btn` мигает, остальной innerHTML не дёргается; DOM inspector не показывает полный rerender)
      6. ⏭/⏮ работают
      7. 👎/👍 отправляют POST /wave/feedback
      8. 🤍 (пустое сердце) → POST /library/like
      9. ▾ закрывает FP (Router.closeFullscreen → Router.back compatible)
      10. Переключить таб с играющим треком → mini-player по-прежнему виден и реагирует на ⏸/⏵
  </acceptance_criteria>
  <done>MiniPlayer и FullscreenPlayer — IIFE-компоненты с mount/destroy. isPlaying патчит play-icon в месте, не перерисовывая innerHTML. fullscreenOpen пишется только через Router. Scrubber не теряет drag-состояние.</done>
</task>

<task type="auto">
  <name>Task 6.2: Создать QueuePanel-скелет с mount/render/destroy</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (markup overlay секция строки 437–474; узнать куда добавить root для queue panel)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§4 строка про QueuePanel)
  </read_first>
  <action>
1. **В markup** добавить root-контейнер для QueuePanel сразу после `</div>` закрывающего `#full-player` (после строки 474 в pre-refactor, либо после соответствующего места):

```html
<div id="queue-panel" style="display:none"></div>
```

CSS для `#queue-panel` пока не нужен — компонент в Phase 6 добавит свой стиль (bottom-sheet с slide-up анимацией). Сейчас он просто скрыт.

2. **В секцию `// ===== COMPONENTS =====`** добавить QueuePanel-скелет:

```js
const QueuePanel = (() => {
  let root, unsub, unsubOpen;

  function template(state) {
    // Phase 1 stub. Real bottom-sheet UI with History/Now Playing/Up Next sections,
    // SortableJS reorder, swipe-left remove, long-press action sheet — Phase 6.
    const queue = state.queue || [];
    const idx = state.queueIndex;
    if (!queue.length) return `<p class="empty">Очередь пуста</p>`;
    return `
      <div class="qp-list">
        ${queue.map((t, i) => `
          <div class="qp-row${i === idx ? ' is-current' : ''}">
            <span class="qp-title">${Utils.escapeHtml(t.title || '')}</span>
            <span class="qp-artist">${Utils.escapeHtml(t.artist || '')}</span>
          </div>
        `).join('')}
      </div>
    `;
  }

  function render() {
    if (!root) return;
    root.innerHTML = template({
      queue: Store.get('queue'),
      queueIndex: Store.get('queueIndex'),
    });
  }

  function syncOpen() {
    const open = Store.get('queuePanelOpen');
    root.style.display = open ? 'block' : 'none';
  }

  function mount(el) {
    root = el;
    unsub     = Store.subscribe(['queue', 'queueIndex'], render);
    unsubOpen = Store.subscribe(['queuePanelOpen'], syncOpen);
    render();
    syncOpen();
  }

  function destroy() {
    if (unsub)     unsub();
    if (unsubOpen) unsubOpen();
    if (root) root.innerHTML = '';
  }

  return { mount, render, destroy };
})();
```

3. **Mount** в Boot-секции (после MiniPlayer/FullscreenPlayer mount):
```js
QueuePanel.mount(document.getElementById('queue-panel'));
```
  </action>
  <verify>
    <automated>grep -c "const QueuePanel = (() =>" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "const QueuePanel = (() =>" index.html` → `1`
    - `grep -c "id=\"queue-panel\"" index.html` → `1`
    - `grep -c "QueuePanel.mount(document.getElementById('queue-panel'))" index.html` → `1`
    - В DevTools: `Store.set({ queuePanelOpen: true })` → div `#queue-panel` становится видимым; `Store.set({ queuePanelOpen: false })` → скрывается
    - В DevTools: после проигрывания трека `Store.set({ queuePanelOpen: true })` → видна список текущей очереди (Wave-треки)
    - tg.BackButton тест: открыть FP → BackButton показан, открыть queue panel поверх (`Store.set({ queuePanelOpen: true })`) → BackButton по-прежнему показан → нажать BackButton → закроется queue panel (precedence!), FP остаётся открытым → ещё раз BackButton → закроется FP.
  </acceptance_criteria>
  <done>QueuePanel-скелет существует, mount работает, BackButton precedence соблюдён (queuePanel выше fullscreen).</done>
</task>

</tasks>

<threat_model>
| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-09 | Information Disclosure | innerHTML из API-ответов | mitigate | `Utils.escapeHtml` применён ко всем title/artist/thumb перед template literal interpolation |
</threat_model>

<verification>
- Все 5 табов работают
- Mini-player → fullscreen → queue-panel → BackButton precedence работает
- 10 skip'ов всё ещё работают (Plan 04 acceptance не сломан)
</verification>

<success_criteria>
- 3 из 12 компонентов ARCH-3 реализованы: MiniPlayer, FullscreenPlayer, QueuePanel
- Временные подписки из Plan 04 удалены — рендер плеера теперь полностью в компонентах
</success_criteria>

<output>
После завершения создать `.planning/phases/01-architecture-refactor/01-06-SUMMARY.md` со списком реализованных компонентов и подтверждением BackButton precedence теста.
</output>
