# Phase 2: Connection and Auth - Research

**Researched:** 2026-03-28
**Domain:** QR pairing, multi-server auth, biometric app lock, server endpoint additions
**Confidence:** HIGH

## Summary

Phase 2 connects the Myrlin Mobile app to one or more Myrlin desktop servers. The work spans two codebases: the existing Express server (adding pairing endpoints) and the React Native mobile app (QR scanning, auth flows, biometric lock, multi-server management). The existing server auth system is straightforward (in-memory token set, timing-safe password comparison, Bearer tokens) and the new pairing endpoints follow the same pattern with the addition of short-lived single-use pairing tokens.

The mobile side requires expo-camera for QR scanning, expo-secure-store for encrypted token storage, expo-local-authentication for biometric gating, and a Zustand server-store that persists connection metadata to SecureStore. The Phase 1 foundation (Expo SDK 55 canary, Zustand 5, MMKV, Reanimated 4, typed routes) provides all the building blocks. No new native dependencies are required beyond the Expo SDK bundled packages.

**Primary recommendation:** Build server-side pairing endpoints first (they are testable with curl), then build the mobile connection flow against real endpoints. Do not mock the server.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| CONN-01 | Scan QR code from desktop Myrlin UI to pair | expo-camera CameraView with barcodeScannerSettings, QR payload format from design doc, SRVR-01/SRVR-02 endpoints |
| CONN-02 | Manually enter server URL and password to connect | Standard form screen, calls existing POST /api/auth/login, stores token in SecureStore |
| CONN-03 | Auth token stored securely in expo-secure-store | SecureStore Zustand adapter pattern (see Code Examples), Keychain/Keystore encryption |
| CONN-04 | Auto-reconnect when connection drops | AppState listener + periodic /api/auth/check polling, SSE reconnection pattern |
| CONN-05 | Connect to multiple servers | ServerConnection[] in SecureStore, server-store Zustand with persist |
| CONN-06 | Switch between connected servers | server-store.switchServer() clears TanStack Query cache, swaps api-client instance |
| CONN-07 | Connection status visible at all times | Zustand connectionStatus field, ConnectionDot component in tab bar or header |
| CONN-08 | Onboarding flow guides first-time user | Conditional routing: no servers = onboarding screen, uses expo-router redirect |
| AUTH-01 | Log in with password on known server | Calls POST /api/auth/login on stored server URL, receives Bearer token |
| AUTH-02 | Biometric app lock (Face ID / fingerprint) | expo-local-authentication, lock gate wraps root layout, MMKV stores lock preference |
| AUTH-03 | Log out from any screen | Calls POST /api/auth/logout, removes token from SecureStore, clears query cache |
| AUTH-04 | Session token persists across app restarts | SecureStore persistence via server-store Zustand persist middleware |
| SRVR-01 | GET /api/auth/pairing-code | New endpoint in src/web/pairing.js, generates short-lived pairing token + QR payload |
| SRVR-02 | POST /api/auth/pair | New endpoint in src/web/pairing.js, exchanges pairing token for long-lived Bearer token |
</phase_requirements>

## Standard Stack

### Core (Phase 2 additions)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| expo-camera | ~16.x (SDK 55 bundled) | QR code scanning | CameraView with built-in barcode scanning; replaces deprecated expo-barcode-scanner |
| expo-secure-store | ~14.x (SDK 55 bundled) | Encrypted token storage | Keychain (iOS) / Keystore (Android). The only correct place for auth tokens. |
| expo-local-authentication | ~14.x (SDK 55 bundled) | Biometric auth (Face ID, fingerprint) | Bundled with Expo. Falls back to device passcode. |
| react-native-qrcode-svg | 6.x | QR code generation (server-side desktop UI) | SVG-based QR rendering for the desktop web pairing modal |
| qrcode | 1.x | QR code generation (Node.js server) | Generates QR data URL for the pairing-code endpoint |

### Already Installed (from Phase 1)

| Library | Version | Used For in Phase 2 |
|---------|---------|---------------------|
| zustand | 5.0.x | server-store, auth-store with persist middleware |
| react-native-mmkv | 4.3.x | Biometric lock preference, onboarding state |
| expo-router | 55.x canary | Auth group routing, onboarding redirect |
| react-native-reanimated | 4.2.x | Screen transitions, connection status animations |

### Not Needed Yet

