---
phase: 1
slug: foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-28
---

# Phase 1 - Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Maestro (E2E), Jest (unit via Expo) |
| **Config file** | `mobile/maestro/config.yaml` (Maestro), `mobile/jest.config.ts` (Jest) |
| **Quick run command** | `cd mobile && npx expo run:ios --configuration Debug` |
| **Full suite command** | `cd mobile && maestro test maestro/flows/` |
| **Estimated runtime** | ~60 seconds (Maestro on Simulator) |

---

## Sampling Rate

- **After every task commit:** Build succeeds (`npx expo export --platform ios`)
- **After every plan wave:** Maestro basic flow test passes
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 60 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 1-01-01 | 01 | 1 | FOUND-01 | build | `npx expo run:ios` | ❌ W0 | ⬜ pending |
| 1-01-02 | 01 | 1 | FOUND-02 | build | `npx expo run:android` | ❌ W0 | ⬜ pending |
| 1-02-01 | 02 | 1 | FOUND-03 | maestro | `maestro test maestro/flows/navigation.yaml` | ❌ W0 | ⬜ pending |
| 1-02-02 | 02 | 1 | FOUND-04 | visual | Screenshot comparison (no theme flash) | ❌ W0 | ⬜ pending |
| 1-02-03 | 02 | 1 | FOUND-05 | visual | `maestro test maestro/flows/theme-switch.yaml` | ❌ W0 | ⬜ pending |
| 1-02-04 | 02 | 1 | FOUND-06 | visual | Font rendering screenshot check | ❌ W0 | ⬜ pending |
| 1-02-05 | 02 | 1 | FOUND-07 | unit | Component snapshot tests | ❌ W0 | ⬜ pending |
| 1-03-01 | 03 | 2 | FOUND-08 | maestro | `maestro test maestro/flows/basic.yaml` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending, ✅ green, ❌ red, ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `mobile/maestro/flows/basic.yaml` - basic app launch and navigation flow
- [ ] `mobile/maestro/flows/navigation.yaml` - tab navigation verification
- [ ] `mobile/maestro/flows/theme-switch.yaml` - theme switching verification
- [ ] Maestro CLI installed on Mac Mini
- [ ] Jest configured in Expo project

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Theme has no flash on cold start | FOUND-04 | Requires physical device cold boot observation | Cold kill app, relaunch, observe first frame |
| Custom fonts render correctly | FOUND-06 | Visual inspection of font rendering | Compare screenshots with web app font rendering |
| 13 themes match web CSS | FOUND-05 | Side-by-side visual comparison | Screenshot each theme on mobile and web, compare |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 60s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
