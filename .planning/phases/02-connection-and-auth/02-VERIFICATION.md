---
phase: 02-connection-and-auth
verified: 2026-03-29T01:15:00Z
status: passed
score: 5/5 must-haves verified
re_verification: false
---

# Phase 2: Connection and Auth Verification Report

**Phase Goal:** User can pair their phone with a Myrlin server via QR code or manual URL, authenticate, and switch between multiple connected servers
**Verified:** 2026-03-29T01:15:00Z
**Status:** passed
**Re-verification:** No, initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User can scan a QR code displayed on the desktop Myrlin web UI and connect to that server | VERIFIED | `scan-qr.tsx` uses expo-camera CameraView with barcodeScannerSettings for QR scanning. Parses JSON payload, calls `client.pair()`, stores server via `addServer()`, navigates to tabs. Server exposes `GET /api/auth/pairing-code` generating QR payload and `POST /api/auth/pair` exchanging pairing token for Bearer token. |
| 2 | User can manually enter a server URL and password to connect when QR is not available | VERIFIED | `manual-connect.tsx` has URL + password + optional name form, URL normalization, validation, calls `client.login()`, stores server on success. Keyboard avoidance and error handling present. |
| 3 | User can enable Face ID or fingerprint to lock the app, and the lock gate blocks all screens until authenticated | VERIFIED | `auth-store.ts` manages `biometricEnabled` (persisted to MMKV) and `isLocked` (transient). `BiometricGate.tsx` renders full-screen lock overlay with auto-prompt via `expo-local-authentication`, retry button, Reanimated fade-in. Root `_layout.tsx` renders `{biometricEnabled && isLocked && <BiometricGate />}` above all navigation. `useConnectionHealth.ts` locks on app background. |
| 4 | User can add multiple servers and switch between them; each server maintains its own auth token | VERIFIED | `server-store.ts` has `servers[]` array with `addServer`, `removeServer`, `switchServer`, `updateToken`, `logout` actions. Each `ServerConnection` has its own `token`, `url`, `name`. `ServerMenu.tsx` lists other servers with "Switch" action, calls `switchServer(id)`. Persisted to SecureStore. |
| 5 | Connection status indicator is visible on every screen showing connected or disconnected state | VERIFIED | `ConnectionDot.tsx` reads `connectionStatus` from server store, renders colored dot (green/yellow/red/peach) with Reanimated pulse for connecting state. Wired into `(tabs)/_layout.tsx` as `headerRight` on all tab screens. `useConnectionHealth` polls `/api/auth/check` with exponential backoff (5s to 60s). |

