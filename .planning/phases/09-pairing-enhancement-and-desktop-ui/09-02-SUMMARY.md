---
phase: 09-pairing-enhancement-and-desktop-ui
plan: 02
subsystem: ui
tags: [qrcode, pairing, modal, vanilla-js, catppuccin]

requires:
  - phase: 09-01
    provides: pairing-code endpoint, device-manager API, URL detection
provides:
  - Pair Mobile header button with device count badge
  - QR code modal with SVG rendering and auto-refresh
  - Connection URL display with copy-to-clipboard
  - Paired devices tab with revoke and test push actions
affects: [mobile-app-pairing, push-notifications, device-management]

tech-stack:
  added: [qrcode (npm, browser bundle via esbuild)]
  patterns: [client-side QR SVG generation with theme-aware colors, tab-based modal with dedicated overlay]

key-files:
  created:
    - src/web/public/vendor/qrcode.min.js
  modified:
    - src/web/public/index.html
    - src/web/public/app.js
    - src/web/public/styles.css
    - package.json

key-decisions:
  - "Client-side QR generation via esbuild-bundled qrcode browser build (no server-side SVG rendering)"
  - "QR SVG uses theme --text color with transparent background for all theme compatibility"
  - "4-minute auto-refresh interval for 5-minute token expiry window"

patterns-established:
  - "Vendor bundling: esbuild from node_modules to public/vendor/ for browser-only libs"
  - "Tab-based modal: pair-tab buttons with data-tab attribute, show/hide content divs"

requirements-completed: [DWUI-01, DWUI-02, DWUI-03, DWUI-04, DWUI-05, DWUI-06, DWUI-07, DWUI-08]

duration: 4min
completed: 2026-03-29
---

# Phase 9 Plan 02: Desktop Web UI for Mobile Pairing Summary

**Pair Mobile header button with QR code modal, 4-minute auto-refresh, connection URLs, and device management tab with revoke/test-push actions**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-29T08:41:10Z
- **Completed:** 2026-03-29T08:44:43Z
- **Tasks:** 3 (2 auto + 1 checkpoint auto-approved)
- **Files modified:** 5

## Accomplishments
- Phone+QR icon button in header bar with paired device count badge
- QR code modal with theme-aware SVG rendering via client-side qrcode library
- Auto-refresh timer at 4-minute intervals with visible countdown
- Connection URL display (local, LAN, Tailscale, tunnel) with copy buttons
- Paired devices tab showing device name, platform, online status, push registration, last seen
- Revoke button with confirmation dialog, test push button with loading state

## Task Commits

Each task was committed atomically:

1. **Task 1: Install qrcode, add HTML/CSS for Pair Mobile button and modal** - `4bd12a1` (feat)
2. **Task 2: Implement QR modal logic, auto-refresh, and device management** - `d16d31b` (feat)
3. **Task 3: Visual verification** - auto-approved (auto mode active)

## Files Created/Modified
- `src/web/public/vendor/qrcode.min.js` - Browser-bundled QR code generator (23.7kb)
- `src/web/public/index.html` - Pair Mobile button in header, modal overlay with QR and devices tabs
- `src/web/public/app.js` - initPairMobile, showPairMobileModal, loadPairingCode, loadPairedDevices, handleDeviceAction, updatePairBadge methods
- `src/web/public/styles.css` - Modal, tabs, QR container, URL rows, device cards, badge, action buttons
- `package.json` - Added qrcode dependency

## Decisions Made
- Used esbuild to bundle qrcode browser build to vendor/ directory (no CDN dependency, follows existing vendor pattern for xterm)
- QR SVG renders with theme text color and transparent background so it works across all 13 Catppuccin themes
- Auto-refresh at 4 minutes leaves a 1-minute buffer before the 5-minute token expiry
- Device actions use event delegation on the device list container for efficient click handling

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] qrcode package has no pre-built browser bundle**
- **Found during:** Task 1 (installing qrcode)
- **Issue:** Plan referenced `node_modules/qrcode/build/qrcode.min.js` but the package does not ship a pre-built browser bundle
- **Fix:** Used npx esbuild to bundle `node_modules/qrcode/lib/browser.js` into `src/web/public/vendor/qrcode.min.js` as an IIFE with global name QRCode
- **Files modified:** src/web/public/vendor/qrcode.min.js (created)
- **Verification:** Bundle verified with node, 24kb, exports QRCode.toString
- **Committed in:** 4bd12a1 (Task 1 commit)

**2. [Rule 3 - Blocking] Plan referenced server.js vendor route but vendor files are served via static middleware**
- **Found during:** Task 1 (serving qrcode to browser)
- **Issue:** Plan suggested adding a vendor route in server.js, but existing vendor files (xterm) are served directly from public/vendor/ via express.static
- **Fix:** Placed bundled qrcode.min.js in public/vendor/ to match existing pattern, no server.js changes needed
- **Files modified:** None extra (used existing static serving)
- **Verification:** Script tag /vendor/qrcode.min.js will be served by express.static
- **Committed in:** 4bd12a1 (Task 1 commit)

---

**Total deviations:** 2 auto-fixed (2 blocking)
**Impact on plan:** Both fixes were necessary to deliver client-side QR generation. No scope creep, cleaner than planned approach.

## Issues Encountered
None beyond the auto-fixed deviations above.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Desktop UI complete for mobile pairing flow
- Requires Plan 09-01 backend endpoints to be functional for end-to-end testing
- Ready for Phase 10 (push notifications) and Phase 13 (error handling) integration

---
*Phase: 09-pairing-enhancement-and-desktop-ui*
*Completed: 2026-03-29*
