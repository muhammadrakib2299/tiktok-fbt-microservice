# 06 — Deployment & Observability

> A new microservice means a new deploy target, new env, new monitoring tags. Most of this section is "do what we already do for tiktok-microservice, but for the new one." A few items are specific to the separation.

---

## 1. New service deployment summary

| Resource | Value |
|---|---|
| Repo | `D:\office porjects\tiktok-fbt-microservice` |
| Port | **5018** (verify against current production port allocations before deploy — `reference_port_allocation.md` memory shows 5018 unallocated as of 2026-04-07; amazon-microservice port not in memory, may use 5014) |
| Database | `tiktok_fbt_microservice` (Postgres) |
| Gateway route | `/api/tiktok-fbt/*` → `http://localhost:5018/api/*` |
| OAuth redirect URI | `${BASE_URL}/api/tiktok-fbt/auth/callback` |
| RabbitMQ exchange | `wms_multichannel` (existing — no change) |
| RabbitMQ routing key prefix (in) | `tiktok_fbt.to.wms.*` |
| RabbitMQ routing key prefix (out) | `wms.to.tiktok_fbt.*` |
| Webhook endpoint | `${BASE_URL}/api/tiktok-fbt/webhooks` |

---

## 2. Environment variables

### `tiktok-fbt-microservice/.env`

```bash
NODE_ENV=development
PORT=5018

# Database
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=admin
DB_NAME=tiktok_fbt_microservice

# WMS DB (read-only access for catalogue + variations lookup)
WMS_DB_HOST=localhost
WMS_DB_PORT=5432
WMS_DB_USERNAME=postgres
WMS_DB_PASSWORD=admin
WMS_DB_NAME=wms_v2          # IMPORTANT: per existing memory feedback_wms_db_name_trap.md,
                            # local WMS DB is wms_v2, NOT combosoft_wms_db. Always set this.

# TikTok app credentials
# Open question (week-1 research): do we register a separate TikTok app for FBT,
# or reuse the existing TIKTOK_APP_KEY/TIKTOK_APP_SECRET from tiktok-microservice?
TIKTOK_FBT_APP_KEY=...
TIKTOK_FBT_APP_SECRET=...
TIKTOK_FBT_REDIRECT_URI=http://localhost:5018/api/tiktok-fbt/auth/callback
TIKTOK_FBT_API_BASE_URL=https://open-api.tiktokglobalshop.com

# Token encryption (AES-256-GCM, 32-byte hex)
# IMPORTANT: must be a different key from tiktok-microservice's TOKEN_ENCRYPTION_KEY.
# Cross-service token decryption is not supported and never needed.
TOKEN_ENCRYPTION_KEY=<generate-fresh-32-byte-hex>

# RabbitMQ
RABBITMQ_ENABLED=true
RABBITMQ_URL=amqp://guest:guest@localhost:5672
RABBITMQ_EXCHANGE=wms_multichannel

# CORS
FRONTEND_URL=http://localhost:3000

# Region scope this instance supports
FBT_ENABLED_REGIONS=GB

# Phase-1 feature gates (server-side defence in depth, in addition to per-merchant flag)
FBT_ENABLE_INVENTORY_SYNC=true
FBT_ENABLE_ORDER_SYNC=true
FBT_ENABLE_WEBHOOKS=true

# Phase 2 — off until Phase 2 ships
FBT_ENABLE_INBOUND_SHIPMENTS=false

# Phase 3 — off until Phase 3 ships
FBT_ENABLE_FEES_INGESTION=false
FBT_ENABLE_DISPUTE_SUBMISSION=false

# Cron schedules
FBT_INVENTORY_CRON='0 */15 * * * *'
FBT_ORDERS_CRON='0 */5 * * * *'
FBT_INBOUND_STATUS_CRON='0 */30 * * * *'
FBT_STATEMENT_CRON='15 3 * * *'
FBT_TOKEN_REFRESH_CRON='0 5,35 * * * *'

# Per-account stagger to spread rate-limit load
FBT_ACCOUNT_STAGGER_SECONDS=30

# Hard ceilings on outbound TikTok requests per minute, per region
FBT_RATE_LIMIT_PER_MINUTE_GB=50
FBT_RATE_LIMIT_PER_MINUTE_US=50
```

### `core-service/.env` — additions

```bash
# Feature flag default for new merchants (per-merchant value lives in DB)
TIKTOK_FBT_DEFAULT_ENABLED=false

# Reservation buffer (seconds): how long an unconfirmed FBT order's inventory stays reserved
TIKTOK_FBT_ORDER_RESERVATION_TTL=900
```

