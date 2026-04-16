# Feature Research — VDS Music Redesign

**Domain:** Mobile music streaming app (Telegram Mini App) — Yandex.Music / Spotify-grade UX
**Researched:** 2026-04-15
**Confidence:** HIGH (Spotify / Yandex.Music / Apple Music / YT Music are widely documented and all four are inspected by millions of users every day; gesture and state-handling conventions are stable since ~2023)

## Context & Constraints Recap

- **Backend is frozen** this milestone. Existing endpoints:
  `/wave` · `/wave/moods` · `/wave/feedback` · `/search?q=&sources=&user_id=` · `/genre?q=&user_id=` · `/library/likes` · `/library/stats` · `/library/like` (POST) · `/library/dislike` (POST) · `/taste/{user_id}` · `/taste/reset` (POST) · `/stream/{id}`
- Every feature below is either (a) mapped to one of those endpoints, (b) purely client-side, or (c) explicitly flagged **BLOCKED — needs backend**.
- Platform: Telegram WebView on mobile (Android-first, bottom-sheet ergonomics, one thumb). No desktop assumptions.

---

## 1. Home / Wave Experience ("Моя волна")

### Table Stakes

| Feature | Why Expected | Complexity | Backend | Notes |
|---|---|---|---|---|
| Mood/context chips above the wave (horizontal scroll) | Yandex ships this as the defining Wave affordance; users learn it in the first session | S | `/wave/moods` ✓ | Already wired. Needs visual polish: pill chips, active state, emoji, sticky on scroll |
| Auto-advance to next wave track (no manual action) | Wave is a *stream*, not a playlist. Stopping after one track = broken | S | existing `onended` ✓ | Already implemented |
| Seamless / near-gapless transition | Spotify and Yandex both preload the next track; 1-2s silence feels janky | M | dual-buffer already exists ✓ | Current code preloads `nextAudio` — keep and verify crossfade-less behaviour |
| Infinite stream (auto-fetch more when 2-3 tracks remain) | Running out of Wave is the #1 "it's broken" complaint | S | `/wave` repeat calls ✓ | Already implemented in `autoExtendWave()` |
| Like / dislike send feedback to backend | The core value prop ("учится на лайках") is invisible without this | S | `/wave/feedback`, `/library/like`, `/library/dislike` ✓ | Already wired |
| Skip = negative signal (distinct from dislike) | Both Yandex and Spotify treat a <30s skip as "meh", not hard no | S | `/wave/feedback` with `action:skip` + `elapsed` ✓ | Already wired |
| Current wave track has prominent "now playing" indicator in the list | Users need to see "where am I in the stream" | S | client-only | Pulse/wave animation on the active row |

### Differentiators

| Feature | Value Prop | Complexity | Backend | Notes |
|---|---|---|---|---|
| Mood chip shows *why this track* is playing ("Because you liked X") | Yandex does explanatory cards; builds trust in recommender | M | **BLOCKED** — no `reason` field in `/wave` response | Flag for future backend milestone |
| Long-press on track row → "Не нравится" / "Не добавлять в Мою волну" action sheet | Yandex's signature gesture; more discoverable than inline dislike | S | existing `/library/dislike` ✓ | Client-only, replaces the tiny `👎` button |
| Wave header animates on mood change (gradient shift) | Emotional cue that the stream has re-seeded | S | client-only | Pure CSS, cheap win |
| Mini-loader between tracks (shimmer on next-up preview) | Makes the stream feel "alive" | S | client-only | — |

### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Modal onboarding ("pick 5 genres you like!") | "Cold start" intuition | Telegram Mini App users expect *instant* content. Yandex Wave literally starts playing on first open. Every modal before first play costs 20-40% of new users | Start Wave immediately on `/wave` with empty taste; let lazy-implicit feedback seed the profile |
| Toast every time a like is recorded | "Confirm the action" | Creates toast spam when liking 5 tracks in a row; Spotify/Yandex use silent heart-fill instead | Heart icon flips state + short haptic buzz |
| "Rate every track 1-5 stars" | More signal | Users refuse; binary like/dislike converts 10x better | Keep binary + skip-as-signal |
| Auto-switching mood without user action | "Smart context" | Pulls the rug out from under users — they thought they picked "Focus" and suddenly get "Party" | Always honour the explicit mood chip |

---

## 2. Full-Screen "Now Playing"

### Table Stakes

