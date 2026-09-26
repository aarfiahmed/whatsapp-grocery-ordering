# WhatsApp Grocery Ordering System — Detailed Requirements (v2, with AI Layer)

## 1. Overview

A WhatsApp-based ordering platform for a grocery shop, using the WhatsApp Business Platform (Cloud API) as the customer-facing channel, backed by a Java/Spring Boot service that manages catalog, cart, orders, notifications, and an AI-driven conversational layer for natural-language queries. No payment gateway — Cash on Delivery / Pay on Pickup only.

Three actors: **Customer** (via WhatsApp), **Shop Admin/Staff** (via a lightweight admin web app), and the **AI Assistant** (an intent-routing layer sitting between customer messages and the backend — not a replacement for the backend, a front door to it).

---

## 2. Customer-Side Functional Requirements

### 2.1 Catalog Browsing
- FR1: View item categories via Interactive List Message or WhatsApp Catalog.
- FR2: View items in a category with image, name, price, unit, and stock availability.
- FR3: Search for an item by name.
- FR4: Out-of-stock items shown as unavailable, not hidden.

### 2.2 Cart & Ordering
- FR5: Add item to cart with quantity.
- FR6: View current cart (items, quantities, subtotal).
- FR7: Remove item / update quantity before checkout.
- FR8: Place order — capture address or pickup choice, delivery slot, note.
- FR9: Order confirmation with human-readable order ID and summary.
- FR10: Cart persists across sessions until cleared or ordered.

### 2.3 Order Tracking
- FR11: Check status of active order.
- FR12: View order history.
- FR13: Reorder a previous order in one step.
- FR14: Cancel order while still in "Placed"/"Confirmed" status (before packing starts).

### 2.4 Notifications (Customer)
- FR15: Notify on every order status change (Placed → Confirmed → Packing → Packed → Out for Delivery/Ready for Pickup → Delivered/Completed, or Cancelled).
- FR16: Notify if an item in a placed order becomes unavailable.
- FR17: Notify (opt-in only) about new arrivals/promotions.
- FR18: Separate opt-out for promotional notifications, independent of transactional notifications.

---

## 3. AI Conversational Layer — Requirements

This is the new addition: customers should be able to ask things in plain language ("Hindi/English mixed is fine") instead of only navigating menus/buttons. This needs a clear split between **answering** and **acting**, because those have very different risk profiles.

### 3.1 Two categories of AI-handled queries

**A. Informational queries (RAG-answered, no side effects)**
Examples: "What's your order cancellation policy?", "Do you deliver on Sundays?", "What's the minimum order value?", "Can I return spoiled vegetables?"
- FR-AI1: AI answers these by retrieving from a curated knowledge base (shop policies, FAQs, delivery area, timings) — a RAG pipeline over your own policy documents, not the model's general knowledge.
- FR-AI2: If the knowledge base has no relevant answer, AI must say so and offer to connect to a human/admin — never fabricate a policy.
- FR-AI3: Every AI-generated policy answer must be traceable to a specific source document/section for later audit (so if a customer disputes "but your bot told me X", you can check what it actually pulled from).

**B. Actionable queries (intent → backend function call, no free-text guessing)**
Examples: "Cancel my order", "Where is my order", "What did I order last time"
- FR-AI4: AI performs **intent classification + entity extraction** only (e.g., intent = `cancel_order`, entity = order ID or "my last order"). The actual cancellation/status-lookup is executed by your existing deterministic backend APIs (Section 2), never by the LLM generating an answer from memory.
- FR-AI5: For any state-changing action (especially cancellation), AI must present a confirmation step before executing ("You want to cancel order #1234 — confirm?") rather than acting on a single ambiguous message.
- FR-AI6: If intent confidence is low or the message is ambiguous, AI falls back to the structured menu/buttons flow rather than guessing.
- FR-AI7: AI must respect the same business rules as the manual flow — e.g., it cannot "cancel" an order that's already in Packing/Packed status; it must return the same rejection the deterministic API would.

### 3.2 Conversation quality & safety
- FR-AI8: AI must never invent prices, stock levels, or order data — all facts must come from a live backend call or the RAG knowledge base, never generated.
- FR-AI9: AI must not request or process sensitive data (payment details, government ID) — consistent with your no-payment-integration design and WhatsApp's own restriction on sensitive identifiers.
- FR-AI10: Multi-turn context within a session (e.g., customer says "cancel it" after being told their order status) must resolve correctly to the order in question.
- FR-AI11: Clear escalation path: if AI can't resolve a query after 1–2 clarifying attempts, hand off to admin/staff with the conversation context attached.
- FR-AI12: Log every AI interaction (input, matched intent, retrieved source, action taken) for review and for improving the FAQ knowledge base over time.

### 3.3 Language & tone
- FR-AI13: Support Hindi/English mixed input (Hinglish) — matches your actual customer base for a local shop, and is a natural extension since your own working style already mixes both.
- FR-AI14: Consistent tone matching the shop's brand voice, configurable, not just a raw LLM default.

---

## 4. Admin-Side Functional Requirements

### 4.1 Order Management
- FR19: Live queue of new/incoming orders.
- FR20: Admin notified immediately on new order.
- FR21: Full order detail view (items, customer, address, notes).
- FR22: Advance order status through stages, each transition triggers FR15.
- FR23: Cancel order with reason (admin-initiated), notifies customer.
- FR24: Edit order before confirming (partial availability), customer notified.

### 4.2 Catalog & Inventory Management
- FR25: Add/edit/remove categories and items.
- FR26: Mark item out-of-stock/in-stock; auto-notify customers with that item in active cart or on a "notify me" watchlist.
- FR27: Stock auto-decrements on order confirmation; manual adjustment supported.
- FR28: Low-stock threshold alert (internal, admin-only).

