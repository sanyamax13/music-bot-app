# VDS Music — Roadmap

**Milestone:** Redesign v1 — гибридный Яндекс/Spotify UX
**Source:** `REQUIREMENTS.md` + `research/{FEATURES,ARCHITECTURE,STACK,PITFALLS}.md`
**Last updated:** 2026-04-15 (Phase 1 Plan 08 — Boot() entry-point + final shim cleanup; 1564 lines; ARCH-1/3/4/5/6 closed; Task 8.2 regression smoke-test PENDING USER VERIFICATION)

## Принципы последовательности

1. **Сначала фундамент, потом фичи.** Архитектура и аудио-ядро раньше визуала — иначе редизайн ляжет на сломанные абстракции.
2. **Pitfalls критичного уровня идут в Phase 2.** iOS autoplay lock и MediaSession positionState блокируют восприятие качества — без них редизайн выглядит «игрушечным» независимо от стилей.
3. **Gesture handler — единый модуль (Phase 3),** потому что 9 разных P1-фич зависят от него (FEATURES §10).
4. **Компонентные фазы (4–11) выстроены по зависимостям,** не по важности: mini-player раньше fullscreen (fullscreen ссылается на «свёрнутое состояние»), queue после player (queue открывается из FP), feedback после queue (long-press action-sheet нужен в очереди тоже).
5. **Визуальный pass (Phase 13) — в конце.** Дизайн-токены и typography полируются когда структура зафиксирована, чтобы не переделывать 3 раза.
6. **QA / perf / device matrix (Phase 14) — финальный gate.** Acceptance по всем PERF-* и ручная проверка на iOS Telegram + бюджетном Android.

---

## Phase 1: Architecture Refactor (фундамент)

**Goal:** Перестроить `index.html` в модульную структуру со `Store` + `PlaybackEngine` + компонентами, не ломая текущий функционал.
**Covers:** ARCH-1, ARCH-2, ARCH-3, ARCH-4, ARCH-5, ARCH-6
**Depends on:** —
**Scope:**
- Разнести файл на секции: `<style>` / skeleton markup / Config / Utils / Telegram / Store / Api / PlaybackEngine / Feedback / Router / components / Boot
- `Store` IIFE с key-scoped `subscribe`, один владелец на ключ (см. ARCHITECTURE §2)
- `PlaybackEngine` IIFE — единственный владелец `audioA`/`audioB`, очереди, переключений
- `Api` wrapper с дедупом fetch-ов и error-envelope
- `Router` — state-machine экранов + `tg.BackButton` precedence
- Компоненты как IIFE-модули c `mount/render/destroy`, subscribe по ключам
**Acceptance:**
- Весь текущий функционал (Wave/Search/Genre/Library/Profile) работает как раньше
- `grep Store.set` показывает ровно одного писателя на каждый key
- Никаких `window.*` глобалов для состояния
- Файл ≤ 2.5k строк, визуальная секционная разбивка видна в outline
**Risks:** Регрессия в playback — митигация: feature-parity тест после каждого модуля перед следующим
**Est:** L

## Phase 2: Audio Core & MediaSession Hardening

**Goal:** Закрыть критические pitfalls воспроизведения (iOS autoplay, двойные skip, лживый scrubber на локскрине).
**Covers:** AUDIO-1..7, FP-10
**Depends on:** Phase 1 (`PlaybackEngine`)
**Scope:**
- iOS WKWebView autoplay unlock (silent-mp3 на первом тапе, флаг `audioUnlocked`, запрет init из non-gesture путей до unlock)
- `addEventListener('ended', …)` + `removeEventListener` с named handlers на обоих буферах; обёртка `_ended = fn`
- `mediaSession.setPositionState()` throttled ~1Hz + на `seeked`/`ratechange`
- Полный набор actions: play, pause, nexttrack, previoustrack, seekbackward (15s), seekforward (15s), seekto
- `mediaSession.metadata` с artwork обновляется на каждый трек
- `audio.crossOrigin = "anonymous"` + проверка CORS (для Phase 5 dominant-color); fallback-хэш-палитра если заблокировано
- `error`/`stalled`/`waiting`-хендлеры с одноразовым retry и skip c toast
**Acceptance:**
- На iOS Telegram первый тап Wave запускает трек без `NotAllowedError`
- 10 подряд skip'ов → `queueIndex` продвигается ровно на 10
- Android lock-screen scrubber движется и реагирует на seek
- Один протухший `/stream/{id}` → один toast, очередь идёт дальше
**Risks:** Регрессия preload/gapless — ручной тест переходов 20 треков подряд
**Est:** M

