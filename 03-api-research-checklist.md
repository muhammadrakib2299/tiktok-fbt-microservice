# 03 — TikTok FBT API Research Checklist

> This file is a **week-1 critical-path artefact**. Most of our risk lives here: TikTok FBT API documentation is sparse compared to the seller API. We must verify every endpoint, payload shape, and rate limit against TikTok's live sandbox before we commit to the schemas in [02-data-model-changes.md](./02-data-model-changes.md).
>
> Owner: backend lead. Output: this file fully filled in with verified examples by end of week 1.

---

## 1. Apply for access — day 1 actions

| Step | Action | Owner | Status |
|---|---|---|---|
| 1 | Apply for TikTok FBT seller programme via UK Seller Center → Fulfilment → "Apply for FBT" | merchant or you with their login | ☐ |
| 2 | Request TikTok Shop Partner API expanded scopes — specifically the FBT scope group (verify exact scope names in TikTok Partner Center; do NOT assume from prior reading) | dev | ☐ |
| 3 | Spin up a TikTok sandbox shop with FBT capabilities. Sandbox may not support all FBT endpoints — confirm coverage with TikTok support | dev | ☐ |
| 4 | Verify which TikTok app credentials we use: same `TIKTOK_APP_KEY`/`TIKTOK_APP_SECRET` as seller integration, or a separate registered app | dev | ☐ |
| 5 | If separate app required: register new app, store credentials in `accounts.app_key`/`app_secret` (already encrypted via AES-GCM in existing schema) | dev | ☐ |

> ⚠️ **Memory check**: per existing memory `feedback_etsy_oauth_scopes.md`, OAuth scope names are not always what they appear in third-party guides. Apply the same rigour to TikTok FBT scopes — test each scope individually against the TikTok auth flow.

---

## 2. Endpoints to discover and verify

For each endpoint below, capture in a separate `endpoint-payloads.md` (to be created in week 1):
- Full URL path with API version
- Required scopes
- Request shape (headers, query params, body)
- Sample response (sandbox)
- Rate limit
- Error codes observed

### 2a. Authorisation & shop discovery

| Purpose | Likely endpoint (verify) | Notes |
|---|---|---|
| List authorised shops, discover FBT capability per shop | `GET /authorization/202309/shops` (existing in our codebase) | Check if the response includes an FBT-enabled flag, or if FBT is a separate `service_type` field per shop |
| Discover available FCs/warehouses per region | `GET /logistics/202309/warehouses` (we already call this for FBM tracking) | Confirm FBT FCs appear here and are distinguishable from merchant warehouses |

### 2b. Inventory

| Purpose | Likely endpoint | Notes |
|---|---|---|
| Get FBT inventory snapshot per shop | `GET /fulfillment/202309/inventory` or similar — **verify exact path** | Must support pagination. Document refresh cadence. |
| Inventory adjustment history | Possibly under `/fulfillment/202309/inventory/movements` | For audit, optional |

### 2c. Orders

| Purpose | Likely endpoint | Notes |
|---|---|---|
| List FBT orders | Likely the same `GET /orders/202309/orders` we use for FBM, with a `fulfillment_type=FBT` filter | Confirm filter syntax in sandbox |
| Get FBT order details, including TikTok-assigned tracking | `GET /orders/202309/orders/{order_id}` | Verify tracking/dispatch fields populated by TikTok automatically |
| Cancel FBT order | May or may not be permitted by TikTok | Likely not permitted post-dispatch; verify |

### 2d. Inbound shipments (Phase 2)

| Purpose | Likely endpoint | Notes |
|---|---|---|
| Create inbound shipping plan | `POST /fulfillment/202309/inbound_shipments/plans` (placeholder — verify) | Returns plan_id; payload includes SKUs, quantities, source address |
| Confirm inbound plan | `POST /fulfillment/202309/inbound_shipments/plans/{plan_id}/confirm` | Locks in selected FC |
| Generate inbound label | `GET /fulfillment/202309/inbound_shipments/{shipment_id}/labels` | Returns PDF or label URL |
| Get inbound shipment status | `GET /fulfillment/202309/inbound_shipments/{shipment_id}` | Status: submitted, in_transit, partially_received, received |
| Cancel inbound shipment | `POST /fulfillment/202309/inbound_shipments/{shipment_id}/cancel` | Pre-shipping only |

### 2e. Fees & financials (Phase 3)

| Purpose | Likely endpoint | Notes |
|---|---|---|
| Statement / settlement report | `GET /finance/202309/statements` or `/finance/202309/order_statements` | Daily and monthly cadences likely differ |
| FBT-specific fee breakdown per order | May be inline in statement, or under `/finance/202309/orders/{order_id}/fees` | Verify storage fee cadence (monthly?) |
| Reimbursements | Likely also in statements, filtered by line_type | Verify dispute-claim API existence (may be web-only) |

### 2f. Returns

| Purpose | Likely endpoint | Notes |
|---|---|---|
| Get returns | `GET /return_refund/202309/returns` or under `/aftersale/` | Verify path and filtering by fulfilment_type |
| Get return details + condition assessment | `GET /return_refund/202309/returns/{return_id}` | Look for fields: condition, restockable, reimbursement_eligible |

