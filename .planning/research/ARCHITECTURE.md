# Architecture Research: VDS Music (Telegram Mini App Audio Player)

**Domain:** Single-file, no-bundler Telegram Mini App audio player
**Researched:** 2026-04-15
**Confidence:** HIGH

## TL;DR

At 2–3k lines, you do **not** need a framework and you do **not** need signals. You need exactly three things the current file is missing:

1. A **single mutable `state` object** owned by a `Store` IIFE with a pub/sub bus.
2. A **`PlaybackEngine` IIFE** that owns both `<audio>` elements and is the *only* writer of playback state — everything else subscribes.
3. A **view layer split into IIFE components** that render to their own root `<div>` via template literals, and re-render only on the store keys they subscribe to.

Everything else (routing, queue, wave feedback, API client, future sources) plugs into those three. The current file fails because state is scattered across module-level `let`s and `window.*` globals, and both UI and playback logic write to the DOM directly from async handlers — which is exactly what will collapse at 2–3k lines during a redesign.

---

## 1. Module boundaries inside one HTML file

**Pattern chosen: Revealing-IIFE modules sharing a single namespace.**

Inside one `<script>` (or better, several `<script>` blocks in declaration order — they share globals but give you visual separation), define each module as:

```js
const Store = (() => {
  const state = { /* ... */ };
  const subs = new Map();  // key -> Set<fn>
  function get(key)        { return state[key]; }
  function set(patch)      { Object.assign(state, patch); notify(Object.keys(patch)); }
  function subscribe(keys, fn) { /* ... */ }
  function notify(keys)    { /* call fns whose keys intersect */ }
  return { get, set, subscribe, state };
})();
```

Why this over alternatives at 2–3k lines:

| Option | Verdict |
|---|---|
| Plain top-level `let`s (current) | Fails — no encapsulation, no subscription, no way to find who mutates what |
| IIFE modules (recommended) | Zero runtime cost, clean `Module.publicFn()` call sites, trivially greppable |
| ES modules via `<script type="module">` with blob URLs | Works on Pages but adds complexity and async init ordering issues for no real gain in a single file |
| `@preact/signals-core` via CDN | Tempting but overkill: 2k lines do not have enough components to pay back the mental tax of fine-grained reactivity, and you lose the "I can read this file top-to-bottom" property that makes single-file projects worth it |
| Lightweight custom signals (30 LOC) | Acceptable but strictly inferior to `Store.subscribe(['currentTrack'], fn)` at this scale — signals shine at 50+ reactive components, you'll have ~12 |

**Module list (target file layout, single script block, ~2.5k lines):**

```
<style>              ~600 lines — design tokens + component CSS
<body markup>        ~80 lines — just skeleton roots, components render into them
<script>
  Config             API base, constants, feature flags              ~30
  Utils              escapeHtml, fmtTime, haptic, qs/qsa              ~60
  Telegram           WebApp SDK wrapper, BackButton, theme            ~80
  Store              state + pub/sub                                 ~80
  Api                fetch wrapper, dedupe, retry, error envelope    ~150
  PlaybackEngine     dual <audio>, queue semantics, MediaSession     ~350
  Feedback           fire-and-forget event queue                     ~80
  Router             screen state machine + BackButton glue          ~120
  components/
    MiniPlayer       subscribes: currentTrack, isPlaying, position   ~120
    FullscreenPlayer subscribes: currentTrack, isPlaying, position   ~250
    QueuePanel       subscribes: queue, currentIndex                 ~150
    TrackCard        pure render fn (no state)                       ~60
    HomeScreen       Wave + moods + genres                           ~250
    SearchScreen     source chips + results                          ~180
    LibraryScreen    likes + stats                                   ~120
    ProfileScreen    taste bars                                      ~100
  Boot               wire everything, initial fetches                ~60
</script>
```

The ordering is load-bearing: every module below `Store` can reference it; `PlaybackEngine` is defined before `components/` so components can call `PlaybackEngine.play(track)` and subscribe to store keys it writes.

---

## 2. State management

**One `Store` object. One `state` blob. Key-scoped subscriptions. No signals lib.**

