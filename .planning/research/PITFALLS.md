# Pitfalls Research

**Domain:** Telegram Mini App audio player (single `index.html`, mobile WebView, GitHub Pages)
**Researched:** 2026-04-15
**Confidence:** HIGH for Telegram/WebView/MediaSession quirks (verified against current docs + existing code), MEDIUM for iOS 17/18 audio behavior (fast-moving), HIGH for the code-specific issues visible in the current `index.html`.

This document is written against the actual state of `/root/music-bot-frontend/index.html` (1023 lines, dual `<audio>` A/B already present, MediaSession wired but incomplete, no gesture code, no `BackButton`, no error recovery on stream URLs). Pitfalls below are ordered roughly by blast radius.

---

## Critical Pitfalls

### Pitfall 1: iOS WKWebView autoplay lock on first track after cold-start

**What goes wrong:**
On iOS Telegram (WKWebView), `audio.play()` rejects with `NotAllowedError` unless the call stack originates from a user gesture. The current code calls `playTrack()` from many places (wave auto-load, queue navigation, MediaSession handlers, `onended`). The very first `play()` after app launch **must** be synchronous inside a `click`/`touchend`. The existing flow does `await fetch('/wave/...')` and then calls `playTrack()` — by then the gesture token is gone and iOS silently refuses.

**Why it happens:**
Developers test on Android Chrome (permissive), the first track plays fine, then iOS users see a dead play button. iOS also invalidates the gesture token across `await` boundaries — any `await` between the tap and `.play()` kills it.

**How to avoid:**
- On the very first tap that should produce sound, call `audioA.play()` synchronously with a silent dummy source (`data:audio/mp3;base64,...` or a 1-frame silent mp3 hosted on the same origin) **before** any fetch. Then replace `.src` and call `.play()` again — iOS carries the unlock forward.
- Keep a global `audioUnlocked` flag set on first successful `play()`. Until it is true, never initiate playback from non-gesture paths (wave auto-load that completes while the user is idle).
- Never put `await` between the tap handler and the first `.play()` of the session.

**Warning signs:**
Mini-player appears, title/artist update, but `activeAudio.paused === true` forever. Console shows `NotAllowedError: The request is not allowed by the user agent`. Works on Android, dead on iOS.

**Phase to address:** Audio core / player refactor phase. **Severity: BLOCKER** — without this, iOS users cannot play anything.

---

### Pitfall 2: Event handler leaks via `audio.onended = ...` on dual-buffer swap

**What goes wrong:**
Current code does `activeAudio.onended = onTrackEnded` on every `playTrack()`. Because buffers A and B swap, the **previous** `activeAudio` (now `nextAudio`) still has its `onended` from the track that just ended. When `preloadNext()` later assigns `nextAudio.src = url`, an earlier queued load/seek can fire a stale `ended` on the wrong buffer and trigger a double-skip, advancing the queue by 2.

**Why it happens:**
Property-assignment handlers (`.onended = fn`) don't stack, but they also don't get cleared on buffer swap. Mixing A/B with preload and also with `onloadedmetadata` (closure-captured `activeAudio`) creates race windows where the "wrong" audio element is the closure subject by the time the callback fires.

**How to avoid:**
- Use `addEventListener('ended', handler)` + explicit `removeEventListener` with a named handler stored on the element (`el._ended = handler`).
- Pass the specific element as a parameter into callbacks, do not read `activeAudio` from the outer scope inside `onloadedmetadata`/`ontimeupdate` — it may have swapped.
- On every `playTrack()`, wipe handlers on **both** buffers first, then attach fresh ones only to the new active one.
- Unit test: simulate 10 rapid skips, assert `currentIndex` advanced by exactly 10, not 11–20.

**Warning signs:**
Queue silently skips a track. Progress bar jumps. Feedback `finish` event fires twice for the same track_id. `sendFeedback('finish')` races `sendFeedback('skip')`.

**Phase to address:** Audio core refactor. **Severity: MAJOR.**

---

### Pitfall 3: MediaSession `positionState` never updated → lock-screen scrubber is a liar

**What goes wrong:**
Current `setupMediaSession()` sets metadata + play/pause/prev/next handlers but **never** calls `navigator.mediaSession.setPositionState({ duration, position, playbackRate })`. Result: the lock-screen / notification shade progress bar on Android stays at 0 or jumps randomly, scrub-back/scrub-forward actions do nothing useful, and the elapsed time is stale. This is the single most obvious "not-Spotify" tell on Android.

**Why it happens:**
`setPositionState` is documented but optional — everything "works" without it, so devs forget. Also, calling it on every `timeupdate` (4×/sec) is wasteful; devs try and then bail.

**How to avoid:**
- Call `setPositionState` once on `loadedmetadata` (with final duration) and again on every `play`/`pause`/`seek` — **not** on `timeupdate`. Android extrapolates between updates from `playbackRate`.
- Register `seekto`, `seekbackward`, `seekforward`, `stop` action handlers. `seekto` must read `details.fastSeek` and `details.seekTime`.
- Wrap in `try/catch` — `setPositionState` throws `InvalidStateError` if `position > duration` (happens on first paint when duration is NaN).

