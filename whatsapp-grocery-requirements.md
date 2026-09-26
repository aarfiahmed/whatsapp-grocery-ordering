# WhatsApp Grocery Ordering System — Requirements Document

## 1. Overview

A WhatsApp-based ordering platform for a grocery shop, using the WhatsApp Business Platform (Cloud API) as the customer-facing interface, backed by a Java/Spring Boot service that manages catalog, cart, orders, and notifications. No payment gateway — Cash on Delivery / Pay at Pickup only.

Two actors: **Customer** (via WhatsApp) and **Shop Admin/Staff** (via a lightweight admin app — WhatsApp alone isn't practical for packing/inventory work at scale).

---

## 2. Customer-Side Functional Requirements

### 2.1 Catalog Browsing
- FR1: Customer can view a list of item categories (e.g., Vegetables, Dairy, Snacks) via WhatsApp List Message or Interactive Buttons.
- FR2: Customer can select a category and view items in it, each with image, name, price, unit (kg/pack/litre), and stock availability.
- FR3: Customer can search for an item by name (text query) instead of browsing category-by-category.
- FR4: Out-of-stock items are shown greyed out / marked unavailable rather than hidden, so customers know they exist.

### 2.2 Cart & Ordering
- FR5: Customer can add an item to cart with a quantity.
- FR6: Customer can view current cart (items, quantities, subtotal) at any time via a command like "cart".
- FR7: Customer can remove an item or update quantity in cart before checkout.
- FR8: Customer can place an order from the cart — capturing delivery address (or pickup choice), preferred delivery slot, and any note (e.g., "ring the bell").
- FR9: System generates a human-readable order ID and confirms order placement with an order summary.
- FR10: Cart persists across sessions (customer can add items today, checkout tomorrow) until explicitly cleared or order placed.

### 2.3 Order Tracking
- FR11: Customer can check current status of an active order ("track order" / "where is my order").
- FR12: Customer can view order history (past N orders with date, items, amount, status).
- FR13: Customer can repeat/reorder a previous order in one step.
- FR14: Customer can cancel an order **while it is still in "Placed" or "Confirmed" status** (before packing starts).

### 2.4 Notifications (Customer)
- FR15: Notify customer on every order status change: Placed → Confirmed → Packing → Packed → Out for Delivery / Ready for Pickup → Delivered/Completed → (or Cancelled).
- FR16: Notify customer if an item in their **placed** order turns out to be unavailable (substitution or removal, with admin decision — see 3.4).
- FR17: Notify customer (opt-in only) about new product arrivals or promotions/discounts.
- FR18: Customer can opt out of promotional notifications independently of transactional (order-status) notifications — WhatsApp policy requires this distinction (see Section 5).

---

## 3. Admin-Side Functional Requirements

### 3.1 Order Management
- FR19: Admin sees a live queue of new/incoming orders.
- FR20: Admin gets notified (app push/sound, not necessarily WhatsApp) immediately on new order.
- FR21: Admin can view full order detail: items, quantities, customer name/phone/address, notes.
- FR22: Admin can advance order status: Confirmed → Packing → Packed → Out for Delivery/Ready for Pickup → Delivered. Each transition triggers FR15.
- FR23: Admin can mark an order Cancelled (with reason), which notifies the customer.
- FR24: Admin can edit an order before confirming (e.g., adjust quantity if partially unavailable) — customer is notified of the change and must implicitly or explicitly accept it.

### 3.2 Catalog & Inventory Management
- FR25: Admin can add/edit/remove categories and items (name, price, image, unit, stock count).
- FR26: Admin can mark an item out-of-stock/in-stock; system auto-notifies customers who have that item in an active cart or a "notify me" watchlist (FR30).
- FR27: Stock count auto-decrements on order confirmation and can be manually adjusted (damage, spot-check correction).
- FR28: Low-stock threshold alert for admin (internal), separate from customer-facing out-of-stock notice.

### 3.3 Broadcast / Marketing
- FR29: Admin can send a broadcast (new arrival, sale) to all opted-in customers, ideally with basic segmentation (e.g., customers who bought a category before).

### 3.4 Roles
- FR30: Support multiple staff logins (e.g., packer vs. owner) with at least two roles: Owner/Admin (full access, catalog + broadcast) and Staff (order queue + status updates only).

---

## 4. Additional Functionality Worth Considering

These weren't in your list but are natural gaps once the above is built:

| Feature | Why it matters |
|---|---|
| **"Notify me" for out-of-stock items** | Customer taps "notify when available" instead of you guessing who wants it |
| **Minimum order value / delivery charge rules** | Common real-world grocery constraint |
| **Delivery slot selection** | Reduces "where's my order" queries |
| **Address book (save frequent addresses)** | Faster repeat checkout |
| **Order edit window** | Let customer amend order for a short grace period (e.g., 5 min) before admin starts packing |
| **Ratings/feedback after delivery** | Cheap quality signal, one tap |
| **Multi-language (Hindi/English toggle)** | High-value for a local grocery shop's real customer base |
| **Session/idle timeout with re-greeting** | WhatsApp bot UX hygiene |
| **Admin daily sales summary** | Simple report, high owner value, low effort given you already have the order data |
| **Delivery partner/rider assignment** | If you outgrow self-delivery |

---

## 5. Important Platform Constraint (WhatsApp Business Policy)

This affects your architecture, not just a "nice to have":
- WhatsApp only allows **free-form messaging within a 24-hour customer-service window** after the customer last messaged you. Outside that window, you can only send **pre-approved Message Templates** (used for order-status updates, and required for any promotional/broadcast message).
- Promotional templates require **explicit opt-in** and Meta's Business Verification for your WhatsApp Business Account — factor this into your notification design (FR17/FR29) from day one, not as an afterthought.
- You'll need to go through Meta/a Business Solution Provider (BSP) to get Cloud API access, or self-host against the on-prem API (being deprecated) — plan for a BSP or direct Cloud API integration.

---

## 6. Non-Functional Requirements

- NFR1: Idempotent order placement (customer double-tap/network retry must not create duplicate orders).
- NFR2: Audit trail of every order status change (who changed it, when).
- NFR3: Notification delivery should be async/queued (Kafka fits well here) so WhatsApp API slowness never blocks order processing.
- NFR4: Basic rate limiting/anti-abuse on cart and order-placement endpoints per phone number.
- NFR5: PII (customer phone, address) encrypted at rest; access limited to admin roles.
- NFR6: Horizontal scalability for catalog reads (this is the highest-traffic path).

---

## 7. Suggested Phased Roadmap

**Phase 1 — MVP**
Categories → items → cart → place order (COD) → admin order queue → manual status update → status-change notification.

**Phase 2**
Order history, reorder, cancellation, out-of-stock notify, low-stock alert, staff roles.

**Phase 3**
Broadcast/marketing opt-in flow, delivery slots, ratings, daily sales summary for admin.

**Phase 4**
Multi-language, address book, delivery partner assignment, analytics dashboard.

---

## 8. Suggested Architecture (aligned to your stack)

- **WhatsApp Cloud API** (via Meta or a BSP like Gupshup/Interakt) as the channel.
- **Spring Boot** service exposing a webhook for inbound WhatsApp messages, backed by a conversation-state machine (Redis for cart/session state — fast, ephemeral, matches WhatsApp's chat-turn nature).
- **Kafka** for order events (`order.placed`, `order.status.changed`, `item.out_of_stock`) — decouples notification delivery from order processing, and gives you a clean audit log for free.
- **Postgres/MySQL** for catalog, orders, order history (durable, relational — orders/items/customers are naturally relational).
- **S3** for item images.
- **Admin app**: a small internal web app (not WhatsApp) for packing/catalog/broadcast — trying to run inventory management *inside* WhatsApp chat is the most common mistake in these projects.
- **EKS/Kubernetes** for deployment, consistent with what you already run.

---

*Let me know if you'd like this broken into user stories / API contracts for Phase 1, or a data model (entities + relationships) next.*
