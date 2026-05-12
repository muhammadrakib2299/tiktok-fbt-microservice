# 09 — Folder Structure (Before / After)

> Concrete map of every file that gets added, extended, or left alone inside `D:\office porjects\tiktok-microservice`. Markers:
>
> - `[UNCHANGED]` — existing file, zero edits
> - `[+ extend]` — existing file, small additive edits only (new method, new column read)
> - `(NEW)` — does not exist today; created in Phase 1, 2, or 3

---

## A. Top level — `tiktok-microservice/`

```
tiktok-microservice/
├── src/                                       [+ extend]   (one new subtree)
├── TIKTOK_SERVICE_PLAN.md                     [UNCHANGED]
├── tsconfig.json                              [UNCHANGED]
├── tsconfig.build.json                        [UNCHANGED]
├── nest-cli.json                              [UNCHANGED]
├── .prettierrc                                [UNCHANGED]
├── .gitignore                                 [UNCHANGED]
├── .env                                       [+ extend]   (new FBT env vars appended)
├── .env.example                               [+ extend]   (new FBT env vars appended)
├── package.json                               [+ extend]   (likely no new deps; PDF generation only if Phase 2 builds labels client-side)
└── node_modules/                              [UNCHANGED]
```

---

## B. Source tree — `src/`

