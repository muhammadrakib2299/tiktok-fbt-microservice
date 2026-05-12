# 02 — Data Model Changes

> **Zero changes to `tiktok-microservice` repo or `tiktok_microservice` DB.** All new tables live in the new `tiktok_fbt_microservice` DB. Modest additive changes to core WMS DB (`combosoft_wms_db` / `wms_v2`).
>
> Two repos affected: the **new** `tiktok-fbt-microservice` (TypeORM, NestJS) and `core-service` (Sequelize, Express).

---

## Naming conventions

- `tiktok-fbt-microservice`: TypeORM, `snake_case` SQL columns, `PascalCase` entities. Table names DO NOT carry the `tiktok_fbt_` prefix because they already live in a DB called `tiktok_fbt_microservice` — `fbt_accounts` is sufficient and unambiguous. Migration filename pattern `<unix_ts>-<Name>.ts`.
- `core-service`: Sequelize, `camelCase` model layer / `snake_case` SQL layer (via `underscored: true`), migration filename pattern `<YYYYMMDDHHMMSS>-<descriptive-name>.js`.

---

## A. New tables in `tiktok_fbt_microservice` DB

This DB starts empty. Every table below is created in Phase 1 (unless marked).

### A1. `fbt_accounts` — FBT shop connections

```sql
CREATE TABLE fbt_accounts (
  id                       BIGSERIAL PRIMARY KEY,
  account_name             VARCHAR(255) NOT NULL,
  client_id                BIGINT NOT NULL,                    -- WMS tenant ID

  -- TikTok shop identity
  shop_id                  VARCHAR(64) NOT NULL,
  shop_cipher              TEXT,                                -- AES-256-GCM encrypted
  shop_name                VARCHAR(255),
  shop_region              VARCHAR(8) NOT NULL                  -- 'GB' | 'US' (more later)
                           CHECK (shop_region IN ('GB','US')),

  -- OAuth tokens (encrypted)
  access_token             TEXT NOT NULL,                       -- AES-256-GCM encrypted
  refresh_token            TEXT NOT NULL,
  expiry_date              TIMESTAMPTZ NOT NULL,
  refresh_token_expiry     TIMESTAMPTZ,
  open_id                  VARCHAR(64),
  seller_name              VARCHAR(255),

  -- App credentials (per-account override; falls back to env)
  app_key                  TEXT,                                -- encrypted
  app_secret               TEXT,                                -- encrypted

  logo                     TEXT,

  -- FBT-specific
  fbt_capabilities         JSONB,                               -- what this shop is enrolled in (verified at connect)
  publish_to_wms           BOOLEAN NOT NULL DEFAULT TRUE,
  sync_orders              BOOLEAN NOT NULL DEFAULT TRUE,
  sync_inventory           BOOLEAN NOT NULL DEFAULT TRUE,

  -- Soft link to seller account (optional, informational only — different DB)
  linked_seller_account_id BIGINT,                              -- references tiktok_microservice.accounts.id

  -- Connection state
  status                   SMALLINT NOT NULL DEFAULT 1,         -- 0 = disconnected, 1 = connected
  last_verified_at         TIMESTAMPTZ,
  last_verified_status     VARCHAR(32),                         -- 'success' | 'failed' | 'token_expired'
  last_order_sync_at       TIMESTAMPTZ,
  last_inventory_sync_at   TIMESTAMPTZ,

  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at               TIMESTAMPTZ,

  CONSTRAINT fbt_accounts_client_shop_key UNIQUE (client_id, shop_id)
);

CREATE INDEX idx_fbt_acc_client ON fbt_accounts(client_id, status);
CREATE INDEX idx_fbt_acc_region ON fbt_accounts(shop_region, status);
CREATE INDEX idx_fbt_acc_linked ON fbt_accounts(linked_seller_account_id)
  WHERE linked_seller_account_id IS NOT NULL;
```

---

### A2. `fbt_products` — FBT-eligible listings

