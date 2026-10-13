# Reference: snippets and device test script

## Base CSS

```css
html, body { overscroll-behavior: none; height: 100%; }
body {
  -webkit-tap-highlight-color: transparent;
  -webkit-touch-callout: none;
  touch-action: manipulation;
}
.app-scroll { height: 100dvh; overflow-y: auto; overscroll-behavior-y: contain; -webkit-overflow-scrolling: touch; }
.chrome { user-select: none; -webkit-user-select: none; }
input, textarea, select { font-size: 16px; }          /* prevents iOS focus zoom */
.header { padding-top: env(safe-area-inset-top); }
.bottom-nav { padding-bottom: env(safe-area-inset-bottom); }
button:active, .tap:active { transform: scale(.97); opacity: .85; }
```

Viewport: `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">`

## Central back-button handler (Android)

```js
import { App } from '@capacitor/app';
import { Capacitor } from '@capacitor/core';

const overlays = [];                      // push close() fns as modals/sheets open; pop on close
if (Capacitor.isNativePlatform()) {
  App.addListener('backButton', ({ canGoBack }) => {
    if (overlays.length) return overlays.pop()();   // 1. close topmost overlay
    if (canGoBack || history.length > 1) return history.back(); // 2. pop screen
    if (currentTab !== 'home') return goTab('home'); // 3. back to root tab
    App.exitApp();                                   // 4. exit only at root
  });
}
```

## Browser/PWA back: history-synced overlays (works in Capacitor too)

```js
const stack = [];                         // open overlays, topmost last
function openOverlay(id, show, hide) {
  show(); stack.push({ id, hide });
  history.pushState({ overlay: id }, '');  // back now has something to pop
}
function closeOverlay() {                 // call from X / scrim / Escape
  if (stack.length) history.back();       // popstate does the actual hiding
}
window.addEventListener('popstate', () => {
  if (stack.length) { stack.pop().hide(); return; }  // 1. overlay first
  routeFromPath();                                    // 2. then screen
});
// Screen changes: pushState on forward navigation, not replaceState.
// Capacitor Android: backButton handler just calls history.back() when canGoBack,
// so one popstate path serves browser, PWA and native.

window.addEventListener('pageshow', e => { if (e.persisted) refreshStale(); });
document.addEventListener('visibilitychange', () => {
  document.visibilityState === 'hidden' ? saveDraft() : refreshStale();
});
```

## Browser edge-case snippets

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover, interactive-widget=resizes-content">
<meta name="color-scheme" content="light dark">
<meta name="theme-color" content="#FFFFFF" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#0A0A0A" media="(prefers-color-scheme: dark)">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
```

```css
html { -webkit-text-size-adjust: 100%; background: var(--bg); }
@media (hover: hover) { .btn:hover { filter: brightness(1.05); } }  /* no sticky hover on touch */
.map, .canvas { touch-action: none; }                                /* app-owned gestures only here */
@media (prefers-reduced-motion: reduce) { * { animation-duration: .01ms !important; transition-duration: .01ms !important; } }
```

```js
// Keyboard-aware fixed bar (iOS keeps layout viewport; track visualViewport)
const vv = window.visualViewport;
function fit() {
  const kb = Math.max(0, window.innerHeight - vv.height - vv.offsetTop);
  document.documentElement.style.setProperty('--kb', kb + 'px');
}
vv?.addEventListener('resize', fit); vv?.addEventListener('scroll', fit);
document.addEventListener('focusin', e => setTimeout(() => e.target.scrollIntoView({ block: 'center', behavior: 'smooth' }), 300));

// Body scroll lock for modals (restore position on close)
let lockY = 0;
function lockScroll() { lockY = scrollY; document.body.style.cssText += `position:fixed;top:-${lockY}px;left:0;right:0;`; }
function unlockScroll() { document.body.style.position = document.body.style.top = ''; scrollTo(0, lockY); }

