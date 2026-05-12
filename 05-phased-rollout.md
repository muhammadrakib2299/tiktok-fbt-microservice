# 05 — Phased Rollout

> ~11-12 weeks of work, split into 3 ship-able phases plus 1 week of scaffolding to set up the new microservice.
>
> Capacity assumption: **2 backend engineers + 1 frontend engineer + 0.5 QA**, with you (Mohammad) doing tech lead + integration testing. Adjust dates if the team is smaller.

---

## Calendar at a glance (anchored to today, 2026-05-12)

| Phase | Window | Deliverable |
|---|---|---|
| **Pre-week** | 2026-05-12 → 05-16 | TikTok FBT programme application; sandbox access; API research |
| **Phase 0 — Scaffolding** | 2026-05-19 → 05-23 | New repo, DB, gateway entry, CI/CD, deploy pipeline. **No business logic.** |
| **Phase 1 — Read-only mirror** | 2026-05-26 → 06-20 (4w) | Connect account, mirror inventory, pass-through orders, dashboard |
| **Phase 2 — Inbound shipments** | 2026-06-23 → 07-11 (3w) | Create plan, generate labels, track shipments |
| **Phase 3 — Fees & reimbursements** | 2026-07-14 → 08-01 (3w) | Statement ingestion, charges/reimbursements display, basic dispute flow |
| **Hardening + Public launch** | 2026-08-04 → 08-15 (2w) | Public docs, marketing enablement, first paying merchants |

Total: ~13 weeks calendar. The architectural change (full microservice separation) added ~1 week vs the embedded-module plan.

US-region expansion (after UK) is a separate ~2-week effort post-Phase 3.

---

## Pre-week (5 days)

**Goal:** unblock everything before code starts.

| Day | Owner | Task |
|---|---|---|
| Mon 05-12 | tech lead | Apply for TikTok FBT seller programme; request expanded API scopes. |
| Mon 05-12 | tech lead | Determine whether existing TikTok app credentials cover FBT or a new app registration is needed. |
| Tue 05-13 | backend | Begin payload research against any docs that exist publicly. Set up Postman collection. |
| Wed 05-14 | tech lead | Update `reference_port_allocation.md` memory file — add planned port 5018 for tiktok-fbt-microservice. **Verify 5014 isn't claimed by amazon-microservice** (memory is 35 days stale; check production manifests). |
| Thu 05-15 | full team | Plan-review meeting. Walk through this folder. Lock or adjust phasing. |
| Fri 05-16 | tech lead | Provision: new git repo `tiktok-fbt-microservice`, new Postgres DB `tiktok_fbt_microservice`, environment variables documented. |

**Exit criterion:** TikTok sandbox FBT shop accessible OR escalation path to TikTok partner manager identified.

---

## Phase 0 — Scaffolding (1 week)

**Goal:** The new microservice exists, runs, is reachable through the gateway, and has CI/CD. Zero FBT functionality yet.

| Day | Workstream | Task |
|---|---|---|
| Mon | backend | Clone `tiktok-microservice` repo as starter into `tiktok-fbt-microservice`. Rename package, project, swagger title. Strip seller modules (`products`, `orders`, `inventory`, `migration`). Keep: `common/`, `config/`, `integrations/rabbitmq/`, auth scaffolding. |
| Mon | backend | Rename DB connection target to `tiktok_fbt_microservice`. Update env defaults. |
| Tue | backend | Verify cloned files work: NestJS boots, `/api/docs` (Swagger) loads, `/health` returns 200, DB migrations table created. |
| Tue | DevOps | Add `tiktok-fbt` service registration in gateway `load-balancer.js`. Health check every 15s. Circuit breaker config matches other services. |
| Wed | backend | RabbitMQ wiring: new service connects to existing `wms_multichannel` exchange. Set up new routing-key publisher/consumer scaffolding. |
| Wed | backend | First migration in new repo: `<ts>-CreateFbtAccounts.ts`. Run migration in staging. Verify `\d fbt_accounts`. |
| Thu | backend | Clone-and-adapt the auth module (`tiktok-auth` → `fbt-auth`). Repoint redirect URI. **Do NOT yet validate against TikTok — just compile and run.** |
| Thu | backend | Clone token encryption util, signing service, webhook signature guard. Verify with existing test vectors. |
| Fri | DevOps | CI/CD pipeline: lint, type-check, build, deploy to staging. Match existing tiktok-microservice pipeline. |
| Fri | full team | Phase 0 demo: hit `https://staging.gateway/api/tiktok-fbt/health` from outside — get a 200 from the new service. |

