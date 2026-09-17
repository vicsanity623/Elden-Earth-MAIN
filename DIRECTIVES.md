# ELDEN EARTH — DEVELOPER & AI AGENT OPERATIONAL DIRECTIVES
> **Repository:** `vicsanity623/Elden-Earth-MAIN`  
> **Target Audience:** GitHub Copilot, OpenCode Agents, Autonomous Coding Assistants, and Human Maintainers.  
> **Mandate:** Implement security, anti-spoofing, and financial safeguards systematically in strict phases. Ensure zero regressions, zero orphan code, zero duplicate variable/function declarations, and preserve all existing gameplay mechanics.

---

## NON-NEGOTIABLE OPERATIONAL LAWS

Every AI Agent modifying this codebase MUST enforce the following rules:

1. **Phase-Gated Execution (One Phase at a Time):**
   * Do NOT implement multiple phases simultaneously.
   * Complete Phase 1, verify and test, bump `sw.js` cache, and only then proceed to Phase 2.
2. **Zero Orphan / Duplicate Code:**
   * When modifying a function, NEVER leave old closures, dangling event listeners, duplicate variable declarations (e.g., duplicate `const db`), or abandoned callbacks at the bottom of files.
   * Every edit must be surgical: find the exact line/block and replace it completely.
3. **No Breaking Core Gameplay:**
   * All modifications must preserve:
     * 3D Map Rendering & Dynamic Lighting (MapLibre GL JS + Three.js).
     * Plot claim & boundary rendering (`js/grid.js`).
     * Real-time multiplayer presence, activity feeds, and chat (`js/chat.js`, `js/feed.js`).
     * Territory Mayorship/Governorship/Presidency dividends (`js/leaderboard.js`).
     * Offline earnings & Extractor Beacon timers (`js/storage.js`).
     * Responsive, touch-optimized modal styling (`css/style.css`).
4. **Strict Scope Protection:**
   * Do NOT expose internal tokens or sensitive server configuration to public client-side scopes.
   * Always verify `typeof window !== "undefined"` when calling browser APIs.
5. **Mandatory Cache Invalidation:**
   * Every merged change must bump `CACHE_NAME` in `sw.js` (e.g., `elden-earth-v900.X.XXb`) to force client PWA cache refresh.

---

## IMPLEMENTATION PHASES & ROADMAP

---

### PHASE 1: Multi-Layered Location & Network Verification
**Status: COMPLETED**

#### Goal:
Ensure players cannot claim parcels, harvest diamonds, or collect rewards via GPS spoofing, VPNs, or mock coordinate injectors.

#### Tasks:
* [x] **1.1 IP Geolocation & VPN Cross-Referencing:**
  * `js/geo.js` — `NetworkVerifier` module queries `ip-api.com`, compares IP-derived location vs hardware GPS.
  * Flags VPN, datacenter, proxy connections via ISP keyword matching.
  * Flags sessions where IP-GPS distance exceeds 500 km (exempting mobile/cellular).
  * Results stored on `state.networkVerification` for game-wide access.
  * `gateCashoutButtons()` dims withdrawal UI when VPN/datacenter detected.
* [x] **1.2 Speed & Teleportation Sanity Watchdog:**
  * `js/main.js` — tracks `(lastLat, lastLon, lastTimestamp)` across GPS updates.
  * Calculates velocity: `km / hours`.
  * Blocks position if speed > 900 km/h over distances > 5 km.
  * Increments `state.antiCheatStrikes` per violation; triggers `triggerInstantBan()` at 3 strikes.
  * Ban logs to Firestore `cheat_reports`, signs out, and redirects.
* [x] **1.3 GPS Accuracy & Mock Provider Detection:**
  * `js/main.js` — `detectMockProvider()` function.
  * Detects `accuracy <= 0` (impossible on real hardware).
  * Detects exact simulator defaults (`accuracy === 5` with `altitudeAccuracy === 0`).
  * Detects artificial precision (`5.000000` with trailing zeros).
  * Detects `altitude === null && speed === null` while player is moving > 10m.

#### Verification & Exit Criteria:
* Local mock location tests passed.
* Authentic walking players (0-15 km/h) experience zero false positives.
* `sw.js` cache bumped to `v15.13b`.

---