```js
const state = {
  // Playback (owned by PlaybackEngine — only it writes)
  currentTrack: null,
  isPlaying: false,
  position: 0,          // seconds
  duration: 0,
  playbackCtx: 'wave',  // 'wave' | 'genre' | 'search' | 'library'

  // Queue (owned by PlaybackEngine)
  queue: [],            // mainline queue (wave/genre/search/etc.)
  queueIndex: -1,
  userQueue: [],        // explicit "play next" injection
  history: [],          // for prev button across ctx switches

  // Wave-specific (owned by HomeScreen → read by PlaybackEngine for auto-extend)
  waveMood: '',
  waveEnded: false,

  // UI (owned by Router)
  screen: 'home',       // 'home' | 'search' | 'library' | 'profile'
  fullscreenOpen: false,
  queuePanelOpen: false,

  // Per-user (owned by Api/boot)
  userId: 0,
  likedIds: new Set(),  // for instant heart state
  taste: null,
  stats: null,
};
```

**Rule: exactly one module writes each key.** Violating this is the #1 cause of the current file's redesign brittleness. Document the owner in a comment next to each key.

**Subscription API** — key-scoped, not selector-based:

```js
Store.subscribe(['currentTrack', 'isPlaying'], ({ currentTrack, isPlaying }) => {
  MiniPlayer.render(currentTrack, isPlaying);
});
```

Why key subscriptions beat signals here: your update frequency is dominated by `position` ticks (4–10 Hz during playback). With a Store you simply don't subscribe `MiniPlayer` to `position` — instead `MiniPlayer` reads `PlaybackEngine.getPosition()` directly on a `requestAnimationFrame` loop while visible. Signals would force you to thread computed values just to avoid re-rendering on every tick.

**The "now playing" propagation problem** is solved by:
- MiniPlayer subscribes `['currentTrack', 'isPlaying']` → re-renders title/art/play-icon
- MiniPlayer runs its own `rAF` loop reading `PlaybackEngine.getPosition()` when mounted
- FullscreenPlayer does the same when open; when closed it stops its `rAF`
- QueuePanel subscribes `['queue', 'queueIndex', 'userQueue']`
- TrackCard instances in lists don't subscribe at all — they re-render when their parent screen re-renders. The "currently playing row" highlight is handled by MiniPlayer toggling a CSS class on `[data-track-id]` elements after each track change. One DOM query, no reactive layer.

This is the architecture Spotify/Yandex web clients use internally too — a central `PlaybackController` + `PlayerStore` with pub/sub, not per-component signals.

---

## 3. Routing

**Pattern: Screen state machine + Telegram BackButton as the only back affordance.**

Do not use hash routing or History API in a Mini App. Telegram renders inside a WebView where:
- The user has no visible browser URL bar.
- `popstate` is unreliable across Android/iOS Telegram clients.
- `tg.BackButton` is the one true back control, and it has to be programmatically shown/hidden/rewired per screen.

```js
const Router = (() => {
  const stack = [];  // screens pushed for "back" semantics
  const tg = window.Telegram.WebApp;

  function go(screen, opts = {}) {
    const prev = Store.get('screen');
    if (opts.push !== false && prev !== screen) stack.push(prev);
    Store.set({ screen });
    updateBackButton();
  }

  function back() {
    // Precedence: fullscreen overlay > queue panel > stack
    if (Store.get('queuePanelOpen'))  return Store.set({ queuePanelOpen: false });
    if (Store.get('fullscreenOpen'))  return Store.set({ fullscreenOpen: false });
    if (stack.length)                 return Store.set({ screen: stack.pop() });
    tg.close();
  }

  function updateBackButton() {
    const canGoBack = Store.get('fullscreenOpen')
                   || Store.get('queuePanelOpen')
                   || stack.length > 0;
    canGoBack ? tg.BackButton.show() : tg.BackButton.hide();
  }

  tg.BackButton.onClick(back);
  Store.subscribe(['screen','fullscreenOpen','queuePanelOpen'], updateBackButton);
  return { go, back };
})();
```

**Critical rules:**
- Fullscreen player is an *overlay*, not a screen. Opening it does NOT push onto `stack`, it flips `fullscreenOpen: true`. BackButton closes the overlay first before unwinding screens. This matches Spotify/Yandex behavior and is what users expect.
- Queue panel is the same: overlay, not screen.
- The `fullscreenOpen: true` state must survive screen switches (user can close fullscreen, switch to Library, reopen fullscreen from MiniPlayer — same playback).
- `tg.BackButton` is idempotent; always derive its visibility from state, never imperatively toggle from multiple places.

