# 04 — Frontend Pages & Workflows

> 6 new pages under `/clients/tiktok-fbt/*`. Top-level sidebar entry **TikTok FBT** is a sibling of the existing **TikTok Shop** entry — not a child. A merchant with both subscriptions sees both groups in the sidebar, independently.
>
> Total page count by phase: Phase 1 = 4 pages, Phase 2 = +1 (Inbound), Phase 3 = +1 (Fees). Reuse map at the bottom.

---

## Sidebar registration

In `frontend_wms_v/src/constants/clientSidebarNavLinks.constant.js`, add a **new top-level** entry after the existing TikTok Shop block (which ends around line 413):

```javascript
// EXISTING — do not modify
{
  id: 19,
  name: "TikTok Shop",
  icon: <SiTiktok size={24} />,
  link: "#",
  children: [
    { id: 1901, name: "Active Products", link: "/clients/tiktok/active-products" },
    { id: 1902, name: "Pending Products", link: "/clients/tiktok/pending-products" },
    { id: 1903, name: "Accounts", link: "/clients/tiktok/accounts" },
  ],
},

// NEW — sibling, top-level, gated by feature flag tiktok_fbt
{
  id: 20,
  name: "TikTok FBT",
  icon: <MdLocalShipping size={24} />,                  // or a TikTok-FBT-specific icon
  link: "#",
  featureFlag: "tiktok_fbt",
  children: [
    { id: 2001, name: "Dashboard",         link: "/clients/tiktok-fbt/dashboard"  },
    { id: 2002, name: "Accounts",          link: "/clients/tiktok-fbt/accounts"   },
    { id: 2003, name: "Inventory",         link: "/clients/tiktok-fbt/inventory"  },
    { id: 2004, name: "Orders",            link: "/clients/tiktok-fbt/orders"     },
    { id: 2005, name: "Inbound Shipments", link: "/clients/tiktok-fbt/inbound",
      featureFlag: "tiktok_fbt_inbound" },
    { id: 2006, name: "Fees & Reimbursements", link: "/clients/tiktok-fbt/fees",
      featureFlag: "tiktok_fbt_fees" },
  ],
},
```

`useModulePermissionStore.fetchSideMenus()` already filters server-driven. A merchant without `tiktok_fbt` feature doesn't see the entire group. A merchant with only seller subscription sees only "TikTok Shop". A merchant with only FBT subscription sees only "TikTok FBT". A merchant with both sees both.

---

## Service layer

### New file: `src/services/TiktokFbtService.js`

Completely separate from `src/services/TiktokService.js`. Different base URL (`/api/tiktok-fbt` vs `/api/tiktok`).

```javascript
import { createApiRequest } from "@/helpers/axios";
import { API_URL } from "@/helpers/apiUrl";

const api = createApiRequest(API_URL);  // axios instance; baseURL handled by gateway routing

const Queries = {
  // Accounts
  getAccounts:        ()       => api.get("/tiktok-fbt/accounts"),
  getAccount:         (id)     => api.get(`/tiktok-fbt/accounts/${id}`),

  // Dashboard
  getDashboard:       (params) => api.get("/tiktok-fbt/dashboard", { params }),

  // Inventory
  getInventory:       (params) => api.get("/tiktok-fbt/inventory", { params }),
  getInventoryItem:   (id)     => api.get(`/tiktok-fbt/inventory/${id}`),

  // Orders
  getOrders:          (params) => api.get("/tiktok-fbt/orders", { params }),
  getOrder:           (id)     => api.get(`/tiktok-fbt/orders/${id}`),

  // Phase 2
  getInboundList:     (params) => api.get("/tiktok-fbt/inbound", { params }),
  getInbound:         (id)     => api.get(`/tiktok-fbt/inbound/${id}`),
  getFcWarehouses:    (accId)  => api.get(`/tiktok-fbt/accounts/${accId}/fcs`),

  // Phase 3
  getFees:            (params) => api.get("/tiktok-fbt/fees", { params }),
  getReimbursements:  (params) => api.get("/tiktok-fbt/reimbursements", { params }),
};

const Commands = {
  // Accounts
  connectAccount:     (payload)         => api.post("/tiktok-fbt/accounts/connect", payload),
  disconnectAccount:  (id)              => api.post(`/tiktok-fbt/accounts/${id}/disconnect`),
  syncAccount:        (id)              => api.post(`/tiktok-fbt/accounts/${id}/sync`),
  linkSellerAccount:  (id, sellerAccId) => api.patch(`/tiktok-fbt/accounts/${id}/link-seller`,
                                                     { sellerAccountId: sellerAccId }),

  // Phase 2
  createInbound:      (payload)    => api.post("/tiktok-fbt/inbound", payload),
  submitInbound:      (id)         => api.post(`/tiktok-fbt/inbound/${id}/submit`),
  confirmInbound:     (id)         => api.post(`/tiktok-fbt/inbound/${id}/confirm`),
  cancelInbound:      (id)         => api.post(`/tiktok-fbt/inbound/${id}/cancel`),
  getInboundLabel:    (id)         => api.get(`/tiktok-fbt/inbound/${id}/label`),
  markShipped:        (id, body)   => api.post(`/tiktok-fbt/inbound/${id}/ship`, body),

  // Phase 3
  submitDispute:      (chargeId, body) => api.post(`/tiktok-fbt/charges/${chargeId}/dispute`, body),
};

export default { Queries, Commands };
```

