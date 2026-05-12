# 04 — Frontend Pages & Workflows

> 6 new merchant-facing pages + 1 admin-only page + 2 existing pages get modifications. The sidebar renders **three possible views** under "TikTok Shop" — General-only, FBT-only, or Hybrid — driven by per-account `fbm_enabled` / `fbt_enabled` flags aggregated across the client's TikTok accounts.

---

## 1. Sidebar — three conditional views

The two sub-groups under "TikTok Shop" are rendered **independently**, each gated by its own aggregated flag:

| Client's accounts collectively have... | Sidebar shows under "TikTok Shop" |
|---|---|
| At least one with `fbm_enabled=true` | General TikTok submodules (Accounts, Active/Pending/Deactivated products) |
| At least one with `fbt_enabled=true` | FBT submodules (Dashboard, Inventory, Orders, Inbound, Fees) |
| Both conditions true | Both groups, in that order |

**One edge case worth surfacing explicitly:** the "Accounts" page itself sits inside the General group, but it must remain reachable even on FBT-only clients because that's where the FBT toggle lives. We solve this by always including the Accounts entry — it's the *product/listing* entries that disappear for FBT-only merchants, not Accounts.

In `frontend_wms_v/src/constants/clientSidebarNavLinks.constant.js`:

```javascript
// Inside the sidebar builder (called whenever accounts change)
const buildTiktokGroup = (accounts) => {
  const anyFbmEnabled = accounts.some(a => a.fbm_enabled === true);
  const anyFbtEnabled = accounts.some(a => a.fbt_enabled === true);

  // Always-visible: Accounts page (entry point for both modes' toggles)
  const accountsEntry = { id: 1901, name: 'Accounts', link: '/clients/tiktok/accounts' };

  // General TikTok product/listing entries — hidden on FBT-only clients
  const generalEntries = anyFbmEnabled ? [
    { id: 1902, name: 'Active products',    link: '/clients/tiktok/active-products' },
    { id: 1903, name: 'Pending products',   link: '/clients/tiktok/pending-products' },
    { id: 1904, name: 'Deactivated',        link: '/clients/tiktok/deactivated-products' },
  ] : [];

  // FBT entries — hidden on FBM-only clients
  const fbtEntries = anyFbtEnabled ? [
    { id: 1910, name: '── FBT ──',                 isHeading: true },
    { id: 1911, name: 'FBT Dashboard',             link: '/clients/tiktok/fbt/dashboard' },
    { id: 1912, name: 'FBT Inventory',             link: '/clients/tiktok/fbt/inventory' },
    { id: 1913, name: 'FBT Orders',                link: '/clients/tiktok/fbt/orders' },
    { id: 1914, name: 'FBT Inbound shipments',     link: '/clients/tiktok/fbt/inbound',
      featureFlag: 'tiktok_fbt_inbound' },
    { id: 1915, name: 'FBT Fees',                  link: '/clients/tiktok/fbt/fees',
      featureFlag: 'tiktok_fbt_fees' },
  ] : [];

  // Insert a heading above the general group only when both groups render,
  // so hybrid clients see clear separation. Single-mode clients see no headings.
  const generalHeading = (anyFbmEnabled && anyFbtEnabled)
    ? [{ id: 1900, name: '── General ──', isHeading: true }]
    : [];

  return {
    id: 19,
    name: 'TikTok Shop',
    icon: <SiTiktok size={24} />,
    link: '#',
    children: [
      accountsEntry,
      ...generalHeading,
      ...generalEntries,
      ...fbtEntries,
    ]
  };
};
```

The builder re-runs whenever `useTiktokAccountsStore` accounts change (after connect, FBT enable/disable, or FBM disable), so the sidebar reshapes within one render tick. `useModulePermissionStore` still filters server-driven; the inline `featureFlag` check handles Phase 2/3 progressive rollout.

**Residual-data safeguard:** even when a merchant flips an account to FBT-only, the General entries stay visible if `tiktok_orders` still contains rows with `fulfilment_mode='fbm'` for that client (so the merchant can reach historical orders and returns). The aggregate check becomes:

```javascript
const anyFbmEnabled = accounts.some(a => a.fbm_enabled === true)
                   || clientHasHistoricalFbmOrders;  // boolean from accounts store
```

`clientHasHistoricalFbmOrders` is computed server-side in the same payload that returns `accounts` (cheap `EXISTS` query) so the sidebar needs no second round-trip.

---

## 2. Existing pages that get modified (2 pages)

### M1. Accounts page — `/clients/tiktok/accounts`

