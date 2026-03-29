---
phase: 2
slug: connection-and-auth
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-28
---

# Phase 2 - Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest (unit), Maestro (E2E), curl (API smoke) |
| **Config file** | `mobile/jest.config.ts`, `mobile/maestro/config.yaml` |
| **Quick run command** | `cd mobile && npx tsc --noEmit` |
| **Full suite command** | `npm test && cd mobile && npx tsc --noEmit` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** TypeScript compiles (`npx tsc --noEmit`)
- **After every plan wave:** Full test suite passes
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 2-01-01 | 01 | 1 | SRVR-01, SRVR-02 | integration | `curl localhost:3456/api/auth/pairing-code` | ❌ W0 | ⬜ pending |
| 2-01-02 | 01 | 1 | CONN-03 | unit | `cd mobile && npx tsc --noEmit` | ❌ W0 | ⬜ pending |
| 2-02-01 | 02 | 2 | CONN-01 | maestro | `maestro test maestro/flows/qr-scan.yaml` | ❌ W0 | ⬜ pending |
| 2-02-02 | 02 | 2 | CONN-02, CONN-08 | maestro | `maestro test maestro/flows/manual-connect.yaml` | ❌ W0 | ⬜ pending |
| 2-03-01 | 03 | 3 | CONN-05, CONN-06 | unit | `cd mobile && npx tsc --noEmit` | ❌ W0 | ⬜ pending |
| 2-03-02 | 03 | 3 | AUTH-02 | manual | Face ID gate verification | ❌ W0 | ⬜ pending |
| 2-03-03 | 03 | 3 | CONN-04, CONN-07 | unit | TypeScript check for reconnect logic | ❌ W0 | ⬜ pending |

*Status: ⬜ pending, ✅ green, ❌ red, ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] Server must be running for pairing endpoint tests
- [ ] Jest configured for server-side tests (already exists in test/)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| QR code scanning with camera | CONN-01 | Requires physical device camera | Open app, tap Scan QR, point at desktop QR code |
| Face ID biometric lock | AUTH-02 | Requires physical device biometrics | Enable lock in settings, background app, re-open, verify Face ID prompt |
| Connection status across screens | CONN-07 | Visual verification across multiple screens | Navigate all tabs, verify indicator visible |
| Auto-reconnect after network drop | CONN-04 | Requires toggling airplane mode | Connect, enable airplane mode, disable, verify reconnection |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
