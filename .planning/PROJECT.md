# VDS Music — Telegram Mini App

## What This Is

VDS Music — это Telegram Mini App аудиоплеер, открывающийся через Telegram-бота. Каждый пользователь получает персональный бесконечный поток «Моя волна» с обучением по лайкам/дизлайкам, поиск по YouTube/SoundCloud/Jamendo, жанровые сборники и коллекцию лайкнутых треков. Вся музыка хранится на ПК-хранилище (931GB) и отдаётся через бэкенд `api.vdsmusic.ru:8443` по Tailscale.

## Core Value

Мгновенно получать персональную «Мою волну», которая учится на твоих лайках и играет только то, что нравится именно тебе — без регистраций, просто открыв бота.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- Текущий scope: полный фронт под существующий API + редизайн до уровня Spotify/Яндекс.Музыка. -->

- [ ] Гибридный редизайн UI (Яндекс-подобная «Волна» на главной + Spotify-подобная библиотека и плеер), тёмная тема, мобильный
- [ ] Персистентный мини-плеер внизу экрана, не пропадает при навигации, тап = раскрыть на весь экран
- [ ] Очередь воспроизведения с просмотром и drag-to-reorder
- [ ] Жесты свайпов: влево/вправо — пред/след трек, swipe down — свернуть полноэкранный плеер
- [ ] MediaSession API + управление с локскрина (обложка, название, контролы в шторке уведомлений)
- [ ] «Моя волна» — бесконечный поток через `/wave/*`, выбор настроений, лайки/дизлайки отправляются на бэк
- [ ] Поиск по YouTube / SoundCloud / Jamendo через `/search` с выбором источника
- [ ] Жанровые сборники через `/genre`
- [ ] Коллекция лайкнутых треков через `/library/likes` + статистика `/library/stats`
- [ ] Профиль вкуса пользователя через `/taste/{user_id}` + сброс `/taste/reset`
- [ ] Интеграция с Telegram WebApp SDK: expand, theme colors, user_id из initDataUnsafe, haptic feedback, back button
- [ ] Один файл `index.html` без сборки, деплой на GitHub Pages через `git push`

### Out of Scope

<!-- Explicit boundaries — эти пункты в будущих milestone'ах. -->

- VK Music / Яндекс.Музыка как источники — отдельный milestone, требует бэкенд-работы
- AI-тегирование треков — отдельный milestone, инфраструктурная задача
- Персональная медиатека с жанровыми папками по user_id — отдельный milestone, файловая раскладка на СХД
- Любые изменения бэкенда / новые API-эндпоинты — работаем только с тем, что уже отдаёт `api.vdsmusic.ru:8443`
- Серверная валидация Telegram initData (HMAC-подпись) — осознанно оставлено на будущее, пока доверяем `user_id` из `initDataUnsafe`
- Сборка через Vite / фреймворки — противоречит ограничению «один файл на Pages»
- Оффлайн-кэширование и Service Worker — не в этом цикле

## Context

**Текущее состояние кода:**
- Один файл `index.html` (~1023 строки) с inline CSS/JS
- Telegram WebApp SDK уже подключён (`telegram-web-app.js`), используется `tg.expand()`, `tg.setHeaderColor()`
- `USER_ID` берётся из `tg.initDataUnsafe.user.id`
- Подключены эндпоинты: `/wave/moods`, `/wave/feedback`, `/search?q=&src=&user_id=`, `/genre?q=&user_id=`, `/library/likes`, `/library/stats`, `/library/like` (POST), `/library/dislike` (POST), `/taste/{user_id}`, `/taste/reset` (POST)
- Почти весь функционал уже работает — главная цель milestone именно **редизайн** и доведение UX до уровня Spotify/Яндекс.Музыка
- Недавние коммиты: фикс ID3-тегов и имён из метаданных, фикс URL стриминга, постоянный API URL

**Бэкенд:**
- `https://api.vdsmusic.ru:8443` — постоянный HTTPS-эндпоинт
- Вся рекомендательная логика «Моей волны» (обучение по лайкам/истории/ранжирование) уже живёт на бэкенде; фронт только показывает результат и отправляет события
- Музыка хранится на ПК (931GB) и раздаётся через Tailscale
- Планируемые бэкенд-улучшения: AI-тегирование, медиатека с жанровыми папками, VK/Яндекс — все в будущих milestone'ах

**Пользователи:**
- Неограниченное количество; доступ через Telegram-бот
- Каждый пользователь идентифицируется по Telegram ID
- У каждого свой профиль вкуса, своя волна, своя коллекция

## Constraints

- **Tech stack**: один `index.html` + inline CSS/JS + CDN-библиотеки — деплой `git push` на GitHub Pages, никаких bundler'ов
- **Platform**: Telegram Mini App (WebApp SDK), мобильный первым, тёмная тема
- **Backend**: фиксированный API `https://api.vdsmusic.ru:8443`, новые эндпоинты в этом milestone не добавляются
- **UX**: уровень Яндекс.Музыка / Spotify — это планка качества, а не nice-to-have
- **Auth**: `user_id` из `initDataUnsafe` без серверной валидации подписи (осознанный trade-off этого milestone)
- **Performance**: работает на бюджетных Android-устройствах в Telegram WebView; никаких тяжёлых фреймворков
- **Deployment**: GitHub Pages, статический хостинг, только клиентский код

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Один `index.html` без сборки | Простота деплоя (`git push` → Pages), минимум движущихся частей, понятная отладка | — Pending |
| Гибридный UI (Яндекс «Волна» + Spotify библиотека/плеер) | «Волна» — главная ценность продукта (Яндекс решил это лучше всех), а библиотека/очередь/плеер — более зрелые у Spotify | — Pending |
| Рекомендации только на бэкенде, фронт отображает | Бэк уже владеет историей/лайками/хранилищем — дублировать логику на фронте бессмысленно и небезопасно | — Pending |
| Без серверной валидации initData в этом milestone | Аудитория — друзья/ограниченный круг, ускоряет фокус на UX; подпись добавим когда появится реальная угроза | ⚠️ Revisit |
| MediaSession API обязателен с первого milestone | Пользователи Telegram Mini App ожидают управления с локскрина как у Spotify/Яндекса; без этого плеер ощущается «игрушечным» | — Pending |
| Фокус milestone — редизайн, а не новые фичи | Функционал уже работает, узкое место — визуал и UX до уровня конкурентов | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-15 after initialization*
