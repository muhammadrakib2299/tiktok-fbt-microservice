# 01 — Architecture & Alignment

> Two TikTok-flavoured microservices, side by side. They don't talk to each other. They both talk to core-service via RabbitMQ. They both read WMS variations from core DB. They never share schemas or row data.

---

## 1. Service topology — before vs after

### Before (today)

```
Frontend (Next.js)
        │
        ▼
Gateway (port 9000) ──┬──► core-service (5000)                  ──► WMS DB (combosoft_wms_db / wms_v2)
                      ├──► tiktok-microservice (5011)            ──► tiktok_microservice DB
                      │       (seller flows only)                    └─► reads WMS DB (read-only)
                      │
                      ├──► ebay (5001), shopify (5008), amazon (?), onbuy (4002),
                      │    wowcher (5006), fruugo (4000), etsy (5013), ...
                      │
                      └─── RabbitMQ wms_multichannel (topic exchange, shared by all services)
```

### After

```
Frontend (Next.js)
        │
        ├── /clients/tiktok/*       (existing pages, unchanged)
        └── /clients/tiktok-fbt/*   (6 new pages)
        │
        ▼
Gateway (port 9000)
        │
        ├──► tiktok-microservice (5011)         tiktok_microservice DB     (UNCHANGED)
        │    /api/tiktok/*                        seller accounts, products, orders, inventory
        │
        ├──► tiktok-fbt-microservice (5018)     tiktok_fbt_microservice DB (NEW)
        │    /api/tiktok-fbt/*                    fbt_accounts, fbt_products, fbt_orders,
        │                                         fbt_inventory, fbt_inbound_shipments,
        │                                         fbt_fees, fbt_activity_logs, fbt_webhook_events
        │
        ├──► core-service (5000)
        │
        └─── RabbitMQ wms_multichannel (one exchange, two topic families)
             tiktok.*                            (existing — unchanged)
             tiktok_fbt.*                        (new)
```

Two services, two databases, one shared message broker, one shared core-service.

---

## 2. Why a separate microservice (and not a new module inside the existing one)

The user picked full separation. Concrete benefits and costs:

**Benefits:**
- **Zero regression risk on existing seller flow.** The seller integration is production code serving real merchants today. A schema migration on `accounts` or `tiktok_orders`, even additive, is a non-zero risk. Full isolation eliminates that risk.
- **Independent deployment cadence.** FBT bugs don't ship together with seller bugs. Roll back one without affecting the other.
- **Independent rate-limit budget.** Each service has its own outbound HTTP client. Separate retry queues. TikTok seller traffic doesn't compete with FBT polls for connection slots.
- **Easier ownership.** A team or sub-team can own FBT end-to-end without needing review approval on the seller codebase.
- **Cleaner audit logs.** Service name in every log line — no need to filter by `account_type` field.

**Costs (accepted):**
- **Cloned code.** OAuth flow, AES-GCM utility, HMAC signing, token refresh cron, webhook signature guard, RabbitMQ scaffolding — all duplicated. Estimated ~1500 lines of cloned code.
- **Drift risk.** A security fix to HMAC signing in one service can be forgotten in the other. Mitigation: documented file-pair list in [07-risks-and-open-questions.md](./07-risks-and-open-questions.md) §B7.
- **Operational surface doubled.** Two `.env` files, two deploy targets, two Sentry projects (or two tags inside one), two cron schedules. Existing memory `reference_port_allocation.md` needs updating to add 5018.
- **One more service for the gateway to health-check.** Trivial.

This trade-off is appropriate for a B2B WMS where some clients need only seller, some need only FBT, and some need both. The separation maps to the real business model.

---

## 3. The "virtual warehouse" concept (unchanged from previous design)

A TikTok FBT fulfilment centre is not a physical warehouse the merchant owns, but for inventory accounting we model it as one — a **virtual warehouse** in core WMS.

Each merchant who connects an FBT account gets one row inserted into `warehouses`:

