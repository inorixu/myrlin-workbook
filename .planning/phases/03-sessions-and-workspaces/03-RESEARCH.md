# Phase 3: Sessions and Workspaces - Research

**Researched:** 2026-03-28
**Domain:** React Native screens, SSE real-time sync, TanStack Query data layer, FlashList
**Confidence:** HIGH

## Summary

Phase 3 is the largest phase in the mobile app (30 requirements), covering session listing, session detail, session CRUD, workspace management, search, and conflict detection. The server already has every API endpoint needed; this phase is purely mobile-side implementation.

The core technical challenge is wiring SSE events through TanStack Query cache invalidation to drive real-time UI updates in FlashList-rendered session lists. The existing `api-client.ts` (Phase 2) provides the HTTP client foundation but currently only has auth endpoints. It needs to be extended with 20+ methods covering sessions, workspaces, groups, templates, search, and conflicts.

**Primary recommendation:** Decompose into 4-5 plans: (1) API client extension + types + SSE hook, (2) session list screen + session card component, (3) session detail screen + AI features, (4) workspace management screens, (5) search + quick switcher + conflicts. Plans 2-5 can partially parallelize after plan 1 completes.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SESS-01 | Scrollable session list with real-time SSE status | SSE client hook + FlashList + TanStack Query invalidation pattern |
| SESS-02 | Filter sessions by workspace (filter chips) | GET /api/sessions?mode=workspace&workspaceId=X + Chip component |
| SESS-03 | Search sessions by name/topic | GET /api/search?q=X (keyword search endpoint) |
| SESS-04 | Session detail (metadata, cost, logs, subagents, tags) | GET session + GET /api/sessions/:id/cost + GET /api/sessions/:id/subagents |
| SESS-05 | Start, stop, restart session | POST /api/sessions/:id/start, /stop, /restart |
| SESS-06 | Create new session (with template support) | POST /api/sessions + GET /api/templates + GET /api/discover |
| SESS-07 | Rename session | PUT /api/sessions/:id { name } |
| SESS-08 | Delete (hide) session | DELETE /api/sessions/:id |
| SESS-09 | Move session to different workspace | PUT /api/sessions/:id { workspaceId } |
| SESS-10 | Add/remove tags | PUT /api/sessions/:id { tags } |
| SESS-11 | View and use session templates | GET /api/templates, POST /api/sessions from template data |
| SESS-12 | Auto-title (AI) | POST /api/sessions/:id/auto-title |
| SESS-13 | Summarize (AI) | POST /api/sessions/:id/summarize |
| SESS-14 | Recently active sessions | GET /api/sessions?mode=recent |
| SESS-15 | Save session as template | POST /api/templates (populate from session data) |
| SESS-16 | Activity indicator on cards | SSE session:updated events carry status field |
| SESS-17 | Needs-input badge | SSE session:updated or session:log events; detect from session status/logs |
| SESS-18 | Bulk stop sessions | POST /api/sessions/:id/stop in batch |
| WORK-01 | View workspaces with session counts | GET /api/workspaces (includes sessions array per workspace) |
| WORK-02 | Create workspace | POST /api/workspaces { name, description, color } |
| WORK-03 | Rename workspace | PUT /api/workspaces/:id { name } |
| WORK-04 | Delete workspace | DELETE /api/workspaces/:id |
| WORK-05 | Reorder workspaces (drag-drop) | PUT /api/workspaces/reorder { order: string[] } |
| WORK-06 | Collapsible workspace groups | GET /api/groups, POST/PUT/DELETE /api/groups/:id |
| WORK-07 | Workspace colors | PUT /api/workspaces/:id { color } |
| SRCH-01 | Search by name/topic/content | GET /api/search?q=X (keyword JSONL search) |
| SRCH-02 | AI semantic search | POST /api/search-conversations { query, workspaceId? } |
| SRCH-03 | Quick switcher modal | Modal with SearchBar + recent sessions + search results |
| CNFL-01 | View file conflicts | GET /api/conflicts |
| CNFL-02 | Conflict badge in More tab | Periodic conflict polling or SSE-triggered refetch |
</phase_requirements>

## Standard Stack