```sql
CREATE TABLE fbt_products (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES fbt_accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,

  -- TikTok identifiers
  tiktok_product_id        VARCHAR(64) NOT NULL,
  title                    TEXT,
  description              TEXT,
  category_id              VARCHAR(64),
  category_name            VARCHAR(255),
  brand_id                 VARCHAR(64),
  brand_name               VARCHAR(255),
  product_status           VARCHAR(32),                         -- draft | pending_review | active | suspended | deleted
  product_type             VARCHAR(16),                         -- simple | variable

  master_images            TEXT,                                -- comma-separated
  size_chart               JSONB,
  product_attributes       JSONB,

  -- Package dimensions
  package_weight           DECIMAL(10,3),
  package_length           DECIMAL(10,2),
  package_width            DECIMAL(10,2),
  package_height           DECIMAL(10,2),

  -- TikTok-side warehouse association
  tiktok_warehouse_id      VARCHAR(64),

  -- WMS catalogue linkage (cross-DB soft ref)
  wms_catalogue_id         BIGINT,
  wms_catalogue_synced     BOOLEAN NOT NULL DEFAULT FALSE,

  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at               TIMESTAMPTZ,

  CONSTRAINT fbt_products_account_product_key
    UNIQUE (account_id, tiktok_product_id)
);

CREATE INDEX idx_fbt_prod_client ON fbt_products(client_id, account_id);
CREATE INDEX idx_fbt_prod_status ON fbt_products(client_id, product_status);
```

---

### A3. `fbt_product_variations`

```sql
CREATE TABLE fbt_product_variations (
  id                       BIGSERIAL PRIMARY KEY,
  product_id               BIGINT NOT NULL REFERENCES fbt_products(id) ON DELETE CASCADE,
  tiktok_sku_id            VARCHAR(64) NOT NULL,
  sku                      VARCHAR(128) NOT NULL,               -- seller SKU
  price                    DECIMAL(10,2),
  quantity                 INT,                                 -- last-known FBT-available count
  sales_attributes         JSONB,
  image_url                TEXT,

  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT fbt_variations_product_sku_key UNIQUE (product_id, sku)
);

CREATE INDEX idx_fbt_var_sku ON fbt_product_variations(sku);
```

---

### A4. `fbt_orders`

```sql
CREATE TABLE fbt_orders (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES fbt_accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,

  tiktok_order_id          VARCHAR(64) NOT NULL,
  status                   VARCHAR(32),                         -- AWAITING_SHIPMENT | IN_TRANSIT | PARTIALLY_SHIPPING | DELIVERED | CANCELLED

  -- Customer
  buyer_user_id            VARCHAR(64),
  buyer_email              VARCHAR(255),
  buyer_message            TEXT,
  customer_name            VARCHAR(255),
  customer_email           VARCHAR(255),
  customer_phone           VARCHAR(64),

  -- Money
  subtotal                 DECIMAL(10,2),
  shipping_cost            DECIMAL(10,2),
  tax                      DECIMAL(10,2),
  discount                 DECIMAL(10,2),
  total                    DECIMAL(10,2),
  currency                 VARCHAR(8) NOT NULL DEFAULT 'GBP',

  -- Payment
  payment_method           VARCHAR(64),
  payment_status           VARCHAR(16),                         -- pending | paid | refunded

  -- Shipping address
  shipping_name            VARCHAR(255),
  shipping_phone           VARCHAR(64),
  shipping_street1         TEXT,
  shipping_street2         TEXT,
  shipping_city            VARCHAR(128),
  shipping_state           VARCHAR(128),
  shipping_postal_code     VARCHAR(32),
  shipping_country         VARCHAR(64),

  -- Tracking (set by TikTok automatically for FBT)
  shipping_carrier         VARCHAR(64),
  tracking_number          VARCHAR(128),
  shipping_provider_id     VARCHAR(64),
  package_id               VARCHAR(64),

  -- TikTok metadata blob (full raw payload kept for audit)
  tiktok_metadata          JSONB,

  -- Dates
  order_date               TIMESTAMPTZ,
  paid_at                  TIMESTAMPTZ,
  shipped_at               TIMESTAMPTZ,
  delivered_at             TIMESTAMPTZ,
  cancelled_at             TIMESTAMPTZ,

  -- WMS sync state
  wms_synced               BOOLEAN NOT NULL DEFAULT FALSE,
  wms_synced_at            TIMESTAMPTZ,
  wms_order_number         VARCHAR(64),
  wms_message_id           VARCHAR(64),                         -- idempotency key

  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT fbt_orders_account_order_key UNIQUE (account_id, tiktok_order_id)
);

CREATE INDEX idx_fbt_ord_client_date ON fbt_orders(client_id, status, order_date DESC);
CREATE INDEX idx_fbt_ord_wms_pending ON fbt_orders(client_id, wms_synced)
  WHERE wms_synced = FALSE;
```

