# 02 — Data Model Changes

> Two repos affected: `tiktok-microservice` (TypeORM, NestJS) and `core-service` (Sequelize, Express). Inside `tiktok-microservice`, the existing tables are **additively extended** (new columns) and **new `fbt_*` tables** are created **in the same database**. No new microservice, no new DB.

---

## A. Changes inside `tiktok-microservice` DB (`tiktok_microservice`)

### A1. Extend `accounts` — fulfilment capability flags + ops kill-switch

Two layers of flags live on the row:

- **Capability flags** — merchant-facing. `fbm_enabled` / `fbt_enabled` reflect what the merchant has chosen on the account card. A CHECK constraint guarantees at least one is true.
- **Operational publish gate** — admin-only. `publish_fbt_to_wms` is the kill-switch ops flips when a downstream consumer breaks without touching merchant-facing state. **Pattern borrowed from Amazon FBA's `publish_fba_to_wms`** — see Amazon's `LESSONS.md` line 6: a fail-open guard once leaked 421 FBA orders into the picker queue. This column gives us the same defensive layer.

```sql
ALTER TABLE accounts
  ADD COLUMN fbm_enabled BOOLEAN NOT NULL DEFAULT TRUE,
  ADD COLUMN fbt_enabled BOOLEAN NOT NULL DEFAULT FALSE,
  ADD COLUMN publish_fbt_to_wms BOOLEAN NOT NULL DEFAULT TRUE,
  ADD COLUMN fbt_capabilities JSONB NULL,
  ADD COLUMN fbt_enabled_at TIMESTAMPTZ NULL,
  ADD COLUMN fbt_last_verified_at TIMESTAMPTZ NULL,
  ADD COLUMN fbt_last_inventory_sync_at TIMESTAMPTZ NULL,
  ADD COLUMN fbt_last_order_sync_at TIMESTAMPTZ NULL,
  ADD CONSTRAINT chk_accounts_any_mode
    CHECK (fbm_enabled = TRUE OR fbt_enabled = TRUE);

CREATE INDEX idx_accounts_fbt_enabled
  ON accounts(client_id, fbt_enabled, status)
  WHERE fbt_enabled = TRUE;
CREATE INDEX idx_accounts_fbm_enabled
  ON accounts(client_id, fbm_enabled, status)
  WHERE fbm_enabled = TRUE;
```

**No backfill needed** — existing rows default to `fbm_enabled = true`, `fbt_enabled = false`, `publish_fbt_to_wms = true`. The `chk_accounts_any_mode` constraint guarantees we never end up with an account that can do neither fulfilment mode.

**`fbt_capabilities` JSONB content** — populated by the eligibility probe (see `07-risks-and-open-questions.md` §A0 "probe pattern"). Whatever shop-level metadata TikTok includes in the FBT inventory response envelope lands here verbatim. May be `{}` (empty object) if TikTok exposes no extra metadata — the empty object still proves the shop responded successfully to an FBT call, which is the only fact we need to gate the toggle. Frontend reads this for the "Capabilities" line on the account card; if empty, that line collapses to a single "FBT enabled" pill.

**`publish_fbt_to_wms` semantics:**
- Default `TRUE`. Merchants never see this column.
- Flipped to `FALSE` only by engineers — there is no UI control. The intended access path is a SQL update or a future admin tool.
- Every FBT-side publisher (orders, inventory, inbound, fees) MUST check `account.fbt_enabled === true && account.publish_fbt_to_wms === true` **before any work**, fail-closed (see `06-deployment-and-observability.md` §12).
- The flag is independent of `fbt_enabled`: flipping `publish_fbt_to_wms = false` keeps the merchant-facing toggle ON but mutes RabbitMQ publishing — useful when core-service's FBT consumer is being patched and we don't want a queue backlog or a leak.