### Zustand stores

```
src/stores/tiktok-fbt/
├── useTiktokFbtAccountsStore.js
├── useTiktokFbtDashboardStore.js
├── useTiktokFbtInventoryStore.js
├── useTiktokFbtOrdersStore.js
├── useTiktokFbtInboundStore.js    (Phase 2)
├── useTiktokFbtFeesStore.js       (Phase 3)
└── index.js                       (barrel export)
```

Each store mirrors the existing TikTok store pattern: `loading`, `items[]`, `pagination`, `filters`, `selectedIds[]`, `get…` / `setFilters` / command methods.

### Constants

```
src/constants/tiktok-fbt/
├── tiktokFbtInventory.constant.js     (table header definitions)
├── tiktokFbtOrders.constant.js
├── tiktokFbtInbound.constant.js       (Phase 2)
├── tiktokFbtFees.constant.js          (Phase 3)
└── tiktokFbtHealth.constant.js        (green/amber/red thresholds)
```

Also: register channel ID in `src/constants/channelRoutes.constant.js` so dashboard channel tiles can deep-link.

---

## Page-by-page

### P1.1 — FBT Dashboard

**Route:** `/clients/tiktok-fbt/dashboard`
**Phase:** 1
**Purpose:** Single-screen health view across all the merchant's FBT accounts.
**Reuses:** `<StatCard>`, `<CounterCard>`, MUI X Charts (line chart for sales-7d), `<PageHeadingWithBreadcrumb>`, `useTiktokFbtDashboardStore`.

**Layout (top to bottom):**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  TikTok FBT Dashboard                            [Last synced: 7m ago]  │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐            │
│  │ Connected  │ │ FBT SKUs   │ │ Units in   │ │ FBT orders │            │
│  │ accounts   │ │ active     │ │ FBT FCs    │ │ today      │            │
│  │     2      │ │    147     │ │   3,840    │ │     58     │            │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘            │
│                                                                          │
│  ┌────────────────────────────┐  ┌─────────────────────────────────┐    │
│  │ Stock health               │  │ FBT sales (last 30 days)         │    │
│  │  ● Green:  124             │  │ [line chart]                     │    │
│  │  ● Amber:   18             │  │                                  │    │
│  │  ● Red:      5  →[click]   │  │                                  │    │
│  └────────────────────────────┘  └─────────────────────────────────┘    │
│                                                                          │
│  ┌─ Reorder alerts (red < 14 days cover) ───────────────────────────┐   │
│  │  SKU         Days cover    Fulfillable    Velocity/day    Action │   │
│  │  WIDGET-RED      6.5            42           6.5         [Plan]  │   │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

**User actions:**
- Click "Red: 5" → `/clients/tiktok-fbt/inventory?health=red`.
- Click `[Plan]` next to a red SKU → opens Phase-2 inbound wizard pre-filled with that SKU. Disabled in Phase 1 with tooltip.

