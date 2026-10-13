---
name: native-feel-mobile
description: Audits and fixes a responsive web app wrapped in Capacitor (or Cordova/PWA) so it behaves like a native mobile app: back gesture/button, pull-to-refresh, swipe navigation, safe areas, keyboard, scrolling, tap feedback, offline and lifecycle. Use when the user mentions mobile app feel, native behavior, gestures, Android back button, pull to refresh, safe area/notch, overscroll, Capacitor, or "feels like a website" on mobile.
---

# Native-feel mobile (Capacitor / webview apps)

Goal: nothing a user does with thumb, system back, keyboard or app switcher should reveal "this is a website".

## Quick start

1. Detect stack: `capacitor.config.*`, `android/`, `ios/`, viewport meta, router type (hash/history/none), existing `App.addListener` calls.
2. Run the checklist below; for each item mark PASS / FAIL / N-A with file:line evidence.
3. Fix FAILs smallest-first. Snippets are in [REFERENCE.md](REFERENCE.md).
4. Verify on a mobile viewport (emulate 375x812, touch) AND on a device/emulator for items marked **[device]** — a browser cannot prove them.
5. Report: what passed, what was fixed, what still needs a device check.

## Checklist

**Navigation & gestures**
- [ ] Android hardware/gesture back: closes modal/sheet/menu first, then pops screen history, only exits app at root (`App.addListener('backButton')`). Never dead-ends or exits mid-flow. **[device]**
- [ ] iOS edge-swipe back works (real history entries, or `allowsBackForwardNavigationGestures`). **[device]**
- [ ] Every screen/modal that opens pushes a history entry (or is handled in the back handler); deep links resume correctly.
- [ ] Pull-to-refresh on list/feed screens only; browser default PTR disabled (`overscroll-behavior-y: contain`) and a custom one refreshes data, not the whole page. Never fires while scrolled, inside maps, or inside nested scrollers.
- [ ] Swipe gestures (dismiss sheet, swipe row) don't fight horizontal scroll or system back edge (keep 20px edge dead zone).
- [ ] Bottom tab/nav keeps per-tab scroll position and state; re-tap scrolls to top.

**Mobile browser / installed PWA (no Capacitor `backButton` event; back = `history` + `popstate`)**
- [ ] Each screen change, modal, sheet, lightbox and menu does `pushState` on open; `popstate` closes the top overlay first, then navigates. A screen change that only uses `replaceState` makes back leave the site/app.
- [ ] Closing an overlay with its own X/scrim calls `history.back()` (not just hides it), so history stays in sync and back doesn't close two things.
- [ ] Refresh/reload on any deep URL restores that screen (server fallback to the SPA entry; no 404). Direct-link entry has a sane "back" (go to parent screen, not exit).
- [ ] bfcache/`pageshow` (`event.persisted`) and `visibilitychange` re-sync stale state when returning via back or from another tab/app.
- [ ] Browser pull-to-refresh and iOS Safari edge-swipe don't destroy in-progress work: unsaved form/draft is persisted or guarded (`beforeunload` is unreliable on mobile; save on `visibilitychange: hidden`).
- [ ] Collapsing address bar doesn't cause layout jumps (`dvh`/`svh`, no `100vh` fixed bars).
- [ ] Standalone PWA: `display: standalone` + manifest icons/theme color; external links and `target=_blank` work (they silently fail in some standalone contexts); no browser-only UI assumed.
- [ ] Gesture conflicts: custom horizontal swipes keep a 20px edge dead zone (iOS back swipe, Android gesture nav); map/canvas areas use `touch-action` so they don't hijack page scroll or back.