**Warning signs:**
Android lock screen shows progress stuck at 0. Scrubbing from notification does nothing. Auto media controls on Android Auto / Bluetooth show wrong elapsed time.

**Phase to address:** MediaSession polish phase. **Severity: MAJOR** — directly visible on the lock screen, kills credibility.

---

### Pitfall 4: Stream URL 403 / expiry mid-playback with no recovery

**What goes wrong:**
Stream URLs for YouTube/SoundCloud sources often include signed query params that expire (commonly 6h for YouTube `googlevideo.com`). If the user pauses a track overnight and hits play next morning, the browser gets 403 on the Range request and `audio` silently stops (`error` event fires, current code does not listen for it). The mini-player shows "playing" but nothing happens.

**Why it happens:**
Current code has no `audio.addEventListener('error', ...)` and no `stalled`/`suspend` handling. The URL is built on the frontend from `t.stream_url` which was returned by `/search` / `/wave` and is already stale by the time of playback.

**How to avoid:**
- Listen for `error` (code `MEDIA_ERR_NETWORK` / `MEDIA_ERR_SRC_NOT_SUPPORTED`) and for `stalled` > 5s.
- On error, re-fetch the stream URL: ask the backend for a fresh redirect via `/stream/{id}?...` — the backend already proxies, so the frontend URL shape stays the same but the backend should follow redirects to a fresh signed URL. Verify with backend owner that the `/stream/*` endpoint is **not** returning a 302 to an expiring signed URL — if it is, that is a backend issue to document in PITFALLS against the frozen backend constraint.
- On 403 specifically, retry once with `?_cb=${Date.now()}` to bust any intermediate cache.
- Show a toast "Источник истёк, обновляю…" rather than dying silently.

**Warning signs:**
"Track plays at launch, dies after phone sleep." Users report "the player froze." Network tab shows 403 on `/stream/*` with no UI feedback.

**Phase to address:** Audio core / error-handling phase. **Severity: MAJOR.**

---

### Pitfall 5: CORS + Range requests on audio break seeking

**What goes wrong:**
If `api.vdsmusic.ru:8443` does not send `Accept-Ranges: bytes` **and** does not allow `Range` in `Access-Control-Allow-Headers`, iOS Safari's media element may either (a) refuse to seek past 0, (b) redownload the entire file on every seek, or (c) fail to fire `loadedmetadata` until the full body is loaded so `duration` is `Infinity`. Also, `crossOrigin="anonymous"` is **not** set on the `<audio>` elements in the current code — that's fine for playback but blocks any future WebAudio analyzer / canvas visualizer from reading samples.

**Why it happens:**
GitHub Pages origin `*.github.io` ≠ API origin `api.vdsmusic.ru:8443`. All audio loads are cross-origin. Without proper Range + CORS, WebKit silently degrades.

**How to avoid:**
- **Verify with a curl test** early (Phase 0 smoke test): `curl -H "Origin: https://vdsmusic.github.io" -H "Range: bytes=0-100" -I https://api.vdsmusic.ru:8443/stream/...` — expect `206 Partial Content`, `Accept-Ranges: bytes`, `Access-Control-Allow-Origin: *` (or echo), `Access-Control-Expose-Headers: Content-Range, Accept-Ranges, Content-Length`.
- If any are missing, this is the **one** backend change that is unavoidable — document as "backend must expose Range headers" even though the milestone says backend is frozen. Without it, seeking is broken and "Spotify feel" is impossible.
- Do **not** set `crossOrigin="anonymous"` unless you need WebAudio — it adds a preflight and can break playback if ACAO is missing.

**Warning signs:**
`audioElement.duration === Infinity` or `NaN` for the entire track. Seek bar snaps back to 0. Scrubbing triggers a full re-download (watch Network tab).

**Phase to address:** Phase 0 smoke test (before any UI work). **Severity: BLOCKER if Range is broken.**

---

### Pitfall 6: Telegram swipe-down-to-close hijacks the vertical swipe on the fullscreen player

**What goes wrong:**
Telegram WebView (iOS especially) uses a vertical swipe-down on the top of the mini-app to close it. The spec requires a "swipe down to minimize fullscreen player" gesture. These fight. The user tries to minimize the player, Telegram closes the whole app instead.

**Why it happens:**
By default, Telegram treats any vertical drag starting near the top as a close gesture. Dev doesn't know about `tg.disableVerticalSwipes()` (added in Bot API 7.7, April 2024).

**How to avoid:**
- Call `tg.disableVerticalSwipes()` on init. Re-enable with `tg.enableVerticalSwipes()` only when the fullscreen player is closed and the user is on a scroll view that should allow swipe-to-close.
- Gate on `tg.isVersionAtLeast('7.7')` — older Telegram clients will ignore the call, and there you must accept the conflict and warn in docs.
- Put the drag handle (thumb) visually at the top of the fullscreen player but register the touch listener on the whole sheet.

**Warning signs:**
Testers report "the app closes when I try to minimize the player." Only reproducible in real Telegram, never in a browser.

**Phase to address:** Fullscreen player + gestures phase. **Severity: MAJOR.**

