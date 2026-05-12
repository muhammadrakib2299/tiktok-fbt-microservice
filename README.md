# TikTok FBT Integration Plan — WMS360

**Status:** Planning · **Owner:** Mohammad Rabbani · **Created:** 2026-05-12 (revised after architecture pivot) · **Target Phase 1 launch:** 2026-07-17 (9 weeks out)

> Fulfilled by TikTok (FBT) is TikTok Shop's marketplace-fulfilment programme. Merchant ships inventory to TikTok fulfilment centres → TikTok stores, picks, packs, ships orders → merchant only handles inbound, returns reconciliation, and stock replenishment. Conceptually identical to Amazon FBA.

---

## 1. Executive summary

**Decision matrix (locked in):**

| Decision | Choice | Why it matters |
|---|---|---|
| Launch region | **UK first, US next** | Single currency + single FC network for v1. Region-aware code paths from day 1 so US drop-in is config, not code. |
| Hybrid SKU support | **Yes — same SKU can be FBM + FBT simultaneously** | Inventory must split per fulfilment-mode. Drives `variation_listings` + per-warehouse-type inventory model in core WMS. |
| MVP scope | **Phase 1: read-only mirror (4w) → Phase 2: inbound wizard (3w) → Phase 3: fees & returns (3w)** | Ship value early, validate TikTok FBT API quirks (rate limits, payload shapes, sandbox gaps) before heavy investment. |
| Architecture | **Brand-new microservice `tiktok-fbt-microservice`. Existing `tiktok-microservice` is untouched.** | Some clients need both TikTok seller and TikTok FBT. Full isolation = zero regression risk on the live seller integration. Two services, two DBs, two gateway routes, two sidebar entries. |
| Credentials model | **Independent OAuth per merchant. No shared accounts table.** | New service has its own `fbt_accounts` table. Optional soft reference `linked_seller_account_id` (no FK, no cross-DB join) lets the UI display "this FBT shop is the same TikTok shop as your seller connection X". |

**End-state in one paragraph:** A brand-new `tiktok-fbt-microservice` (port 5018, DB `tiktok_fbt_microservice`, gateway route `/api/tiktok-fbt`) lives alongside the unchanged `tiktok-microservice`. Each merchant onboards a TikTok FBT shop independently of any seller connection they may already have. The new service polls FBT inventory every 15 min, pulls FBT orders every 5 min, exposes an inbound-shipment workflow, and emits RabbitMQ events under the `tiktok_fbt.*` topic family. Core-service learns one new channel slug (`tiktok_fbt`), one new warehouse type (`fbt_tiktok`), and one new fulfilment type on `OrderFulfillment` (`fbt`). FBT orders flow into the central WMS order list tagged "Fulfilled by TikTok" and **never enter any picker's queue**. The frontend gets a top-level sidebar entry "TikTok FBT" (sibling to existing "TikTok Shop"), with 6 new pages under `/clients/tiktok-fbt/*`.

---

## 2. Plan structure (read in this order)

1. **[01-architecture-and-alignment.md](./01-architecture-and-alignment.md)** — How the new microservice slots into the existing WMS architecture. Service boundaries, data flows, virtual-warehouse concept, hybrid-SKU inventory math.
2. **[02-data-model-changes.md](./02-data-model-changes.md)** — Every schema change. **Zero changes to existing tiktok-microservice DB.** New tables in the new service DB. Modest additive changes to core WMS DB.
3. **[03-api-research-checklist.md](./03-api-research-checklist.md)** — TikTok FBT endpoints to verify against current docs, sandbox onboarding plan, rate-limit budget, payload-shape unknowns to resolve before coding.
4. **[04-pages-and-workflows.md](./04-pages-and-workflows.md)** — 6 new frontend pages, screen-by-screen workflows, component reuse map.
5. **[05-phased-rollout.md](./05-phased-rollout.md)** — Week-by-week task breakdown for all 3 phases, including 1 extra week for service scaffolding.
6. **[06-deployment-and-observability.md](./06-deployment-and-observability.md)** — Feature flags, env vars, RabbitMQ topology changes, logging/metrics, rollback plan.
7. **[07-risks-and-open-questions.md](./07-risks-and-open-questions.md)** — What we don't know yet, blockers, decisions to escalate.

---

## 3. What we re-use vs what we clone vs what we build new

Because the new microservice is fully separate, we don't extend existing TikTok code in-place. We **clone tiktok-microservice as a starter template**, strip the seller-specific modules, and replace them with FBT modules. Some patterns are copy-once (acceptable); some are extracted to a shared lib (only if we end up with three+ TikTok-flavoured services).