| Library | Deferred Until |
|---------|---------------|
| @tanstack/react-query | Phase 3 (server data fetching). Phase 2 only does auth requests, no cached queries needed. |
| react-native-sse | Phase 3 (real-time session updates). Phase 2 tests connectivity but does not subscribe to SSE. |

**Installation (mobile):**
```bash
cd mobile
npx expo install expo-camera expo-secure-store expo-local-authentication
```

**Installation (server, for QR generation):**
```bash
npm install qrcode
```

## Architecture Patterns

### Recommended File Structure (Phase 2 additions)

```
mobile/
  app/
    (auth)/                     # NEW: unauthenticated route group
      _layout.tsx               # Stack layout for auth screens
      onboarding.tsx            # First-launch screen (CONN-08)
      scan-qr.tsx               # QR camera scanner (CONN-01)
      manual-connect.tsx        # URL + password form (CONN-02)
      login.tsx                 # Password entry for known server (AUTH-01)
    (tabs)/                     # Existing, gated behind auth
      ...
    _layout.tsx                 # MODIFIED: add auth gate + biometric lock
  services/                     # NEW directory
    api-client.ts               # HTTP client with typed methods
  stores/
    server-store.ts             # NEW: multi-server state (CONN-05, CONN-06)
    auth-store.ts               # NEW: biometric lock state (AUTH-02)
  components/
    ConnectionDot.tsx           # NEW: status indicator (CONN-07)
    BiometricGate.tsx           # NEW: lock screen overlay (AUTH-02)

src/web/                        # Existing server
  pairing.js                    # NEW: pairing endpoints (SRVR-01, SRVR-02)
  server.js                     # MODIFIED: mount pairing routes
```

### Pattern 1: Auth-Gated Root Layout

**What:** The root `_layout.tsx` checks for an active server with a valid token. If none exists, it redirects to `(auth)/onboarding`. If biometric lock is enabled and the app just foregrounded, it shows the BiometricGate overlay.

**When:** Every app launch and every foreground resume.

**Example:**

```typescript
// app/_layout.tsx (modified)
import { Redirect } from 'expo-router';
import { useServerStore } from '@/stores/server-store';
import { useAuthStore } from '@/stores/auth-store';
import { BiometricGate } from '@/components/BiometricGate';

export default function RootLayout() {
  const activeServer = useServerStore((s) => s.activeServer);
  const isLocked = useAuthStore((s) => s.isLocked);

  // No servers paired yet, go to onboarding
  if (!activeServer) {
    return <Redirect href="/(auth)/onboarding" />;
  }

  return (
    <>
      {isLocked && <BiometricGate />}
      <RootLayoutNav />
    </>
  );
}
```

### Pattern 2: SecureStore Zustand Adapter

**What:** A custom Zustand persist storage adapter that wraps expo-secure-store. Since SecureStore is async, the adapter uses `createJSONStorage` with async getItem/setItem.

**When:** Persisting server connections and auth tokens.

**Example:**

```typescript
// stores/secure-storage-adapter.ts
import * as SecureStore from 'expo-secure-store';
import { createJSONStorage, type StateStorage } from 'zustand/middleware';

const secureStorage: StateStorage = {
  getItem: async (name: string): Promise<string | null> => {
    return await SecureStore.getItemAsync(name);
  },
  setItem: async (name: string, value: string): Promise<void> => {
    await SecureStore.setItemAsync(name, value);
  },
  removeItem: async (name: string): Promise<void> => {
    await SecureStore.deleteItemAsync(name);
  },
};

export const createSecureStorage = () => createJSONStorage(() => secureStorage);
```

### Pattern 3: Pairing Token Flow (Server-Side)

**What:** Two new endpoints that follow the existing auth.js pattern. GET /api/auth/pairing-code generates a short-lived token (5 min TTL). POST /api/auth/pair exchanges it for a long-lived Bearer token.

**When:** QR pairing flow.

**Key design decisions:**
- Pairing tokens stored in-memory Map (same pattern as startupTokens in auth.js)
- Pairing tokens are single-use and expire after 5 minutes
- The QR payload is a JSON string: `{ url, pairingToken, serverName, version }`
- POST /api/auth/pair is rate-limited (same as login)
- The Bearer token returned is identical to login tokens (added to activeTokens Set)

**Example:**