### Core (already installed in Phase 1-2)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @tanstack/react-query | 5.x | Server data cache, SSE invalidation | Single source of truth for all server data |
| @shopify/flash-list | 2.x | Session list rendering | 60fps even with complex SessionCard items |
| react-native-sse | 1.x | SSE subscription for real-time events | Pure JS, supports custom auth headers |
| zustand | 5.x | UI state (filters, sort, collapse states) | Already used for server/auth/theme stores |
| react-native-reanimated | 4.x | Layout animations, skeleton loaders | 60fps UI-thread animations |
| react-native-gesture-handler | 2.x | Swipe-to-delete, long-press, drag-reorder | Foundation for all gesture interactions |

### Supporting (new for Phase 3)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| expo-haptics | ~13.x | Tactile feedback on actions | Long press, delete, start/stop, reorder |

### No New Dependencies Required

Phase 3 uses only libraries already installed in Phases 1-2. No additional `npm install` needed.

## Architecture Patterns

### Recommended File Structure for Phase 3

```
mobile/
  services/
    api-client.ts          # Extend with session/workspace/search/conflict methods
    sse-client.ts          # NEW: SSE subscription manager
  hooks/
    useSSE.ts              # NEW: SSE hook that invalidates TanStack Query keys
    useSessions.ts         # NEW: TanStack Query hooks for sessions
    useWorkspaces.ts       # NEW: TanStack Query hooks for workspaces
    useSearch.ts           # NEW: search hook with debouncing
  types/
    api.ts                 # Extend with Session, Workspace, SSEEvent, etc. types
  stores/
    ui-store.ts            # NEW: UI preferences (filter, sort, collapse states)
  components/
    sessions/
      SessionCard.tsx      # NEW: session list item
      SessionActions.tsx   # NEW: action sheet wrapper for session operations
      ActivityIndicator.tsx # NEW: activity kind label
      NewSessionModal.tsx  # NEW: create session form
    workspaces/
      WorkspaceItem.tsx    # NEW: workspace list item with session count
      WorkspaceActions.tsx # NEW: action sheet for workspace ops
      NewWorkspaceModal.tsx # NEW: create workspace form
    search/
      QuickSwitcher.tsx    # NEW: modal with search + recent results
    conflicts/
      ConflictRow.tsx      # NEW: file conflict list item
  app/
    (tabs)/sessions/
      index.tsx            # REPLACE placeholder with full session list
      [id].tsx             # NEW: session detail screen
    (tabs)/more/
      workspaces.tsx       # NEW: workspace management screen
      search.tsx           # NEW: search screen
      conflicts.tsx        # NEW: conflict center screen
      session-manager.tsx  # NEW: bulk session controls
      recent.tsx           # NEW: recently active sessions
      templates.tsx        # NEW: template list screen
```

### Pattern 1: SSE Client Service (Pure, no React)

**What:** A standalone SSE client that connects to `/api/events?token=X`, parses events, and calls a callback. Does NOT manage React state directly.

**When:** Created once when a server connection is established.

```typescript
// services/sse-client.ts
import RNEventSource from 'react-native-sse';

type SSEEventHandler = (event: { type: string; data: any }) => void;

export class SSEClient {
  private es: RNEventSource | null = null;
  private handler: SSEEventHandler;

  constructor(handler: SSEEventHandler) {
    this.handler = handler;
  }

  connect(baseUrl: string, token: string): void {
    this.disconnect();
    const url = `${baseUrl}/api/events?token=${encodeURIComponent(token)}`;
    this.es = new RNEventSource(url);
    this.es.addEventListener('message', (event) => {
      try {
        const parsed = JSON.parse(event.data);
        this.handler(parsed);
      } catch {}
    });
    this.es.addEventListener('error', () => {
      // Reconnection is handled internally by RNEventSource
    });
  }

  disconnect(): void {
    if (this.es) {
      this.es.close();
      this.es = null;
    }
  }
}
```

### Pattern 2: SSE-Driven TanStack Query Invalidation

**What:** A React hook that creates an SSE client and maps event types to query key invalidations. SSE events NEVER update state directly; they only mark queries as stale.

**When:** Called once at the top-level tab layout, alongside useConnectionHealth.