---

## 4. Component boundaries

**Pattern: IIFE component modules with `mount(root)`, `render(state)`, optional `destroy()`. Template literals for HTML, one DOM query to hydrate event handlers.**

No `<template>` elements — they save no meaningful LOC and add indirection. No DOM-building functions — they're 3× the code of template literals. Template literals + `innerHTML` are fine here because:
- Telegram WebView is a modern Chromium/WebKit.
- You control all content (API returns JSON, not HTML), and you `escapeHtml` strings.
- At 60–200 items per list, `innerHTML` assignment is faster than `document.createElement` loops and dramatically more readable.

**Natural component inventory:**

| Component | Mounts at | Subscribes to | Owner writes |
|---|---|---|---|
| `MiniPlayer` | `#mini-player-root` (always mounted) | `currentTrack, isPlaying` | nothing |
| `FullscreenPlayer` | `#fullscreen-root` (always mounted, CSS-hidden) | `currentTrack, isPlaying, duration` | `fullscreenOpen` (via close btn) |
| `QueuePanel` | `#queue-root` (always mounted, CSS-hidden) | `queue, queueIndex, userQueue` | `queuePanelOpen` |
| `HomeScreen` | `#screen-root` (when `screen==='home'`) | `waveMood, likedIds` | `waveMood` |
| `SearchScreen` | `#screen-root` (when `screen==='search'`) | `likedIds` | — |
| `LibraryScreen` | `#screen-root` (when `screen==='library'`) | `likedIds, stats` | — |
| `ProfileScreen` | `#screen-root` (when `screen==='profile'`) | `taste, stats` | — |
| `TrackCard` | pure fn `renderTrack(t, ctx) -> html string` | — | — |
| `MoodPicker` | child of HomeScreen | `waveMood` | `waveMood` |
| `SourceChips` | child of SearchScreen | local | local |
| `TasteBars` | child of ProfileScreen | `taste` | — |
| `BottomNav` | `#nav-root` (always mounted) | `screen` | `screen` (via taps) |

**Component skeleton:**

```js
const MiniPlayer = (() => {
  let root, rafId;

  function template(track, isPlaying) {
    if (!track) return '';
    return `
      <div class="mp-thumb"><img src="${escapeHtml(track.thumb||'')}"></div>
      <div class="mp-info">
        <div class="mp-title">${escapeHtml(track.title||'')}</div>
        <div class="mp-artist">${escapeHtml(track.artist||'')}</div>
      </div>
      <button class="mp-play" data-action="toggle">${isPlaying?'⏸':'▶'}</button>
      <div class="mp-progress"><div class="mp-progress-fill" id="mp-fill"></div></div>
    `;
  }

  function render() {
    const t = Store.get('currentTrack');
    root.classList.toggle('visible', !!t);
    root.innerHTML = template(t, Store.get('isPlaying'));
    hydrate();
    tickProgress();
  }

  function hydrate() {
    root.querySelector('[data-action="toggle"]')?.addEventListener('click', PlaybackEngine.toggle);
    root.addEventListener('click', e => {
      if (e.target.closest('[data-action]')) return;
      Store.set({ fullscreenOpen: true });
    }, { once: true });
  }

  function tickProgress() {
    cancelAnimationFrame(rafId);
    const fill = root.querySelector('#mp-fill');
    const loop = () => {
      const pos = PlaybackEngine.getPosition();
      const dur = PlaybackEngine.getDuration();
      if (fill && dur) fill.style.width = `${(pos/dur)*100}%`;
      rafId = requestAnimationFrame(loop);
    };
    loop();
  }

  function mount(el) {
    root = el;
    Store.subscribe(['currentTrack','isPlaying'], render);
    render();
  }
  return { mount };
})();
```

**Delegated event handling**: for lists (track cards), attach one `click` listener on the list root and read `e.target.closest('[data-track-idx]')`. Do not inline `onclick="foo(...)"` — this is the worst part of the current file, it forces `JSON.stringify` + attribute escaping and makes it impossible to pass non-serializable data.