```javascript
// src/web/pairing.js
const crypto = require('crypto');
const { activeTokens, generateToken } = require('./auth-internals');

const PAIRING_TTL_MS = 5 * 60 * 1000; // 5 minutes
const pairingTokens = new Map();

function setupPairing(app, serverName, serverUrl) {
  // GET /api/auth/pairing-code - requires auth (only logged-in desktop users)
  app.get('/api/auth/pairing-code', requireAuth, (req, res) => {
    const pairingToken = crypto.randomBytes(32).toString('hex');
    const expiresAt = new Date(Date.now() + PAIRING_TTL_MS).toISOString();
    pairingTokens.set(pairingToken, { createdAt: Date.now(), used: false });

    const qrPayload = JSON.stringify({
      url: serverUrl,
      pairingToken,
      serverName,
      version: require('../../package.json').version,
    });

    res.json({ pairingToken, expiresAt, qrPayload });
  });

  // POST /api/auth/pair - public (mobile calls this)
  app.post('/api/auth/pair', (req, res) => {
    const { pairingToken, deviceName, platform } = req.body || {};
    const entry = pairingTokens.get(pairingToken);
    if (!entry || entry.used || Date.now() - entry.createdAt > PAIRING_TTL_MS) {
      return res.status(403).json({ success: false, error: 'Invalid or expired pairing token.' });
    }
    pairingTokens.delete(pairingToken);
    const token = generateToken();
    activeTokens.add(token);
    res.json({ success: true, token, serverName });
  });
}
```

### Pattern 4: Connection Health Check

**What:** The mobile app periodically calls GET /api/auth/check on the active server to verify connectivity. On failure, it sets connectionStatus to 'disconnected' and starts exponential backoff reconnection.

**When:** App is in foreground with an active server.

**Example:**

```typescript
// hooks/useConnectionHealth.ts
function useConnectionHealth() {
  const { activeServer, setConnectionStatus } = useServerStore();

  useEffect(() => {
    if (!activeServer) return;
    let timeout: NodeJS.Timeout;
    let backoff = 5000; // Start at 5s

    async function check() {
      try {
        const res = await fetch(`${activeServer.url}/api/auth/check`, {
          headers: { Authorization: `Bearer ${activeServer.token}` },
        });
        const { authenticated } = await res.json();
        setConnectionStatus(authenticated ? 'connected' : 'token-expired');
        backoff = 5000; // Reset on success
      } catch {
        setConnectionStatus('disconnected');
        backoff = Math.min(backoff * 2, 60000); // Max 60s
      }
      timeout = setTimeout(check, backoff);
    }

    check();
    return () => clearTimeout(timeout);
  }, [activeServer?.id]);
}
```

### Anti-Patterns to Avoid

- **Storing tokens in MMKV:** MMKV is not encrypted. Auth tokens MUST go in expo-secure-store. Only non-sensitive preferences (biometric lock enabled, onboarding completed) go in MMKV.
- **Hardcoding server URL:** Every API call must route through the active server from server-store. Never import a constant URL.
- **Skipping rate limiting on pairing:** POST /api/auth/pair is public. It must be rate-limited to prevent brute-force attempts on pairing tokens.
- **Blocking UI on SecureStore reads:** SecureStore is async. The Zustand store hydrates asynchronously. Show splash screen until hydration completes. Do not render auth-dependent screens before hydration.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| QR code scanning | Custom camera + image processing | expo-camera CameraView with barcodeScannerSettings | expo-camera uses platform-native QR detection (Google Code Scanner on Android, VisionKit on iOS). Faster and more reliable than any JS solution. |
| Encrypted storage | Custom encryption + AsyncStorage | expo-secure-store | Keychain/Keystore integration is complex and platform-specific. SecureStore handles it correctly. |
| Biometric prompt | Custom native module | expo-local-authentication | Platform biometric APIs differ significantly. Expo abstracts Face ID, Touch ID, fingerprint, and passcode fallback. |
| QR code generation | Canvas-based QR renderer | qrcode npm package (server) | Battle-tested, generates data URLs. The desktop web UI just displays an `<img>` tag. |
| Token refresh/rotation | Custom token rotation scheme | Simple re-login on 401 | The server uses stateless in-memory tokens. Server restarts invalidate all tokens. The mobile app should detect 401, prompt re-login, and exchange password for a new token. No refresh token complexity needed. |

**Key insight:** The existing server auth is intentionally simple (local dev tool, not a production SaaS). The mobile auth should match this simplicity. No OAuth, no refresh tokens, no JWT. Just Bearer tokens in an in-memory Set.