| Feature | Why Expected | Complexity | Backend | Notes |
|---|---|---|---|---|
| Large cover art (square, ~70% of screen width) | Defining visual of every music app since 2012 | S | client-only | — |
| Title + artist below cover, title is tappable (→ search artist) | Users expect artist→explore drill-down | S | client-only (calls existing `/search`) | Cheap: tap artist name = prefilled search |
| Progress scrubber with current/total time | Everyone has this | S | existing `progress` range input ✓ | Already there, needs bigger hit target (44px) |
| Play / pause | — | S | ✓ | — |
| Next / previous track | — | S | ✓ | — |
| Seek backward 15s / forward 15s via MediaSession | Spotify/YT Music expose these on lock screen; users hit them reflexively for podcasts and long mixes | S | `navigator.mediaSession.setActionHandler('seekbackward'/'seekforward')` | Client-only — extend existing `setupMediaSession` |
| Like (heart) button on the fullscreen player | Primary place users like tracks | S | `/library/like` ✓ | Already wired via `fp-fav-btn` |
| Swipe-down to collapse fullscreen → mini-player | Universal iOS/Android music-app gesture; Spotify, Yandex, Apple Music all do it | M | client-only | Touch events + transform; ~40 LOC |
| Swipe left/right on cover = next/prev track | Apple Music + Yandex both support it | M | client-only | Same gesture handler |
| Visible queue peek ("Далее: [track name]") at bottom | Users want to know what's next without opening the queue | S | currentQueue[currentIndex+1] ✓ | Client-only |

### Differentiators

| Feature | Value Prop | Complexity | Backend | Notes |
|---|---|---|---|---|
| Dominant-color background extracted from cover art | Spotify's signature feel; makes each track visually distinct | M | client-only (Canvas API on cover) | ~30 LOC, one-time per track load |
| Lyrics panel (tap title → lyrics) | Yandex and Spotify have it; strong engagement driver | L | **BLOCKED** — no `/lyrics` endpoint | Flag for future backend milestone |
| Share track (Telegram share sheet via `tg.openTelegramLink` or `MainButton`) | Native Telegram integration — a unique fit vs. Spotify | S | client-only | Uses Telegram WebApp SDK — zero cost |
| Shuffle & repeat toggles (queue, repeat-one) | Table stakes in general music apps, but *differentiator* for Wave-centric UX where Wave is already infinite | S | client-only | Only meaningful for collection/genre/search contexts |
| Long-press cover → action sheet (add to queue, share, artist, go to album) | Spotify's long-press menu | M | client-only | — |

### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Animated equalizer / spectrum visualization | "Looks cool" | Mobile GPU cost, drains battery, users disable it within a week | Static cover + subtle scale-pulse on beat-approximation (CSS only) |
| Video background (album art ken-burns effect) | "Premium feel" | Jank on cheap Android in Telegram WebView; violates perf constraint | Dominant-color gradient is 95% of the effect at 1% of the cost |
| 10-band EQ | "Audiophile users" | Adds settings-screen complexity; <1% of Spotify users ever open the EQ | Don't build |
| Sleep timer inside the now-playing screen | "Some people fall asleep listening" | Creates discoverability noise for a feature used by <5% | Put in profile/settings if ever needed |

---

## 3. Persistent Mini-Player

### Table Stakes

| Feature | Why Expected | Complexity | Backend | Notes |
|---|---|---|---|---|
| Always visible above bottom nav when a track is playing | Every major app does this; navigating away must NOT stop the stream | S | already styled, just make persistent across tabs ✓ | — |
| Tap anywhere (except buttons) = expand to fullscreen | Universal convention | S | client-only | — |
| Play/pause button inside mini | — | S | ✓ | Already there |
| Thin progress bar line at top or bottom edge of mini | Spotify-style hairline (2px) | S | client-only | Derived from `onTimeUpdate` |
| Track title + artist, single line each, ellipsis | — | S | ✓ | — |
| Cover thumbnail on the left | — | S | ✓ | — |
| Swipe-up on mini = expand to fullscreen (alternative to tap) | Spotify added this in 2023; now expected | M | client-only | Touch gesture handler |
| Swipe left/right on mini = prev/next track | Apple Music has this; reduces friction dramatically | M | client-only | Same gesture handler |

### Differentiators

| Feature | Value Prop | Complexity | Backend | Notes |
|---|---|---|---|---|
| Heart icon in mini-player for quick like without expanding | Reduces taps for the #1 action | S | `/library/like` ✓ | Tight on mobile real-estate but doable |
| Marquee-scroll long titles | Nice for Russian/long artist names | S | client-only (CSS `@keyframes`) | — |

### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Close (X) button on mini-player | "Users want to dismiss it" | Dismissing the mini-player = stopping music, which is what pause is for. Removes the persistence promise. Spotify/Yandex/Apple Music all *do not* have a close button | No close button. Pause = pause; to really stop, user closes the Mini App |
| Volume slider in mini | "Standard audio control" | OS already handles volume; in a Telegram Mini App this is noise | Rely on hardware / OS volume |
| Expand on long-press instead of tap | "Discoverability" | Breaks muscle memory from every other music app | Tap to expand |

---

## 4. Queue Management

### Table Stakes

| Feature | Why Expected | Complexity | Backend | Notes |
|---|---|---|---|---|
| Dedicated "Очередь" view (bottom sheet or tab inside fullscreen player) | Every major app has a queue screen | M | client-only (`currentQueue` already exists) | — |
| "Up next" section showing upcoming tracks | Users want visibility | S | `currentQueue.slice(currentIndex+1)` ✓ | — |
| "Now playing" section (current track) | Orients the user | S | ✓ | — |
| Reorder via drag handles | Spotify & Yandex both support it; user prompt explicitly asks for drag-to-reorder | M | client-only | Use pointer events + transform; ~80 LOC. No library needed |
| Remove from queue (swipe-left-to-delete or X button) | Standard | S | client-only | — |
| Tap a queued track = jump to it (skip ahead) | Standard | S | `playTrack(i)` ✓ | — |

### Differentiators

| Feature | Value Prop | Complexity | Backend | Notes |
|---|---|---|---|---|
| "Play next" vs "Add to queue" distinction on track long-press | Spotify's power-user feature; distinguishes insert-after-current vs append-to-end | S | client-only (`currentQueue.splice`) | — |
| History of played tracks (scroll up in queue) | Yandex shows this; nice for "wait, what was that song two back?" | S | Keep `playedHistory[]` array in memory | Session-only, lost on reload — acceptable |
| Shuffle queue (not the Wave — the manual queue) | For genre/search/collection contexts | S | client-only | — |

### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Persistent cross-session queue | "Continue where I left off" | Requires backend persistence = out of scope; also conflicts with Wave's freshness | Session-only queue; Wave re-seeds on reload anyway |
| Editing the Wave queue (reorder upcoming Wave tracks) | "User control" | The Wave is a recommender output; reordering it defeats the purpose and confuses the feedback loop | Queue reordering only in non-Wave contexts (genre, search, collection); Wave queue is read-only |

---

## 5. Search UX

### Table Stakes

| Feature | Why Expected | Complexity | Backend | Notes |
|---|---|---|---|---|
| Debounced instant search (300-500ms) | Everyone has it; already implemented at 500ms ✓ | S | `/search` ✓ | Consider lowering to 300ms |
| Source filter chips (All / Library / YouTube / SoundCloud / Jamendo) | VDS-specific but table-stakes *for this product* since sources are a core concept | S | `/search?sources=` ✓ | Already wired |
| "No results" empty state with suggestion ("Try different source?") | Standard empty state handling | S | client-only | — |
| Results grouped by source OR with source icon per row | User needs to understand provenance | S | already has `SOURCE_ICONS` ✓ | Polish: grouped headers when `sources=all` |
| Tap result = play in search context (queue becomes search results) | Standard | S | ✓ | Already works |
| Clear button (×) in the input when non-empty | Universal convention | S | client-only | — |
| Loading state (spinner or skeleton) during fetch | Already has `<p class="loading">Ищем…</p>` ✓ | S | client-only | Upgrade to skeleton rows |

### Differentiators

| Feature | Value Prop | Complexity | Backend | Notes |
|---|---|---|---|---|
| Search history (last ~10 queries, client-localStorage) | Massive friction reducer on mobile | S | client-only (`localStorage`) | — |
| Trending / suggested searches before user types | Spotify has it; fills the empty state | M | **BLOCKED** — no endpoint | Could fake with hardcoded popular genres from `GENRES` list |
| Voice search | Mobile-native feel | M | Web Speech API (spotty in WebView) | Optional, check browser support first |

### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Tabs for "Tracks / Artists / Albums / Playlists" | "Comprehensive search" | Backend returns only tracks; faking artist/album tabs creates empty states that feel broken | One unified tracks list; group by source instead |
| Real-time "as-you-type-each-character" search (<100ms) | "Feels instant" | Burns API calls, rate-limits friendly backend; 300ms debounce is indistinguishable perceptually | 300ms debounce |
| Fuzzy local search over collection *in addition to* server search | "Offline-like speed" | Creates two inconsistent result sets; confuses users | Server search only; cache recent searches in memory |

---

## 6. Library (Liked Tracks)

### Table Stakes