**Three possible per-account states** (and the matching client-scoped sidebar view, aggregated across that client's accounts):

| Account state | Meaning | Contributes to sidebar |
|---|---|---|
| `fbm_enabled=true, fbt_enabled=false` | Seller-only TikTok shop (today's default) | General TikTok submodules |
| `fbm_enabled=false, fbt_enabled=true` | FBT-only shop | FBT submodules only |
| `fbm_enabled=true, fbt_enabled=true` | Hybrid shop | Both submodule groups |

Disabling FBM is reserved for shops the merchant has explicitly converted to FBT-only (toggle on the same account card as the FBT toggle, but inverted — see §M1 in `04-pages-and-workflows.md`). The default for a freshly-connected shop is always FBM-on / FBT-off.

---

### A2. Extend `tiktok_orders` — fulfilment mode

```sql
ALTER TABLE tiktok_orders
  ADD COLUMN fulfilment_mode VARCHAR(8) NOT NULL DEFAULT 'fbm'
    CHECK (fulfilment_mode IN ('fbm', 'fbt'));

-- Backfill from existing tiktok_metadata JSONB
UPDATE tiktok_orders
   SET fulfilment_mode = 'fbt'
 WHERE tiktok_metadata->>'fulfillment_type' IN ('FULFILLED_BY_TIKTOK', 'FBT');

CREATE INDEX idx_tiktok_orders_mode
  ON tiktok_orders(client_id, fulfilment_mode, status, order_date);
```

---

### A3. New table — `fbt_inventory` (snapshot pattern)

```sql
CREATE TABLE fbt_inventory (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,
  region                   VARCHAR(8) NOT NULL,

  seller_sku               VARCHAR(128) NOT NULL,
  tiktok_sku_id            VARCHAR(64),
  tiktok_product_id        VARCHAR(64),
  product_name             TEXT,
  variation_name           TEXT,

  tiktok_warehouse_id      VARCHAR(64),
  tiktok_warehouse_name    TEXT,

  fulfillable_quantity     INT NOT NULL DEFAULT 0,
  inbound_quantity         INT NOT NULL DEFAULT 0,
  reserved_quantity        INT NOT NULL DEFAULT 0,
  unfulfillable_quantity   INT NOT NULL DEFAULT 0,
  in_transit_quantity      INT NOT NULL DEFAULT 0,
  total_quantity           INT NOT NULL DEFAULT 0,

  inbound_breakdown        JSONB,
  reserved_breakdown       JSONB,
  unfulfillable_breakdown  JSONB,

  units_sold_7d            INT,
  units_sold_30d           INT,
  velocity_per_day         DECIMAL(10,2),
  days_of_cover            DECIMAL(10,1),
  health_status            VARCHAR(8),

  is_latest                BOOLEAN NOT NULL DEFAULT TRUE,
  snapshot_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  sync_status              VARCHAR(16) NOT NULL DEFAULT 'success',
  sync_error               TEXT,

  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_fbt_inv_latest
  ON fbt_inventory(client_id, account_id, region, seller_sku)
  WHERE is_latest = TRUE;
CREATE INDEX idx_fbt_inv_history
  ON fbt_inventory(account_id, seller_sku, snapshot_at DESC);
CREATE INDEX idx_fbt_inv_health
  ON fbt_inventory(client_id, health_status, is_latest)
  WHERE is_latest = TRUE;
```

---

### A4. New table — `fbt_webhook_events` (idempotency log)

```sql
CREATE TABLE fbt_webhook_events (
  id              BIGSERIAL PRIMARY KEY,
  event_key       VARCHAR(255) NOT NULL UNIQUE,
  event_type      VARCHAR(64) NOT NULL,
  shop_id         VARCHAR(64),
  account_id      BIGINT REFERENCES accounts(id) ON DELETE CASCADE,
  received_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  processed_at    TIMESTAMPTZ,
  status          VARCHAR(16) NOT NULL DEFAULT 'received',
  payload         JSONB
);
CREATE INDEX idx_fbt_we_type ON fbt_webhook_events(event_type, received_at DESC);
CREATE INDEX idx_fbt_we_account ON fbt_webhook_events(account_id, received_at DESC);
```

7-day retention.

---

### A5. New table — `fbt_inbound_shipments` (Phase 2)

```sql
CREATE TABLE fbt_inbound_shipments (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,
  region                   VARCHAR(8) NOT NULL,

  wms_inbound_shipment_id  BIGINT,
  tiktok_plan_id           VARCHAR(64),
  tiktok_shipment_id       VARCHAR(64),

  status                   VARCHAR(32) NOT NULL DEFAULT 'draft',
  source_warehouse_id      BIGINT,
  destination_fc_code      VARCHAR(32),
  destination_fc_name      TEXT,

  carrier_name             VARCHAR(64),
  tracking_number          VARCHAR(128),
  label_url                TEXT,

  total_units_planned      INT NOT NULL DEFAULT 0,
  total_units_shipped      INT NOT NULL DEFAULT 0,
  total_units_received     INT NOT NULL DEFAULT 0,

  submitted_at             TIMESTAMPTZ,
  shipped_at               TIMESTAMPTZ,
  expected_arrival_at      TIMESTAMPTZ,
  received_at              TIMESTAMPTZ,
  cancelled_at             TIMESTAMPTZ,

  notes                    TEXT,
  external_metadata        JSONB,
  created_by               BIGINT,
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT fbt_inb_account_plan_key
    UNIQUE (account_id, tiktok_plan_id) DEFERRABLE INITIALLY DEFERRED
);

CREATE INDEX idx_fbt_inb_status ON fbt_inbound_shipments(client_id, status, created_at DESC);
CREATE INDEX idx_fbt_inb_account ON fbt_inbound_shipments(account_id, status);
```

---

### A6. New table — `fbt_inbound_shipment_items` (Phase 2)

```sql
CREATE TABLE fbt_inbound_shipment_items (
  id              BIGSERIAL PRIMARY KEY,
  shipment_id     BIGINT NOT NULL REFERENCES fbt_inbound_shipments(id) ON DELETE CASCADE,
  seller_sku      VARCHAR(128) NOT NULL,
  tiktok_sku_id   VARCHAR(64),
  product_name    TEXT,
  units_planned   INT NOT NULL,
  units_shipped   INT NOT NULL DEFAULT 0,
  units_received  INT NOT NULL DEFAULT 0,
  units_damaged   INT NOT NULL DEFAULT 0,
  units_short     INT NOT NULL DEFAULT 0,
  unit_cost       DECIMAL(10,2),
  external_metadata JSONB,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_fbt_inb_items_shipment ON fbt_inbound_shipment_items(shipment_id);
CREATE INDEX idx_fbt_inb_items_sku ON fbt_inbound_shipment_items(seller_sku);
```

---

### A7. New table — `fbt_fees` (Phase 3)

```sql
CREATE TABLE fbt_fees (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,
  region                   VARCHAR(8) NOT NULL,

  tiktok_order_id          VARCHAR(64),
  tiktok_statement_id      VARCHAR(64),
  statement_date           DATE,

  fee_type                 VARCHAR(32) NOT NULL,
  seller_sku               VARCHAR(128),
  units                    INT,

  amount                   DECIMAL(12,2) NOT NULL,
  currency                 VARCHAR(8) NOT NULL,
  volume_cubic_cm_days     DECIMAL(14,2),

  description              TEXT,
  external_metadata        JSONB,
  wms_charge_id            BIGINT,
  reconciled_at            TIMESTAMPTZ,

  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_fbt_fees_acc_date ON fbt_fees(account_id, statement_date DESC);
CREATE INDEX idx_fbt_fees_type ON fbt_fees(client_id, fee_type, statement_date);
CREATE INDEX idx_fbt_fees_unreconciled ON fbt_fees(account_id, reconciled_at)
  WHERE reconciled_at IS NULL;
```

---

### A8. New table — `fbt_returns` (Phase 3)

```sql
CREATE TABLE fbt_returns (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,
  tiktok_return_id         VARCHAR(64) NOT NULL,
  tiktok_order_id          VARCHAR(64),
  seller_sku               VARCHAR(128),
  units                    INT,
  reason                   TEXT,
  condition                VARCHAR(32),
  status                   VARCHAR(32),
  reimbursement_eligible   BOOLEAN,
  external_metadata        JSONB,
  received_at              TIMESTAMPTZ,
  reimbursed_at            TIMESTAMPTZ,
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT fbt_returns_account_return_key UNIQUE (account_id, tiktok_return_id)
);

CREATE INDEX idx_fbt_ret_order ON fbt_returns(tiktok_order_id);
CREATE INDEX idx_fbt_ret_status ON fbt_returns(client_id, status, received_at DESC);
```

---

## B. Changes inside `core-service` DB (WMS DB)

### B1. New row in `channels`

```sql
INSERT INTO channels (channel, channel_slug, channel_type, is_active, logo, created_at, updated_at)
VALUES ('TikTok FBT', 'tiktok_fbt', 1, 1, '/assets/marketplaces/tiktok-fbt.png', NOW(), NOW())
ON CONFLICT (channel_slug) DO NOTHING;
```

---

### B2. Extend `warehouses`

```sql
ALTER TABLE warehouses
  ADD COLUMN warehouse_type            VARCHAR(16) NOT NULL DEFAULT 'own'
    CHECK (warehouse_type IN ('own', '3pl', 'virtual', 'fbt_tiktok', 'fba_amazon', 'external')),
  ADD COLUMN external_marketplace      VARCHAR(32),
  ADD COLUMN external_warehouse_id     VARCHAR(64),
  ADD COLUMN external_account_id       BIGINT,
  ADD COLUMN external_region           VARCHAR(8),
  ADD COLUMN external_warehouse_config JSONB;

CREATE INDEX idx_warehouses_type ON warehouses(client_id, warehouse_type, status);
CREATE INDEX idx_warehouses_external
  ON warehouses(external_marketplace, external_warehouse_id)
  WHERE external_marketplace IS NOT NULL;
```

When `tiktok_fbt.to.wms.account_fbt_enabled` fires, core-service inserts one virtual warehouse row per region.

---

### B3. `order_fulfillments.fulfillment_type` — new value by convention

Already `STRING(20)`. No DDL needed; document via comment.

---

### B4. New table — `variation_listings`

The hybrid-SKU bridge.

```sql
CREATE TABLE variation_listings (
  id               BIGSERIAL PRIMARY KEY,
  client_id        INTEGER NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
  variation_id     INTEGER NOT NULL REFERENCES variations(id) ON DELETE CASCADE,
  channel_id       INTEGER NOT NULL REFERENCES channels(id),
  account_id       BIGINT,                 -- soft FK to accounts.id in tiktok-microservice DB

  external_listing_id VARCHAR(128),
  external_sku_id     VARCHAR(128),
  listing_url         TEXT,

  fulfilment_mode  VARCHAR(8) NOT NULL DEFAULT 'fbm'
    CHECK (fulfilment_mode IN ('fbm', 'fbt')),
  warehouse_id     INTEGER REFERENCES warehouses(id),

  is_active        BOOLEAN NOT NULL DEFAULT TRUE,
  last_synced_at   TIMESTAMPTZ,
  external_metadata JSONB,

  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at       TIMESTAMPTZ,

  CONSTRAINT uniq_active_listing UNIQUE NULLS NOT DISTINCT
    (client_id, account_id, variation_id, fulfilment_mode, deleted_at)
);

CREATE INDEX idx_var_listing_lookup
  ON variation_listings(client_id, channel_id, account_id, external_sku_id)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_var_listing_variation
  ON variation_listings(variation_id, fulfilment_mode)
  WHERE deleted_at IS NULL AND is_active = TRUE;
```

---

### B5. `inbound_shipments` + items (Phase 2)

```sql
CREATE TABLE inbound_shipments (
  id                       BIGSERIAL PRIMARY KEY,
  client_id                INTEGER NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
  shipment_number          VARCHAR(32) NOT NULL,
  marketplace              VARCHAR(32) NOT NULL,
  marketplace_account_id   BIGINT,
  fulfilment_mode          VARCHAR(8) NOT NULL DEFAULT 'fbt',
  external_plan_id         VARCHAR(64),
  external_shipment_id     VARCHAR(64),
  source_warehouse_id      INTEGER REFERENCES warehouses(id),
  destination_warehouse_id INTEGER REFERENCES warehouses(id),
  destination_fc_code      VARCHAR(32),
  destination_fc_name      TEXT,
  destination_address      JSONB,
  status                   VARCHAR(32) NOT NULL DEFAULT 'draft',
  carrier_name             VARCHAR(64),
  tracking_number          VARCHAR(128),
  label_url                TEXT,
  total_units_planned      INTEGER NOT NULL DEFAULT 0,
  total_units_shipped      INTEGER NOT NULL DEFAULT 0,
  total_units_received     INTEGER NOT NULL DEFAULT 0,
  submitted_at             TIMESTAMPTZ,
  shipped_at               TIMESTAMPTZ,
  expected_arrival_at      TIMESTAMPTZ,
  received_at              TIMESTAMPTZ,
  cancelled_at             TIMESTAMPTZ,
  notes                    TEXT,
  metadata                 JSONB,
  created_by               INTEGER REFERENCES users(id),
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at               TIMESTAMPTZ,
  UNIQUE (client_id, shipment_number)
);

CREATE TABLE inbound_shipment_items (
  id              BIGSERIAL PRIMARY KEY,
  shipment_id     BIGINT NOT NULL REFERENCES inbound_shipments(id) ON DELETE CASCADE,
  variation_id    INTEGER NOT NULL REFERENCES variations(id),
  units_planned   INTEGER NOT NULL,
  units_shipped   INTEGER NOT NULL DEFAULT 0,
  units_received  INTEGER NOT NULL DEFAULT 0,
  units_damaged   INTEGER NOT NULL DEFAULT 0,
  units_short     INTEGER NOT NULL DEFAULT 0,
  metadata        JSONB,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### B6. `marketplace_charges` + `marketplace_reimbursements` (Phase 3)

Identical to prior plan iteration — generalised tables marketed via `marketplace` column ('tiktok_fbt' for now, 'amazon_fba' later).

### B7. Extend `return_requests` — marketplace metadata (Phase 3)

```sql
ALTER TABLE return_requests
  ADD COLUMN marketplace             VARCHAR(32),
  ADD COLUMN marketplace_return_id   VARCHAR(64),
  ADD COLUMN marketplace_return_reason TEXT,
  ADD COLUMN return_destination      VARCHAR(32) DEFAULT 'merchant'
    CHECK (return_destination IN ('merchant', 'marketplace_fbt_warehouse', 'marketplace_fba_warehouse'));
```

---

## C. Feature-flag plumbing

```sql
INSERT INTO subscription_features (key, label, description, is_premium)
VALUES
  ('tiktok_fbt', 'TikTok FBT Integration', 'Fulfilled by TikTok mirror, orders, dashboard.', true),
  ('tiktok_fbt_inbound', 'TikTok FBT Inbound Shipments', 'Create and manage inbound shipments to TikTok FCs.', true),
  ('tiktok_fbt_fees', 'TikTok FBT Fee & Reimbursement Reconciliation', 'Statement ingestion and dispute submission.', true);
```

`checkPackageFeature('tiktok_fbt')` middleware gates every `/api/tiktok/fbt/*` route.

---

## D. Migration ordering

### Phase 1 — inside `tiktok-microservice/src/migrations/`

1. `<ts>-AddFulfilmentFlagsToAccounts.ts` (A1) — adds `fbm_enabled`, `fbt_enabled`, `publish_fbt_to_wms`, FBT capability/timestamp columns, plus `chk_accounts_any_mode`
2. `<ts>-AddFulfilmentModeToTiktokOrders.ts` (A2)
3. `<ts>-CreateFbtInventory.ts` (A3)
4. `<ts>-CreateFbtWebhookEvents.ts` (A4)

### Phase 1 — inside `core-service/migrations/`

1. `<ts>-SeedTiktokFbtChannel.js` (B1)
2. `<ts>-AddWarehouseTypeColumns.js` (B2)
3. `<ts>-CommentFulfillmentTypeOnOrderFulfillments.js` (B3)
4. `<ts>-CreateVariationListings.js` (B4)
5. `<ts>-SeedSubscriptionFeaturesForFbt.js` (C)

### Phase 2

- tiktok-microservice: A5, A6
- core-service: B5

### Phase 3

- tiktok-microservice: A7, A8
- core-service: B6, B7

---

## E. Rollback policy

All `down()` migrations DROP new columns/tables. Existing rows unaffected — all new columns have DEFAULT values. Production rollback: flip `tiktok_fbt=false` flag, disable FBT crons via env, restart service. Existing seller flow runs unchanged whether `fbt_enabled` column exists or not.