## Common Pitfalls

### Pitfall 1: SecureStore Hydration Race Condition

**What goes wrong:** The app renders and tries to read the active server before the Zustand SecureStore adapter has finished async hydration. The app thinks there are no servers and redirects to onboarding.
**Why it happens:** SecureStore is async, unlike MMKV. The persist middleware hydrates asynchronously.
**How to avoid:** Use Zustand's `onRehydrateStorage` callback or a `_hasHydrated` flag. Gate the root layout behind hydration completion. Show splash screen until hydrated.
**Warning signs:** App briefly shows onboarding screen then snaps to the main screen on every launch.

### Pitfall 2: Camera Permission Denied with No Fallback

**What goes wrong:** User denies camera permission, app shows blank screen or crashes.
**Why it happens:** expo-camera requires explicit camera permission. If denied, `CameraView` renders nothing.
**How to avoid:** Always check permission status before showing camera. If denied, show the manual connect form as fallback with a "Grant Camera Permission" button that opens system settings.
**Warning signs:** QR scanning screen is blank with no error message.

### Pitfall 3: QR Scanning Fires Multiple Times

**What goes wrong:** `onBarcodeScanned` callback fires rapidly (multiple times per second) for the same QR code, causing multiple pairing requests.
**Why it happens:** The camera continuously scans and reports detected barcodes every frame.
**How to avoid:** Set a `scanned` state flag. Once a QR is detected, set `scanned = true` and stop processing. Reset only if the user explicitly taps "Scan Again".
**Warning signs:** Multiple POST /api/auth/pair requests in server logs for one scan action.

### Pitfall 4: Server URL Validation

**What goes wrong:** User enters an invalid URL (missing protocol, trailing slash, localhost vs IP), and the app crashes or silently fails.
**Why it happens:** Manual URL entry is freeform text.
**How to avoid:** Normalize the URL: strip trailing slashes, default to http:// if no protocol, validate with URL constructor, test connectivity before saving. Show clear error messages for each failure mode.
**Warning signs:** "Network request failed" errors with no user-facing message.

### Pitfall 5: Biometric Auth on Simulator

**What goes wrong:** Face ID does not work in Expo Go or on simulators without enrolled biometrics.
**Why it happens:** expo-local-authentication requires a development build for Face ID on iOS. Simulator requires enrolled face via Features > Face ID > Enrolled.
**How to avoid:** Check `hasHardwareAsync()` and `isEnrolledAsync()` before offering biometric lock. Gracefully degrade to no biometric lock if unavailable. Test on a physical device for Face ID validation.
**Warning signs:** `authenticateAsync()` returns `{ success: false, error: 'not_enrolled' }`.

### Pitfall 6: Auth Module Circular Dependency

**What goes wrong:** The new pairing.js needs to add tokens to `activeTokens` from auth.js, but auth.js does not export `activeTokens` or `generateToken`.
**Why it happens:** auth.js was designed as a self-contained module. The `activeTokens` Set and `generateToken` function are internal.
**How to avoid:** Export `generateToken` and either export `activeTokens` or expose an `addToken(token)` function from auth.js. Keep the pairing module as a thin layer that delegates token management to auth.js.
**Warning signs:** Pairing returns a token that does not work for authenticated requests.

## Code Examples

### QR Scanner Screen (Mobile)

```typescript
// app/(auth)/scan-qr.tsx
import { CameraView, useCameraPermissions } from 'expo-camera';
import { useState } from 'react';
import { useServerStore } from '@/stores/server-store';
import { router } from 'expo-router';

export default function ScanQRScreen() {
  const [permission, requestPermission] = useCameraPermissions();
  const [scanned, setScanned] = useState(false);
  const addServer = useServerStore((s) => s.addServer);

  const handleBarCodeScanned = async ({ data }: { data: string }) => {
    if (scanned) return;
    setScanned(true);

    try {
      const payload = JSON.parse(data);
      const { url, pairingToken, serverName, version } = payload;

      const res = await fetch(`${url}/api/auth/pair`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          pairingToken,
          deviceName: 'iPhone', // or Device.modelName
          platform: 'ios',
        }),
      });

      const result = await res.json();
      if (result.success) {
        await addServer({
          name: serverName,
          url,
          token: result.token,
          tunnelType: 'lan',
        });
        router.replace('/(tabs)/sessions');
      }
    } catch {
      setScanned(false); // Allow retry
    }
  };

  if (!permission?.granted) {
    // Show permission request UI
    return <PermissionPrompt onRequest={requestPermission} />;
  }

  return (
    <CameraView
      style={{ flex: 1 }}
      barcodeScannerSettings={{ barcodeTypes: ['qr'] }}
      onBarcodeScanned={scanned ? undefined : handleBarCodeScanned}
    />
  );
}
```

