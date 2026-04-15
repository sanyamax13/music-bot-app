---
phase: 01-architecture-refactor
plan: 01
subsystem: architecture
tags: [refactor, iife, module-boundaries, config, utils, telegram]
requires: []
provides:
  - "index.html script: 10 section markers (CONFIG..BOOT) in ARCHITECTURE.md §1 order"
  - "Config IIFE: { API, SOURCES, SOURCE_ICONS, GENRES }"
  - "Utils IIFE: { fmtTime, escapeHtml, escapeAttr }"
  - "Tg IIFE: { tg, USER_ID } (expand + setHeaderColor side effects on init)"
  - "TEMP shims bridging old top-level identifiers → IIFE exports (removed plan-by-plan, final cleanup in Plan 08)"
affects: [index.html]
tech_stack:
  added: []
  patterns: ["Revealing IIFE module pattern (shared <script> scope)", "Shim aliases for incremental refactor"]
key_files:
  created: []
  modified:
    - path: index.html
      what: "Inserted section banner comments + extracted Config/Utils/Tg into IIFE, removed top-level declarations, added TEMP shim aliases"
decisions:
  - "Использовать IIFE + единую namespace-shim прослойку, чтобы call-sites не менять в Plan 01. Альтернатива (global replace fetch(${API}) → fetch(${Config.API})) отвергнута — она раздувает диф и теряет атомарность плана."
  - "Оставить `activeSource` / `activeModalSource` / `audioA` / `audioB` / очередь-let'ы на топ-уровне — они UI-state и перенос их в Store это задача Plan 02."
metrics:
  duration: "~7 min"
  completed: "2026-04-15"
  tasks_completed: 2
  files_modified: 1
  lines_before: 1023
  lines_after: 1071
---

# Phase 1 Plan 01: Module Skeleton Summary

Зафиксировали секционный каркас скрипта в `index.html` и вынесли Config / Utils / Telegram-обёртки в IIFE с shim-мостом, сохранив 100% существующего поведения для следующих планов.

## What Was Built

### Task 1.1 — Section banner comments (commit 395cd1b)

В начало `<script>` на строке 498 вставлен header-комментарий со ссылкой на `ARCHITECTURE.md §1` и 10 секционных плейсхолдеров в строго фиксированном порядке:

```
CONFIG → UTILS → TELEGRAM → STORE → API → PLAYBACK ENGINE → FEEDBACK → ROUTER → COMPONENTS → BOOT
```

Этот порядок load-bearing: каждый следующий план заполняет свою секцию, и никто ниже Store не должен объявляться раньше Store. Порядок проверен `grep -n "===== " index.html` — STORE < PLAYBACK ENGINE < COMPONENTS < BOOT.

### Task 1.2 — Config/Utils/Tg IIFE + TEMP shims (commit ca2d663)

1. **CONFIG section** получил `const Config = (() => { … })()` — владеет `API`, `SOURCES`, `SOURCE_ICONS`, всеми 18 объектами `GENRES`.
2. **UTILS section** получил `const Utils = (() => { … })()` — владеет `fmtTime`, `escapeHtml`, `escapeAttr`. Тела функций перенесены один-в-один.
3. **TELEGRAM section** получил `const Tg = (() => { … })()` — внутри IIFE вызываются `tg.expand()` и `tg.setHeaderColor('#0f0f13')` (side effects сохранены); экспорт — `{ tg, USER_ID }`.
4. **Удалено** со старых позиций (бывшие строки 499–553, 1004–1013):
   - `const API = "https://api.vdsmusic.ru:8443";`
   - блок `const tg = window.Telegram.WebApp; tg.expand(); tg.setHeaderColor(...); const USER_ID = ...;`
   - `const SOURCES = [...]`, `const SOURCE_ICONS = {...}`, `const GENRES = [...]`
   - `function fmtTime`, `function escapeHtml`, `function escapeAttr` (внизу скрипта, секция `// ============ UTILS ============` вместе с заголовком удалена)
5. **Оставлено** как было (плановое, уйдёт в Plan 02): `activeSource`, `activeModalSource`, `audioA`/`audioB`/`activeAudio`/`nextAudio`, очередь-let'ы, `waveMood`, `waveLoading`, `waveEnded`, `currentTrackStart`, `lastFeedbackSent`.
6. **TEMP shims block** — сразу под `Tg` IIFE добавлено 9 алиасов:

   ```js
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

   Шимы существуют, чтобы ~30 call-sites ниже (`fetch(\`${API}/wave?user_id=${USER_ID}...\`)`, `escapeHtml(t.title)`, `fmtTime(activeAudio.currentTime)` и т.д.) продолжали работать без изменений. Это позволило удержать Plan 1 атомарным. Шимы будут удаляться плей за плеем по мере миграции call-sites на прямые `Config.*` / `Utils.*` / `Tg.*` — финальная чистка в Plan 08.

## Acceptance Criteria — все пройдены