---

### Pitfall 7: `100vh` / viewport jumps when Telegram keyboard opens

**What goes wrong:**
Using `height: 100vh` on the app shell or on the mini-player causes the layout to jump when the Telegram search-box keyboard opens. The mini-player ends up hidden behind the keyboard, or worse, the fullscreen player's controls scroll out of view. On iOS, `100vh` includes the URL bar area and does not shrink when the keyboard opens.

**Why it happens:**
`100vh` is broken by design on mobile. Telegram provides `tg.viewportHeight` and `tg.viewportStableHeight` plus a `viewportChanged` event. Devs ignore them and use CSS.

**How to avoid:**
- Use `tg.viewportStableHeight` to size the app shell. On `viewportChanged` with `isStateStable: true`, update a CSS variable `--app-h`.
- Use `100dvh` (dynamic viewport) as fallback for non-Telegram browsers.
- Pin the mini-player with `position: fixed; bottom: var(--safe-bottom, 0); transform: translateZ(0)` and set `--safe-bottom` from `env(safe-area-inset-bottom)`.
- Never use `100vh` anywhere in the stylesheet — grep-forbid it in a pre-commit check.

**Warning signs:**
Mini-player hides under the keyboard. Fullscreen player controls disappear when a text input is focused. Blank strip at the bottom on iPhone with a home indicator.

**Phase to address:** Layout / shell phase. **Severity: MAJOR.**

---

### Pitfall 8: `tg.expand()` race condition — measuring viewport before expand completes

**What goes wrong:**
Current code calls `tg.expand()` on line 502 and immediately starts rendering. On Android Telegram, `expand()` is async — the viewport grows over ~150ms. If layout is measured (e.g., mini-player absolute position) during that window, it lands in the wrong place and jumps after expand finishes.

**Why it happens:**
`tg.expand()` returns `void` synchronously but the animation is asynchronous. There is no promise. Devs assume it is instant.

**How to avoid:**
- Call `tg.expand()` once at init, then render from inside the first `viewportChanged` event (with `isStateStable: true`). Only then measure and position absolute elements.
- Use CSS for layout wherever possible — pure `position: fixed; bottom: 0; left: 0; right: 0` for the mini-player does not need JS measurement and is immune to the race.
- Avoid JS-based layout calc for anything that must survive viewport changes.

**Warning signs:**
Mini-player flashes at the wrong Y position on app launch, then snaps into place. A 200ms flicker on cold start. Reported on Android but not iOS (iOS expand is near-instant).

**Phase to address:** Shell / init phase. **Severity: MINOR but visible.**

---

### Pitfall 9: `initDataUnsafe.user` empty in certain launch contexts

**What goes wrong:**
Current code: `const USER_ID = (tg.initDataUnsafe && tg.initDataUnsafe.user && tg.initDataUnsafe.user.id) || 0;`. When launched from an inline button without a `start_param`, from Telegram Desktop (some versions), or from a shared link, `initDataUnsafe.user` can be **undefined**. `USER_ID = 0` then gets sent to every `/wave/*` and `/library/*` call — the backend serves a global profile or errors. All users collide on `user_id=0` or see each other's likes.

**Why it happens:**
Silent fallback to 0 instead of failing loud. Also, `initDataUnsafe` is empty when the app is opened outside a proper Telegram context (testers opening the Pages URL directly in a browser for debugging).

**How to avoid:**
- If `tg.initDataUnsafe.user?.id` is missing, show a full-screen error "Открой приложение через Telegram" and refuse to send anything to the backend.
- Log out-of-context launches to the console with the full `initData` string for debugging.
- Parse `initData` (the raw string) as a fallback — sometimes `initDataUnsafe` is empty but `initData` is populated (rare but seen on Telegram Desktop ≤ 4.x).
- Do not default to 0 ever.

**Warning signs:**
Single user's likes appear in everyone's library. Backend logs show `user_id=0` for many distinct sessions. "I liked track X and now my friend sees it too."

**Phase to address:** Auth / init phase. **Severity: MAJOR** (cross-user data leak).

---

### Pitfall 10: BackButton state desync with internal router

**What goes wrong:**
The app has multiple "screens" (wave, search, library, genres) and a fullscreen player. `tg.BackButton` is a single system-level button that either shows or hides. If the dev forgets to call `BackButton.hide()` when reaching the root screen or to swap its handler when navigating, tapping the back button either closes the whole mini-app (because no handler) or triggers a stale handler pointing at a screen that no longer exists.

**Why it happens:**
BackButton is global singleton state, but the internal router is usually per-screen. Developers forget to treat it as a reducer side-effect.

**How to avoid:**
- Centralize navigation in one `navigate(screen)` function that, as a side-effect, calls `tg.BackButton.show()` / `.hide()` and **replaces** the handler via `tg.BackButton.onClick(handler)` + stores the previous one to unbind.
- On root screen: `tg.BackButton.hide()`.
- On fullscreen player open: treat it as a navigation step and bind a back handler that closes it.
- Before binding a new handler, always `offClick(previousHandler)` or Telegram stacks them and both fire.