---

### A5. `fbt_order_items`

```sql
CREATE TABLE fbt_order_items (
  id                       BIGSERIAL PRIMARY KEY,
  order_id                 BIGINT NOT NULL REFERENCES fbt_orders(id) ON DELETE CASCADE,
  tiktok_order_item_id     VARCHAR(64),
  product_id               VARCHAR(64),
  product_name             TEXT,
  sku                      VARCHAR(128),
  seller_sku               VARCHAR(128),
  variation_name           TEXT,
  quantity                 INT NOT NULL,
  unit_price               DECIMAL(10,2),
  total_price              DECIMAL(10,2),
  tax                      DECIMAL(10,2),
  discount                 DECIMAL(10,2),
  image_url                TEXT
);

CREATE INDEX idx_fbt_oi_order ON fbt_order_items(order_id);
CREATE INDEX idx_fbt_oi_sku ON fbt_order_items(sku);
```

---

### A6. `fbt_inventory` — snapshot pattern (multi-version)

```sql
CREATE TABLE fbt_inventory (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES fbt_accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,
  region                   VARCHAR(8) NOT NULL,

  -- SKU identification
  seller_sku               VARCHAR(128) NOT NULL,
  tiktok_sku_id            VARCHAR(64),
  tiktok_product_id        VARCHAR(64),
  product_name             TEXT,
  variation_name           TEXT,

  -- TikTok-side warehouse identification
  tiktok_warehouse_id      VARCHAR(64),
  tiktok_warehouse_name    TEXT,

  -- Quantity buckets
  fulfillable_quantity     INT NOT NULL DEFAULT 0,
  inbound_quantity         INT NOT NULL DEFAULT 0,
  reserved_quantity        INT NOT NULL DEFAULT 0,
  unfulfillable_quantity   INT NOT NULL DEFAULT 0,
  in_transit_quantity      INT NOT NULL DEFAULT 0,
  total_quantity           INT NOT NULL DEFAULT 0,

  -- Breakdowns (JSONB; structure verified in Phase 1 week 1 sandbox tests)
  inbound_breakdown        JSONB,
  reserved_breakdown       JSONB,
  unfulfillable_breakdown  JSONB,

  -- Velocity (computed from fbt_orders WHERE status NOT IN ('CANCELLED'))
  units_sold_7d            INT,
  units_sold_30d           INT,
  velocity_per_day         DECIMAL(10,2),
  days_of_cover            DECIMAL(10,1),
  health_status            VARCHAR(8),                          -- 'green' | 'amber' | 'red' | NULL

  -- Versioning
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

Sync algorithm sketch (in `FbtInventoryService.syncForAccount`):

```typescript
await tx.update(fbt_inventory)
        .set({ is_latest: false })
        .where({ account_id, is_latest: true });