// Scroll restoration per screen
history.scrollRestoration = 'manual';
const scrollPos = {};                      // key: screen path
function saveScroll(path) { scrollPos[path] = document.querySelector('.app-scroll').scrollTop; }
function restoreScroll(path) { document.querySelector('.app-scroll').scrollTop = scrollPos[path] || 0; }

// Wake lock while recording/navigating
let wl; async function keepAwake() { try { wl = await navigator.wakeLock.request('screen'); } catch {} }
document.addEventListener('visibilitychange', () => { if (document.visibilityState === 'visible' && recording) keepAwake(); });

// Share with fallback
async function share(data) {
  if (navigator.share) { try { return await navigator.share(data); } catch (e) { if (e.name === 'AbortError') return; } }
  await navigator.clipboard?.writeText(data.url); toast('Link copied');
}

// Screen transitions
function go(update) {
  if (document.startViewTransition && !matchMedia('(prefers-reduced-motion: reduce)').matches) document.startViewTransition(update);
  else update();
}
```

Input attributes cheat sheet: email `type=email inputmode=email autocomplete=email autocapitalize=off`; phone `type=tel autocomplete=tel`; number `inputmode=numeric pattern="[0-9]*"`; search `type=search enterkeyhint=search`; names `autocomplete=name autocapitalize=words`.

## Pull-to-refresh (custom, container-scoped)

```js
function pullToRefresh(el, onRefresh, threshold = 70) {
  let startY = 0, pulling = false;
  el.addEventListener('touchstart', e => { pulling = el.scrollTop === 0; startY = e.touches[0].clientY; }, { passive: true });
  el.addEventListener('touchmove', e => {
    if (!pulling) return;
    const dy = e.touches[0].clientY - startY;
    if (dy > 0) showIndicator(Math.min(dy / threshold, 1));   // indicator only; don't preventDefault
  }, { passive: true });
  el.addEventListener('touchend', async e => {
    if (!pulling) return; pulling = false;
    if (e.changedTouches[0].clientY - startY > threshold) { await onRefresh(); haptic(); }
    hideIndicator();
  });
}
```
Skip on map containers. Refresh data via fetch, not `location.reload()`.

## Lifecycle, network, keyboard, status bar

```js
App.addListener('appStateChange', ({ isActive }) => isActive ? refreshStale() : persistState());
import { Network } from '@capacitor/network';
Network.addListener('networkStatusChange', s => setOffline(!s.connected));
import { Keyboard } from '@capacitor/keyboard';
Keyboard.addListener('keyboardWillShow', i => document.documentElement.style.setProperty('--kb', i.keyboardHeight + 'px'));
Keyboard.addListener('keyboardWillHide', () => document.documentElement.style.setProperty('--kb', '0px'));
import { StatusBar, Style } from '@capacitor/status-bar';
StatusBar.setStyle({ style: Style.Dark }); StatusBar.setBackgroundColor({ color: '#0A0A0A' }); // Android
```

`capacitor.config.ts` worth checking: `plugins.SplashScreen.launchAutoHide:false` (hide after first render), `Keyboard.resize`, `android.allowMixedContent:false`, correct `server`/start path.

## Manual device test script (10 min)

1. Open a modal, press back: modal closes, screen stays.
2. Go 3 screens deep, press back x3: returns step by step; back at root exits (or does nothing on iOS).
3. iOS: swipe from left edge on a pushed screen: goes back.
4. Pull down on a list at top: refresh indicator, data updates; mid-list pull does nothing.
5. Scroll hard past top/bottom: no white gap, no page bounce.
6. Tap an input at the bottom: field visible above keyboard, nav not floating mid-screen.
7. Rotate / notch device: header and nav clear of cutouts.
8. Background app 30s, return: data refreshes, state intact.
9. Airplane mode: offline notice, no blank screen; reconnect recovers.
10. Long-press buttons/images: no browser menu. Double-tap: no zoom.