**Score:** 5/5 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `src/web/pairing.js` | Server pairing endpoints | VERIFIED | 175 lines. GET /api/auth/pairing-code (auth required), POST /api/auth/pair (public, rate-limited). Single-use tokens, 5-min TTL, in-memory Map. |
| `src/web/auth.js` | Auth exports for pairing | VERIFIED | 411 lines. Exports generateToken, addToken, isRateLimited used by pairing module. |
| `mobile/types/api.ts` | API response types | VERIFIED | 112 lines. ServerConnection, ConnectionStatus, PairingCodeResponse, PairResponse, LoginResponse, AuthCheckResponse, APIError. |
| `mobile/stores/secure-storage-adapter.ts` | SecureStore Zustand adapter | VERIFIED | 64 lines. Wraps expo-secure-store in Zustand StateStorage interface. |
| `mobile/stores/server-store.ts` | Multi-server state store | VERIFIED | 230 lines. Zustand + persist + SecureStore. Full CRUD: addServer, removeServer, switchServer, updateToken, logout. Hydration gate. |
| `mobile/stores/auth-store.ts` | Biometric lock store | VERIFIED | 140 lines. Zustand + MMKV. biometricEnabled (persisted), isLocked (transient). Hardware/enrollment checks. |
| `mobile/services/api-client.ts` | Typed HTTP client | VERIFIED | 163 lines. MyrlinAPIClient with checkAuth, login, logout, pair methods. APIError handling. |
| `mobile/app/(auth)/_layout.tsx` | Auth stack navigator | VERIFIED | 36 lines. Headerless Stack with 4 screens. |
| `mobile/app/(auth)/onboarding.tsx` | Welcome screen | VERIFIED | 136 lines. Myrlin branding, "Scan QR" and "Connect Manually" buttons. |
| `mobile/app/(auth)/scan-qr.tsx` | QR scanner screen | VERIFIED | 407 lines. CameraView with barcodeScannerSettings, viewfinder overlay, permission handling, error display. |
| `mobile/app/(auth)/manual-connect.tsx` | Manual connect form | VERIFIED | 291 lines. URL + password + name inputs, validation, keyboard avoidance. |
| `mobile/app/(auth)/login.tsx` | Token-refresh login | VERIFIED | 225 lines. Password re-entry for expired servers with route params. |
| `mobile/app/_layout.tsx` | Root layout with auth gate | VERIFIED | 120 lines. Hydration gate, conditional auth vs tabs Stack, BiometricGate overlay. |
| `mobile/components/BiometricGate.tsx` | Lock overlay | VERIFIED | 178 lines. Full-screen lock, auto-prompt, retry button, Reanimated animation. |
| `mobile/components/ConnectionDot.tsx` | Status indicator | VERIFIED | 157 lines. Color-coded dot with pulse animation for connecting state. |
| `mobile/components/ServerMenu.tsx` | Server management menu | VERIFIED | 384 lines. Active server info, switch list, add server, disconnect with server-side logout. |
| `mobile/hooks/useConnectionHealth.ts` | Auto-reconnect hook | VERIFIED | 149 lines. Exponential backoff polling, AppState lifecycle, biometric lock on background. |
| `mobile/app/(tabs)/_layout.tsx` | Tab layout with dot wiring | VERIFIED | 132 lines. Calls useConnectionHealth(), renders ConnectionDot + ServerMenu in header. |
| `test/pairing.test.js` | Pairing integration tests | VERIFIED | 272 lines. 24 tests covering pairing flow. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `pairing.js` | `auth.js` | Dependency injection (requireAuth, addToken, generateToken, isRateLimited) | WIRED | `setupPairing(app, { requireAuth, addToken, generateToken, isRateLimited })` called in server.js line 276 |
| `scan-qr.tsx` | `api-client.ts` | `createAPIClient().pair()` | WIRED | Creates client, calls pair(), stores result via addServer() |
| `manual-connect.tsx` | `api-client.ts` | `createAPIClient().login()` | WIRED | Creates client, calls login(), stores result via addServer() |
| `login.tsx` | `api-client.ts` | `createAPIClient().login()` | WIRED | Creates client, calls login(), updates token via updateToken() |
| `scan-qr.tsx` | `server-store.ts` | `useServerStore().addServer()` | WIRED | Imports and calls addServer on successful pairing |
| `manual-connect.tsx` | `server-store.ts` | `useServerStore().addServer()` | WIRED | Imports and calls addServer on successful login |
| `_layout.tsx (root)` | `server-store.ts` | Conditional rendering based on activeServer | WIRED | `getActiveServer()` determines auth vs tabs Stack |
| `_layout.tsx (root)` | `auth-store.ts` | BiometricGate rendering | WIRED | `biometricEnabled && isLocked && <BiometricGate />` |
| `_layout.tsx (tabs)` | `useConnectionHealth` | Hook invocation | WIRED | `useConnectionHealth()` called at top of TabLayout |
| `_layout.tsx (tabs)` | `ConnectionDot` | headerRight rendering | WIRED | ConnectionDot rendered in `screenOptions.headerRight` |
| `_layout.tsx (tabs)` | `ServerMenu` | State-driven modal | WIRED | `<ServerMenu visible={serverMenuVisible} onClose={...} />` |
| `useConnectionHealth.ts` | `api-client.ts` | `createAPIClient().checkAuth()` | WIRED | Polls checkAuth with exponential backoff |
| `ServerMenu.tsx` | `server-store.ts` | `switchServer`, `removeServer` | WIRED | Calls both actions for server management |
| `ServerMenu.tsx` | `api-client.ts` | `createAPIClient().logout()` | WIRED | Best-effort server-side token invalidation on disconnect |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| CONN-01 | 02-02 | Scan QR code from desktop UI to pair | SATISFIED | `scan-qr.tsx` with CameraView, QR JSON parsing, pair endpoint call |
| CONN-02 | 02-02 | Manually enter URL and password | SATISFIED | `manual-connect.tsx` with URL validation, login flow |
| CONN-03 | 02-01 | Auth token in expo-secure-store | SATISFIED | `secure-storage-adapter.ts` wraps SecureStore, server-store uses it |
| CONN-04 | 02-03 | Auto-reconnect on connection drop | SATISFIED | `useConnectionHealth.ts` with exponential backoff, AppState transitions |
| CONN-05 | 02-01, 02-03 | Connect to multiple servers | SATISFIED | `server-store.ts` has `servers[]` array, addServer/removeServer |
| CONN-06 | 02-03 | Switch between connected servers | SATISFIED | `ServerMenu.tsx` lists other servers, calls `switchServer()` |
| CONN-07 | 02-03 | Connection status visible at all times | SATISFIED | `ConnectionDot.tsx` in all tab headers via `(tabs)/_layout.tsx` |
| CONN-08 | 02-02 | Onboarding flow guides first-time user | SATISFIED | `onboarding.tsx` with "Scan QR" and "Connect Manually" actions |
| AUTH-01 | 02-01, 02-02 | Log in with password on known server | SATISFIED | `login.tsx` for re-auth, `manual-connect.tsx` for first connect |
| AUTH-02 | 02-03 | Biometric app lock (Face ID / fingerprint) | SATISFIED | `auth-store.ts` + `BiometricGate.tsx` + root layout wiring |
| AUTH-03 | 02-01, 02-03 | Log out from any screen | SATISFIED | `ServerMenu.tsx` disconnect calls logout() then removeServer() |
| AUTH-04 | 02-01 | Token persists across app restarts | SATISFIED | SecureStore persistence in server-store, hydration gate in root layout |
| SRVR-01 | 02-01 | GET /api/auth/pairing-code endpoint | SATISFIED | `pairing.js` line 76, auth-required, generates QR payload |
| SRVR-02 | 02-01 | POST /api/auth/pair endpoint | SATISFIED | `pairing.js` line 105, public with rate limiting, exchanges pairing token for Bearer |