**Warning signs:**
Back button closes the entire app from a sub-screen. Back button does nothing. Back button fires two navigations at once.

**Phase to address:** Navigation / router phase. **Severity: MAJOR.**

---

### Pitfall 11: Swipe (prev/next) conflicts with vertical scroll on track lists

**What goes wrong:**
A horizontal swipe gesture on the mini-player for prev/next conflicts with vertical scrolling of the list above. If the touch listener calls `preventDefault()` to suppress vertical scroll during a horizontal swipe, it must be registered with `{ passive: false }`. But passing `passive: false` to `touchmove` on iOS penalizes scroll FPS across the whole page if the detector is slow. Worse, if the listener is on `document`, the fullscreen player's drag-to-dismiss and the list's vertical scroll fight each other.

**Why it happens:**
Devs copy-paste a swipe library that uses `passive: true` (for perf) — it cannot `preventDefault`, so vertical scroll always wins and horizontal swipe never triggers. Or they use `passive: false` everywhere and iOS scroll becomes sluggish.

**How to avoid:**
- Attach the swipe listener **only** to the mini-player element, not `document`.
- Use `{ passive: false }` only on that element's `touchmove`, and only after an axis lock: on first `touchmove`, measure dx/dy; if `|dx| > |dy| * 1.5` lock horizontal (prevent default), else lock vertical (do nothing).
- Once axis is locked for the gesture, do not switch.
- Threshold: 60px horizontal + 400ms max duration for swipe to count.
- Test on iOS 17/18 specifically — passive listener behavior changed subtly in iOS 16.4.

**Warning signs:**
Horizontal swipes cause the page to scroll vertically instead. Or: vertical scroll feels "stuck" / janky on iOS. Or: the fullscreen player closes when the user tries to scroll the queue inside it.

**Phase to address:** Gestures phase. **Severity: MAJOR.**

---

### Pitfall 12: Bot message arrival interrupts audio session (iOS)

**What goes wrong:**
On iOS, when the user's Telegram bot sends a message while music is playing, the system "notification" can briefly duck or pause audio. Telegram on iOS does not properly restore it. The track stays "playing" in the UI but the audio element is paused, and MediaSession play button on the lock screen shows the wrong state.

**Why it happens:**
iOS audio session category handling inside WKWebView is opaque. WebView does not get `audiointerruption` events the way native apps do. The `pause` event fires but no `play` fires when interruption ends.

**How to avoid:**
- Listen for `pause` events that are not user-initiated: set a `userPaused` flag in the togglePlay handler. On `pause` events where `userPaused === false`, attempt `activeAudio.play()` after a 500ms delay.
- Also listen for `visibilitychange` → if visible and not user-paused and the audio is paused, resume.
- Accept that some interruptions cannot be recovered without a user tap — show a subtle "tap to resume" hint if auto-resume fails twice.

**Warning signs:**
"Music stops when I get a message." UI shows playing but no sound. MediaSession button on lock screen is out of sync.

**Phase to address:** Audio core / error-handling phase. **Severity: MAJOR on iOS.**

---

### Pitfall 13: MediaSession artwork not updating or showing stale cover

**What goes wrong:**
Current code sets `artwork: [{ src: t.thumb, sizes: '512x512', type: 'image/jpeg' }]`. Problems: (a) the `sizes` is a lie if the thumb is 64×64 — Android will upscale and show a blurry cover; (b) Android Auto / lock screen aggressively caches artwork by URL — if two tracks share a thumb URL, the new metadata update is a no-op and stale art persists; (c) cross-origin artwork without `Access-Control-Allow-Origin` will be rejected silently by Chrome and the default Chrome icon appears in the notification (this is the "Android shows Chrome instead of the app" symptom).

**Why it happens:**
Copy-paste boilerplate with a single `sizes: '512x512'` entry. Artwork URLs are from YouTube/SoundCloud CDNs that may or may not have CORS. No fallback.

**How to avoid:**
- Provide multiple sizes in the `artwork` array: `96x96`, `256x256`, `512x512` — list the same URL or different sizes if available. Android picks the best.
- Always include `type: 'image/jpeg'` or `'image/png'`.
- For source artwork that is not CORS-friendly, proxy through the backend: `GET /art?url=...` — but the backend is frozen this milestone. Fallback: use a generic local-bundled cover (data URI) when the remote one fails to load (`Image.onerror`).
- Call `setupMediaSession` **after** the `thumb` has actually been preloaded via `new Image(); img.src = t.thumb; img.onload = ...` — setting metadata with an unresolved URL triggers the Android "generic Chrome" fallback.
- Verify on a real Android device, not Chrome DevTools device emulator.

**Warning signs:**
Android notification shows a generic Chrome icon or nothing. Lock-screen cover is the **previous** track's cover. Cover is blurry at high resolution.

**Phase to address:** MediaSession polish phase. **Severity: MAJOR** — visible on lock screen, kills credibility.

---

### Pitfall 14: Preloading the next track without releasing the previous one (memory blowup)

