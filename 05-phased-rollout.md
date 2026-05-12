# 05 — Phased Rollout

> ~10 weeks of work, 3 phases. No Phase 0 scaffolding needed — we're modifying an existing service, not standing up a new one.

---

## Calendar at a glance (anchored to today, 2026-05-12)

| Phase | Window | Deliverable |
|---|---|---|
| **Pre-week** | 2026-05-12 → 05-16 | TikTok FBT enrollment application + API research + sandbox access |
| **Phase 1 — Read-only mirror** | 2026-05-19 → 06-13 (4w) | Enable-FBT toggle, FBT inventory mirror, FBT orders flow, dashboard |
| **Phase 2 — Inbound shipments** | 2026-06-16 → 07-04 (3w) | Wizard, labels, tracking |
| **Phase 3 — Fees & reimbursements** | 2026-07-07 → 07-25 (3w) | Statement ingestion, charges, reimbursements |
| **Hardening + Public launch** | 2026-07-28 → 08-08 (2w) | Docs, marketing, first paying merchants |

Total: ~12 weeks calendar including pre-week and launch polish.

---

## Pre-week (5 days)

| Day | Owner | Task |
|---|---|---|
| Mon 05-12 | tech lead | Submit TikTok FBT enrollment application via Seller Center UK |
| Mon 05-12 | tech lead | Verify the existing TikTok app's scope list — does it cover FBT endpoints, or do we need to extend scopes / register a fresh app? |
| Tue 05-13 | backend | API research: hit any sandbox endpoints we can; build a Postman collection |
| Wed 05-14 | tech lead | Reconcile port discrepancy (.env.example says 5010, gateway routes to 5011) — gateway is canonical |
| Thu 05-15 | full team | Plan review meeting. Walk this folder. Confirm phasing |
| Fri 05-16 | backend | Create feature branch `feat/tiktok-fbt-phase-1` on tiktok-microservice, core-service, frontend |

**Exit criterion:** TikTok FBT sandbox accessible OR partner-manager escalation path identified.

---

## Phase 1 — Read-only mirror (4 weeks)

### Week 1 — Schema + enable-FBT toggle

> **Day-1 blocker first.** Before any controller or frontend work, Mon AM is dedicated to answering open questions C1–C3 (eligibility endpoint, app scope, sandbox availability — see `07-risks-and-open-questions.md`). The toggle UX downstream of these is speculative until resolved.

| Day | Workstream | Task |
|---|---|---|
| **Mon AM** | **backend** | **🚧 BLOCKER — Verify eligibility endpoint.** Call `GET /authorization/202309/shops` against sandbox or real FBT shop. Capture full JSON in `endpoint-payloads.md`. Confirm presence of FBT enrollment field. If absent, decide fallback path (trust-the-merchant + post-hoc validation, or admin-gated toggle) **before any UX or controller code is written.** Update risk A0 to closed. |
| **Mon AM** | **backend** | **🚧 BLOCKER — Check app scopes** in TikTok Partner Center. Confirm FBT scopes are available on existing app, or decide on second-app registration. Update risk A3. |
| Mon PM | backend | Migrations A1, A2, A3, A4 (tiktok-microservice) + B1, B2, B3, B4 (core-service). Apply to staging. **A1 now includes `publish_fbt_to_wms` operational kill-switch (admin-only, default TRUE).** |
| Mon PM | backend | Continue API research for remaining payload shapes |
| Tue AM | backend | **Write `canPublishFbt()` guard + spec FIRST** (test table from `06-deployment-and-observability.md` §12). Commit on green. Every later publisher imports this guard rather than re-deriving the check. |
| Tue | backend | Extend `accounts.service.ts` with `enableFbt()`, `disableFbt()`, `disableFbm()`, `enableFbm()`. Add controller endpoints. Create `fbt-accounts/` module with `verifyShopEligibility()` (shape determined by Mon AM finding) |
| Tue | backend | Skeleton `fbt/` module tree — empty modules wired into app.module.ts |
| Wed | frontend | Extend Accounts page with `<FulfilmentModesPanel>` component containing both `<FbmAccountToggle>` and `<FbtAccountToggle>`. Hit the four enable/disable endpoints. Refresh sidebar on either toggle change |
| Wed | frontend | Three-view conditional sidebar logic — General sub-items gated on aggregated `fbm_enabled` (+ `clientHasHistoricalFbmOrders`); FBT sub-items gated on aggregated `fbt_enabled`; Accounts entry always rendered |
| Thu | backend (core) | New RabbitMQ consumer for `tiktok_fbt.to.wms.account_fbt_enabled` — inserts virtual warehouse row, channel row. **Consumer also calls `canPublishFbt()` on receipt** (defence in depth — re-check flags before mutating state). |
| Thu | backend | Webhook dispatcher — extend webhooks.service to route by payload to seller or FBT handler |
| Fri | full team | EoW1 review. End-to-end test matrix: (a) FBM-only client sees only General submodules, (b) connect a sandbox FBT-enrolled shop and flip FBT on → hybrid view appears, (c) toggle FBM off on the same shop → FBT-only view, (d) toggle FBM back on → hybrid restored. Verify sidebar reshapes in one render tick each time. **(e) Fail-closed regression: flip `publish_fbt_to_wms=false` via SQL, place an FBT order in sandbox, verify zero RabbitMQ messages emitted and `fbt_publish_skipped_total{reason='publish_gate_off'}` increments.** |