```typescript
// hooks/useSSE.ts
import { useEffect, useRef } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { useServerStore } from '@/stores/server-store';
import { SSEClient } from '@/services/sse-client';

const EVENT_TO_QUERY_KEYS: Record<string, string[][]> = {
  'session:created':   [['sessions']],
  'session:updated':   [['sessions']],  // also invalidate ['session', id] in handler
  'session:deleted':   [['sessions']],
  'session:log':       [['sessions']],
  'workspace:created': [['workspaces']],
  'workspace:updated': [['workspaces']],
  'workspace:deleted': [['workspaces']],
  'group:created':     [['groups']],
  'group:updated':     [['groups']],
  'group:deleted':     [['groups']],
  'template:created':  [['templates']],
  'template:deleted':  [['templates']],
  'settings:updated':  [['settings']],
};

export function useSSE(): void {
  const queryClient = useQueryClient();
  const getActiveServer = useServerStore((s) => s.getActiveServer);
  const connectionStatus = useServerStore((s) => s.connectionStatus);
  const clientRef = useRef<SSEClient | null>(null);

  useEffect(() => {
    const server = getActiveServer();
    if (!server || !server.token || connectionStatus !== 'connected') {
      clientRef.current?.disconnect();
      return;
    }

    const sse = new SSEClient((event) => {
      // Invalidate mapped query keys
      const keys = EVENT_TO_QUERY_KEYS[event.type];
      if (keys) {
        for (const key of keys) {
          queryClient.invalidateQueries({ queryKey: key });
        }
      }
      // For session events with an id, also invalidate the specific session
      if (event.type.startsWith('session:') && event.data?.id) {
        queryClient.invalidateQueries({ queryKey: ['session', event.data.id] });
      }
    });

    sse.connect(server.url, server.token);
    clientRef.current = sse;

    return () => {
      sse.disconnect();
    };
  }, [connectionStatus, getActiveServer, queryClient]);
}
```

### Pattern 3: API Client Hook for Active Server

**What:** A hook that returns a configured MyrlinAPIClient for the active server. Changes when the active server changes.

```typescript
// hooks/useAPIClient.ts
import { useMemo } from 'react';
import { useServerStore } from '@/stores/server-store';
import { createAPIClient, MyrlinAPIClient } from '@/services/api-client';

export function useAPIClient(): MyrlinAPIClient | null {
  const server = useServerStore((s) => s.getActiveServer());
  return useMemo(() => {
    if (!server || !server.token) return null;
    return createAPIClient(server.url, server.token);
  }, [server?.url, server?.token]);
}
```

### Pattern 4: Query Hooks per Domain

**What:** Each data domain gets its own hook file wrapping TanStack Query.

```typescript
// hooks/useSessions.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { useAPIClient } from './useAPIClient';

export function useSessions() {
  const client = useAPIClient();
  return useQuery({
    queryKey: ['sessions'],
    queryFn: () => client!.getSessions(),
    enabled: !!client,
    staleTime: 5000,
  });
}

export function useSession(id: string) {
  const client = useAPIClient();
  return useQuery({
    queryKey: ['session', id],
    queryFn: () => client!.getSession(id),
    enabled: !!client && !!id,
    staleTime: 5000,
  });
}

export function useStartSession() {
  const client = useAPIClient();
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => client!.startSession(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['sessions'] });
    },
  });
}
```

### Pattern 5: Optimistic Updates for Mutations

**What:** For fast UI feedback on rename, move, delete, use TanStack Query's optimistic update pattern. SSE will confirm or overwrite.

**When:** Any mutation that changes visible UI immediately (rename, move, tag edit).

### Anti-Patterns to Avoid

- **Direct SSE state mutations:** Never set session data from SSE events. Always invalidate and let queries refetch. This prevents dual-source-of-truth bugs.
- **Polling instead of SSE:** Do not poll `/api/sessions` on a timer. SSE handles real-time. Polling wastes battery.
- **Unbounded FlashList data:** For users with 100+ sessions, consider that the server returns all sessions in one call. FlashList handles rendering efficiently, but keep `staleTime` reasonable (5s) to avoid excessive refetches.
- **Missing enabled guard:** Always check `enabled: !!client` in queries. If the client is null (no active server), queries must not fire.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| SSE connection management | Custom fetch + streaming parser | react-native-sse | Handles reconnection, auth headers, event parsing |
| Session list performance | Custom virtualized list | FlashList v2 | Cell recycling, no size estimates needed, 60fps |
| Server data caching | Manual fetch + useState | TanStack Query | Stale-while-revalidate, cache invalidation, background refetch |
| Debounced search | Custom setTimeout wrapper | TanStack Query with `keepPreviousData` + lodash.debounce | Previous results shown while typing |
| Bottom sheet modals | Custom animated View | ModalSheet component (already built in Phase 1) | Gesture-based dismiss, snap points |
| Action sheets | Custom action menu | ActionSheet component (already built in Phase 1) | iOS-native feel, destructive action styling |

