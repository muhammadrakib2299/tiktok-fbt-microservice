# TikTok FBT Integration Plan — WMS360

**Status:** Planning · **Owner:** Mohammad Rabbani · **Architecture finalised:** 2026-05-12 · **Target Phase 1 launch:** 2026-06-13 (4 weeks out)

> Fulfilled by TikTok (FBT) is TikTok Shop's marketplace-fulfilment programme. Merchant ships inventory to TikTok fulfilment centres → TikTok stores, picks, packs, ships orders → merchant only handles inbound, returns reconciliation, and stock replenishment.

---

## 1. Final architecture (locked)

| Decision | Choice | Rationale |
|---|---|---|
| Where the code lives | **Inside the existing `D:\office porjects\tiktok-microservice` repo, in a new `src/modules/fbt/` submodule.** No new microservice. | One TikTok shop = one connection. FBT is a *capability* of that connection, not a separate service. |
| Launch region | UK first, US next | Single currency + single FC network for v1 |
| Hybrid SKU support | Yes — same SKU can be sold as FBM and FBT simultaneously | Inventory split via per-warehouse-type counts in core WMS |
| Account model | **One row per shop in the existing `accounts` table.** FBT is enabled via a setting toggle on that account, gated by `fbt_enabled` boolean + `fbt_capabilities` JSONB + verification call to TikTok. | Mirrors how TikTok itself models FBT — a programme you enrol an existing shop in. |
| Sidebar | **Three views under "TikTok Shop"**: General-only (today), FBT-only, or Hybrid — driven by per-account `fbm_enabled` / `fbt_enabled` flags aggregated across the client's accounts. | Conditional rendering keeps the sidebar tight for both directions: FBM-only merchants don't see FBT items, FBT-only merchants don't see seller listing items. |
| Orders | **FBT orders live on a dedicated `/clients/tiktok/fbt/orders` page only.** The central `/clients/order` page is unchanged — it shows all other channels (eBay, Shopify, Amazon, TikTok seller, etc.) but never FBT. | FBT orders are read-only and need zero merchant action; they don't belong in the operational queue. Keeping them separate keeps the central page focused on orders that need attention. |
| MVP scope | **Phase 1: read-only mirror (4w) → Phase 2: inbound wizard (3w) → Phase 3: fees & reimbursements (3w)** | Ship value early, validate TikTok FBT API quirks before heavy investment. |

---

## 2. End-state in one paragraph

The existing `tiktok-microservice` (port 5011) gains a new `src/modules/fbt/` directory containing FBT-specific modules: inventory polling, inbound shipments, fees ingestion, dashboard aggregation. The existing `accounts` entity gains a few FBT-aware columns; existing `tiktok_orders` gains a `fulfilment_mode` column. FBT-only data (inventory snapshots, inbound shipments, fees) lives in **new tables in the same database** prefixed `fbt_*`. RabbitMQ uses a new `tiktok_fbt.*` routing-key family (disjoint from existing `tiktok.*`) so core-service can distinguish events. Core-service learns a new channel slug (`tiktok_fbt`), a new warehouse type (`fbt_tiktok`), and a new fulfilment type (`fbt`) on `order_fulfillments`. The merchant sees FBT submenus appear under "TikTok Shop" the moment they flip the "Enable FBT" toggle on any account card.

---

## 3. Plan structure (read in this order)

