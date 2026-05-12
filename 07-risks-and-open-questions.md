# 07 — Risks & Open Questions

> What we don't know, what could break, what needs a decision. This shrinks over the first 1-2 weeks as sandbox access lands and TikTok answers questions.

---

## A. Critical risks (could block or delay shipping)

### A1. TikTok FBT seller account onboarding lead time

**Risk:** Manual programme application; review can take 1-4 weeks. We do not yet have a live FBT seller account or sandbox.

**Impact:** Phase 1 cannot complete without working API access. Worst case: 4-week slip.

**Mitigation:**
- Apply day 1.
- Engage TikTok partner manager if regional contact exists.
- Plan B: develop against pre-recorded API fixtures from public partners' samples — gets us most of Phase 1 done with final smoke testing waiting on real access.

**Decision needed:** which entity (internal vs friendly merchant) submits the FBT application? — **Mohammad to confirm.**

---

### A2. API documentation gaps for FBT-specific endpoints

**Risk:** Seller API is well-documented; FBT-specific endpoints (inbound shipments, fees, returns) are sparser. Community write-ups describe workflows, rarely show payloads.

**Impact:** Schemas in [02](./02-data-model-changes.md) may need revision once we see real payloads. Most affected: JSONB shapes (`inbound_breakdown`, `reserved_breakdown`, `unfulfillable_breakdown`); possibly the `fbt_fees` table structure.

**Mitigation:**
- Week 1 dedicated to live testing.
- Use JSONB columns for marketplace-shaped data — pivoting later is a query change, not a migration.

---

### A3. Hybrid SKU oversell / split-pool race conditions

**Risk:** Same SKU listed as FBM (in old service) + FBT (in new service). Both listings advertise stock; if our FBM feed lags, TikTok may accept FBM orders we haven't decremented yet.

**Impact:** Single-SKU oversells, especially for high-velocity items.

**Mitigation:**
- Feed-out values decoupled: own-warehouse stock → FBM listing (set by old service); TikTok FC stock → FBT listing (we never push, TikTok owns).
- Existing `feeder_flag` + `customFeederQuantity` in `tiktok_products` (old service) cap FBM listing stock. Phase 1 verifies this continues to work.
- Reservation TTL (`TIKTOK_FBT_ORDER_RESERVATION_TTL=900`) holds FBT inventory in cache 15 min after webhook arrival.

---

### A4. Reimbursement reconciliation accuracy (Phase 3)

**Risk:** Settlement reports may not have clean 1:1 mapping between fee lines and our orders. TikTok may bundle, defer, amend.

**Impact:** Any discrepancy between `marketplace_charges` total and merchant's TikTok payout erodes trust.

**Mitigation:**
- Store raw statement payload in `fbt_fees.external_metadata` — source-of-truth preserved.
- Reconciliation cron flags discrepancies; never silently mutates data.
- UI offers raw statement as downloadable artefact alongside parsed view.
- Frame Phase 3 as "visibility + alerting", not "automated bookkeeping". Merchant's accountant still owns the books.

---

### A5. Cloned code drift between two TikTok services

**Risk:** Phase 0 clones ~1500 lines of auth, signing, token-refresh, webhook-verification, RabbitMQ scaffolding from existing tiktok-microservice. A security fix or bug fix in one service can be forgotten in the other.

**Impact:** Most likely failure: HMAC signing logic gets a patch in tiktok-microservice when TikTok rotates an algorithm, FBT service still uses the old algorithm and starts failing webhook verification. Worst-case impact: silent vulnerability where one service is hardened and the other is not.

**Mitigation:**
- Document a "file-pair list" in the new repo README. Files cloned from old service are explicitly named, with the source file path noted at the top of each.
- Quarterly audit: diff the cloned files against their source. PR-required reconciliation. Add to engineering team's recurring agenda.
- If a third TikTok-flavoured service ever lands (TikTok Live Shopping?), extract a shared lib then. Until then, two-way duplication is acceptable.

**Cloned files to track:**
- `common/utils/token-encryption.util.ts` — AES-256-GCM
- `common/guards/webhook-signature.guard.ts` — HMAC verification
- `common/services/{tiktok|tiktok-fbt}-api.service.ts` — signing logic (core function ~lines 87-110 in original)
- `common/services/token-refresh-cron.service.ts`
- `integrations/rabbitmq/rabbitmq.module.ts`
- OAuth controller scaffolding in `modules/{tiktok-auth|fbt-auth}/`

---

## B. Non-critical risks

### B1. 15-min inventory refresh feels slow to merchants
Same mitigation as before: surface lag in UI prominently; optimistic decrement on webhook arrival.

### B2. TikTok Seller Center UI changes break user mental model
Keep merchant-facing docs minimal. Don't screenshot. Link to TikTok's own enablement page.

### B3. Webhook delivery reliability
Best-effort by TikTok with retries. 5-min order-reconciliation cron is defence in depth.

### B4. Currency / multi-currency reporting
Every monetary column has `currency`. Dashboard aggregates per currency, never silently converts.

### B5. PII redaction / GDPR
FBT orders contain customer PII. Verify in Phase 1 week 4 that the new service's PII retention cron is implemented (or that existing core-service retention covers `orders.createdVia='tiktok_fbt'`).