## Common Pitfalls

### Pitfall 1: SSE Breaks in Expo Dev Mode (CDP Interceptor)

**What goes wrong:** SSE connection opens (200 status) but no events are delivered during development.
**Why it happens:** Expo's ExpoRequestCdpInterceptor tries to peek the infinite SSE body stream, blocking it.
**How to avoid:** Test SSE exclusively in preview/production builds. Use `react-native-sse` (XMLHttpRequest-based, less susceptible). Do not waste time debugging SSE in dev mode.
**Warning signs:** `onmessage` never fires despite successful connection. Works in production.

### Pitfall 2: Stale API Client After Server Switch

**What goes wrong:** Queries return data from the wrong server after switching.
**Why it happens:** TanStack Query cache retains data keyed by `['sessions']` regardless of which server fetched it.
**How to avoid:** Clear the entire query cache on server switch: `queryClient.clear()`. The `switchServer` action in server-store should trigger this. Alternatively, prefix all query keys with the serverId: `[serverId, 'sessions']`.
**Warning signs:** Seeing old sessions after switching to a different server.

### Pitfall 3: FlashList estimatedItemSize Warning

**What goes wrong:** Console warning about missing `estimatedItemSize` or poor recycling performance.
**Why it happens:** FlashList v2 no longer requires `estimatedItemSize`, but if you pass it incorrectly (wrong value), recycling breaks.
**How to avoid:** Do NOT pass `estimatedItemSize` with FlashList v2. It auto-measures. If you see recycling issues, check that `keyExtractor` returns stable, unique IDs.
**Warning signs:** Items flash or show wrong data during fast scrolling.

### Pitfall 4: Search Input Triggering Too Many API Calls

**What goes wrong:** Every keystroke fires a search API call, overwhelming the server and causing jank.
**Why it happens:** Not debouncing the search input before triggering the query.
**How to avoid:** Debounce the search term (300ms) before passing it to the TanStack Query hook. Use `keepPreviousData: true` to show previous results while the new query loads.
**Warning signs:** Visible loading flicker on every keystroke in search.

### Pitfall 5: Missing Loading/Error/Empty States

**What goes wrong:** Screen shows blank content when data is loading, errors silently, or shows nothing for empty lists.
**Why it happens:** Only handling the success case in the query hook.
**How to avoid:** Every screen must handle: loading (Skeleton components), error (EmptyState with retry), empty (EmptyState with create CTA). Use TanStack Query's `isLoading`, `isError`, `data` states consistently.
**Warning signs:** Blank white screens, confused users.

### Pitfall 6: Session Card Re-render Performance

**What goes wrong:** Session list stutters when SSE events arrive frequently (e.g., running sessions update every few seconds).
**Why it happens:** SSE invalidates the sessions query, which refetches and causes all cards to re-render.
**How to avoid:** Memoize SessionCard with `React.memo` and use `keyExtractor` returning `session.id`. Keep card render cheap (no inline object creation in styles). FlashList v2's recycling helps, but memo is still critical.
**Warning signs:** Scroll jank during active sessions.

## Code Examples

### SessionCard Component Pattern

```typescript
// components/sessions/SessionCard.tsx
import React, { memo, useCallback } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';
import { useTheme } from '@/hooks/useTheme';
import { StatusDot, Badge, Card } from '@/components/ui';
import type { Session } from '@/types/api';

interface SessionCardProps {
  session: Session;
  onPress: (id: string) => void;
  onLongPress: (id: string) => void;
  showWorkspace?: boolean;
}

const SessionCard = memo<SessionCardProps>(({ session, onPress, onLongPress, showWorkspace }) => {
  const { theme } = useTheme();
  const handlePress = useCallback(() => onPress(session.id), [session.id, onPress]);
  const handleLongPress = useCallback(() => onLongPress(session.id), [session.id, onLongPress]);

  return (
    <Card pressable onPress={handlePress} onLongPress={handleLongPress}>
      <View style={styles.row}>
        <StatusDot status={session.status} animated={session.status === 'running'} />
        <View style={styles.content}>
          <Text style={[styles.name, { color: theme.colors.textPrimary }]}>{session.name}</Text>
          {session.topic ? (
            <Text style={[styles.topic, { color: theme.colors.textSecondary }]} numberOfLines={1}>
              {session.topic}
            </Text>
          ) : null}
        </View>
        {/* Cost badge, workspace badge, etc. */}
      </View>
    </Card>
  );
});
```