### Server Store (Mobile)

```typescript
// stores/server-store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { createSecureStorage } from './secure-storage-adapter';

type ConnectionStatus = 'connected' | 'disconnected' | 'connecting' | 'token-expired';

interface ServerConnection {
  id: string;
  name: string;
  url: string;
  token: string;
  pairedAt: string;
  lastConnected: string;
  tunnelType: 'lan' | 'cloudflare' | 'tailscale' | 'relay';
}

interface ServerStore {
  servers: ServerConnection[];
  activeServerId: string | null;
  connectionStatus: ConnectionStatus;
  activeServer: ServerConnection | null; // derived
  addServer: (server: Omit<ServerConnection, 'id' | 'pairedAt' | 'lastConnected'>) => Promise<void>;
  removeServer: (id: string) => void;
  switchServer: (id: string) => void;
  setConnectionStatus: (status: ConnectionStatus) => void;
  updateToken: (serverId: string, token: string) => void;
  logout: (serverId: string) => void;
}

export const useServerStore = create<ServerStore>()(
  persist(
    (set, get) => ({
      servers: [],
      activeServerId: null,
      connectionStatus: 'disconnected',
      get activeServer() {
        const { servers, activeServerId } = get();
        return servers.find((s) => s.id === activeServerId) ?? null;
      },
      addServer: async (serverData) => {
        const id = crypto.randomUUID();
        const server: ServerConnection = {
          ...serverData,
          id,
          pairedAt: new Date().toISOString(),
          lastConnected: new Date().toISOString(),
        };
        set((state) => ({
          servers: [...state.servers, server],
          activeServerId: id,
          connectionStatus: 'connected',
        }));
      },
      removeServer: (id) =>
        set((state) => {
          const servers = state.servers.filter((s) => s.id !== id);
          const activeServerId =
            state.activeServerId === id
              ? servers[0]?.id ?? null
              : state.activeServerId;
          return { servers, activeServerId };
        }),
      switchServer: (id) =>
        set({ activeServerId: id, connectionStatus: 'connecting' }),
      setConnectionStatus: (status) => set({ connectionStatus: status }),
      updateToken: (serverId, token) =>
        set((state) => ({
          servers: state.servers.map((s) =>
            s.id === serverId ? { ...s, token } : s
          ),
        })),
      logout: (serverId) =>
        set((state) => ({
          servers: state.servers.map((s) =>
            s.id === serverId ? { ...s, token: '' } : s
          ),
          connectionStatus:
            state.activeServerId === serverId ? 'disconnected' : state.connectionStatus,
        })),
    }),
    {
      name: 'myrlin-servers',
      storage: createSecureStorage(),
      partialize: (state) => ({
        servers: state.servers,
        activeServerId: state.activeServerId,
      }),
    }
  )
);
```

### Biometric Gate (Mobile)

```typescript
// components/BiometricGate.tsx
import * as LocalAuthentication from 'expo-local-authentication';
import { useEffect } from 'react';
import { View, Text, Pressable } from 'react-native';
import { useAuthStore } from '@/stores/auth-store';
import { useTheme } from '@/hooks/useTheme';

export function BiometricGate() {
  const { unlock } = useAuthStore();
  const theme = useTheme();

  const authenticate = async () => {
    const result = await LocalAuthentication.authenticateAsync({
      promptMessage: 'Unlock Myrlin',
      fallbackLabel: 'Use Passcode',
      disableDeviceFallback: false,
    });
    if (result.success) {
      unlock();
    }
  };

  useEffect(() => {
    authenticate();
  }, []);

  return (
    <View style={{
      ...StyleSheet.absoluteFillObject,
      backgroundColor: theme.colors.base,
      justifyContent: 'center',
      alignItems: 'center',
      zIndex: 9999,
    }}>
      <Text style={{ color: theme.colors.text, fontSize: 18 }}>
        Myrlin is locked
      </Text>
      <Pressable onPress={authenticate}>
        <Text style={{ color: theme.colors.blue, marginTop: 16 }}>
          Tap to unlock
        </Text>
      </Pressable>
    </View>
  );
}
```