The biggest UI change: each account card gains a **"Fulfilment modes"** panel with two independent toggles — one for FBM (seller-fulfilled, default ON) and one for FBT (default OFF). The card must reach a state where at least one toggle is ON (DB constraint `chk_accounts_any_mode`); the UI disables the last remaining ON toggle and shows a tooltip explaining why.

**Before** (today): card shows account name, shop ID, status, sync toggles, view/disconnect actions.

**After**: same card + a new section near the bottom titled "Fulfilment modes":

```
Fulfilment modes
────────────────
[●━━] Seller-fulfilled (FBM)        Active · 3,420 listings synced
[━━○] Fulfilled by TikTok (FBT)     Not enabled — [Enable FBT]
```

Each toggle has its own ON/OFF surface:

- **FBM toggle**
  - ON (default): "Seller-fulfilled — TikTok orders flow into your WMS picker queue. X listings active." No action buttons; this is the standard mode.
  - OFF: "Seller fulfilment disabled. Orders for this shop are handled exclusively by TikTok FBT." Shown only when the merchant has explicitly converted the shop to FBT-only. `[Re-enable FBM]` button.
  - Disabling FBM requires confirmation modal: "All future TikTok orders for this shop will be handled by TikTok FBT. Existing FBM orders in progress will complete normally. Continue?"

- **FBT toggle**
  - OFF (default): "Enable FBT to mirror TikTok-fulfilled inventory and orders for this shop. [Learn more]"
  - ON: capability summary (regions covered, FBT shop ID), last verification time, last inventory sync, `[Sync now]` / `[Re-verify capabilities]` / `[Disable FBT]` buttons.

**Flow when FBT clicked OFF→ON:**

```
1. Click toggle → confirmation modal "Verify FBT eligibility for shop X?"
2. POST /api/tiktok/accounts/:id/fbt/enable
3. Backend calls TikTok to verify
4. If not enrolled → toast "This shop is not enrolled in FBT. Apply: <link>"
5. If enrolled → toggle flips, capability summary appears, sidebar refreshes (FBT items now visible)
```

**Flow when FBM clicked ON→OFF:**

```
1. Click toggle → confirmation modal as above
2. POST /api/tiktok/accounts/:id/fbm/disable
3. Backend rejects with 400 if fbt_enabled=false on the same row (constraint guard before DB)
4. Backend rejects with 400 if any open FBM order exists (configurable: warn-only vs hard-block)
5. Otherwise: UPDATE accounts SET fbm_enabled=false; publish activity log; sidebar reshapes
```

The reverse `[Re-enable FBM]` is one click with no eligibility check — FBM is always available to any connected TikTok shop.

**Edge state — admin-paused publishing.** The `accounts.publish_fbt_to_wms` column is an admin-only kill-switch (default TRUE; see `02-data-model-changes.md` §A1). When ops sets it to `FALSE`, the merchant's FBT toggle stays ON but no RabbitMQ messages flow to core-service. The Accounts card surfaces this state with a single amber banner above the Fulfilment modes panel:

> **FBT publishing temporarily paused.** Your shop is enabled, but our team has paused data flow to your WMS while we resolve a downstream issue. New FBT orders and inventory updates will resume once paused is lifted — no action needed from you. Contact support if this banner is still here after 24 hours.

The banner appears whenever `account.fbt_enabled=true AND account.publish_fbt_to_wms=false`. Merchants cannot dismiss it — it disappears the moment ops re-enables publishing. The FBT toggle controls in the card are not greyed out (the merchant can still disable FBT entirely if they want), but the `[↻ Sync FBT now]` and `[Re-verify capabilities]` buttons are disabled with a tooltip explaining why.

### M2. Central Orders page — `/clients/order` (existing core WMS page)

**Visually unchanged.** The only change is a single SQL clause added to the listing query: `WHERE orders.created_via != 'tiktok_fbt'`. FBT orders are filtered out entirely. No filter chip, no FBT badge, no FBT rows — the merchant goes to the dedicated FBT Orders page (P7 below) for those.

The picker queue `/clients/order/awaiting-pick` already filters by `order_fulfillments.fulfillment_type != 'fbt'` so FBT is already excluded there. Add an info callout at the bottom of that page reading: "FBT orders are not shown here — TikTok fulfils them. See [TikTok Shop › FBT › Orders]."

---

## 3. New merchant-facing pages (6 pages)

### P1. FBT Dashboard — `/clients/tiktok/fbt/dashboard`

