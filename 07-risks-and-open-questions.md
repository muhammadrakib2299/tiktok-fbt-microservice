# 07 — Risks & Open Questions

> What we don't know, what could break, what needs a decision. Shrinks as week 1 lands.

---

## A. Critical risks (could block or delay shipping)

### A0. Eligibility verification endpoint — **RESOLVED 2026-05-12 via probe pattern**

**Original risk:** the Enable-FBT flow assumed `GET /authorization/202309/shops` exposed a per-shop FBT enrollment flag. It does not — that endpoint only returns `id` and `region` per authorized shop. Verified by live response inspection on 2026-05-12:

```json
{
  "code": 0,
  "data": { "shops": [ { "id": "36123502970007", "region": "GB" } ] },
  "message": "Success",
  "request_id": "..."
}
```

There is no `fbt_enrolled`, `capabilities`, or `fulfillment_programs` field. The endpoint name was misleading — "authorization/shops" returns "which shops authorized this app", not "this shop's authorization details".

**Resolution: probe pattern.** Instead of reading a metadata flag, `verifyShopEligibility()` makes a minimal, cheap call to an actual FBT endpoint and infers enrollment from the response. Confirmed available: **FBT inventory endpoint** (per Mohammad's docs check, 2026-05-12).

**Verification algorithm** (probe semantics — same UX externally as the original design):

```
1. POST to FBT inventory endpoint with shop_id=<account.shop_id>, limit=1.
2. Inspect response. TikTok wraps every response in an envelope
   { code, data, message, request_id } — code=0 means success.

   HTTP 200 + envelope code=0      → ENROLLED.
                                     Store any returned shop metadata in
                                     fbt_capabilities JSONB.
                                     Set fbt_enabled = true,
                                         fbt_last_verified_at = NOW().

   HTTP 200 + code≠0 with a        → NOT ENROLLED.
   "not enrolled"-class error        Show "apply on Seller Center" toast.

   HTTP 200 + code≠0 with other    → UNKNOWN.
   error code                        Generic error toast, do NOT flip flag,
                                     log for triage.

   HTTP 4xx                         → AUTH/PERMISSION ISSUE.
                                     "Permission issue, contact support" toast.

   HTTP 5xx / network               → TIKTOK DOWN / RETRYABLE.
                                     "Try again in a moment" toast, no flag flip.
```

**Why this works:** the probe is the cheapest possible call (`limit=1`), takes one round-trip, and asks the only authoritative source — TikTok — by *trying to use* FBT. Any answer other than "200 + code=0" means "not currently usable", which is functionally what we needed to know. We never store the inventory rows from this probe; we discard the response body after reading the envelope.

**Trade-off accepted:** we lose the ability to pre-populate a rich capabilities summary (regions covered, sub-features) on the modal success path unless TikTok's inventory response carries shop-level metadata. If it doesn't, the success state shows "FBT enabled — first inventory sync starting" instead of the per-capability bullet list we mocked. Acceptable: the merchant needs confirmation, not a feature inventory.

**Still required before production:** one real successful probe against the sandbox FBT shop (once provisioned per risk A1). The probe's exact request path, payload shape, and the specific TikTok error-code values for "not enrolled" must be captured in `fbt-accounts/fbt-accounts.README.md`. Until then we build against the documented FBT inventory contract.

**Owner:** Mohammad. **Status:** RESOLVED (verification mechanic), pending sandbox-side smoke test for the exact error-code mapping.

---

### A1. TikTok FBT seller account onboarding lead time

**Risk:** Manual programme application via Seller Center. Review can take days to a few weeks.

**Impact:** Phase 1 can scaffold without it, but cannot smoke-test or ship without a real or sandbox FBT-enabled shop.

**Mitigation:**
- Apply day 1.
- Engage TikTok partner manager if regional contact exists.
- Plan B: develop against pre-recorded API fixtures.

**Decision needed:** which entity submits the FBT application — your company's TikTok shop, or a friendly merchant's? — **Mohammad to confirm.**

---

### A2. API documentation gaps for FBT-specific endpoints

**Risk:** Seller API is well-documented; FBT inbound/fees/returns are sparser. Community write-ups describe workflows, rarely show payloads.

**Impact:** Schemas in [02](./02-data-model-changes.md) may need revision once we see real payloads. Most affected: JSONB shapes (`inbound_breakdown`, `reserved_breakdown`); possibly the `fbt_fees` table.

**Mitigation:**
- Week 1 dedicated to live testing.
- JSONB columns for all marketplace-shaped data — pivoting later is a query change, not a migration.

---

### A3. Existing TikTok app scope coverage — **RESOLVED 2026-05-12**

**Outcome:** The existing TikTok app already carries FBT API permissions. No second app registration needed, no new credential model. The plan's "reuse existing credentials" assumption holds. Verified by Mohammad via TikTok Partner Center scope inspection.

**Implication for the plan:**
- No changes to `.env` shape — same `TIKTOK_APP_KEY` / `TIKTOK_APP_SECRET` cover both seller and FBT endpoints.
- No changes to the OAuth flow in `tiktok-microservice/src/modules/tiktok-auth/`.
- `verifyShopEligibility()` in `fbt-accounts.service.ts` can call FBT endpoints using the existing per-account access token without scope upgrade.

(Original risk text retained below for audit trail.)

> ~~The existing TikTok app's OAuth scope grant may not cover FBT endpoints. TikTok may require either: (a) adding FBT scopes to the existing app's permission set, or (b) registering a brand-new app dedicated to FBT.~~

---

### A4. Hybrid SKU oversell / split-pool race conditions

**Risk:** Same SKU listed as FBM (seller path) + FBT (new path). Both listings advertise stock; if our FBM feed lags, TikTok may accept FBM orders before we decrement.

**Impact:** Single-SKU oversells, especially high-velocity items.

**Mitigation:**
- Feed values decoupled: own-warehouse stock → FBM listing (seller flow), TikTok FC stock → FBT listing (TikTok owns, we never push).
- Existing `feeder_flag` + `customFeederQuantity` continues to cap FBM listing stock. Phase 1 verifies.
- Reservation TTL (`TIKTOK_FBT_ORDER_RESERVATION_TTL=900`) holds FBT inventory in cache 15 min after webhook arrival.

---

### A5. Reimbursement reconciliation accuracy (Phase 3)

**Risk:** Settlement reports may not have a clean 1:1 mapping between fee lines and our orders. TikTok may bundle, defer, or amend.

**Impact:** Discrepancy between `marketplace_charges` total and merchant's actual TikTok payout erodes trust.

**Mitigation:**
- Store raw statement payload in `fbt_fees.external_metadata` — source of truth preserved.
- Reconciliation cron flags discrepancies; never silently mutates data.
- UI offers raw statement as downloadable artefact alongside parsed view.
- Frame Phase 3 as "visibility + alerting", not "automated bookkeeping". Merchant's accountant still owns the books.

---

## B. Non-critical risks

### B1. Conditional sidebar rendering bug class

**Risk:** Three independent flags (`fbm_enabled`, `fbt_enabled`, `clientHasHistoricalFbmOrders`) drive the three-view sidebar. If any of them is mis-read, mis-aggregated across accounts, or the store isn't refreshed after a toggle change, merchants either see menus they shouldn't or miss menus they should — including the worst case where an FBT-only client loses access to the Accounts page entry that holds the toggle to re-enable FBM.

**Mitigation:** explicit test matrix in Phase 1 EoW3:
- 0 accounts → no FBT or FBM submodules, Accounts entry still visible
- 1 FBM-only account → General view only
- 1 FBT-only account, no historical FBM orders → FBT-only view
- 1 FBT-only account, historical FBM orders present → Hybrid view (General stays visible)
- 1 hybrid account (both flags ON) → Hybrid view
- Mixed: 1 FBM-only + 1 FBT-only on same client → Hybrid view
- Transition tests: toggle FBT on/off and FBM off/on, verify sidebar reshapes within one render tick without a hard refresh
- Constraint test: attempt to disable both modes on the last toggle — DB rejects via `chk_accounts_any_mode`, UI disables the control before submit

### B2. 15-min inventory refresh feels slow

**Mitigation:** UI surfaces lag prominently; optimistic decrement on webhook arrival.

### B3. TikTok Seller Center UI changes break merchant docs

**Mitigation:** keep merchant-facing docs minimal. Don't screenshot. Link to TikTok's own pages.

### B4. Webhook delivery reliability

**Mitigation:** 5-min reconciliation cron is defence in depth.

### B5. Currency / multi-currency reporting

**Mitigation:** every monetary column has `currency`. Dashboard aggregates per currency; never silently converts.

### B6. PII redaction / GDPR

**Mitigation:** verify in Phase 1 week 4 that existing PII retention cron handles `tiktok_orders WHERE fulfilment_mode='fbt'` rows. It already filters by date, not mode, so should be fine.

### B7. Two RabbitMQ message families on one exchange

**Risk:** routing keys `tiktok.*` and `tiktok_fbt.*` are disjoint, so no collision in normal operation. But a careless wildcard binding (`tiktok*.to.wms.*`) would catch both.

**Mitigation:** explicit routing-key tests in the consumer test suite. Document naming convention in the codebase README.

### B8. Module dependency cycles

**Risk:** the `fbt/` submodules depend on `accounts/`, `orders/`, `webhooks/`. If anyone wires the inverse, NestJS DI breaks.

**Mitigation:** explicit module-import direction — FBT modules import existing modules' services via `forwardRef` only if absolutely necessary (rarely should be).

---

## C. Open questions — re-ranked by criticality

> **Top three are week-1 day-1 blockers.** Answering them gates almost every downstream design decision. Items 4–9 can be answered in parallel during weeks 1–2.

### Must answer before any FBT-side code is written (day 1)

1. ~~**Eligibility verification endpoint shape**~~ — **RESOLVED 2026-05-12 via probe pattern.** `/authorization/202309/shops` doesn't expose FBT enrollment; we probe the FBT inventory endpoint with `limit=1` and infer enrollment from the response envelope. See risk A0 (closed).
2. ~~**App scope coverage**~~ — **RESOLVED 2026-05-12.** Existing TikTok app already carries FBT API permissions. See risk A3 (closed).
3. **Sandbox availability** — does TikTok provide an FBT-enabled sandbox shop? See risk A1. Gates the live error-code mapping for the probe and end-to-end smoke testing for the rest of Phase 1. **Partner Center ticket drafted 2026-05-12, submission pending.**

### Must answer by end of week 1

4. **Exact OAuth scope names** — per existing memory `feedback_etsy_oauth_scopes.md`, third-party guides are unreliable for scope strings.
5. **Webhook event types for FBT** — any FBT-specific events, or do we rely on existing events + payload differentiation? Gates the `webhooks/` dispatcher extension.

### Phase 2 prerequisites

6. **Sandbox label generation** — does it return real PDFs, dummy URLs, or 501?
7. **Hazmat / restricted categories** — list of FBT-ineligible categories (nice-to-have Phase 1, must-have Phase 2).

### Phase 3 prerequisites

8. **Statement format** — exact JSON shape of `/finance/202309/statements`.
9. **Long-term storage threshold** — day count at which TikTok starts charging LTS.
10. **Return-to-merchant option** — for unsellable returns, does TikTok offer return-to-merchant or only destruction?

---

## D. Decisions already made (closed)

| Decision | Choice | Date |
|---|---|---|
| Microservice strategy | **Extend existing `tiktok-microservice` with `fbt/` submodule.** No new microservice. | 2026-05-12 (re-confirmed after pivoting back from separate-service plan) |
| Account model | One `accounts` row per shop. FBT enabled via toggle (boolean + capabilities JSONB). No `fbt_accounts` table. | 2026-05-12 |
| Region scope v1 | UK only first, US next | 2026-05-12 |
| Hybrid SKU support | Yes — same SKU FBM + FBT simultaneously, split inventory via `variation_listings` | 2026-05-12 |
| Sidebar | One TikTok Shop parent with **three conditional views** (General-only · FBT-only · Hybrid) driven by aggregated `fbm_enabled` / `fbt_enabled` flags + residual-FBM-orders safeguard. Accounts entry always present. | 2026-05-12 |
| Orders page | Single central orders page, filter chip `All/Seller/FBT`, per-row badge | 2026-05-12 |
| Enable-FBT flow | Toggle + verification API call to TikTok to confirm shop eligibility | 2026-05-12 |
| Inventory polling cadence | 15 min | 2026-05-12 |
| Pick/pack handling for FBT | Published to core WMS with `createdVia='tiktok_fbt'`, `order_fulfillments.fulfillment_type='fbt'`. Picker queries filter on this. | 2026-05-12 |
| Inbound shipment model | Generalised `inbound_shipments` table in core-service; service-side `fbt_inbound_shipments` is the marketplace mirror | 2026-05-12 |
| Channel registration | New `Channel(channelSlug='tiktok_fbt')` row in core WMS, sibling to existing `tiktok` | 2026-05-12 |
| Operational kill-switch | `accounts.publish_fbt_to_wms` (admin-only, default TRUE) — separate from merchant-facing `fbt_enabled`. Pattern borrowed from Amazon FBA's `publish_fba_to_wms` (`LESSONS.md` line 6 records a 421-order leak that motivated this column). Every publisher fail-closes against both flags. | 2026-05-12 |
| Fail-closed guard pattern | All FBT publishers and consumers must check `account.fbt_enabled === true && account.publish_fbt_to_wms === true` with strict equality, treating null/undefined/coerced values as `false`. Required test cases enumerated in `06-deployment-and-observability.md` §12. | 2026-05-12 |
| App scope coverage | Existing TikTok app already carries FBT API permissions — no second app, no scope upgrade, no credential-model change. Verified via Partner Center scope inspection. | 2026-05-12 |
| Eligibility verification mechanic | **Probe pattern.** `GET /authorization/202309/shops` does NOT expose FBT enrollment (only `id` + `region`). Instead, `verifyShopEligibility()` calls the FBT inventory endpoint with `limit=1` and infers enrollment from the response envelope: `code=0` → enrolled; specific error codes → not enrolled; other errors → unknown. UX is unchanged from the original design (same modal, same toasts) — only the underlying API call differs. | 2026-05-12 |
| RabbitMQ topic family | `tiktok_fbt.*` (new) — disjoint from existing `tiktok.*` (untouched) | 2026-05-12 |
| Phasing | 4w (mirror) + 3w (inbound) + 3w (fees) | 2026-05-12 |

---

## E. Future scope (post-launch backlog)

- Multi-country FBT (UK seller selling on US FBT and vice versa)
- TikTok Live Shopping order fulfilment integration
- Direct supplier-to-FBT inbound (drop-ship-to-FBT model)
- Forecasting / replenishment AI based on velocity trends
- Native carrier integration (DPD, Royal Mail) for inbound label printing
- Public API exposing FBT data to merchants' own systems
- Reverse logistics: returned-unsellable units returning to merchant warehouse instead of disposal
- Amazon FBA inbound shipments reusing `inbound_shipments` table
