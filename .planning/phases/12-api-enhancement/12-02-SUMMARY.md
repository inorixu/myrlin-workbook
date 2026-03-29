---
phase: 12-api-enhancement
plan: 02
subsystem: api
tags: [scrollback, pagination, token-refresh, pty, auth]

requires:
  - phase: 08-persistent-auth
    provides: pairedDevices store, activeTokens set, device token management
provides:
  - GET /api/sessions/:id/scrollback with lines/from pagination
  - GET /api/sessions/:id/logs with limit/offset pagination
  - POST /api/auth/refresh for device token renewal
  - store.refreshDeviceToken method
  - ptyManager.getScrollbackLines method
affects: [13-error-handling, mobile-app]

tech-stack:
  added: []
  patterns: [paginated-response-shape, token-rotation]

key-files:
  created: []
  modified:
    - src/web/pty-manager.js
    - src/web/server.js
    - src/web/auth.js
    - src/state/store.js

key-decisions:
  - "Scrollback joins all buffer chunks then splits by newline for accurate line counting"
  - "Token refresh only for device tokens (403 for browser tokens) as security boundary"
  - "Scrollback returns 200 with empty lines when PTY session not found (session may exist but PTY is dead)"

patterns-established:
  - "Paginated response shape: { items, total, hasMore } used by scrollback and logs endpoints"
  - "Token rotation: delete old, add new, persist to store in single request"

requirements-completed: [SCRL-01, SCRL-02, SCRL-03, TOKN-05, TOKN-06]

duration: 3min
completed: 2026-03-28
---

# Phase 12 Plan 02: Scrollback, Logs Pagination, and Token Refresh Summary

**Scrollback pagination via PTY buffer line splitting, logs pagination with offset/limit, and device token refresh with 90-day rotation**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-28T21:00:43Z
- **Completed:** 2026-03-28T21:03:50Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- Scrollback endpoint splits PTY buffer into lines with from=end (last N) or from=index pagination
- Logs endpoint paginates session.logs array with limit/offset and hasMore indicator
- Token refresh generates new 90-day token, atomically swaps old for new in activeTokens and store
- Security boundary: browser session tokens get 403 on refresh (device tokens only)

## Task Commits

Each task was committed atomically:

1. **Task 1: Scrollback pagination endpoint and logs pagination** - `ea66b45` (feat)
2. **Task 2: Token refresh endpoint** - `0dab260` (feat)

## Files Created/Modified
- `src/web/pty-manager.js` - Added getScrollbackLines method for paginated scrollback access
- `src/web/server.js` - Added GET /api/sessions/:id/scrollback and GET /api/sessions/:id/logs routes
- `src/web/auth.js` - Added POST /api/auth/refresh route inside setupAuth
- `src/state/store.js` - Added refreshDeviceToken method for atomic token swap

## Decisions Made
- Scrollback joins all buffer chunks then splits by newline for accurate line counting (buffer chunks are raw PTY output, not line-aligned)
- Token refresh returns 403 for browser tokens to maintain the security boundary between short-lived password-based tokens and long-lived device pairing tokens
- Scrollback returns 200 with empty data when PTY session is missing (not 404), since the session may exist in the store but the PTY process has exited

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All three new endpoints ready for mobile client consumption
- Phase 13 (error handling) can add standardized error shapes to these endpoints
- Token refresh enables silent re-auth in the mobile app before token expiry

---
*Phase: 12-api-enhancement*
*Completed: 2026-03-28*