1. **[01-architecture-and-alignment.md](./01-architecture-and-alignment.md)** — how the new `fbt/` submodule plugs in; conditional sidebar logic.
2. **[02-data-model-changes.md](./02-data-model-changes.md)** — additive columns on existing tables + new `fbt_*` tables in the same DB; core WMS DB changes too.
3. **[03-api-research-checklist.md](./03-api-research-checklist.md)** — TikTok FBT endpoints to verify in sandbox.
4. **[04-pages-and-workflows.md](./04-pages-and-workflows.md)** — pages that change (Accounts) and pages that get added.
5. **[05-phased-rollout.md](./05-phased-rollout.md)** — week-by-week task breakdown. No "Phase 0 scaffolding" anymore.
6. **[06-deployment-and-observability.md](./06-deployment-and-observability.md)** — env vars added to existing service, migrations, RabbitMQ routing keys, rollback.
7. **[07-risks-and-open-questions.md](./07-risks-and-open-questions.md)** — what we don't know yet.
8. **[08-ui-lifecycle-mockups.md](./08-ui-lifecycle-mockups.md)** — ASCII mockups for every screen state.
9. **[09-folder-structure.md](./09-folder-structure.md)** — before/after layout of `tiktok-microservice/src/` showing every file added or extended.
10. **`mockup.html`** — open in a browser for the full visual lifecycle.

---

## 4. What stays the same (existing service guarantees)

- Existing `accounts` table, seller OAuth, seller order sync, seller inventory push to TikTok — **untouched at the behavioural level**. The only column additions to `accounts` are `fbt_enabled`, `fbt_capabilities`, `fbt_enabled_at`, `fbt_last_verified_at`; defaults keep existing rows behaving identically.
- Existing modules (`accounts/`, `orders/`, `products/`, `inventory/`, `webhooks/`, `tiktok-auth/`) keep working. Two of them (`orders/`, `webhooks/`) get small extensions to route FBT events to the new submodule.
- Existing port (5011), existing DB (`tiktok_microservice`), existing RabbitMQ exchange (`wms_multichannel`). No infra changes.
- Existing TikTok app credentials reused. FBT scopes added to the existing app's permission set — no separate app registration unless TikTok forces it.

---

## 5. What gets added

### Backend (inside `tiktok-microservice`)

