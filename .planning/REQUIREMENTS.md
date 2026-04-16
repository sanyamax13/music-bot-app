# VDS Music — Requirements

**Milestone:** Redesign v1 — гибридный Яндекс/Spotify UX поверх существующего API
**Source:** `.planning/research/{FEATURES,ARCHITECTURE,STACK,PITFALLS}.md` + `PROJECT.md`
**Last updated:** 2026-04-15

## Легенда

- **Priority:** P1 = обязательно в этот milestone · P2 = после валидации v1 · P3 = заблокировано бэкендом
- **Type:** F = функциональное · NF = нефункциональное
- **Status:** PENDING · IN_PROGRESS · DONE · BLOCKED
- Каждое требование мапится на эндпоинт (если есть) и на фазу в `ROADMAP.md`.

---

## 1. Архитектурные требования (фундамент)

| ID | Type | Priority | Requirement | Acceptance |
|---|---|---|---|---|
| ARCH-1 | NF | P1 | Единый `Store` (IIFE + pub/sub, key-scoped subscribe) — один владелец на ключ | Весь playback-state пишется только `PlaybackEngine`; UI-state только `Router`/компонентом-владельцем; grep по `Store.set` показывает ровно одного писателя на ключ |
| ARCH-2 | NF | P1 | `PlaybackEngine` IIFE — единственный владелец `audioA`/`audioB`, очереди, MediaSession | Компоненты не читают `<audio>` напрямую; `PlaybackEngine.play/pause/next/prev/seek` — единственный API |
| ARCH-3 | NF | P1 | Компоненты как IIFE с `mount(root)` / `render(state)` / `destroy()` — 12 штук согласно ARCHITECTURE §4 | Каждый компонент в собственном логическом блоке, subscribe по ключам store |
| ARCH-4 | NF | P1 | Роутинг — машина состояний экранов + `tg.BackButton` как единственный back, без hash/History API | BackButton precedence: queuePanel → fullscreen → screen stack → `tg.close()` |
| ARCH-5 | NF | P1 | Один `index.html`, inline CSS/JS, только CDN-зависимости (SortableJS, Lucide), без сборки | `git push` → GitHub Pages → работает; нет `package.json`, нет импорт-мапы |
| ARCH-6 | NF | P1 | Файл ≤ ~2.5k строк, разделён на логические секции согласно ARCHITECTURE §1 | Визуальная иерархия: `<style>` → skeleton markup → Config → Utils → Telegram → Store → Api → PlaybackEngine → Feedback → Router → components → Boot |

## 2. Аудио-ядро и надёжность (pitfalls)

| ID | Type | Priority | Requirement | Acceptance |
|---|---|---|---|---|
| AUDIO-1 | NF | P1 | iOS WKWebView autoplay unlock на первом тапе: синхронный `audioA.play()` с silent-mp3 до любого `await` | На iOS Telegram первый тап Wave запускает трек; флаг `audioUnlocked` выставляется; `NotAllowedError` не возникает |
| AUDIO-2 | NF | P1 | `addEventListener('ended', …)` + явный `removeEventListener` на обоих буферах при каждом `playTrack` | Тест: 10 быстрых skip'ов → `queueIndex` продвигается ровно на 10; `finish`-feedback не дублируется |
| AUDIO-3 | NF | P1 | `navigator.mediaSession.setPositionState()` обновляется throttled ~1× сек + на `seeked`/`ratechange` | Locks-screen scrubber на Android показывает реальную позицию и реагирует на seek |
| AUDIO-4 | F  | P1 | MediaSession handlers: play, pause, nexttrack, previoustrack, seekbackward (15s), seekforward (15s), seekto | Все 7 actions зарегистрированы; работают с локскрина/шторки/BT-гарнитуры |
| AUDIO-5 | NF | P1 | Preload следующего трека в `nextAudio.src` по достижении 70% текущего, swap на `ended`, gap < 100 мс | Слышимая пауза между треками не заметна на глаз в логе `timeupdate` |
| AUDIO-6 | NF | P1 | Обработка `error`/`stalled`/`waiting` на audio element — retry раз, потом skip с toast | Один протухший `/stream/{id}` не роняет очередь |
| AUDIO-7 | NF | P1 | `audio.crossOrigin = "anonymous"` и проверка CORS на `/stream` и thumb-URL'ах | Canvas `getImageData` для dominant-color не падает; если CORS закрыт — fallback на палитру по хэшу ID |

## 3. Wave / «Моя волна»