### SERVER-SIDE ANTI-CHEAT ARCHITECTURE
**Status: DEPLOYED**

#### Goal:
Move all critical validation from client (untrusted) to server (trusted). Client-side checks remain as first-pass filters to reduce Cloud Function calls.

#### Cloud Functions (deployed to `us-central1`):
| Function | Purpose |
|---|---|
| `validatePosition` | Server-side velocity tracking, teleport/mock GPS detection, strike system with auto-ban via `admin.auth().revokeRefreshTokens()` |
| `validatePurchase` | Server-authoritative land buying — validates EB balance, cooldown, proximity, velocity, computes rarity server-side, writes plot atomically |
| `validateCollect` | Server-side diamond collection — validates proximity, velocity, prevents double-collect via `diamond_collects` collection |

#### Client Integration:
| File | Integration Point |
|---|---|
| `js/server-anticheat.js` | Bridge module — initializes Firebase Functions, provides `sendPosition()`, `validatePurchase()`, `validateCollect()` |
| `js/main.js` | Sends position to `validatePosition` every ~25s after local checks pass |
| `js/grid.js` | `executeBuy()` calls `validatePurchase` BEFORE any client Firestore write; uses server-computed rarity |
| `js/diamonds.js` | `attemptCollect()` calls `validateCollect` before allowing collection |

#### Firestore Collections (admin SDK only — client cannot write):
| Collection | Purpose |
|---|---|
| `player_positions/{uid}` | Server-side position checkpoints with velocity tracking |
| `diamond_collects/{uid}_{diamondId}` | Double-collect prevention records |

#### Deployment:
```bash
firebase deploy --only functions
firebase deploy --only firestore:rules
```

---

### PHASE 2: Hardware & Client Environment Integrity
**Status: IN PROGRESS**

#### Goal:
Detect tampered browser environments, mobile dev-tools mock locations, and unauthorized automation scripts.

#### Tasks:
* [ ] **2.1 Developer Mock Location & Automation Detection:**
  * In `js/main.js`, check for automation signals:
    * `navigator.webdriver === true` (Headless Chrome / Puppeteer / Selenium).
    * Unusual User-Agent strings or missing hardware sensors (`window.DeviceOrientationEvent`).
  * If automated browser is detected, abort game initialization and redirect to `#banned-screen`.
* [ ] **2.2 DOM & Console Tampering Traps:**
  * Protect core state variables (`state.eb`, `state.cash`, `state.diamonds`).
  * Implement `Object.freeze` or property setters with sanity boundary checks so executing `Store.get().cash = 999999` in DevTools console triggers an immediate tamper strike instead of modifying game memory.
* [ ] **2.3 PWA Standalone Mode Priority:**
  * Prioritize standalone PWA mode (`window.matchMedia('(display-mode: standalone)').matches` or `navigator.standalone`).
  * For mobile users, display guidance recommending "Add to Home Screen" to run within an isolated webview container, reducing exposure to browser extension script injectors.

#### Verification & Exit Criteria:
* Attempt console modifications in DevTools; verify tampering traps catch and halt execution.
* Verify genuine iOS Safari and Android Chrome users can play without friction.
* Bump `sw.js` cache.

---

### PHASE 3: Financial Anti-Fraud & Withdrawal Ledger

#### Goal:
Safeguard the game treasury by preventing sudden draining of real-money payouts, enforcing KYC readiness, and logging all cashout actions into an immutable ledger.

#### Tasks:
* [ ] **3.1 90-Day Account Age & 500-Plot Eligibility Gate:**
  * In `js/storage.js` / withdrawal logic:
    * Check `state.createdAt` and `Object.keys(state.plots || {}).length`.
    * Accounts created post-launch must satisfy:
      * Account Age >= 90 Days OR Plots Owned >= 500.
    * Early Adopters (accounts created prior to launch date with `accountHealedV1 === true` or registered before cut-off) bypass this gate into the VIP Founder queue.
* [ ] **3.2 Strictly Enforced Weekly Withdrawal Limits (TBD Thresholds):**
  * All withdrawal requests are capped equally across all players (e.g., minimum $2.00, maximum weekly cap per player).
  * Prevent any account from claiming more than the global per-player weekly ceiling, regardless of total accumulated in-game rent balance.