### `frontend_wms_v/.env.local`

No new env vars. Server-driven sidebar via `useModulePermissionStore`.

### `gateway/` — register service

In `D:\office porjects\backend_wms_v2\gateway\load-balancer.js`:

```javascript
const serviceConfig = {
  // ... existing entries ...
  tiktok:     'http://localhost:5011/api',
  // NEW:
  'tiktok-fbt': 'http://localhost:5018/api',
};
```

And in `server.js` (or wherever route factories live):

```javascript
createLBProxy('/api/tiktok-fbt', 'tiktok-fbt', '^/api/tiktok-fbt');
```

---

## 3. Feature-flag rollout

Three feature keys:
1. `tiktok_fbt` — master switch for all FBT pages and APIs.
2. `tiktok_fbt_inbound` — gates Phase 2.
3. `tiktok_fbt_fees` — gates Phase 3.

Rollout policy:

| Audience | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|
| Internal pilot | EoW4 ship day | EoW7 | EoW10 |
| External pilot (1 merchant) | week 12 | same | same |
| Limited GA (5-10 merchants) | week 12 | week 12 | week 12 |
| Open to all UK merchants on appropriate tier | post-launch | post-launch | post-launch |

`checkPackageFeature` middleware enforces. Flag is per-merchant — flipping for one merchant doesn't affect others.

---

## 4. RabbitMQ topology

Zero infrastructure change. The new service connects to the existing `wms_multichannel` topic exchange and creates new queues bound to new routing keys.

### New routing keys

| Direction | Routing key | Producer | Consumer |
|---|---|---|---|
| → WMS | `tiktok_fbt.to.wms.account_connected` | tiktok-fbt-microservice (on OAuth success) | core-service (inserts virtual warehouse) |
| → WMS | `tiktok_fbt.to.wms.account_disconnected` | tiktok-fbt-microservice | core-service (marks warehouse stale) |
| → WMS | `tiktok_fbt.to.wms.inventory_synced` | tiktok-fbt-microservice | core-service (audit only — P1) |
| → WMS | `tiktok_fbt.to.wms.order_created` | tiktok-fbt-microservice | core-service (FbtOrderConsumer) |
| → WMS | `tiktok_fbt.to.wms.order_status_changed` | tiktok-fbt-microservice | core-service |
| → WMS | `tiktok_fbt.to.wms.inbound_status_changed` (P2) | tiktok-fbt-microservice | core-service |
| → WMS | `tiktok_fbt.to.wms.return_received` (P2/P3) | tiktok-fbt-microservice | core-service |
| → WMS | `tiktok_fbt.to.wms.fee_incurred` (P3) | tiktok-fbt-microservice | core-service |
| → WMS | `tiktok_fbt.to.wms.reimbursement_received` (P3) | tiktok-fbt-microservice | core-service |
| → service | `wms.to.tiktok_fbt.inbound_create` (P2) | core-service | tiktok-fbt-microservice |
| → service | `wms.to.tiktok_fbt.inbound_confirm` (P2) | core-service | tiktok-fbt-microservice |
| → service | `wms.to.tiktok_fbt.dispute_submit` (P3) | core-service | tiktok-fbt-microservice |

### Queue topology per consumer

Each new consumer gets:
- Dedicated durable queue bound to the routing key.
- Dead-letter binding to existing `wms_multichannel_dlx`.
- Retry policy via existing `wms_retry_exchange` (1s, 5s, 30s).
- Prefetch count 5.

### Idempotency keys

Every FBT-side message carries:
- `wms_message_id` (unique per event) — used by core-service as primary idempotency check.
- `event_type` and `event_id` from TikTok — used by `fbt_webhook_events` for inbound dedup.

---

## 5. Cron jobs

All registered via NestJS `@Cron` decorator inside `tiktok-fbt-microservice`.

| Cron | Schedule | Notes |
|---|---|---|
| Token refresh | `0 5,35 * * * *` | Refresh expiring access tokens (cloned from existing service) |
| FBT inventory sync | `0 */15 * * * *` | Runs only if `FBT_ENABLE_INVENTORY_SYNC=true` |
| FBT order sync (defence in depth) | `0 */5 * * * *` | Reconciles webhook-missed orders |
| FBT inbound status poll (P2) | `0 */30 * * * *` | Only polls shipments in non-terminal states |
| FBT statement ingestion (P3) | `15 3 * * *` | Daily UTC; offset to avoid backups |
| FBT discrepancy detector (P3) | `0 5 * * *` | In core-service; runs after statement ingestion |