| ID | Type | Priority | Endpoint | Requirement | Acceptance |
|---|---|---|---|---|---|
| WAVE-1 | F | P1 | `/wave` | Бесконечный стрим: авто-подгрузка ещё треков когда остаётся ≤ 3 | Очередь не кончается при непрерывном прослушивании ≥ 30 мин |
| WAVE-2 | F | P1 | `/wave/moods` | Mood-chips горизонтальным скроллом, sticky, активное состояние, emoji | Tap mood → заголовок градиентит, стрим пересеивается |
| WAVE-3 | F | P1 | `/wave/feedback` | Skip < 30 сек = `action:skip` c `elapsed`; ≥ 30 сек = `action:finish` | Оба события видны в сетевом логе с корректным `elapsed` |
| WAVE-4 | F | P1 | `/library/like` `/library/dislike` + `/wave/feedback` | Like/Dislike из Wave; dislike авто-скипает следующий трек | После dislike трек меняется без ручного тапа next |
| WAVE-5 | F | P1 | client | Pulse-индикатор активного трека в списке Wave | Текущий трек визуально отличается, обновляется на смене |
| WAVE-6 | F | P1 | client | Pull-to-refresh на Wave — полный reseed | Жест pull вниз > 60px → вызывает `/wave` заново |
| WAVE-7 | F | P1 | client | Skeleton-rows показываются < 200 мс первого открытия, до ответа `/wave` | Layout не прыгает при замене skeleton → данные |

## 4. Fullscreen Player

| ID | Type | Priority | Endpoint | Requirement | Acceptance |
|---|---|---|---|---|---|
| FP-1 | F | P1 | client | Крупная обложка ~70% ширины экрана, квадрат | Визуально соответствует макету Spotify/Яндекс |
| FP-2 | F | P1 | client | Dominant-color фон извлекается из обложки (Canvas), fallback на хэш-палитру | Фон меняется на смене трека ≤ 300 мс |
| FP-3 | F | P1 | client | Прогресс-скраббер, touch-target ≥ 44 px, время текущее/полное | Scrub пальцем работает, не дёргается |
| FP-4 | F | P1 | `/library/like` | Heart-button, большое попадание, заполнение без модала | Тап → анимация fill, POST, optimistic |
| FP-5 | F | P1 | client | Swipe-down = свернуть в mini-player | Жест вниз > 80 px закрывает FP, BackButton тоже закрывает |
| FP-6 | F | P1 | client | Swipe-left/right на обложке = next/prev | Работает без конфликта с вертикальным скроллом |
| FP-7 | F | P1 | client | Double-tap на обложке = like | Хеарт-анимация поверх обложки |
| FP-8 | F | P1 | client | Queue-peek «Далее: …» внизу FP | Показывает `queue[index+1].title` |
| FP-9 | F | P1 | `/search` | Тап по имени артиста → prefilled search | Switch на Search tab с заполненным query |
| FP-10 | F | P1 | client | MediaSession metadata (title, artist, artwork) обновляется на каждый трек | Шторка Android показывает корректную обложку |

## 5. Mini-Player

| ID | Type | Priority | Endpoint | Requirement | Acceptance |
|---|---|---|---|---|---|
| MINI-1 | F | P1 | client | Persistent, виден всегда когда есть currentTrack, на всех экранах, над BottomNav | Switch tab не останавливает/не прячет плеер |
| MINI-2 | F | P1 | client | Tap = expand FP; swipe-up = expand FP | Оба жеста работают |
| MINI-3 | F | P1 | client | Swipe-left/right = prev/next | Жест с делением по X > 60 px |
| MINI-4 | F | P1 | client | Hairline progress ~2 px вверху | Плавная анимация на requestAnimationFrame |
| MINI-5 | F | P1 | `/library/like` | Heart-button для быстрого лайка | Без expand FP |
| MINI-6 | F | P1 | client | Буферизация: play-icon → spinner на `waiting`, обратно на `canplay` | Видно когда стрим тупит |
| MINI-7 | F | P2 | client | Marquee-scroll длинных названий | CSS @keyframes, только при overflow |

## 6. Queue

| ID | Type | Priority | Endpoint | Requirement | Acceptance |
|---|---|---|---|---|---|
| Q-1 | F | P1 | client | Bottom-sheet overlay с секциями History / Now Playing / Up Next | Открывается из FP; BackButton закрывает |
| Q-2 | F | P1 | client | Drag-to-reorder через SortableJS в non-Wave контекстах | Wave-очередь read-only, genre/search/library reorder работает |
| Q-3 | F | P1 | client | Swipe-left row → remove; tap row → jump | Drag handle слева ≠ зоне swipe |
| Q-4 | F | P1 | client | Long-press → «Play next» / «Add to queue» | Inject в `userQueue` / append `queue` |
| Q-5 | F | P1 | client | History scrollable вверх (in-memory, session-only) | Последние 50 сыгранных треков |