**What goes wrong:**
Dual buffer A/B is correct but `preloadNext()` sets `nextAudio.src = url` without stopping any in-flight load from the earlier "next" that never became current. Buffered data accumulates. On a long "Моя волна" session (50+ tracks), Safari can hit the per-domain media buffer cap and start failing new loads with `MEDIA_ERR_DECODE`.

**Why it happens:**
`audio.src = newUrl` does not cancel an ongoing fetch of the old URL immediately — the browser releases it lazily. Without `audio.pause(); audio.removeAttribute('src'); audio.load()` the previous buffer can linger.

**How to avoid:**
- Before setting a new `src` on the next buffer, call `el.pause(); el.removeAttribute('src'); el.load();` then set the new `src`.
- Do not preload tracks more than one ahead — resist the temptation to triple-buffer.
- Only preload when `currentQueue[currentIndex + 1]` exists AND the current track has fired `canplaythrough` (i.e., current is stable) — otherwise iOS will throttle both loads.

**Warning signs:**
After ~30 tracks, new tracks refuse to load (`MEDIA_ERR_DECODE`). Memory usage grows unbounded in Safari Web Inspector. Device heat on iOS.

**Phase to address:** Audio core / preload refactor. **Severity: MAJOR on long sessions.**

---

### Pitfall 15: Skipping server-side `initData` HMAC validation — blast radius

**What goes wrong:**
The milestone explicitly defers HMAC validation of `initData`. This means the backend trusts whatever `user_id` the frontend sends. An attacker who reads one network request can:
1. Send arbitrary `user_id` to `/library/like`, `/library/dislike`, `/wave/feedback`, `/taste/reset` — **poison another user's taste profile** and wipe their library.
2. Read another user's likes via `/library/likes?user_id=...` and `/taste/{user_id}` — **PII leak** (track history is sensitive).
3. Impersonate a user for `/wave/*` — serve them their friend's wave.

The attack surface is: anyone who knows the API URL (it is in the frontend, so public) + a valid Telegram user_id (numeric, enumerable or guessable via group member lists).

**Why it happens:**
Skipping HMAC feels like "auth later" — but there is no auth layer at all right now, so it is effectively "public write access indexed by user_id."

**How to avoid (this milestone):**
- Document the risk explicitly in `PROJECT.md` → Accepted Risks. Audience = friend circle. Acceptable **only** while API URL is unadvertised. The moment the bot is posted publicly, HMAC becomes a blocker.
- Add rate limiting at the backend edge (nginx/Caddy) per IP — not full mitigation but slows enumeration.
- Do **not** expose destructive endpoints (`/taste/reset`, bulk delete) in the frontend without a confirmation modal — an attacker forging requests still works, but prevents accidental self-inflicted damage via shared links.
- Add a visible "Milestone N+1: server-side initData validation" item at the top of the future roadmap.
- Consider obfuscation as stop-gap: include the `auth_date` from `initDataUnsafe` in every request and have the backend reject anything older than 24h. Not real security but raises the bar from trivial.

**Warning signs:**
"Why did my likes disappear?" reports. Taste profile behaving erratically. Backend logs showing `user_id` values that don't match any real Telegram user (test-ids like 1, 123, 99999).