**Browser edge cases that give the game away**
- [ ] iOS Safari: input focus doesn't zoom (>=16px), and the page doesn't stay zoomed after blur; `maximum-scale` is NOT used (accessibility), use font-size instead.
- [ ] Keyboard via `visualViewport`: fixed bars/sheets track `visualViewport.height/offsetTop` (iOS doesn't resize layout viewport); `interactive-widget=resizes-content` in viewport meta for Chrome Android; focused field scrolled into view with `scrollIntoView({block:'center'})` after the keyboard opens.
- [ ] Bottom bars don't jump when the URL bar collapses (use `position: fixed` + `dvh`, or `env(safe-area-inset-bottom)`); no `100vh` hero that hides the CTA under browser chrome.
- [ ] Scroll containers: lock body scroll while a modal/sheet is open (`overflow:hidden` + `overscroll-behavior: contain` on the sheet, restore scroll position on close); no scroll chaining from sheet to page.
- [ ] Scroll restoration: `history.scrollRestoration='manual'` plus saved per-screen scroll, so back returns to the same list position.
- [ ] Double-tap and pinch zoom disabled only on app chrome/maps via `touch-action`, never globally.
- [ ] Text/selection: UI chrome not selectable, content is; no blue selection flash on tap; `-webkit-text-size-adjust: 100%` so landscape doesn't inflate text.
- [ ] Inputs: correct `type`, `inputmode`, `autocomplete`, `enterkeyhint`, `autocapitalize`, `autocorrect`; Enter/Go submits or moves to next field; date/time use native pickers.
- [ ] Forms: `<input type=file accept=image/* capture>` for camera, native share via `navigator.share` with copy-link fallback, clipboard via `navigator.clipboard` with fallback.
- [ ] Links: `tel:`, `mailto:`, `geo:`/maps links open the native handler; no 300ms delay; no sticky `:hover` after tap (`@media (hover:hover)` for hover styles).
- [ ] Transitions: screen changes animate (slide for push/pop, fade for tabs) honoring `prefers-reduced-motion`; use View Transitions API where available; no white flash between screens (set `html` background and `<meta name="theme-color">` matching, including dark mode variant).
- [ ] Dark mode: `color-scheme` + `prefers-color-scheme` (or app setting) applied before first paint to avoid a white flash.
- [ ] Performance feel: first paint under ~1s on mid-range phone, no layout shift (reserve image sizes), lazy-load below fold, 60fps scroll (animate `transform`/`opacity` only), long lists virtualized or paginated.
- [ ] Wake lock (`navigator.wakeLock`) during active navigation/recording; re-acquire on `visibilitychange`.
- [ ] Service worker: offline shell, update prompt ("New version, reload"), no stale-cache trap after deploy.
- [ ] Haptics: `navigator.vibrate` on Android browsers only, subtle; feature-detected (iOS Safari lacks it).
- [ ] Permissions (location, camera, notifications) asked in context after a user gesture, with a pre-prompt explanation and a "denied" recovery path (how to enable in settings).
- [ ] Install/standalone: manifest (`display`, `start_url`, `scope`, maskable icons, `theme_color`), iOS `apple-mobile-web-app-capable`, `apple-touch-icon`, and `apple-mobile-web-app-status-bar-style`; hide "install" prompts once installed.
- [ ] Orientation and resize: layout survives rotate and split-screen/foldables without losing state.
- [ ] Error resilience: failed fetch shows retry in place, double-tap on submit is debounced/disabled while pending, optimistic UI where safe.

**Layout & chrome**
- [ ] `viewport-fit=cover` + `env(safe-area-inset-*)` on fixed headers, bottom nav, FABs, sheets.
- [ ] Use `100dvh`/`svh`, not `100vh`; no page-level bounce or rubber-banding revealing white (`overscroll-behavior: none` on html/body, scroll inside a container).
- [ ] Status bar / navigation bar styled to match theme (`@capacitor/status-bar`); splash hides only after first render.
- [ ] Keyboard: focused input scrolls into view, bottom nav/fixed bars don't ride above the keyboard, `Keyboard` resize mode set, no layout jump on dismiss.
- [ ] Orientation behavior is intentional (lock or handle).

**Touch feel**
- [ ] Tap targets >= 44x44px; `touch-action: manipulation` (no 300ms delay / double-tap zoom); `-webkit-tap-highlight-color: transparent` with a real `:active` pressed state.
- [ ] `user-select: none` on UI chrome (keep selectable for content text); no long-press image/link callout (`-webkit-touch-callout: none`).
- [ ] No hover-only affordances; no text zoom on input focus (inputs >= 16px).
- [ ] Haptics on key actions (`@capacitor/haptics`), sparingly.
- [ ] Scrolling is momentum-based (`-webkit-overflow-scrolling: touch` inside containers); no scroll-jank from heavy listeners (use `passive: true`).

**App lifecycle & connectivity**
- [ ] `appStateChange`: refresh stale data / resume timers on foreground; pause or persist on background. Long-running features (e.g. GPS) declared correctly. **[device]**
- [ ] Offline: detect with `@capacitor/network`, show state, queue writes, never blank-screen.
- [ ] Loading uses skeletons/spinners in place; transitions between screens are animated, not hard cuts.
- [ ] External links open in in-app browser (`@capacitor/browser`); share via native sheet (`@capacitor/share`); permissions requested in context with a rationale.
- [ ] App doesn't show browser UI artifacts: no URL-looking text, no right-click menu, no "Install app" banner inside the native shell.

## Rules

- Platform-gate native code with `Capacitor.isNativePlatform()`; web/PWA must degrade gracefully and not throw.
- Desktop-only affordances need a mobile guard; mobile fixes must not alter desktop layout. Check project instructions (CLAUDE.md) for protected mobile elements before editing.
- Prefer one central back/gesture handler over per-screen listeners; remove listeners on teardown.
- Don't claim a [device] item verified from a browser run — say it needs a device check and list the steps.

## Verification

Real mobile browser (Chrome Android, Safari iOS) and installed PWA: back after opening each overlay, back from a deep link, reload on a deep URL, switch app and return.
Browser (mobile emulation): safe-area CSS present, no horizontal scroll, tap states, PTR logic, overscroll, keyboard layout via viewport resize.
Device/emulator (`npx cap sync && npx cap run android|ios`): back button/gesture, edge-swipe, status bar, keyboard, background/foreground, offline toggle.

See [REFERENCE.md](REFERENCE.md) for code snippets and the manual test script.