### Re-use directly (no copy)

| Asset | Path | How we reuse it |
|---|---|---|
| RabbitMQ topic exchange `wms_multichannel` with DLX, retry queues, 24h TTL | `D:\office porjects\backend_wms_v2\core-service\rabbitmq\` | New routing keys on the existing exchange. No infra change. |
| Gateway proxy + circuit breaker + health checks | `D:\office porjects\backend_wms_v2\gateway\` | Add one line in `load-balancer.js` registering `tiktok-fbt` service. No code in gateway itself. |
| Core WMS DB + Sequelize models | `D:\office porjects\backend_wms_v2\core-service\` | New service reads `Variations`, `Catalogues`, `Channels` via the same read-only connection pattern the existing tiktok-microservice uses. |
| Frontend component library (tables, filters, BulkActionBar, StatCard, sidebar pattern) | `D:\office porjects\frontend_wms_v\src\components\` | All new FBT pages built from these. No new design system work. |
| Amazon FBA inventory snapshot pattern (multi-version, `is_latest`, velocity, days-of-cover) | `D:\office porjects\amazon-microservice\src\amazon\fba-inventory\` | Read for design inspiration. Copy structure, write fresh in new service. |

### Clone-and-adapt from existing tiktok-microservice (one-time copy at scaffolding)

These are patterns we re-implement in the new service so we don't depend on the old one being available. Cost: code duplication. Benefit: full isolation.

| Pattern | Where it lives today | What we clone |
|---|---|---|
| NestJS project shell (modules, main.ts, swagger setup) | `tiktok-microservice/src/` | Copy whole repo as starter, then prune |
| OAuth flow + state mgmt | `tiktok-microservice/src/modules/tiktok-auth/` | Copy, repoint redirect URI, rename to `fbt-auth` |
| AES-256-GCM token encryption util | `tiktok-microservice/src/common/utils/token-encryption.util.ts` | Verbatim copy. Same algorithm, same key-derivation logic. |
| HMAC request signing | `tiktok-microservice/src/common/services/tiktok-api.service.ts` (lines ~87–110) | Copy core signing function. TikTok's auth algorithm doesn't differ for FBT. |
| Token refresh cron | `tiktok-microservice/src/common/services/token-refresh-cron.service.ts` | Copy, scope to fbt_accounts |
| Webhook signature guard | `tiktok-microservice/src/common/guards/webhook-signature.guard.ts` | Verbatim copy. |
| RabbitMQ producer / consumer scaffolding | `tiktok-microservice/src/integrations/rabbitmq/` | Verbatim copy. Same exchange, same DLX. |
| Multi-tenant `client_id` filtering pattern | throughout tiktok-microservice | Same pattern in new service. |

### Build new in the new service

| Module | Phase |
|---|---|
| `fbt-accounts` (separate from seller accounts) | 1 |
| `fbt-orders` | 1 |
| `fbt-products` (FBT-eligible listings) | 1 |
| `fbt-inventory` + 15-min cron | 1 |
| `fbt-webhooks` (FBT-specific event types if TikTok publishes them) | 1 |
| `fbt-inbound` | 2 |
| `fbt-fees` (statement ingestion) | 3 |
| `fbt-returns` (sync from TikTok) | 3 |

---

## 4. New artefacts we are creating

### New microservice: `tiktok-fbt-microservice`

| Resource | Value |
|---|---|
| Repo path | `D:\office porjects\tiktok-fbt-microservice` |
| Port | **5018** (verify against latest port allocation — Amazon may use 5014) |
| Database | `tiktok_fbt_microservice` (PostgreSQL, local credentials per existing memory) |
| Gateway route | `/api/tiktok-fbt` → `http://localhost:5018/api` |
| OAuth redirect URI | `http://localhost:5018/api/fbt-auth/callback` (dev), production set per env |
| TikTok app credentials | **Open question — see [03-api-research-checklist.md](./03-api-research-checklist.md) §1** for whether to register a separate TikTok app or reuse existing seller app credentials |

### New tables in `tiktok_fbt_microservice` DB

- `fbt_accounts`
- `fbt_orders` + `fbt_order_items`
- `fbt_products` + `fbt_product_variations`
- `fbt_inventory` (snapshot pattern, multi-version)
- `fbt_inbound_shipments` + `fbt_inbound_shipment_items` (Phase 2)
- `fbt_fees` (Phase 3)
- `fbt_returns` (Phase 3)
- `fbt_activity_logs`
- `fbt_webhook_events` (idempotency log)

