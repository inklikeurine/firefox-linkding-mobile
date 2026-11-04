# Plan: on-page "saved in linkding" indicator

## Why

On desktop, the toolbar icon gets a ★ badge when the current page is already
bookmarked (`setDynamicBadge` in `src/background.js`). On mobile (Firefox for
Android, Edge/Kiwi on Android) the extension button lives inside the browser
menu, so that badge is never visible. A small icon drawn on the page itself
gives the same quick check without opening any menu.

## What the user sees

- A small round linkding icon with a ★ (about 32px), fixed in a corner of
  the page, shown only when the page's URL is already saved.
- It fades to about 40% opacity after 3 seconds so it doesn't block the
  page, and goes back to full opacity when tapped.
- Tap: opens the bookmark's detail page in linkding
  (`<baseUrl>/bookmarks?details=<id>`, same URL that PR #154 uses).
- Long-press or a small ×: hides it on this page until the next page load.
- Archived bookmarks get a slightly different look (grey star) so
  "saved but archived" (from PR #155) can also be seen at a glance.

## Settings (options page)

- **Show indicator on bookmarked pages** (checkbox, off by default).
- **Indicator position**: bottom-right (default), bottom-left, top-right,
  top-left.

The indicator needs the same per-page lookup as the ★ badge, which today only
runs when **Pre-load page information while browsing** (`precacheEnabled`) is
on. Turning the indicator on should run that lookup too, and the option text
should give the same privacy note (every visited URL is sent to the linkding
server).

## How it works

1. **Content script** `src/content.js`, built to `build/content.js` as a third
   entry in `rollup.config.js`, plain JS with no Lit to keep it small. It is
   registered in `manifest.json` under `content_scripts` for `http://*/*` and
   `https://*/*`, with `run_at: document_idle` and `all_frames: false`.
2. **On load**, the content script asks the background script for the page's
   status: `runtime.sendMessage({ type: "ld-get-status", url: location.href })`.
   This avoids a race: `tabs.onUpdated` often fires before the content script
   is ready to receive anything.
3. **Background** (`src/background.js`) adds a `runtime.onMessage` handler. If
   the setting is on, it calls `loadServerMetadata(url, true)` (which already
   checks the cache first) and replies `{ bookmarked, archived, id, detailsUrl }`.
4. **Updates pushed to the page**: in the existing `tabs.onActivated` /
   `tabs.onUpdated` handlers next to `setDynamicBadge`, also call
   `tabs.sendMessage(tabId, { type: "ld-status", ... })`. This covers
   single-page apps that change the URL without reloading. Errors are ignored,
   since tabs with no content script will reject the message.
5. **After popup actions**: after save, delete or archive/unarchive in
   `src/popup-form.js`, and after the context-menu save in `background.js`,
   send an updated status to the current tab *before* `window.close()` so the
   icon shows up or disappears right away.
6. **Rendering**: a single host element attached to `document.documentElement`
   with a **closed Shadow DOM**, so page CSS can't change it and it can't
   change the page. Styles: `position: fixed`, the largest `z-index`,
   offsets that use `env(safe-area-inset-*)` for phones with notches or
   gesture bars, `pointer-events` only on the icon, and a tap target of at
   least 44px with the visible circle smaller. The icon is inline SVG with
   no network requests.
7. **Skip** the linkding server's own pages (URL starts with `baseUrl`) and
   non-http(s) pages.

## Files touched

| File | Change |
| --- | --- |
| `src/content.js` (new) | Ask for status, draw/update/remove the icon, handle tap and hide |
| `src/background.js` | `onMessage` status handler; push status on tab changes and context-menu save |
| `src/popup-form.js` | Notify the tab after save/delete/archive |
| `src/browser.js` | `sendTabMessage(tabId, msg)` helper that swallows errors |
| `src/configuration.js` | `showPageIndicator: false`, `pageIndicatorPosition: "bottom-right"` defaults |
| `src/cache.js` | Run the lookup when `precacheEnabled` **or** `showPageIndicator` is on |
| `src/options.js` | New checkbox and position select |
| `rollup.config.js` | Add `src/content.js` entry, IIFE output |
| `manifest.json` | `content_scripts` entry |

## Open questions

- Should tapping the icon open the detail page (as planned) or show a small
  card with title and tags on the page? Opening the detail page is simpler
  and reuses PR #154.
- The cache only holds the most recent URL, so switching back and forth
  between tabs causes repeat lookups. That's fine for a first version; a
  small LRU cache could come later if it causes trouble.

## Testing

- Desktop Firefox/Chrome: load unpacked, turn on the setting, visit a saved
  and an unsaved URL, save/delete/archive from the popup and check the icon
  updates without a reload. Also test a single-page app (e.g. GitHub) that
  changes the URL without reloading.
- Firefox for Android: `web-ext run -t firefox-android`, then check
  placement against the bottom toolbar, safe-area insets, tap, and
  long-press.
- Pages with aggressive CSS (`* { all: unset }`, high z-index overlays) to
  confirm the shadow DOM keeps the icon isolated.