**Phase to address:** Document in Phase 0 (scope acknowledgement). Implement HMAC validation in the next milestone. **Severity: MAJOR risk, accepted trade-off for this milestone, must be explicit.**

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|---|---|---|---|
| Property assign `el.onended = fn` instead of `addEventListener` | Less code, less bookkeeping | Handler leaks across buffer swap, double-fires, impossible to debug | Never in this codebase — buffer swap makes it unsafe |
| `USER_ID \|\| 0` fallback | App doesn't crash in browser testing | Cross-user data leak when Telegram context is missing | Never — fail loud instead |
| Single `artwork` entry in MediaSession | Boilerplate | Blurry lock-screen cover on Android, "Chrome icon" bug | Never — multi-size array is trivial |
| Inline `<style>` with 1000+ lines | Zero build, single file | CSS specificity wars, dead selectors, no dedup | Acceptable **only** because the milestone mandates single-file; mitigate with strict naming (BEM-ish) and a section index |
| Polling `/wave/feedback` instead of batching | Simple | Burns cellular data, backend load | Acceptable at current scale (<100 users) — revisit if scaling |
| Inline SVG sprite duplicated per icon use | Simple DOM | Page weight, parse time | Never — define `<symbol>` once in a hidden `<svg>` and `<use>` elsewhere |
| No error boundary around `fetch` failures | Less code | Silent failures, user sees spinner forever | Never — always toast the error |
| Skipping `initData` HMAC | Speed to ship this milestone | Data integrity + PII risk (see Pitfall 15) | Only while audience is private |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|---|---|---|
| `Telegram.WebApp` | Calling `tg.expand()` and measuring layout immediately | Wait for `viewportChanged` with `isStateStable: true`; prefer pure-CSS layout |
| `Telegram.WebApp.BackButton` | `onClick(fn)` stacks handlers if not unbound | Always `offClick(previousFn)` before `onClick(newFn)` |
| `Telegram.WebApp.HapticFeedback` | Calling on unsupported version throws | Guard with `tg.isVersionAtLeast('6.1')` before every haptic |
| `Telegram.WebApp.themeParams` | Reading once on load and caching | Listen for `themeChanged` event; theme can change mid-session if user switches Telegram theme |
| MediaSession `metadata` | Setting `new MediaMetadata({...})` without awaiting artwork load | Preload `new Image()` first, then set metadata — avoids "Chrome icon" on Android |
| MediaSession `setActionHandler('seekto')` | Ignoring `details.fastSeek` | Honor `details.fastSeek` to avoid triggering a full reload on scrub |
| `<audio>` + CORS | Assuming `crossOrigin="anonymous"` is free | It adds a CORS preflight; omit unless you need WebAudio analysis |
| YouTube/SoundCloud stream URLs | Caching the URL on the client | URLs are signed + expire; always proxy through `/stream/*` and handle 403 |
| GitHub Pages | Not cache-busting `index.html` | Set a `<meta http-equiv="Cache-Control" content="no-cache">` **and** append `?v=${commit_sha}` to CDN script tags |
| Telegram in-app browser cache | Telegram caches the Mini App aggressively | Version query string on the Pages URL in the bot's WebAppInfo: `https://.../?v=20260415` updated per deploy |
| `fetch` to `api.vdsmusic.ru:8443` | Mixed content with audio served from HTTP | Verify **all** audio URLs are HTTPS; browser blocks HTTP audio on HTTPS pages silently without error in some Telegram versions |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|---|---|---|---|
| Re-rendering full track list on every state change | Scroll jank, dropped frames during playback transitions | Render once, update only affected rows (`querySelector` + targeted class toggles). No virtual DOM possible without a framework, so be strict about mutation | >50 visible tracks |
| `timeupdate` handler doing DOM writes 4×/sec | Progress bar stutters, audio glitches during transitions | Write to a CSS variable (`--progress`) instead of `innerText`/`value`; use `requestAnimationFrame` to batch | Always — `timeupdate` fires ~4Hz but stacks with cover art loads |
| Large inline SVG sprite re-parsed on every `innerHTML` set | Slow initial paint, CLS | Define `<symbol>` in one hidden `<svg>`, reference via `<use xlink:href="#id">` | >20 icons |
| Synchronous layout reads during scroll (`offsetHeight` etc.) | Scroll FPS drops on iOS | Use `IntersectionObserver` and `ResizeObserver`; never mix reads and writes | iOS Telegram WebView always, Android at >100 list items |
| Cover art downloaded at full resolution on cellular | Bandwidth cost, slow list paint | Use thumb URL at list-appropriate size (96–128px), fullsize only in fullscreen player; `<img loading="lazy">` for off-screen | Every page load on cellular |
| Overeager `/wave/feedback` — firing on every skip within 500ms | Backend load, network congestion | Debounce feedback (300ms) but **not** on `finish` | Rapid-skip sessions |
| Autoextend wave polling on every `timeupdate` | Redundant requests | Current code guards via `currentIndex >= length - 3`, which is correct — keep this guard explicit | Already fine |
| CSS transitions on `width`/`height` (triggers layout) | Janky animations | Transition `transform` and `opacity` only; use `will-change: transform` on the fullscreen player sheet | Always — the 60fps line |
| Main-thread JSON.parse of large `/wave` responses during playback | Audio glitch at track boundary | Parse off the playback critical path — defer non-essential wave extension until after track start | Wave sessions with large payloads |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---|---|---|
| Trusting `initDataUnsafe.user.id` without HMAC (Pitfall 15) | Cross-user data poisoning, PII leak | Accepted risk this milestone; document + add HMAC next milestone |
| Sending `user_id` as query param on GET | Leaks to backend access logs, proxies, referer headers | Already done via query; acceptable but prefer POST body for destructive endpoints |
| Rendering backend-sourced `title`/`artist` via `innerHTML` | XSS if backend is compromised or source data contains HTML | Always use `textContent`/`innerText` — current code uses `innerText` in `playTrack`, good. Audit all other render paths |
| Using `eval` / `new Function` for dynamic logic | RCE | Never; no reason in this codebase |
| Trusting redirect target of `/stream/*` | Backend redirects to arbitrary URLs → SSRF-adjacent (frontend side is safer but still a data exfil vector via referer) | Send `Referrer-Policy: no-referrer` header meta |
| Logging `initData` to the console | Leaks user data if testers share screens | Gate debug logs behind `?debug=1` query param |
| CSP missing | Injected scripts via network MITM (unlikely given HTTPS) | Add a `<meta http-equiv="Content-Security-Policy">` — `connect-src 'self' https://api.vdsmusic.ru:8443; media-src https:; script-src 'self' 'unsafe-inline' https://telegram.org;` — note `unsafe-inline` required for single-file design |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---|---|---|
| 30fps animations (CSS transitions on `width`, JS-driven layout) | "Feels cheap," kills Spotify illusion | 60fps via `transform`+`opacity` only; measure with Chrome DevTools FPS meter on a real mid-range Android |
| Loading spinners instead of skeletons | Perceived as slow even if fast | Skeleton placeholders for track lists, cover art (shimmer CSS) |
| Touch targets <44×44 CSS px | Missed taps, especially on like/skip buttons | Minimum 44×44 hit area — pad with transparent `::before` if the visual is smaller |
| Double-tap zoom triggering on buttons | Accidental zoom on rapid taps | `touch-action: manipulation` on all interactive elements + `<meta name="viewport" ... user-scalable=no>` (current code has it, good) |
| No haptic on key actions (play, like, dislike, swipe-triggered skip) | Feels "unresponsive" vs native apps | `tg.HapticFeedback.impactOccurred('light')` on play/skip, `'medium'` on like, `'rigid'` on dislike — guarded by version check |
| Mini-player reflows when navigating between screens | Flicker, broken "persistent player" illusion | Mini-player lives in a separate stacking context (fixed position, outside the router) — never re-render on navigation |
| Fullscreen player has no drag handle visual | Users don't know they can swipe down | Explicit `—` grab handle at top, plus subtle "pull down" hint animation on first open |
| No haptic on swipe-to-skip confirmation | User can't tell if the swipe was registered | Trigger haptic on swipe threshold crossed, not on release |
| Tap on mini-player thumbnail does nothing | Expected behavior: opens fullscreen player | Whole mini-player row opens fullscreen; only explicit play button toggles playback |
| No "now playing" indicator in the source list | User forgets which track is playing after scrolling | Highlight the current row with an equalizer glyph + accent color |
| Search input focus causing layout jump on keyboard open | Search field hidden by keyboard | Scroll-into-view the focused input **after** `viewportChanged` fires, not immediately |
| Queue reorder via long-press without visual handle | Users don't discover it | Explicit drag handle icon on the right of each queue row |
| Fullscreen player cover art not filling on tablets / large phones | Looks small, unpolished | `aspect-ratio: 1; max-width: min(90vw, 90svh)` |
| Status bar / notch overlap | Content hidden under the notch | `env(safe-area-inset-top)` + Telegram `themeParams.header_bg_color` for the top band; set `tg.setHeaderColor` to match the app |