```
Warehouse {
  id: 99,
  clientId: <merchant>,
  name: "TikTok FBT UK",
  warehouseType: 'fbt_tiktok',
  externalMarketplace: 'tiktok_fbt',          -- matches Channel.channelSlug
  externalAccountId: <fbt_accounts.id>,        -- soft ref to fbt_accounts row in the new service's DB
  externalRegion: 'GB',
  externalWarehouseId: <TikTok's FC ID if known>,
  externalWarehouseConfig: { ... }
}
```

When an FBT order arrives, core-service deducts from `warehouse_id=99`. The merchant's own warehouses are untouched. When the inventory poll runs in `tiktok-fbt-microservice` and publishes `tiktok_fbt.to.wms.inventory_synced`, core-service updates the virtual warehouse's reported stock.

---

## 4. Hybrid-SKU listing model (unchanged from previous design)

Same SKU can be listed on TikTok as **both** FBM (via the existing seller integration) and FBT (via the new integration). They appear to the buyer as one product with two delivery options.

The bridge is `variation_listings` in core WMS DB:

```
variation_listings
  id, client_id, variation_id (FK -> variations), channel_id (FK -> channels),
  account_id,                    -- which marketplace account owns this listing
                                 -- semantically: seller→accounts.id (in tiktok ms DB);
                                 --                FBT→fbt_accounts.id (in tiktok-fbt ms DB)
                                 -- stored as a plain INT, not enforced FK (cross-DB)
  external_listing_id,           -- TikTok product_id
  external_sku_id,               -- TikTok sku_id
  fulfilment_mode,               -- 'fbm' | 'fbt'  (kept in WMS as a routing flag)
  warehouse_id,                  -- which warehouse this listing draws from
  is_active, last_synced_at, external_metadata
```

Indices: `(client_id, channel_id, account_id, external_sku_id)` — fast order-to-variation lookup.

For an FBT listing: `channel_id` references the new TikTok FBT channel row, `fulfilment_mode='fbt'`, `warehouse_id` is the virtual fbt_tiktok warehouse.

For a seller listing: `channel_id` references the existing TikTok Shop channel row, `fulfilment_mode='fbm'`, `warehouse_id` is one of the merchant's own warehouses.

Both rows can coexist for the same `variation_id`. Buyer-side, that's the same SKU with two fulfilment options.

---

## 5. The 15-min inventory refresh — coping with lag

Same mitigations as before:
1. Poll every 15 min, stagger 30s per account.
2. Webhook subscription if TikTok publishes inventory-change events (research item).
3. UI surfaces "Last synced 7 min ago" prominently.
4. Optimistic decrement on FBT order webhook arrival.
5. UI's "Available to sell" = `fulfillable - in_flight_orders` (reservation buffer).

Implementation lives in the new service. Identical logic to what we'd have built inside the old service.

---

## 6. Fulfilment mode routing — the order's journey

Step-by-step. This is the single most important code path.