**Edge cases:** zero accounts → empty state with CTA "Connect your first FBT account". One account but no SKUs synced → "First sync in progress — back in 15 minutes." Last sync > 60 min ago → red banner "Sync stalled — check account status".

---

### P1.2 — FBT Accounts

**Route:** `/clients/tiktok-fbt/accounts`
**Phase:** 1
**Purpose:** Add/remove FBT account connections. **Completely separate from `/clients/tiktok/accounts` (the seller page).**
**Reuses:** `<AccountContainer>`, `<Card>`, `<AddAccountModal>`, `<EditAccountModal>` patterns from existing TikTok accounts. Cloned visually, points at TikTokFbtService.

**Layout:**

```
TikTok FBT Accounts                                    [+ Connect FBT account]

┌──── Card: Acme Ltd — UK FBT ─────────────────────────────────────────┐
│  Region: GB              Shop ID: 7290xxxxxxx                          │
│  Status: ● Connected     Last verified: 2026-05-12 09:14               │
│  Sync orders: ✓   Sync inventory: ✓   Publish to WMS: ✓                │
│  Linked seller account: Acme Ltd — Seller (id 142)  [Change link]      │
│  [View capabilities] [Disconnect] [Sync now]                           │
└─────────────────────────────────────────────────────────────────────────┘
```

**Connect flow (most important UX moment in Phase 1):**

```
1. Click "+ Connect FBT account"
   → Modal opens with: shop region dropdown (GB | US), continue button.

2. Click "Continue"
   → Redirect to TikTok OAuth (separate redirect URI from seller flow)
   → User authorises FBT shop on TikTok side
   → TikTok redirects back to /api/tiktok-fbt/auth/callback
   → tiktok-fbt-microservice exchanges code for tokens
   → Stores in fbt_accounts (encrypted)
   → Verifies FBT capabilities via /authorization/202309/shops
   → If shop is NOT enrolled in FBT: shows error "This shop is not enrolled in TikTok FBT. Apply here: [TikTok link]"
   → If enrolled: redirects to /clients/tiktok-fbt/accounts with success toast

3. After connect, an interstitial asks:
   "Is this the same TikTok shop as one of your existing seller connections?"
   - Dropdown of seller accounts (read from /api/tiktok/accounts — different service)
   - "None of the above" option
   - Stores linked_seller_account_id in fbt_accounts (informational only)
```

**Edge cases:**
- Same TikTok shop already connected as FBT → unique constraint blocks → "Already connected as FBT" message.
- User has no seller accounts in TikTok → skip the link step, link can be added later via `[Change link]`.
- Disconnect while orders in flight → row soft-deletes; FBT orders already in core-service WMS continue to be tracked. New inventory polls stop.

---

### P1.3 — FBT Inventory

**Route:** `/clients/tiktok-fbt/inventory`
**Phase:** 1
**Purpose:** SKU-level mirror of TikTok FBT stock. Most-used page after launch.
**Reuses:** Custom table primitives (`<TableContainer>`, `<TableHeader>`, etc.), `<TextFilter>`, `<OperatorFilter>`, `<OptionsSelector>`, `<BulkActionBar>`, `<PreviewableImage>`. Mirrors Amazon `fba-inventory.jsx` visual pattern.

**Layout:**

```
FBT Inventory                                  Last synced: 7m ago [Refresh]

[Region: UK ▾]  [Account: All ▾]  [Health: All ▾]  [Search SKU…]

┌─────┬───────┬─────────────┬──────┬──────┬─────┬───────┬───────┬─────────┬─────────┐
│  ☐  │ Image │ SKU / Name  │Fulfil│Inbnd │Resv │ Unfull│ Days  │ Health  │ WMS own │
│     │       │             │-lable│      │     │       │ cover │         │ stock   │
├─────┼───────┼─────────────┼──────┼──────┼─────┼───────┼───────┼─────────┼─────────┤
│  ☐  │ [img] │ WIDGET-RED-L│  42  │ 120  │  3  │   1   │  6.5  │ ● Red   │  85     │
│  ☐  │ [img] │ WIDGET-BLU-L│ 200  │   0  │  8  │   0   │ 41.7  │ ● Green │   0     │
│  …                                                                                 │
└─────┴───────┴─────────────┴──────┴──────┴─────┴───────┴───────┴─────────┴─────────┘

[3 selected]  [Mark for replenishment]  [Export CSV]
```

