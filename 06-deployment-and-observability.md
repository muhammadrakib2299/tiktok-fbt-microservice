# 06 — Deployment & Observability

> No new service to deploy. We're adding modules, migrations, env vars, and cron jobs to the existing `tiktok-microservice` deployment artefact.

---

## 1. Environment variables to add

Append to `tiktok-microservice/.env` and `.env.example`:

```bash
# ===== FBT configuration =====

# Regions this instance serves
FBT_ENABLED_REGIONS=GB

# Server-side feature gates (in addition to per-merchant flag)
FBT_ENABLE_INVENTORY_SYNC=true
FBT_ENABLE_ORDER_SYNC=true
FBT_ENABLE_WEBHOOKS=true

# Phase 2 — off until shipped
FBT_ENABLE_INBOUND_SHIPMENTS=false

# Phase 3 — off until shipped
FBT_ENABLE_FEES_INGESTION=false
FBT_ENABLE_DISPUTE_SUBMISSION=false

# Cron schedules
FBT_INVENTORY_CRON=0 */15 * * * *
FBT_ORDERS_CRON=0 */5 * * * *
FBT_INBOUND_STATUS_CRON=0 */30 * * * *
FBT_STATEMENT_CRON=15 3 * * *

# Per-account stagger
FBT_ACCOUNT_STAGGER_SECONDS=30

# Hard ceilings on outbound TikTok requests per minute, per region
FBT_RATE_LIMIT_PER_MINUTE_GB=50
FBT_RATE_LIMIT_PER_MINUTE_US=50
```

### `core-service/.env` — additions

```bash
TIKTOK_FBT_DEFAULT_ENABLED=false                  # per-merchant feature flag default
TIKTOK_FBT_ORDER_RESERVATION_TTL=900              # seconds — how long unconfirmed FBT order holds inventory
```

### `frontend_wms_v/.env.local`

No new env vars. Server-driven sidebar via `useModulePermissionStore`.

---

## 2. Feature-flag rollout

Three feature keys (from [02-data-model-changes.md §C](./02-data-model-changes.md)):

1. `tiktok_fbt` — master switch
2. `tiktok_fbt_inbound` — Phase 2 gate
3. `tiktok_fbt_fees` — Phase 3 gate

| Audience | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|
| Internal pilot | EoW4 ship day | EoW7 | EoW10 |
| External pilot (1 merchant) | week 11 | week 11 | week 11 |
| Limited GA (5-10 merchants) | week 12 | week 12 | week 12 |
| All UK on appropriate tier | post-launch | post-launch | post-launch |

`checkPackageFeature('tiktok_fbt')` middleware enforces.

---

## 3. RabbitMQ topology

Zero infrastructure change. New routing keys on the existing `wms_multichannel` topic exchange.

| Direction | Routing key | Producer | Consumer |
|---|---|---|---|
| → WMS | `tiktok_fbt.to.wms.account_fbt_enabled` | tiktok-microservice (on toggle) | core-service (inserts virtual warehouse) |
| → WMS | `tiktok_fbt.to.wms.account_fbt_disabled` | tiktok-microservice | core-service (marks warehouse stale) |
| → WMS | `tiktok_fbt.to.wms.inventory_synced` | tiktok-microservice | core-service (audit / inventory aggregator) |
| → WMS | `tiktok_fbt.to.wms.order_created` | tiktok-microservice | core-service (FbtOrderConsumer) |
| → WMS | `tiktok_fbt.to.wms.order_status_changed` | tiktok-microservice | core-service |
| → WMS | `tiktok_fbt.to.wms.inbound_status_changed` (P2) | tiktok-microservice | core-service |
| → WMS | `tiktok_fbt.to.wms.return_received` (P3) | tiktok-microservice | core-service |
| → WMS | `tiktok_fbt.to.wms.fee_incurred` (P3) | tiktok-microservice | core-service |
| → WMS | `tiktok_fbt.to.wms.reimbursement_received` (P3) | tiktok-microservice | core-service |
| → service | `wms.to.tiktok_fbt.inbound_create` (P2) | core-service | tiktok-microservice |
| → service | `wms.to.tiktok_fbt.dispute_submit` (P3) | core-service | tiktok-microservice |

Each new consumer gets: a dedicated durable queue, DLX binding to existing `wms_multichannel_dlx`, retry via existing `wms_retry_exchange`, prefetch count 5.