```
1. TikTok webhook ORDER_STATUS_CHANGE fires
   → POST /api/tiktok-fbt/webhooks  (gateway routes to tiktok-fbt-microservice)
   → WebhookSignatureGuard verifies HMAC using the fbt_accounts.app_secret
   → Webhook handler returns 200 immediately, queues async processing

2. tiktok-fbt-microservice fetches full order:
   GET https://open-api.tiktokglobalshop.com/orders/202309/orders/{order_id}
   with signing/timestamp via cloned tiktok-fbt-api.service.ts
   
3. Upsert fbt_orders row with full payload.
   (No co-existing fbt_orders for the same tiktok_order_id — unique constraint enforces.)

4. Publish to RabbitMQ:
   exchange = 'wms_multichannel'
   routing_key = 'tiktok_fbt.to.wms.order_created'
   payload = {
     wms_message_id,                    -- unique, used for idempotency
     source: 'tiktok_fbt',
     channelSlug: 'tiktok_fbt',
     marketplace: 'tiktok_fbt',
     eventType: 'order_created',
     clientId, accountId,                -- fbt_accounts.id (NOT accounts.id)
     orderId, orderCode,
     items: [{ external_sku_id, sku, quantity, price, ... }],
     shippingAddress: { ... },
     totals: { ... },
     orderData: {
       fulfilment_mode: 'fbt',           -- redundant with channelSlug but explicit
       tiktok_fulfilment_status,
       region, currency, ...
     }
   }

5. core-service consumer (FbtOrderConsumer) handles tiktok_fbt.to.wms.order_created:
   a. Idempotency check: SELECT id FROM orders WHERE wms_message_id = $1 LIMIT 1; skip if exists.
   b. Resolve variation_listings row:
      SELECT variation_id, warehouse_id
        FROM variation_listings
       WHERE client_id=$1
         AND channel_id=$ttok_fbt_channel
         AND account_id=$fbt_account_id
         AND external_sku_id=$external_sku
         AND fulfilment_mode='fbt'
         AND deleted_at IS NULL;
   c. If no listing row → log "Unmapped FBT SKU" warning, still create the order
      with NULL variation_id (so revenue tracking is preserved) and notify support.
   d. Create Order with createdVia='tiktok_fbt'.
   e. Create OrderProduct rows.
   f. Create OrderFulfillment with fulfillment_type='fbt', warehouse_id=<fbt virtual warehouse>,
      status='received', metadata={ tiktok_order_id, tiktok_fulfilment_status }.
   g. Atomically deduct virtual warehouse inventory (decrement WHERE variation_id = $1 AND warehouse_id = $2).
   h. DO NOT enqueue pick/pack. Do NOT assign a picker.
   i. ACK message.

6. UI:
   - Order appears in /clients/order with badge "Fulfilled by TikTok".
   - Order is filtered out of /clients/order/awaiting-pick (WHERE order_fulfillments.fulfillment_type != 'fbt').
   - Order also appears in /clients/tiktok-fbt/orders (lens specific to FBT).
```

---

## 7. Cross-DB references (no FKs across services)

A few values are stored without foreign-key enforcement because they cross database boundaries:

| Where | Field | Points to | Notes |
|---|---|---|---|
| core-service `warehouses.external_account_id` | INT | `tiktok_fbt_microservice.fbt_accounts.id` | informational; lets UI display "this warehouse belongs to merchant's FBT shop X" |
| core-service `variation_listings.account_id` | BIGINT | `fbt_accounts.id` OR seller `accounts.id` depending on `channel_id` | which DB it points to is determined by `channel_id` |
| core-service `marketplace_charges.marketplace_account_id` | BIGINT | `fbt_accounts.id` for `marketplace='tiktok_fbt'` | per phase 3 |
| `tiktok_fbt_microservice.fbt_accounts.linked_seller_account_id` | BIGINT, NULL | `tiktok_microservice.accounts.id` | optional UX hint: "this FBT shop is the same TikTok shop as your seller connection X". User sets this manually during onboarding. |

All cross-DB references are treated as soft refs — code that uses them must handle the case where the target row no longer exists (e.g., merchant disconnected the seller account). No `JOIN` queries across services. Where the UI needs combined data, it fetches from each service independently and merges client-side (or via a thin core-service aggregator).

---

## 8. Reimbursement & fees flow (Phase 3)

```
TikTok publishes settlement reports → daily for orders, weekly for storage fees.

tiktok-fbt-microservice fbt-fees-cron (03:15 UTC daily):
   GET https://open-api.tiktokglobalshop.com/finance/202309/statements?statement_date=YYYY-MM-DD
   For each line item:
      - parse fee_type: pick_pack | storage | return_processing | reimbursement
      - insert into fbt_fees
      - publish:
          if charge:        routing_key = 'tiktok_fbt.to.wms.fee_incurred'
          if reimbursement: routing_key = 'tiktok_fbt.to.wms.reimbursement_received'

core-service consumes:
   - inserts marketplace_charges or marketplace_reimbursements
   - matches by external_order_id where possible
   - flags unmatched lines for manual review

Display in /clients/tiktok-fbt/fees:
   - Fee breakdown by SKU, by month
   - Reimbursement claims pending
   - Discrepancy alerts (e.g., charged for storage but our inventory snapshot shows 0 units)
```

---

## 9. Returns flow