```
src/
├── app.controller.ts                          [UNCHANGED]
├── app.controller.spec.ts                     [UNCHANGED]
├── app.service.ts                             [UNCHANGED]
├── main.ts                                    [UNCHANGED]
├── data-source.ts                             [+ extend]   (register new fbt-* entities)
│
├── app.module.ts                              [+ extend]   (imports new FBT modules)
│
├── common/                                    [mostly unchanged — see notes]
│   ├── constants/
│   │   ├── tiktok-regions.ts                  [UNCHANGED]
│   │   └── fbt-fee-types.ts                   (NEW, Phase 3 — enum of fee categories)
│   │
│   ├── filters/
│   │   └── http-exception.filter.ts           [UNCHANGED]
│   │
│   ├── guards/
│   │   ├── auth.module.ts                     [UNCHANGED]
│   │   ├── client-auth.guard.ts               [UNCHANGED]
│   │   ├── webhook-signature.guard.ts         [UNCHANGED]   (reused for both seller and FBT webhooks)
│   │   ├── webhook-signature.guard.spec.ts    [UNCHANGED]
│   │   └── index.ts                           [UNCHANGED]
│   │
│   ├── interceptors/
│   │   └── response.interceptor.ts            [UNCHANGED]
│   │
│   ├── interfaces/
│   │   └── response.interface.ts              [UNCHANGED]
│   │
│   ├── services/
│   │   ├── tiktok-api.module.ts               [UNCHANGED]
│   │   ├── tiktok-api.service.ts              [UNCHANGED]   (HMAC signing + token mgmt reused by FBT)
│   │   ├── tiktok-api.service.spec.ts         [UNCHANGED]
│   │   ├── token-refresh-cron.service.ts      [UNCHANGED]   (already refreshes all account rows; FBT-enabled accounts inherit)
│   │   └── index.ts                           [UNCHANGED]
│   │
│   ├── utils/
│   │   ├── token-encryption.util.ts           [UNCHANGED]
│   │   └── token-encryption.util.spec.ts      [UNCHANGED]
│   │
│   └── index.ts                               [UNCHANGED]
│
├── config/
│   ├── database.config.ts                     [UNCHANGED]
│   ├── wms-database.config.ts                 [UNCHANGED]
│   └── tiktok.config.ts                       [+ extend]   (optional FBT scope list, FBT cron tunables)
│
├── integrations/
│   └── rabbitmq/
│       ├── rabbitmq.module.ts                 [+ extend]   (register new fbt routing keys)
│       └── dlq-consumer.service.ts            [UNCHANGED]
│
├── migrations/
│   ├── 1742900000000-InitialSchema.ts                          [UNCHANGED]
│   ├── 1745160000000-CascadeVariationFk.ts                     [UNCHANGED]
│   ├── 1745170000000-DedupeAndUniqueVariationSku.ts            [UNCHANGED]
│   ├── <ts>-AddFulfilmentFlagsToAccounts.ts                    (NEW, Phase 1 — adds fbm_enabled + fbt_enabled + chk_accounts_any_mode)
│   ├── <ts>-AddFulfilmentModeToTiktokOrders.ts                 (NEW, Phase 1)
│   ├── <ts>-CreateFbtInventory.ts                              (NEW, Phase 1)
│   ├── <ts>-CreateFbtWebhookEvents.ts                          (NEW, Phase 1 — idempotency log)
│   ├── <ts>-CreateFbtInboundShipments.ts                       (NEW, Phase 2)
│   ├── <ts>-CreateFbtInboundShipmentItems.ts                   (NEW, Phase 2)
│   ├── <ts>-CreateFbtFees.ts                                   (NEW, Phase 3)
│   └── <ts>-CreateFbtReturns.ts                                (NEW, Phase 3)
│
├── models/
│   ├── account.entity.ts                      [+ extend]   (fbm_enabled, fbt_enabled, publish_fbt_to_wms, fbt_capabilities, fbt_enabled_at, fbt_last_verified_at; chk_accounts_any_mode constraint)
│   ├── activity-log.entity.ts                 [UNCHANGED]   (logs both seller and FBT events)
│   ├── tiktok-order.entity.ts                 [+ extend]   (add fulfilment_mode column)
│   ├── tiktok-order-item.entity.ts            [UNCHANGED]
│   ├── tiktok-product.entity.ts               [UNCHANGED]   (catalog management is seller-only; FBT reuses listings)
│   ├── tiktok-product-variation.entity.ts     [UNCHANGED]
│   ├── wms-catalogue.entity.ts                [UNCHANGED]
│   ├── wms-variation.entity.ts                [UNCHANGED]
│   ├── fbt-inventory.entity.ts                (NEW, Phase 1)
│   ├── fbt-webhook-event.entity.ts            (NEW, Phase 1)
│   ├── fbt-inbound-shipment.entity.ts         (NEW, Phase 2)
│   ├── fbt-inbound-shipment-item.entity.ts    (NEW, Phase 2)
│   ├── fbt-fee.entity.ts                      (NEW, Phase 3)
│   ├── fbt-return.entity.ts                   (NEW, Phase 3)
│   └── index.ts                               [+ extend]   (export new entities)
│
└── modules/
    │
    ├── accounts/                              [+ extend — gains FBT toggle endpoints]
    │   ├── accounts.controller.ts             [+ extend]   (+ POST :id/fbt/enable, :id/fbt/disable)
    │   ├── accounts.module.ts                 [+ extend]   (imports FbtAccountVerifier service)
    │   ├── accounts.service.ts                [+ extend]   (+ enableFbt(), disableFbt() with TikTok verification)
    │   ├── accounts.service.spec.ts           [+ extend]   (cover FBT toggle paths)
    │   └── dto/
    │       ├── create-account.dto.ts          [UNCHANGED]
    │       ├── update-account.dto.ts          [UNCHANGED]
    │       ├── enable-fbt.dto.ts              (NEW, Phase 1)
    │       └── index.ts                       [+ extend]
    │
    ├── activity-logs/                         [UNCHANGED]   (works for both)
    │   ├── activity-logs.controller.ts        [UNCHANGED]
    │   ├── activity-logs.module.ts            [UNCHANGED]
    │   └── activity-logs.service.ts           [UNCHANGED]
    │
    ├── inventory/                             [UNCHANGED]   (seller-side stock push to TikTok)
    │   ├── inventory.controller.ts            [UNCHANGED]
    │   ├── inventory.module.ts                [UNCHANGED]
    │   ├── inventory.service.ts               [UNCHANGED]
    │   ├── inventory.service.spec.ts          [UNCHANGED]
    │   ├── inventory-consumer.service.ts      [UNCHANGED]
    │   └── quantity-sync.service.ts           [UNCHANGED]
    │
    ├── migration/                             [UNCHANGED]
    │   ├── migration.controller.ts            [UNCHANGED]
    │   ├── migration.module.ts                [UNCHANGED]
    │   ├── migration.service.ts               [UNCHANGED]
    │   └── wms-catalogue-publisher.service.ts [UNCHANGED]
    │
    ├── orders/                                [+ extend — routes by fulfilment_mode]
    │   ├── orders.controller.ts               [+ extend]   (filter chip support: ?fulfilment_mode=fbt)
    │   ├── orders.module.ts                   [+ extend]
    │   ├── orders.service.ts                  [+ extend]   (read fulfilment_type from TikTok payload, set tiktok_orders.fulfilment_mode)
    │   ├── orders.service.spec.ts             [+ extend]
    │   ├── orders-consumer.service.ts         [+ extend]
    │   ├── orders-cron.service.ts             [+ extend]   (poll handles both modes; FBT uses separate cron in fbt-orders/ if rates differ)
    │   ├── orders-publisher.service.ts        [UNCHANGED]
    │   ├── wms-actions-consumer.service.ts    [UNCHANGED]   (FBT actions go to fbt-orders consumer instead)
    │   ├── wms-order-publisher.service.ts     [+ extend]   (route to tiktok_fbt.to.wms.* if fulfilment_mode=fbt)
    │   └── dto/                               [UNCHANGED]
    │
    ├── products/                              [UNCHANGED]   (catalog mgmt = seller domain only)
    │   ├── products.controller.ts             [UNCHANGED]
    │   ├── products.module.ts                 [UNCHANGED]
    │   ├── products.service.ts                [UNCHANGED]
    │   ├── products.service.spec.ts           [UNCHANGED]
    │   ├── products-cron.service.ts           [UNCHANGED]
    │   └── dto/                               [UNCHANGED]
    │
    ├── tiktok-auth/                           [+ extend — only if FBT scopes need an extra OAuth round]
    │   ├── tiktok-auth.controller.ts          [+ extend]   (optional: callback may set fbt_capabilities if scope grant detected)
    │   ├── tiktok-auth.module.ts              [UNCHANGED]
    │   └── tiktok-auth.service.ts             [+ extend]   (request FBT scopes alongside existing seller scopes)
    │
    ├── webhooks/                              [+ extend — routes events to FBT handlers]
    │   ├── webhooks.controller.ts             [UNCHANGED]   (HMAC guard still does verification; just one endpoint)
    │   ├── webhooks.module.ts                 [+ extend]   (imports fbt-webhooks)
    │   ├── webhooks.service.ts                [+ extend]   (dispatcher: routes to seller or FBT handler based on payload + account.fbt_enabled)
    │   └── webhooks.service.spec.ts           [+ extend]
    │
    └── fbt/                                   (NEW PARENT DIRECTORY — Phase 1)
        │
        ├── fbt.module.ts                                                   (NEW)  parent NestJS module that imports all FBT submodules
        │
        ├── common/                                                         (NEW, Phase 1 — shared FBT primitives)
        │   ├── guards/
        │   │   ├── can-publish-fbt.ts                                       fail-closed guard: checks fbt_enabled === true && publish_fbt_to_wms === true
        │   │   └── can-publish-fbt.spec.ts                                  required test table (see 06-deployment-and-observability.md §12) — written Tue AM, before any publisher
        │   └── metrics/
        │       └── fbt-publish-skipped.counter.ts                           wraps the `fbt_publish_skipped_total` counter with typed labels
        │
        ├── fbt-accounts/                                                   (NEW, Phase 1)
        │   ├── fbt-accounts.controller.ts                                  GET /fbt/accounts (lens-only — lists rows where fbt_enabled=true)
        │   ├── fbt-accounts.module.ts
        │   ├── fbt-accounts.service.ts                                      includes verifyShopEligibility() that hits TikTok before flipping fbt_enabled
        │   ├── fbt-accounts.README.md                                       documents the verifyShopEligibility() contract — populated Mon AM after open question C1 is resolved
        │   └── dto/
        │       ├── verify-fbt-eligibility.dto.ts
        │       └── index.ts
        │
        ├── fbt-inventory/                                                  (NEW, Phase 1)
        │   ├── fbt-inventory.controller.ts                                  GET /fbt/inventory · GET /fbt/inventory/:sku
        │   ├── fbt-inventory.module.ts
        │   ├── fbt-inventory.service.ts                                     syncForAccount(), getLatest()
        │   ├── fbt-inventory-cron.service.ts                                @Cron('0 */15 * * * *')
        │   ├── fbt-velocity-calculator.service.ts                           7d/30d sales → days_of_cover, health colour
        │   ├── fbt-inventory.service.spec.ts
        │   └── dto/
        │       ├── list-inventory.dto.ts
        │       ├── sync-inventory.dto.ts
        │       └── index.ts
        │
        ├── fbt-orders/                                                     (NEW, Phase 1 — thin lens over tiktok_orders)
        │   ├── fbt-orders.controller.ts                                     GET /fbt/orders (filters tiktok_orders WHERE fulfilment_mode='fbt')
        │   ├── fbt-orders.module.ts
        │   ├── fbt-orders.service.ts                                        listFbtOrders(), getFbtOrder()
        │   └── dto/
        │       ├── list-fbt-orders.dto.ts
        │       └── index.ts
        │
        ├── fbt-dashboard/                                                  (NEW, Phase 1)
        │   ├── fbt-dashboard.controller.ts                                  GET /fbt/dashboard
        │   ├── fbt-dashboard.module.ts
        │   └── fbt-dashboard.service.ts                                     aggregates: counts, health summary, sales trend
        │
        ├── fbt-webhooks/                                                   (NEW, Phase 1)
        │   ├── fbt-webhooks.service.ts                                      processFbtEvent() — called by webhooks/ dispatcher
        │   └── fbt-webhooks.module.ts
        │
        ├── fbt-inbound/                                                    (NEW, Phase 2)
        │   ├── fbt-inbound.controller.ts                                    POST /fbt/inbound, GET /fbt/inbound, GET /fbt/inbound/:id
        │   ├── fbt-inbound.module.ts
        │   ├── fbt-inbound.service.ts                                       createPlan(), submitPlan(), confirmPlan(), cancel(), getLabel()
        │   ├── fbt-inbound-cron.service.ts                                  @Cron('0 */30 * * * *') — status poll for non-terminal shipments
        │   └── dto/
        │       ├── create-inbound-plan.dto.ts
        │       ├── confirm-inbound.dto.ts
        │       └── index.ts
        │
        ├── fbt-fees/                                                       (NEW, Phase 3)
        │   ├── fbt-fees.controller.ts                                       GET /fbt/fees, GET /fbt/reimbursements
        │   ├── fbt-fees.module.ts
        │   ├── fbt-fees.service.ts                                          ingestStatement(), getFees(), discrepancyCheck()
        │   ├── fbt-statement-cron.service.ts                                @Cron('15 3 * * *') — daily statement pull
        │   └── dto/
        │       ├── list-fees.dto.ts
        │       └── index.ts
        │
        └── fbt-returns/                                                    (NEW, Phase 3)
            ├── fbt-returns.controller.ts                                    GET /fbt/returns
            ├── fbt-returns.module.ts
            └── fbt-returns.service.ts                                       processReturnWebhook()
```