Idempotency: every message carries `wms_message_id` (used by core-service for dedup) and TikTok's `event_id` (used by `fbt_webhook_events` table for inbound dedup).

---

## 4. Cron jobs added to the existing service

All registered via NestJS `@Cron` decorator inside `tiktok-microservice`. Existing crons unchanged.

| Cron | Schedule | Module | Notes |
|---|---|---|---|
| (existing) Token refresh | `0 5,35 * * * *` | common/services | already handles FBT-enabled accounts |
| (existing) Seller order sync | `0 15,45 * * * *` | modules/orders | unchanged |
| (existing) Failed tracking retry | `0 50 * * * *` | modules/orders | unchanged |
| (existing) Pending product status check | `0 30 */2 * * *` | modules/products | unchanged |
| **NEW** FBT inventory sync | `0 */15 * * * *` | modules/fbt/fbt-inventory | only runs for accounts where `fbt_enabled=true` |
| **NEW** FBT order sync (defence in depth) | `0 */5 * * * *` | modules/fbt/fbt-orders | |
| **NEW** FBT inbound status poll (P2) | `0 */30 * * * *` | modules/fbt/fbt-inbound | only polls shipments in non-terminal states |
| **NEW** FBT statement ingestion (P3) | `15 3 * * *` | modules/fbt/fbt-fees | daily UTC |
| **NEW** FBT discrepancy detector (P3) | `0 5 * * *` | core-service | runs after statement ingestion |

All new crons honour an in-process semaphore: previous run still in-flight → new run skips + logs warning.

---

## 5. Logging

`winston` logger in both repos. No new logger.

**Tag every FBT log line:**
- `module=fbt-<submodule>` (e.g., `fbt-inventory`, `fbt-orders`)
- `account_id`
- `region`

Use the existing `activity_logs` table for per-account user-facing events.

Examples:

```
[fbt-inventory] module=fbt-inventory account_id=42 region=GB sync started, expected ~150 SKUs
[fbt-inventory] module=fbt-inventory account_id=42 region=GB sync ok, 147 SKUs, 3.2s
[fbt-orders]    module=fbt-orders    account_id=42 region=GB order TK-12345 published to wms (fulfilment_mode=fbt)
[fbt-accounts]  module=fbt-accounts  account_id=42 region=GB fbt_enabled=true, capabilities verified
```

Sensitive data (tokens, addresses) passes through existing PII redaction utility.

---

## 6. Metrics

Tag every metric with `service=tiktok-microservice` (no change) and `flow=fbt` (NEW) to distinguish from seller flow.

| Metric | Type | Labels |
|---|---|---|
| `fbt_inventory_sync_total` | counter | account, region, status |
| `fbt_inventory_sync_duration_seconds` | histogram | account, region |
| `fbt_inventory_stale_minutes` | gauge | account, region (alert > 30) |
| `fbt_orders_published_total` | counter | account, region |
| `fbt_orders_publish_errors_total` | counter | account, region, error_code |
| `fbt_webhook_received_total` | counter | event_type, region |
| `fbt_webhook_duplicate_total` | counter | event_type |
| `fbt_api_rate_limited_total` | counter | region, endpoint |
| `fbt_rabbitmq_queue_depth` | gauge | queue |
| `fbt_enable_toggle_calls_total` | counter | account, outcome (success, not_enrolled, error) |
| `fbt_publish_skipped_total` | counter | account, module, reason (`no_account`, `fbt_disabled`, `publish_gate_off`) |
| `fbt_fees_ingested_total` (P3) | counter | account, region, fee_type |
| `fbt_reimbursements_pending` (P3) | gauge | account, region |

---

## 7. Alerts

| Alert | Condition | Severity |
|---|---|---|
| FBT inventory stale | `fbt_inventory_stale_minutes` > 30 | P2 |
| FBT order publish errors | error rate > 5% in 15-min window | P1 |
| FBT rate-limit pressure | `fbt_api_rate_limited_total` > 1/min for 5 min | P2 |
| Webhook duplicate rate | duplicate/received > 50% | P2 |
| Queue depth growing | FBT queue grows linearly for 10 min | P2 |
| Statement ingestion failure (P3) | daily cron failed | P2 |
| **Existing seller integration regression** | any error spike in tiktok-microservice seller-flow metrics during FBT deploys | **P1** |