### Week 2 — Inventory pipeline

| Day | Workstream | Task |
|---|---|---|
| Mon-Tue | backend | `fbt-inventory.service.syncForAccount()`: paginate TikTok FBT inventory endpoint, snapshot with `is_latest` flip |
| Mon-Tue | backend | `fbt-velocity-calculator.service`: 7d/30d sales from `tiktok_orders` (mode='fbt'), health colour assignment |
| Wed | backend | `fbt-inventory-cron.service` @ `0 */15 * * * *`. Stagger 30s per account. Manual trigger endpoint (rate-limited) |
| Wed | backend | RabbitMQ publisher: `tiktok_fbt.to.wms.inventory_synced`. Core-service stub consumer (log+ack only Phase 1) |
| Thu-Fri | frontend | FBT Inventory page — table, filters, search, drawer with breakdown, CSV export |

**Exit criterion EoW2:** sandbox account's FBT inventory visible in UI, refreshes every 15 min.

### Week 3 — Order pipeline

| Day | Workstream | Task |
|---|---|---|
| Mon-Tue | backend | Extend `orders.service.upsertOrder()`: detect `fulfilment_type` from payload, set `tiktok_orders.fulfilment_mode` |
| Mon-Tue | backend | Extend `wms-order-publisher`: route to `tiktok_fbt.to.wms.order_created` when `fulfilment_mode='fbt'` |
| Wed | backend (core) | New consumer `FbtOrderConsumer`: resolve `variation_listings`, create Order with `createdVia='tiktok_fbt'`, OrderFulfillment(`type='fbt'`), deduct virtual warehouse inventory |
| Wed | backend (core) | **Critical regression test**: existing pick-queue queries (`/clients/order/awaiting-pick`) must exclude `fulfillment_type='fbt'`. Add automated integration test |
| Thu | backend | `fbt-orders-cron` @ 5min — defence in depth: poll for any orders the webhook missed |
| Thu-Fri | frontend | Build dedicated FBT Orders page at `/clients/tiktok/fbt/orders` (between Inventory and Inbound in sidebar). Modify central `/clients/order` page query to add `WHERE created_via != 'tiktok_fbt'` — visual changes none, FBT just doesn't appear there. |

**Exit criterion EoW3:** sandbox FBT order flows TikTok → tiktok-microservice → core WMS → order list. Never appears in picker queue.

### Week 4 — Dashboard + polish + ship

| Day | Workstream | Task |
|---|---|---|
| Mon | frontend | FBT Dashboard — stat cards, donut, reorder alerts table |
| Mon | backend | `fbt-dashboard.service` — aggregates from latest `fbt_inventory` + 30-day `tiktok_orders` |
| Tue | full team | **Critical regression**: verify existing TikTok seller integration unaffected. Full regression pass on seller order sync, inventory push, product status |
| Tue | full team | Smoke test against sandbox: 1 FBT-enabled account + same merchant has the underlying seller flow. Both work, no cross-contamination |
| Wed | full team | Fix smoke-test bugs |
| Thu | dev + QA | Final pass: error states, account disconnect mid-sync, expired token, malformed webhook. PII redaction verified |
| Fri | full team | Phase 1 ship: deploy to staging, run 24h, then prod. Feature flag enabled for 1 internal pilot |

---

## Phase 2 — Inbound shipments (3 weeks)

### Week 5 — Inbound backend