**Phase 0 Definition of Done:**
- [ ] `tiktok-fbt-microservice` runs locally and in staging.
- [ ] Gateway routes `/api/tiktok-fbt/*` correctly.
- [ ] DB `tiktok_fbt_microservice` exists with TypeORM migrations table.
- [ ] CI/CD green.
- [ ] Existing `tiktok-microservice` is verified untouched.
- [ ] Port 5018 confirmed (or alternate allocated and documented).

---

## Phase 1 — Read-only mirror (4 weeks)

### Week 1 — Foundation: schema + auth

| Day | Workstream | Task |
|---|---|---|
| Mon | backend | All Phase-1 migrations (A1–A6, A11 in new repo + B1–B4 in core-service) applied to staging. |
| Mon | backend | Finish API research: payload shapes captured in `endpoint-payloads.md`. Confirm any schema deltas. |
| Tue | backend | Implement `FbtAccountsService`: connect (OAuth → token exchange → fbt_accounts insert), list, disconnect, sync-now. |
| Tue | backend | Webhook controller with FBT-specific event handlers (stubbed; logic in week 2). |
| Wed | frontend | Scaffold `src/pages/clients/tiktok-fbt/` directory. Empty placeholder pages with `<PageHeadingWithBreadcrumb>`. Sidebar entry added behind feature flag. |
| Wed | frontend | Build `TiktokFbtService.js`. Build first Zustand store `useTiktokFbtAccountsStore`. |
| Thu | frontend | Build `<TiktokFbtAccountsPage>` — list, connect modal, disconnect confirmation, link-seller-account UI. |
| Thu | backend (core) | Insert "TikTok FBT" channel row. Add new RabbitMQ consumer stub in core-service for `tiktok_fbt.to.wms.account_connected`. When triggered, inserts the virtual warehouse row. |
| Fri | full team | EoW1 review. End-to-end test: connect a sandbox FBT account, see card appear, virtual warehouse exists in core WMS. |

### Week 2 — Inventory pipeline

| Day | Workstream | Task |
|---|---|---|
| Mon-Tue | backend | Implement `FbtInventoryService.syncForAccount(accountId)`: pagination, retry, error handling, snapshot insert with `is_latest` flip. |
| Mon-Tue | backend | Implement `FbtVelocityCalculator` (7d/30d sales from `fbt_orders`; placeholder until orders pipeline in week 3). Health scoring (green/amber/red). |
| Wed | backend | Wire `FbtInventoryCronService` to `@Cron('0 */15 * * * *')`. Stagger 30s per account. Manual trigger endpoint (rate-limited). |
| Wed | backend | RabbitMQ publisher: `tiktok_fbt.to.wms.inventory_synced`. Core-service stub consumer (log + ack only — no business logic Phase 1). |
| Thu-Fri | frontend | Build `<FbtInventoryPage>` — table, filters, search, CSV export. Wire to `useTiktokFbtInventoryStore`. |

**Exit criterion EoW2:** sandbox account's inventory visible in UI, refreshes every 15 min.

### Week 3 — Order pipeline

| Day | Workstream | Task |
|---|---|---|
| Mon-Tue | backend | Implement `FbtOrdersService.syncForAccount(accountId)`: fetch order list, fetch each order detail, upsert `fbt_orders` + `fbt_order_items`. |
| Mon-Tue | backend | Webhook handler for `ORDER_STATUS_CHANGE`: idempotency check via `fbt_webhook_events`, then fetch + upsert. |
| Wed | backend | RabbitMQ publisher: `tiktok_fbt.to.wms.order_created` with full payload schema documented in `01-architecture-and-alignment.md §6`. |
| Wed | backend (core) | Implement `FbtOrderConsumer` in core-service: resolve `variation_listings` (where `channel_id=tiktok_fbt`, `account_id`, `external_sku_id`), create Order with `createdVia='tiktok_fbt'`, OrderProduct, OrderFulfillment(type='fbt'). Idempotency via `wms_message_id`. |
| Wed | backend (core) | **Regression-test pick/pack queries**: all `getOrdersAwaitingPick()` etc. must filter `WHERE order_fulfillments.fulfillment_type != 'fbt'`. Add an integration test that creates one FBT and one FBM order and asserts only FBM appears in picker queue. |
| Thu | backend | `FbtOrdersCronService` — every 5 minutes, poll for orders not yet seen via webhook (defence in depth). Stagger per account. |
| Thu-Fri | frontend | Build `<FbtOrdersPage>` (list + detail drawer). Add `<FulfilledByTikTokBadge>` to central order list rows. |