**Purpose:** the daily morning view.
**Layout:** 4 stat cards (Connected FBT accounts · Active FBT SKUs · Units in FBT FCs · Orders today) + a Stock-health donut + a 30-day sales chart + a Reorder Alerts table.
**Key action:** click a red SKU row → opens the Inbound wizard pre-filled.

### P2. FBT Inventory — `/clients/tiktok/fbt/inventory`

**Purpose:** SKU-level mirror of TikTok FBT stock.
**Layout:** filter bar (region, account, health, search) + table with columns: SKU, image, name, fulfillable, inbound, reserved, unfulfillable, days-of-cover, health badge, **WMS own-warehouse stock** (the hybrid SKU give-away).
**Key action:** click row → drawer with breakdown JSONB, 30-day sales bars, [Create inbound shipment].

### P3. FBT Orders — `/clients/tiktok/fbt/orders`

**Purpose:** the only place FBT orders appear. Read-only lens — TikTok fulfils these, the merchant just monitors.
**Layout:** filter bar (status, date range, account, search) + table with columns: order #, date, items, total, status, tracking, customer.
**Sidebar position:** between FBT Inventory and FBT Inbound shipments (user-specified order).
**Key actions:**
- Click row → drawer with order detail, items, customer address (PII masked), TikTok-side status timeline (AWAITING_SHIPMENT → IN_TRANSIT → DELIVERED).
- `[Sync now]` — on-demand poll, rate-limited.
- `[Export CSV]` — current filtered set.
**What this page is NOT:** an action queue. The merchant doesn't pick, pack, ship, or confirm anything here. Every row is informational.

### P4. FBT Inbound shipments — list — `/clients/tiktok/fbt/inbound` (Phase 2)

**Purpose:** track all shipments to TikTok FCs.
**Layout:** filter bar + table: shipment#, created, destination, units, status, tracking link.
**Key action:** [+ New] opens wizard; row → detail page.

### P5. FBT Inbound shipments — wizard — `/clients/tiktok/fbt/inbound/new` (Phase 2)

**Purpose:** 4-step wizard to create an inbound shipment.
**Steps:** Products & quantities → Destination FC → Review → Labels & ship.
**Key action:** [Submit plan] on step 3 commits to TikTok and creates the shipment row.

### P6. FBT Inbound shipment — detail — `/clients/tiktok/fbt/inbound/[id]` (Phase 2)

**Purpose:** live status of one shipment.
**Layout:** header with status pill, key dates, 3 stat cards (from/to/units), timeline of state transitions, items table with planned/shipped/received/damaged/short columns.
**Key actions:** [Cancel shipment] (pre-shipment), [Open in TikTok ↗], discrepancy review.

### P7. FBT Fees & Reimbursements — `/clients/tiktok/fbt/fees` (Phase 3)

**Purpose:** financial reconciliation.
**Layout:** 4 stat cards (Total fees, Pick & pack, Storage, Reimbursed) + bar chart (fees by type, by week) + discrepancy alerts panel + fee history table + reimbursement claims table.
**Key action:** [Submit dispute] on a charge → opens dispute form or external TikTok link.

---

## 4. New admin-only page (1 page)

### A1. Webhook events admin — `/clients/tiktok/fbt/admin/webhooks`

**Purpose:** engineers/support debug "why didn't this order arrive?"
**Layout:** filters + table: received timestamp, event type, status (processed/duplicate/failed), latency, account. Click row → full payload viewer.
**Access:** super-admin only. Not in normal sidebar.

---

## 5. Service layer additions

### File: `src/services/TiktokFbtService.js`

Separate from existing `TiktokService.js`. Different base URL prefix.

