---
phase: 10-push-enhancement
plan: 01
subsystem: push
tags: [expo, push-notifications, retry, batching, deep-links]

# Dependency graph
requires:
  - phase: 08-token-persistence
    provides: pairedDevices store with pushToken field, getPairedDevices/updatePairedDevice methods
provides:
  - sendPushWithRetry with exponential backoff (1s, 2s, 4s), max 3 attempts
  - queuePush/flushPushQueue batching with 2-second coalescing window
  - Rich push payloads with route (deep link) and badge (iOS running session count)
  - Stale token cleanup on DeviceNotRegistered (nullifies pushToken in pairedDevices)
affects: [10-push-enhancement, mobile-app]

# Tech tracking
tech-stack:
  added: []
  patterns: [exponential-backoff-retry, event-batching-queue, push-token-lifecycle]

key-files:
  created: []
  modified: [src/web/push.js]

key-decisions:
  - "Retry sends per-device (not batch POST) so one stale token does not block others"
  - "Batching uses module-level Map and setTimeout, flushed every 2 seconds"
  - "Badge count derived from store sessions with status running"
  - "Legacy pushDevices cleanup kept alongside pairedDevices cleanup for backward compat"

patterns-established:
  - "Push retry: sendPushWithRetry wraps fetch with exponential backoff loop"
  - "Push batching: queuePush collects, flushPushQueue dispatches after BATCH_WINDOW_MS"
  - "Stale token lifecycle: DeviceNotRegistered triggers updatePairedDevice(id, { pushToken: null })"

requirements-completed: [PUSH-01, PUSH-02, PUSH-05, PUSH-06, PUSH-07]

# Metrics
duration: 2min
completed: 2026-03-29
---

# Phase 10 Plan 01: Push Enhancement Summary

**Retry with exponential backoff, 2-second batching queue, rich payloads with deep links and badge, and stale token cleanup from pairedDevices**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-29T08:34:06Z
- **Completed:** 2026-03-29T08:36:25Z
- **Tasks:** 2
- **Files modified:** 1

## Accomplishments
- sendPushWithRetry retries up to 3 times with 1s/2s/4s exponential backoff before giving up
- Push events within a 2-second window are batched into a single summary notification per device
- Rich payloads include route field for mobile deep linking and badge count for iOS
- DeviceNotRegistered errors automatically nullify the stale pushToken in the paired device record
- All push sourced from store.getPairedDevices() instead of legacy pushDevices array

## Task Commits

Each task was committed atomically:

1. **Task 1: Add sendPushWithRetry and stale token cleanup** - `0cdbd10` (feat)
2. **Task 2: Add push batching queue with 2-second flush window** - `b364218` (feat)

## Files Created/Modified
- `src/web/push.js` - Enhanced with retry, batching, rich payloads, stale token cleanup

## Decisions Made
- Retry sends per-device individually (not batch POST) so one stale token does not block delivery to other devices
- Batching uses module-level Map and setTimeout with 2-second window; flushPushQueue exported for test-time flushing
- Badge count derived from counting sessions with status 'running' in the store
- Legacy pushDevices cleanup preserved alongside pairedDevices cleanup for backward compatibility

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- push.js exports queuePush and flushPushQueue for Plan 10-02 (event trigger wiring in pairing.js)
- sendPush still available for direct push from device-manager.js (test-push endpoint)
- All 52 existing tests continue to pass

---
*Phase: 10-push-enhancement*
*Completed: 2026-03-29*