The "WMS own stock" column joins to core WMS variation actualQuantity — gives the merchant context on whether they have stock to ship into FBT.

**User actions:**
- Click row → drawer with full breakdown (inbound: working/shipped/receiving, etc.), 30-day sales line chart.
- Filter by health → URL syncs (`?health=red`).
- Bulk-mark for replenishment → opens Phase-2 wizard with pre-selected SKUs.
- Export CSV → client-side via `papaparse`.

**Edge cases:**
- New account, no data yet → empty state with countdown to next 15-min cron.
- SKU in FBT inventory but NOT in WMS catalogue → flagged "Unmapped" with action to create a catalogue entry.
- SKU in WMS catalogue with `variation_listing.fulfilment_mode='fbt'` but no inventory in TikTok yet → shown with all-zero counts and badge "Awaiting first inbound".

---

### P1.4 — FBT Orders

**Route:** `/clients/tiktok-fbt/orders`
**Phase:** 1
**Purpose:** FBT-specific lens on orders that flow through `createdVia='tiktok_fbt'`.
**Reuses:** Existing order table pattern from `amazon/orders.jsx`.

```
FBT Orders                                            [Sync now]  [Export CSV]

[Region ▾]  [Account ▾]  [Status ▾]  [Date range]  [Search…]

┌──────────────┬──────────────┬─────────┬──────────┬──────────┬───────────┬────────────┐
│ Order #      │ Date         │ SKUs    │ Total    │ Status   │ Tracking  │ Customer   │
├──────────────┼──────────────┼─────────┼──────────┼──────────┼───────────┼────────────┤
│ TK-12345     │ 12 May 09:14 │ 2 items │ £42.00   │ Shipped  │ AB123... ↗│ John Doe   │
│ TK-12346     │ 12 May 09:30 │ 1 item  │ £15.00   │ Await…   │  —        │ Jane Smith │
└──────────────┴──────────────┴─────────┴──────────┴──────────┴───────────┴────────────┘
```

Every row carries a small "Fulfilled by TikTok" badge.

These same orders also appear in `/clients/order` (the central view) with the same badge. The FBT-specific page is a focused subset.

**User actions:**
- Click row → drawer with full detail, item list, customer address (masked per PII rules), TikTok-side fulfilment status timeline.
- `[Sync now]` → on-demand poll (rate-limited).
- `[Export CSV]` → current filtered set.

---

### P2.1 — Inbound Shipments

**Route:** `/clients/tiktok-fbt/inbound` (list) and `/clients/tiktok-fbt/inbound/[id]` (detail)
**Phase:** 2
**Purpose:** Plan, create, track inbound shipments to TikTok FCs.
**Reuses:** `<TableContainer>` for the list, MUI Stepper for the wizard (built bespoke from existing form-section components).

**List view layout:**

```
Inbound Shipments                                          [+ Create new]

[Status ▾]  [Region ▾]  [Search shipment# or SKU…]

Shipment #     Created      Destination   Units  Status            Tracking
INB-2026-0001  10 May       FBT UK (LDN)  150    Shipped           AB123 ↗
INB-2026-0002  11 May       FBT UK (MAN)   80    Submitted         —
INB-2026-0003  12 May       FBT UK (LDN)  220    Draft             —
```

**Wizard (4 steps):**