---

## 5. Audio + queue architecture

**PlaybackEngine is a module that owns both `<audio>` elements and is the sole writer of `currentTrack`, `isPlaying`, `position`, `duration`, `queue`, `queueIndex`, `userQueue`, `history`.**

### Queue semantics

Three separate concepts:

1. **Mainline queue** (`state.queue`) — the source-context tracks: wave, genre, search result, library. Mutable by context: when user taps a search result, mainline queue becomes search tracks.
2. **User queue** (`state.userQueue`) — explicit "play next" / "add to queue" injections. Always drained *before* advancing the mainline.
3. **History** (`state.history`) — every track that actually played, in order. `prev` pops from history.

**Next-track resolution:**

```js
function next() {
  sendFeedback('skip');  // fire-and-forget
  // 1. user queue has priority
  if (state.userQueue.length) {
    const t = state.userQueue.shift();
    Store.set({ userQueue: [...state.userQueue] });
    return playTrack(t, { fromUserQueue: true });
  }
  // 2. mainline advance
  const nextIdx = state.queueIndex + 1;
  if (nextIdx < state.queue.length) {
    return playTrack(state.queue[nextIdx], { queueIndex: nextIdx });
  }
  // 3. wave auto-extend (only in wave ctx)
  if (state.playbackCtx === 'wave' && !state.waveEnded) {
    return Api.loadWave(state.waveMood).then(tracks => {
      Store.set({ queue: [...state.queue, ...tracks] });
      return next();  // recurse once
    });
  }
  // 4. dead end
  Store.set({ isPlaying: false });
}

function onTrackEnded() { sendFeedback('finish'); next(); }
```

**Interruption semantics:**
- User plays from search while wave is playing → mainline queue is *replaced*, `playbackCtx: 'search'`, history preserved. To go "back to wave" user re-enters Home tab (wave queue is regenerated from server state). This is the Spotify/Yandex behavior and avoids trying to keep parallel queues alive.
- User taps "Play next" on a track while wave is playing → inserted into `userQueue`, wave keeps playing in background. Next skip plays the injected track, then resumes wave. Mainline `queueIndex` is *not* advanced across userQueue plays.

**Auto-extend:** trigger on `queueIndex >= queue.length - 3`, not on `queueIndex === queue.length - 1`. Prefetch, don't block.

**Dual-buffer preloading:** keep the current `audioA`/`audioB` pattern, but encapsulate it. PlaybackEngine holds references internally; nothing else touches `<audio>` elements.

**MediaSession:** lives inside PlaybackEngine, updated from `playTrack()`. Action handlers call `PlaybackEngine.toggle/next/prev` — never the UI layer. This is what gives you reliable lock-screen controls: one call site, one owner.

**Position state**: do NOT put `position` in the Store as a reactive key. Expose `PlaybackEngine.getPosition()` and let whoever needs smooth progress read it in a `rAF` loop while visible. Writing `position` into the Store at 10 Hz would thrash every subscriber.

---

## 6. Data flow to backend

**Api module:**

```js
const Api = (() => {
  const base = 'https://api.vdsmusic.ru:8443';
  const inflight = new Map();  // key -> Promise (dedupe)

  async function req(path, opts = {}) {
    const key = `${opts.method||'GET'} ${path} ${JSON.stringify(opts.body||'')}`;
    if (inflight.has(key)) return inflight.get(key);
    const p = (async () => {
      for (let attempt = 0; attempt < 3; attempt++) {
        try {
          const r = await fetch(base + path, {
            method: opts.method || 'GET',
            headers: opts.body ? { 'Content-Type': 'application/json' } : undefined,
            body: opts.body ? JSON.stringify(opts.body) : undefined,
          });
          if (!r.ok) throw new Error(`HTTP ${r.status}`);
          return await r.json();
        } catch (e) {
          if (attempt === 2) throw e;
          await new Promise(r => setTimeout(r, 300 * (attempt + 1)));
        }
      }
    })();
    inflight.set(key, p);
    p.finally(() => inflight.delete(key));
    return p;
  }

  return {
    wave:        (mood)     => req(`/wave?user_id=${USER_ID}&mood=${encodeURIComponent(mood)}`),
    waveMoods:   ()         => req(`/wave/moods?user_id=${USER_ID}`),
    search:      (q, src)   => req(`/search?q=${encodeURIComponent(q)}&user_id=${USER_ID}${src?`&sources=${src}`:''}`),
    genre:       (g)        => req(`/genre?q=${encodeURIComponent(g)}&user_id=${USER_ID}`),
    likes:       ()         => req(`/library/likes?user_id=${USER_ID}`),
    stats:       ()         => req(`/library/stats?user_id=${USER_ID}`),
    taste:       ()         => req(`/taste/${USER_ID}`),
    tasteReset:  ()         => req(`/taste/reset?user_id=${USER_ID}`, { method: 'POST' }),
    like:        (t)        => req(`/library/like`, { method: 'POST', body: { user_id: USER_ID, track_id: t.id, track: t } }),
    dislike:     (t)        => req(`/library/dislike`, { method: 'POST', body: { user_id: USER_ID, track_id: t.id } }),
    feedback:    (ev)       => req(`/wave/feedback`, { method: 'POST', body: ev }),  // routed via Feedback module
  };
})();
```