| Day | Task |
|---|---|
| Mon | Migrations A5, A6 (tiktok-microservice) + B5 (core-service) |
| Mon-Tue | `fbt-inbound.service.createPlan()`: validates SKU stock, calls TikTok plan API, persists shipment row |
| Wed | `confirmPlan`, `cancelPlan`, `getLabel` endpoints. Each one TikTok API call + state update |
| Thu | `fbt-inbound-cron.service` @ 30min — poll non-terminal shipments for status updates |
| Thu | Webhook handler for `FBT_INBOUND_RECEIVED` if TikTok publishes; otherwise rely on cron |
| Fri | core-service side: on receipt webhook, decrement source warehouse, trust next FBT inventory poll for fulfillable count |

### Week 6 — Inbound wizard frontend

| Day | Task |
|---|---|
| Mon | `<InboundShipmentsListPage>` — table, status filters |
| Tue-Thu | 4-step wizard: products → FC → review → labels & ship. Shared form state via `useTiktokFbtInboundStore` |
| Fri | Detail page (`/clients/tiktok/fbt/inbound/[id]`) with timeline, line items, dispute stub |

### Week 7 — Hardening + ship

| Day | Task |
|---|---|
| Mon-Tue | Test partial receipt, damaged units, short receipts, cancel-before-ship, oversell at plan creation |
| Wed | Reimbursement stub: on partial/damaged receipt, auto-create placeholder `marketplace_reimbursements` row |
| Thu | In-app + email notifications: shipment arrives, discrepancy detected |
| Fri | Phase 2 ship to staging then prod (flag-gated) |

---

## Phase 3 — Fees, reimbursements, returns (3 weeks)

### Week 8 — Fee ingestion

| Day | Task |
|---|---|
| Mon | Migrations A7, A8 (tiktok-microservice) + B6, B7 (core-service) |
| Mon | `fbt-statement-cron.service` @ 03:15 daily — fetch statement, parse line items, insert `fbt_fees` rows |
| Tue-Wed | Publish per-line: `tiktok_fbt.to.wms.fee_incurred`, core-service consumer → `marketplace_charges` |
| Thu | Reimbursement ingestion: same flow → `marketplace_reimbursements` |
| Fri | Discrepancy detector cron in core-service: flags suspicious lines for review |

### Week 9 — Fees UI + returns UI

| Day | Task |
|---|---|
| Mon-Wed | FBT Fees page — stat cards, bar chart, fee table, reimbursement table, discrepancy alerts |
| Thu | Returns: extend existing core WMS returns UI with FBT marketplace metadata. New filter "Initiated by marketplace" |
| Fri | End-to-end: simulate FBT order → ship → return → reimbursement appears in fees page |

### Week 10 — Dispute, hardening, ship

| Day | Task |
|---|---|
| Mon-Tue | Dispute submission flow: UI → `wms.to.tiktok_fbt.dispute_submit` → service calls TikTok dispute API (or surfaces external link) |
| Wed | Currency display: USD-vs-GBP separation. No silent conversion |
| Thu | QA. Edge cases. Performance check (statement payloads can be large) |
| Fri | Phase 3 ship |

---

## Hardening + public launch (2 weeks)

### Week 11
- Public docs, sales enablement, load test (100 accounts × 1000 SKUs × 5000 orders/day), observability dashboards.

### Week 12
- 2nd pilot merchant (real volume). Open to all UK merchants on appropriate subscription tier.

### Post-launch: US region (~2 weeks)
- Add `region='US'` paths. USD currency. US addressing. US FCs in dropdown. Regression-test UK.

---

## Critical path

If any of these slip, the whole plan slips:

1. **Pre-week TikTok FBT enrollment + sandbox access** (5 days)
2. **Phase 1 week 1 API research + enable-FBT verification call** — schemas in [02](./02-data-model-changes.md) depend on real payload shapes
3. **Phase 1 week 3 order routing** — FBT orders must reliably skip pick/pack
4. **Phase 2 week 5 inbound API surface** — if it differs materially from assumptions, plan slips
5. **Phase 3 week 8 statement format** — most data-format-sensitive work

Each item has 1-2 days of buffer within its week.

---

## Team ownership matrix (suggested)

| Role | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|
| Tech lead | API research, schema review, regression risk owner | Workflow design | Financial reconciliation review |
| Backend eng A | tiktok-microservice fbt module + accounts toggle | Inbound API | Statement ingestion + dispute |
| Backend eng B | core-service consumer + order routing + regression test | Inbound consumer + inventory side-effects | Charge/reimbursement consumers + discrepancy detector |
| Frontend eng | Toggle UI + Dashboard + Inventory + central orders chip | Inbound list + wizard + detail | Fees page + returns extension |
| DevOps | New env vars + migration deployment | Same | Same |
| QA | EoW2, EoW3, EoW4 regression | EoW7 | EoW10 |