1. **Choose products & quantities.** Source warehouse dropdown (merchant's own warehouses from core WMS). Add-row interface for SKU + quantity. Validates `units_planned ≤ available stock in source warehouse`.

2. **Choose destination FC.** Calls `GET /api/tiktok-fbt/accounts/{accountId}/fcs` (which proxies to TikTok's logistics endpoint). Merchant picks one, or "Let TikTok decide".

3. **Review & submit.** Summary screen → `[Submit plan]` calls `POST /api/tiktok-fbt/inbound` → TikTok plan API → plan_id stored.

4. **Print labels & ship.** Calls `GET /api/tiktok-fbt/inbound/{id}/label` → PDF link. Merchant inputs tracking number → `[Mark as shipped]` → status `in_transit`. Wizard exits to detail page.

**Detail page:** header with status pill, line items table with `planned / shipped / received / damaged / short`, status timeline, webhook-driven real-time updates.

---

### P3.1 — Fees & Reimbursements

**Route:** `/clients/tiktok-fbt/fees`
**Phase:** 3
**Purpose:** Financial reconciliation.
**Reuses:** MUI X Charts (bar chart), `<StatCard>`, custom table.

```
Fees & Reimbursements                                          [Export CSV]

[Date range: Last 30 days ▾]  [Account ▾]  [Type ▾]

┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│ Total fees │ Pick & Pack  │  Storage     │ Reimbursed   │
│  £2,840    │   £1,720     │   £620       │   £312       │
└────────────┘ └────────────┘ └────────────┘ └────────────┘

[Bar chart: fees by type, by week]

Fee history table — fee_type, sku, order, amount, status …
Reimbursement claims table — reason, order, amount, status …
Discrepancy alerts panel
```

---

## Workflow summaries

**Connecting an FBT account:** merchant goes to `/clients/tiktok-fbt/accounts`, clicks `[+ Connect FBT account]`, picks region, completes OAuth in a popup, optionally links to an existing seller account. First inventory sync runs within 15 min; first orders sync within 5 min.

**Checking stock:** Dashboard → reorder alert → click red SKU → land on Inventory filtered to that SKU → click `[Plan]` (Phase 2) → wizard pre-populates.

**Receiving an FBT order:** Buyer places order on TikTok. TikTok webhook → tiktok-fbt-microservice. Service fetches full order, inserts into `fbt_orders`, publishes `tiktok_fbt.to.wms.order_created`. Core-service consumes, creates Order with `createdVia='tiktok_fbt'`, OrderFulfillment(`type='fbt'`, warehouse=virtual_fbt_uk). Virtual warehouse inventory deducts. **No picker is shown this order.** TikTok ships. Webhook `PACKAGE_UPDATE` arrives. Tracking number updates.

**Reconciling fees (Phase 3):** 03:15 cron pulls statement, inserts rows into `fbt_fees`, publishes per-line `tiktok_fbt.to.wms.fee_incurred`, core-service inserts `marketplace_charges`. Merchant opens `/clients/tiktok-fbt/fees` and reviews.

---

## Component reuse map

| New screen | Reuses |
|---|---|
| Dashboard | `<StatCard>`, MUI X LineChart, `<PageHeadingWithBreadcrumb>`, `<CounterCard>` |
| FBT Accounts | `<AccountContainer>`, `<Card>`, `<AddAccountModal>` (cloned with FBT-specific OAuth target), `<EditAccountModal>` |
| FBT Inventory | `<TableContainer>`, `<TableHeader>`, `<TableRow>`, `<TextFilter>`, `<OperatorFilter>`, `<OptionsSelector>`, `<BulkActionBar>`, `<PreviewableImage>` |
| FBT Orders | Existing order table pattern + `<OrderStatusChip>` + new badge |
| Inbound Wizard | MUI Stepper, form-section components from `create-product/`, `<ConfirmActionModal>`, `<RequiredAsterisk>` |
| Fees | MUI X BarChart, `<StatCard>`, table primitives |

**New components to build (3 total):**
1. `<FbtHealthChip>` — green/amber/red pill with days-of-cover tooltip.
2. `<FulfilledByTikTokBadge>` — appended to order rows. Trivial.
3. `<InboundShipmentStepper>` — wizard container (Phase 2 only).

---

## Auth / API gateway considerations

- Frontend axios client uses the same JWT (existing user session). No new auth.
- Base URL pattern: `/api/tiktok-fbt/*` (gateway proxies to localhost:5018).
- Same `X-Client-Id` header pattern as existing services.
- Webhook endpoints (`/api/tiktok-fbt/webhooks`) are unauthenticated but verify HMAC signature via cloned `WebhookSignatureGuard`.

---

## A11y & responsive

- Keyboard-navigable tables (existing primitives support this).
- Mobile: existing `<ResponsiveTableWrapper>` and `<MobileCardView>` apply.
- Dark/light mode via `useThemeStore()` mode flag.

No new design system work needed.
