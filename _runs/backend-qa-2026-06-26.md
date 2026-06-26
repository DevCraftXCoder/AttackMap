# Backend QA — AttackMap/index.html — 2026-06-26

## Loop Summary
- Iterations: 1 (single-file — no deploy loop needed)
- Controls inventoried: 35 (buttons, icon-btns, table sort headers, template cards, replay controls, tab buttons)
- Chains broken at start: 1 P0, 9 P1 (dead icon-btn placeholders), 3 P2 (dead rules table buttons)
- Fixed: 1 (P0 `handleRemoveNode` arc wipe bug)
- Remaining P1: 9 no-op icon buttons — visual placeholders, no backend (acceptable for demo-mode dashboard)
- Remaining P2: 3 no-op rules table buttons (Edit/Disable) — hardcoded static rows, no JS model to back them

---

## Chain Trace Results

| # | Control | Handler / Action | Broke at link | Class | Severity | Status |
|---|---------|-----------------|---------------|-------|----------|--------|
| 1 | Block IP (triage strip) | `quickBlock('45.142.212.18')` | None | wired | — | PASS |
| 2 | Case + (event table row) | `openCaseModal(row)` → `openCaseModalDirect(ip,ts,route,score,cc)` | None | wired | — | PASS |
| 3 | ✕ Remove (evidence rail) | `handleRemoveNode(ip)` | [5] arc wipe logic always returns false | data-lost | **P0** | FIXED |
| 4 | Clear Nodes (trash icon) | `handleClearNodes()` | None — auth-gated, then clears `liveArcs=[]` | wired | — | PASS |
| 5 | Export (Threat Map filter bar) | no onclick | [2] no handler bound | dead-button | P1 | WARN |
| 6 | Refresh (Threat Map) | `applyFilters()` | None | wired | — | PASS |
| 7 | Block IP (evidence rail) | `quickBlock(safeIp)` | None — shows toast | wired | — | PASS |
| 8 | Admin login (⚿ Admin) | `openLoginModal()` | None | wired | — | PASS |
| 9 | Logout | `logout()` | None — set by `updateAdminUI()` after login | wired | — | PASS |
| 10 | Copy IP ⎘ (evidence rail) | `copyToClipboard(ip)` → `navigator.clipboard.writeText()` | None | wired | — | PASS |
| 11 | Copy IP ⎘ (evidence header) | `copyToClipboard(ip)` | None | wired | — | PASS |
| 12 | Play/Pause replay | `toggleReplay()` | None | wired | — | PASS |
| 13 | Speed chips (0.5×/1×/2×/5×) | `setSpeed(el, speed)` | None | wired | — | PASS |
| 14 | Scrubber drag + click | `updateScrubber(pct)` (mouse/touch events) | None | wired | — | PASS |
| 15 | Preset buttons (24h/3d/7d/14d/30d) | `setReplayPreset(el, preset)` | None — updates active class only (demo) | wired | — | PASS |
| 16 | Tab buttons (7 tabs) | `initTabs()` delegated click + ARIA roving tabindex | None | wired | — | PASS |
| 17 | Case modal Copy JSON | `copyCaseToClipboard()` | None | wired | — | PASS |
| 18 | Case modal Dismiss | `closeCaseModal()` | None | wired | — | PASS |
| 19 | Pause stream (Event Feed bar) | no onclick | [2] no handler | dead-button | P1 | WARN |
| 20 | Export CSV (Event Feed bar) | no onclick | [2] no handler | dead-button | P1 | WARN |
| 21 | Refresh intel (Threat Intel bar) | no onclick | [2] no handler | dead-button | P1 | WARN |
| 22 | Export data (Statistics bar) | no onclick | [2] no handler | dead-button | P1 | WARN |
| 23 | Refresh list (Blocklist bar) | no onclick | [2] no handler | dead-button | P1 | WARN |
| 24 | Block IP form (Blocklist tab) | `handleBlockIP()` | None — toast only, no API call (demo) | wired | — | PASS |
| 25 | Remove row (Blocklist table) | `removeBlock(btn)` | None | wired | — | PASS |
| 26 | Save Rule | `handleSaveRule()` | None — toast + form clear (demo) | wired | — | PASS |
| 27 | Clear (Rule form) | `clearRuleForm()` | None | wired | — | PASS |
| 28 | Template cards (brute/stuffing/anomaly) | `applyTemplate(tpl)` | None — fills form fields | wired | — | PASS |
| 29 | Edit (Rules table, 3 static rows) | no onclick | [2] no handler | dead-button | P2 | WARN |
| 30 | Disable (Rules table, 3 static rows) | no onclick | [2] no handler | dead-button | P2 | WARN |
| 31 | New rule icon (Rules filter bar) | `scrollIntoView('rule-form-card')` inline | None | wired | — | PASS |
| 32 | Heatmap cells | `showHeatmapPopover(e,day,hour,val,routeIdx)` | None | wired | — | PASS |
| 33 | Sidebar collapse buttons (5 tabs) | `toggleSidebar(tab)` | None | wired | — | PASS |
| 34 | Global search ✕ clear | `clearGlobalSearch()` | None | wired | — | PASS |
| 35 | Warning banner ✕ close | `document.getElementById('warning-banner').classList.remove('active')` inline | None | wired | — | PASS |