- New `src/modules/fbt/` directory tree (full layout in [09-folder-structure.md](./09-folder-structure.md))
- New entities: `fbt_inventory`, `fbt_webhook_events`, `fbt_inbound_shipments` + items (Phase 2), `fbt_fees` (Phase 3), `fbt_returns` (Phase 3)
- New columns on `accounts`: `fbm_enabled` (default TRUE, merchant-facing), `fbt_enabled` (default FALSE, merchant-facing), `publish_fbt_to_wms` (default TRUE, **admin-only operational kill-switch** — pattern borrowed from Amazon FBA's `publish_fba_to_wms`), `fbt_capabilities`, `fbt_enabled_at`, `fbt_last_verified_at`, `fbt_last_inventory_sync_at`, `fbt_last_order_sync_at`. DB constraint `chk_accounts_any_mode` guarantees at least one fulfilment mode is enabled per row.
- New column on `tiktok_orders`: `fulfilment_mode ('fbm' | 'fbt')`
- New cron jobs: FBT inventory sync (15min), FBT inbound status poll (30min, Phase 2), FBT statement ingestion (daily 03:15, Phase 3)
- New RabbitMQ routing keys (same exchange): `tiktok_fbt.to.wms.*` and `wms.to.tiktok_fbt.*`
- Two new endpoints on existing accounts controller: `POST /accounts/:id/fbt/enable`, `POST /accounts/:id/fbt/disable`

### Core WMS (additive only)

- New channel row: `Channel(channelSlug='tiktok_fbt')`
- New columns on `warehouses`: `warehouse_type`, `external_marketplace`, `external_warehouse_id`, `external_account_id`
- New `variation_listings` table (the hybrid-SKU bridge)
- New tables: `inbound_shipments` + items (Phase 2), `marketplace_charges` + `marketplace_reimbursements` (Phase 3)
- `order_fulfillments.fulfillment_type` accepts new value `'fbt'` (column is permissive string)
- Feature flag `tiktok_fbt` via existing `checkPackageFeature` middleware

### Frontend (inside `frontend_wms_v`)

- **7 new merchant-facing pages** under `/clients/tiktok/fbt/*` (Dashboard, Orders, Inventory, Inbound list, Inbound new, Inbound detail, Fees)
- 1 new admin-only page: `/clients/tiktok/fbt/admin/webhooks`
- Existing Accounts page gains a "Fulfilment modes" panel per account card with **two independent toggles** (FBM + FBT), each with its own settings sub-panel
- Existing TikTok Shop sidebar entry renders one of three views (General-only · FBT-only · Hybrid) based on aggregated per-account flags
- Existing central `/clients/order` page is **UNCHANGED** — FBT orders are filtered out of this view entirely (via `WHERE created_via != 'tiktok_fbt'`)
- New `TiktokFbtService.js` for FBT-specific API calls

---

## 6. Hard alignment principles (non-negotiable)

1. **No new microservice, no new database.** Everything lives in the existing `tiktok-microservice` and its existing DB.
2. **No bypass of core-service.** Cross-channel state lives in WMS DB. Source of truth is core-service.
3. **Hybrid SKU is the default.** Same SKU can have an FBM listing AND an FBT listing. Per-warehouse inventory split via `variation_listings`.
4. **FBT orders skip pick/pack.** Core-service creates `order_fulfillments` row with `fulfillment_type='fbt'`. Picker queries filter on this.
5. **One TikTok shop = one accounts row.** Enabling FBT is a capability toggle, not a second row.
6. **Conditional sidebar — three views.** The "TikTok Shop" parent renders one of: General-only (today), FBT-only, or Hybrid. General sub-items appear if any account has `fbm_enabled = true` (or any historical FBM orders remain); FBT sub-items appear if any account has `fbt_enabled = true`. The "Accounts" entry is always present, because that's where both toggles live.
7. **Two separate order surfaces.** TikTok seller orders + every other channel land on the central `/clients/order` page like today. **FBT orders never appear on the central page.** They live exclusively on the dedicated `/clients/tiktok/fbt/orders` page. The merchant looks at whichever page matches their intent: actionable vs FBT-status.
8. **TikTok is the source of truth for FBT inventory.** WMS only mirrors via 15-min poll. UI surfaces the lag.
9. **All TikTok API calls go through existing `common/services/tiktok-api.service.ts`.** No new HTTP client.
10. **Phase 1 ships zero outbound writes to TikTok FBT.** Read-only mirror only.

---

## 7. Top 6 risks

1. **No TikTok FBT seller account yet.** Critical path. Apply for FBT enrollment on Seller Center week-1 day-1.
2. **API documentation gaps.** Week 1 dedicated to live-testing sandbox endpoints before schema lock.
3. **15-min inventory lag → oversell risk.** Mitigated by optimistic decrement on webhook + UI lag surfacing.
4. **Reorder alert UX must be loud.** Red banner + dashboard tile + email when days-of-cover < 14.
5. **Existing TikTok app scope coverage.** Does it cover FBT endpoints, or does TikTok require a separate app? Research item.
6. **Conditional sidebar rendering bug class.** Test matrix expanded to cover three views: 0 accounts; 1 FBM-only; 1 FBT-only (with and without historical FBM orders); 1 hybrid; mixed accounts (FBM-only + FBT-only on the same client); transitions (toggle FBT on/off, toggle FBM off/on) and verify sidebar reshapes in one tick.

---

## 8. Definition of done (Phase 1)

- [ ] `tiktok-microservice` deployed with `fbt/` submodule. Existing seller flows regression-tested.
- [ ] Merchant can click "Enable FBT" on an account card; verification call confirms eligibility; toggle either succeeds or shows clear "not enrolled" error.
- [ ] Sidebar conditionally shows FBT sub-group based on `fbt_enabled` count.
- [ ] FBT inventory polled every 15 min, visible at `/clients/tiktok/fbt/inventory`.
- [ ] FBT orders appear on the dedicated `/clients/tiktok/fbt/orders` page only. **They do NOT appear on the central `/clients/order` page.** They never appear in the picker queue.
- [ ] FBT orders show TikTok-supplied tracking without merchant action.
- [ ] Reorder alert fires when days-of-cover < 14 (red), < 30 (amber).
- [ ] Feature flag `tiktok_fbt` gates all FBT routes per merchant.
- [ ] Smoke test: one merchant with seller + FBT enabled on same TikTok shop, both work, no cross-contamination.
- [ ] **Fail-closed publisher regression:** flip `accounts.publish_fbt_to_wms=false` via SQL, place a sandbox FBT order, confirm zero `tiktok_fbt.*` messages emitted and `fbt_publish_skipped_total{reason='publish_gate_off'}` counter increments.
- [ ] **`canPublishFbt()` spec passes** all enumerated cases in `06-deployment-and-observability.md` §12 (null/undefined/string-coerced/numeric-coerced inputs all return false).
- [ ] **Open question C1 resolved before any Accounts-page toggle work begins:** eligibility verification endpoint shape captured in `endpoint-payloads.md`; if endpoint absent, fallback path chosen and documented in `fbt-accounts.README.md`.

---

## 9. Alignment with Amazon FBA (lessons folded in)

This plan was reviewed against the existing `amazon-microservice` FBA implementation on 2026-05-12. The two integrations are ~80% structurally aligned (both are submodules inside the marketplace service; both use the snapshot-inventory pattern; both publish to the `wms_multichannel` exchange). We adopted three concrete patterns from Amazon and improved on three others:

**Adopted from Amazon:**
- **Snapshot inventory with `is_latest` flag** — append-only history, never upsert (see `amazon-fba-inventory.entity.ts` lines 106–109; mirrored in our `fbt_inventory` DDL §A3).
- **Pre-compute velocity at snapshot time** — `units_sold_7d`, `units_sold_30d`, `velocity_per_day`, `days_of_cover` stored on the row, not derived at query time.
- **`publish_*_to_wms` operational kill-switch** — admin-only column, separate from user-facing capability flag. Amazon's `LESSONS.md` line 6 records a fail-open guard bug that leaked **421 FBA orders** into the picker queue. Our `publish_fbt_to_wms` + canonical `canPublishFbt()` guard + required spec table closes the same bug class on day one.

**Improved over Amazon:**
- **Strict `fulfilment_mode` enum + CHECK constraint** vs Amazon's plain-text `fulfillment_channel` — routing code can't silently swallow a typo.
- **Split RabbitMQ routing keys** (`tiktok_fbt.to.wms.*` distinct from `tiktok.to.wms.*`) vs Amazon's single `amazon.to.wms.*` namespace — per-mode observability and per-mode consumer scaling.
- **Explicit Enable-FBT verification flow** with eligibility API call vs Amazon's admin-only flag toggle — better merchant UX (modal + clear error states), at the cost of a single open question (C1) about whether TikTok actually exposes an eligibility endpoint.

---

## 10. Out of scope (explicitly)

- Multi-warehouse FBT (TikTok routing across FCs — TikTok handles internally).
- TikTok Live Shopping fulfilment integration.
- Cross-border FBT (UK seller selling US FBT) — Phase 3+.
- Bulk creation of FBT-only listings.
- Merchant-facing public API for FBT data.

---

## 11. Quick links

- Existing service to modify: `D:\office porjects\tiktok-microservice`
- This planning workspace: `D:\office porjects\tiktok-fbt-microservice` (planning artefacts only — code goes in the line above)
- Core-service: `D:\office porjects\backend_wms_v2\core-service`
- Gateway: `D:\office porjects\backend_wms_v2\gateway`
- Frontend: `D:\office porjects\frontend_wms_v`
- TikTok FBT UK reference: https://seller-uk.tiktok.com/university/essay?knowledge_id=7753849801213697&default_language=en-GB