All DDL in [02-data-model-changes.md](./02-data-model-changes.md).

### Additive changes to core WMS DB

These are the **only** changes to existing infrastructure. All additive — no destructive operations.

- New channel row: `Channel(channelSlug='tiktok_fbt', name='TikTok FBT')`.
- `warehouses` gains `warehouse_type`, `external_marketplace`, `external_warehouse_id`, `external_account_id`, `external_region`, `external_warehouse_config` columns.
- `order_fulfillments.fulfillment_type` already STRING(20) — new value `'fbt'` added by convention (no enum migration; existing column accepts any string).
- New tables: `variation_listings`, `inbound_shipments` + items (Phase 2), `marketplace_charges` (Phase 3), `marketplace_reimbursements` (Phase 3).
- `return_requests` gains marketplace metadata columns (Phase 3).

### Frontend additions

- New top-level sidebar entry **"TikTok FBT"** (sibling to existing "TikTok Shop"), gated by feature flag `tiktok_fbt`.
- New pages directory: `src/pages/clients/tiktok-fbt/` (sibling to existing `src/pages/clients/tiktok/`).
- New service file: `src/services/TiktokFbtService.js` (sibling to existing `TiktokService.js`).
- New Zustand store directory: `src/stores/tiktok-fbt/`.

### RabbitMQ routing keys (new topic family, on existing exchange)

- `tiktok_fbt.to.wms.inventory_synced` (Phase 1)
- `tiktok_fbt.to.wms.order_created` (Phase 1)
- `tiktok_fbt.to.wms.order_status_changed` (Phase 1)
- `tiktok_fbt.to.wms.inbound_status_changed` (Phase 2)
- `tiktok_fbt.to.wms.return_received` (Phase 3)
- `tiktok_fbt.to.wms.fee_incurred` (Phase 3)
- `tiktok_fbt.to.wms.reimbursement_received` (Phase 3)
- `wms.to.tiktok_fbt.inbound_create` (Phase 2)
- `wms.to.tiktok_fbt.inbound_confirm` (Phase 2)
- `wms.to.tiktok_fbt.dispute_submit` (Phase 3)

### Cron jobs (in the new service)

- FBT inventory sync — every 15 min
- FBT orders sync — every 5 min (defence in depth alongside webhooks)
- FBT token refresh — at minute 5 and 35 of every hour (cloned from existing service)
- Inbound shipment status poll — every 30 min (Phase 2)
- Statement ingestion — daily 03:15 UTC (Phase 3)

---

## 5. Hard alignment principles (must not break)

Non-negotiable. Every PR for this initiative must respect them:

1. **Zero changes to `tiktok-microservice` repo or `tiktok_microservice` DB.** If a PR touches either, it does not belong in this initiative.
2. **No bypass of core-service.** All cross-marketplace state (orders, inventory, financials) lives in WMS DB. The new service writes its own caches; the **source of truth** for "what is fulfillable, what is owed" is core-service.
3. **Hybrid SKU is the default model.** Same SKU can be FBM (sold via seller integration) AND FBT (sold via new integration). Inventory is split through `variation_listings` + per-warehouse inventory.
4. **FBT orders skip pick/pack.** When core-service consumes a `tiktok_fbt.to.wms.order_created` message, it creates an `OrderFulfillment` row with `fulfillment_type='fbt'`. Status flows: `received → fulfilled_by_tiktok → shipped → delivered`. Pickers never see these orders.
5. **Read-only mirror first (Phase 1).** Phase 1 ships zero outbound API writes to TikTok FBT. We only read state. Validates API quirks before write-side blast radius.
6. **Region-aware from day 1.** Every code path takes `region` ∈ `{GB, US}`. UK ships first; US is a config flag, not a refactor.
7. **Feature flag per merchant.** `tiktok_fbt` feature key gates every page and every route. Same pattern as `checkPackageFeature` middleware in core-service.
8. **TikTok is the source of truth for FBT inventory.** WMS never writes FBT stock counts directly — it only mirrors what TikTok reports. The 15-min refresh lag is documented to merchants and surfaced in the UI.
9. **Reimbursement reconciliation is eventually-consistent.** Statement reports arrive on TikTok's own cadence (usually daily/weekly). Reconciliation pipeline matches charges/credits async; never block order display on financials.
10. **All TikTok API calls go through the new service's `tiktok-fbt-api.service.ts`** (the cloned-and-adapted descendant of the existing `tiktok-api.service.ts`). No direct axios anywhere else in the new repo.