## 7. Поиск

| ID | Type | Priority | Endpoint | Requirement | Acceptance |
|---|---|---|---|---|---|
| SR-1 | F | P1 | `/search` | Debounce 300 мс, source-chips (All/YT/SC/Jamendo) | Типинг не спамит бэкенд |
| SR-2 | F | P1 | client | Search history в localStorage, последние 10 | Показывается до ввода |
| SR-3 | F | P1 | client | Clear (×) в инпуте, skeleton на время запроса | — |
| SR-4 | F | P1 | client | Empty state с CTA; error state с retry | Сообщения не обрывают flow |
| SR-5 | F | P1 | client | Группировка по источнику при `sources=all` (headers с иконкой) | Заголовки «YouTube», «SoundCloud», «Jamendo» |

## 8. Библиотека (Likes)

| ID | Type | Priority | Endpoint | Requirement | Acceptance |
|---|---|---|---|---|---|
| LIB-1 | F | P1 | `/library/likes` | Список лайков, reverse-chrono, счётчик и длительность в шапке | — |
| LIB-2 | F | P1 | client | Play all / Shuffle all кнопки | Формируют `queue` из всей библиотеки |
| LIB-3 | F | P1 | client | Search within library (клиентский фильтр по title/artist) | Мгновенный фильтр |
| LIB-4 | F | P1 | `/library/dislike` | Unlike из библиотеки → удаление строки | ⚠ Проверить: `dislike` vs toggle `like` — подтвердить семантику до wiring (см. FEATURES §6) |
| LIB-5 | F | P2 | client | Сортировка: recent / A-Z / artist | Dropdown в шапке |

## 9. Профиль вкуса

| ID | Type | Priority | Endpoint | Requirement | Acceptance |
|---|---|---|---|---|---|
| PROF-1 | F | P1 | `/taste/{user_id}` | Top-8 жанров/тегов с градиентными барами | Бары пропорциональны `weight` |
| PROF-2 | F | P1 | `/library/stats` | 4 big-number box'а (tracks, likes, listens, artists) | Иконки Lucide на каждом |
| PROF-3 | F | P1 | Telegram SDK | Avatar + имя из `initDataUnsafe.user.photo_url` + `first_name` | Показывается в шапке профиля |
| PROF-4 | F | P1 | `/taste/reset` | Reset через `tg.showConfirm(…)` вместо `window.confirm` | Родной Telegram-диалог |
| PROF-5 | F | P2 | client | Shareable «Your taste» canvas-card + `tg.shareMessage`/MainButton | Картинка рендерится, шарится через Telegram |

## 10. Жесты и фидбэк

| ID | Type | Priority | Requirement | Acceptance |
|---|---|---|---|---|
| GEST-1 | NF | P1 | Единый gesture-handler (touchstart/move/end, direction, threshold) — используется всеми | Один модуль ~120 LOC, greppable |
| GEST-2 | F | P1 | Long-press на row → action sheet {Like/Dislike/Play next/Add to queue/Share/Search artist} | 500 мс hold, haptic на срабатывании |
| GEST-3 | F | P1 | Swipe-right row = quick like, swipe-left row = quick dislike (в Wave/search/genre) | Не конфликтует с remove в Queue |
| FB-1 | F | P1 | Optimistic heart fill до ответа POST; rollback на error | — |
| FB-2 | F | P1 | Undo snackbar для dislike (3 сек до фактического POST) | Tap «Отменить» отменяет |
| FB-3 | F | P1 | `tg.HapticFeedback.impactOccurred('light')` на like/dislike/play/pause/expand | Работает на iOS/Android где SDK поддерживает |

## 11. Loading / Empty / Error / Offline

| ID | Type | Priority | Requirement | Acceptance |
|---|---|---|---|---|
| UX-1 | F | P1 | Skeleton-система (shimmer rows) вместо спиннеров во всех списках | CSS-only, < 30 LOC |
| UX-2 | F | P1 | Empty states: emoji + текст + CTA-кнопка | Wave / search / library имеют собственные |
| UX-3 | F | P1 | Error states: inline, с retry-кнопкой; нет auto-retry циклов | — |
| UX-4 | F | P1 | Cover fallback: градиент с первой буквой при `onerror` / отсутствии `thumb` | Применяется к YT/SC без обложки |
| UX-5 | F | P1 | `loading="lazy"` на всех обложках в длинных списках | — |
| UX-6 | F | P1 | Offline-banner через `online`/`offline` events | Видим пока офлайн, скрывается сам |