The last alert is unique to this initiative: because we share infrastructure with the existing seller flow, an FBT deploy could in principle affect seller flows. Alert catches it early.

---

## 8. Health checks

Gateway already does HTTP health checks on `/health`. No change needed.

Optional deeper probe: `GET /tiktok/fbt/health/detailed`:

```json
{
  "status": "ok",
  "service": "tiktok-microservice",
  "checks": {
    "db": "ok",
    "rabbitmq": "ok",
    "tiktok_api_reachable": "ok",
    "fbt_accounts_enabled": 12,
    "last_fbt_inventory_sync_age_seconds": 412
  }
}
```

---

## 9. Backup, restore, retention

- **DB backups:** `tiktok_microservice` DB already in nightly pg_dump rotation. New tables included automatically.
- **Snapshot retention:** `fbt_inventory` accumulates one row per SKU per sync. **Delete snapshots older than 90 days where `is_latest = FALSE`** — schedule weekly Sundays 04:00.
- **Activity log retention:** existing 30-day policy.
- **Webhook events retention:** `fbt_webhook_events` 7 days. Long enough to catch retry storms; short enough to keep the table small.

---

## 10. Rollback plan

Each phase has a defined rollback. Because we're modifying the existing service, rollback is simpler than tearing down a new service.

### Phase 1 rollback
1. Flip `tiktok_fbt = false` for all merchants — UI hides, API gates 403
2. Stop new crons: `FBT_ENABLE_INVENTORY_SYNC=false`, `FBT_ENABLE_ORDER_SYNC=false`. Restart service
3. Existing seller flow continues unaffected (it's the same service)
4. In-flight FBT orders already in core-service are valid `Order` rows — they continue normally
5. Optional: `UPDATE accounts SET fbt_enabled=false` to suppress any remaining UI

### Phase 2 rollback
1. Flip `tiktok_fbt_inbound = false`. Disable `FBT_ENABLE_INBOUND_SHIPMENTS=false`
2. Restart
3. In-flight shipments: rows stay, state transitions freeze. Banner "Inbound shipments temporarily disabled — track in TikTok Seller Center directly"

### Phase 3 rollback
1. Flip `tiktok_fbt_fees = false`. Disable `FBT_ENABLE_FEES_INGESTION=false`
2. Already-ingested rows remain. UI hides the page. Re-enabling resumes from where it left off (cron is idempotent)

---

## 11. Pre-deployment checklist (per phase)

### Phase 1
- [ ] All Phase-1 migrations applied; verified with `\d` on both DBs
- [ ] **Regression test passed**: existing TikTok seller flows behave identically (full regression on order sync, inventory push, product listing)
- [ ] Sandbox FBT account verified, enable-FBT toggle works end-to-end
- [ ] Feature flag per-merchant (one on, one off — results differ as expected)
- [ ] **Pick/pack regression**: automated test that an FBT order does NOT appear in `/clients/order/awaiting-pick`
- [ ] Webhook duplicate test — replay same payload, second attempt returns 200 with `duplicate=true` log
- [ ] Token refresh test — token set to expire in 2 min, auto-refresh, no downtime
- [ ] PII redaction verified in logs
- [ ] Rollback drill — run in staging, verify clean state

### Phase 2 additions
- [ ] Inbound-creation smoke (sandbox plan + label generation)
- [ ] Partial-receipt scenario

### Phase 3 additions
- [ ] Statement ingestion smoke (replay one historical statement payload)
- [ ] Reimbursement display
- [ ] Dispute submission (or external-link fallback)

---

## 12. Fail-closed publisher guards

**Lesson borrowed from Amazon FBA.** Amazon's `LESSONS.md` line 6 documents an incident: a null-coalescing bug on `publish_fba_to_wms` leaked **421 FBA orders** into the picker queue before anyone noticed. The fix was structural: every publisher must guard fail-closed, not fail-open. We adopt the same pattern from day one.

### The rule

Every FBT-side publisher (orders, inventory, inbound, fees, returns) MUST:

1. Load the `account` row first.
2. Check **both** flags in a single boolean expression, with explicit `=== true` comparison.
3. Treat **null / undefined / missing / type-coerced** values as "no" — return without publishing.
4. Log the skip at `warn` level with `account_id`, `module`, and which flag failed.

### Canonical guard

```typescript
// src/modules/fbt/common/guards/can-publish-fbt.ts
export function canPublishFbt(account: AccountEntity | null | undefined): boolean {
  // FAIL CLOSED: missing account = no publish
  if (!account) return false;

  // Strict equality. A truthy non-boolean (e.g. "false" string from a bad migration)
  // must NOT pass. Same for null/undefined.
  if (account.fbt_enabled !== true) return false;
  if (account.publish_fbt_to_wms !== true) return false;

  return true;
}
```

### Usage at every publish site

```typescript
// e.g. fbt-orders.publisher.ts
async publishOrderCreated(order: TiktokOrderEntity): Promise<void> {
  const account = await this.accountRepo.findOneBy({ id: order.account_id });

  if (!canPublishFbt(account)) {
    this.logger.warn(
      `[fbt-orders] publish skipped`,
      {
        module: 'fbt-orders',
        account_id: order.account_id,
        order_id: order.id,
        fbt_enabled: account?.fbt_enabled ?? null,
        publish_fbt_to_wms: account?.publish_fbt_to_wms ?? null,
      },
    );
    return;
  }

  await this.rabbitmq.publish('tiktok_fbt.to.wms.order_created', { /* … */ });
}
```

### Required tests

Each publisher gets a test file with at minimum these cases (failing any one of them blocks merge):

```typescript
// src/modules/fbt/common/guards/can-publish-fbt.spec.ts
describe('canPublishFbt — fail-closed contract', () => {
  it.each([
    ['null account',          null,                                                false],
    ['undefined account',     undefined,                                           false],
    ['fbt_enabled=null',      { fbt_enabled: null,  publish_fbt_to_wms: true },    false],
    ['fbt_enabled=undefined', {                     publish_fbt_to_wms: true },    false],
    ['fbt_enabled="true"',    { fbt_enabled: 'true' as any, publish_fbt_to_wms: true }, false],  // string, not boolean
    ['fbt_enabled=1',         { fbt_enabled: 1 as any, publish_fbt_to_wms: true }, false],      // number, not boolean
    ['fbt_enabled=false',     { fbt_enabled: false, publish_fbt_to_wms: true },    false],
    ['publish=false',         { fbt_enabled: true,  publish_fbt_to_wms: false },   false],
    ['publish=null',          { fbt_enabled: true,  publish_fbt_to_wms: null },    false],
    ['both=true',             { fbt_enabled: true,  publish_fbt_to_wms: true },    true ],
  ])('%s → %s', (_label, account, expected) => {
    expect(canPublishFbt(account as any)).toBe(expected);
  });
});
```

The truthy-coercion cases (`'true'` string, `1` number) catch the exact Amazon bug class: a future ORM upgrade or migration that accidentally stores booleans as text won't silently re-open the gate.

### Inverse — consumer side

Core-service's `FbtOrderConsumer` and `FbtAccountConsumer` apply the same pattern on receipt: re-check the flags before mutating state. This is defence in depth — if a stale message arrives after `fbt_enabled` was flipped off, the consumer drops it instead of writing.

### Observability

`fbt_publish_skipped_total` (registered in §6) labels every skip with `reason`. Alert if `reason='publish_gate_off'` rises above zero unexpectedly — that signals ops flipped the gate without an incident ticket, which is itself worth investigating.

---

## 13. Cost & capacity notes

- **Outbound TikTok API calls per merchant per day** (worst case, fully active FBT account):
  - Inventory: 96 polls × ~3 pages = ~288 calls
  - Orders (FBT): 288 polls × ~1-2 pages = ~500 calls
  - Inbound (P2): 48 polls × ~1 call = ~48 calls
  - Statements (P3): 1 daily call
  - **Total ~840 calls/day/account.** Well under typical rate limit.

- **Storage growth:** ~150 SKUs × 96 snapshots/day × 1 KB ≈ 15 MB/day/account. At 100 accounts → 1.5 GB/day → pruned to 90 days → ~135 GB peak. Trivial.

- **RabbitMQ:** small messages (<4 KB), low volume. No impact.

- **Compute:** new modules run inside the same `tiktok-microservice` process. Profile expected to add 10-20% CPU at full FBT volume. No node-count change needed at Phase 1 scale.

- **DB connections:** existing connection pool serves both flows. No change.

No capacity planning required for Phase 1. Re-evaluate if FBT crosses 500 merchants.
