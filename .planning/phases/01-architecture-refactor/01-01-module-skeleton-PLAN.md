---
phase: 01-architecture-refactor
plan: 01
type: execute
wave: 1
depends_on: []
files_modified: [index.html]
autonomous: true
requirements: [ARCH-5, ARCH-6]
must_haves:
  truths:
    - "Файл index.html по-прежнему открывается в браузере и проигрывает Wave-трек"
    - "Визуальные секции скрипта обозначены комментариями в порядке Config → Utils → Telegram → Store(stub) → Api(stub) → PlaybackEngine(stub) → Feedback(stub) → Router(stub) → components(stub) → Boot"
    - "Config, Utils и Telegram-обёртки изолированы в IIFE и доступны как Config.*, Utils.*, Tg.*"
  artifacts:
    - path: "index.html"
      provides: "Секционная разметка скрипта + IIFE Config/Utils/Tg"
      contains: "// ===== CONFIG ====="
  key_links:
    - from: "Boot section"
      to: "Config.API"
      via: "global Config namespace"
      pattern: "Config\\.API"
---

<objective>
Внести в `index.html` визуальную секционную разбивку и вынести **только** Config / Utils / Telegram-обёртки в IIFE-модули. Никакого изменения поведения. Это первый кирпич рефактора — следующие плеи будут добавлять новые секции в уже существующий каркас.

Purpose: Зафиксировать порядок секций в скрипте (load-bearing — Store должен быть выше PlaybackEngine выше components, см. ARCHITECTURE §1) и убрать первые источники неявных глобалов (`API`, `tg`, `USER_ID`, `SOURCES`, `SOURCE_ICONS`, `GENRES`, `escapeHtml`, `escapeAttr`, `fmtTime`).

Output: Один отредактированный `index.html` с явными секциями-комментариями и тремя работающими IIFE: `Config`, `Utils`, `Tg`.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/REQUIREMENTS.md
@.planning/research/ARCHITECTURE.md
@.planning/research/STACK.md
@index.html
</context>

<interfaces>
Текущие глобальные идентификаторы в `index.html` (строки 499–1020), которые этот план переносит в IIFE:

```js
// строка 499
const API = "https://api.vdsmusic.ru:8443";

// строки 501–504
const tg = window.Telegram.WebApp;
tg.expand();
tg.setHeaderColor('#0f0f13');
const USER_ID = (tg.initDataUnsafe && tg.initDataUnsafe.user && tg.initDataUnsafe.user.id) || 0;

// строки 517–532 — SOURCES, SOURCE_ICONS
// строки 534–553 — GENRES
// строки 1004–1013 — fmtTime, escapeHtml, escapeAttr
```

Целевые модульные обёртки (новые):

```js
// ===== CONFIG =====
const Config = (() => {
  const API = "https://api.vdsmusic.ru:8443";
  const SOURCES = [/* перенести как есть */];
  const SOURCE_ICONS = {/* перенести как есть */};
  const GENRES = [/* перенести как есть */];
  return { API, SOURCES, SOURCE_ICONS, GENRES };
})();

// ===== UTILS =====
const Utils = (() => {
  function fmtTime(s) { /* перенести тело из строк 1004–1009 */ }
  function escapeHtml(s) { /* перенести тело из строк 1010–1012 */ }
  function escapeAttr(s) { /* перенести тело из строки 1013 */ }
  return { fmtTime, escapeHtml, escapeAttr };
})();

// ===== TELEGRAM =====
const Tg = (() => {
  const tg = window.Telegram.WebApp;
  tg.expand();
  tg.setHeaderColor('#0f0f13');
  const USER_ID = (tg.initDataUnsafe && tg.initDataUnsafe.user && tg.initDataUnsafe.user.id) || 0;
  return { tg, USER_ID };
})();
```
</interfaces>

<tasks>

<task type="auto">
  <name>Task 1.1: Вставить полные секционные комментарии-плейсхолдеры в скрипт</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (полностью, особенно строки 498–1021 — текущий `<script>` блок)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§1 «Module boundaries inside one HTML file», особенно таблица модулей со строками-якорями)
  </read_first>
  <action>
В начале блока `<script>` (сразу после открывающего тега `<script>` на строке 498) вставить **точно этот** баннер секций, чтобы зафиксировать обязательный порядок загрузки модулей:

```
// =====================================================================
// VDS Music — single-file modular layout (Phase 1 refactor)
// Sections must appear in this order — every module below Store may
// reference Store; PlaybackEngine must be defined before any component.
// See .planning/research/ARCHITECTURE.md §1.
// =====================================================================

// ===== CONFIG =====
// (filled in Plan 01)

// ===== UTILS =====
// (filled in Plan 01)

// ===== TELEGRAM =====
// (filled in Plan 01)

// ===== STORE =====
// (filled in Plan 02)

// ===== API =====
// (filled in Plan 03)

// ===== PLAYBACK ENGINE =====
// (filled in Plan 04)

// ===== FEEDBACK =====
// (filled in Plan 04 — minimal stub; full queue is Phase 2)

// ===== ROUTER =====
// (filled in Plan 05)

// ===== COMPONENTS =====
// (filled in Plans 06–07)

// ===== BOOT =====
// (filled in Plan 08)
```

ВАЖНО: ниже этого баннера ОСТАВИТЬ существующий код index.html без изменений, кроме того, что описано в задачах 1.2 и 1.3. Не удалять и не переставлять ничего, что не указано явно.
  </action>
  <verify>
    <automated>grep -c "// ===== CONFIG =====" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "// ===== CONFIG =====" index.html` → `1`
    - `grep -c "// ===== STORE =====" index.html` → `1`
    - `grep -c "// ===== PLAYBACK ENGINE =====" index.html` → `1`
    - `grep -c "// ===== ROUTER =====" index.html` → `1`
    - `grep -c "// ===== COMPONENTS =====" index.html` → `1`
    - `grep -c "// ===== BOOT =====" index.html` → `1`
    - Порядок секций в файле соответствует ARCHITECTURE.md §1 (проверить `grep -n "===== " index.html` → STORE раньше PLAYBACK, PLAYBACK раньше COMPONENTS, COMPONENTS раньше BOOT)
  </acceptance_criteria>
  <done>В скрипте есть 11 секционных комментариев в правильном порядке; код по-прежнему работает (страница открывается, Wave грузит треки).</done>
</task>

<task type="auto">
  <name>Task 1.2: Перенести Config, Utils, Tg в IIFE и удалить топ-уровневые объявления</name>
  <files>index.html</files>
  <read_first>
    - /root/music-bot-frontend/index.html (строки 499–504, 517–553, 1004–1013)
    - /root/music-bot-frontend/.planning/research/ARCHITECTURE.md (§1 пример Store IIFE — тот же шаблон применяется к Config/Utils/Tg)
  </read_first>
  <action>
1. **В секцию `// ===== CONFIG =====`** вставить:
```js
const Config = (() => {
  const API = "https://api.vdsmusic.ru:8443";
  const SOURCES = [
    { id: 'all', name: 'Всё' },
    { id: 'library', name: 'Библиотека' },
    { id: 'youtube', name: 'YouTube' },
    { id: 'soundcloud', name: 'SoundCloud' },
    { id: 'jamendo', name: 'Jamendo' },
  ];
  const SOURCE_ICONS = {
    local: '🎵', library: '🎵',
    youtube: '▶️',
    soundcloud: '☁️',
    jamendo: '🎶',
  };
  const GENRES = [
    /* перенести ВСЕ 18 объектов из строк 535–552 index.html без изменений */
  ];
  return { API, SOURCES, SOURCE_ICONS, GENRES };
})();
```

2. **В секцию `// ===== UTILS =====`** вставить:
```js
const Utils = (() => {
  function fmtTime(s) {
    if (!s || !isFinite(s)) return '0:00';
    const m = Math.floor(s / 60);
    const sec = Math.floor(s % 60);
    return `${m}:${sec < 10 ? '0' : ''}${sec}`;
  }
  function escapeHtml(s) {
    return String(s).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
  }
  function escapeAttr(s) { return String(s).replace(/'/g, "\\'"); }
  return { fmtTime, escapeHtml, escapeAttr };
})();
```

3. **В секцию `// ===== TELEGRAM =====`** вставить:
```js
const Tg = (() => {
  const tg = window.Telegram.WebApp;
  tg.expand();
  tg.setHeaderColor('#0f0f13');
  const USER_ID = (tg.initDataUnsafe && tg.initDataUnsafe.user && tg.initDataUnsafe.user.id) || 0;
  return { tg, USER_ID };
})();
```