## Phase 3: Gesture Foundation

**Goal:** Единый модуль жестов, от которого зависят 9 P1-фич.
**Covers:** GEST-1
**Depends on:** Phase 1
**Scope:**
- `Gestures` IIFE: `onSwipe(el, {dir, threshold, onStart, onMove, onEnd})`, `onLongPress`, `onDoubleTap`, `onPullToRefresh`
- Touch events (не PointerEvents — iOS Telegram WebView разное поведение), учёт `passive: true` где можно
- Учёт конфликта вертикальный scroll vs horizontal swipe: первый `move` решает направление
- Haptic hook через `tg.HapticFeedback` на срабатывании long-press
**Acceptance:**
- Демо: swipe влево/вправо на див, swipe-down, long-press, double-tap, pull-to-refresh — все работают изолированно
- Модуль ~120 LOC, без зависимостей
**Est:** S

## Phase 4: Mini-Player Polish

**Goal:** Persistent mini-player с жестами, прогрессом и heart-button.
**Covers:** MINI-1..6
**Depends on:** Phase 1 (компонент), Phase 2 (buffering events), Phase 3 (swipes)
**Scope:**
- Всегда mounted, виден когда `currentTrack != null`
- Hairline progress (rAF, перерисовка только пока виден)
- Swipe-up / tap = expand FP; swipe left/right = prev/next
- Heart-button с optimistic UI (FB-1 ставится в Phase 8, но hook'ается тут)
- Buffering indicator: spinner при `waiting`, play-icon при `canplay`
**Acceptance:**
- Switch между экранами (Home ↔ Library ↔ Search ↔ Profile) не останавливает и не прячет плеер
- Прогресс обновляется плавно на 60 fps
- Все жесты работают
**Est:** M

**Plans:** 1/1 plans complete

Plans:
- [x] 04-01-PLAN.md — Heart-button, buffering spinner, likedIds population, FP heart sync

## Phase 5: Fullscreen Player Redesign

**Goal:** Визуальный и тактильный апгрейд fullscreen-плеера до уровня Spotify/Яндекс.
**Covers:** FP-1..9
**Depends on:** Phase 2 (CORS/artwork), Phase 3 (swipe-down, swipe lr, double-tap), Phase 4 (свёрнутое состояние)
**Scope:**
- Большая обложка ~70% ширины, квадрат
- Dominant-color фон через Canvas `getImageData` (fallback — хэш-палитра)
- Прогресс-скраббер с touch-target ≥ 44 px, время
- Swipe-down = collapse, swipe lr на обложке = prev/next, double-tap = like
- Queue-peek «Далее: …» внизу
- Tap по имени артиста → Search с prefilled query
- MediaSession metadata и artwork обновляются здесь тоже
**Acceptance:**
- Все жесты работают синхронно с BackButton
- Фон меняется ≤ 300 мс на смене трека
- Визуальное сравнение со скринами Яндекс/Spotify — паритет P1-пунктов
**Est:** L

**Plans:** 2 plans

Plans:
- [ ] 05-01-PLAN.md — CSS + template foundation (cover resize, scrubber touch-target, heart overlay, queue-peek, artist tap)
- [ ] 05-02-PLAN.md — JS behavior (double-tap like, queue-peek subscription, artist tap-to-search, SearchScreen API)

## Phase 6: Queue Bottom Sheet

**Goal:** Просмотр и управление очередью с drag-to-reorder.
**Covers:** Q-1..5, GEST-2 (long-press action sheet)
**Depends on:** Phase 3 (long-press, drag), Phase 5 (открытие из FP)
**Scope:**
- Bottom-sheet overlay с секциями History / Now Playing / Up Next
- SortableJS из CDN для drag-to-reorder, только в non-Wave контекстах; Wave queue — read-only
- Swipe-left на row = remove, tap = jump
- Long-press → ActionSheet {Like/Dislike/Play next/Add to queue/Share/Search artist}
- In-memory history (последние 50)
- BackButton precedence: queuePanel выше fullscreen
**Acceptance:**
- Reorder в очереди Library/Search работает, Wave ordering не меняется
- History отматывается вверх
- Конфликт swipe-to-remove vs drag-handle разрешён (handle слева)
**Est:** M

## Phase 7: Wave Redesign

**Goal:** Доделать Wave до уровня Яндекс-«Моей волны» визуально и тактильно.
**Covers:** WAVE-1..7
**Depends on:** Phase 2 (auto-skip на dislike), Phase 3 (pull-to-refresh), Phase 4/5 (плеер готов)
**Scope:**
- Sticky mood-chips с emoji, активное состояние, градиент шапки на смене mood
- Pulse-индикатор активного трека
- Skeleton rows < 200 мс
- Auto-skip после dislike (связка с AUDIO/Feedback)
- Pull-to-refresh = reseed `/wave`
- Auto-extend порог ≤ 3 оставшихся трека
- Skip < 30s = `action:skip`, ≥ 30s = `action:finish` с `elapsed`
**Acceptance:**
- Непрерывное прослушивание ≥ 30 мин без «конца очереди»
- Feedback-события видны в network log с корректным `elapsed`
- Смена mood визуально ощущается
**Est:** M

## Phase 8: Feedback UX (haptics, optimistic, undo, swipes)

**Goal:** Дать пользователю ощущение мгновенной реакции на лайк/дизлайк.
**Covers:** GEST-3, FB-1..3, TG-3
**Depends on:** Phase 3 (swipes), Phase 6 (action sheet — уже есть)
**Scope:**
- Optimistic heart fill до POST, rollback на error
- Undo snackbar 3 сек для dislike — фактический POST откладывается
- Haptic на like/dislike/play/pause/expand/collapse/mood change
- Swipe-right row = quick like, swipe-left row = quick dislike в Wave/search/genre (не конфликтует с queue swipe-to-remove)
**Acceptance:**
- Heart заполняется до ответа сети; при искусственной ошибке — rollback
- Undo работает: ничего не POST'ится если нажато «Отменить»
- Haptic срабатывает на iOS Telegram где SDK поддерживает
**Est:** S

## Phase 9: Library Upgrade

**Goal:** Сделать библиотеку лайков реально usable за пределами 50 треков.
**Covers:** LIB-1..4
**Depends on:** Phase 4/5 (плеер), Phase 6 (action sheet)
**Scope:**
- Шапка с count + total duration, Play all / Shuffle all кнопки
- Client-side filter search within library
- Unlike из библиотеки — **сначала подтвердить семантику бэкенда** (`/library/dislike` vs toggle `/library/like`) ручным curl'ом; задокументировать выбор в `PROJECT.md` Key Decisions
- Row = стандартный TrackCard, long-press = action sheet
**Acceptance:**
- Shuffle меняет порядок, play-all стартует с 0
- Фильтр работает < 50 мс на 500 треках
- Unlike удаляет строку и не возвращается после refresh
**Risks:** Backend-семантика unlike — блокер для LIB-4; fallback: скрыть кнопку unlike и вынести в P2 до проверки
**Est:** S

## Phase 10: Search UX Polish

**Goal:** Поиск ощущается нативным, не грузит бэкенд.
**Covers:** SR-1..5
**Depends on:** Phase 4 (плеер для проигрывания результата), Phase 12 (skeleton/empty/error — или делаем inline)
**Scope:**
- Debounce 300 мс (сейчас 500)
- Clear × в инпуте
- Search history в localStorage (10 последних), видна до ввода
- Группировка результатов по source при `sources=all` с header-иконкой
- Empty/error states с CTA/retry
**Acceptance:**
- Типинг быстрых букв не спамит бэкенд (один fetch на пачку)
- History переживает reload
**Est:** S

## Phase 11: Profile Polish

**Goal:** Профиль вкуса — живой, с телеграм-идентичностью.
**Covers:** PROF-1..4, TG-4
**Depends on:** Phase 1
**Scope:**
- Telegram avatar + `first_name` в шапке (с fallback на emoji если нет `photo_url`)
- Top-8 жанров с градиентными барами
- 4 big-number stat boxes с Lucide-иконками
- Reset через `tg.showConfirm` вместо `window.confirm`
**Acceptance:**
- Шапка показывает реального Telegram-пользователя
- Reset работает с нативным диалогом
**Est:** S

## Phase 12: Loading / Error / Offline System

**Goal:** Единая система состояний загрузки/ошибок/офлайна во всех экранах.
**Covers:** UX-1..6, AUDIO-6 (retry/toast политика)
**Depends on:** Phase 1 (компоненты)
**Scope:**
- Skeleton-миксин (CSS `@keyframes shimmer`) — используется Wave/Library/Search/Queue
- Empty states: emoji + текст + CTA-кнопка
- Error states: inline retry; нет auto-retry loop'ов
- Cover fallback: градиент с первой буквой
- `loading="lazy"` на всех `<img>` в списках
- Offline-banner через `online`/`offline` events
**Acceptance:**
- Каждый экран имеет empty/error/loading и скриншот каждого
- Принудительный offline → banner появляется, network запросы показывают inline error
**Est:** S

## Phase 13: Visual Polish & Design Tokens

**Goal:** Финальный визуальный pass до уровня Spotify/Яндекс — типографика, отступы, палитра, микро-анимации.
**Covers:** визуальные acceptance всех предыдущих фаз
**Depends on:** Phases 4–12 (структура зафиксирована)
**Scope:**
- CSS custom properties: цвета, тени, radii, spacing, typography scale
- `@layer` для reset / tokens / components / utilities
- Tap-фидбэк (`:active` scale/brightness) на всех интерактивах
- Анимации: transitions ≤ 200 мс, easings cubic-bezier(.2,.8,.2,1)
- Safe-area insets для iOS (notch/home indicator)
- Темизация от `tg.themeParams` (background_color, text_color) как primary palette с fallback
**Acceptance:**
- Визуальное сравнение со скринами Яндекс «Моя волна» и Spotify Now Playing — паритет ощущения
- 60 fps во всех анимациях (Chrome DevTools Performance)
- iOS safe-area корректно отступает
**Est:** M

## Phase 14: QA, Perf, Device Matrix

**Goal:** Финальный gate перед релизом: проверка всех acceptance criteria на реальных устройствах.
**Covers:** PERF-1..5 + весь acceptance по `REQUIREMENTS.md`
**Depends on:** Phases 1–13
**Scope:**
- Device matrix: iPhone (Telegram iOS), Redmi/Pixel бюджетный Android (Telegram Android), Desktop Telegram для smoke
- Perf-бюджеты: FMP < 200 мс, Interaction-to-sound < 1.5 сек на 4G, 60 fps, heap стабилен после 100 скипов
- Regression suite ручной: все 14 фаз acceptance прокликать
- Размер `index.html` ≤ 120 KB, gzip < 30 KB
- Memory leak check: 30 мин прослушивания + 100 tab-switches → `usedJSHeapSize` стабилен ±10%
- Backend contract audit: каждый эндпоинт из `PROJECT.md` context попадает под реальный вызов в коде
**Acceptance:**
- Все требования `REQUIREMENTS.md` с Priority=P1 и Status≠BLOCKED помечены DONE
- Нет console-ошибок на happy path любого экрана
- Ни один pitfall из `research/PITFALLS.md` не воспроизводится
**Est:** M

---

## Диаграмма зависимостей

```
Phase 1  (architecture)
  └── Phase 2  (audio core)
  └── Phase 3  (gestures)
        ├── Phase 4  (mini-player)        ← depends on 1,2,3
        │     └── Phase 5  (fullscreen)    ← depends on 2,3,4
        │           └── Phase 6  (queue)   ← depends on 3,5
        │                 └── Phase 7  (wave)      ← depends on 2,3,4,5
        │                       └── Phase 8  (feedback UX) ← depends on 3,6
        │                             └── Phase 9  (library)    ← depends on 4,5,6
        │                                   └── Phase 10 (search)  ← depends on 4
        │                                         └── Phase 11 (profile) ← depends on 1
        │                                               └── Phase 12 (states) ← depends on 1
        │                                                     └── Phase 13 (visual) ← depends on 4–12
        │                                                           └── Phase 14 (QA/perf) ← gate
```

Параллелизация возможна между Phase 9/10/11/12 — они независимы друг от друга после того как Phase 1–6 готовы.

## Что НЕ в этом roadmap (future milestones)

- **Backend-blocked P3:** lyrics, «because you liked X» в Wave, top artists, trending searches, discover-weekly, custom playlists (см. REQUIREMENTS §14 BLK-1..6)
- **Backend milestones:** VK/Яндекс как источники (BLK-7), HMAC-валидация initData (BLK-8), AI-тегирование, файловая раскладка медиатеки по user_id
- **Оффлайн / Service Worker** — осознанно out of scope (PROJECT.md)
- **P2 фичи из FEATURES.md** (shareable taste card, voice search, marquee-scroll, sort dropdown, shuffle/repeat toggles) — отдельный мини-milestone после валидации v1 на живых пользователях
