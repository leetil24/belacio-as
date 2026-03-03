# GHL + WooCommerce Inventory Sync via n8n

## Updated Option A: n8n Implementation

---

### Scenario 1: GHL Invoice/Order → WooCommerce Stock Update

**Workflow structure:**

1. **Webhook Node (Trigger)**
   - n8n gives you a webhook URL → paste this into your GHL Workflow's Webhook Action
   - GHL fires this on `Invoice Paid` and `Order Submitted`

2. **HTTP Request Node — Fetch WooCommerce Product by SKU**
   - Method: `GET`
   - URL: `https://yoursite.com/wp-json/wc/v3/products?sku={{$json["sku"]}}`
   - Auth: Basic Auth using WooCommerce Consumer Key + Secret

3. **Set Node — Calculate New Stock**
   - `new_stock = {{ $json["stock_quantity"] }} - {{ $json["qty_sold"] }}`
   - Add a floor of `0` to prevent negative stock: `Math.max(0, stock - qty)`

4. **HTTP Request Node — Push Stock to WooCommerce**
   - Method: `PUT`
   - URL: `https://yoursite.com/wp-json/wc/v3/products/{{product_id}}`
   - Body:
   ```json
   {
     "manage_stock": true,
     "stock_quantity": "{{ new_stock }}"
   }
   ```

5. **HTTP Request Node — Update GHL Inventory** *(optional but keeps GHL count clean)*
   - Method: `POST`
   - URL: `https://services.leadconnectorhq.com/products/inventory`
   - Updates GHL's own stock count to confirm the deduction

---

### Scenario 2: WooCommerce Order → GHL Stock Update

**Workflow structure:**

1. **Webhook Node (Trigger)**
   - Set up a WooCommerce webhook under **WooCommerce → Settings → Advanced → Webhooks**
   - Topic: `Order: Created` → point to this n8n webhook URL

2. **IF Node — Check Order Status**
   - Only proceed if `status == "processing"` or `status == "completed"`
   - This is the critical gate (see payment failure section below)

3. **HTTP Request Node — Update GHL Inventory**
   - Call GHL's inventory API to deduct the qty sold from GHL's stock

---

## What Happens When a WooCommerce Payment Fails?

This is an important edge case to design around. Here's the order status flow in WooCommerce:

| Status | Meaning | Should it deduct stock? |
|--------|---------|------------------------|
| `pending` | Order created, no payment yet | ❌ No |
| `failed` | Payment attempted and failed | ❌ No — WooCommerce **restores** stock automatically |
| `on-hold` | Awaiting manual payment (bank transfer, etc.) | ⚠️ Depends on your settings |
| `processing` | Payment confirmed, fulfilling | ✅ Yes |
| `completed` | Fulfilled | ✅ Yes |
| `cancelled` | Cancelled after payment | ✅ Restore stock |
| `refunded` | Refunded | ✅ Restore stock |

**The key behavior:** WooCommerce natively holds and restores stock on failed payments if you have **"Hold stock (minutes)"** configured under `WooCommerce → Settings → Products → Inventory`. When an order goes `pending → failed`, WooCommerce automatically adds the qty back to its own stock.

**The problem** is your n8n scenario — if you trigger on `Order: Created` (which fires at `pending`), you'll deduct from GHL prematurely. A payment failure would restore WooCommerce stock but GHL would still show the wrong count.

---

### The Fix: Trigger on the Right Event

In Scenario 2, change your WooCommerce webhook topic to:
- ✅ **`Order: Updated`** — then use the **IF Node** in n8n to filter only when `status == "processing"` or `status == "completed"`

This way your n8n workflow only fires and touches GHL inventory **after confirmed payment**. For refunds/cancellations, add a second branch in the IF Node that **adds stock back** to GHL when status becomes `cancelled`, `refunded`, or `failed`.

---

### IF Node Logic in n8n for Scenario 2

| Condition | Action |
|-----------|--------|
| `status == processing` OR `status == completed` | **Deduct** qty from GHL inventory |
| `status == cancelled` OR `status == refunded` OR `status == failed` | **Add back** qty to GHL inventory |
| `status == pending` OR `status == on-hold` | **Do nothing** — wait for next status update |
