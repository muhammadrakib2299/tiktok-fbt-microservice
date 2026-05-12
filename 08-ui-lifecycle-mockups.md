# 08 — UI Visualization & Full Lifecycle Mockup

> Written for a non-coder. Reflects the **submodule architecture** — FBT lives inside the existing tiktok-microservice. Each TikTok shop has two independent fulfilment-mode toggles (FBM + FBT) on its account card; the sidebar renders one of three views based on aggregated per-account flags.
>
> For the visual rendered version, open `mockup.html` in any browser.

---

## Table of contents

- [How to read this document](#how-to-read-this-document)
- [The big picture](#the-big-picture)
- [Step 1: existing TikTok account (no change to connect flow)](#step-1-existing-tiktok-account)
- [Step 2: the Enable FBT toggle](#step-2-the-enable-fbt-toggle)
- [Step 3: conditional sidebar — three views](#step-3-conditional-sidebar--three-views)
- [Step 4: FBT Dashboard](#step-4-fbt-dashboard)
- [Step 5: FBT Inventory](#step-5-fbt-inventory)
- [Step 6: FBT Inbound shipments (Phase 2)](#step-6-fbt-inbound-shipments)
- [Step 7: FBT Orders — dedicated page](#step-7-fbt-orders-dedicated-page)
- [Step 8: fees & reimbursements (Phase 3)](#step-8-fees-and-reimbursements)
- [Day-in-the-life scenarios](#day-in-the-life-scenarios)

---

## How to read this document

ASCII mockups using:

```
┌──────────────────────────────┐
│  Title bar / section label   │
├──────────────────────────────┤
│ [Button]   ← clickable        │
│ ☐ checkbox   ▼ dropdown       │
│ ● green   ● amber   ● red     │
│ ⚡ FBT badge                  │
└──────────────────────────────┘
```

URLs above every mockup tell you which page the merchant is on. FBT pages live under `/clients/tiktok/fbt/*`.

---

## The big picture

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│  1. Existing TikTok account (fbm_enabled=true, fbt_enabled=false by default)                  │
│     ↓                                                                                          │
│  2. Fulfilment modes panel on account card — two independent toggles (FBM + FBT)             │
│     ↓                                                                                          │
│  3. Sidebar resolves to one of three views based on aggregated flags across the client's     │
│     accounts:  General-only  ·  FBT-only  ·  Hybrid                                          │
│     ↓                                                                                          │
│  4-8. FBT Dashboard, Inventory, Inbound, Orders (dedicated page), Fees                       │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1 — Existing TikTok account

No change to the connect flow. Newly connected shops default to `fbm_enabled=true`, `fbt_enabled=false` — the "Seller-only" state, which is today's behaviour.

URL: `/clients/tiktok/accounts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › TikTok Shop › Accounts                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   TikTok Accounts                                  [+ Connect TikTok shop]  │
│                                                                              │
│  ┌─ Acme Ltd — UK   [Seller-only] ───────────────────────────────────┐    │
│  │  Shop ID 7290xxxxxxx · Region GB · Status ● Connected                │    │
│  │                                                                       │    │
│  │  Sync orders: ●● Active   Sync inventory: ●● Active   Token: ✓ Valid  │    │
│  │                                                                       │    │
│  │  ┌── Fulfilment modes ─────────────────────────────────────────┐    │    │
│  │  │  [FBM]  Seller-fulfilled (FBM)        ● Active     [●━] (locked) │ │    │
│  │  │         1,180 listings · orders flow into picker queue.       │ │    │
│  │  │  ─────────────────────────────────────────────────────────── │ │    │
│  │  │  [⚡]   Fulfilled by TikTok (FBT)     ● Off        [○━]      │ │    │
│  │  │         Enable FBT to mirror TikTok-fulfilled inventory.      │ │    │
│  │  │         Your shop must be enrolled in FBT on Seller Center.   │ │    │
│  │  │         › Learn more                                          │ │    │
│  │  └─────────────────────────────────────────────────────────────┘    │    │
│  │                                                                       │    │
│  │       [↻ Sync now]  [View activity]              [Disconnect]         │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

The FBM toggle shows as `(locked)` because at least one mode must stay ON (`chk_accounts_any_mode`). It unlocks the moment FBT flips to ON.

---

## Step 2 — The Enable FBT toggle

Merchant flips the switch. Confirmation modal appears, then verification call to TikTok.

```
        ┌────────────────────────────────────────────────────────┐
        │  Verify FBT eligibility for Acme Ltd — UK?     [X]    │
        ├────────────────────────────────────────────────────────┤
        │                                                          │
        │   We'll check with TikTok that this shop is enrolled in │
        │   the FBT programme. If yes, FBT data will start         │
        │   mirroring within 15 minutes.                           │
        │                                                          │
        │                              [Cancel]  [Verify and enable →] │
        └────────────────────────────────────────────────────────┘
```

**Eligible path:** verification succeeds → toggle flips green → FBT panel expands to show capabilities → sidebar refreshes within 1-2 seconds.

**Not eligible path:**

```
        ┌────────────────────────────────────────────────────────┐
        │  ⨯ This shop is not enrolled in FBT             [X]   │
        ├────────────────────────────────────────────────────────┤
        │                                                          │
        │   We verified with TikTok and this shop is not enrolled │
        │   in the Fulfilled-by-TikTok programme.                 │
        │                                                          │
        │   → Apply for FBT on Seller Center                       │
        │                                                          │
        │                                          [Got it]       │
        └────────────────────────────────────────────────────────┘
```

After successful enable, the card becomes a **Hybrid shop** — both toggles ON, neither locked:

```
│  ┌── Fulfilment modes ─────────────────────────────────────────┐
│  │  [FBM]  Seller-fulfilled (FBM)        ● Active     [●━]      │
│  │         3,420 listings · orders flow into picker queue.       │
│  │  ─────────────────────────────────────────────────────────── │
│  │  [⚡]   Fulfilled by TikTok (FBT)     ● Enabled    [●━]      │
│  │         FBT verified 12 May 09:14                              │
│  │         Capabilities: inventory · orders · inbound · fees     │
│  │         Last FBT inventory sync: 7 min ago                    │
│  │                                                                │
│  │         [↻ Sync FBT now] [Re-verify capabilities] [Disable FBT]│
│  └────────────────────────────────────────────────────────────────┘
```

To convert to FBT-only, the merchant flips the FBM toggle OFF. That triggers a confirmation modal ("All future TikTok orders for this shop will be handled by TikTok FBT…") and, on confirm, sets `fbm_enabled=false`. The FBT toggle is now `(locked)` because at least one mode must stay ON.

### Edge state — admin-paused FBT publishing

Independent of the user-facing toggles, an admin-only column `accounts.publish_fbt_to_wms` (default TRUE) can be flipped to FALSE by engineers during a downstream consumer outage. The merchant's FBT toggle stays ON, but the canonical `canPublishFbt()` guard short-circuits every publisher, so no RabbitMQ messages flow. The Accounts card surfaces this with an amber banner:

```
┌────────────────────────────────────────────────────────────────────┐
│  ⚠ FBT publishing temporarily paused                                │
│                                                                      │
│  Your shop is enabled, but our team has paused data flow to your    │
│  WMS while we resolve a downstream issue. New FBT orders and        │
│  inventory updates will resume once paused is lifted — no action    │
│  needed from you. Contact support if this banner persists > 24h.    │
└────────────────────────────────────────────────────────────────────┘
```

The merchant cannot dismiss the banner; it disappears the moment ops re-enables publishing. The `[↻ Sync FBT now]` and `[Re-verify capabilities]` buttons grey out with a tooltip. The toggles themselves remain operable — the merchant can still disable FBT entirely if they want. Pattern adopted from Amazon FBA's `publish_fba_to_wms` after a 421-order fail-open incident (see `amazon-microservice/LESSONS.md` line 6).

---

## Step 3 — Conditional sidebar — three views

The "TikTok Shop" parent renders one of three layouts based on aggregated per-account flags. The **Accounts** entry is always present in every view because that's where both fulfilment-mode toggles live.

### View 1 — General-only (today's default, FBM-on / FBT-off everywhere)

```
WMS360
├ Catalogue
├ Orders
├ Warehouses
└ MARKETPLACES
    ├ eBay
    ├ Shopify
    └ TikTok Shop
        ├ Accounts
        ├ Active products
        ├ Pending products
        └ Deactivated
```

### View 2 — FBT-only (every account converted; no residual FBM orders)

```
WMS360
├ Catalogue
├ Orders
├ Warehouses
└ MARKETPLACES
    ├ eBay
    ├ Shopify
    └ TikTok Shop
        ├ Accounts                ← always present (holds the toggles)
        ├ ── FBT ──              ← divider
        ├ FBT Dashboard
        ├ FBT Inventory
        ├ FBT Orders
        ├ FBT Inbound shipments    (Phase 2)
        └ FBT Fees                 (Phase 3)
```

The General product listing pages are hidden because they have nothing to show — this client manages no seller-fulfilled TikTok listings.

### View 3 — Hybrid (at least one account has each flag, or one account has both)

```
WMS360
├ Catalogue
├ Orders
├ Warehouses
└ MARKETPLACES
    ├ eBay
    ├ Shopify
    └ TikTok Shop
        ├ Accounts
        ├ ── General ──          ← divider (only appears when both groups render)
        ├ Active products
        ├ Pending products
        ├ Deactivated
        ├ ── FBT ──              ← divider
        ├ FBT Dashboard
        ├ FBT Inventory
        ├ FBT Orders
        ├ FBT Inbound shipments    (Phase 2)
        └ FBT Fees                 (Phase 3)
```

The change is instant after any toggle flips. No app reload required — the sidebar reads from the accounts Zustand store, which re-aggregates `fbm_enabled` and `fbt_enabled` on every refresh.

**Residual-data safeguard:** when a merchant flips an account to FBT-only, the General group stays visible as long as historical FBM orders still exist on `tiktok_orders` (`fulfilment_mode='fbm'`). A precomputed boolean `clientHasHistoricalFbmOrders` rides along with the accounts payload, so the merchant never loses their path back to legacy orders and returns.

---

## Step 4 — FBT Dashboard

URL: `/clients/tiktok/fbt/dashboard`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › TikTok Shop › FBT › Dashboard                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   FBT Dashboard       Live inventory · refreshed every 15 min · last 7m ago│
│                                                                              │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│   │ FBT accounts │ │ FBT SKUs     │ │ Units in     │ │ Orders today │       │
│   │ enabled      │ │ active       │ │ FBT FCs      │ │              │       │
│   │     2        │ │    147       │ │   3,840      │ │     58       │       │
│   └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘       │
│                                                                              │
│   ┌─ Stock health ────────────┐  ┌─ FBT sales · last 30 days ─────────┐    │
│   │ ● Green: 124              │  │ [line chart]                        │    │
│   │ ● Amber:  18              │  │                                      │    │
│   │ ● Red:     5  →           │  │                                      │    │
│   └────────────────────────────┘  └─────────────────────────────────────┘    │
│                                                                              │
│   ┌─ Reorder alerts · stock < 14 days ───────────────────────────────┐    │
│   │  SKU             Fulfillable    Days cover    Health    Action    │    │
│   │  WIDGET-GRN-L         42           6.5      ● Red    [Plan]      │    │
│   │  STICKER-GLAM         12           8.0      ● Red    [Plan]      │    │
│   │  PLUSH-CAT-S           5           2.5      ● Red    [Plan]      │    │
│   └────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 5 — FBT Inventory

URL: `/clients/tiktok/fbt/inventory`

Same as before but with the **WMS own stock** column joined from the merchant's own warehouses.

```
FBT Inventory                                  Last synced 7m ago [↻ Sync now]

[Region: UK ▾] [Account: All ▾] [Health: All ▾] [🔍 Search…]

┌─────┬───────┬─────────────┬──────┬──────┬─────┬───────┬─────────┬─────────┐
│  ☐  │ Image │ SKU / Name  │Fulfl │Inbnd │Resv │ Days  │ Health  │ WMS own │
├─────┼───────┼─────────────┼──────┼──────┼─────┼───────┼─────────┼─────────┤
│  ☐  │ [img] │ WIDGET-GRN-L│  42  │ 120  │  3  │  6.5  │ ● Red   │   85    │
│  ☐  │ [img] │ WIDGET-BLU-L│ 200  │   0  │  8  │ 41.7  │ ● Green │  120    │
│  ☐  │ [img] │ GIZMO-S     │ 102  │   0  │  2  │ 20.4  │ ● Amber │  410    │
└─────┴───────┴─────────────┴──────┴──────┴─────┴───────┴─────────┴─────────┘
```

---

## Step 6 — FBT Inbound shipments (Phase 2)

URL: `/clients/tiktok/fbt/inbound`

List view and 4-step wizard. Wizard steps are unchanged from the previous mockup — the sidebar context just looks different (TikTok Shop parent with FBT children visible).

```
Inbound Shipments                                          [+ Create new]

[Status: All ▾]  [Region ▾]  [🔍 Search…]

Shipment      Created    Destination       Units  Status        Tracking
INB-2026-0003 12 May    FBT UK Bedford     700   ● Label gen.    —
INB-2026-0002 11 May    FBT UK Manchester   80   ● Submitted     —
INB-2026-0001 10 May    FBT UK Bedford    150   ● In transit    DPD123 ↗
```

Wizard stepper:

```
Products ─► Destination FC ─► Review ─► Labels & ship
●●            ○                  ○         ○
```

---

## Step 7 — FBT Orders — dedicated page

**FBT orders live exclusively on their own page.** The central `/clients/order` page is unchanged — it shows all other channels but FBT orders never appear there. Sidebar position: between FBT Inventory and FBT Inbound shipments.

### The FBT Orders page

URL: `/clients/tiktok/fbt/orders`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › TikTok Shop › FBT › Orders                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  FBT Orders                          58 today · 4 awaiting · 52 in transit  │
│                                                                              │
│  [Status: All ▾] [Account: All ▾] [Date: Last 30 days ▾]  [🔍 Search…]       │
│  [↻ Sync now]  [Export CSV]                                                  │
│                                                                              │
│  Order #          Date          Items  Total    Status         Tracking      │
│  ─────────────────────────────────────────────────────────────────────────── │
│  TK-12345 ⚡FBT   12 May 09:14  2     £42.00   ● In transit    AB123 ↗      │
│  TK-12344 ⚡FBT   12 May 08:30  1     £15.00   ● Awaiting      —             │
│  TK-12343 ⚡FBT   11 May 19:01  3     £67.50   ● Delivered     AB121 ↗      │
│  TK-12342 ⚡FBT   11 May 15:22  1     £15.00   ⨯ Cancelled     —             │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ You do not need to pick or pack these orders. TikTok fulfils them.  │    │
│  │ This page is read-only and informational.                            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Central `/clients/order` page — FBT is filtered out

The central order page query adds `WHERE created_via != 'tiktok_fbt'`. Operators see only orders they need to action.

URL: `/clients/order`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Home › Orders                                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│  Orders                                  All channels (non-FBT) · 189 today │
│                                                                              │
│  Order #         Channel              Items  Total    Status                 │
│  ─────────────────────────────────────────────────────────────────────────── │
│  EB-7788         eBay                  3    £28.00   ● Awaiting pick         │
│  SH-2200         Shopify               1    £18.00   ● Awaiting pick         │
│  TK-12333        TikTok (seller)       2    £30.00   ● Awaiting pick         │
│  AM-5511         Amazon                1    £22.00   ● Awaiting pick         │
│                                                                              │
│  ⓘ FBT orders such as TK-12345 are not on this page — they live exclusively  │
│     at TikTok Shop › FBT › Orders.                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Picker queue — also FBT-free

URL: `/clients/order/awaiting-pick`

```
Orders awaiting pick

Order #     Channel        Items  Warehouse
─────────────────────────────────────────────────────────
EB-7788     eBay            3    Manchester DC
SH-2200     Shopify         1    London DC
TK-12333    TikTok (seller) 2    Manchester DC

ⓘ Already excluded via fulfillment_type != 'fbt'. Plus FBT orders never make it
   to the central /clients/order page either — defence in depth.
```

---

## Step 8 — Fees and reimbursements (Phase 3)

URL: `/clients/tiktok/fbt/fees`

```
Fees & Reimbursements                Last statement ingested: 11 May 03:15

[Range: Last 30 days ▾]  [Account ▾]  [Type ▾]                    [Export]

┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│ Total fees │ Pick & pack  │ Storage      │ Reimbursed   │
│  £2,840    │   £1,720     │   £620       │   +£312      │
└────────────┘ └────────────┘ └────────────┘ └────────────┘

⚠ Discrepancy alerts (3)
   3 orders charged for storage but had 0 units in inventory snapshot. [Review]

Fee history                              Reimbursement claims
Date    Type        SKU          Amount   Date    Reason          Order     Amount   Status
12 May  pick_pack   WIDGET-RED-L £1.20    10 May  Damaged in FC   TK-11999  +£15.00  ✓ paid
12 May  storage     all          £620.00  12 May  Lost in FC      TK-12001   +£8.00  pending
12 May  return_proc WIDGET-BLU    £0.80
```

---

## Day-in-the-life scenarios

### Scenario A — Merchant enables FBT for the first time

Starting state: shop is "Seller-only" (FBM on, FBT off — today's default). Ending state: shop is "Hybrid" (both on). Sidebar transitions from View 1 → View 3.

```
09:00  Merchant goes to /clients/tiktok/accounts.
09:01  Sees their existing UK shop card. Fulfilment modes panel shows:
         [FBM] ● Active  [●━] (locked — last enabled mode)
         [⚡]  ● Off     [○━]
09:02  Flips the FBT toggle ON. Modal appears: "Verify FBT eligibility?"
09:02  Clicks "Verify and enable →".
09:02  Backend hits TikTok /authorization/202309/shops, sees fbt_enrolled=true.
09:02  accounts.fbt_enabled set true. Activity log entry. Modal closes.
09:02  FBT row switches to ● Enabled. FBM row unlocks (no longer the last mode).
09:02  Sidebar refreshes from View 1 → View 3 (Hybrid) — FBT submodules appear.
09:02  Merchant clicks "FBT Dashboard". Empty state: "First sync in progress…"
09:17  15-min cron has run. Dashboard populates with 147 SKUs.
       Done. From off to fully synced in 17 minutes.
```

### Scenario A2 — Merchant converts a shop to FBT-only

Starting state: Hybrid shop (both on). Ending state: FBT-only (FBM off). Sidebar transitions from View 3 → View 2 *if* this is the client's only TikTok shop and there are no historical FBM orders.

```
14:00  Merchant goes to /clients/tiktok/accounts on the now-hybrid shop.
14:00  Flips the FBM toggle OFF.
14:00  Modal: "All future TikTok orders for this shop will be handled by TikTok FBT.
        Existing FBM orders in progress will complete normally. Continue?"
14:01  Clicks Confirm.
14:01  POST /api/tiktok/accounts/:id/fbm/disable
14:01  Backend checks: any open FBM orders? (warn-only by default)
14:01  UPDATE accounts SET fbm_enabled=false. Activity log entry.
14:01  FBM row switches to ● Disabled. FBT row becomes (locked).
14:01  Sidebar refreshes. If no other TikTok shop has FBM on and
        clientHasHistoricalFbmOrders is false → View 2 (FBT-only).
        Otherwise the General group stays visible.
```

### Scenario B — Customer places an FBT order

```
14:20  Buyer places order TK-12999 on TikTok. TikTok webhook fires.
14:20  Our webhook controller verifies HMAC, dispatcher sees fulfilment_type=FBT.
14:20  fbt-webhooks handler stores webhook event for idempotency, then
       calls orders.service to upsert. fulfilment_mode set to 'fbt'.
14:20  Publish tiktok_fbt.to.wms.order_created with full payload.
14:20  core-service consumes: resolve variation_listings, create Order with
       createdVia='tiktok_fbt', OrderFulfillment(type='fbt'), deduct virtual
       fbt_tiktok warehouse. NO picker assignment.
14:21  Order appears in /clients/order with ⚡ FBT badge.
       Order does NOT appear in /clients/order/awaiting-pick.
       Merchant glances, sees it, goes back to whatever they were doing.

18:15  TikTok ships. PACKAGE_UPDATE webhook arrives. Tracking AB789 updates.
Day+1  Delivery. Order shows ● Delivered.

Total merchant effort: 0 minutes.
```

### Scenario C — Merchant restocks (Phase 2)

```
11:00  Merchant sees red SKUs in FBT Dashboard reorder alerts.
11:01  Clicks "Plan inbound" on WIDGET-GRN-L row.
11:01  Wizard opens, step 1, that SKU pre-filled with suggested qty.
11:03  Adjusts to 500 units. Adds GIZMO-S at 200. Clicks Next.
11:04  Step 2: picks "Let TikTok decide" FC. Next.
11:05  Step 3: reviews. Total declared value £2,100. Clicks Submit.
11:05  Backend calls TikTok plan API. Plan ID returned. Shipment row created.
11:05  Step 4: PDF labels available. Downloads, prints.
11:30  Warehouse packs 5 boxes, hands to DPD.
11:35  Merchant returns, enters tracking. Status → in_transit.

Day+2  Webhook FBT_INBOUND_RECEIVED. Status → received.
Day+2  Next FBT inventory poll shows the new fulfillable units. Red turns green.
```

---

## What's intentionally NOT shown

- TikTok's internal warehouse routing
- Cron timings, queue depths, retry counts (admin views only)
- Webhook payload JSON (admin only)
- Encrypted token values
- Cross-merchant data
