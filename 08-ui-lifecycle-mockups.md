# 08 — UI Visualization & Full Lifecycle Mockup

> Written for a non-coder. Every screen the merchant will see, every state every entity can be in, and every event that moves the system from one state to the next.
>
> ASCII mockups are deliberately rough — they convey **what is on the page** and **what changes between states**, not pixel-perfect design. Final designs land in Figma during Phase 0.

---

## Table of contents

- [How to read this document](#how-to-read-this-document)
- [The big picture — one journey, eight data stores](#the-big-picture)
- **Entity walkthroughs (with full UI mockups for every state):**
  - [1. `fbt_accounts` — connecting a TikTok FBT shop](#entity-1-fbt_accounts)
  - [2. `fbt_products` — your FBT-eligible listings](#entity-2-fbt_products)
  - [3. `fbt_inventory` — stock TikTok holds for you](#entity-3-fbt_inventory)
  - [4. `fbt_inbound_shipments` — sending stock to TikTok](#entity-4-fbt_inbound_shipments)
  - [5. `fbt_orders` — sales TikTok fulfils on your behalf](#entity-5-fbt_orders)
  - [6. `fbt_fees` — what TikTok charges (and credits)](#entity-6-fbt_fees)
  - [7. `fbt_activity_logs` — the audit trail](#entity-7-fbt_activity_logs)
  - [8. `fbt_webhook_events` — the invisible plumbing (with a debug view)](#entity-8-fbt_webhook_events)
- **Day-in-the-life scenarios (end-to-end stories):**
  - [Scenario A — First-time setup](#scenario-a-first-time-setup)
  - [Scenario B — Restocking a low SKU](#scenario-b-restocking-a-low-sku)
  - [Scenario C — A customer places an order](#scenario-c-a-customer-places-an-order)
  - [Scenario D — Reconciling monthly fees](#scenario-d-reconciling-monthly-fees)
  - [Scenario E — Handling a damaged-in-FC return](#scenario-e-handling-a-damaged-in-fc-return)
- **State machines (one-page references):**
  - [FBT order journey](#state-machine-fbt-order-journey)
  - [Inbound shipment journey](#state-machine-inbound-shipment-journey)

---

## How to read this document

Each entity section has the same shape:

1. **Why it exists** — one paragraph, plain English.
2. **The lifecycle** — a small flow diagram of every state the entity can be in.
3. **Where the merchant sees it** — which page(s), which sidebar entry.
4. **Mockups for every state** — what the screen looks like when the entity is in that state.
5. **What triggers each transition** — exactly what the merchant clicks, or what TikTok sends us, to move the entity to the next state.
6. **What lives invisibly behind the scenes** — the bits the merchant never sees but you should know about.

The mockups use these conventions:

```
┌──────────────────────────────┐
│  Title bar / section label   │   ← always at top of a box
├──────────────────────────────┤
│ [Button]   ← clickable        │
│ ☐ checkbox   ▼ dropdown       │
│ ● green   ● amber   ● red     │   ← health indicators
│ ──────────────────────────    │   ← visual separator
└──────────────────────────────┘
```

URLs above every mockup tell you which page the merchant is on. Routes start with `/clients/tiktok-fbt/...`.

---

## The big picture

Here is the whole merchant journey on one page. Every entity has its own column. Read top to bottom.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  Connect      ──► Listings/        ──► Inbound          ──► Inventory      ──► Orders         ──► Fees       │
│  account      │   Products         │   shipments        │                  │                  │   & Returns  │
│               │                    │                    │                  │                  │              │
│  fbt_accounts │   fbt_products     │   fbt_inbound_     │   fbt_inventory  │   fbt_orders     │   fbt_fees   │
│               │   fbt_product_     │   shipments        │                  │   fbt_order_     │   (Phase 3)  │
│               │   variations       │   _items           │                  │   items          │              │
│  Phase 1      │   Phase 1          │   Phase 2          │   Phase 1        │   Phase 1        │   Phase 3    │
└──────────┬────┴─────────┬──────────┴──────────┬─────────┴────────┬─────────┴─────────┬────────┴──────┬───────┘
           │              │                     │                  │                   │               │
           ▼              ▼                     ▼                  ▼                   ▼               ▼
                                fbt_activity_logs  (a running diary of everything above)

                                fbt_webhook_events (invisible: dedupes events TikTok pushes to us)
```

Reading top-to-bottom: a merchant first **connects** their TikTok FBT shop. Once connected, we pull their **listings** (products eligible for FBT). The merchant then creates **inbound shipments** — physically sending boxes to TikTok's warehouses. Once those arrive, TikTok updates **inventory** (which we mirror every 15 minutes). Buyers place **orders** which TikTok fulfils. Monthly **fees** are settled. Everything that happens is recorded in **activity logs**. **Webhook events** is the internal plumbing that keeps things in sync — the merchant never looks at it directly, but engineers and support staff do.

---

## Entity 1 — `fbt_accounts`

### Why it exists

A merchant might run multiple TikTok shops (UK, US, EU). Each FBT-enrolled shop is one row in `fbt_accounts`. This is the entry point for everything FBT-related — until an account exists, the rest of the system has nothing to do.

### The lifecycle

```
   ┌─────────────┐    [Connect]    ┌────────────────┐  OAuth ok    ┌─────────────┐
   │  none yet   │ ─────────────► │  pending OAuth  │ ───────────► │  connected  │
   └─────────────┘                 └────────────────┘                └──────┬──────┘
                                                                            │
                                                       [Disconnect] OR token expires
                                                                            ▼
                                                                    ┌──────────────┐
                                                                    │ disconnected │
                                                                    └──────────────┘
                                          (a connected account also has periodic verifications
                                           that update `last_verified_at` — same state, different data)
```

### Where the merchant sees it

- Sidebar: **TikTok FBT > Accounts**
- URL: `/clients/tiktok-fbt/accounts`

### State A — Empty (no accounts yet)

URL: `/clients/tiktok-fbt/accounts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › TikTok FBT › Accounts                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   TikTok FBT Accounts                                  [+ Connect FBT shop] │
│                                                                              │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                       │   │
│   │           🛒  No FBT shops connected yet                              │   │
│   │                                                                       │   │
│   │     Connect a TikTok shop that's enrolled in Fulfilled by TikTok      │   │
│   │     to start mirroring inventory, orders, and fees in WMS.            │   │
│   │                                                                       │   │
│   │     Don't have FBT enabled?                                           │   │
│   │     › Apply on TikTok Seller Center                                   │   │
│   │                                                                       │   │
│   │                       [+ Connect FBT shop]                            │   │
│   │                                                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**What the merchant does:** clicks **[+ Connect FBT shop]**.

### State B — Connect modal (pre-OAuth)

```
        ┌────────────────────────────────────────────────────┐
        │  Connect a TikTok FBT shop                    [X]  │
        ├────────────────────────────────────────────────────┤
        │                                                      │
        │   Shop region:    [ United Kingdom (GB)         ▼ ] │
        │                                                      │
        │   You'll be redirected to TikTok to authorise us.   │
        │   We only request scopes needed for FBT.            │
        │                                                      │
        │   ─────────────────────────────────────────────     │
        │                                                      │
        │                   [Cancel]    [Continue to TikTok →]│
        │                                                      │
        └────────────────────────────────────────────────────┘
```

**What the merchant does:** picks the region, clicks **[Continue to TikTok →]**.

**What happens behind the scenes:** browser redirects to `auth.tiktok-shops.com/oauth/authorize?...`. The merchant logs into TikTok (if not already) and clicks Authorise. TikTok redirects back to `/api/tiktok-fbt/auth/callback?code=...`. Our service exchanges the code for access + refresh tokens, encrypts them (AES-256-GCM), inserts the `fbt_accounts` row, then redirects back to the accounts page with a success toast.

### State C — Account just connected (interstitial: link to seller account)

URL: `/clients/tiktok-fbt/accounts?just_connected=42` (id 42 is the new account)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ✓ Acme Ltd — UK FBT connected                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Is this the same TikTok shop as one of your seller connections?           │
│                                                                              │
│   Linking lets us show side-by-side stock (own warehouse + FBT) in your     │
│   dashboards. Linking is optional and can be changed any time.              │
│                                                                              │
│   Existing seller accounts:                                                  │
│   ○  Acme Ltd — Seller (id 142, GB)                                         │
│   ○  Acme Trade — Seller (id 156, GB)                                       │
│   ○  None of the above / skip for now                                       │
│                                                                              │
│                                                  [Skip]   [Save link]       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**What the merchant does:** picks one (or Skip).

**What happens:** updates `fbt_accounts.linked_seller_account_id` if a row was chosen.

### State D — Account list with one connected account

URL: `/clients/tiktok-fbt/accounts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TikTok FBT Accounts                                   [+ Connect FBT shop] │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─ Acme Ltd — UK FBT ─────────────────────────────────────────────────┐    │
│  │                                                                        │    │
│  │  Region:      GB                  Status:    ● Connected               │    │
│  │  Shop ID:     7290xxxxxxx          Last verified: 12 May 09:14         │    │
│  │  Linked to:   Acme Ltd — Seller (id 142)        [Change link]         │    │
│  │                                                                        │    │
│  │  ┌─ Sync settings ────────────────────────────────────────────┐       │    │
│  │  │  Sync orders         ●● ON       Last: 12 May 09:31         │       │    │
│  │  │  Sync inventory      ●● ON       Last: 12 May 09:26         │       │    │
│  │  │  Publish to WMS      ●● ON                                  │       │    │
│  │  └─────────────────────────────────────────────────────────────┘       │    │
│  │                                                                        │    │
│  │                            [Sync now]  [Capabilities]  [Disconnect]    │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**What the merchant does most often:** clicks `[Sync now]` to force a fresh poll without waiting 15 minutes.

### State E — Disconnected account (kept for history)

```
┌─ Acme Ltd — UK FBT ─────────────────────────────────────────────────┐
│                                                                        │
│  Region:      GB                  Status:    ⨯ Disconnected            │
│  Shop ID:     7290xxxxxxx          Disconnected: 18 May 14:02          │
│                                                                        │
│  Past data is preserved (in-flight orders continue to be tracked).     │
│  No new inventory polls or order syncs will run.                       │
│                                                                        │
│                                                       [Reconnect]      │
└────────────────────────────────────────────────────────────────────────┘
```

### Triggers — what moves the state forward

| From | To | Trigger |
|---|---|---|
| none | pending OAuth | merchant clicks [Continue to TikTok →] |
| pending OAuth | connected | TikTok redirects back with `?code=...` and we successfully exchange for tokens |
| connected | connected (verified) | hourly verification cron confirms token still works |
| connected | disconnected | merchant clicks [Disconnect] **or** refresh token expires and re-OAuth fails |
| disconnected | connected | merchant clicks [Reconnect], full OAuth again |

### Invisible side-effects of connecting an account

1. A row is inserted into `fbt_accounts` (encrypted tokens, shop info).
2. A RabbitMQ message `tiktok_fbt.to.wms.account_connected` is published.
3. Core-service consumes it and inserts one **virtual warehouse** row into `warehouses` with `warehouse_type='fbt_tiktok'`. This is the warehouse that future FBT inventory will live in.
4. First inventory poll is scheduled (runs within 15 min of connection).
5. First order poll is scheduled (runs within 5 min).
6. An entry is added to `fbt_activity_logs` saying "Connected".

---

## Entity 2 — `fbt_products`

### Why it exists

Once an account is connected, the service pulls every product (listing) the merchant has on that TikTok shop that is FBT-eligible. Each product has one or more variations (sizes, colours) tracked in `fbt_product_variations`. This entity is the canonical list of "what could possibly be sold through FBT."

### The lifecycle

```
                       ┌──────────────┐
TikTok publish ──►     │   pending    │ TikTok approves   ┌──────────┐
                       │   review     │ ───────────────►  │  active  │
                       └──────────────┘                    └─────┬────┘
                                                                  │
                                                  TikTok suspends │
                                                                  ▼
                                                            ┌──────────┐
                                                            │ suspended│ ──► [merchant rectifies]
                                                            └──────────┘
                                                                  │
                                                       merchant deletes (TikTok side)
                                                                  ▼
                                                            ┌──────────┐
                                                            │  deleted │  (kept as tombstone for history)
                                                            └──────────┘
```

### Where the merchant sees it

- Sidebar: **TikTok FBT > Inventory** (products and stock are merged into one view in our UI)
- Also visible per row in `/clients/tiktok-fbt/inventory` and as a filter facet

### View: empty state (no products synced yet)

URL: `/clients/tiktok-fbt/inventory`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › TikTok FBT › Inventory                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   FBT Inventory                                                              │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                       │   │
│   │           ⏳  First sync in progress…                                  │   │
│   │                                                                       │   │
│   │     We're pulling your FBT products from TikTok now.                  │   │
│   │     This page will populate within ~15 minutes.                        │   │
│   │                                                                       │   │
│   │     Next sync at: 09:30:00  (in 7 min)                                │   │
│   │                                                                       │   │
│   │                           [Refresh manually]                          │   │
│   │                                                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### View: product listed but not yet mapped to WMS catalogue

```
┌─────┬────────┬──────────────────────┬────────┬─────────┬─────────┬────────────┐
│ ☐   │ Image  │ SKU / Name           │ Status │ Fulfil. │ Inbound │ WMS map    │
├─────┼────────┼──────────────────────┼────────┼─────────┼─────────┼────────────┤
│ ☐   │ [img]  │ NEW-WIDGET-RED-L     │ active │   0     │   0     │ ⚠ Unmapped │
│     │        │ Red widget (large)    │        │         │         │ [Map now]  │
└─────┴────────┴──────────────────────┴────────┴─────────┴─────────┴────────────┘
```

"Unmapped" means the SKU exists on TikTok but we don't yet have a matching row in our WMS `variations` table. Clicking **[Map now]** opens a search-and-link dialog.

### View: mapped, active, healthy product

```
┌─────┬────────┬──────────────────────┬────────┬─────────┬─────────┬────────────┐
│ ☐   │ Image  │ SKU / Name           │ Status │ Fulfil. │ Inbound │ WMS map    │
├─────┼────────┼──────────────────────┼────────┼─────────┼─────────┼────────────┤
│ ☐   │ [img]  │ WIDGET-RED-L         │ active │  200    │   0     │ ✓ #4521    │
│     │        │ Red widget (large)    │        │         │         │            │
└─────┴────────┴──────────────────────┴────────┴─────────┴─────────┴────────────┘
```

### View: suspended product

```
┌─────┬────────┬──────────────────────┬────────────┬──────────────────────┐
│ ☐   │ Image  │ SKU / Name           │ Status     │ Notes                │
├─────┼────────┼──────────────────────┼────────────┼──────────────────────┤
│ ☐   │ [img]  │ WIDGET-BAD-L         │ suspended  │ TikTok policy: image │
│     │        │ Bad widget            │ ⚠ red      │ resolution too low   │
│     │        │                       │            │ [Fix in TikTok]      │
└─────┴────────┴──────────────────────┴────────────┴──────────────────────┘
```

The "Fix in TikTok" link opens TikTok Seller Center deep-linked to the suspended product. We don't fix listings from WMS — TikTok owns the listing surface.

### Triggers

| From | To | Trigger |
|---|---|---|
| (new) | pending review | TikTok product appears in our 15-min listings poll for the first time |
| pending review | active | Webhook `PRODUCT_STATUS_CHANGE` with new status `active`, OR next poll picks up the change |
| active | suspended | Webhook `PRODUCT_STATUS_CHANGE` with `suspended` |
| suspended | active | merchant fixes in TikTok, then status change webhook fires |
| any | deleted | merchant deletes in TikTok |

### Invisible side-effects

- When a new product appears, we look up the merchant's WMS catalogue by SKU. If found, auto-create a `variation_listings` row in core WMS DB (with `fulfilment_mode='fbt'`, pointing at the virtual fbt_tiktok warehouse). If not found, leave it unmapped and flag in UI.

---

## Entity 3 — `fbt_inventory`

### Why it exists

This is the most operationally important entity. Every 15 minutes we poll TikTok for **how much stock TikTok currently holds for each SKU.** Each poll inserts a fresh snapshot. The merchant's real-time decisions (replenish? raise price? pause listing?) all depend on this data.

### Two important concepts

**Snapshot pattern.** Each row in `fbt_inventory` is a snapshot at a point in time. We mark the most recent one `is_latest = TRUE`; older rows stay around for history (pruned after 90 days). When you see "Last synced 7 min ago" on a page, that 7 min is the age of the latest snapshot.

**Health colour.** Each SKU gets a green/amber/red badge:
- **● Green** — more than 30 days of stock at current sales velocity. Safe.
- **● Amber** — 14 to 30 days of stock. Plan a restock soon.
- **● Red** — under 14 days of stock. Restock now or risk stockout.

### Where the merchant sees it

- Sidebar: **TikTok FBT > Inventory**
- Plus a summary stat tile on **TikTok FBT > Dashboard**
- URL: `/clients/tiktok-fbt/inventory`

### View: healthy inventory

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › TikTok FBT › Inventory               Last synced: 7 min ago [↻]      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   FBT Inventory                                                              │
│                                                                              │
│   [Region: UK ▾]  [Account: All ▾]  [Health: All ▾]  [🔍 Search SKU…]        │
│                                                                              │
│   ┌─────┬───────┬───────────────────┬───────┬───────┬──────┬───────┬───────┐│
│   │  ☐  │ Image │ SKU / Name        │Fulfil-│Inbnd  │Resv  │ Days  │Health ││
│   │     │       │                   │ lable │       │      │ cover │       ││
│   ├─────┼───────┼───────────────────┼───────┼───────┼──────┼───────┼───────┤│
│   │  ☐  │ [img] │ WIDGET-RED-L      │  200  │   0   │   8  │  41.7 │● Green││
│   │  ☐  │ [img] │ WIDGET-BLU-L      │  150  │   0   │   3  │  37.5 │● Green││
│   │  ☐  │ [img] │ WIDGET-GRN-L      │   42  │ 120   │   3  │   6.5 │● Red  ││
│   │  ☐  │ [img] │ GIZMO-S           │  102  │   0   │   2  │  20.4 │● Amber││
│   │  ☐  │ [img] │ GIZMO-M           │  390  │   0   │   1  │  78.0 │● Green││
│   └─────┴───────┴───────────────────┴───────┴───────┴──────┴───────┴───────┘│
│                                                                              │
│   Showing 5 of 147 items   ◄  Page 1 of 30  ►                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### View: row drawer (clicking on WIDGET-GRN-L for full breakdown)

```
        ┌────────────────────────────────────────────────────────┐
        │  WIDGET-GRN-L — Green widget (large)             [X]   │
        ├────────────────────────────────────────────────────────┤
        │                                                          │
        │   TikTok FC:   TikTok UK Bedford                         │
        │   Latest snapshot: 12 May 09:23 (7 min ago)              │
        │                                                          │
        │   ┌──────────────────────────────────────────────┐      │
        │   │  Fulfillable          42                      │      │
        │   │  Inbound (in transit) 120                     │      │
        │   │    - working:    20                            │      │
        │   │    - shipped:    80                            │      │
        │   │    - receiving:  20                            │      │
        │   │  Reserved              3                      │      │
        │   │    - pending order: 3                          │      │
        │   │  Unfulfillable         1                      │      │
        │   │    - damaged:    1                             │      │
        │   └──────────────────────────────────────────────┘      │
        │                                                          │
        │   Sold last 7 days:   46    Sold last 30 days: 195       │
        │   Velocity / day:    6.5    Days of cover:     6.5       │
        │   Health: ● Red                                          │
        │                                                          │
        │   30-day sales:                                          │
        │      ▁▂▃▅▆▇█▇▆▅▆▇█▇▆▅▆▇█▇▆▅▆▇█▇▆▅▆▇█▇                  │
        │                                                          │
        │   WMS own-warehouse stock:                               │
        │      Manchester DC:  85                                  │
        │      London DC:       0                                  │
        │                                                          │
        │              [Create inbound shipment]   [Close]         │
        └────────────────────────────────────────────────────────┘
```

### View: stale-snapshot warning (sync hasn't run for >30 min)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ⚠ Inventory data is 47 minutes old — last sync attempt failed.              │
│    Check account connection or click [Sync now].                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Triggers

| Event | Result |
|---|---|
| 15-min cron fires | poll TikTok, mark previous `is_latest=false`, insert fresh rows with `is_latest=true` |
| Merchant clicks [↻ Sync now] | same as above, but on demand (rate-limited to once / 5 min per account) |
| FBT order arrives (webhook) | optimistic local decrement of fulfillable by ordered qty; next 15-min poll reconciles |
| Inbound shipment received | nothing on our side — TikTok's next inventory report will reflect the new stock |
| Two snapshots fail in a row | UI shows stale-snapshot warning; an alert fires to ops |

### Invisible side-effects

- Velocity recalculated after every sync (7d / 30d sales from `fbt_orders`).
- Health colour assigned per SKU based on `days_of_cover`.
- A RabbitMQ message `tiktok_fbt.to.wms.inventory_synced` is published with the new snapshot summary so core-service can keep the virtual warehouse's reported stock in sync.

---

## Entity 4 — `fbt_inbound_shipments`

### Why it exists

The merchant ships boxes of stock to TikTok FCs. Each shipment is tracked end-to-end: from the moment they decide what to send, through label printing, carrier hand-off, partial receipt, and final reconciliation. This entity is the source of truth for "what's on the way to TikTok."

### The lifecycle (long — has the most states)

```
   draft  ─[Submit]─►  submitted  ─[Confirm]─►  confirmed  ─[Print labels]─►  label_generated
                                                                                       │
                                                                          [Mark shipped]│
                                                                                       ▼
   ┌─────── cancelled (any time before in_transit)                                in_transit
   │                                                                                   │
   │                                                          first units arrive at FC │
   │                                                                                   ▼
   │                                                                       partially_received
   │                                                                                   │
   │                                                              all units arrive (or │
   │                                                              shortage finalised)  │
   │                                                                                   ▼
   │                                                                              received
   │                                                                                   │
   │                                                              (terminal — read-only history)
```

### Where the merchant sees it

- Sidebar: **TikTok FBT > Inbound Shipments** (Phase 2; feature flag `tiktok_fbt_inbound`)
- URLs: `/clients/tiktok-fbt/inbound` (list) · `/clients/tiktok-fbt/inbound/[id]` (detail) · `/clients/tiktok-fbt/inbound/new` (wizard)

### Wizard — Step 1 of 4: choose products and quantities

URL: `/clients/tiktok-fbt/inbound/new`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  New inbound shipment                                            Step 1 of 4│
├─────────────────────────────────────────────────────────────────────────────┤
│   Products ──► FC ──► Review ──► Labels                                      │
│   ●●            ○         ○          ○                                       │
│                                                                              │
│   Source warehouse:    [ Manchester DC               ▼ ]                     │
│                                                                              │
│   Add products to ship:                                                      │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │ SKU            │ Available (own) │ Days at FBT │ Units to ship     │    │
│   ├────────────────────────────────────────────────────────────────────┤    │
│   │ WIDGET-GRN-L   │  85             │ ● Red 6.5d  │ [   500       ]   │    │
│   │ GIZMO-S        │  410            │ ● Amber 20d │ [   200       ]   │    │
│   │ + add another                                                       │    │
│   └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│   Total units to ship: 700                                                   │
│                                                                              │
│                                            [Cancel]              [Next →]    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Wizard — Step 2 of 4: choose destination FC

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  New inbound shipment                                            Step 2 of 4│
├─────────────────────────────────────────────────────────────────────────────┤
│   Products ──► FC ──► Review ──► Labels                                      │
│   ✓             ●●        ○          ○                                       │
│                                                                              │
│   Destination FC:                                                            │
│                                                                              │
│   ○  Let TikTok decide   (recommended)                                       │
│   ●  TikTok FBT — Bedford   (capacity OK, ~2-day inbound)                    │
│   ○  TikTok FBT — Manchester  (capacity tight, ~3-day inbound)               │
│   ○  TikTok FBT — London   (full, accepting partial only)                    │
│                                                                              │
│                                            [← Back]              [Next →]    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Wizard — Step 3 of 4: review

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  New inbound shipment                                            Step 3 of 4│
├─────────────────────────────────────────────────────────────────────────────┤
│   Products ──► FC ──► Review ──► Labels                                      │
│   ✓             ✓        ●●         ○                                        │
│                                                                              │
│   Summary                                                                    │
│                                                                              │
│   From:    Manchester DC                                                     │
│   To:      TikTok FBT — Bedford                                              │
│   Units:   700  (across 2 SKUs)                                              │
│                                                                              │
│   ┌────────────────────────────────────────────────────────────┐            │
│   │ SKU            │ Units │ Unit cost │ Line value             │            │
│   ├────────────────────────────────────────────────────────────┤            │
│   │ WIDGET-GRN-L   │   500 │  £2.40    │  £1,200.00             │            │
│   │ GIZMO-S        │   200 │  £4.50    │    £900.00             │            │
│   └────────────────────────────────────────────────────────────┘            │
│   Total declared value:  £2,100.00                                           │
│                                                                              │
│   Expected dispatch by:  15 May 2026 (2 working days)                        │
│                                                                              │
│   Notes (optional): [                                                      ] │
│                                                                              │
│                                            [← Back]    [Submit plan →]       │
└─────────────────────────────────────────────────────────────────────────────┘
```

**What happens on submit:** we call TikTok's "create inbound plan" endpoint, get a `plan_id`, persist as a new `fbt_inbound_shipments` row with status `submitted`.

### Wizard — Step 4 of 4: labels & ship

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Inbound shipment INB-2026-0001                                  Step 4 of 4│
├─────────────────────────────────────────────────────────────────────────────┤
│   Products ──► FC ──► Review ──► Labels                                      │
│   ✓             ✓        ✓        ●●                                         │
│                                                                              │
│   ✓ Plan submitted successfully  (TikTok plan ID: PLN-99887)                │
│                                                                              │
│   Print your shipping labels:                                                │
│                                                                              │
│        ┌──────────────────────────────────┐                                 │
│        │  📄  inbound_labels_INB-2026-0001.pdf  │  [Download]                │
│        │  3 pages · 700 units · 5 boxes    │                                 │
│        └──────────────────────────────────┘                                 │
│                                                                              │
│   After you hand the boxes to the carrier:                                   │
│                                                                              │
│   Carrier:           [ DPD                              ▼ ]                  │
│   Tracking number:   [                                       ]               │
│                                                                              │
│                                  [Save as draft]    [Mark as shipped →]      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### List view (all shipments)

URL: `/clients/tiktok-fbt/inbound`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Inbound Shipments                                              [+ New]      │
├─────────────────────────────────────────────────────────────────────────────┤
│   [Status: All ▾]  [Region ▾]  [🔍 Search…]                                  │
│                                                                              │
│   Shipment        Created   Dest.            Units  Status              Track│
│   ─────────────────────────────────────────────────────────────────────────  │
│   INB-2026-0003   12 May    FBT UK Bedford    700  ● Label generated   —    │
│   INB-2026-0002   11 May    FBT UK Manch.      80  ● Submitted         —    │
│   INB-2026-0001   10 May    FBT UK Bedford    150  ● In transit  DPD123↗    │
│   INB-2026-0000   08 May    FBT UK Bedford     90  ● Received   DPD110↗    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Detail view of an in-transit shipment

URL: `/clients/tiktok-fbt/inbound/0001`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › TikTok FBT › Inbound › INB-2026-0001                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   INB-2026-0001                                            ● In transit      │
│                                                                              │
│   From:        Manchester DC          To:    TikTok FBT — Bedford            │
│   Created:     10 May 14:00           Shipped: 11 May 09:14                  │
│   Carrier:     DPD                    Tracking: DPD123XYZ ↗                  │
│   Expected:    13 May 2026                                                   │
│                                                                              │
│   Timeline                                                                   │
│   ●  Draft                  10 May 13:42                                     │
│   ●  Submitted              10 May 14:00                                     │
│   ●  Plan confirmed         10 May 14:01                                     │
│   ●  Labels generated       10 May 14:02                                     │
│   ●  Shipped (handed off)   11 May 09:14                                     │
│   ○  Receiving at FC                                                         │
│   ○  Received                                                                │
│                                                                              │
│   Items                                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │ SKU             │ Planned │ Shipped │ Received │ Damaged │ Short │       │
│   ├─────────────────────────────────────────────────────────────────┤       │
│   │ WIDGET-GRN-L    │   500   │   500   │     0    │    0    │   0   │       │
│   │ GIZMO-S         │   200   │   200   │     0    │    0    │   0   │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│                                                                              │
│                              [Cancel shipment]   [Open in TikTok ↗]          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Detail view after partial / damaged receipt

```
   Items
   ┌─────────────────────────────────────────────────────────────────┐
   │ SKU             │ Planned │ Shipped │ Received │ Damaged │ Short │
   ├─────────────────────────────────────────────────────────────────┤
   │ WIDGET-GRN-L    │   500   │   500   │   495    │    3    │   2   │
   │ GIZMO-S         │   200   │   200   │   200    │    0    │   0   │
   └─────────────────────────────────────────────────────────────────┘

   ⚠ Discrepancy detected:
      - 3 units damaged on arrival (WIDGET-GRN-L)  →  reimbursement claim created
      - 2 units short                                →  see [Dispute] action below
```

### Triggers

| From | To | Trigger |
|---|---|---|
| draft | submitted | merchant clicks [Submit plan] |
| submitted | confirmed | TikTok confirms via API |
| confirmed | label_generated | merchant clicks [Print labels] (we call the labels endpoint) |
| label_generated | in_transit | merchant enters tracking and clicks [Mark as shipped] |
| in_transit | partially_received | TikTok webhook `FBT_INBOUND_RECEIVED` (first arrival) **or** 30-min status poll |
| partially_received | received | all expected units arrived |
| any pre-shipment | cancelled | merchant clicks [Cancel shipment] |

### Invisible side-effects

- Each transition publishes `tiktok_fbt.to.wms.inbound_status_changed` so core-service `inbound_shipments` mirror stays in sync.
- On `received`, core-service decrements the source own-warehouse inventory.
- On any discrepancy (damaged or short), Phase-3 logic creates a placeholder `marketplace_reimbursements` row.

---

## Entity 5 — `fbt_orders`

### Why it exists

Every order placed by a TikTok buyer where TikTok fulfils the order on your behalf. The merchant doesn't pick or pack these — TikTok does. **But the merchant still needs to see them** for revenue, customer service, refunds, and analytics.

### The lifecycle

```
   webhook arrives     ┌──────────────────────┐
   from TikTok    ──►  │  AWAITING_SHIPMENT   │
                       └──────────┬───────────┘
                                  │
                       TikTok ships
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │     IN_TRANSIT       │
                       └──────────┬───────────┘
                                  │
                       carrier delivers
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │     DELIVERED        │
                       └──────────────────────┘

   (any time before delivery) ──► CANCELLED
```

### Where the merchant sees it

- Sidebar: **TikTok FBT > Orders**
- Also folded into the central order list at `/clients/order` with a "Fulfilled by TikTok" badge
- URL: `/clients/tiktok-fbt/orders`

### View: order list

URL: `/clients/tiktok-fbt/orders`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › TikTok FBT › Orders                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   FBT Orders                                            [Sync now]  [Export]│
│                                                                              │
│   [Status: All ▾]  [Date: Last 30 days ▾]  [🔍 Search order# / SKU…]         │
│                                                                              │
│   Order #        Date          Items  Total    Status         Tracking      │
│   ─────────────────────────────────────────────────────────────────────────  │
│   TK-12345       12 May 09:14  2     £42.00   ● In transit   AB123 ↗       │
│   TK-12344       12 May 08:30  1     £15.00   ● Awaiting…    —             │
│   TK-12343       11 May 19:01  3     £67.50   ● Delivered    AB121 ↗       │
│   TK-12342       11 May 15:22  1     £15.00   ⨯ Cancelled    —             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

Every row carries the same visual tag everywhere it appears in WMS:

```
 [ ⚡ Fulfilled by TikTok ]
```

### View: order detail drawer

```
        ┌────────────────────────────────────────────────────────┐
        │  TK-12345                                       [X]    │
        │  ● In transit          [⚡ Fulfilled by TikTok]         │
        ├────────────────────────────────────────────────────────┤
        │                                                          │
        │   Order date:  12 May 09:14                              │
        │   Customer:    John Doe                                  │
        │   Address:     12 Example Street, London, SW1 0AA, GB    │
        │                                                          │
        │   Items                                                  │
        │   ┌────────────────────────────────────────────────┐    │
        │   │ WIDGET-RED-L · 2 × £15.00   = £30.00            │    │
        │   │ STICKER-PACK · 1 × £12.00   = £12.00            │    │
        │   └────────────────────────────────────────────────┘    │
        │                                                          │
        │   Subtotal       £42.00                                  │
        │   Shipping        £0.00                                  │
        │   Tax             £0.00                                  │
        │   Total          £42.00  GBP                             │
        │                                                          │
        │   Tracking:  AB123XYZ ↗   carrier: Royal Mail            │
        │                                                          │
        │   TikTok-side status timeline                            │
        │   ●  AWAITING_SHIPMENT      12 May 09:14                 │
        │   ●  IN_TRANSIT             12 May 14:20                 │
        │   ○  DELIVERED                                            │
        │                                                          │
        │   ────────────────────────────────────────────────       │
        │   You do not need to pick/pack this order.               │
        │   TikTok handles fulfilment.                             │
        │                                                          │
        │                          [Open in TikTok Seller Center ↗]│
        └────────────────────────────────────────────────────────┘
```

### Picker view (proof FBT orders never reach pickers)

URL: `/clients/order/awaiting-pick`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Orders awaiting pick                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  Order #         Channel       Items  Warehouse                              │
│  ──────────────────────────────────────────────────────────────────────────  │
│  EB-7788         eBay          3     Manchester DC                           │
│  SH-2200         Shopify       1     London DC                               │
│  TK-12999        TikTok        2     Manchester DC                           │
│                                                                              │
│  (FBT orders such as TK-12345 are NOT listed here — TikTok fulfils them.)    │
└─────────────────────────────────────────────────────────────────────────────┘
```

That last comment is real UI text, not a developer note. The merchant should see the system **explicitly say** "FBT orders are not here, by design" — so they don't wonder where the TK-12345 went.

### Triggers

| From | To | Trigger |
|---|---|---|
| (new) | AWAITING_SHIPMENT | webhook `ORDER_STATUS_CHANGE` arrives with that status |
| AWAITING_SHIPMENT | IN_TRANSIT | webhook `ORDER_STATUS_CHANGE` or `PACKAGE_UPDATE` |
| IN_TRANSIT | DELIVERED | webhook `PACKAGE_UPDATE` final status |
| any | CANCELLED | webhook `ORDER_STATUS_CHANGE` with `CANCELLED` |

### Invisible side-effects of order creation

1. `fbt_orders` + `fbt_order_items` rows inserted.
2. RabbitMQ message `tiktok_fbt.to.wms.order_created` published.
3. Core-service creates an `orders` row with `createdVia='tiktok_fbt'`.
4. Core-service creates `order_fulfillments` row with `fulfillment_type='fbt'`, `warehouse_id=<virtual fbt warehouse>`.
5. Virtual warehouse inventory decremented by ordered quantity.
6. No picker queue entry created.
7. `fbt_activity_logs` entry added.

---

## Entity 6 — `fbt_fees`

### Why it exists (Phase 3)

TikTok bills the merchant for:
- **Pick & pack** per order line
- **Storage** per cubic-cm-day
- **Long-term storage** (if stock lingers > N days)
- **Return processing** when a buyer returns an item

And TikTok **credits** the merchant for:
- **Damage** while stock was in TikTok's care
- **Lost** units
- **Dispute won**

All of this lands in the merchant's TikTok Seller Center statement, daily and monthly. We ingest those statements once a day at 03:15 and lay them out so the merchant can see — by SKU, by month, by fee type — what TikTok has billed them, what they've been credited, and where the numbers diverge from expectation.

### Where the merchant sees it

- Sidebar: **TikTok FBT > Fees & Reimbursements** (Phase 3; feature flag `tiktok_fbt_fees`)
- URL: `/clients/tiktok-fbt/fees`

### View: dashboard

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › TikTok FBT › Fees & Reimbursements                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Fees & Reimbursements                            [Export CSV]             │
│                                                                              │
│   [Range: Last 30 days ▾]  [Account ▾]  [Type ▾]                             │
│                                                                              │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐               │
│   │ Total fees │ Pick & Pack  │  Storage     │ Reimbursed   │               │
│   │  £2,840    │   £1,720     │   £620       │   £312       │               │
│   └────────────┘ └────────────┘ └────────────┘ └────────────┘               │
│                                                                              │
│   ┌─ Fees by week ────────────────────────────────────────────┐             │
│   │   £                                                         │             │
│   │   ┊  ▓▓▓▓                                                   │             │
│   │ 800 ▓▓▓▓ ▒▒▒▒                                                │             │
│   │   ┊ ▓▓▓▓ ▒▒▒▒ ▓▓▓▓                                          │             │
│   │ 400 ▓▓▓▓ ▒▒▒▒ ▓▓▓▓ ▒▒▒▒                                     │             │
│   │   ┊ ▓▓▓▓ ▒▒▒▒ ▓▓▓▓ ▒▒▒▒                                     │             │
│   │   └───────────────────────                                   │             │
│   │     Wk1   Wk2   Wk3   Wk4                                    │             │
│   │     ▓ pick/pack    ▒ storage    █ returns                    │             │
│   └─────────────────────────────────────────────────────────────┘             │
│                                                                              │
│   Fee history                                                                │
│   Date       Type            SKU          Order      Amount   Status         │
│   ─────────────────────────────────────────────────────────────────────────  │
│   12 May     pick_pack       WIDGET-RED   TK-12345   £1.20    settled        │
│   12 May     storage_monthly all          —          £620.00  settled        │
│   12 May     return_proc     WIDGET-BLU   TK-12100    £0.80   pending        │
│                                                                              │
│   Reimbursement claims                                                       │
│   Date       Reason                Order       Amount   Status               │
│   ─────────────────────────────────────────────────────────────────────────  │
│   10 May     Damaged in FC         TK-11999   £15.00   ✓ paid 11 May        │
│   12 May     Lost in FC            TK-12001    £8.00   pending               │
│                                                                              │
│   ⚠ Discrepancies                                                            │
│   ─────────────────────────────────────────────────────────────────────────  │
│   3 orders charged for storage but had 0 units in inventory snapshot.        │
│   [Review]                                                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Triggers

| Event | Result |
|---|---|
| 03:15 daily statement cron | pull yesterday's statement, parse each line, insert `fbt_fees` rows |
| Each fee line | publish `tiktok_fbt.to.wms.fee_incurred` → core-service `marketplace_charges` |
| Each reimbursement line | publish `tiktok_fbt.to.wms.reimbursement_received` → core-service `marketplace_reimbursements` |
| Discrepancy detector cron 05:00 | flags suspicious lines for merchant review |
| Merchant clicks [Submit dispute] | publishes `wms.to.tiktok_fbt.dispute_submit` if TikTok exposes an API; otherwise opens TikTok Seller Center dispute page |

### Invisible side-effects

- Each fee row is matched against an order in `fbt_orders` by `tiktok_order_id` when possible. Match-rate is a metric we track.
- Storage fees aggregate at SKU + month level; no per-order match.
- Raw statement JSON is preserved in `external_metadata` for audit.

---

## Entity 7 — `fbt_activity_logs`

### Why it exists

A merchant-readable diary of everything our service did for a given account. When something looks wrong, this is the first place support and the merchant look. Each entry is a single sentence in plain English plus a structured payload behind it.

### Where the merchant sees it

- On each account card: a "View activity" link opens a side panel.
- URL: `/clients/tiktok-fbt/accounts/[id]/activity`

### View: activity log panel

```
        ┌──────────────────────────────────────────────────────────┐
        │  Activity — Acme Ltd UK FBT                       [X]    │
        ├──────────────────────────────────────────────────────────┤
        │                                                            │
        │   Filter: [ All events ▾ ]                                 │
        │                                                            │
        │   12 May 09:31  ✓ Synced 4 new orders, 0 updates           │
        │   12 May 09:26  ✓ Inventory synced — 147 SKUs              │
        │   12 May 09:14  ⓘ Webhook ORDER_STATUS_CHANGE processed    │
        │                    (TK-12345)                              │
        │   12 May 08:30  ⓘ Webhook ORDER_STATUS_CHANGE processed    │
        │                    (TK-12344)                              │
        │   12 May 05:00  ✓ Token refreshed (next expiry 19 May)     │
        │   11 May 23:11  ⚠ Rate-limited by TikTok, backed off 3s    │
        │   11 May 16:42  ⓘ Inbound shipment INB-2026-0001 received  │
        │                    at FBT UK Bedford                       │
        │   11 May 09:14  ✓ Inbound shipment INB-2026-0001 shipped   │
        │   10 May 14:02  ✓ Inbound plan INB-2026-0001 confirmed     │
        │   10 May 14:00  ✓ Inbound plan INB-2026-0001 submitted     │
        │   ...                                                       │
        │                                                            │
        │                                              [Load more]   │
        └──────────────────────────────────────────────────────────┘
```

### What gets logged (non-exhaustive)

| Event | Sample log line |
|---|---|
| Account connected | ✓ Account connected |
| Account disconnected | ⨯ Account disconnected |
| Inventory sync ok | ✓ Inventory synced — 147 SKUs |
| Inventory sync failed | ⚠ Inventory sync failed: <reason> |
| Order sync ok | ✓ Synced 4 new orders, 0 updates |
| Webhook processed | ⓘ Webhook <event_type> processed (<entity_id>) |
| Webhook duplicate | ⓘ Webhook <event_type> duplicate suppressed |
| Token refreshed | ✓ Token refreshed |
| Rate-limited | ⚠ Rate-limited by TikTok, backed off Ns |
| Inbound submitted | ✓ Inbound plan <id> submitted |
| Inbound received | ⓘ Inbound shipment <id> received |
| Fee ingested | ⓘ <N> fee lines ingested from statement <date> |

### Triggers

Every service operation that affects the account writes a log row. There is no separate UI to create or edit log entries — they are immutable.

### Invisible side-effects

- Pruned to last 30 days for normal viewing.
- Older logs retained for 1 year in a cheaper archive table (or as JSON to S3 if the table grows too large).
- Logs are **per-account**, not global — multi-merchant isolation is enforced via `client_id`.

---

## Entity 8 — `fbt_webhook_events`

### Why it exists

TikTok pushes us webhooks at unpredictable moments — order status changes, package updates, return requests. Webhooks can be retried by TikTok (a delivery they think we missed), so the same event sometimes arrives twice. This table is our **idempotency guard**: every incoming webhook gets a `event_key` (event_type + shop_id + event_id), and if we've seen that key before we acknowledge it and skip processing.

The merchant doesn't see this table directly. It exists for engineers and support staff to debug "why didn't this order arrive?" or "why is the data out of date?"

### Where engineers / support see it

- Internal admin only: `/clients/tiktok-fbt/admin/webhooks` (not in the standard sidebar; available via super-admin / system-admin roles)
- Also: every entry that did something visible is **mirrored** as a friendly line in `fbt_activity_logs`

### View: admin webhook log (engineer-facing)

URL: `/clients/tiktok-fbt/admin/webhooks` (super-admin only)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TikTok FBT — Webhook events (last 24h)                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   [Type: All ▾]  [Status: All ▾]  [Account: All ▾]                           │
│                                                                              │
│   Received   Event                          Status        Latency  Account   │
│   ─────────────────────────────────────────────────────────────────────────  │
│   09:31:14  ORDER_STATUS_CHANGE             ✓ processed   142ms    UK FBT    │
│   09:30:42  PACKAGE_UPDATE                  ✓ processed    98ms    UK FBT    │
│   09:30:41  ORDER_STATUS_CHANGE             ◌ duplicate    14ms    UK FBT    │
│   09:14:02  ORDER_STATUS_CHANGE             ✓ processed   201ms    UK FBT    │
│   08:55:11  FBT_RETURN_RECEIVED             ✓ processed   245ms    UK FBT    │
│   08:54:18  PRODUCT_STATUS_CHANGE           ⚠ failed      431ms    UK FBT    │
│                ↳ retry attempt 1 scheduled 09:00:00                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

Clicking a row reveals the full JSON payload, our HMAC signature verification result, processing latency breakdown, and any error.

### Triggers (every life of a webhook)

```
   TikTok POSTs to /api/tiktok-fbt/webhooks
       │
       ▼
   1. WebhookSignatureGuard verifies HMAC          ──fail──► 401 (silent retry by TikTok)
       │ ok
       ▼
   2. Compute event_key (event_type + shop_id + event_id)
       │
       ▼
   3. INSERT INTO fbt_webhook_events (...)
        ON CONFLICT (event_key) DO NOTHING        ──conflict──► return 200 status='duplicate'
       │ inserted
       ▼
   4. Hand off to async processor; return 200 immediately
       │
       ▼
   5. Processor fetches full payload from TikTok if needed,
      updates fbt_orders / fbt_returns / etc, publishes
      RabbitMQ event, marks fbt_webhook_events.status='processed'
```

### Invisible side-effects

- Retention: 7 days. Long enough to catch TikTok retry storms; short enough to keep the table small.
- A daily metric tracks `duplicate_total / received_total`. If this exceeds 50%, an alert fires (something is wrong with our ack flow).

---

## Brief note: `fbt_returns` (Phase 3)

Not in the eight you listed, but I include it for completeness because returns are part of the FBT lifecycle. The shape is essentially the same as fbt_orders: webhook in → row created → status flows → reimbursement may follow. UI lives under the existing WMS returns module extended with FBT metadata (return destination = `marketplace_fbt_warehouse`), not as a new top-level page.

---

## Scenario A — First-time setup

A merchant just signed up for TikTok FBT and wants to start mirroring data in WMS. Monday morning, 09:00.

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  Time   What the merchant does                  What the system does                     │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ 09:00  Open WMS → sidebar shows TikTok FBT      Feature flag tiktok_fbt is enabled       │
│        group (because their subscription tier   for this client. Sidebar item visible.   │
│        includes it).                                                                      │
│                                                                                            │
│ 09:01  Click "TikTok FBT > Accounts".           Empty state shown.                       │
│                                                                                            │
│ 09:02  Click [+ Connect FBT shop].              Modal opens.                              │
│                                                                                            │
│ 09:02  Pick "United Kingdom (GB)".              -                                         │
│        Click [Continue to TikTok →].                                                      │
│                                                                                            │
│ 09:02  Browser redirects to TikTok.             Service generates OAuth state, stores    │
│                                                  it in session.                           │
│                                                                                            │
│ 09:03  Login (or already logged in).            -                                         │
│        Authorise our app on the FBT shop.                                                 │
│                                                                                            │
│ 09:03  Browser redirects back to                Service exchanges code for tokens,       │
│        /api/tiktok-fbt/auth/callback?code=…     encrypts, inserts fbt_accounts row.      │
│                                                  Publishes account_connected event.       │
│                                                  Core-service creates virtual warehouse  │
│                                                  "TikTok FBT UK".                         │
│                                                                                            │
│ 09:03  Sees interstitial: link to seller?       -                                         │
│        Picks "Acme Ltd — Seller (142)".                                                  │
│        Click [Save link].                                                                 │
│                                                                                            │
│ 09:04  Lands on accounts page, sees             First inventory poll is queued for       │
│        the connected card.                       09:15 (next quarter-hour).               │
│                                                  First order poll is queued for 09:05.   │
│                                                                                            │
│ 09:05  Merchant clicks "TikTok FBT > Orders".   Order cron fires, pulls 0 orders         │
│        Sees "No orders yet — first sync         (new shop, no history yet). Activity log │
│        complete."                                gets "Synced 0 new orders, 0 updates".  │
│                                                                                            │
│ 09:15  Merchant refreshes Inventory page.       Inventory cron fires. Pulls 47 SKUs.    │
│        Sees the 47 SKUs populate.               Snapshot inserted. Activity log gets     │
│                                                  "Inventory synced — 47 SKUs".           │
│                                                                                            │
│ 09:16  Looks at Dashboard.                      Numbers reflect: 1 connected account,    │
│                                                  47 active SKUs, 0 orders today.         │
│        Done. Setup complete in 16 minutes.                                                │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Scenario B — Restocking a low SKU

The merchant has been running for a few weeks. Inventory has drained on a popular SKU.

```
Tuesday 11:00.

  1. Dashboard tile "Stock health" shows: ● Red: 3
     Merchant clicks it.

  2. Lands on Inventory filtered to health=red. Sees 3 rows:
        WIDGET-GRN-L     42 fulfillable    6.5 days cover    ● Red
        STICKER-GLAM     12 fulfillable    8.0 days cover    ● Red
        PLUSH-CAT-S       5 fulfillable    2.5 days cover    ● Red

  3. Selects all three (☐ → ☑ in row checkboxes).
     Clicks [Mark for replenishment].

  4. Opens the inbound wizard with those 3 SKUs pre-filled.
     Adjusts quantities (500 / 200 / 300).
     Picks source warehouse (Manchester DC) and destination FC (Bedford).
     Reviews. Clicks [Submit plan].

  5. INB-2026-0007 created. Status: submitted.
     Plan ID PLN-99887 returned by TikTok.
     Activity log: "Inbound plan INB-2026-0007 submitted (3 SKUs, 1000 units)."

  6. Merchant clicks [Print labels]. PDF downloads (3 pages).

  7. Warehouse staff packs the 5 boxes, drops with DPD.
     Merchant comes back, enters tracking DPD550XYZ, clicks [Mark as shipped].
     Status → in_transit.

  8. Tuesday afternoon: DPD delivers. TikTok receives.
     Wednesday morning: webhook FBT_INBOUND_RECEIVED arrives.
     Status → partially_received, then within an hour → received.
     Activity log: "Inbound shipment INB-2026-0007 received at FBT UK Bedford."

  9. Wednesday 09:15: next inventory poll.
     fbt_inventory snapshot now shows:
        WIDGET-GRN-L     537 fulfillable    (was 42, gained 495, lost 0 to sales overnight)
        STICKER-GLAM     205 fulfillable
        PLUSH-CAT-S      301 fulfillable
     Health colour flips to green.

 10. Dashboard reorder alert clears.
     Merchant relaxes.
```

---

## Scenario C — A customer places an order

This is the most common flow once the merchant is set up. The merchant does **nothing** in this scenario except watch it happen.

```
Buyer Jane buys 2× WIDGET-RED-L on TikTok at 14:20.

  14:20:03  TikTok side: order TK-12999 created. Inventory deducted on TikTok side.
  14:20:05  TikTok side: webhook ORDER_STATUS_CHANGE fired toward our /api/tiktok-fbt/webhooks.

  14:20:06  Our gateway routes to tiktok-fbt-microservice.
            WebhookSignatureGuard verifies HMAC. Pass.
            fbt_webhook_events row inserted with event_key.
            Worker is enqueued.

  14:20:06  Service responds 200 to TikTok (do not wait for processing).

  14:20:07  Worker fetches full order detail from TikTok API.
            Upserts fbt_orders (status=AWAITING_SHIPMENT) and fbt_order_items.
            Publishes RabbitMQ tiktok_fbt.to.wms.order_created.
            Activity log: "Webhook ORDER_STATUS_CHANGE processed (TK-12999)."

  14:20:08  core-service consumes. Resolves variation_listings (WIDGET-RED-L → variation #4521).
            Inserts orders row with createdVia='tiktok_fbt', wms_message_id=…
            Inserts order_products, order_fulfillments(type='fbt', warehouse=fbt_uk).
            Decrements virtual warehouse inventory by 2.
            Acks message.

  14:20:09  Realtime: socket.io pushes 'order_created' to any open WMS browser tab.
            Order appears in /clients/order and /clients/tiktok-fbt/orders.

  Picker queue (/clients/order/awaiting-pick): UNCHANGED. TK-12999 is NOT there.

  18:15     TikTok ships the order. Webhook PACKAGE_UPDATE arrives with tracking AB789.
            fbt_orders.tracking_number, shipped_at populated.
            order updates in core-service. Order row shows "● In transit  AB789↗".

  Day + 1  TikTok delivers. PACKAGE_UPDATE again. Status → DELIVERED.
            Order row in WMS shows "● Delivered".

The merchant logs in next morning, sees one new sale and one tracking update.
Total merchant effort: 0 minutes.
```

---

## Scenario D — Reconciling monthly fees (Phase 3)

End of month. Merchant wants to know exactly what TikTok charged them.

```
  1. Merchant opens "TikTok FBT > Fees & Reimbursements".

  2. Sets date range to "April 2026".

  3. Sees:
        Total fees:        £2,840
        Pick & pack:       £1,720   (1,433 orders × £1.20 average)
        Storage:             £620
        Reimbursed:          £312

  4. Drills into "Storage £620":
        - 47 SKUs charged storage
        - Sorted by cost, top item is WIDGET-RED-XL at £142 for the month
          (slow mover; merchant decides to drop the price or run a promo)

  5. Sees the discrepancy alert: "3 orders charged for storage but had 0 units in
     inventory snapshot." Clicks [Review].
        - 3 rows shown, each with the raw statement JSON viewable
        - Likely a timing edge case (units sold same day as month-end storage tally)
        - Merchant clicks [Submit dispute] on one of them
        - WMS publishes wms.to.tiktok_fbt.dispute_submit
        - Either we call TikTok's dispute API (if available) or surface a deep
          link to TikTok Seller Center's dispute page

  6. Sees the reimbursement claim from 10 May: ✓ paid 11 May £15 (damaged in FC).
     This was auto-created when the inbound damage was detected (Scenario from earlier).

  7. Clicks [Export CSV]. Sends to their accountant.
```

---

## Scenario E — Handling a damaged-in-FC return (Phase 3)

A buyer returns a damaged item to TikTok's FC. TikTok marks it unsellable and credits the merchant.

```
  Day 0   Buyer ships item back to TikTok (not to merchant).

  Day 3   Webhook FBT_RETURN_RECEIVED arrives.
          fbt_returns row created. condition='damaged'. reimbursement_eligible=true.
          Activity log: "Return TK-RET-9988 received, condition damaged."
          
          tiktok_fbt.to.wms.return_received published.
          core-service inserts return_requests row with marketplace='tiktok_fbt',
          return_destination='marketplace_fbt_warehouse'.

  Day 4   In-app notification appears to merchant: "1 FBT return received (damaged)."

          Merchant opens the central returns page (existing WMS UI) and sees the row
          tagged with the "Fulfilled by TikTok" badge.
          Drawer shows reason, condition, photo (if provided), and "reimbursement
          status: pending".

  Day 7   TikTok's next daily statement includes a reimbursement credit of £15
          for the damaged unit.
          fbt_fees row inserted (fee_type=reimbursement, amount=£15).
          fbt_returns row updated to status='reimbursed'.
          
          tiktok_fbt.to.wms.reimbursement_received published.
          core-service inserts marketplace_reimbursements row (status=paid).

  Day 7   Merchant opens Fees & Reimbursements page, sees the £15 credit listed.
          Done.
```

---

## State machine: FBT order journey

One-page reference for engineers and support. Print this and pin it.

```
                                ┌──────────────────┐
       webhook arrives          │                  │
       ──────────────────────►  │ AWAITING_SHIPMENT│
                                │                  │
                                └────────┬─────────┘
                                         │
            ┌────────────────────────────┼───────────────────────────┐
            │                            │                           │
TikTok ships│                  buyer cancels (rare)         TikTok    │ (terminal,
            ▼                            ▼                  cancels   │  inventory
   ┌──────────────────┐         ┌──────────────────┐                  │  restored)
   │                  │         │                  │                  ▼
   │   IN_TRANSIT     │ ──────► │   CANCELLED      │ ◄────────────────┘
   │                  │         │                  │
   └────────┬─────────┘         └──────────────────┘
            │
            │ carrier confirms delivery
            ▼
   ┌──────────────────┐
   │                  │
   │   DELIVERED      │  (terminal — buyer may still initiate a return)
   │                  │
   └──────────────────┘
```

Pick/pack is **never** part of this journey. The merchant only watches; they do not act.

---

## State machine: Inbound shipment journey

```
   ┌────────┐  [Submit plan]  ┌────────────┐  TikTok confirms  ┌────────────┐
   │ DRAFT  │ ──────────────► │ SUBMITTED  │ ────────────────► │ CONFIRMED  │
   └───┬────┘                  └─────┬──────┘                   └─────┬──────┘
       │                             │                                │
       │      [Cancel] (any time before in_transit)                   │
       │ ───────────────────────────────────────────────────►  CANCELLED
       │                                                              │
       │                                                  [Print labels]
       │                                                              ▼
       │                                                  ┌──────────────────┐
       │                                                  │ LABEL_GENERATED  │
       │                                                  └─────────┬────────┘
       │                                                            │
       │                                              [Mark as shipped]
       │                                                            ▼
       │                                                  ┌──────────────────┐
       │                                                  │   IN_TRANSIT     │
       │                                                  └─────────┬────────┘
       │                                                            │
       │                                    first units arrive at FC│
       │                                                            ▼
       │                                                  ┌──────────────────────┐
       │                                                  │ PARTIALLY_RECEIVED   │
       │                                                  └─────────┬────────────┘
       │                                                            │
       │                                          all units received OR shortage finalised
       │                                                            ▼
       │                                                  ┌──────────────────┐
       └─────────────────────────────────────────────►   │    RECEIVED      │  (terminal)
                                                          └──────────────────┘

   Discrepancy on receipt (damaged or short) → creates a Phase-3 reimbursement claim row.
```

---

## Things that are intentionally NOT shown to the merchant

For clarity, here is what the merchant does **not** see in their FBT pages:

- TikTok's internal warehouse routing (which FC stored each unit)
- Internal sync timings, queue depths, retry counts (those live in admin-only views)
- The raw JSON of webhook payloads (admin-only)
- Encrypted token values (never decryptable to UI)
- Cross-merchant data of any kind (multi-tenant isolation enforced everywhere)
- The full `variation_listings` table — they see the **effect** (FBT inventory column) but not the row

Showing too much would be noise; the goal is "tell the merchant what they need to act on, and nothing else."

---

## Next steps after sign-off

This document is **the brief for design**. Once you (or whoever) signs off on the structure here:

1. Hand the mockups to whoever owns the Figma file. They translate the ASCII into pixel-perfect screens — but the **information architecture** in this document is binding. Don't reshape what data lives on what page.
2. Use this document as the QA script for Phase 1 acceptance. Every screen state described here should match what gets built.
3. Sales/CS enablement: this is the doc to walk pilot merchants through.
4. When TikTok's actual API payloads come back in Phase 1 week 1 ([03-api-research-checklist.md](./03-api-research-checklist.md)) and they differ from assumptions, revise the affected sections here first, then code.