---

## "Looks Done But Isn't" Checklist

- [ ] **Audio playback:** Often missing error handlers (`error`, `stalled`) — verify by throttling network in DevTools and killing a stream URL mid-track.
- [ ] **MediaSession:** Often missing `setPositionState` and `seekto` handler — verify lock-screen scrubbing actually seeks on a real Android device.
- [ ] **MediaSession artwork:** Often missing multi-size array and CORS-ok URLs — verify on real Android lock screen that art is sharp and not a Chrome icon.
- [ ] **BackButton:** Often left showing on root screen or with stale handler — verify navigating back from each screen matches expectation, including from fullscreen player.
- [ ] **Telegram swipe-down:** Often forgotten `disableVerticalSwipes()` — verify fullscreen player can be minimized without closing the app.
- [ ] **Viewport:** Often uses `100vh` somewhere — grep for `100vh` and `vh` in the stylesheet, replace with `var(--app-h)` or `100dvh`.
- [ ] **iOS autoplay:** Often untested on iOS real device — verify first track plays on cold start with only one tap.
- [ ] **Preload next:** Often doesn't release previous buffer — verify memory usage stays flat over 30+ tracks in Safari Web Inspector.
- [ ] **CORS + Range:** Often assumed to work — verify `curl -I -H "Range: bytes=0-100" ...` returns 206 with proper headers, and that seek on iOS re-downloads only the requested range.
- [ ] **Haptics:** Often present on Android but absent on iOS WebView (HapticFeedback API support) — verify on real iOS; fall back to no-op.
- [ ] **initData fallback:** Often defaults `user_id` to 0 — verify the app refuses to start outside Telegram context.
- [ ] **Safe area insets:** Often missing on fullscreen player bottom — verify on iPhone with home indicator that controls are not under the bar.
- [ ] **Theme params:** Often only read once at init — verify that toggling Telegram theme mid-session updates the app colors (listen to `themeChanged`).
- [ ] **Queue persistence across tab sleep:** Often MediaSession works but the audio element is paused silently — verify playing music, locking the phone for 2 minutes, and checking that audio resumes correctly.
- [ ] **Deploy cache:** Often stale `index.html` served by GitHub Pages CDN — verify the bot's WebAppInfo URL includes a cache-busting query param that changes per deploy, and that `index.html` has `Cache-Control: no-cache` via a `<meta>` tag.
- [ ] **Service Worker residue:** Often a lingering SW from earlier experiments — verify `navigator.serviceWorker.getRegistrations()` is empty; unregister any found in a top-of-file cleanup script.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---|---|---|
| Handler leak double-skip (P2) | LOW | Refactor to `addEventListener`, store handler refs on element, unit test rapid skip |
| Missing `setPositionState` (P3) | LOW | Add 3 lines to `setupMediaSession`; verify on Android |
| Stream URL 403 (P4) | MEDIUM | Add error listener + retry with cache-bust; if backend returns expired signed URL, escalate to backend owner |
| CORS/Range broken (P5) | HIGH | Requires backend config change; negotiate exception to "backend frozen" rule |
| Telegram swipe conflict (P6) | LOW | One line: `tg.disableVerticalSwipes()` — but gated on version |
| `100vh` layout bug (P7) | MEDIUM | Global CSS refactor to use dvh / JS-set variable |
| `USER_ID=0` fallback (P9) | LOW | Fail-loud check at init; backend cleanup of polluted data is MEDIUM cost |
| BackButton desync (P10) | MEDIUM | Centralize in router function; audit every navigation path |
| Swipe vs scroll conflict (P11) | MEDIUM | Axis-lock algorithm; requires real-device testing |
| Audio session interruption (P12) | MEDIUM | Auto-resume on pause + visibilitychange; some edge cases unrecoverable |
| Stale artwork on lock screen (P13) | LOW | Multi-size array + preload via `new Image()` |
| Preload memory blowup (P14) | MEDIUM | Explicit buffer release; may require architectural rethink if Safari still complains |
| Skipped HMAC → data poisoning (P15) | HIGH (post-incident) | Next milestone: add HMAC validation; this milestone: document + rate-limit |
| Stale Telegram cache showing old version | LOW | Version query param on WebAppInfo URL per deploy |

