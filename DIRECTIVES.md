# 🏛️ ELDEN EARTH — DEVELOPER & AI AGENT OPERATIONAL DIRECTIVES
> **Repository:** `vicsanity623/Elden-Earth-MAIN`  
> **Target Audience:** GitHub Copilot, OpenCode Agents, Autonomous Coding Assistants, and Human Maintainers.  
> **Mandate:** Implement security, anti-spoofing, and financial safeguards systematically in strict phases. Ensure zero regressions, zero orphan code, zero duplicate variable/function declarations, and preserve all existing gameplay mechanics.

---

## 🛑 NON-NEGOTIABLE OPERATIONAL LAWS

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

## 🗺️ IMPLEMENTATION PHASES & ROADMAP

---

### 📍 PHASE 1: Multi-Layered Location & Network Verification

#### Goal:
Ensure players cannot claim parcels, harvest diamonds, or collect rewards via GPS spoofing, VPNs, or mock coordinate injectors.

#### Tasks:
* [ ] **1.1 IP Geolocation & VPN Cross-Referencing:**
  * In `js/geo.js`, implement a lightweight network origin check (e.g., query public IP lookup API during initial load or land purchases).
  * Compare the coarse IP-derived location (Country/Region/City) with the hardware GPS coordinates.
  * If distance between IP location and GPS coordinates exceeds **500 km** (and user is not on cellular carrier roaming), flag session as `isNetworkSuspicious = true`.
  * If a known VPN or Datacenter IP is detected, disable real-money withdrawal buttons and display a soft advisory.
* [ ] **1.2 Speed & Teleportation Sanity Watchdog:**
  * In `js/main.js` (`beginWatch` / `handlePosition`), track `(lastLat, lastLon, lastTimestamp)`.
  * Calculate realistic ground velocity:
    $$\text{Velocity} = \frac{\text{Distance (km)}}{\text{Delta Time (hours)}}$$
  * If a player travels at speeds $>900\text{ km/h}$ over distances $>5\text{ km}$, immediately block the position update.
  * Log infraction to `state.antiCheatStrikes`. If strikes $\ge 3$, call `triggerInstantBan()`.
* [ ] **1.3 GPS Accuracy & Mock Provider Detection:**
  * Inspect `coords.accuracy` in `navigator.geolocation`. If `coords.accuracy <= 0` or exactly equal to artificial simulator defaults (e.g., `accuracy === 5.000000` with 0 altitude variance), flag the position.
  * If `coords.altitude === null` and `coords.speed === null` while moving across tiles, flag as potential emulator.

#### Verification & Exit Criteria:
* Run local mock location tests.
* Ensure authentic walking players (0–15 km/h) experience zero false positives.
* Bump `sw.js` cache.

---

### 🛡️ PHASE 2: Hardware & Client Environment Integrity

#### Goal:
Detect tampered browser environments, mobile dev-tools mock locations, and unauthorized automation scripts.

#### Tasks:
* [ ] **2.1 Developer Mock Location & Automation Detection:**
  * In `js/auth.js` / `js/main.js`, check for automation signals:
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

### 💰 PHASE 3: Financial Anti-Fraud & Withdrawal Ledger

#### Goal:
Safeguard the game treasury by preventing sudden draining of real-money payouts, enforcing KYC readiness, and logging all cashout actions into an immutable ledger.

#### Tasks:
* [ ] **3.1 90-Day Account Age & 500-Plot Eligibility Gate:**
  * In `js/storage.js` / withdrawal logic:
    * Check `state.createdAt` and `Object.keys(state.plots || {}).length`.
    * Accounts created post-launch must satisfy:
      $$\text{Account Age} \ge 90\text{ Days} \quad \text{OR} \quad \text{Plots Owned} \ge 500$$
    * Early Adopters (accounts created prior to launch date with `accountHealedV1 === true` or registered before cut-off) bypass this gate into the VIP Founder queue.
* [ ] **3.2 Strictly Enforced Weekly Withdrawal Limits (TBD Thresholds):**
  * All withdrawal requests are capped equally across all players (e.g., minimum $2.00, maximum weekly cap per player).
  * Prevent any account from claiming more than the global per-player weekly ceiling, regardless of total accumulated in-game rent balance.
* [ ] **3.3 Delayed Withdrawal Review Pipeline (24–72 Hour Audit Window):**
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

### 🏰 PHASE 4: Territory Sovereign Rules & Multi-Language Normalization

#### Goal:
Guarantee that Mayors, Governors, and Presidents are strictly mapped to international territories without text corruption or cross-border leakage.

#### Tasks:
* [ ] **4.1 Universal ISO 3166-1 Alpha-2 Country & Flag Mapping:**
  * Ensure `normalizeCountry()` in `js/leaderboard.js` handles English, French (`États-Unis d'Amérique`), German (`Vereinigte Staaten`), Spanish (`Estados Unidos`), and international variants cleanly into standard country buckets.
  * Maintain mathematical Unicode flag generation via `getFlagEmoji(countryCode)`.
* [ ] **4.2 North Korea & Embargoed Territory Blacklist:**
  * Keep bounding box geofence ($37.6^\circ\text{N} - 43.1^\circ\text{N}, 124.1^\circ\text{E} - 130.7^\circ\text{E}$) and `country_code === "kp"` strictly blocked from all land purchases, presences, and leaderboard listings.
  * Any attempt to ping within this box triggers immediate session termination.
* [ ] **4.3 Real-Time Royalty Stacking Integrity:**
  * Verify `awardTerritoryDividends()` in `js/leaderboard.js` correctly awards stackable royalties (Mayor + Governor + President) up to +6 EB without dropping intermediate titles.
  * Maintain real-time WebSocket listening via `initDividendMailbox()` so online rulers receive immediate HUD balance updates.

#### Verification & Exit Criteria:
* Test plot purchases in foreign cities (UK, France, Germany, Canada); verify local leaders receive royalties and US President only receives royalties from US plots.
* Bump `sw.js` cache.

---

## 🧹 CLEAN CODE & ARCHITECTURE CHECKLIST FOR ALL AGENTS

Before completing any task, every agent MUST verify:

1. [ ] **No Duplicate Variable Declarations:** Check that variables (such as `db`, `state`, `now`) are not declared multiple times in the same scope.
2. [ ] **No Unclosed Blocks:** Verify every `{` matches `}`, every `(` matches `)`, and event listener callbacks terminate with `});`.
3. [ ] **No Silent Failures:** Ensure `try/catch` blocks log warnings with `console.warn("[Module] Notice:", e)` rather than failing silently.
4. [ ] **CSS Bounding Check:** Test all modal and HUD additions against mobile screen widths down to `360px` to guarantee zero horizontal overflow or overlapping text.
5. [ ] **Version Bump:** Update `GAME_VERSION` in `js/config.js` and `CACHE_NAME` in `sw.js`.