**Exit criterion EoW3:** sandbox FBT order flows TikTok → new service → core WMS → order list. No picker sees it.

### Week 4 — Dashboard, polish, smoke test

| Day | Workstream | Task |
|---|---|---|
| Mon | frontend | Build `<FbtDashboardPage>` — stat cards, health chart, reorder alert table. |
| Mon | backend | `FbtDashboardService` — aggregates from `fbt_inventory` + `fbt_orders`. Cache 60s. |
| Tue | full team | **Critical regression test:** verify existing TikTok seller integration is completely unaffected. Full pass: connect a seller account, sync products, sync orders, push inventory. Compare against pre-Phase-1 baseline. |
| Tue | full team | Smoke test against sandbox: 1 FBT account + 1 seller account on **same client** — both work, no cross-contamination. 5 FBT SKUs, 10 FBT orders, 5 seller orders end-to-end. |
| Wed | full team | Fix smoke-test bugs. |
| Thu | dev + QA | Final pass: error states, edge cases (account disconnect mid-sync, expired token, malformed webhook). PII redaction verified in logs. |
| Fri | full team | Phase 1 ship: deploy to staging, run for 24h, then prod. Feature flag enabled for 1 internal pilot merchant. |

**Phase 1 Definition of Done:** see [README §7](./README.md).

---

## Phase 2 — Inbound shipments (3 weeks)

### Week 5 — Inbound creation backend

| Day | Workstream | Task |
|---|---|---|
| Mon | backend | Migrations A7, A8 (new repo) and B5 (core-service). |
| Mon-Tue | backend | `FbtInboundService.createPlan()`: validates SKU stock in source warehouse, calls TikTok plan API, persists `fbt_inbound_shipments` + items. Publishes `tiktok_fbt.to.wms.inbound_created` → core-service inserts `inbound_shipments`. |
| Wed | backend | `confirmPlan`, `cancelPlan`, `getLabel` endpoints. |
| Thu | backend | Status polling cron @ 30min. Updates status, `units_received`. Webhook handler for `FBT_INBOUND_RECEIVED` if TikTok publishes. |
| Fri | backend (core) | Inventory side-effect: on receipt, decrement source warehouse, **rely on next FBT inventory poll** to surface new fulfillable. No double-write. |

### Week 6 — Inbound wizard frontend

| Day | Workstream | Task |
|---|---|---|
| Mon | frontend | `<InboundShipmentsListPage>` — table, status filters. |
| Tue-Thu | frontend | 4-step wizard: products → FC → review → labels & ship. Shared wizard state via `useTiktokFbtInboundStore`. |
| Fri | frontend | Detail page (`/clients/tiktok-fbt/inbound/[id]`) with timeline, line items, dispute stub. |

### Week 7 — Hardening, edge cases, ship

| Day | Workstream | Task |
|---|---|---|
| Mon-Tue | full team | Test partial receipt, damaged units, short receipts, cancel-before-ship, oversell at plan creation. |
| Wed | backend | Reimbursement stub: on partial/damaged receipt, auto-create `marketplace_reimbursement` placeholder (status pending). |
| Thu | frontend | In-app + email notifications: shipment arrives, discrepancy detected. |
| Fri | full team | Phase 2 ship to staging, then prod (flag-gated). |

**Phase 2 Definition of Done:**
- [ ] Wizard creates plan in TikTok sandbox successfully.
- [ ] Label PDF renders and is downloadable.
- [ ] Tracking entry moves status to `in_transit`.
- [ ] Receive event updates status to `received`.
- [ ] Source warehouse decremented; FBT inventory shows receipt within next poll.
- [ ] Pilot merchant ships 1 real shipment.

---

## Phase 3 — Fees, reimbursements, returns (3 weeks)

### Week 8 — Fee ingestion

