---
phase: 01-architecture-refactor
plan: 05
type: execute
wave: 5
depends_on: ["01-04"]
files_modified: [index.html]
autonomous: true
requirements: [ARCH-4]
must_haves:
  truths:
    - "Существует Router IIFE с методами go(screen) и back()"
    - "Router — единственный writer ключа `screen` в Store"
    - "tg.BackButton показывается/скрывается из одного места (Router.updateBackButton)"
    - "BackButton precedence соблюдён: queuePanelOpen → fullscreenOpen → screen stack → tg.close()"
    - "Все 5 табов переключаются через Router.go и продолжают работать как раньше"
  artifacts:
    - path: "index.html"
      provides: "Router IIFE + state-machine экранов"
      contains: "const Router = (() =>"
  key_links:
    - from: "BottomNav (inline onclick switchTab)"
      to: "Router.go"
      via: "function call"
      pattern: "Router\\.go\\("
    - from: "tg.BackButton"
      to: "Router.back"
      via: "tg.BackButton.onClick"
      pattern: "tg\\.BackButton\\.onClick\\(.*back"
---

<objective>
Внедрить `Router` IIFE как state-машину экранов. Перевести существующий `switchTab(tab, el)` на `Router.go(screen)`. Подключить `tg.BackButton` с правилом приоритета: сначала закрывается queue-panel, затем full-screen player, затем стек экранов, в самом конце — `tg.close()`. Полноэкранный плеер становится overlay (флаг `fullscreenOpen`), а не отдельным экраном — чтобы сохранить состояние воспроизведения при переключении между табами с открытым плеером.

Purpose: ARCH-4 целиком. Также готовит почву для Plans 06 (FullscreenPlayer как overlay) и Phase 6 (QueuePanel как overlay с тем же precedence-правилом).

Output: Router IIFE; 5 табов переключаются через `Router.go`; BackButton показывается/скрывается из одного места.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/REQUIREMENTS.md
@.planning/research/ARCHITECTURE.md
@index.html
@.planning/phases/01-architecture-refactor/01-04-SUMMARY.md
</context>

<interfaces>
**Целевой Router** (verbatim из ARCHITECTURE §3 с адаптацией под текущие 5 табов: wave / search / genres / collection / profile):

```js
const Router = (() => {
  const stack = [];

  function go(screen, opts = {}) {
    const prev = Store.get('screen');
    if (opts.push !== false && prev !== screen) stack.push(prev);
    Store.set({ screen });
    updateBackButton();
  }

  function back() {
    // Precedence order (must NOT change without updating ARCHITECTURE.md §3):
    //   1. queuePanelOpen → close queue panel
    //   2. fullscreenOpen → close fullscreen
    //   3. stack non-empty → pop screen
    //   4. otherwise → tg.close()
    if (Store.get('queuePanelOpen')) { Store.set({ queuePanelOpen: false }); return; }
    if (Store.get('fullscreenOpen')) { Store.set({ fullscreenOpen: false }); return; }
    if (stack.length) { Store.set({ screen: stack.pop() }); updateBackButton(); return; }
    Tg.tg.close();
  }

  function updateBackButton() {
    const canGoBack =
         Store.get('fullscreenOpen')
      || Store.get('queuePanelOpen')
      || stack.length > 0;
    if (canGoBack) Tg.tg.BackButton.show();
    else           Tg.tg.BackButton.hide();
  }

  // === Overlay mutators — Router is the ONLY writer of fullscreenOpen / queuePanelOpen ===
  // Components (MiniPlayer, FullscreenPlayer, QueuePanel) MUST call these instead of
  // writing Store.set({ fullscreenOpen: ... }) directly. This enforces ARCH-1
  // "exactly one module writes each key" for the overlay flags.
  function openFullscreen()  { Store.set({ fullscreenOpen: true });  }
  function closeFullscreen() { Store.set({ fullscreenOpen: false }); }
  function openQueuePanel()  { Store.set({ queuePanelOpen: true });  }
  function closeQueuePanel() { Store.set({ queuePanelOpen: false }); }

  // Wire BackButton once
  Tg.tg.BackButton.onClick(back);

  // React to overlay flag changes from other modules
  Store.subscribe(['fullscreenOpen', 'queuePanelOpen'], updateBackButton);

  return { go, back, updateBackButton, openFullscreen, closeFullscreen, openQueuePanel, closeQueuePanel };
})();
```