```javascript
import { createApiRequest } from '@/helpers/axios';
import { API_URL } from '@/helpers/apiUrl';

const api = createApiRequest(API_URL);

const Queries = {
  // Dashboard
  getDashboard:       (params) => api.get('/tiktok/fbt/dashboard', { params }),

  // Inventory
  getInventory:       (params) => api.get('/tiktok/fbt/inventory', { params }),
  getInventoryItem:   (id)     => api.get(`/tiktok/fbt/inventory/${id}`),

  // Orders (lens — proxies to existing endpoint with fulfilment_mode filter)
  getFbtOrders:       (params) => api.get('/tiktok/fbt/orders', { params }),

  // Phase 2
  getInbound:         (params) => api.get('/tiktok/fbt/inbound', { params }),
  getInboundOne:      (id)     => api.get(`/tiktok/fbt/inbound/${id}`),
  getFcOptions:       (accId)  => api.get(`/tiktok/fbt/accounts/${accId}/fcs`),

  // Phase 3
  getFees:            (params) => api.get('/tiktok/fbt/fees', { params }),
  getReimbursements:  (params) => api.get('/tiktok/fbt/reimbursements', { params }),
};

const Commands = {
  // FBT toggle (calls existing accounts controller — not under /fbt/)
  enableFbt:          (accId)            => api.post(`/tiktok/accounts/${accId}/fbt/enable`),
  disableFbt:         (accId)            => api.post(`/tiktok/accounts/${accId}/fbt/disable`),
  syncFbtAccount:     (accId)            => api.post(`/tiktok/fbt/accounts/${accId}/sync`),

  // Phase 2
  createInbound:      (payload)          => api.post('/tiktok/fbt/inbound', payload),
  submitInbound:      (id)               => api.post(`/tiktok/fbt/inbound/${id}/submit`),
  cancelInbound:      (id)               => api.post(`/tiktok/fbt/inbound/${id}/cancel`),
  getInboundLabel:    (id)               => api.get(`/tiktok/fbt/inbound/${id}/label`),
  markShipped:        (id, body)         => api.post(`/tiktok/fbt/inbound/${id}/ship`, body),

  // Phase 3
  submitDispute:      (chargeId, body)   => api.post(`/tiktok/fbt/charges/${chargeId}/dispute`, body),
};

export default { Queries, Commands };
```

### Zustand stores

```
src/stores/tiktok-fbt/
├── useTiktokFbtDashboardStore.js
├── useTiktokFbtInventoryStore.js
├── useTiktokFbtOrdersStore.js
├── useTiktokFbtInboundStore.js     (Phase 2)
├── useTiktokFbtFeesStore.js        (Phase 3)
└── index.js
```

The existing `useTiktokAccountsStore` gets a small extension: `enableFbt(accountId)` and `disableFbt(accountId)` action methods that hit the new endpoints and refresh the account list (so the sidebar recomputes).

---

## 6. Workflow summaries

**Enabling FBT for the first time:** merchant opens `/clients/tiktok/accounts`, clicks the "Enable FBT" toggle on a connected account → modal confirms → verification call to TikTok → toggle flips green → sidebar refreshes with FBT items → within 15 min, FBT Inventory populates.

**Daily check:** merchant opens FBT Dashboard, sees reorder alerts. Clicks red SKU → lands on FBT Inventory filtered to that SKU → clicks [Create inbound] → wizard pre-fills → 5 min later, shipment created.

**A buyer places an FBT order:** TikTok webhook → tiktok-microservice → routes to FBT publisher → core-service creates Order with `createdVia='tiktok_fbt'`, OrderFulfillment(`type='fbt'`) → order appears in central order list with badge → picker queue does NOT show it.

**Monthly fee reconciliation (Phase 3):** 03:15 cron pulls statement → fees ingested → merchant opens FBT Fees the next morning, exports CSV.

---

## 7. Component reuse map

| New page | Reuses existing |
|---|---|
| FBT Dashboard | `<StatCard>`, MUI X LineChart, `<PageHeadingWithBreadcrumb>` |
| FBT Inventory | `<TableContainer>`, `<TextFilter>`, `<OperatorFilter>`, `<BulkActionBar>`, `<PreviewableImage>` |
| Inbound list | `<TableContainer>` |
| Inbound wizard | MUI Stepper + form-section components from `create-product/` |
| Inbound detail | `<StatCard>`, `<Timeline>` (new tiny component), table primitives |
| Fees | MUI X BarChart, `<StatCard>`, table primitives |
| Webhook admin | `<TableContainer>` |

**New components to build (3 total):**
1. `<FbtHealthChip>` — green/amber/red pill with days-of-cover tooltip.
2. `<FulfilledByTikTokBadge>` — small badge appended to order rows.
3. `<FbtAccountToggle>` — the in-account-card toggle for enabling FBT.

The wizard stepper component can be cobbled from MUI's Stepper without a new wrapper.

---

## 8. Page count summary

| Pages | Count | Phase |
|---|---|---|
| Existing pages modified | 2 (Accounts — gains toggle; central Orders — adds SQL filter only, no visual change) | 1 |
| New merchant-facing pages | 7 (Dashboard, Inventory, Orders, Inbound list, Inbound new, Inbound detail, Fees) | 1 (4 of them: Dashboard, Inventory, Orders, regression to central orders), 2 (3 of them: Inbound list/wizard/detail), 3 (1: Fees) |
| New admin pages | 1 (Webhook events) | 1 |
| **Total new routes added** | **8** | |