4. **Удалить** старые топ-уровневые объявления:
   - `const API = "https://api.vdsmusic.ru:8443";` (строка 499)
   - `const tg = window.Telegram.WebApp;` блок строки 501–504
   - `const SOURCES = [...]` строки 517–523
   - `const SOURCE_ICONS = {...}` строки 524–529
   - `let activeSource = 'all';` и `let activeModalSource = 'all';` ОСТАВИТЬ — это UI-состояние, его уберём в Plan 02
   - `const GENRES = [...]` строки 534–553
   - `function fmtTime(s) {...}` строки 1004–1009
   - `function escapeHtml(s) {...}` строки 1010–1012
   - `function escapeAttr(s) {...}` строка 1013

5. **Создать compatibility shim** (НЕ убирать в этом плане — следующие плеи постепенно отрефакторят call sites). Сразу под `Tg` IIFE добавить:
```js
// === TEMP shims for Plans 02–08 — will be removed plan by plan ===
const API = Config.API;
const SOURCES = Config.SOURCES;
const SOURCE_ICONS = Config.SOURCE_ICONS;
const GENRES = Config.GENRES;
const tg = Tg.tg;
const USER_ID = Tg.USER_ID;
const fmtTime = Utils.fmtTime;
const escapeHtml = Utils.escapeHtml;
const escapeAttr = Utils.escapeAttr;
```
Это «мост»: старые call sites (`fetch(${API}/wave...)`, `${USER_ID}`, `escapeHtml(...)`) продолжают работать ровно как раньше, и Plan 1 не ломает функционал. Шим уйдёт в Plan 08.
  </action>
  <verify>
    <automated>grep -c "const Config = (() =>" /root/music-bot-frontend/index.html</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "const Config = (() =>" index.html` → `1`
    - `grep -c "const Utils = (() =>" index.html` → `1`
    - `grep -c "const Tg = (() =>" index.html` → `1`
    - `grep -cE "^\s*const API = \"https" index.html` → `1` (один — внутри Config IIFE; топ-уровневое объявление удалено и заменено shim'ом `const API = Config.API`)
    - `grep -c "Config.API" index.html` ≥ `1`
    - `grep -c "TEMP shims" index.html` → `1`
    - Manual smoke: открыть index.html в браузере (или Telegram WebApp) — Wave-таб грузит треки, тап на трек начинает воспроизведение, поиск работает, лайк отправляется (см. DevTools Network).
  </acceptance_criteria>
  <done>Config/Utils/Tg существуют как IIFE; топ-уровневые `const API`, `const tg`, `const USER_ID`, `const SOURCES`, `const SOURCE_ICONS`, `const GENRES`, `function fmtTime`, `function escapeHtml`, `function escapeAttr` удалены и заменены shim-присваиваниями из IIFE; страница работает.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Telegram WebView → backend api.vdsmusic.ru | user_id из `initDataUnsafe` без HMAC (осознанный trade-off, PROJECT.md Key Decision) |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-01 | Tampering | `Config.API` | accept | Refactor не меняет URL и не открывает новых каналов; backend frozen в этот milestone |
| T-01-02 | Spoofing | `Tg.USER_ID` | accept | Зафиксировано в PROJECT.md Key Decisions как осознанный trade-off; HMAC — отдельный milestone (BLK-8) |
</threat_model>

<verification>
- Все acceptance criteria задач 1.1 и 1.2 выполнены
- `wc -l index.html` ≤ 2500 (после Plan 1 будет ~1090–1110)
- В DevTools console при загрузке нет ошибок
- Wave / Поиск / Жанры / Коллекция / Профиль — все 5 табов открываются
</verification>

<success_criteria>
- Phase 1 каркас секций виден в outline скрипта
- Config / Utils / Tg — IIFE с публичным API
- Существующий функционал не сломан (визуально и функционально идентично pre-refactor)
- Shim-блок задокументирован как временный
</success_criteria>

<output>
После завершения создать `.planning/phases/01-architecture-refactor/01-01-SUMMARY.md` со списком вынесенных идентификаторов и подтверждением smoke-теста.
</output>
