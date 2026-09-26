# WhatsApp Grocery Ordering System — Project Overview

## 1. What this project is

A WhatsApp-based ordering platform for a local grocery shop. Customers browse categories and items, build a cart, place orders, and track them — all through WhatsApp conversation, with an AI layer handling natural-language questions ("what's your cancellation policy", "cancel my order", "where's my order"). No payment gateway — Cash on Delivery / Pay on Pickup only. Shop admin/staff manage orders, catalog, stock, and broadcasts through a separate lightweight admin web app.

## 2. Who's involved

- **Customer** — interacts entirely through WhatsApp.
- **Shop Admin / Staff** — manages orders, catalog, and knowledge base through an internal admin web app.
- **AI Assistant** — an internal service that classifies customer intent and either answers from a policy knowledge base (RAG) or triggers the same backend actions a human-driven menu flow would trigger — it never bypasses the deterministic order/catalog rules.

## 3. Core functionality

**Customer-side**
- Browse categories → items (image, price, stock status)
- Search items by name
- Add/remove/update cart items
- Place order (address/pickup + delivery slot + note)
- Check order status, view order history, reorder, cancel (before packing starts)
- Get notified on every order-status change, out-of-stock alerts on active-order items, and (opt-in) new-arrival/promo broadcasts
- Ask natural-language questions (policy FAQs, "cancel my order", "where's my order") handled by the AI layer

**Admin-side**
- Live order queue with notification on new orders
- Advance order status (Placed → Confirmed → Packing → Packed → Out for Delivery/Ready for Pickup → Delivered), each step notifying the customer
- Catalog CRUD, stock updates (triggers customer out-of-stock alerts), low-stock internal alerts
- Broadcast composition to opted-in customers
- Manage the AI's knowledge-base documents (policies/FAQs) and review escalated/low-confidence AI conversations

## 4. Key platform constraint driving several design decisions

WhatsApp Business Platform only allows free-form messaging within a 24-hour window after the customer's last message; anything sent outside that window needs a pre-approved Message Template, split into **Utility** (transactional — order status, out-of-stock) and **Marketing** (promotional — requires separate, explicit opt-in). This is why the data model keeps `marketingOptIn` and `transactionalOptIn` as two distinct flags, and why the Notification module always checks the service window before choosing which message type to send.

## 5. Technical stack (locked)

| Layer | Choice |
|---|---|
| Language | Java 25 LTS |
| Framework | Spring Boot 4.1.1 |
| AI | Spring AI 2.0.1 (OpenAI + Ollama), RAG via pgvector, tool-calling for actionable intents |
| Messaging channel | WhatsApp Cloud API (direct with Meta) |
| Event backbone | Apache Kafka |
| Session/cart state | Redis |
| Database | PostgreSQL 17 + pgvector |
| Object storage | AWS S3 |
| Deployment | Docker + Kubernetes (EKS) |
| Concurrency model | Java virtual threads (not WebFlux) — matches the I/O-bound webhook workload without reactive complexity |

## 6. Architecture at a glance

Modular monolith to start (one Spring Boot app, clean package boundaries per component) rather than microservices from day one — split out Notification Service and AI Orchestration Service later only if they need independent scaling. Core flow:

```
WhatsApp Cloud API → Webhook Controller → Conversation Service
   ├─ structured menu path → Catalog / Cart / Order services
   └─ free-text path → AI Orchestration → RAG (pgvector) or tool-call → Order/Catalog services

Order/Catalog services → Kafka events → Notification Service → WhatsApp Cloud API → Customer
Admin Web App → Admin Service → Catalog / Order / Knowledge-base management
```

Design principle carried through every layer: **AI classifies and retrieves; it never executes directly.** Every state-changing action (cancel order, place order) goes through the same deterministic service and business-rule validation whether triggered by a button tap or an AI-parsed intent.

## 7. Data model (core entities)

`Customer`, `Category`, `Item`, `Cart`/`CartItem`, `Order`/`OrderItem`/`OrderStatusHistory`, `StockWatch`, `Conversation`/`ConversationTurn`, `KnowledgeDocument`/`KnowledgeChunk`, `AdminUser`, `Broadcast`/`BroadcastRecipient`.

Notable design choices:
- `OrderItem` snapshots price/name at order time — catalog changes never rewrite order history.
- `OrderStatusHistory.changedBy` includes `AI` as a value alongside `CUSTOMER`/`ADMIN`/`SYSTEM` — full accountability once the AI layer can act.
- `Order.idempotencyKey` is a unique DB constraint — the real guarantee against duplicate orders from retried WhatsApp webhooks, not just an application-layer check.

## 8. API surface

REST APIs for Catalog, Cart, Order (customer-facing) and Admin (order queue, catalog CRUD, broadcasts, knowledge-base management), plus an internal AI contract (`/internal/ai/query` → classify, `/internal/ai/confirm` → execute only after explicit confirmation). Full OpenAPI 3.1 spec generated and available separately.

## 9. Build strategy

Given the system's size, it's being built in **vertical slices**, not layer-by-layer — each phase is a fully working, tested unit end-to-end before the next begins, so a coding agent never needs to hold the whole system in context at once.

| Phase | Module |
|---|---|
| 0 | Project scaffolding |
| 1 | Catalog |
| 2 | Cart |
| 3 | Order |
| 4 | Kafka events + Notifications |
| 5 | WhatsApp webhook + Conversation (manual/menu flow) |
| 6 | Admin module |
| 7 | AI conversational layer |
| 8 | Broadcast + knowledge-base management (full) |
| 9 | Observability, security hardening, deployment |

A `PROJECT_CONTEXT.md` file (stack decisions, module boundaries, conventions, phase log) is maintained throughout and fed to every new coding-agent session alongside only the current phase's spec — not the full document set — to prevent context and functionality loss across the build.

## 10. Document set

This overview sits alongside five companion documents already produced for this project:
1. Detailed functional requirements (customer + admin + AI layer)
2. Technical stack rationale
3. Complete architecture (component diagram, sequence flows, deployment, security)
4. Full ER data model
5. API contract definitions + OpenAPI 3.1 spec
6. Phase-wise detailed build specification (classes, methods, API mapping per phase)

## 11. Current status

Requirements, architecture, data model, and API contracts are fully specified. Build has not yet started — next step is drafting the literal first coding-agent brief for Phase 1 (Catalog module).

---

*Next natural step: `PHASE_1_BRIEF.md` — the literal first prompt to hand your coding agent to begin the Catalog module.*