**Текущий switchTab (строки 556–567 в pre-refactor):**
```js
function switchTab(tab, el) {
    document.querySelectorAll('.tab-view').forEach(v => v.classList.remove('active'));
    document.getElementById('tab-' + tab).classList.add('active');
    document.querySelectorAll('.nav-item').forEach(n => n.classList.remove('active'));
    el.classList.add('active');
    if (tab === 'wave' && !document.getElementById('wave-list').innerHTML) { loadMoods(); loadWave(true); }
    if (tab === 'collection') loadCollection();
    if (tab === 'profile') loadProfile();
}
```
Inline onclick'и в HTML markup (строки 480–484): `onclick="switchTab('wave', this)"` и т.п.
Кнопки fullscreen (строки 439, 449): `onclick="openFullPlayer()"`, `onclick="closeFullPlayer()"`.
</interfaces>

<tasks>

<task type="auto">
  <name>Task 5.1: Создать Router IIFE и подписать его на screen/fullscreenOpen/queuePanelOpen</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (секция `// ===== ROUTER =====` после Plan 04, текущий switchTab строки 556–567, openFullPlayer/closeFullPlayer строки 942–943)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§3 «Routing» — verbatim Router IIFE и правило precedence)
  </read_first>
  <action>
В секцию `// ===== ROUTER =====` вставить ровно тот код Router IIFE, что приведён в `<interfaces>`. Подтверждение начального состояния: `Store.get('screen')` уже инициализирован в `'wave'` ещё в Plan 02 (см. blob default). Никаких дополнительных Store.set перед Router не нужно.
  </action>
  <verify>
    <automated>grep -c "const Router = (() =>" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "const Router = (() =>" index.html` → `1`
    - `grep -c "Tg.tg.BackButton.onClick(back)" index.html` → `1`
    - `grep -cE "Store\.subscribe\(\['fullscreenOpen', ?'queuePanelOpen'\]" index.html` → `1`
    - `grep -c "queuePanelOpen.*fullscreenOpen.*stack" index.html` ≥ `0` (комментарий-документация precedence; не критично, но желательно)
    - `grep -c "function openFullscreen()" index.html` → `1`
    - `grep -c "function closeFullscreen()" index.html` → `1`
    - `grep -c "function openQueuePanel()" index.html` → `1`
    - `grep -c "function closeQueuePanel()" index.html` → `1`
    - В DevTools console: `typeof Router.go === 'function'` → `true`
    - В DevTools console: `typeof Router.openFullscreen === 'function' && typeof Router.closeFullscreen === 'function'` → `true`
  </acceptance_criteria>
  <done>Router IIFE существует, BackButton подключён, подписка на overlay-флаги работает, экспортируются overlay-мутаторы для компонентов.</done>
</task>

<task type="auto">
  <name>Task 5.2: Перевести switchTab/openFullPlayer/closeFullPlayer на Router и подписки</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (switchTab, openFullPlayer, closeFullPlayer, HTML onclick'и)
  </read_first>
  <action>
1. **Заменить тело `switchTab`** на тонкий делегат + screen-driven render через подписку:

```js
function switchTab(tab, el) {
  Router.go(tab, { push: false });   // tabs не пушатся в стек — это «root» уровень
}
```

2. Подписаться на изменение `screen` и переключать DOM:

```js
// === Screen visibility sync (will be replaced by component mount/unmount in Plan 07) ===
Store.subscribe(['screen'], (state) => {
  const s = state.screen;
  document.querySelectorAll('.tab-view').forEach(v => v.classList.remove('active'));
  const tabEl = document.getElementById('tab-' + s);
  if (tabEl) tabEl.classList.add('active');
  document.querySelectorAll('.nav-item').forEach(n => {
    n.classList.toggle('active', n.dataset.tab === s);
  });
  // Lazy-load existing screens (preserves current behavior)
  if (s === 'wave' && !document.getElementById('wave-list').innerHTML) { loadMoods(); loadWave(true); }
  if (s === 'collection') loadCollection();
  if (s === 'profile')    loadProfile();
});
```

3. **Заменить openFullPlayer/closeFullPlayer** на тонкие делегаты в Router (Router — единственный writer ключа `fullscreenOpen`):
```js
function openFullPlayer()  { Router.openFullscreen();  }
function closeFullPlayer() { Router.closeFullscreen(); }
```
Это — legacy-глобалы для inline `onclick` из markup'а. Компоненты (MiniPlayer в Plan 06, FullscreenPlayer в Plan 06) ОБЯЗАНЫ звать `Router.openFullscreen()` / `Router.closeFullscreen()` напрямую, а не `Store.set({ fullscreenOpen })`. Это enforce'ит ARCH-1 «один writer на ключ».

4. Подписаться на `fullscreenOpen` для синхронизации DOM:
```js
// === Fullscreen overlay visibility sync (will be replaced by FullscreenPlayer component in Plan 06) ===
Store.subscribe(['fullscreenOpen'], (state) => {
  const fp = document.getElementById('full-player');
  if (!fp) return;
  fp.classList.toggle('open', state.fullscreenOpen);
});
```

5. **Сделать первоначальный вызов** `Router.go('wave', { push: false })` либо просто `Store.set({ screen: 'wave' })` после всех подписок — чтобы навешать активный класс на BottomNav. Поместить в самый конец скрипта, но **до** `loadMoods()/loadWave(true)/setupSearchTab()/setupSearchModal()/renderGenres()` initial-вызовов, чтобы подписки были навешаны.

6. **Не трогать** `data-tab="wave"` атрибуты в markup — они уже есть на `.nav-item` (строки 480–484) и используются строкой `n.dataset.tab === s` выше.

7. Проверить: `tg.BackButton` после открытия FP должна показаться, после закрытия — скрыться.
  </action>
  <verify>
    <automated>grep -cE "function switchTab\(tab, el\) \{[^}]*Router\.go" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "Router.go(tab" index.html` → `1`
    - `grep -cE "function openFullPlayer\(\) +\{ +Router\.openFullscreen" index.html` → `1`
    - `grep -cE "function closeFullPlayer\(\) +\{ +Router\.closeFullscreen" index.html` → `1`
    - `grep -cE "Store\.subscribe\(\['screen'\]" index.html` → `1`
    - `grep -cE "Store\.subscribe\(\['fullscreenOpen'\]" index.html` → `1`
    - **One-writer audit для `screen`:** `grep -c "Store.set({ screen:" index.html` — каждое совпадение должно быть **только внутри Router IIFE**. Допустимые строки: `Store.set({ screen });` внутри `Router.go` и `Store.set({ screen: stack.pop() })` внутри `Router.back`. Проверить через `grep -n` что других нет.
    - **One-writer audit для `fullscreenOpen`:** все `Store.set({ fullscreenOpen:` присвоения должны быть **только внутри Router IIFE** (в функциях `openFullscreen`, `closeFullscreen` и в `back()` при queuePanelOpen/fullscreenOpen precedence). Проверка: `grep -n "Store.set({ fullscreenOpen" index.html` — все совпадения между `const Router = (() =>` и соответствующим `})();`. Компоненты MiniPlayer/FullscreenPlayer НЕ должны содержать `Store.set({ fullscreenOpen` — только `Router.openFullscreen()` / `Router.closeFullscreen()`. Аналогично для `queuePanelOpen`.
    - Browser smoke: открыть страницу → видна Wave (трансформация подписки) → тапнуть Поиск → переключение → tg.BackButton НЕ показывается (мы на root-уровне табов) → тапнуть Wave → играть трек → тап на mini-player → fullscreen открывается → tg.BackButton ПОЯВЛЯЕТСЯ → нажать BackButton (или ▾) → fullscreen закрывается → BackButton ИСЧЕЗАЕТ → переключить таб с открытым FP → fullscreen остаётся открытым (overlay не привязан к screen) → переключить таб обратно → плеер по-прежнему играет.
  </acceptance_criteria>
  <done>Router владеет навигацией; BackButton управляется из одного места; полноэкранный плеер — overlay, переживающий смену таба.</done>
</task>

</tasks>

<threat_model>
| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-08 | Denial of Service | BackButton race | accept | Подписка на flag changes идемпотентна (`show()`/`hide()` сами по себе идемпотентны) |
</threat_model>

<verification>
- Все 5 табов переключаются
- BackButton появляется только когда есть что закрыть (FP open / queue panel open / стек не пуст)
- FP остаётся открытым при смене таба
- Plan 04 acceptance не сломан (10 skip'ов всё ещё работают)
</verification>

<success_criteria>
- ARCH-4 закрыт
- Подготовлено для FullscreenPlayer-компонента (Plan 06): он будет mount'иться один раз и реагировать на `fullscreenOpen`
</success_criteria>

<output>
После завершения создать `.planning/phases/01-architecture-refactor/01-05-SUMMARY.md` с подтверждением precedence-теста (BackButton при FP open / FP close / tab switch с открытым FP).
</output>