### 2g. Webhooks for FBT

We already subscribe to `ORDER_STATUS_CHANGE`, `PACKAGE_UPDATE`, `PRODUCT_STATUS_CHANGE`, `RETURN_REQUEST`. Need to determine if TikTok publishes additional FBT-specific webhook event types:

| Possible event | What we'd do |
|---|---|
| `FBT_INBOUND_RECEIVED` | Update `tiktok_fbt_inbound_shipments.status='received'`; trigger inventory poll |
| `FBT_INVENTORY_CHANGE` | Trigger out-of-cycle inventory poll for affected SKU |
| `FBT_ORDER_SHIPPED` | Update order tracking immediately (don't wait for 5-min order cron) |
| `FBT_FEE_INCURRED` | Real-time fee insertion (vs daily statement poll) |

If TikTok publishes none of these → we rely entirely on polling. Acceptable for Phase 1.

---

## 3. Sandbox onboarding plan — week 1 day-by-day

**Day 1 (Mon):** Apply for FBT programme, sandbox shop, scope expansion. While waiting, read TikTok Partner docs for FBT in full.

**Day 2 (Tue):** Build a throwaway Node script (or use Postman collection) that:
- OAuths a sandbox shop
- Calls `GET /authorization/202309/shops`
- Calls `GET /logistics/202309/warehouses`
- Captures every field returned

**Day 3 (Wed):** Test FBT-specific endpoints listed in §2b–§2g, one at a time. Document each response into `endpoint-payloads.md`.

**Day 4 (Thu):** Try error cases — invalid SKU, oversell, out-of-region — to catalogue error codes. We need these for our `sync_error` columns.

**Day 5 (Fri):** Decision point: do payloads match our schema in [02-data-model-changes.md](./02-data-model-changes.md)? If yes → proceed to week 2 (cron + service skeletons). If no → revise schema, push timeline by ≤1 week.

---

## 4. Rate-limit budget

Existing TikTok-microservice already implements a 300ms minimum gap between inventory writes. For FBT, we add:

| Cron | Frequency | Max accounts × pages per run | Per-account stagger |
|---|---|---|---|
| FBT inventory poll | every 15 min | accounts × pages_per_account | 30s |
| FBT orders poll | every 5 min | accounts × ~50 orders/page | 10s |
| FBT inbound status poll | every 30 min | only "in_transit" shipments | 30s |
| FBT statement poll | daily 03:15 | accounts | 60s |

**Budget assumption:** TikTok FBT API rate limit is at least 60 req/min per shop (consistent with seller API documented limit). Verify in week 1.

If the limit is lower, fall back strategy:
- Inventory: poll only the SKUs with non-zero `fulfillable_quantity` (skip cold inventory after 30d of zero movement).
- Orders: rely more heavily on webhooks; reduce poll to every 15 min as fallback reconciliation.

---

## 5. Webhook signature & idempotency

Existing `WebhookSignatureGuard` already does HMAC-SHA256 verification with the account's `app_secret`. New webhooks for FBT inherit this — no security work needed.

Idempotency contract for **all** FBT-relevant webhook handlers:

```typescript
const eventKey = `${eventType}:${shop_id}:${event_id || hash(body)}`;
const seen = await this.idempotencyStore.checkAndSet(eventKey, ttl=86400);
if (seen) return { status: 200, body: { duplicate: true } };
// ...process...
```

Add this to the existing webhook handler. Phase 1 must not double-create FBT orders when TikTok retries a webhook.

---

## 6. Known unknowns — escalation items

These need answers from TikTok support directly. They are not in public docs:

1. **Does TikTok FBT sandbox support inbound shipment label generation?** Likely no — most marketplaces sandbox only allows planning, not real label printing. If true, Phase 2 needs a manual-fallback test plan.
2. **Storage fee calculation cadence and exact formula** — per-cubic-cm-day rate varies by region and FC. Required for Phase 3 reconciliation.
3. **Long-term storage fee threshold** — Amazon FBA is 180d; TikTok FBT's threshold is undocumented in community write-ups. Verify.
4. **Return-to-merchant option** — does TikTok offer "return to merchant" for unsellable FBT returns, or only destroy? Affects our return_destination enum.
5. **FBT-eligible categories** — some product categories may not qualify for FBT (e.g., hazardous goods). We need the list to validate `variation_listings` creation.
6. **Bulk inventory adjustment** — if a merchant wants to mark FBT inventory as "do not sell", is there an API or only Seller Center? Affects Phase 1 surface.

---

## 7. Output of this exercise

By end of week 1:

- [ ] `endpoint-payloads.md` filled in with verified examples for every endpoint in §2.
- [ ] Confirmed rate limits in `rate-limits.md`.
- [ ] Decision: do we need schema changes vs [02-data-model-changes.md](./02-data-model-changes.md)? Documented in `schema-deltas.md`.
- [ ] Sandbox merchant account fully configured and accessible to the team.
- [ ] Production app credentials path defined (separate registration or shared).