**Feedback module (fire-and-forget but reliable):**

Wave feedback events (like/dislike/skip/finish) must not block UI and must not be lost on rapid navigation. Route them through an in-memory queue with localStorage backing:

```js
const Feedback = (() => {
  const KEY = 'vds_feedback_queue_v1';
  let queue = JSON.parse(localStorage.getItem(KEY) || '[]');
  let flushing = false;

  function persist() { localStorage.setItem(KEY, JSON.stringify(queue)); }

  function send(event) {
    queue.push({ ...event, ts: Date.now() });
    persist();
    flush();
  }

  async function flush() {
    if (flushing || !queue.length) return;
    flushing = true;
    while (queue.length) {
      const ev = queue[0];
      try {
        await Api.feedback(ev);
        queue.shift();
        persist();
      } catch (e) {
        break;  // retry on next send() or next visibility change
      }
    }
    flushing = false;
  }

  document.addEventListener('visibilitychange', () => { if (!document.hidden) flush(); });
  window.addEventListener('online', flush);
  return { send };
})();
```

This guarantees feedback survives Telegram backgrounding and brief network drops — the actual failure mode on mobile. The current file uses raw `fetch(...).catch(()=>{})` which silently loses events.

**Caching:** deliberately thin. Likes list caches for the session only; taste cached 60s. Wave is never cached (server is authoritative). No IndexedDB, no SW — out of scope.

---

## 7. Build order inside this milestone

Dependency-aware order. Each step must be shippable before the next begins.

```
1. Design tokens (CSS variables, type scale, spacing, radii, shadows, gradients)
   └─ no JS changes. Visual regression is immediate and cheap to iterate.

2. Layout shell (body skeleton + screen-root / mini-player-root / fullscreen-root
   / queue-root / nav-root containers; ScreenSwitch CSS only)
   └─ depends on: tokens.

3. Store + Router + Telegram wrapper
   └─ depends on: shell (needs roots to toggle). Testable by logging state changes.

4. Api + Feedback module
   └─ depends on: nothing in UI. Can be unit-tested in DevTools console.

5. PlaybackEngine (dual buffer, queue semantics, MediaSession)
   └─ depends on: Store, Api. No UI yet — drive from console:
       PlaybackEngine.playContext('wave', tracks). This is the correct
       order because playback is the riskiest integration and must be
       validated headless before any component consumes it.

6. MiniPlayer + FullscreenPlayer (in parallel with #5 once PlaybackEngine API is frozen)
   └─ depends on: PlaybackEngine, Store, tokens. Both subscribe to same keys —
       build them together to catch the shared-state contract early.

7. BottomNav + HomeScreen (wave + moods)
   └─ depends on: Router, Api, PlaybackEngine. This is the first full user flow.

8. SearchScreen (tab + source chips)
   └─ depends on: same as HomeScreen. Validates playbackCtx switching.

9. LibraryScreen (likes + stats)
   └─ depends on: same. Validates likedIds Set and live heart sync.

10. ProfileScreen (taste bars + reset)
    └─ depends on: same. Lowest-risk, can slip.

11. QueuePanel (view + drag-to-reorder)
    └─ depends on: stable PlaybackEngine queue semantics. Build LAST because
       reordering is the most subtle interaction and benefits from having all
       entry points that populate the queue already working.

12. Gestures (swipe prev/next, swipe-down to close fullscreen)
    └─ depends on: FullscreenPlayer, PlaybackEngine. Additive layer.

13. Polish: haptics, animations, skeleton loaders, error states, empty states.
```