```
TikTok webhook FBT_RETURN_RECEIVED or RETURN_REQUEST
→ POST /api/tiktok-fbt/webhooks
→ tiktok-fbt-microservice fbt-webhooks handler:
   - parses return reason, condition (sellable/damaged/destroyed)
   - inserts fbt_returns row
   - publishes tiktok_fbt.to.wms.return_received

core-service:
   - upserts return_requests with marketplace='tiktok_fbt',
     marketplace_return_id=<TikTok return id>,
     return_destination='marketplace_fbt_warehouse'
   - if condition='sellable': do nothing — virtual warehouse will reflect on next 15-min inventory poll
   - if condition='damaged': do NOT increment our fulfillable; expect a reimbursement record from settlement cron
   - notifies merchant via in-app notification + email
```

Phase 1 = display only. Phase 3 = active reconciliation + dispute submission to TikTok.

---

## 10. Where this diverges from Amazon FBA

| Concern | Amazon FBA (existing) | TikTok FBT (new) |
|---|---|---|
| Microservice | Single `amazon-microservice` handles MFN+FBA | Two separate services: `tiktok-microservice` (seller), `tiktok-fbt-microservice` (FBT) |
| Fulfilment-channel field | `fulfillment_channel` ENUM on amazon_orders | Separate tables in separate DBs. No co-located enum. |
| Account separation | One `amazon_account` row per merchant; FBA vs MFN distinguished by per-order field | Two account rows in two different services |
| Publish flag default | `publish_fba_to_wms` defaults FALSE (FBA orders skipped) | We publish 100% of FBT orders (merchant needs visibility); pick/pack skip happens in core-service via `fulfillment_type='fbt'` filter |
| Listing model | Single Amazon listing per SKU | Hybrid via `variation_listings` (FBM listing + FBT listing co-exist for same SKU) |
| Velocity calc | Same pattern (7d/30d sales from order items) | Same pattern, separate fact table in separate DB |
| Reimbursement model | Not implemented | Built fresh in Phase 3 |

---

## 11. Multi-tenancy boundaries

Same model as the rest of WMS360:

- Every row in `tiktok_fbt_microservice` DB has `client_id` as its first WHERE-clause column.
- Indices lead with `client_id`.
- `fbt_accounts (client_id, shop_id)` is the unique key — a merchant cannot connect the same TikTok shop twice as FBT.
- A merchant can have a seller account AND an FBT account for the same TikTok shop. These are two rows in two different DBs, linked optionally via `fbt_accounts.linked_seller_account_id`.
- Per-tenant feature flag `tiktok_fbt` gates all FBT routes via `checkPackageFeature('tiktok_fbt')` middleware on the gateway side OR client-side route guards.

---

## 12. Failure-mode handling

| Failure | Response |
|---|---|
| TikTok FBT API returns 429 (rate limit) | Retry with exponential backoff (cloned logic). Metric tag `fbt.rate_limited`. |
| 15-min inventory poll fails mid-batch | Next poll picks up. Stale-snapshot warning after 2 missed polls (~30min). |
| Order webhook missed | 5-min reconciliation cron diffs TikTok order list vs `fbt_orders`, re-fetches missing. |
| TikTok deletes an FBT listing | Mark `variation_listings.is_active=false` (in core WMS) via a periodic reconciliation. Inventory count drops to 0 next poll. |
| Settlement report not yet available (P3) | Idempotent cron — re-runs daily. Discrepancy alert after 7 days. |
| Merchant disconnects FBT account | All FBT inventory snapshots for that account marked stale. UI shows "Account disconnected — reconnect to resume FBT". In-flight FBT orders continue to be tracked (existing rows are read-only after disconnect). |
| `tiktok-fbt-microservice` is down | Gateway circuit breaker opens after threshold. `/api/tiktok-fbt/*` returns 503. Existing tiktok-microservice (seller) is unaffected. |
| Cloned auth code diverges (security bug fixed in old service, not new) | Periodic audit (in [07](./07-risks-and-open-questions.md) §B7). Documented file-pair list. |

Full failure-mode matrix in [06-deployment-and-observability.md](./06-deployment-and-observability.md).