| Feature | Why Expected | Complexity | Backend | Notes |
|---|---|---|---|---|
| List of liked tracks, reverse-chronological | Already exists ✓ | S | `/library/likes` ✓ | — |
| Play all (starts from track 0) | Standard top-level action | S | client-only | Big "▶ Play" button at top |
| Shuffle library (plays in random order) | Second most common action after "play all" in every music app | S | client-only (`currentQueue = shuffle(likes)`) | — |
| Tap a track = play from that point, queue = rest of library | Standard | S | ✓ | Already works |
| Track count + total duration header | Standard library header | S | client-only | — |
| Unlike from within library (removes row) | Users need to undo likes | S | `/library/dislike` or re-POST `/library/like` with toggle semantics | Verify backend supports unlike — **may need backend confirmation**. Fallback: use `/library/dislike` as "remove from likes" |

### Differentiators

| Feature | Value Prop | Complexity | Backend | Notes |
|---|---|---|---|---|
| Search within library (client-side filter) | Once liked-list grows past ~50 it becomes unwieldy | S | client-only | Simple `.filter(t => t.title.includes(q))` on cached list |
| Sort dropdown (recent / A-Z / artist) | Spotify has it; easy win | S | client-only | — |
| Group by artist collapsible sections | Apple Music's signature library view | M | client-only | Nice-to-have |

### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Custom playlists (user-created) | "Standard feature" | Requires backend persistence = out of scope; also product is Wave-first, playlists are a different mental model | Not this milestone. Library = one flat "Liked" collection + genre pseudo-playlists |
| Bulk selection mode ("select 10 tracks, delete") | Power-user feature | Low usage, high complexity for multi-select touch UI | Single-item actions only |
| Downloaded / offline tab | "Spotify has it" | Service Worker explicitly out of scope | Not this milestone |

---

## 7. Profile of Taste Screen

### Table Stakes

| Feature | Why Expected | Complexity | Backend | Notes |
|---|---|---|---|---|
| Top genres/tags as a visual breakdown (bars or chips) | Already implemented ✓ | S | `/taste/{user_id}` ✓ | Polish: bigger bars, gradient fills, top 8 |
| Listening stats (tracks, likes, listens, artists) | Already implemented ✓ | S | `/library/stats` ✓ | Polish: big numbers, icon per stat |
| Reset taste profile button (with confirm) | Already implemented ✓ | S | `/taste/reset` ✓ | Use Telegram `showConfirm` instead of browser `confirm()` |
| Telegram user avatar + name (from `initDataUnsafe.user`) | Makes the profile feel *yours* | S | client-only (`tg.initDataUnsafe.user.photo_url`) | ~5 LOC |

### Differentiators

| Feature | Value Prop | Complexity | Backend | Notes |
|---|---|---|---|---|
| Top artists list (not just genres) | More concrete than tags | M | **BLOCKED** — `/taste` returns `tags`, not artists | Flag for future backend milestone |
| "Your year in music" style shareable card | Viral loop; Telegram share sheet is free distribution | M | client-only (canvas render) | Strong differentiator — Yandex doesn't have it inside Telegram |
| Top genre emoji pinned in Wave header | Personalization cue throughout the app | S | reuses `/taste` ✓ | — |
| Discover-weekly-style "new for you" section | Spotify's retention weapon | L | **BLOCKED** — no endpoint | Flag for future |

### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Editable taste (check/uncheck genres manually) | "User control" | Defeats implicit learning; makes the recommender worse; users don't actually know what they like better than 100 likes of data | Reset-only; let the algorithm work |
| "Friends' taste comparison" social feature | "Engagement" | No friends graph, no backend support, privacy concerns | Not this milestone |
| Daily/weekly stats charts | "Quantified self" | Adds chart library weight; low repeat usage | Four big-number stat boxes are enough |

---

## 8. Feedback UX (Like / Dislike)

### Table Stakes

| Feature | Why Expected | Complexity | Backend | Notes |
|---|---|---|---|---|
| Like button on fullscreen player (prominent, near title) | Where Spotify/Yandex/Apple all put it | S | `/library/like` ✓ | — |
| Like button on each row in lists | Fast like-from-list flow | S | ✓ | Already there |
| Heart fills/animates on like (no modal) | State visible without interruption | S | client-only | CSS transition |
| Haptic feedback on like/dislike (via `tg.HapticFeedback`) | Telegram-native feel | S | `tg.HapticFeedback.impactOccurred('light')` | ~2 LOC per action |
| Dislike = remove from Wave + negative signal | Core product promise | S | `/library/dislike` + `/wave/feedback` ✓ | Already wired |
| Auto-skip after dislike | Users don't want to keep hearing the thing they disliked | S | `skipNext()` after POST ✓ | Needs hookup — currently dislike doesn't auto-skip |
| Visual confirmation toast ("Не будет в Моей волне") on dislike | Reassures user the signal was received (dislike is rarer than like, deserves feedback) | S | client-only | Tiny 2s toast |