| Day | Workstream | Task |
|---|---|---|
| Mon | backend | Migrations A9, A10 (new repo) and B6, B7 (core-service). |
| Mon | backend | `FbtStatementService` daily cron @ 03:15. Fetches yesterday's statement, parses by line type, inserts `fbt_fees`. |
| Tue-Wed | backend | Publish to `tiktok_fbt.to.wms.fee_incurred` per line. Core-service consumer → `marketplace_charges`. Reconciliation linker matches by `(external_order_id, charge_type)`. |
| Thu | backend | Reimbursement ingestion: same flow, → `marketplace_reimbursements`. |
| Fri | backend | Discrepancy detector cron in core-service: flags suspicious lines. |

### Week 9 — Fees UI + returns UI

| Day | Workstream | Task |
|---|---|---|
| Mon-Wed | frontend | `<FbtFeesPage>` — stat cards, bar chart, fee table, reimbursement table, discrepancy alerts. |
| Thu | frontend | Returns: extend existing returns UI with FBT marketplace metadata. Filter "Initiated by marketplace". |
| Fri | full team | End-to-end: simulate FBT order → ship → return → reimbursement appears in fees page. |

### Week 10 — Dispute flow, hardening, ship

| Day | Workstream | Task |
|---|---|---|
| Mon-Tue | backend | Dispute submission flow: UI → `wms.to.tiktok_fbt.dispute_submit` → service calls TikTok dispute API (or surfaces external link if web-only). |
| Wed | backend | Currency display: USD-vs-GBP separation throughout dashboards. No silent conversion. |
| Thu | full team | QA. Edge cases. Performance check (statement payloads can be large). |
| Fri | full team | Phase 3 ship. |

**Phase 3 Definition of Done:**
- [ ] Daily statement ingested for 7 consecutive days without errors.
- [ ] Fees by type and SKU display correctly.
- [ ] One reimbursement claim observed end-to-end in pilot data.
- [ ] One dispute submitted (or external link surfaced).
- [ ] Discrepancy detector finds at least one real edge case in pilot data.

---

## Hardening + public launch (2 weeks)

### Week 11
- Public docs: how to enable FBT, OAuth steps, troubleshooting.
- Sales enablement one-pager.
- Load test: 100 accounts × 1000 SKUs × 5000 orders/day.
- Observability dashboards: sync latencies, error rates per region, queue depths.

### Week 12
- 2nd pilot merchant (real volume).
- Iterate on feedback.
- Open feature to all UK merchants on appropriate subscription tier.

### Post-launch: US region (~2 weeks)
- Add `region='US'` paths in account onboarding.
- USD currency support.
- US addressing.
- US-specific FCs in destination dropdown.
- Regression-test UK paths.

---

## Critical-path summary

If any of these slip, the whole plan slips:

1. **Pre-week sandbox access** (5 days). Without it, Phase 1 cannot finish.
2. **Phase 0 scaffolding** (5 days). New service must be up before Phase 1 week 1 has anywhere to land code.
3. **Phase 1 week 1 API research + auth wiring**. Schemas in [02](./02-data-model-changes.md) assume payload shapes that must be verified.
4. **Phase 1 week 3 order routing**. If FBT orders don't reliably skip pick/pack, value proposition breaks. Test must be a first-class integration test.
5. **Phase 2 week 5 inbound API**. If endpoint surface differs materially from assumptions, Phase 2 needs a re-plan day.
6. **Phase 3 week 8 statement format**. Most data-format-sensitive work in the plan.

Each item has 1-2 days of buffer within its week.

---

## Team and ownership matrix (suggested)

| Role | Phase 0 | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|---|
| Tech lead | Scaffolding review, schema review | API research, integration test | Workflow design | Financial reconciliation review |
| Backend eng A | Clone & strip starter repo | New service fbt-accounts + inventory | Inbound API | Statement ingestion + dispute |
| Backend eng B | RabbitMQ + gateway wiring | Core-service consumer + order routing | Inbound consumer + inventory side-effects | Charge/reimbursement consumers + discrepancy detector |
| Frontend eng | (off rotation; catch up on existing tickets) | All 4 Phase 1 pages | Inbound list + wizard | Fees page + returns extension |
| DevOps | CI/CD, gateway registration, deploy pipeline | Monitoring & alerting | Performance | Statement-volume capacity check |
| QA | (scaffolding sanity) | EoW2, EoW3, EoW4 | EoW7 | EoW10 |

If team is smaller (1 backend), shift Phase 2 +1 week and Phase 3 +1 week.