---

## C. Summary: files that change vs files that are new

### Existing files modified (8 files)

1. `src/app.module.ts` — register FBT modules
2. `src/data-source.ts` — register FBT entities
3. `src/config/tiktok.config.ts` — FBT-specific env knobs
4. `src/models/index.ts` — export FBT entities
5. `src/models/account.entity.ts` — 4 new columns
6. `src/models/tiktok-order.entity.ts` — 1 new column
7. `src/modules/accounts/accounts.{controller,service,module}.ts` + DTOs — 2 new endpoints, FBT verifier
8. `src/modules/orders/orders.service.ts` + `wms-order-publisher.service.ts` — fulfilment_mode routing
9. `src/modules/webhooks/webhooks.{service,module}.ts` — dispatch FBT events
10. `src/integrations/rabbitmq/rabbitmq.module.ts` — new routing keys
11. `.env`, `.env.example` — FBT env vars

### New files (8 entities + 1 fbt.module + ~30 service/controller/dto files)

- **Migrations:** 8 new (4 in Phase 1, 2 in Phase 2, 2 in Phase 3)
- **Entities:** 6 new (2 in Phase 1, 2 in Phase 2, 2 in Phase 3)
- **Module trees under `fbt/`:** 8 modules total (5 in Phase 1, 1 in Phase 2, 2 in Phase 3)

Approximate count of new files in the existing service: **~50 files**, all under `src/migrations/` (8) or `src/models/` (6) or `src/modules/fbt/` (rest).

### Files explicitly untouched (the existing seller flow)

The entire `src/modules/inventory/`, `src/modules/products/`, `src/modules/migration/`, `src/modules/activity-logs/` directories are zero-touch. The shared `src/common/` directory is zero-touch. This is the safety guarantee: the seller integration is unaffected.

---

## D. What about the planning workspace folder?

The folder `D:\office porjects\tiktok-fbt-microservice` continues to hold **planning artefacts only** — the markdown files in this directory plus `mockup.html`. **No code lives there.** Code goes into the existing `D:\office porjects\tiktok-microservice` repo.

You can think of `tiktok-fbt-microservice/` as the "FBT initiative workspace" — design docs, decisions, mockups — and `tiktok-microservice/` as the production code that implements those decisions.

If you'd rather move these planning files into the code repo (e.g. `tiktok-microservice/docs/fbt/`), that's a clean rename. Either works.