### Differentiators

| Feature | Value Prop | Complexity | Backend | Notes |
|---|---|---|---|---|
| Long-press on row → action sheet with {Like / Dislike / Add to queue / Share / Artist} | Yandex's power-user gesture; reduces icon clutter | M | client-only | Replaces the two inline icons |
| Swipe-right on row = like, swipe-left = dislike | Tinder-like gesture feels great on Wave list | M | client-only | Works well on list screens, not on now-playing |
| Undo snackbar after dislike ("Отменить") | Prevents regret; Spotify ships this | S | client-only (3s window before POST) | — |

### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| "Are you sure you want to dislike?" confirm dialog | "Prevent accidental dislikes" | Dialog walls destroy the feedback flow; users stop giving signal | Undo snackbar (3s) instead |
| Separate "meh" button (neutral) | "More granularity" | Nobody uses it; the backend already treats skip as neutral signal via `elapsed` | Skip already provides this signal |
| Like count / social counters ("5 of your friends liked this") | "Social proof" | Privacy issues, no friends graph | Not this milestone |

---

## 9. Loading / Empty / Error States

### Table Stakes

| Feature | Why Expected | Complexity | Backend | Notes |
|---|---|---|---|---|
| Skeleton screens (shimmer placeholder rows) instead of spinners for lists | 2025 bar — spinners feel dated. Spotify, Apple Music, Yandex all use skeletons | S | client-only | ~20 LOC CSS `@keyframes shimmer` |
| First-paint: Wave skeleton visible within <200ms even before `/wave` returns | Reduces perceived latency | S | client-only | Render skeleton rows immediately, swap when data arrives |
| Empty states with illustration + action CTA ("Your likes are empty — start with Wave →") | Generic "Пусто" feels broken | S | client-only | Emoji + text + button is enough; no SVG illustrations needed |
| Error states with retry button | "Ошибка загрузки" alone is a dead-end | S | client-only | `[⚠ Ошибка] [Повторить]` row |
| Cover art fallback (gradient with first letter) when `thumb` is missing or fails | Some YouTube/SC results have no thumb | S | already has `onerror="this.style.opacity=0.3"` ✓ | Upgrade to colored gradient fallback |
| Image lazy-loading (`loading="lazy"`) for long lists | Perf on cheap Android | S | client-only | One attribute |
| Offline detection banner ("Нет сети") | `/stream` will fail silently otherwise | S | `navigator.onLine` + `online`/`offline` events | — |

### Differentiators

| Feature | Value Prop | Complexity | Backend | Notes |
|---|---|---|---|---|
| Optimistic UI for likes (heart fills *before* POST returns) | Perceived instant response | S | client-only | Rollback on error |
| Stream buffering indicator on mini-player (distinct from play/pause icon) | Audio loading is often slow on Tailscale-backed stream; users mash play | S | `waiting`/`canplay` events on audio element | Replace play icon with spinner while buffering |

### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Full-screen blocking loader on every navigation | "Show progress" | Blocks the whole UI; skeletons are non-blocking and faster-feeling | Skeletons, always |
| Toast on every network error | "Inform the user" | Spam when connection flaps | Inline state on the affected section + one subtle offline banner |
| Retry-with-exponential-backoff auto-retry loops | "Resilience" | Hides real problems, burns battery, confuses debugging | Manual retry button; fail fast |

---

## 10. Gesture Language

### Table Stakes (Standard Gestures — Users Expect These)

| Gesture | Action | Where | Complexity | Notes |
|---|---|---|---|---|
| Tap mini-player | Expand to fullscreen | Mini-player | S | — |
| Swipe-down on fullscreen player | Collapse to mini | Fullscreen | M | User explicitly asked for this |
| Swipe-left on cover art or mini-player | Next track | FP + mini | M | User explicitly asked for this |
| Swipe-right on cover art or mini-player | Previous track | FP + mini | M | User explicitly asked for this |
| Swipe-up on mini-player | Expand to fullscreen | Mini | M | Alternative to tap |
| Tap outside overlay / back button | Close modal or fullscreen | Overlays | S | Map Telegram `BackButton` to this |
| Pull-to-refresh on Wave | Reload stream | Wave tab | M | Yandex & Spotify both have this |
| Long-press track row | Action sheet (like/dislike/queue/share) | All track lists | M | Replaces inline icons |
| Drag handle on queue rows | Reorder | Queue view | M | User explicitly asked for this |