**Critical ordering insight:** PlaybackEngine must be console-testable before any UI consumes it. The current file's tight coupling between playback and DOM is the single biggest refactor hazard — decouple first, skin second.

**Do not start with redesigning Home.** Home is what everyone wants to redesign first because it's visible, but it depends on every architectural decision downstream. Start with the shell + Store + PlaybackEngine. Home falls out naturally once those are right.

---

## 8. Escape hatches for future milestones

The constraint: make future work (VK/Yandex sources, AI tagging, per-user library with genre folders) additive, not rewrite-inducing. Four concrete hatches:

### 8.1 Source registry (for VK/Yandex)

Define sources as data, not hardcoded branches:

```js
const Sources = {
  youtube:    { id:'youtube',    name:'YouTube',    icon:'▶️', enabled: true },
  soundcloud: { id:'soundcloud', name:'SoundCloud', icon:'☁️', enabled: true },
  jamendo:    { id:'jamendo',    name:'Jamendo',    icon:'🎶', enabled: true },
  library:    { id:'library',    name:'Библиотека', icon:'🎵', enabled: true },
  vk:         { id:'vk',         name:'VK Music',   icon:'🅥',  enabled: false },  // milestone+1
  yandex:     { id:'yandex',     name:'Я.Музыка',   icon:'🎧', enabled: false },
};
```

`SourceChips` iterates `Object.values(Sources).filter(s => s.enabled)`. Adding VK later = flip one boolean + backend work. Zero UI refactor.

### 8.2 Track shape contract

Freeze the track shape now, even for fields that are only used later:

```js
// Track = {
//   id: string, source: string, title: string, artist: string,
//   thumb?: string, stream_url?: string, duration?: number,
//   tags?: string[],       // AI-tagging future
//   genre_path?: string,   // per-user library folders future
//   user_meta?: object,    // per-user overrides future
// }
```

Document it in a comment block above `TrackCard`. Every component reads through `track.title` etc. — never destructures unknown fields. When AI tagging lands, `tags` appears and `TrackCard` can start rendering badges without any other component caring.

### 8.3 Playback context is data, not an enum branch

Current file has `if (ctx === 'wave') ... else if (ctx === 'genre') ...`. Replace with a context object:

```js
const ctx = {
  id: 'wave',
  canAutoExtend: true,
  loadMore: () => Api.wave(state.waveMood),
  onTrackChange: (track) => Feedback.send({ action:'play', track_id: track.id }),
};
PlaybackEngine.playContext(ctx, tracks, 0);
```

`PlaybackEngine` calls `ctx.loadMore()` when near end. Adding "personal library genre folder" later = new context object with `loadMore: () => Api.libraryGenre(folder)`. Zero PlaybackEngine changes.

### 8.4 Feature flags

A single `Config.features = { vk: false, yandex: false, aiTags: false, personalLibrary: false }` read at render time. Makes it safe to land half-built features behind a flag — critical when the constraint is "one file, git push to deploy" and rollback means `git revert`.

### What NOT to build now

