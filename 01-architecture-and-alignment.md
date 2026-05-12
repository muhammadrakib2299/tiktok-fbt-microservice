# 01 — Architecture & Alignment

> FBT lives **inside the existing tiktok-microservice** as a new `src/modules/fbt/` submodule. One service, one DB, one app credential set, one OAuth flow. FBT is a *capability* of an existing TikTok shop, not a separate connection.

---

## 1. Service topology — before vs after

### Before (today)

```
Frontend → Gateway → tiktok-microservice (port 5011)
                       ↓
                   tiktok_microservice DB
                       ↓
                   RabbitMQ wms_multichannel ← → core-service
```

### After

```
Frontend → Gateway → tiktok-microservice (port 5011, UNCHANGED)
                       ├── src/modules/accounts/         (existing — gains 2 endpoints)
                       ├── src/modules/orders/           (existing — gains fulfilment_mode routing)
                       ├── src/modules/products/         (existing — UNCHANGED)
                       ├── src/modules/inventory/        (existing — UNCHANGED, seller-only)
                       ├── src/modules/webhooks/         (existing — gains FBT event dispatch)
                       ├── src/modules/tiktok-auth/      (existing — may extend OAuth scopes)
                       ├── src/modules/activity-logs/    (existing — UNCHANGED)
                       └── src/modules/fbt/              (NEW — all FBT-specific logic)
                              ├── fbt-accounts/           (verify shop eligibility, toggle on/off)
                              ├── fbt-inventory/          (15-min poll)
                              ├── fbt-orders/             (lens over tiktok_orders WHERE fulfilment_mode='fbt')
                              ├── fbt-dashboard/          (aggregates for dashboard tiles)
                              ├── fbt-webhooks/           (FBT-specific event handlers)
                              ├── fbt-inbound/            (Phase 2)
                              ├── fbt-fees/               (Phase 3)
                              └── fbt-returns/            (Phase 3)
                       ↓
                   tiktok_microservice DB (same DB)
                       (existing tables UNCHANGED at behaviour level — only additive columns
                        on `accounts` and `tiktok_orders`; new fbt_* tables alongside)
                       ↓
                   RabbitMQ wms_multichannel  ← new routing keys: tiktok_fbt.*
                                              ← existing routing keys: tiktok.* (UNCHANGED)
                       ↓
                   core-service
                       ├── learns one new channel slug 'tiktok_fbt'
                       ├── learns warehouse_type='fbt_tiktok'
                       └── learns order_fulfillments.fulfillment_type='fbt'
```

One service, one DB, two integration modes routed by a single boolean (`accounts.fbt_enabled`) and a single enum (`tiktok_orders.fulfilment_mode`).

---

## 2. Why this architecture

We previously considered a separate `tiktok-fbt-microservice`. Reasoning for the current choice:

| Concern | Separate service | Submodule (chosen) |
|---|---|---|
| One TikTok shop with seller + FBT both | 2 account rows in 2 DBs, soft-linked | 1 account row, 1 toggle |
| OAuth | Separate flow | Reuse existing |
| Token refresh cron | Duplicated | Reuses existing |
| HMAC signing, AES-GCM | Cloned (~1500 lines drift risk) | Reused |
| Webhook endpoint | Two endpoints | One endpoint, dispatcher inside |
| Sidebar | Two top-level entries | One TikTok parent, conditional sub-group |
| Order page | Two URLs to look at | One central order page, filter chip |
| Code reviews touch two repos | Yes | No |
| Engineer mental model | "Are these the same shop or different?" | One shop, one toggle |

The submodule approach has tighter integration coupling but a smaller blast radius for changes, simpler operations, and a UX that matches how TikTok itself models FBT (programme enrolment, not separate account).

---

## 3. The conditional sidebar — three views

Two symmetric per-account flags (`fbm_enabled`, `fbt_enabled`) drive **three possible sidebar layouts** under the "TikTok Shop" parent:

1. **FBM-only client** — only General TikTok submodules visible (today's behaviour)
2. **FBT-only client** — only FBT submodules visible
3. **Hybrid client** — both groups, headed "General" and "FBT"

The "Accounts" entry is always rendered regardless of mode because that's where both fulfilment toggles live — an FBT-only client still needs to reach it to re-enable FBM, and an FBM-only client uses it to flip FBT on for the first time.

**Frontend logic** (full builder in `04-pages-and-workflows.md` §1):

```js
// In ClientSidebar.jsx — pseudo-code
const accounts = useTiktokAccountsStore().accounts;
const anyFbmEnabled = accounts.some(a => a.fbm_enabled === true)
                   || clientHasHistoricalFbmOrders;
const anyFbtEnabled = accounts.some(a => a.fbt_enabled === true);

const tiktokGroup = {
  name: 'TikTok Shop',
  children: [
    { name: 'Accounts', link: '/clients/tiktok/accounts' },              // always
    ...(anyFbmEnabled && anyFbtEnabled
        ? [{ name: '— General —', isHeading: true }] : []),
    ...(anyFbmEnabled ? [
      { name: 'Active products',  link: '/clients/tiktok/active-products' },
      { name: 'Pending products', link: '/clients/tiktok/pending-products' },
      { name: 'Deactivated',      link: '/clients/tiktok/deactivated-products' },
    ] : []),
    ...(anyFbtEnabled ? [
      { name: '— FBT —', isHeading: true },
      { name: 'FBT Dashboard',         link: '/clients/tiktok/fbt/dashboard' },
      { name: 'FBT Inventory',         link: '/clients/tiktok/fbt/inventory' },
      { name: 'FBT Orders',            link: '/clients/tiktok/fbt/orders' },
      { name: 'FBT Inbound shipments', link: '/clients/tiktok/fbt/inbound' },
      { name: 'FBT Fees',              link: '/clients/tiktok/fbt/fees' },
    ] : [])
  ]
};
```

Re-evaluates whenever accounts are refetched (after connect, after FBT enable/disable, after FBM disable/re-enable). No extra API calls — `accounts` is already in the store; `clientHasHistoricalFbmOrders` rides along on the same payload as a precomputed boolean.

**Why `clientHasHistoricalFbmOrders` is part of the FBM aggregate:** even when every account is flipped to FBT-only, residual FBM orders / returns may still exist in `tiktok_orders` (`fulfilment_mode='fbm'`). Hiding the General group entirely would orphan that data. The boolean keeps the group visible until those rows are archived.

**Backend gating** (defence in depth, both directions):

```typescript
@UseGuards(ClientAuthGuard, FbtCapabilityGuard)
@Controller('fbt')
class FbtController {
  // FbtCapabilityGuard returns 403 if no account for this clientId has fbt_enabled
}

@UseGuards(ClientAuthGuard, FbmCapabilityGuard)
@Controller('accounts')  // listings, products, seller orders
class AccountsController {
  // FbmCapabilityGuard returns 403 on product-listing routes if no account
  // has fbm_enabled. The /accounts CRUD itself stays unguarded so FBT-only
  // clients can still reach the toggles.
}
```

Even if a merchant pokes at the URL directly, they get a clean 403 unless they have at least one account with the matching capability.

---

## 4. The "Enable FBT" flow

```
1. Merchant clicks [Enable FBT] toggle on account row.
   Frontend confirms with modal: "Verify FBT eligibility for shop X?"
   Merchant confirms.

2. Frontend POST /api/tiktok/accounts/:id/fbt/enable

3. accounts.service.enableFbt(accountId):
   a. Verify account belongs to this clientId (multi-tenant guard).
   b. Call FbtAccountsService.verifyShopEligibility(account):
        → tiktok-api.service.get('/authorization/202309/shops', accountId)
        → parse response for FBT capability flags
        → returns { eligible: true|false, capabilities: {...} }
   c. If NOT eligible:
        → throw 400 "Shop not enrolled in FBT — apply at <TikTok URL>"
        → frontend shows toast with link
   d. If eligible:
        → UPDATE accounts
             SET fbt_enabled = true,
                 fbt_capabilities = <jsonb from TikTok>,
                 fbt_enabled_at = NOW(),
                 fbt_last_verified_at = NOW()
           WHERE id = accountId
           -- publish_fbt_to_wms keeps its existing value (default TRUE for new rows)
        → call canPublishFbt(account) — if false, skip the RabbitMQ publish but
          still flip fbt_enabled. This is the fail-closed pattern: ops may have
          pre-emptively set publish_fbt_to_wms=false (e.g. during a consumer outage)
          and we must respect that even on the happy path.
        → publish RabbitMQ tiktok_fbt.to.wms.account_fbt_enabled
           (core-service creates virtual warehouse "TikTok FBT UK")
           — only if canPublishFbt() returned true
        → queue immediate first inventory poll (don't wait 15 min)
        → log activity row "FBT enabled"
        → return { success: true, capabilities, publishing: canPublishFbt(account) }
           — the publishing flag tells the frontend whether to show a banner like
             "FBT enabled, but admin has paused WMS publishing — data will start
              flowing once paused is lifted."

4. Frontend refetches account, sidebar re-renders with FBT sub-group visible.

5. Within 15 seconds: FBT inventory poll completes, FBT Inventory page populates.
```

The "Disable FBT" path is symmetric: set `fbt_enabled = false`, publish `tiktok_fbt.to.wms.account_fbt_disabled` (core-service marks virtual warehouse stale but keeps existing orders intact), log activity. Disabling does NOT delete historical data.

---

## 5. The fulfilment_mode routing — order's journey

```
1. TikTok webhook ORDER_STATUS_CHANGE → webhooks controller (one endpoint).
2. WebhookSignatureGuard verifies HMAC.
3. webhooks.service dispatches by payload:
     - Parse fulfillment_type field
     - 'FULFILLED_BY_TIKTOK' / 'FBT' → fulfilment_mode = 'fbt'
     - else                          → fulfilment_mode = 'fbm'

4. orders.service.upsertOrder():
     a. UPSERT tiktok_orders SET fulfilment_mode = computed_mode.
     b. Branch by mode:
        - fbm: publish tiktok.to.wms.order_created           (existing path)
        - fbt: publish tiktok_fbt.to.wms.order_created       (new path)

5. core-service consumes:
     - tiktok.to.wms.order_created      → existing handler, picker-bound fulfillment, created_via='tiktok'
     - tiktok_fbt.to.wms.order_created  → new handler, fulfillment_type='fbt', created_via='tiktok_fbt'

6. Where the orders show up:
   - Seller orders (created_via='tiktok')      → central /clients/order page (existing behaviour)
   - FBT orders   (created_via='tiktok_fbt')   → dedicated /clients/tiktok/fbt/orders page only
   The central order page query adds `WHERE created_via != 'tiktok_fbt'` so FBT orders are excluded.
   The picker queue already filters `WHERE fulfillment_type != 'fbt'` so FBT orders are skipped twice over.
```

For the merchant: FBT orders are invisible on the central order page and the picker queue. They live exclusively on the dedicated FBT Orders page where the merchant goes to check on TikTok's fulfilment status.

---

## 6. Hybrid SKU model

Same SKU listed as both FBM and FBT simultaneously. Mechanism:

- Core WMS `variations` table = single physical SKU
- Core WMS `variation_listings` table = one row per (channel × fulfilment_mode) listing
- Per-warehouse inventory: own warehouses for FBM stock, virtual `fbt_tiktok` warehouse for FBT stock

When TikTok ships an FBT order: virtual fbt_tiktok warehouse inventory decrements.
When the merchant ships an FBM order: their own warehouse decrements.

The Inventory page surfaces both side-by-side.

---

## 7. 15-min inventory lag mitigation

1. Poll every 15 min, stagger 30s per account.
2. Optimistic decrement on FBT order webhook arrival.
3. UI surfaces "Live inventory, refreshed every 15 minutes · last sync 7 min ago".
4. Webhook subscription for inventory-change events if TikTok publishes them (research item).
5. Reservation buffer: UI's "Available to sell" = `fulfillable − in_flight_orders`.

---

## 8. Fees & reimbursement flow (Phase 3)

```
03:15 daily fbt-statement-cron.service:
   GET /finance/202309/statements?date=…
   parse line items → insert fbt_fees rows
   publish tiktok_fbt.to.wms.fee_incurred per charge
   publish tiktok_fbt.to.wms.reimbursement_received per credit
core-service consumers insert marketplace_charges / marketplace_reimbursements
```

---

## 9. Returns flow (Phase 3)

```
TikTok webhook FBT_RETURN_RECEIVED → webhooks.service routes to fbt-webhooks
fbt-returns.service:
   - insert fbt_returns row
   - publish tiktok_fbt.to.wms.return_received
core-service: insert/update return_requests with marketplace='tiktok_fbt'
```

---

## 10. Where seller flow vs FBT flow diverge inside the same service

| Concern | Seller flow | FBT flow |
|---|---|---|
| Inventory direction | WMS → TikTok (push listing qty) | TikTok → WMS (poll FBT FC stock) |
| Order fulfilment | Merchant picks/packs, pushes tracking | TikTok ships; we receive tracking |
| Pick/pack queue | Order goes to picker | Order excluded from picker queue |
| Cron cadence | Orders 15min, products 2h | Inventory 15min, orders 5min defence-in-depth, statement daily |
| RabbitMQ key | `tiktok.*` and `wms.to.tiktok.*` | `tiktok_fbt.*` and `wms.to.tiktok_fbt.*` |
| Sidebar visibility | Always (any TikTok account) | Only if `fbt_enabled` somewhere |

---

## 11. Multi-tenancy boundaries

- `accounts` already has `client_id` as leading filter.
- New `fbt_*` tables follow the same pattern: `client_id` is column 1, indexed.
- A merchant can have multiple shops; each shop is one accounts row; each can independently have `fbt_enabled` true or false.
- The conditional sidebar evaluates across all of one merchant's accounts.

---

## 12. Failure-mode handling

| Failure | Response |
|---|---|
| TikTok 429 on FBT endpoint | Existing retry logic. Metric `fbt.rate_limited`. |
| 15-min inventory poll fails | Next poll picks up. Stale-snapshot warning after 30 min. |
| Order webhook missed | 5-min reconciliation cron polls and backfills. |
| Merchant disables FBT mid-flight | `fbt_enabled = false`. New polls stop. In-flight orders resolve. UI banner. |
| Statement report unavailable (P3) | Idempotent cron. Discrepancy alert after 7 days. |
| Token refresh fails | Existing seller flow already handles. Activity log "Token expired". |

Full failure matrix in [06-deployment-and-observability.md](./06-deployment-and-observability.md).