* [ ] **3.3 Delayed Withdrawal Review Pipeline (24-72 Hour Audit Window):**
  * Withdrawals must NEVER be executed instantaneously.
  * When a player requests a cashout, write a pending request to Firestore `/withdrawals/{requestId}` with:
    * `userId`, `amount`, `paypalEmail`, `status: "pending_review"`.
    * Snapshot of recent territory claims and movement velocity.
  * Allow 24 to 72 hours for automated fraud heuristics to confirm no speed bans, multiple IP collisions, or banned device fingerprints before marking `status: "approved"`.
* [ ] **3.4 KYC (Know Your Customer) Infrastructure Hook:**
  * In `#player-info-modal` / withdrawal modal, create a placeholder status badge: `KYC Status: [Unverified | Verified]`.
  * Require verified status before processing payouts above minimum thresholds, eliminating multi-account bot farms.

#### Verification & Exit Criteria:
* Simulate withdrawal request with insufficient account age; confirm UI correctly explains the 90-day / 500-plot requirement.
* Verify Firestore `/withdrawals` documents contain complete audit payloads.
* Bump `sw.js` cache.

---

### PHASE 4: Territory Sovereign Rules & Multi-Language Normalization

#### Goal:
Guarantee that Mayors, Governors, and Presidents are strictly mapped to international territories without text corruption or cross-border leakage.

#### Tasks:
* [ ] **4.1 Universal ISO 3166-1 Alpha-2 Country & Flag Mapping:**
  * Ensure `normalizeCountry()` in `js/leaderboard.js` handles English, French, German, Spanish, and international variants cleanly into standard country buckets.
  * Maintain mathematical Unicode flag generation via `getFlagEmoji(countryCode)`.
* [ ] **4.2 North Korea & Embargoed Territory Blacklist:**
  * Keep bounding box geofence and `country_code === "kp"` strictly blocked from all land purchases, presences, and leaderboard listings.
  * Any attempt to ping within this box triggers immediate session termination.
* [ ] **4.3 Real-Time Royalty Stacking Integrity:**
  * Verify `awardTerritoryDividends()` in `js/leaderboard.js` correctly awards stackable royalties (Mayor + Governor + President) up to +6 EB without dropping intermediate titles.
  * Maintain real-time WebSocket listening via `initDividendMailbox()` so online rulers receive immediate HUD balance updates.

#### Verification & Exit Criteria:
* Test plot purchases in foreign cities (UK, France, Germany, Canada); verify local leaders receive royalties and US President only receives royalties from US plots.
* Bump `sw.js` cache.

---

## CLEAN CODE & ARCHITECTURE CHECKLIST FOR ALL AGENTS

Before completing any task, every agent MUST verify:

1. [ ] **No Duplicate Variable Declarations:** Check that variables (such as `db`, `state`, `now`) are not declared multiple times in the same scope.
2. [ ] **No Unclosed Blocks:** Verify every `{` matches `}`, every `(` matches `)`, and event listener callbacks terminate with `});`.
3. [ ] **No Silent Failures:** Ensure `try/catch` blocks log warnings with `console.warn("[Module] Notice:", e)` rather than failing silently.
4. [ ] **CSS Bounding Check:** Test all modal and HUD additions against mobile screen widths down to `360px` to guarantee zero horizontal overflow or overlapping text.
5. [ ] **Version Bump:** Update `GAME_VERSION` in `js/config.js` and `CACHE_NAME` in `sw.js`.

---

## FILE MANIFEST

| File | Purpose |
|---|---|
| `functions/index.js` | Cloud Functions: validatePosition, validatePurchase, validateCollect |
| `functions/package.json` | Cloud Functions dependencies |
| `js/server-anticheat.js` | Client bridge to Cloud Functions |
| `js/geo.js` | Geometry + NetworkVerifier (IP/VPN detection) |
| `js/anticheat.js` | Client-side GPS validation, embargo, rate limiting |
| `js/main.js` | Game loop + teleport watchdog + mock GPS detection |
| `js/grid.js` | Land purchases (server-validated) |
| `js/diamonds.js` | Diamond collection (server-validated) |
| `js/storage.js` | Save data + Firestore cloud sync |
| `firestore.rules` | Firestore security rules |
| `sw.js` | Service worker + PWA cache |