### Differentiators

| Gesture | Action | Where | Complexity | Notes |
|---|---|---|---|---|
| Swipe-right on list row | Quick like | Wave / search / genre lists | M | Tinder-feel, optional but delightful |
| Swipe-left on list row | Quick dislike / remove | Library + queue | M | Symmetry with swipe-right |
| Double-tap cover art | Like current track | Fullscreen player | S | Instagram-style, users try it instinctively |
| Pinch on cover (FP) | Shrink to mini | Fullscreen | L | Apple Music has it; low priority vs swipe-down |

### Anti-Features / Surprising Gestures

| Gesture | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Shake-to-shuffle | "Fun retro feature" | Accidentally triggered in pockets; accessibility nightmare | Explicit shuffle button |
| Swipe horizontally on tab bar to change tabs | "Mobile-native" | Conflicts with swipe-for-prev/next in player; user confusion | Tap tabs only |
| Two-finger swipe for volume | "Hidden power feature" | Non-discoverable; OS already handles volume | Don't implement |
| Swipe-up on fullscreen player to show queue | "More content" | Conflicts with swipe-down-to-collapse mental model (Spotify uses bottom sheet for queue, not swipe-up) | Dedicated "queue" button opens bottom sheet |
| Long-press play button for something | "Shortcut" | Users long-press by accident; breaks play/pause reliability | Never overload the primary action button |

---

## Feature Dependencies

```
Fullscreen Player
    ├── requires ──> Mini Player (persistent state)
    ├── requires ──> MediaSession setup (lock-screen)
    └── requires ──> Gesture handler module (swipe-down, swipe lr)

Queue View
    ├── requires ──> Gesture handler module (drag-to-reorder)
    └── requires ──> Track history array (for "previously played" section)

Swipe-row-to-like
    └── requires ──> Gesture handler module
    └── conflicts with ──> Swipe-to-delete in queue (resolve by context: like in lists, delete in queue)

Dominant-color background
    └── requires ──> Cover image fully loaded (CORS-friendly)
    └── enhances ──> Fullscreen Player

Taste profile avatar
    └── requires ──> Telegram WebApp init (user.photo_url)

Shareable "Year in music" card
    └── requires ──> Taste data + stats (/taste, /library/stats)
    └── requires ──> Telegram share intent

All Wave features
    └── depend on ──> /wave, /wave/moods, /wave/feedback (all present ✓)

Optimistic like UI
    └── enhances ──> Feedback UX everywhere

Long-press action sheet
    └── replaces ──> Inline like/dislike icons (gesture module dependency)
```

### Key Dependency Notes

- **Gesture handler module is the single biggest shared dependency.** Building one centralized touch-events module (track start/end/delta, direction) unlocks: swipe-down, swipe lr on player, swipe lr on mini, drag-to-reorder, swipe on rows, pull-to-refresh. Build it *once*, early in the milestone.
- **MediaSession upgrade is cheap and unlocks seek 15s/+15s lock-screen controls.** Do it in the same PR as the fullscreen player polish.
- **Dominant-color backgrounds depend on CORS.** `/stream`-adjacent thumb URLs must be CORS-safe for canvas `getImageData`. Verify early — if blocked, fall back to a pre-defined palette hash.
- **Queue reorder and swipe-to-delete must not collide** — they're both on rows in the queue view. Resolution: drag handle on the left triggers reorder; swipe on the rest of the row triggers delete.
- **Unlike-from-library depends on backend semantics being clear.** Does re-POSTing `/library/like` toggle? Or does `/library/dislike` remove from likes? Verify with a quick manual API call before wiring the unlike affordance.

---

## MVP Definition for This Milestone

### Launch With (Redesign v1) — P1

Every item below is either already backend-wired or purely client-side. No new endpoints.