---

## 6. Top 6 risks (read [07-risks-and-open-questions.md](./07-risks-and-open-questions.md) for full list)

1. **No TikTok FBT seller account yet.** Sandbox onboarding is the critical path for week 1. → Apply for TikTok FBT seller programme + dev sandbox **week 1, day 1**.
2. **TikTok FBT API surface is less documented than seller API.** → Allocate week 1 to live-testing every endpoint in sandbox before committing to schema.
3. **15-min inventory lag means oversell risk.** A unit can sell on TikTok before our sync runs. → Make UI display lag prominently; consider event-driven webhook for stock decrement if TikTok publishes one.
4. **48-hour dispatch SLA is TikTok's responsibility, but merchant gets the blame for stockouts.** → Reorder alerts in Phase 1 dashboard must be loud (red banner + email).
5. **Code drift between tiktok-microservice and tiktok-fbt-microservice.** The cloned auth, signing, and token-refresh code may diverge over time. → If a third TikTok-flavoured service ever appears (TikTok Live Shopping?), extract a shared lib then; for two services this is acceptable duplication. Document the cloned files in [07](./07-risks-and-open-questions.md) so reviewers know to keep both in sync for security fixes.
6. **TikTok app registration.** Need to confirm whether one TikTok app can serve both seller and FBT integrations, or whether we register a separate app. → Research item, week 1 day 1. See [03-api-research-checklist.md](./03-api-research-checklist.md).

---

## 7. Definition of done (Phase 1)

Phase 1 ships when **all** of the following are true:

- [ ] `tiktok-fbt-microservice` is running in production on port 5018 with its own DB.
- [ ] Gateway routes `/api/tiktok-fbt/*` correctly with circuit-breaker active.
- [ ] A merchant can connect a TikTok FBT account from `/clients/tiktok-fbt/accounts`. Independent OAuth, independent token storage.
- [ ] Existing TikTok seller integration is verified unchanged — full regression pass on seller order sync, inventory sync, product listing.
- [ ] FBT inventory is polled every 15 min and visible at `/clients/tiktok-fbt/inventory` with: fulfillable, inbound, reserved, unfulfillable, days-of-cover, last-synced timestamp.
- [ ] FBT orders flow into core-service via `tiktok_fbt.to.wms.order_created`, appear in the central orders list tagged "Fulfilled by TikTok" (via `createdVia='tiktok_fbt'`), and **do not** appear in any picker's queue.
- [ ] FBT orders show tracking number (from TikTok) without merchant action.
- [ ] Reorder alert fires when FBT days-of-cover < 14 (red), < 30 (amber), surfaces in dashboard tile.
- [ ] Feature flag `tiktok_fbt` correctly gates all UI and API access per merchant.
- [ ] Sentry/Winston logs distinguish FBT vs seller via the service name in log lines (no shared service → no risk of ambiguity).
- [ ] Smoke test: a single merchant with **both** a TikTok seller account AND a TikTok FBT account — both work, independently, no cross-contamination.

---

## 8. Out of scope (explicitly)

- Multi-warehouse FBT (TikTok routing across multiple FCs) — TikTok handles internally.
- TikTok Live Shopping fulfilment integration — separate surface.
- Cross-border FBT (UK seller selling US FBT) — Phase 3+, requires VAT + customs work.
- Bulk listing creation directly to FBT (vs reusing existing seller listings) — Phase 2 only if needed.
- Migration of historical TikTok seller orders to FBT analytics — separate ticket.
- Merging or auto-linking of merchant's seller account with their FBT account — surfaces a manual "is this the same TikTok shop?" UI link in the FBT account card; no auto-link.
- Shared lib for the cloned auth/signing code — defer until 3rd TikTok service exists.

---

## 9. Quick links

- TikTok Shop FBT seller documentation entry point: https://seller-us.tiktok.com/university/essay?knowledge_id=8644984183162670&lang=en
- Existing TikTok microservice (DO NOT MODIFY): `D:\office porjects\tiktok-microservice`
- New TikTok FBT microservice (TO BE CREATED): `D:\office porjects\tiktok-fbt-microservice`
- Core-service: `D:\office porjects\backend_wms_v2\core-service`
- Gateway: `D:\office porjects\backend_wms_v2\gateway`
- Frontend: `D:\office porjects\frontend_wms_v`
- Amazon FBA reference impl: `D:\office porjects\amazon-microservice\src\amazon\fba-inventory\`