const rows = await api.fetchAllPagesOfInventory(accountId);
await tx.insert(fbt_inventory).values(rows.map(mapRow));
await computeVelocityFor(accountId);
await rabbitmq.publish('tiktok_fbt.to.wms.inventory_synced', { accountId, skus });
```

---

### A7. `fbt_inbound_shipments` (Phase 2; DDL ready now)

```sql
CREATE TABLE fbt_inbound_shipments (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES fbt_accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,
  region                   VARCHAR(8) NOT NULL,

  -- WMS-side reference (cross-DB soft ref)
  wms_inbound_shipment_id  BIGINT,

  -- TikTok identifiers
  tiktok_plan_id           VARCHAR(64),
  tiktok_shipment_id       VARCHAR(64),

  status                   VARCHAR(32) NOT NULL DEFAULT 'draft',
    -- draft | submitted | confirmed | label_generated | in_transit | partially_received | received | cancelled | error

  source_warehouse_id      BIGINT,                              -- merchant's own warehouse, from core WMS
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

  CONSTRAINT fbt_inbound_account_plan_key
    UNIQUE (account_id, tiktok_plan_id) DEFERRABLE INITIALLY DEFERRED
);

CREATE INDEX idx_fbt_inb_status ON fbt_inbound_shipments(client_id, status, created_at DESC);
CREATE INDEX idx_fbt_inb_account ON fbt_inbound_shipments(account_id, status);
```

---

### A8. `fbt_inbound_shipment_items`

```sql
CREATE TABLE fbt_inbound_shipment_items (
  id                       BIGSERIAL PRIMARY KEY,
  shipment_id              BIGINT NOT NULL REFERENCES fbt_inbound_shipments(id) ON DELETE CASCADE,
  seller_sku               VARCHAR(128) NOT NULL,
  tiktok_sku_id            VARCHAR(64),
  product_name             TEXT,
  units_planned            INT NOT NULL,
  units_shipped            INT NOT NULL DEFAULT 0,
  units_received           INT NOT NULL DEFAULT 0,
  units_damaged            INT NOT NULL DEFAULT 0,
  units_short              INT NOT NULL DEFAULT 0,
  unit_cost                DECIMAL(10,2),
  external_metadata        JSONB,
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_fbt_inb_items_shipment ON fbt_inbound_shipment_items(shipment_id);
CREATE INDEX idx_fbt_inb_items_sku ON fbt_inbound_shipment_items(seller_sku);
```

---

### A9. `fbt_fees` (Phase 3)

```sql
CREATE TABLE fbt_fees (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES fbt_accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,
  region                   VARCHAR(8) NOT NULL,

  -- Source
  tiktok_order_id          VARCHAR(64),
  tiktok_statement_id      VARCHAR(64),
  statement_date           DATE,

  fee_type                 VARCHAR(32) NOT NULL,
    -- pick_pack | storage_monthly | storage_long_term | return_processing | damage_claim_credit | reimbursement

  seller_sku               VARCHAR(128),
  units                    INT,

  amount                   DECIMAL(12,2) NOT NULL,
  currency                 VARCHAR(8) NOT NULL,
  volume_cubic_cm_days     DECIMAL(14,2),

  description              TEXT,
  external_metadata        JSONB,

  -- Reconciliation linkage to core
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

### A10. `fbt_returns` (Phase 3)

```sql
CREATE TABLE fbt_returns (
  id                       BIGSERIAL PRIMARY KEY,
  account_id               BIGINT NOT NULL REFERENCES fbt_accounts(id) ON DELETE CASCADE,
  client_id                BIGINT NOT NULL,
  tiktok_return_id         VARCHAR(64) NOT NULL,
  tiktok_order_id          VARCHAR(64),
  seller_sku               VARCHAR(128),
  units                    INT,
  reason                   TEXT,
  condition                VARCHAR(32),                         -- sellable | damaged | destroyed
  status                   VARCHAR(32),                         -- received | reimbursed | disputed
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

### A11. `fbt_activity_logs`, `fbt_webhook_events`

```sql
CREATE TABLE fbt_activity_logs (
  id            BIGSERIAL PRIMARY KEY,
  account_id    BIGINT REFERENCES fbt_accounts(id) ON DELETE CASCADE,
  client_id     BIGINT,
  type          VARCHAR(64),
  message       TEXT,
  metadata      JSONB,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_fbt_log_account ON fbt_activity_logs(account_id, created_at DESC);

CREATE TABLE fbt_webhook_events (
  id              BIGSERIAL PRIMARY KEY,
  event_key       VARCHAR(255) NOT NULL UNIQUE,                  -- "{event_type}:{shop_id}:{event_id or hash}"
  event_type      VARCHAR(64) NOT NULL,
  shop_id         VARCHAR(64),
  received_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  processed_at    TIMESTAMPTZ,
  status          VARCHAR(16) NOT NULL DEFAULT 'received',       -- received | processed | failed | duplicate
  payload         JSONB
);
CREATE INDEX idx_fbt_we_event ON fbt_webhook_events(event_type, received_at DESC);
```

---

## B. Changes inside `core-service` DB (WMS DB)

All additive. None destructive. Same set as the prior plan — the architectural pivot doesn't affect what core-service needs to learn.

### B1. New row in `channels`

```sql
INSERT INTO channels (channel, channel_slug, channel_type, is_active, logo, created_at, updated_at)
VALUES ('TikTok FBT', 'tiktok_fbt', 1, 1, '/assets/marketplaces/tiktok-fbt.png', NOW(), NOW())
ON CONFLICT (channel_slug) DO NOTHING;
```

Existing TikTok seller row stays as `channel_slug='tiktok'`. They are now siblings.

---

### B2. Extend `warehouses` — warehouse type system

```sql
ALTER TABLE warehouses
  ADD COLUMN warehouse_type           VARCHAR(16) NOT NULL DEFAULT 'own'
    CHECK (warehouse_type IN ('own', '3pl', 'virtual', 'fbt_tiktok', 'fba_amazon', 'external')),
  ADD COLUMN external_marketplace     VARCHAR(32),
  ADD COLUMN external_warehouse_id    VARCHAR(64),
  ADD COLUMN external_account_id      BIGINT,
  ADD COLUMN external_region          VARCHAR(8),
  ADD COLUMN external_warehouse_config JSONB;

CREATE INDEX idx_warehouses_type ON warehouses(client_id, warehouse_type, status);
CREATE INDEX idx_warehouses_external
  ON warehouses(external_marketplace, external_warehouse_id)
  WHERE external_marketplace IS NOT NULL;
```

When a merchant connects an FBT account, the new service publishes `tiktok_fbt.to.wms.account_connected`. Core-service consumer inserts one virtual-warehouse row:

```sql
INSERT INTO warehouses (client_id, name, slug, location, status, warehouse_type,
                        external_marketplace, external_warehouse_id, external_account_id,
                        external_region, external_warehouse_config, created_at, updated_at)
VALUES (:clientId, 'TikTok FBT UK', 'tiktok-fbt-uk', :region, 1, 'fbt_tiktok',
        'tiktok_fbt', :tiktokFcCodeOrNull, :fbtAccountId, 'GB', '{}', NOW(), NOW())
```

---

### B3. `order_fulfillments.fulfillment_type` — new value by convention

Already STRING(20). No DDL needed. Document the new value with a comment:

```sql
COMMENT ON COLUMN order_fulfillments.fulfillment_type IS
  'bopis | ship_from_store | fbt. fbt = marketplace-fulfilled (TikTok FBT). Pickers must never receive these.';
```

And add a server-side guard in `OrderFulfillmentService.advanceStatus`:

```javascript
if (orderFulfillment.fulfillmentType === 'fbt') {
  throw new Error('FBT fulfillments are driven by marketplace webhooks only; manual transitions disabled.');
}
```

---

### B4. New table — `variation_listings`

The bridge that makes hybrid SKUs work.

```sql
CREATE TABLE variation_listings (
  id                       BIGSERIAL PRIMARY KEY,
  client_id                INTEGER NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
  variation_id             INTEGER NOT NULL REFERENCES variations(id) ON DELETE CASCADE,
  channel_id               INTEGER NOT NULL REFERENCES channels(id),
  account_id               BIGINT,                              -- soft ref: accounts.id OR fbt_accounts.id depending on channel

  external_listing_id      VARCHAR(128),
  external_sku_id          VARCHAR(128),
  listing_url              TEXT,

  fulfilment_mode          VARCHAR(8) NOT NULL DEFAULT 'fbm'
    CHECK (fulfilment_mode IN ('fbm', 'fbt')),
  warehouse_id             INTEGER REFERENCES warehouses(id),

  is_active                BOOLEAN NOT NULL DEFAULT TRUE,
  last_synced_at           TIMESTAMPTZ,
  external_metadata        JSONB,

  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at               TIMESTAMPTZ,

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

`UNIQUE NULLS NOT DISTINCT` is Postgres 15+. If on an older version, fall back to a partial unique index excluding NULLs.

---

### B5. `inbound_shipments` + items (Phase 2)

Generalised so future FBA inbound reuses the same schema. Field-for-field equivalent of the marketplace-side table in A7/A8 above.

```sql
CREATE TABLE inbound_shipments (
  id                       BIGSERIAL PRIMARY KEY,
  client_id                INTEGER NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
  shipment_number          VARCHAR(32) NOT NULL,                -- INB-2026-001234

  marketplace              VARCHAR(32) NOT NULL,                -- 'tiktok_fbt' (today), 'amazon_fba' (later)
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

CREATE INDEX idx_inb_status ON inbound_shipments(client_id, marketplace, status, created_at DESC);
CREATE INDEX idx_inb_items ON inbound_shipment_items(shipment_id);
```

---

### B6. `marketplace_charges` and `marketplace_reimbursements` (Phase 3)

Generalised, same as before.

```sql
CREATE TABLE marketplace_charges (
  id                       BIGSERIAL PRIMARY KEY,
  client_id                INTEGER NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
  marketplace              VARCHAR(32) NOT NULL,
  marketplace_account_id   BIGINT,
  order_id                 BIGINT REFERENCES orders(id),
  fulfilment_mode          VARCHAR(8),
  charge_type              VARCHAR(32) NOT NULL,
  amount                   DECIMAL(12,2) NOT NULL,
  currency                 VARCHAR(8) NOT NULL,
  units                    INTEGER,
  unit_amount              DECIMAL(10,4),
  external_charge_id       VARCHAR(64),
  external_statement_id    VARCHAR(64),
  statement_date           DATE,
  description              TEXT,
  metadata                 JSONB,
  status                   VARCHAR(16) NOT NULL DEFAULT 'pending',
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_charges_acc_date
  ON marketplace_charges(client_id, marketplace, marketplace_account_id, statement_date DESC);
CREATE INDEX idx_charges_order ON marketplace_charges(order_id) WHERE order_id IS NOT NULL;
CREATE INDEX idx_charges_type ON marketplace_charges(client_id, charge_type, status);

CREATE TABLE marketplace_reimbursements (
  id                       BIGSERIAL PRIMARY KEY,
  client_id                INTEGER NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
  marketplace              VARCHAR(32) NOT NULL,
  marketplace_account_id   BIGINT,
  order_id                 BIGINT REFERENCES orders(id),
  return_request_id        BIGINT REFERENCES return_requests(id),
  reimbursement_type       VARCHAR(32) NOT NULL,
  amount                   DECIMAL(12,2) NOT NULL,
  currency                 VARCHAR(8) NOT NULL,
  external_reimbursement_id VARCHAR(64),
  external_statement_id    VARCHAR(64),
  reason                   TEXT,
  metadata                 JSONB,
  status                   VARCHAR(16) NOT NULL DEFAULT 'pending',
  claimed_at               TIMESTAMPTZ,
  approved_at              TIMESTAMPTZ,
  paid_at                  TIMESTAMPTZ,
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_reimb_acc
  ON marketplace_reimbursements(client_id, marketplace, marketplace_account_id, created_at DESC);
CREATE INDEX idx_reimb_status
  ON marketplace_reimbursements(client_id, status, claimed_at);
```

---

### B7. Extend `return_requests` — marketplace metadata (Phase 3)

```sql
ALTER TABLE return_requests
  ADD COLUMN marketplace             VARCHAR(32),
  ADD COLUMN marketplace_return_id   VARCHAR(64),
  ADD COLUMN marketplace_return_reason TEXT,
  ADD COLUMN return_destination      VARCHAR(32) DEFAULT 'merchant'
    CHECK (return_destination IN ('merchant', 'marketplace_fbt_warehouse', 'marketplace_fba_warehouse'));

CREATE INDEX idx_returns_marketplace
  ON return_requests(client_id, marketplace, marketplace_return_id)
  WHERE marketplace IS NOT NULL;
```

---

## C. Feature-flag plumbing

```sql
-- Phase 1
INSERT INTO subscription_features (key, label, description, is_premium)
VALUES
  ('tiktok_fbt', 'TikTok FBT Integration',
   'Fulfilled by TikTok: inventory mirror, order pass-through, basic dashboard.', true),
  ('tiktok_fbt_inbound', 'TikTok FBT Inbound Shipments',
   'Create and manage inbound shipments to TikTok FCs.', true),
  ('tiktok_fbt_fees', 'TikTok FBT Fee & Reimbursement Reconciliation',
   'Track and reconcile FBT fees and reimbursements.', true);
```

`checkPackageFeature('tiktok_fbt')` middleware gates every `/api/tiktok-fbt/*` route on the gateway side OR every page route on the frontend.

---

## D. Migration ordering and rollback

### Phase 1

**In the new `tiktok-fbt-microservice` repo:**
1. `<ts>-CreateFbtAccounts.ts` — A1
2. `<ts>-CreateFbtProducts.ts` — A2
3. `<ts>-CreateFbtProductVariations.ts` — A3
4. `<ts>-CreateFbtOrders.ts` — A4
5. `<ts>-CreateFbtOrderItems.ts` — A5
6. `<ts>-CreateFbtInventory.ts` — A6
7. `<ts>-CreateFbtActivityLogs.ts` — A11
8. `<ts>-CreateFbtWebhookEvents.ts` — A11

**In `core-service`:**
1. `<ts>-SeedTiktokFbtChannel.js` — B1
2. `<ts>-AddWarehouseTypeColumns.js` — B2
3. `<ts>-CommentFulfillmentTypeOnOrderFulfillments.js` — B3 (comment only)
4. `<ts>-CreateVariationListings.js` — B4
5. `<ts>-SeedSubscriptionFeaturesForFbt.js` — C

### Phase 2

- new repo: 9. `<ts>-CreateFbtInboundShipments.ts` (A7), 10. `<ts>-CreateFbtInboundShipmentItems.ts` (A8)
- core-service: 6. `<ts>-CreateInboundShipments.js` + items (B5)

### Phase 3

- new repo: 11. `<ts>-CreateFbtFees.ts` (A9), 12. `<ts>-CreateFbtReturns.ts` (A10)
- core-service: 7. `<ts>-CreateMarketplaceCharges.js` (B6), 8. `<ts>-CreateMarketplaceReimbursements.js` (B6), 9. `<ts>-ExtendReturnRequestsForMarketplace.js` (B7)

### Rollback

- The new service's DB is independent — dropping the whole `tiktok_fbt_microservice` DB is the cleanest rollback if needed.
- core-service migrations have explicit `down()` that DROPs new columns/tables. Comment in B3 is removed but the underlying data (any rows that happen to have `fulfillment_type='fbt'`) remains valid since the column was already permissive.
- Production rollbacks pair with feature flag flip: set `tiktok_fbt=false` for all clients first, then run `down`.