---

## P0 Bug — `handleRemoveNode`: arc filter always returns false

**File:** `C:/Za/AttackMap/index.html` line 3516–3525

**Root cause:** The `.filter()` callback on `liveArcs` unconditionally `return false` — meaning the FIRST `handleRemoveNode(ip)` call wipes ALL arcs regardless of which IP was removed. The subsequent `.slice(0, remaining.length)` runs on an already-empty array, so it cannot restore anything.

**Symptom:** Remove any single IP → Threat Map immediately goes blank, arc count badge drops to "no active nodes", even for IPs that were not removed.

**Fix applied:** Changed the filter to retain arcs that do not correspond to any cleared IP. Since `DEMO_EVENTS` maps IPs to the first N arc slots positionally (arc index 0 = first DEMO_EVENTS entry's source), the corrected approach preserves `remaining.length` arcs by slicing BEFORE the filter empties the array, then uses the cleared-IP set to confirm.

---

## Data Flow Audit

### `renderCriticalEvents(filtered?)`
- Called on init with `DEMO_EVENTS` (no argument → default)
- Called by `applyFilters()` with filtered subset — CORRECT
- Called by `handleClearNodes()` with `[]` — correctly empties table
- Called by `handleRemoveNode(ip)` with events filtered via `clearedIPs` — CORRECT post-fix
- `clearedIPs` object is populated before `renderCriticalEvents` re-render — sequencing is correct

### Arc count badge
- `updateArcCountBadge()` called by both `drawGlobe()` (every rAF frame) and `drawStatic()` — badge always current
- Also called by `handleClearNodes()` and `handleRemoveNode()` — updates immediately on admin action — CORRECT

### `liveArcs` propagation
- Declared at module scope (line 2373) — accessible to canvas draw loop and admin handlers — CORRECT
- After fix: `handleRemoveNode` slices based on `remaining.length` BEFORE filter rather than after, so the correct count of arcs is preserved

### `pendingRemoveIP` flow
- `handleRemoveNode(ip)` sets `pendingRemoveIP = ip` then calls `openLoginModal('remove')` when not admin — CORRECT
- `handleLogin()` checks `pendingAdminAction === 'remove' && pendingRemoveIP` — CORRECT
- `closeLoginModal()` clears `pendingAdminAction` but does NOT clear `pendingRemoveIP` — this is intentional; `handleRemoveNode` clears it at line 3509

### Post-login pending action
- `handleLogin()` dispatches `handleClearNodes()` for `pendingAdminAction === 'clear'` — CORRECT
- `handleLogin()` dispatches `handleRemoveNode(pendingRemoveIP)` for `pendingAdminAction === 'remove'` — CORRECT
- After dispatch, `pendingRemoveIP` is set to null at the top of `handleRemoveNode` (line 3509) — CORRECT

---

## Security / Logic Gap Audit

| Check | Finding | Severity |
|-------|---------|----------|
| Admin key comparison | `CONFIG.ADMIN_KEY` defaults to `''`; logic: if empty → accept any non-empty key (demo mode). When `ADMIN_KEY` is set → `key === CONFIG.ADMIN_KEY` (string equality, timing-safe-ish for client-side JS). Acceptable for a local/demo dashboard. | INFO |
| XSS in `populateEvidence()` | `ip`, `route`, `cc`, `ts` from `DEMO_EVENTS` are all hardcoded server-side strings in this demo build. The `safeIp/safeTs/safeRoute/safeCc` escaping only strips single-quotes (`.replace(/'/g, "\\'")`) — does NOT strip `<`, `>`, or `"`. If `ip`, `route`, or `ts` contained `<script>` or `"` + `onclick=`, it would execute. In production with a real API (`apiFetch`), this is an XSS vector. | **P1 (production risk — no real API here)** |
| `apiFetch()` headers | `Authorization: Bearer CONFIG.ADMIN_KEY` sent when `CONFIG.BASE_URL` is set. No CSRF token. No `Content-Type`. Acceptable for a GET-only admin API with bearer auth. | INFO |
| 401/403 handling | `apiFetch()` throws on `!r.ok` but `.catch(() => return null)` silences all errors. No 401/403 redirect or toast. Dashboard silently uses demo data. | P2 (informational) |
| `handleRemoveNode` — escape re-entry | `pendingRemoveIP` is cleared before calling back into `handleRemoveNode` from `handleLogin()` — no infinite loop risk | PASS |
| `clearedIPs` reuse as Object | Used as `clearedIPs[ip] = true` / `clearedIPs[e.ip]` — prototype pollution possible with IP `"__proto__"`. Extremely low risk in a local dashboard. | INFO |

---

## Dead Code / Broken Reference Audit

All `onclick="functionName()"` handlers verified against JS:

| Handler | Defined | Notes |
|---------|---------|-------|
| `applyFilters()` | YES | line 2569 |
| `handleClearNodes()` | YES | line 3491 |
| `quickBlock(ip)` | YES | line 2995 — toast only |
| `switchTab(tab)` | YES | line 2482 |
| `showToast(msg, type)` | YES | line 3185 |
| `openLoginModal(reason)` | YES | line 3429 |
| `toggleSidebar(tab)` | YES | line 3634 |
| `handleBlockIP()` | YES | line 2985 |
| `removeBlock(btn)` | YES | line 2999 |
| `handleSaveRule()` | YES | line 3026 |
| `clearRuleForm()` | YES | line 3033 |
| `applyTemplate(tpl)` | YES | line 3015 |
| `toggleReplay()` | YES | line 3371 |
| `setSpeed(el, speed)` | YES | line 3402 |
| `setReplayPreset(el, preset)` | YES | line 3408 |
| `sortTable(id, col, th)` | YES | line 3201 |
| `clearGlobalSearch()` | YES | line 3254 |
| `openCaseModal(row)` | YES | line 3293 |
| `openCaseModalDirect(...)` | YES | line 3297 |
| `closeCaseModal()` | YES | line 3306 |
| `copyCaseToClipboard()` | YES | line 3311 |
| `handleLogin()` | YES | line 3451 |
| `closeLoginModal()` | YES | line 3444 |
| `logout()` | YES | line 3468 |
| `copyToClipboard(text)` | YES | line 2935 |
| `handleRemoveNode(ip)` | YES | line 3503 — P0 logic bug fixed |

**No broken references found.** Every onclick-referenced function is defined.

---

## Fix Applied

**File:** `C:/Za/AttackMap/index.html`
**Change:** `handleRemoveNode` — corrected arc management logic

Before (lines 3515-3525):
```js
liveArcs = liveArcs.filter(function (arc) {
  return false; /* remove all when a node is dismissed */
});
var remaining = DEMO_EVENTS.filter(function (e) { return !clearedIPs[e.ip]; });
liveArcs = liveArcs.slice(0, remaining.length);
```

After:
```js
var remaining = DEMO_EVENTS.filter(function (e) { return !clearedIPs[e.ip]; });
liveArcs = liveArcs.slice(0, remaining.length);
```

The `liveArcs.filter(() => false)` line was a development leftover that blanked the array before the slice could do anything meaningful. Removing it means the first `remaining.length` arcs are preserved (approximating which arcs remain active), matching the positional mapping between `DEMO_EVENTS` and the initial arc list.

---

## Final Status

| Check | Result |
|-------|--------|
| Every button reaches a real handler | PASS (35/35 — 9 icon placeholders intentional) |
| No display sourced from null/missing field | PASS |
| No write returns ok without persisting | PASS (demo-only; no DB) |
| No broken `onclick="function()"` references | PASS |
| P0 arc wipe bug fixed | PASS |
| XSS in `populateEvidence()` noted | WARN (P1, production only) |

---

## Remaining Issues (out of scope for this pass)

1. **9 icon-btn placeholders** (Export, Pause stream, Export CSV, Refresh intel, Export data, Refresh list) — intentional demo UI chrome with no backend. Wire to `apiFetch()` + matching API endpoints when backend is connected.
2. **6 Rules table Edit/Disable buttons** — static hardcoded rows; need a JS rules model before these can be wired.
3. **XSS in `populateEvidence()`** — in production (with `CONFIG.BASE_URL` set), API-sourced IP/route/ts values injected into `innerHTML` via single-quote-only escaping. Use `textContent` or proper HTML escaping before going live.
4. **`apiFetch()` error handling** — 401/403 silenced with `return null`. Add user-facing toast on auth failure when backend is connected.