## 12. Telegram Integration

| ID | Type | Priority | Requirement | Acceptance |
|---|---|---|---|---|
| TG-1 | NF | P1 | `tg.expand()`, `setHeaderColor`, `setBackgroundColor` на старте | Приложение раскрыто при открытии |
| TG-2 | F  | P1 | `tg.BackButton` как единственный back-control (см. ARCH-4) | — |
| TG-3 | F  | P1 | `tg.HapticFeedback` — на всех тактильных действиях (FB-3) | — |
| TG-4 | F  | P1 | `tg.showConfirm` вместо `window.confirm` | — |
| TG-5 | F  | P2 | Share track через Telegram share (link/сообщение) | — |
| TG-6 | NF | P1 | `user_id` из `initDataUnsafe.user.id`, без серверной HMAC-валидации в этом milestone | Осознанный trade-off зафиксирован в `PROJECT.md` |

## 13. Производительность / бюджеты

| ID | Type | Priority | Budget | Acceptance |
|---|---|---|---|---|
| PERF-1 | NF | P1 | First meaningful paint Wave-скелетона < 200 мс с момента DOMContentLoaded | Lighthouse/DevTools performance trace |
| PERF-2 | NF | P1 | Interaction-to-sound (тап Play → первый семпл) < 1.5 сек на 4G | Измерение на реальном устройстве |
| PERF-3 | NF | P1 | Размер `index.html` ≤ ~120 KB (gzip < 30 KB) без учёта CDN | Контроль в CI / вручную перед деплоем |
| PERF-4 | NF | P1 | UI 60 fps во время playback на бюджетном Android (Redmi 9 / Pixel 4a) | Ручной тест, Performance panel |
| PERF-5 | NF | P1 | Нет memory-leaks при 100 сменах трека (проверка `performance.memory`) | `usedJSHeapSize` стабилен ±10% |

## 14. Заблокировано бэкендом (P3 — future milestones)

| ID | Feature | Blocker |
|---|---|---|
| BLK-1 | «Because you liked X» в Wave | нет `reason` в `/wave` response |
| BLK-2 | Lyrics-панель | нет `/lyrics` |
| BLK-3 | Top artists в профиле вкуса | `/taste` отдаёт tags, не artists |
| BLK-4 | Trending / suggested searches | нет endpoint |
| BLK-5 | Discover-weekly | нет endpoint |
| BLK-6 | Custom playlists | нет persistence |
| BLK-7 | VK/Яндекс как источники | отдельный backend-milestone |
| BLK-8 | HMAC-валидация initData | отдельное security-решение |

## 15. Anti-requirements (что НЕ делаем осознанно)

- Модальный onboarding «выбери 5 жанров» — FEATURES §1 anti-feature
- Toast на каждый лайк — используется silent heart-fill
- Рейтинг 1–5 — binary like/dislike конвертит лучше
- Custom playlists, offline-кэш, Service Worker — out of scope
- Video/animated equalizer фоны — противоречит PERF-4
- 10-band EQ, sleep-timer внутри now-playing — низкий usage
- Autoplay Wave до первого тапа — iOS WKWebView lock (AUDIO-1)
- Shake-to-shuffle, two-finger volume, long-press play — FEATURES §10 anti-gestures
- Bundler / фреймворки / npm — противоречит ARCH-5
- Дублирование рекомендательной логики на фронте — см. Key Decision в `PROJECT.md`

---

## Mapping: Requirements → Roadmap Phases

| Фаза | Покрывает |
|---|---|
| Phase 1 — Architecture refactor | ARCH-1..6 |
| Phase 2 — Audio core & MediaSession | AUDIO-1..7, FP-10 |
| Phase 3 — Gesture foundation | GEST-1 |
| Phase 4 — Mini-player | MINI-1..6 |
| Phase 5 — Fullscreen player | FP-1..9 |
| Phase 6 — Queue | Q-1..5, GEST-2 |
| Phase 7 — Wave redesign | WAVE-1..7 |
| Phase 8 — Feedback UX | GEST-3, FB-1..3, TG-3 |
| Phase 9 — Library | LIB-1..4 |
| Phase 10 — Search polish | SR-1..5 |
| Phase 11 — Profile polish | PROF-1..4, TG-4 |
| Phase 12 — Loading/error/offline system | UX-1..6, AUDIO-6 |
| Phase 13 — Visual polish & design tokens | Визуальный pass всех фаз |
| Phase 14 — QA, perf, device matrix | PERF-1..5, все acceptance criteria |