### Server Pairing Endpoints (Express)

```javascript
// src/web/pairing.js
const crypto = require('crypto');

const PAIRING_TTL_MS = 5 * 60 * 1000;
const pairingTokens = new Map();

// Cleanup expired tokens every 2 minutes
setInterval(() => {
  const now = Date.now();
  for (const [token, entry] of pairingTokens) {
    if (entry.used || now - entry.createdAt > PAIRING_TTL_MS) {
      pairingTokens.delete(token);
    }
  }
}, 2 * 60 * 1000).unref();

function setupPairing(app, { requireAuth, addToken, generateToken, isRateLimited }) {
  app.get('/api/auth/pairing-code', requireAuth, (req, res) => {
    const pairingToken = crypto.randomBytes(32).toString('hex');
    pairingTokens.set(pairingToken, { createdAt: Date.now(), used: false });

    const pkg = require('../../package.json');
    const serverUrl = req.query.url || `${req.protocol}://${req.get('host')}`;
    const qrPayload = JSON.stringify({
      url: serverUrl,
      pairingToken,
      serverName: pkg.name,
      version: pkg.version,
    });

    res.json({
      pairingToken,
      expiresAt: new Date(Date.now() + PAIRING_TTL_MS).toISOString(),
      qrPayload,
    });
  });

  app.post('/api/auth/pair', (req, res) => {
    const clientIp = req.ip || req.connection.remoteAddress || 'unknown';
    if (isRateLimited(clientIp)) {
      return res.status(429).json({
        success: false,
        error: 'Too many pairing attempts. Try again in 1 minute.',
      });
    }

    const { pairingToken, deviceName, platform } = req.body || {};
    if (!pairingToken || typeof pairingToken !== 'string') {
      return res.status(400).json({ success: false, error: 'Missing pairingToken.' });
    }

    const entry = pairingTokens.get(pairingToken);
    if (!entry) {
      return res.status(403).json({ success: false, error: 'Invalid pairing token.' });
    }
    if (entry.used) {
      return res.status(403).json({ success: false, error: 'Pairing token already used.' });
    }
    if (Date.now() - entry.createdAt > PAIRING_TTL_MS) {
      pairingTokens.delete(pairingToken);
      return res.status(403).json({ success: false, error: 'Pairing token expired.' });
    }

    pairingTokens.delete(pairingToken);
    const token = generateToken();
    addToken(token);

    const pkg = require('../../package.json');
    res.json({
      success: true,
      token,
      serverName: pkg.name,
      serverVersion: pkg.version,
    });
  });
}

module.exports = { setupPairing };
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| expo-barcode-scanner | expo-camera CameraView barcodeScannerSettings | SDK 51 (2024) | expo-barcode-scanner removed. Must use expo-camera. |
| AsyncStorage for tokens | expo-secure-store | Always best practice | AsyncStorage is not encrypted. Tokens must use SecureStore. |
| Single server assumption | Multi-server architecture | Design doc decision | All API calls must go through active server URL. No hardcoded endpoints. |
| OAuth/JWT auth | Simple Bearer tokens | Existing server design | Server uses in-memory token Set. No JWT decoding, no refresh tokens. Keep it simple. |

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Maestro (E2E, YAML flows) + Jest 29.x (unit) |
| Config file | mobile/maestro/ (E2E), jest.config.js (unit) |
| Quick run command | `cd mobile && npx jest --testPathPattern=stores` |
| Full suite command | `maestro test mobile/maestro/` |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| CONN-01 | QR scan pairs device | E2E (manual, needs camera) | Manual on device | N/A |
| CONN-02 | Manual URL connect | E2E | `maestro test mobile/maestro/manual-connect.yaml` | Wave 0 |
| CONN-03 | Token in SecureStore | unit | `npx jest stores/server-store.test.ts` | Wave 0 |
| CONN-04 | Auto-reconnect | unit | `npx jest hooks/useConnectionHealth.test.ts` | Wave 0 |
| CONN-05 | Multiple servers | unit | `npx jest stores/server-store.test.ts` | Wave 0 |
| CONN-06 | Switch servers | unit | `npx jest stores/server-store.test.ts` | Wave 0 |
| CONN-07 | Status indicator | E2E | `maestro test mobile/maestro/connection-status.yaml` | Wave 0 |
| CONN-08 | Onboarding flow | E2E | `maestro test mobile/maestro/onboarding.yaml` | Wave 0 |
| AUTH-01 | Password login | integration | `curl -X POST localhost:3456/api/auth/login` | Existing |
| AUTH-02 | Biometric lock | manual-only | Manual on device (biometric cannot be automated) | N/A |
| AUTH-03 | Logout | unit | `npx jest stores/server-store.test.ts` | Wave 0 |
| AUTH-04 | Token persists | unit | `npx jest stores/server-store.test.ts` | Wave 0 |
| SRVR-01 | GET pairing-code | integration | `npm test -- --testPathPattern=pairing` | Wave 0 |
| SRVR-02 | POST pair | integration | `npm test -- --testPathPattern=pairing` | Wave 0 |