**All 14 requirements accounted for. No orphaned requirements.**

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| - | - | None found in Phase 2 artifacts | - | - |

No TODO, FIXME, HACK, or placeholder patterns found in Phase 2 files. The "placeholder" matches in tab screens (sessions, tasks, costs, docs, more) are Phase 1 scaffolds expected to be built in later phases.

### Human Verification Required

### 1. QR Scanning End-to-End

**Test:** Start Myrlin server, open web UI pairing dialog, scan QR with mobile app
**Expected:** Phone pairs successfully, navigates to sessions tab, ConnectionDot shows green
**Why human:** Requires running server, physical camera, and device interaction

### 2. Biometric Lock Flow

**Test:** Enable biometric in settings, background app, reopen
**Expected:** BiometricGate overlay appears, Face ID/fingerprint prompt fires, unlock dismisses gate
**Why human:** Requires physical biometric hardware on device

### 3. Multi-Server Switching

**Test:** Pair two different servers, use ServerMenu to switch between them
**Expected:** Active server changes, ConnectionDot reconnects, data refreshes for new server
**Why human:** Requires two running Myrlin server instances

### 4. Connection Recovery

**Test:** Connect to server, stop server process, wait 10s, restart server
**Expected:** Dot turns red within 10s, auto-recovers to green after server restart
**Why human:** Requires server process control and timing observation

### Gaps Summary

No gaps found. All 5 observable truths verified, all 14 requirements satisfied, all artifacts are substantive and correctly wired. Server-side pairing endpoints are integrated into the Express server. Mobile auth flow (onboarding, QR scan, manual connect, login) is fully wired through the root layout auth gate. Multi-server management, biometric lock, connection health polling, and status indicators are all implemented and connected.

---

_Verified: 2026-03-29T01:15:00Z_
_Verifier: Claude (gsd-verifier)_