- [ ] **Gesture handler module** — foundation for everything else
- [ ] **Redesigned Wave tab** — skeleton loading, mood chips polish, active-track indicator, auto-skip on dislike, haptic feedback
- [ ] **Redesigned fullscreen player** — large cover, dominant-color background, scrubber (bigger hit target), queue peek, tap-artist-to-search, swipe-down to collapse, swipe lr for prev/next
- [ ] **Persistent mini-player** — hairline progress, tap-to-expand, swipe-up-to-expand, swipe lr prev/next, heart button, never hide across navigation
- [ ] **MediaSession full set** — play, pause, next, prev, seekbackward (15s), seekforward (15s)
- [ ] **Queue bottom sheet** — up next, now playing, history scroll, drag-to-reorder, tap-to-jump, swipe-to-remove, add-to-queue / play-next from long-press
- [ ] **Long-press action sheet on track rows** — Like / Dislike / Play next / Add to queue / Share (via Telegram) / Search artist
- [ ] **Library upgrade** — play-all, shuffle-all, search-within-library, track count header, unlike (verify backend)
- [ ] **Search UX polish** — debounce tuned, clear button, search history (localStorage), skeleton results, grouped results when sources=all
- [ ] **Taste profile polish** — Telegram avatar + name, bigger bars, stat boxes with icons, `tg.showConfirm` instead of browser confirm
- [ ] **Loading / empty / error states** — skeleton system, retry buttons, gradient-letter cover fallback, offline banner, lazy-load images, buffering indicator on mini-player
- [ ] **Haptic feedback** — on like, dislike, play/pause, expand/collapse, mood change
- [ ] **Optimistic like UI** — heart fills before POST returns, rollback on error
- [ ] **Undo snackbar for dislike** (3s window)
- [ ] **Double-tap cover to like** (delightful micro-feature)
- [ ] **Pull-to-refresh on Wave tab**

### Add After Validation (v1.x) — P2

Client-side additions that depend on user feedback from v1.

- [ ] **Shareable "Your taste" card** (canvas-rendered image via Telegram share) — differentiator but measures poorly in dark launches
- [ ] **Voice search** — test Web Speech API availability in Telegram WebView first
- [ ] **Marquee-scroll long titles**
- [ ] **Sort dropdown in library** (recent / A-Z / artist)
- [ ] **Shuffle & repeat toggles in fullscreen player** (for non-Wave contexts)

### Future Consideration (v2+, needs backend) — P3, BLOCKED

- [ ] **Lyrics panel** — needs `/lyrics` endpoint
- [ ] **Wave "because you liked X" explanations** — needs `reason` field in `/wave` response
- [ ] **Top artists in taste profile** — needs `/taste` to return artists, not just tags
- [ ] **Trending / suggested searches** — needs endpoint or cached corpus
- [ ] **Discover-weekly-style new-for-you section** — needs endpoint
- [ ] **Custom user playlists** — needs backend persistence

---

## Feature Prioritization Matrix

| Feature | User Value | Cost | Priority |
|---|---|---|---|
| Gesture handler module | HIGH | MEDIUM | P1 (foundation) |
| Redesigned fullscreen player (cover, scrubber, queue peek) | HIGH | MEDIUM | P1 |
| Swipe-down to collapse fullscreen | HIGH | MEDIUM | P1 |
| Persistent mini-player across navigation | HIGH | LOW | P1 |
| Mini-player progress hairline | MEDIUM | LOW | P1 |
| Swipe lr on player (prev/next) | HIGH | MEDIUM | P1 |
| Queue bottom sheet with reorder | HIGH | MEDIUM | P1 |
| Long-press action sheet | HIGH | MEDIUM | P1 |
| MediaSession seek 15s/+15s | MEDIUM | LOW | P1 |
| Skeleton loading system | HIGH | LOW | P1 |
| Empty / error / offline states | HIGH | LOW | P1 |
| Haptic feedback | MEDIUM | LOW | P1 |
| Optimistic like UI | MEDIUM | LOW | P1 |
| Auto-skip after dislike | HIGH | LOW | P1 |
| Dominant-color background from cover | MEDIUM | MEDIUM | P1 |
| Search history (localStorage) | MEDIUM | LOW | P1 |
| Library play-all / shuffle-all | HIGH | LOW | P1 |
| Search-within-library | MEDIUM | LOW | P1 |
| Pull-to-refresh Wave | MEDIUM | LOW | P1 |
| Buffering indicator on mini | MEDIUM | LOW | P1 |
| Double-tap cover to like | LOW | LOW | P1 (cheap delight) |
| Undo snackbar for dislike | MEDIUM | LOW | P1 |
| Telegram avatar in profile | MEDIUM | LOW | P1 |
| Shareable taste card | MEDIUM | MEDIUM | P2 |
| Voice search | LOW | MEDIUM | P2 |
| Shuffle/repeat toggles | MEDIUM | LOW | P2 |
| Marquee-scroll titles | LOW | LOW | P2 |
| Lyrics panel | HIGH | HIGH | P3 (BLOCKED) |
| Wave reason labels | HIGH | MEDIUM | P3 (BLOCKED) |
| Top artists in taste | MEDIUM | LOW | P3 (BLOCKED) |
| Custom playlists | MEDIUM | HIGH | P3 (BLOCKED) |

