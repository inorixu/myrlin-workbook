---
phase: 03-sessions-and-workspaces
plan: 01
subsystem: mobile-sessions-data-layer
tags: [types, api-client, sse, tanstack-query, flashlist, session-list]
dependency_graph:
  requires: [01-01, 01-02, 02-01, 02-02]
  provides: [session-types, api-client-methods, sse-client, query-hooks, ui-store, session-list-screen]
  affects: [03-02, 03-03, 03-04]
tech_stack:
  added: ["@tanstack/react-query", "@shopify/flash-list", "react-native-sse"]
  patterns: [sse-cache-invalidation, tanstack-query-hooks, zustand-ui-store, flashlist-memo-cards]
key_files:
  created:
    - mobile/services/sse-client.ts
    - mobile/hooks/useSSE.ts
    - mobile/hooks/useAPIClient.ts
    - mobile/hooks/useSessions.ts
    - mobile/hooks/useWorkspaces.ts
    - mobile/stores/ui-store.ts
    - mobile/components/sessions/ActivityIndicator.tsx
    - mobile/components/sessions/SessionCard.tsx
  modified:
    - mobile/types/api.ts
    - mobile/services/api-client.ts
    - mobile/app/_layout.tsx
    - mobile/app/(tabs)/sessions/index.tsx
    - mobile/package.json
    - mobile/package-lock.json
decisions:
  - QueryClientProvider added to root _layout.tsx (required for TanStack Query, was missing)
  - Card component lacks onLongPress, so SessionCard wraps Pressable directly
  - SSE events map to query key invalidation only, never direct state mutation
  - FlashList v2 used without estimatedItemSize (auto-measures)
metrics:
  duration: 6m
  completed: "2026-03-29T00:56:00Z"
  tasks: 2
  files_created: 8
  files_modified: 6
---

# Phase 3 Plan 1: Data Foundation and Session List Summary

Extended types, API client (35 methods), SSE client with TanStack Query cache invalidation, and full session list screen with FlashList, workspace filter chips, and real-time updates.

## What Was Built

### Task 1: Types, API Client, SSE Client, Query Hooks, UI Store

Extended `mobile/types/api.ts` with 11 new interfaces: Session, SessionLog, SessionCost, Workspace, WorkspaceGroup, SessionTemplate, FileConflict, SSEEvent, CreateSessionInput, SearchResult, Subagent.

Extended `mobile/services/api-client.ts` with 35 new methods covering sessions (CRUD, start/stop/restart, cost, subagents, auto-title, summarize), workspaces (CRUD, reorder), groups (CRUD, add workspace), templates (CRUD), search (keyword, semantic), conflicts, discovery, and directory browsing.

Created `mobile/services/sse-client.ts` using react-native-sse (RNEventSource). Connects to `/api/events?token=X`, parses JSON payloads, dispatches to handler callback. RNEventSource handles reconnection internally.

Created `mobile/hooks/useSSE.ts` mapping 15 SSE event types to TanStack Query key invalidations. Session events with IDs also invalidate specific session queries. Connects/disconnects based on server connection status.

Created `mobile/hooks/useAPIClient.ts` returning a memoized MyrlinAPIClient for the active server (null when no server connected).

Created `mobile/hooks/useSessions.ts` with 8 hooks: useSessions (query), useSession (query), useStartSession, useStopSession, useRestartSession, useDeleteSession, useUpdateSession, useCreateSession (mutations).

Created `mobile/hooks/useWorkspaces.ts` with useWorkspaces and useGroups query hooks.

Created `mobile/stores/ui-store.ts` (Zustand, no persistence) with selectedWorkspaceFilter and sessionSortOrder state.

Installed `@tanstack/react-query`, `@shopify/flash-list`, `react-native-sse`.

### Task 2: SessionCard Component and Session List Screen

Created `mobile/components/sessions/ActivityIndicator.tsx` with Reanimated pulsing dot for running sessions. Status-to-label mapping: running (green pulse), idle (yellow), stopped (muted), error (red).

Created `mobile/components/sessions/SessionCard.tsx` as a React.memo'd Pressable card. Shows StatusDot, session name (truncated), time-ago, topic (or "No topic"), ActivityIndicator, up to 3 tag Badges, workspace Chip when showWorkspace is true, and "Needs Input" warning Badge for idle sessions.

Replaced `mobile/app/(tabs)/sessions/index.tsx` placeholder with full implementation: FlashList with SessionCard items, horizontal workspace filter chips, SegmentedControl for All/Recent, pull-to-refresh wired to query refetch, 5-Skeleton loading state, EmptyState with retry for errors, EmptyState with "Create Session" CTA for empty lists, ActionSheet on long-press with Start/Stop/Restart/Rename/Move/Delete stubs.

Added `QueryClientProvider` to `mobile/app/_layout.tsx` (both auth and main branches) with default staleTime of 5s and 2 retries.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Missing dependencies: @tanstack/react-query, @shopify/flash-list, react-native-sse**
- Found during: Task 1 (pre-execution check)
- Issue: These libraries were listed in the research as "already installed in Phase 1-2" but were not in package.json
- Fix: Installed all three with --legacy-peer-deps
- Commit: 0d8eddd

**2. [Rule 3 - Blocking] Missing QueryClientProvider in root layout**
- Found during: Task 2
- Issue: TanStack Query hooks require a QueryClientProvider ancestor, but the root layout did not have one
- Fix: Added QueryClientProvider wrapping both auth and main navigation branches in _layout.tsx
- Commit: bdde2fa

**3. [Rule 2 - Missing functionality] Card component lacks onLongPress support**
- Found during: Task 2
- Issue: The Card UI component only accepts onPress, not onLongPress. SessionCard needs long-press for action sheet.
- Fix: SessionCard uses Pressable directly instead of Card wrapper, applying matching card styles
- Commit: bdde2fa

## Verification Results

- TypeScript: `npx tsc --noEmit` passes with zero errors
- API client: 35 methods (exceeds the 30+ requirement)
- SessionCard: memoized with React.memo
- FlashList: no estimatedItemSize prop (FlashList v2 auto-measures)
- All components use useTheme() for colors (no hardcoded colors)
- SSE client connects to /api/events?token=X
- useSSE maps 15 event types to query key invalidations
- All 8 new files have docstring headers
- All functions have JSDoc comments

## Commits

| Task | Hash | Description |
|------|------|-------------|
| 1 | 0d8eddd | Types, API client, SSE client, query hooks, UI store |
| 2 | bdde2fa | SessionCard component and full session list screen |

## Self-Check: PASSED

All 8 created files verified on disk. Both commits (0d8eddd, bdde2fa) verified in git log. TypeScript compilation passes with zero errors.