### Sampling Rate

- **Per task commit:** `npx jest --testPathPattern=<relevant-test>` (server) or `cd mobile && npx jest stores/` (mobile)
- **Per wave merge:** Full Jest suite + Maestro flows on simulator
- **Phase gate:** All unit/integration tests green, Maestro onboarding flow passes

### Wave 0 Gaps

- [ ] `test/pairing.test.js` - Server pairing endpoint tests (SRVR-01, SRVR-02)
- [ ] `mobile/stores/__tests__/server-store.test.ts` - Server store unit tests
- [ ] `mobile/maestro/onboarding.yaml` - First-launch onboarding flow
- [ ] `mobile/maestro/manual-connect.yaml` - Manual connect flow

## Open Questions

1. **Server URL for QR payload**
   - What we know: GET /api/auth/pairing-code needs to embed the server's externally reachable URL in the QR payload
   - What's unclear: The server may be behind a Cloudflare tunnel, Tailscale, or just on LAN. The `req.get('host')` may not be the right URL for the mobile device.
   - Recommendation: Accept an optional `url` query parameter on the pairing-code endpoint. The desktop UI can pass the tunnel URL or LAN IP. Fall back to `req.protocol + '://' + req.get('host')`.

2. **Token invalidation on server restart**
   - What we know: Server tokens are in-memory. Server restart invalidates all tokens. Mobile app will get 401.
   - What's unclear: How often does the server restart? Should we show a "Server restarted, please re-authenticate" message?
   - Recommendation: On 401, try auto-login with stored password if available (manual connect stores it). For QR-paired devices, show a "Connection expired, scan QR again or enter password" prompt.

3. **Pairing-code endpoint auth requirement**
   - What we know: Design doc shows GET /api/auth/pairing-code requiring auth (desktop user must be logged in)
   - What's unclear: If the server has just started and no one has logged in on desktop yet, how does the user get the QR code?
   - Recommendation: Keep it auth-gated. The user must access the web UI first, log in, then click "Pair Mobile". This is the expected flow for a local dev tool.

## Sources

### Primary (HIGH confidence)

- Existing `src/web/auth.js` source code - token management patterns, rate limiting, timing-safe comparison
- Design doc `docs/plans/2026-03-28-myrlin-mobile-design.md` Section 3.2, 8.1, 8.3 - QR pairing protocol, ServerConnection type
- [expo-camera docs](https://docs.expo.dev/versions/latest/sdk/camera/) - CameraView barcode scanning API
- [expo-secure-store docs](https://docs.expo.dev/versions/latest/sdk/securestore/) - Keychain/Keystore encrypted storage
- [expo-local-authentication docs](https://docs.expo.dev/versions/latest/sdk/local-authentication/) - Biometric prompt API
- [Expo barcode-scanner to expo-camera migration](https://github.com/expo/fyi/blob/main/barcode-scanner-to-expo-camera.md) - Migration guide

### Secondary (MEDIUM confidence)

- [Zustand SecureStore persist discussion](https://github.com/pmndrs/zustand/discussions/2196) - Custom storage adapter patterns
- [Biometric auth in React Native Expo guide](https://sasandasaumya.medium.com/biometric-authentication-in-react-native-expo-a-complete-guide-face-id-fingerprint-732d80e5e423) - Implementation patterns

### Tertiary (LOW confidence)

- None. All findings verified against official sources.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - All libraries are Expo SDK bundled, well documented
- Architecture: HIGH - Follows existing auth.js patterns and design doc contracts
- Pitfalls: HIGH - Based on known Expo/RN patterns and direct source code analysis

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (stable domain, no fast-moving dependencies)