| Критерий | Результат |
|---|---|
| `grep -c "// ===== CONFIG =====" index.html` | 1 ✓ |
| `grep -c "// ===== STORE =====" index.html` | 1 ✓ |
| `grep -c "// ===== PLAYBACK ENGINE =====" index.html` | 1 ✓ |
| `grep -c "// ===== ROUTER =====" index.html` | 1 ✓ |
| `grep -c "// ===== COMPONENTS =====" index.html` | 1 ✓ |
| `grep -c "// ===== BOOT =====" index.html` | 1 ✓ |
| Порядок секций соответствует `ARCHITECTURE.md §1` | ✓ (STORE → PLAYBACK → COMPONENTS → BOOT) |
| `grep -c "const Config = (() =>" index.html` | 1 ✓ |
| `grep -c "const Utils = (() =>" index.html` | 1 ✓ |
| `grep -c "const Tg = (() =>" index.html` | 1 ✓ |
| `grep -cE '^\s*const API = "https' index.html` | 1 ✓ (только внутри Config IIFE; топ-уровень — `const API = Config.API`) |
| `grep -c "Config.API" index.html` ≥ 1 | 1 ✓ |
| `grep -c "TEMP shims" index.html` | 1 ✓ |
| `wc -l index.html` ≤ 2500 | 1071 ✓ |
| `node --check` на извлечённом script-блоке | SYNTAX OK ✓ |

## Syntax & Safety Checks

- **Script extract → `node --check`:** PASS. JS парсится без ошибок, ни одна идентификаторная коллизия не образовалась (каждый перенесённый символ объявлен ровно дважды: один раз внутри IIFE в локальной области, один раз как топ-уровневая shim-константа — дубликатов в одной области видимости нет).
- **Grep на двойные топ-уровневые декларации** (`^\s{0,8}(const|let|function)\s+(API|tg|USER_ID|SOURCES|SOURCE_ICONS|GENRES|fmtTime|escapeHtml|escapeAttr)\b`) — топ-уровневые (8 пробелов) строки появляются только в блоке shim (9 штук, по одному на идентификатор). Внутри IIFE (12+ пробелов) — по одной декларации. Коллизий нет.
- **Side-effect ordering:** `tg.expand()` и `tg.setHeaderColor('#0f0f13')` выполняются внутри `Tg` IIFE при первом вычислении — то есть в тот же момент, в который раньше выполнялись на топ-уровне (сразу после `const API = ...`). Функционально эквивалентно.

## Deviations from Plan

Нет. План исполнен точно как написан. Единственное микро-уточнение: удалил также заголовок `// ============ UTILS ============` (вместе с телами функций) в нижней части скрипта — это было неявно необходимо, потому что оставленный заголовок без тела вводил бы читателя в заблуждение («почему секция пуста?»). План явно требовал удалить функции; удаление их заголовка — логичное следствие.

## Smoke Test — TODO до Plan 08

Физический smoke-test (открыть Telegram Mini App → проверить Wave / Search / Genre / Collection / Profile) в этой среде сделать нельзя — нет браузера/Telegram клиента. Статические гарантии:

- `node --check` script-блока проходит.
- Ни одна буква пользовательского кода ниже TEMP shims не тронута — call-sites `fetch(\`${API}/...\`)`, `escapeHtml(...)`, `fmtTime(...)`, `${USER_ID}`, `SOURCES.map(...)`, `GENRES.forEach(...)` идентифицируются по shim-алиасам и получают те же значения, что раньше.
- `tg.expand()` и `tg.setHeaderColor('#0f0f13')` выполняются ровно один раз при инициализации IIFE (как и раньше при топ-уровневом выполнении).

**TODO:** финальный ручной smoke-test всех 5 табов + проверка Wave-воспроизведения зафиксирован как gate для Plan 08 (cleanup). Если регрессия проявится раньше — `git bisect` между `395cd1b` (только комментарии, не может ничего сломать) и `ca2d663` (функциональное перемещение) локализует проблему за 1 шаг.

## Known Stubs

Нет. TEMP shims — это не stub, а осознанно временный compatibility layer; стоимость отсрочки миграции call-sites (простота Plan 1, атомарный diff) выше стоимости жить с шимами 7 планов.

## Requirements Progress

- **ARCH-5** (один index.html, inline CSS/JS, только CDN, без сборки) — частично подтверждено: структура не меняет констрейнт (файл всё ещё один, deployment всё ещё `git push` → Pages). Полное подтверждение — после Plan 08.
- **ARCH-6** (≤ 2.5k строк, логические секции согласно ARCHITECTURE §1) — частично: секционная разбивка вставлена, 3 из 10 секций наполнены. Остальные заполняются Plans 02–08. Строк: 1071 / 2500.

## Commits

| Hash | Type | Description |
|---|---|---|
| 395cd1b | docs(01-01) | Insert section banner comments into script |
| ca2d663 | refactor(01-01) | Extract Config/Utils/Tg into IIFE + TEMP shims |

## Self-Check: PASSED

- FOUND: `.planning/phases/01-architecture-refactor/01-01-SUMMARY.md`
- FOUND: commit 395cd1b (`docs(01-01): insert section banner comments into script`)
- FOUND: commit ca2d663 (`refactor(01-01): extract Config/Utils/Tg into IIFE + TEMP shims`)
- FOUND: `const Config = (() =>` / `const Utils = (() =>` / `const Tg = (() =>` in index.html
- FOUND: `TEMP shims` marker in index.html
- Script block passes `node --check`