All new crons honour an in-process semaphore: if a previous run is still in flight when next fires, the new run skips and logs a warning.

---

## 6. Logging

Conventions:

- `winston` logger in both repos. No new logger.
- **Tag every log line with the service name** (winston has a `service` field). Service name = `tiktok-fbt-microservice`. This is the easiest way to distinguish from the seller service.
- Also tag `module=<module-name>`, `account_id`, `region`.
- Use the new `fbt_activity_logs` table for per-account user-facing events ("FBT inventory synced — 142 SKUs"). Visible in UI on the account card.
- Sensitive data (tokens, addresses) MUST go through the PII redaction utility before logging.

Example log lines:

```
[fbt-inventory] service=tiktok-fbt-microservice module=fbt-inventory account_id=42 region=GB sync started, expected ~150 SKUs
[fbt-inventory] service=tiktok-fbt-microservice module=fbt-inventory account_id=42 region=GB sync ok, 147 SKUs, 3.2s, next at 09:30:00
[fbt-orders] service=tiktok-fbt-microservice module=fbt-orders account_id=42 region=GB order TK-12345 published to wms
[fbt-webhooks] service=tiktok-fbt-microservice module=fbt-webhooks event=ORDER_STATUS_CHANGE event_key=ORDER_STATUS_CHANGE:7290xxx:abc123 status=processed
```

---

## 7. Metrics

Suggested counters/gauges/histograms. Each metric should include label `service=tiktok-fbt-microservice` to differentiate from the seller service.

| Metric | Type | Labels | Purpose |
|---|---|---|---|
| `fbt_inventory_sync_total` | counter | account, region, status | Throughput, error rate |
| `fbt_inventory_sync_duration_seconds` | histogram | account, region | Latency p50/p95/p99 |
| `fbt_inventory_skus_synced` | gauge | account, region | Last sync size |
| `fbt_inventory_stale_minutes` | gauge | account, region | Alert > 30 |
| `fbt_orders_published_total` | counter | account, region | Volume |
| `fbt_orders_publish_errors_total` | counter | account, region, error_code | Health |
| `fbt_webhook_received_total` | counter | event_type, region | Webhook activity |
| `fbt_webhook_duplicate_total` | counter | event_type | Dedup rate |
| `fbt_api_rate_limited_total` | counter | region, endpoint | TikTok limit pressure |
| `fbt_rabbitmq_queue_depth` | gauge | queue | Backpressure |
| `fbt_inbound_shipments_in_state` | gauge | state | Operational visibility (P2) |
| `fbt_fees_ingested_total` | counter | account, region, fee_type | P3 |
| `fbt_reimbursements_pending` | gauge | account, region | P3 |

---

## 8. Alerts

| Alert | Condition | Severity |
|---|---|---|
| FBT inventory stale | `fbt_inventory_stale_minutes` > 30 | P2 |
| FBT order publish errors | error rate > 5% in 15-min window | P1 |
| FBT rate-limit pressure | `fbt_api_rate_limited_total` rate > 1/min for 5 min | P2 |
| Webhook duplicate rate | duplicate_total / received > 50% | P2 |
| Queue depth growing | any FBT queue depth grows linearly for 10 min | P2 |
| Statement ingestion failure (P3) | daily cron failed | P2 |
| **Existing seller integration regression** | any error spike in tiktok-microservice metrics correlating with FBT deploys | **P1** — protects the existing service |

The last alert is unique to this initiative: because we're sharing infrastructure (gateway, RabbitMQ, core-service), an FBT deploy could in principle affect seller flows. The alert catches it early.

---

## 9. Health checks

Gateway already does HTTP health checks on each registered service via `/health` (15s interval, 5s timeout, circuit breaker). Add `tiktok-fbt-microservice` to the existing health-check rotation in `gateway/server.js`.

Optional deeper probe: `GET /api/tiktok-fbt/health/detailed`:

```json
{
  "status": "ok",
  "service": "tiktok-fbt-microservice",
  "version": "0.1.0",
  "checks": {
    "db": "ok",
    "rabbitmq": "ok",
    "tiktok_api_reachable": "ok",
    "fbt_accounts_synced_recently": "ok",
    "last_inventory_sync_age_seconds": 412
  }
}
```

---

## 10. Backup, restore, retention