- Generic plugin system for sources (YAGNI — you'll have 5 sources max, ever)
- Abstract StreamProvider interface (the backend owns source semantics; frontend just gets a URL)
- State machine library (xstate etc.) — Router's ~50-line hand-rolled machine is enough
- Web Components (`customElements`) — adds complexity, gives nothing at this scale
- Virtual DOM / framework — the whole point of this file is not having one
- Service Worker / offline — explicitly out of scope, and adding it later is additive

---

## Data flow diagrams

### Playback state flow

```
  User taps track in HomeScreen
            ↓
  HomeScreen calls PlaybackEngine.playContext(waveCtx, tracks, i)
            ↓
  PlaybackEngine:
    - Store.set({ queue, queueIndex, currentTrack, playbackCtx, isPlaying:true })
    - activeAudio.src = ...; activeAudio.play()
    - navigator.mediaSession.metadata = ...
    - preloadNext()
            ↓
  Store notifies subscribers:
    ├─ MiniPlayer.render()       (shows new title/art)
    ├─ FullscreenPlayer.render() (if mounted visible)
    └─ HomeScreen.render()       (highlights playing row via data-track-id)
            ↓
  MiniPlayer's rAF loop reads PlaybackEngine.getPosition() for progress bar
            ↓
  <audio> 'ended' fires → PlaybackEngine.next()
            ↓
  Feedback.send({ action:'finish', ... }) — queued, flushed async
            ↓
  next() resolves → playTrack() → Store.set(...) → subscribers re-render
```

### Event ownership table

| Event source | Handler | Writes to Store |
|---|---|---|
| Track tap in any screen | `PlaybackEngine.playContext()` | `queue, queueIndex, currentTrack, isPlaying, playbackCtx` |
| `<audio>` 'ended' | `PlaybackEngine.next()` | `queueIndex, currentTrack, history` |
| `<audio>` 'timeupdate' | (nothing — read-through) | — |
| MediaSession play/pause | `PlaybackEngine.toggle()` | `isPlaying` |
| MediaSession next/prev | `PlaybackEngine.next/prev()` | `queueIndex, currentTrack, history` |
| Like button | `Library.like(track)` | `likedIds` |
| Mood pick | `HomeScreen.selectMood()` | `waveMood, queue` (via reload) |
| BottomNav tap | `Router.go(screen)` | `screen` |
| MiniPlayer tap | direct | `fullscreenOpen` |
| FullscreenPlayer close | direct | `fullscreenOpen` |
| Queue button tap | direct | `queuePanelOpen` |
| Telegram BackButton | `Router.back()` | varies (see §3) |

One row per write. If two rows write the same key, there's a bug.

---

## Anti-patterns to avoid (all present in current file)

1. **`window.searchTabQueue = ...`** — globals as ad-hoc state. Replace with Store + playbackCtx.
2. **`onclick="playFromContext(${i}, '${ctx}')"`** — inline handlers with JSON-escaped payloads. Use event delegation + `data-*` attributes.
3. **`document.getElementById('mp-title').innerText = t.title`** — imperative DOM writes from async handlers. UI should be a function of Store; handlers mutate Store.
4. **Both `<audio>` elements written from many functions** — swap buffers, play, pause, set src, seek all scattered. Centralize in PlaybackEngine.
5. **`setupMediaSession` called inside `playTrack`** — handlers reattached every track. Set up once in PlaybackEngine init; handlers reference live Engine state.
6. **Silent `.catch(() => {})`** on feedback. Use the queued Feedback module so failures retry instead of vanish.
7. **Scroll listener that checks `tab-wave` active class** — UI condition gating data load. Put auto-extend in PlaybackEngine (queue-index driven) and move scroll-based pagination into HomeScreen's own listener scoped to its root.
8. **`confirm('Сбросить вкусовой профиль?')`** — `confirm` is blocked in Telegram WebView on some clients. Use `tg.showConfirm()`.

---

## Relevant file

- `/root/music-bot-frontend/index.html` — current single-file implementation (~1023 lines) that this architecture replaces. Everything in lines 499–1020 needs to be reorganized into the module layout in §1.

## Confidence assessment

| Area | Confidence | Reason |
|---|---|---|
| Module pattern (IIFE + Store) | HIGH | Standard vanilla-JS SPA pattern |
| No signals/no framework at 2-3k LOC | HIGH | Supported by existing file working at 1k LOC with worse structure |
| Router + Telegram BackButton | HIGH | Matches Telegram WebApp SDK docs |
| PlaybackEngine as sole audio owner | HIGH | Standard separation-of-concerns; current file's bugs come from violating this |
| Queue semantics (mainline + userQueue + history) | HIGH | Matches Spotify/Yandex Music user-visible behavior |
| Feedback queue with localStorage | MEDIUM | Pattern is correct; localStorage availability in Telegram WebView is reliable on modern clients but worth sanity-checking on Android Telegram 10.x |
| Build order | HIGH | Dependency-driven — each step has a concrete definition of "done" that unblocks the next |
| Future-proofing via source registry + context objects | HIGH | YAGNI-balanced |

**Overall: HIGH.**
