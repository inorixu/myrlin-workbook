---
phase: 05-data-screens
plan: 04
title: "Resource Monitor Screen"
subsystem: mobile-resources
tags: [resources, monitoring, system-stats, process-management]
dependency_graph:
  requires: [05-00-shared-ui]
  provides: [resource-monitor-screen, useResources-hook, kill-process-mutation]
  affects: [more-tab-navigation]
tech_stack:
  added: []
  patterns: [auto-refresh-query, pulsing-live-indicator, actionsheet-confirm]
key_files:
  created:
    - mobile/hooks/useResources.ts
    - mobile/components/resources/SystemStats.tsx
    - mobile/components/resources/SessionResources.tsx
    - mobile/app/(tabs)/more/resources.tsx
  modified:
    - mobile/types/api.ts
    - mobile/services/api-client.ts
    - mobile/app/(tabs)/more/_layout.tsx
decisions:
  - FlashList scrollEnabled=false inside ScrollView for nested scroll compatibility
  - Pulsing dot via Reanimated opacity loop for live indicator
  - Card style spread instead of array to satisfy ViewStyle type
  - ActionSheet for kill confirmation instead of Alert.alert for design consistency
metrics:
  duration: 4m
  completed: "2026-03-28T00:00:00Z"
  tasks_completed: 1
  tasks_total: 1
  files_created: 4
  files_modified: 3
requirements:
  - RSRC-01
  - RSRC-02
  - RSRC-03
---

# Phase 5 Plan 4: Resource Monitor Screen Summary

Resource monitor screen with system CPU/memory overview, per-session resource consumption list, color-coded CPU thresholds, and process kill with ActionSheet confirmation.

## What Was Built

### Types and API Client (api.ts, api-client.ts)
- Added `SystemInfo`, `ClaudeSessionResource`, and `ResourceMetrics` interfaces
- Added `getResources()` and `killProcess(pid)` methods to MyrlinAPIClient

### useResources Hook (hooks/useResources.ts)
- `useResources()` query with 10s auto-refresh interval and 5s staleTime
- `useKillProcess()` mutation that invalidates both resources and sessions caches

### SystemStats Component (components/resources/SystemStats.tsx)
- Two side-by-side cards: CPU percentage (color-coded) and Memory with progress bar
- Summary row below: uptime formatted as "Xd Xh Xm", total Claude CPU%, total Claude memory
- Color thresholds: green < 50%, yellow 50-80%, red > 80%

### SessionResources Component (components/resources/SessionResources.tsx)
- FlashList of ClaudeSessionResource items
- Each row: StatusDot, session name, workspace, working dir, CPU/memory/PID/port badges
- Kill button with ActionSheet confirmation dialog
- Empty state when no sessions running

### Resources Screen (app/(tabs)/more/resources.tsx)
- SafeAreaView with ScrollView, pull-to-refresh
- Pulsing green "Live" badge in header via Reanimated
- Loading skeleton cards, error empty state
- Composes SystemStats and SessionResources components

### More Layout Update (_layout.tsx)
- Added Stack.Screen entry for "resources" route

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed component prop mismatches**
- Found during: Task 1, initial tsc check
- Issue: EmptyState uses `description` not `subtitle`, `icon` is ReactNode not string; SectionHeader has no `count` prop; ActionSheet has no `title`/`message` props; Card `style` is ViewStyle not ViewStyle[]
- Fix: Used correct prop names, spread operator for Card style, Ionicons for icon ReactNode, count embedded in SectionHeader title string
- Files: SessionResources.tsx, SystemStats.tsx, resources.tsx
- Commit: 97202cd

## Commits

| Hash | Description |
|------|-------------|
| 97202cd | feat(05-04): add resource monitor screen with system stats and kill process |

## Self-Check: PASSED

All 4 created files verified on disk. Commit 97202cd verified in git log.