- **DB backups:** add `tiktok_fbt_microservice` to existing nightly pg_dump rotation.
- **Snapshot retention:** `fbt_inventory` accumulates one row per SKU per sync. Retention job: **delete snapshots older than 90 days where `is_latest = FALSE`**. Schedule weekly Sundays 04:00.
- **Activity log retention:** `fbt_activity_logs` keep 30 days minimum for support-facing diagnostics. Match existing TikTok service policy.
- **Webhook event retention:** `fbt_webhook_events` keep 7 days for idempotency, then archive/delete. Sufficient to dedup TikTok retry storms.

---

## 11. Rollback plan

Each phase has a defined rollback. Because the new service is fully isolated, rollback is cleaner than the embedded-module approach would have been.

### Phase 0 rollback (worst case)
- Tear down the new service entirely. Remove gateway registration. Drop `tiktok_fbt_microservice` DB. **No effect on existing tiktok-microservice or seller flows.**

### Phase 1 rollback
1. Flip `tiktok_fbt=false` for all merchants — UI hides, API gates 403.
2. Stop new crons: `FBT_ENABLE_INVENTORY_SYNC=false`, `FBT_ENABLE_ORDER_SYNC=false`. Restart service.
3. Existing in-flight FBT orders already in core-service are valid `Order` rows — they continue to behave normally (just with `createdVia='tiktok_fbt'` and no picker assignment).
4. Optional: `DELETE FROM fbt_inventory` if we want a clean retry. Not destructive of business data.

### Phase 2 rollback
1. Flip `tiktok_fbt_inbound=false`. Disable `FBT_ENABLE_INBOUND_SHIPMENTS=false`. Restart.
2. In-flight shipments: keep rows, freeze state transitions, surface banner "Inbound shipments temporarily disabled — track in TikTok Seller Center directly."

### Phase 3 rollback
1. Flip `tiktok_fbt_fees=false`. Disable `FBT_ENABLE_FEES_INGESTION=false`.
2. Already-ingested rows remain. UI hides the page. Re-enabling resumes from where it left off (cron is idempotent).

---

## 12. Pre-deployment checklist (per phase)

### Phase 0
- [ ] New service builds, runs, deploys to staging.
- [ ] Gateway routing works (`GET /api/tiktok-fbt/health` returns 200).
- [ ] DB migrations table created.
- [ ] Existing `tiktok-microservice` regression-tested: zero behavioural change.
- [ ] Port 5018 (or alternate) verified free.

### Phase 1
- [ ] All Phase-1 migrations applied; verified with `\d` on both DBs.
- [ ] Sandbox FBT account connected; smoke test passed.
- [ ] Feature flag verified per-merchant (one on, one off — results differ correctly).
- [ ] **Pick/pack regression test:** existing seller order picks correctly; new FBT order does NOT appear in any picker queue. Automated test exists in core-service.
- [ ] Webhook duplicate test (replay same payload) — second attempt returns 200 with `duplicate=true` log.
- [ ] Rate-limit test (60 req/min) — observed back-off and queueing.
- [ ] Token refresh test (token set to expire in 2 min) — auto-refresh, no downtime.
- [ ] Logs reviewed for PII leaks.
- [ ] Sentry/Winston alert routing verified for new service tag.
- [ ] Rollback drill — run rollback playbook in staging, verify clean state.

### Phase 2
- [ ] Add: inbound-creation smoke (sandbox plan + label generation), partial-receipt scenario.

### Phase 3
- [ ] Add: statement ingestion smoke (replay one historical statement payload), reimbursement display, dispute submission (or external link fallback).

---

## 13. Cost & capacity notes

- **Outbound TikTok API calls per merchant per day** (worst case, fully active FBT account):
  - Inventory: 96 polls × ~3 pages = ~288 calls
  - Orders: 288 polls × ~1-2 pages = ~500 calls
  - Inbound (P2): 48 polls × ~1 call = ~48 calls
  - Statements (P3): 1 daily call
  - **Total ~840 calls/day/account.** Well under TikTok's typical rate limit.

- **Storage growth (new DB):** ~150 SKUs × 96 snapshots/day × 1 KB ≈ 15 MB/day/account. At 100 accounts, ~1.5 GB/day, pruned to 90 days → ~135 GB peak. Trivial.

- **RabbitMQ:** small messages (<4 KB), low volume. No infra impact.

- **Compute:** new microservice runs the same shape of workload as existing TikTok microservice. Resource budget: ~1 CPU core, ~512MB RAM per instance is sufficient at Phase-1 volumes.

- **DB connections:** new service holds its own connection pool to `tiktok_fbt_microservice` DB and a read-only pool to WMS DB. Existing connection budget should accommodate.

No capacity planning required for Phase 1. Re-evaluate if FBT crosses 500 merchants.