### B6. Existing TikTok seller account → FBT account linking UX
A merchant may have multiple seller shops and not understand which to link. The `linked_seller_account_id` field is informational only — a "Change link" UI on the account card makes this fixable any time. Default is unlinked; we never auto-link.

### B7. (was A5 — moved to critical) — see above.

### B8. Two RabbitMQ consumers competing for the same exchange
The new service publishes/subscribes on the existing `wms_multichannel` exchange. Routing keys are disjoint (`tiktok.*` vs `tiktok_fbt.*`), so there's no collision in normal operation. But: if someone writes a wildcard binding (`tiktok*.to.wms.*`), both services would process events. Mitigation: explicit routing-key tests in the consumer test suite. Document the naming convention in both services' README files.

### B9. Frontend bundle size grows with two parallel TikTok feature sets
Each marketplace page set adds ~30-50KB of JS. Two TikTok sets ≈ ~80KB combined. Not a problem; documented for awareness.

### B10. Operational complexity doubles per region
Adding US later means: a US tiktok-fbt-microservice (or US-region config inside same instance), a US `fbt_accounts` row policy, US-specific FC dropdown. Same complexity exists for seller integration today; not new, just doubled.

---

## C. Open questions — must be answered by end of week 1

1. **Sandbox availability** — does TikTok provide an FBT-enabled sandbox shop? (Critical path.)
2. **App credentials** — same app key/secret as seller, or separate registration? Most likely the latter for clean credential separation, but TikTok Partner Center may permit one app for multiple capabilities. Test both.
3. **Scope names** — exact OAuth scope strings for FBT inventory, orders, inbound, fees. (Echoes memory `feedback_etsy_oauth_scopes.md`: do not trust third-party guides for scope names.)
4. **Webhook event types for FBT** — any FBT-specific events, or do we rely on existing seller event types and differentiate by payload?
5. **Sandbox label generation** — does sandbox return real PDF labels, dummy URLs, or 501? (Phase 2.)
6. **Statement format** — exact JSON shape of `/finance/202309/statements`. (Phase 3.)
7. **Long-term storage threshold** — day count at which TikTok starts charging long-term storage. (Phase 3.)
8. **Return-to-merchant option** — for unsellable returns, TikTok offers return-to-merchant or only destruction? (Phase 3.)
9. **Hazmat / restricted categories** — list of FBT-ineligible categories. (Phase 1 nice-to-have; Phase 2 must-have.)
10. **Port 5018 final allocation** — verify no production conflict (memory `reference_port_allocation.md` is 35 days old).

---

## D. Decisions already made (closed)

| Decision | Choice | When |
|---|---|---|
| Microservice strategy | **Brand-new `tiktok-fbt-microservice`. Existing `tiktok-microservice` untouched.** | 2026-05-12 (revised) |
| Region scope v1 | UK only first, US next | 2026-05-12 |
| Hybrid SKU support | Yes — same SKU as FBM + FBT simultaneously, with split inventory via `variation_listings` | 2026-05-12 |
| Account model | Independent FBT account row in own DB. Optional soft link to seller account via `linked_seller_account_id` (no FK, no cross-DB join) | 2026-05-12 |
| Phasing | 1w (scaffolding) + 4w (mirror) + 3w (inbound) + 3w (fees) | 2026-05-12 |
| Inventory polling cadence | 15 min | 2026-05-12 |
| Pick/pack handling for FBT orders | Published to core WMS as `createdVia='tiktok_fbt'`, OrderFulfillment(`type='fbt'`). Pickers excluded via `WHERE fulfillment_type != 'fbt'` filter | 2026-05-12 |
| Inbound shipment model | Generalised `inbound_shipments` table in core-service; new service's `fbt_inbound_shipments` is the marketplace-side mirror | 2026-05-12 |
| Channel registration | New `Channel(channelSlug='tiktok_fbt')` row in core WMS. Sibling to existing `Channel(channelSlug='tiktok')` | 2026-05-12 |
| Frontend sidebar | New top-level "TikTok FBT" entry. **Not** a child of "TikTok Shop". | 2026-05-12 |
| RabbitMQ topic family | `tiktok_fbt.*` (new) — disjoint from existing `tiktok.*` (untouched) | 2026-05-12 |

---

## E. Future scope (post-launch backlog)

- Multi-country FBT (UK seller selling on US FBT and vice versa).
- TikTok Live Shopping order fulfilment integration.
- Direct supplier-to-FBT inbound (drop-ship-to-FBT model).
- Forecasting / replenishment AI based on velocity trends.
- Native integration with merchant carriers (DPD, Royal Mail) for inbound label printing.
- Public API exposing FBT data to merchants' own systems.
- Reverse logistics: returned-unsellable units returning to merchant warehouse instead of disposal.
- Comparable rollout for Amazon FBA inbound shipments (reuses `inbound_shipments` table).
- **Shared TikTok lib** — once a 3rd TikTok-flavoured service exists (Live Shopping?), extract auth/signing/token utilities. Until then, two-way clone-and-document is acceptable.