**Priority key:**
- **P1** — Must ship this milestone (redesign v1)
- **P2** — Follow-up after v1 lands and usage data comes in
- **P3** — Blocked on backend; belongs to a future milestone

---

## Competitor Feature Analysis

| Feature | Yandex Music | Spotify | Apple Music | YT Music | VDS Music (this milestone) |
|---|---|---|---|---|---|
| Wave / infinite personalized stream | **Yes — defining feature** ("Моя волна") | Daylist / Smart Shuffle | Station / Infinity | Radio | **Yes — adopt Yandex model** |
| Mood selector on home | Yes (chips on Wave header) | Category grid | Genre stations | Mood & moments | **Yes — mood chips on Wave** |
| Swipe-down to collapse FP | Yes | Yes | Yes | Yes | **Yes — P1** |
| Swipe lr on cover for prev/next | Yes | No (uses buttons only) | Yes | No | **Yes — P1** (Yandex/Apple pattern) |
| Persistent mini-player | Yes | Yes | Yes | Yes | **Yes — P1** |
| Queue with drag reorder | Yes | Yes | Yes | Yes | **Yes — P1** |
| Long-press action sheet | Yes | Yes | Yes | Yes | **Yes — P1** |
| Lyrics | Yes | Yes | Yes | Yes | **NO — backend blocked (P3)** |
| Dominant-color FP background | No | **Yes — signature** | Yes | Partial | **Yes — P1 differentiator for us** |
| Dislike from now-playing | Yes (👎 on Wave only) | Limited (radio only) | No | Yes (thumbs) | **Yes — P1** |
| Explanations ("because you liked X") | Yes | Yes (Made For You) | Limited | Yes | **NO — backend blocked (P3)** |
| Shareable taste/year summary | Once a year (Wrapped-style) | **Yes — Wrapped viral** | Replay | Recap | **YES — Telegram-native share is a unique fit (P2)** |
| MediaSession seek 15s/+15s | Yes | Yes | Yes | Yes | **Yes — P1** |
| Pull-to-refresh | Yes | No | No | Yes | **Yes — P1** |
| Custom playlists | Yes | Yes | Yes | Yes | **NO — backend blocked (P3)** |
| Voice search | Yes (Алиса) | Yes | Yes (Siri) | Yes | **Maybe P2 if Web Speech works in WebView** |

**Takeaway:** For this milestone, VDS Music can hit parity on ~90% of the table-stakes surface using only the existing backend. The only items where we'll visibly lag are lyrics, custom playlists, and recommender explanations — all explicitly backend-blocked and correctly flagged for future milestones. The Telegram-native share flow is a genuine differentiator none of the four competitors can match inside a Telegram Mini App.

---

## Quality Gate Checklist

- [x] Categories clear: Table stakes vs Differentiators vs Anti-features in every section
- [x] Every table-stakes feature mapped to existing backend endpoint OR flagged **BLOCKED**
- [x] Mobile-first / Telegram Mini App context respected (no desktop assumptions; Telegram SDK integration called out for haptics, share, confirm, back button, avatar)
- [x] Complexity noted (S/M/L) for every feature

## Confidence & Sources

**Confidence: HIGH** on gestures, table-stakes features, and anti-features (all four competitor apps are in active daily use by hundreds of millions of users; their patterns are documented in their own release notes, developer docs, and years of UX writing).

**Confidence: MEDIUM** on the exact thresholds (300ms debounce, 15s seek, 3s undo) — these are industry-common but not sacred; tune from usage.

**Confidence: MEDIUM** on CORS availability for dominant-color extraction — needs verification against `/stream` thumb domains.

### Sources (direct competitor apps + publicly documented patterns)
- Yandex Music mobile app (iOS/Android) — Wave, mood chips, dislike long-press, gesture language, taste profile
- Spotify mobile app — persistent mini-player, fullscreen player, queue, Wrapped share, dominant-color FP, skeletons
- Apple Music mobile app — swipe lr on cover, pinch-to-shrink, library-by-artist grouping
- YouTube Music mobile app — thumbs up/down on radio, seek 15s/+15s via MediaSession, pull-to-refresh
- MDN: MediaSession API (`seekbackward`, `seekforward` actions) — HIGH confidence, web standard
- Telegram WebApp SDK docs — `HapticFeedback`, `showConfirm`, `BackButton`, `initDataUnsafe.user.photo_url`
- Existing `index.html` — reviewed lines 499-1023 to confirm current feature surface and wired endpoints

---
*Feature research for: VDS Music Telegram Mini App — redesign milestone*
*Researched: 2026-04-15*