---

## Pitfall-to-Phase Mapping

Roadmap phases below are **suggested** — actual phases defined in roadmap synthesis. This table tells the roadmap author which phase should own prevention of each pitfall.

| Pitfall | Prevention Phase | Verification |
|---|---|---|
| P1 iOS autoplay lock | Phase: Audio core refactor | Cold-start iOS real device, first tap plays |
| P2 Handler leaks on buffer swap | Phase: Audio core refactor | 10-rapid-skip test, `currentIndex` matches |
| P3 MediaSession `setPositionState` | Phase: MediaSession polish | Android lock-screen scrubber works |
| P4 Stream URL expiry | Phase: Audio core / error handling | Sleep phone 1h mid-track, resume works |
| P5 CORS + Range headers | Phase 0 smoke test (before any UI) | `curl` test returns 206 with correct headers |
| P6 Telegram swipe-down conflict | Phase: Fullscreen player / gestures | `disableVerticalSwipes()` called, real Telegram test |
| P7 `100vh` / viewport / keyboard | Phase: Layout / shell | No `100vh` in stylesheet; keyboard test |
| P8 `tg.expand()` race | Phase: Shell / init | No flicker on Android cold start |
| P9 Empty `initDataUnsafe.user` | Phase: Auth / init | Opening Pages URL directly shows error screen |
| P10 BackButton desync | Phase: Navigation / router | All nav paths tested with BackButton |
| P11 Swipe vs scroll | Phase: Gestures | iOS/Android axis-lock test |
| P12 Audio session interruption | Phase: Audio core / error handling | Incoming message mid-playback, auto-resume |
| P13 MediaSession artwork | Phase: MediaSession polish | Real Android lock-screen art is sharp |
| P14 Preload memory blowup | Phase: Audio core / preload | 30+ track session, Safari memory flat |
| P15 Skipped HMAC validation | Phase 0 (document) + future milestone | PROJECT.md Accepted Risks section populated |
| Inline SVG bloat | Phase: Layout / shell | `<symbol>`+`<use>` pattern adopted |
| 30fps animations | Phase: Polish / animation pass | DevTools FPS meter ≥ 58fps on scroll + transitions |
| Stale deploy cache | Phase: Deploy / release | Version query on WebAppInfo URL per release |

---

## Sources

- Telegram Bot API changelog (Bot API 7.7, disableVerticalSwipes) — https://core.telegram.org/bots/api-changelog (HIGH)
- Telegram Mini Apps docs — https://core.telegram.org/bots/webapps (HIGH)
- MDN MediaSession API, `setPositionState` — https://developer.mozilla.org/en-US/docs/Web/API/MediaSession/setPositionState (HIGH)
- WebKit audio autoplay policy — https://webkit.org/blog/6784/new-video-policies-for-ios/ (HIGH, older but still authoritative)
- HTTP Range requests + CORS interplay — https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests (HIGH)
- iOS Safari `100vh` / `dvh` behavior — https://developer.mozilla.org/en-US/docs/Web/CSS/length#viewport-percentage_lengths (HIGH)
- Current codebase: `/root/music-bot-frontend/index.html` lines 476–1008 (dual-buffer audio, MediaSession setup, playTrack/preloadNext) — HIGH confidence, direct source
- Community discussions on Telegram Mini App gotchas: t.me/WebAppsExamples, GitHub issues on telegram-web-app.js (MEDIUM — used as cross-check only)
- Chromium MediaSession Android notification behavior and artwork CORS — chromium.googlesource.com/chromium design docs (MEDIUM)

Confidence caveats:
- iOS 17/18-specific WKWebView audio interruption behavior (Pitfall 12) is not well-documented publicly; the recommended auto-resume strategy is based on field observation more than spec. Treat as MEDIUM.
- "Chrome icon instead of app in Android notification" (Pitfall 13) is consistently reported in Chromium bug tracker but root cause varies — multi-size artwork is the known fix but not a spec guarantee.

---
*Pitfalls research for: Telegram Mini App audio player (VDS Music)*
*Researched: 2026-04-15*