### 4.3 Broadcast / Marketing
- FR29: Send broadcast (new arrival/sale) to opted-in customers, with basic segmentation.

### 4.4 AI Knowledge Base Management (new)
- FR30: Admin can add/edit the policy documents that feed the RAG knowledge base (cancellation policy, delivery policy, timings, etc.) without needing a code change or redeploy.
- FR31: Admin can review a log of AI-handled conversations, especially ones that were escalated or where confidence was low, to catch gaps in the knowledge base.

### 4.5 Roles
- FR32: At least two roles — Owner/Admin (full access) and Staff (order queue + status updates only).

---

## 5. Additional Functionality Worth Considering
| Feature | Why it matters |
|---|---|
| "Notify me" for out-of-stock items | Explicit opt-in instead of guessing who wants a restock alert |
| Minimum order value / delivery charge rules | Common real-world grocery constraint |
| Delivery slot selection | Reduces "where's my order" queries |
| Address book (save frequent addresses) | Faster repeat checkout |
| Order edit window | Short grace period before packing starts |
| Ratings/feedback after delivery | Cheap quality signal |
| Admin daily sales summary | High owner value, low effort given existing order data |
| AI fallback quality report | Weekly view of "questions AI couldn't answer" — your real FAQ backlog |

---

## 6. WhatsApp Platform Constraints (recap — still applies with AI layer)
- Business-initiated messages (status updates, out-of-stock alerts, promos) outside a live 24-hour customer-service window require pre-approved Message Templates, categorized as Utility (status/transactional) or Marketing (promotional) — Marketing requires explicit separate opt-in.
- The AI layer only ever *replies* within a window the customer opened, or triggers a templated proactive message through the same rules as the rest of the system — the AI doesn't get a special exemption from any of this.
- Sensitive identifiers (payment, government ID) must never be requested/processed via WhatsApp — enforce this in the AI layer's guardrails too (FR-AI9).
- Daily unique-customer messaging limits (250 → 2,000 → 10,000 → 100,000 → unlimited) still apply regardless of whether a message originated from AI or a manual admin trigger.
- Pricing shifted from per-conversation to per-message billing on July 1, 2025; free utility/service replies inside the service window begin being charged from October 1, 2026 — factor this into AI-driven proactive nudges too, since more natural-language back-and-forth can mean more messages sent.

---

## 7. Non-Functional Requirements
- NFR1: Idempotent order placement and idempotent AI-triggered actions (no duplicate cancellations from a retried message).
- NFR2: Audit trail for every order status change and every AI-taken action, including which backend API was called and with what parameters.
- NFR3: Async/queued notification delivery (Kafka) so WhatsApp API latency never blocks order processing.
- NFR4: Rate limiting/anti-abuse on cart, order-placement, and AI-query endpoints per phone number.
- NFR5: PII (phone, address) encrypted at rest; access limited to admin roles.
- NFR6: Horizontal scalability for catalog reads (highest-traffic path).
- NFR7: LLM call latency budget — AI response should complete within WhatsApp's expected reply window (a few seconds) to avoid feeling broken; use a smaller/faster model or a cached RAG index for common FAQs.
- NFR8: Cost control on LLM usage — track tokens/cost per conversation; cap or throttle if a single customer session runs unusually long.
- NFR9: Fallback resilience — if the LLM/RAG service is down, the system degrades to the manual button/menu flow, never to silence.

---

## 8. Suggested Phased Roadmap

**Phase 1 — MVP**
Categories → items → cart → place order (COD) → admin order queue → manual status update → status-change notification.

**Phase 2**
Order history, reorder, cancellation, out-of-stock notify, low-stock alert, staff roles.

**Phase 3 — AI Layer**
RAG-based FAQ answering (policies, timings) grounded in admin-managed documents; intent classification for status-check and cancel-order requests routed to existing deterministic APIs; confirmation step before any state-changing AI action; escalation to human on low confidence.

**Phase 4**
Broadcast/marketing opt-in flow, delivery slots, ratings, daily sales summary, AI fallback-quality report for admin.

**Phase 5**
Multi-language expansion, address book, delivery partner assignment, analytics dashboard.

---

## 9. Suggested Architecture (aligned to your stack)

- **WhatsApp Cloud API** as the channel (direct Meta integration, not a BSP, given your team can manage the webhook/onboarding directly).
- **Spring Boot** service exposing the WhatsApp webhook, with a conversation-state machine (Redis for cart/session state).
- **Spring AI** as the orchestration layer between incoming messages and:
  - a **RAG pipeline** (OpenAI or Ollama, per your existing setup) over shop-policy documents for informational queries (FR-AI1–3), and
  - a **tool/function-calling layer** that maps classified intents (cancel_order, check_status, order_history) to your existing Order Service APIs — the LLM never answers these directly, it only decides *which* function to call and with *what* arguments (FR-AI4).
- **Kafka** for order events (`order.placed`, `order.status.changed`, `item.out_of_stock`) — decouples notification delivery, and doubles as the audit log for both manual and AI-triggered actions.
- **Postgres/MySQL** for catalog, orders, order history.
- **S3** for item images and for storing the source policy documents that feed the RAG index.
- **Admin web app** for packing, catalog, broadcast, and now: knowledge-base document management (FR30) and AI conversation review (FR31).
- **EKS/Kubernetes** for deployment, consistent with your current setup.

---

*Next natural step: a data model (entities/relationships) covering Order, OrderItem, Item, Category, Customer, Conversation, Intent, and KnowledgeDocument — happy to draft that next.*