### FlashList Session List Pattern

```typescript
// In sessions/index.tsx
import { FlashList } from '@shopify/flash-list';

function SessionList({ sessions }: { sessions: Session[] }) {
  const renderItem = useCallback(({ item }: { item: Session }) => (
    <SessionCard
      session={item}
      onPress={handlePress}
      onLongPress={handleLongPress}
    />
  ), [handlePress, handleLongPress]);

  return (
    <FlashList
      data={sessions}
      renderItem={renderItem}
      keyExtractor={(item) => item.id}
      refreshing={isRefetching}
      onRefresh={refetch}
    />
  );
}
```

### Workspace Filter Chips Pattern

```typescript
// Horizontal scrollable filter chips at top of session list
function WorkspaceFilters({ workspaces, selected, onSelect }) {
  return (
    <ScrollView horizontal showsHorizontalScrollIndicator={false}>
      <Chip label="All" selected={!selected} onPress={() => onSelect(null)} />
      {workspaces.map((ws) => (
        <Chip
          key={ws.id}
          label={ws.name}
          selected={selected === ws.id}
          onPress={() => onSelect(ws.id)}
        />
      ))}
    </ScrollView>
  );
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| FlatList for long lists | FlashList v2 (auto-measures) | 2025 | No estimatedItemSize needed, better recycling |
| Manual SSE parsing with fetch | react-native-sse library | Stable | Handles reconnection, custom headers |
| Global state for server data | TanStack Query cache | Standard | Stale-while-revalidate, background refetch |
| Polling for real-time | SSE invalidation | Standard | Battery efficient, instant updates |
| AsyncStorage for UI prefs | MMKV (sync reads) | 2024+ | No flash, instant reads |

## Server Endpoint Reference (All Existing)

No new server code is needed for Phase 3. All endpoints already exist.

### Sessions
| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/sessions?mode=all | All sessions |
| GET | /api/sessions?mode=workspace&workspaceId=X | Sessions for workspace |
| GET | /api/sessions?mode=recent&count=N | Recently active |
| POST | /api/sessions | Create session |
| PUT | /api/sessions/:id | Update session (rename, move, tags, etc.) |
| DELETE | /api/sessions/:id | Delete session |
| POST | /api/sessions/:id/start | Start session |
| POST | /api/sessions/:id/stop | Stop session |
| POST | /api/sessions/:id/restart | Restart session |
| GET | /api/sessions/:id/cost | Session cost breakdown |
| GET | /api/sessions/:id/subagents | Subagent tree |
| POST | /api/sessions/:id/auto-title | AI auto-title |
| POST | /api/sessions/:id/summarize | AI summarize |

### Workspaces
| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/workspaces | All workspaces (includes sessions array) |
| GET | /api/workspaces/:id | Single workspace with session objects |
| POST | /api/workspaces | Create workspace |
| PUT | /api/workspaces/:id | Update workspace |
| DELETE | /api/workspaces/:id | Delete workspace |
| PUT | /api/workspaces/reorder | Reorder workspaces |

### Groups
| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/groups | All workspace groups |
| POST | /api/groups | Create group |
| PUT | /api/groups/:id | Update group |
| DELETE | /api/groups/:id | Delete group |
| POST | /api/groups/:id/add | Add workspace to group |

### Templates
| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/templates | All templates |
| POST | /api/templates | Create template |
| DELETE | /api/templates/:id | Delete template |

### Search
| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/search?q=X&limit=N | Keyword search across JSONL files |
| POST | /api/search-conversations | AI semantic search |

### Conflicts
| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/conflicts | Global file conflicts |

### SSE
| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/events?token=X | SSE stream (token as query param, not header) |

**SSE Events Emitted by Server:**
- `session:created`, `session:updated`, `session:deleted`, `session:log`
- `workspace:created`, `workspace:updated`, `workspace:deleted`, `workspace:activated`
- `group:created`, `group:updated`, `group:deleted`
- `template:created`, `template:deleted`
- `settings:updated`, `docs:updated`, `workspaces:reordered`

### Discovery/Browsing
| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/discover | Discover Claude sessions on disk |
| GET | /api/browse?path=X | Browse directories |

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Maestro (E2E) + Jest 29 (unit) |
| Config file | mobile/maestro/config.yaml (Maestro), mobile/package.json jest config (Jest) |
| Quick run command | `cd mobile && npx jest --passWithNoTests` |
| Full suite command | `cd mobile && maestro test maestro/flows/` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SESS-01 | Session list renders with real-time updates | E2E | `maestro test maestro/flows/sessions-list.yaml` | Wave 0 |
| SESS-04 | Session detail shows metadata + cost | E2E | `maestro test maestro/flows/session-detail.yaml` | Wave 0 |
| SESS-05 | Start/stop/restart session | E2E | `maestro test maestro/flows/session-controls.yaml` | Wave 0 |
| SESS-06 | Create new session | E2E | `maestro test maestro/flows/create-session.yaml` | Wave 0 |
| WORK-01 | Workspace list with counts | E2E | `maestro test maestro/flows/workspaces.yaml` | Wave 0 |
| SRCH-03 | Quick switcher opens and searches | E2E | `maestro test maestro/flows/quick-switcher.yaml` | Wave 0 |

### Sampling Rate
- **Per task commit:** `cd mobile && npx jest --passWithNoTests`
- **Per wave merge:** `cd mobile && maestro test maestro/flows/`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `mobile/maestro/flows/sessions-list.yaml` - covers SESS-01, SESS-02, SESS-14, SESS-16, SESS-17
- [ ] `mobile/maestro/flows/session-detail.yaml` - covers SESS-04, SESS-10, SESS-12, SESS-13
- [ ] `mobile/maestro/flows/session-controls.yaml` - covers SESS-05, SESS-07, SESS-08, SESS-09
- [ ] `mobile/maestro/flows/create-session.yaml` - covers SESS-06, SESS-11, SESS-15
- [ ] `mobile/maestro/flows/workspaces.yaml` - covers WORK-01 through WORK-07
- [ ] `mobile/maestro/flows/quick-switcher.yaml` - covers SRCH-01, SRCH-02, SRCH-03
- [ ] `mobile/maestro/flows/conflicts.yaml` - covers CNFL-01, CNFL-02

## Open Questions

1. **Needs-input detection (SESS-17)**
   - What we know: The server sets session status to 'idle' and logs contain activity hints. The web GUI detects needs-input from terminal output patterns.
   - What's unclear: Whether the SSE `session:updated` event carries enough info, or if we need to parse session logs client-side.
   - Recommendation: Start with session status field ('idle' as proxy for needs-input). If insufficient, check session logs for "Waiting for user" patterns. This is a UI-only concern.

2. **Bulk stop (SESS-18)**
   - What we know: No dedicated bulk-stop endpoint exists. Must call POST /api/sessions/:id/stop for each session.
   - What's unclear: Whether we need Promise.allSettled or sequential stops.
   - Recommendation: Use Promise.allSettled for parallel stops. Show progress indicator. Handle partial failures gracefully.

3. **Conflict badge freshness (CNFL-02)**
   - What we know: No SSE event for conflict detection. Conflicts are computed on-demand by the server.
   - What's unclear: How to keep the badge count current without polling.
   - Recommendation: Poll `/api/conflicts` on a 30s interval only when the More tab is visible. Also refetch on app foreground. Accept that conflict detection is near-real-time, not instant.

## Sources

### Primary (HIGH confidence)
- Server source code: `src/web/server.js` - all endpoint shapes, SSE event types, response formats
- Design doc: `docs/plans/2026-03-28-myrlin-mobile-design.md` - screen specs, component API, types
- Existing mobile code: `mobile/services/api-client.ts`, `mobile/stores/server-store.ts`, `mobile/types/api.ts`

### Secondary (MEDIUM confidence)
- [react-native-sse GitHub](https://github.com/binaryminds/react-native-sse) - SSE client API
- [Expo SSE CDP bug](https://github.com/expo/expo/issues/27526) - dev mode limitation
- [TanStack Query RN docs](https://tanstack.com/query/latest/docs/framework/react/react-native) - cache invalidation patterns

### Tertiary (LOW confidence)
- Needs-input detection heuristics (no formal spec, depends on Claude Code output patterns)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries already installed and verified in Phase 1-2
- Architecture: HIGH - patterns established in Phase 2, SSE pattern from design doc
- Server endpoints: HIGH - directly read from server source code
- Pitfalls: HIGH - SSE CDP bug confirmed, other pitfalls from community knowledge
- Needs-input detection: LOW - heuristic-based, no formal protocol

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (stable; no external dependencies changing)
