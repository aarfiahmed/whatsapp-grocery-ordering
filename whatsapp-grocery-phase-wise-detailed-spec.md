# WhatsApp Grocery Ordering System — Phase-Wise Detailed Build Specification

This document is the single reference to hand a coding agent **one phase at a time**. Each phase is self-contained: functional requirements it covers, the classes/methods to build, the API surface it exposes, its dependencies, and its definition of done. Do not feed the agent phases beyond the current one.

---

## Phase 0 — Project Scaffolding

### Scope
Repository structure, build config, infra connections — no business logic yet.

### Deliverables
- Maven/Gradle project: Java 25, Spring Boot 4.1.1.
- Package structure (from architecture doc):
```
com.grocery.whatsapp
├── webhook
├── conversation
├── ai
├── catalog
├── cart
├── order
├── notification
├── admin
├── common
└── config
```
- `application.yml` with profiles: `local`, `dev`, `prod`.
- Datasource config: Postgres (JPA/Hibernate), Redis (Lettuce), Kafka (spring-kafka).
- `docker-compose.yml` for local Postgres + Redis + Kafka.
- CI pipeline skeleton (build + test on push).
- `PROJECT_CONTEXT.md` created (see Section 11).

### Definition of Done
- `./gradlew bootRun` (or Maven equivalent) starts cleanly against docker-compose infra.
- Health check endpoint (`/actuator/health`) returns `UP`.
- No business modules yet — this phase is infra-only.

---

## Phase 1 — Catalog Module

### Functional Requirements Covered
FR1–FR4, FR25–FR28 (from the requirements doc): browse categories, list items, search, out-of-stock display; admin CRUD, stock updates, low-stock alerts.

### Entities (JPA — package `catalog.entity`)

**`Category`**
| Field | Type |
|---|---|
| id | UUID (PK) |
| name | String |
| imageUrl | String |
| displayOrder | int |
| active | boolean |

**`Item`**
| Field | Type |
|---|---|
| id | UUID (PK) |
| category | `Category` (ManyToOne) |
| name | String |
| unit | enum `Unit { KG, LITRE, PACK, PIECE }` |
| price | int (minor units) |
| stockQty | int |
| inStock | boolean |
| imageUrl | String |
| updatedAt | Instant |

### Repositories
- `CategoryRepository extends JpaRepository<Category, UUID>`
  - `List<Category> findByActiveTrueOrderByDisplayOrder()`
- `ItemRepository extends JpaRepository<Item, UUID>`
  - `Page<Item> findByCategoryIdAndInStockTrue(UUID categoryId, Pageable p)` (and a variant without the `inStock` filter, per FR4 — out-of-stock items shown, not hidden)
  - `Page<Item> findByNameContainingIgnoreCase(String query, Pageable p)`

### Service Layer

**`CatalogService`**
| Method | Responsibility |
|---|---|
| `List<CategoryDto> listActiveCategories()` | FR1 |
| `Page<ItemDto> listItemsByCategory(UUID categoryId, Pageable p)` | FR2, FR4 |
| `Page<ItemDto> searchItems(String query, Pageable p)` | FR3 |
| `ItemDto createItem(ItemUpsertRequest req)` | FR25, admin |
| `ItemDto updateItem(UUID itemId, ItemUpsertRequest req)` | FR25 |
| `ItemDto updateStock(UUID itemId, boolean inStock, String reason)` | FR26 — publishes `item.out_of_stock` event when flipped to false (stub the publish call here; real Kafka wiring lands in Phase 4 — leave a `@TODO` or an injected no-op `EventPublisher` interface implemented later) |
| `void adjustStockQty(UUID itemId, int delta)` | FR27 |
| `List<ItemDto> lowStockItems(int threshold)` | FR28, admin-only |

### Controllers

**`CatalogController`** (`/catalog/**`) — see OpenAPI paths `/catalog/categories`, `/catalog/categories/{categoryId}/items`, `/catalog/items/search`.

**`AdminCatalogController`** (`/admin/catalog/**`) — see OpenAPI paths `/admin/catalog/items`, `/admin/catalog/items/{itemId}`, `/admin/catalog/items/{itemId}/stock`. Secured with `@PreAuthorize("hasAnyRole('OWNER_ADMIN','STAFF')")`, stock/price edits further restricted to `OWNER_ADMIN` if you want that split (your call — not in the original requirements, flagging as an option).

### DTOs
- `CategoryDto`, `ItemDto`, `ItemUpsertRequest`, `ItemPageDto` — field-for-field per the OpenAPI spec `Category`/`Item`/`ItemUpsertRequest`/`ItemPage` schemas.

### Dependencies
None on other modules — build and test this fully standalone.

### Definition of Done
- All endpoints in the OpenAPI spec's Catalog + Admin-Catalog paths implemented and passing integration tests (Testcontainers Postgres).
- Out-of-stock items confirmed present in list responses with `inStock: false`, never omitted.
- Unit tests for `CatalogService` covering: normal listing, empty category, search with no match, stock flip triggers the (stubbed) event hook.

---

## Phase 2 — Cart Module

### Functional Requirements Covered
FR5–FR7, FR10.

### Data Model
Redis-backed, key pattern: `cart:{customerId}` → JSON-serialized `CartState`.

**`CartState`** (POJO, not a JPA entity)
| Field | Type |
|---|---|
| customerId | String |
| items | `List<CartLineItem>` |
| updatedAt | Instant |

**`CartLineItem`**
| Field | Type |
|---|---|
| itemId | String |
| itemNameSnapshot | String |
| qty | int |
| unitPriceSnapshot | int |

### Repository
**`CartRepository`** (custom, wraps `RedisTemplate<String, CartState>`)
| Method | Responsibility |
|---|---|
| `Optional<CartState> find(String customerId)` | |
| `void save(CartState cart, Duration ttl)` | TTL enforces FR10's "persists until cleared" with a sane expiry (e.g., 7 days) |
| `void delete(String customerId)` | |

### Service Layer

**`CartService`**
| Method | Responsibility |
|---|---|
| `CartDto getCart(String customerId)` | FR6 — returns empty cart if none exists, never 404 |
| `CartDto upsertItem(String customerId, String itemId, int qty)` | FR5, FR7 — qty 0 removes; **must call `CatalogService` to snapshot current price/name and validate stock** before adding |
| `void clearCart(String customerId)` | Supports checkout completion and explicit clear |

### Controller
**`CartController`** (`/cart/**`) — per OpenAPI `/cart/{customerId}`, `/cart/{customerId}/items`.

### Dependencies
Depends on **Phase 1 (Catalog)** for price/stock validation on add — this is the first cross-module call; keep it behind `CatalogService`'s public interface, not a direct repository reach-through.

### Definition of Done
- Add/remove/update flows tested against a real Redis (Testcontainers).
- Adding an out-of-stock item returns `422` per the OpenAPI contract.
- Cart TTL verified (expires, doesn't linger forever).

---

## Phase 3 — Order Module

### Functional Requirements Covered
FR8, FR9, FR11–FR14, FR19–FR24.

### Entities (JPA — package `order.entity`)

**`Order`**
| Field | Type |
|---|---|
| id | UUID (PK) |
| customerId | String |
| status | enum `OrderStatus { PLACED, CONFIRMED, PACKING, PACKED, OUT_FOR_DELIVERY, READY_FOR_PICKUP, DELIVERED, CANCELLED }` |
| fulfillmentType | enum `FulfillmentType { DELIVERY, PICKUP }` |
| deliveryAddress | String, nullable |
| deliverySlot | String, nullable |
| note | String, nullable |
| subtotal | int |
| idempotencyKey | String (unique) |
| placedAt | Instant |
| updatedAt | Instant |

**`OrderItem`**
| Field | Type |
|---|---|
| id | UUID (PK) |
| order | `Order` (ManyToOne) |
| itemId | String |
| itemNameSnapshot | String |
| qty | int |
| unitPriceSnapshot | int |
| lineTotal | int |

**`OrderStatusHistory`**
| Field | Type |
|---|---|
| id | UUID (PK) |
| order | `Order` (ManyToOne) |
| status | enum (same as `OrderStatus`) |
| changedBy | enum `ChangedBy { CUSTOMER, ADMIN, AI, SYSTEM }` |
| adminUserId | String, nullable |
| reason | String, nullable |
| occurredAt | Instant |

### Repositories
- `OrderRepository extends JpaRepository<Order, UUID>`
  - `Optional<Order> findByIdempotencyKey(String key)` — NFR1 enforcement
  - `Page<Order> findByCustomerId(String customerId, Pageable p)`
  - `Page<Order> findByStatusIn(List<OrderStatus> statuses, Pageable p)` — admin queue
- `OrderStatusHistoryRepository extends JpaRepository<OrderStatusHistory, UUID>`
  - `List<OrderStatusHistory> findByOrderIdOrderByOccurredAt(UUID orderId)`

### Domain Logic — Status Transition Rules

**`OrderStatusTransitionValidator`** (pure domain class, no Spring dependencies — easy to unit test exhaustively)
| Method | Responsibility |
|---|---|
| `boolean canTransition(OrderStatus from, OrderStatus to)` | Encodes the allowed forward sequence; single source of truth used by both manual and AI paths |
| `boolean isCancellable(OrderStatus current)` | True only for `PLACED`, `CONFIRMED` (FR14) |

### Service Layer

**`OrderService`**
| Method | Responsibility |
|---|---|
| `OrderDto placeOrder(PlaceOrderRequest req)` | FR8, FR9 — reads cart via `CartService`, validates stock via `CatalogService`, snapshots prices, decrements stock, checks `idempotencyKey` first and short-circuits with existing order if found |
| `OrderDto getOrder(UUID orderId)` | FR11 |
| `Page<OrderDto> getOrderHistory(String customerId, Pageable p)` | FR12 |
| `ReorderResultDto reorder(UUID orderId)` | FR13 — builds a new cart via `CartService`, flags unavailable items rather than silently dropping them |
| `OrderDto cancelOrder(UUID orderId, ChangedBy cancelledBy, String reason)` | FR14, FR23 — uses `OrderStatusTransitionValidator.isCancellable`; throws `OrderNotCancellableException` (→ `409`) otherwise |
| `OrderDto updateStatus(UUID orderId, OrderStatus newStatus, ChangedBy changedBy, String adminUserId)` | FR22 — uses `canTransition`; throws `InvalidStatusTransitionException` (→ `422`) otherwise; appends `OrderStatusHistory` row; **publishes `order.status.changed`** (stub until Phase 4, same pattern as Phase 1's stock event) |
| `Page<OrderDto> getAdminQueue(List<OrderStatus> statuses, Pageable p)` | FR19 |

### Exceptions (package `order.exception`)
- `OrderNotCancellableException` → mapped to HTTP 409
- `InvalidStatusTransitionException` → mapped to HTTP 422
- `InsufficientStockException` → mapped to HTTP 422

### Controllers
- **`OrderController`** (`/orders/**`) — place, get, history, cancel, reorder.
- **`AdminOrderController`** (`/admin/orders/**`) — queue, status update.

### Dependencies
Depends on **Phase 1** (stock validation, price snapshot) and **Phase 2** (reading/clearing cart on order placement).

### Definition of Done
- Full status-transition matrix unit-tested (every valid and invalid pair).
- Duplicate `idempotencyKey` submission returns the original order, not a new one (test this explicitly with two concurrent-ish requests).
- Cancellation blocked correctly once status is `PACKING` or later.
- `OrderStatusHistory` rows are append-only — verify no update/delete path exists on that repository at all (don't just rely on convention).

---

## Phase 4 — Kafka Events + Notification Module

### Functional Requirements Covered
FR15–FR18.

### Event Schemas (package `common.event`)
POJOs matching the Kafka contracts already defined: `OrderPlacedEvent`, `OrderStatusChangedEvent`, `ItemOutOfStockEvent`. Serialize as JSON (Spring Kafka `JsonSerializer`).

### Wiring back into Phase 1 & 3
Replace the stubbed `EventPublisher` no-op from Phases 1 and 3 with a real `KafkaEventPublisher implements EventPublisher` — this is why those phases were built against an interface, not a concrete Kafka call, from the start.

### Notification Service (package `notification`)

**`NotificationEventListener`**
| Method | Responsibility |
|---|---|
| `@KafkaListener onOrderPlaced(OrderPlacedEvent e)` | Admin-facing alert only (FR20) — not customer notification |
| `@KafkaListener onOrderStatusChanged(OrderStatusChangedEvent e)` | FR15 |
| `@KafkaListener onItemOutOfStock(ItemOutOfStockEvent e)` | FR16 — looks up affected customers via `StockWatchRepository` (see below) and active carts |

**`WhatsAppMessageSender`**
| Method | Responsibility |
|---|---|
| `void sendServiceMessage(String customerId, String text)` | Free-form, only valid inside the 24h window |
| `void sendTemplateMessage(String customerId, String templateName, List<String> params)` | Utility/Marketing templates |
| `boolean isWithinServiceWindow(String customerId)` | Checks last-inbound-message timestamp; decides which of the above two to call |

### New Entity: `StockWatch`
| Field | Type |
|---|---|
| id | UUID (PK) |
| customerId | String |
| itemId | String |
| createdAt | Instant |
| notified | boolean |

### Dependencies
Depends on **Phase 1** and **Phase 3** for the event producers to now be real.

### Definition of Done
- End-to-end test: update order status → Kafka event observed → notification listener invoked (use an embedded/Testcontainers Kafka broker).
- Service-window vs. template-message branching unit tested with mocked "last inbound message" timestamps on both sides of the 24h boundary.

---

## Phase 5 — WhatsApp Webhook + Conversation Service (Manual/Menu Flow)

### Functional Requirements Covered
The full customer-facing menu-driven flow (FR1–FR14) now becomes reachable via WhatsApp, without AI yet.

### Entities
**`Customer`**
| Field | Type |
|---|---|
| id | UUID (PK) |
| phoneNumber | String (unique) |
| name | String, nullable |
| defaultAddress | String, nullable |
| marketingOptIn | boolean, default false |
| transactionalOptIn | boolean, default true |
| createdAt | Instant |

### Webhook Layer (package `webhook`)
**`WhatsAppWebhookController`**
| Method | Responsibility |
|---|---|
| `String verifyWebhook(...)` | Meta handshake (GET) |
| `ResponseEntity<Void> receiveEvent(WhatsAppWebhookPayload payload)` | POST — validates `X-Hub-Signature-256`, deserializes, publishes `InboundMessageEvent` to Kafka, returns 200 immediately |

### Conversation Layer (package `conversation`)
**`ConversationSessionRepository`** — Redis-backed, key `session:{customerId}` → `SessionState` (current menu position, active cart reference, last-activity timestamp for the 24h-window calculation).

**`ConversationRouter`**
| Method | Responsibility |
|---|---|
| `void handleInbound(InboundMessageEvent event)` | Looks up/creates `Customer`; determines message type (text vs. interactive reply); routes structured replies to the relevant module service (`CatalogService`, `CartService`, `OrderService`); free-text routed to AI in Phase 7 (stub as "not yet supported, show menu" until then) |

**`WhatsAppMessageFormatter`**
| Method | Responsibility |
|---|---|
| `InteractiveListPayload formatCategoryList(List<CategoryDto> categories)` | Enforces the 10-row/10-section WhatsApp limit — paginate if exceeded |
| `InteractiveListPayload formatItemList(List<ItemDto> items)` | Same limit handling |
| `TextPayload formatOrderConfirmation(OrderDto order)` | |
| `TextPayload formatOrderStatusUpdate(OrderStatusChangedEvent e)` | |

### Dependencies
Depends on **Phases 1–4** — this phase is largely wiring, not new domain logic.

### Definition of Done
- Full manual flow testable via a simulated WhatsApp payload sequence: browse → add to cart → place order → check status → cancel — all working end-to-end without any AI involvement.
- List-message pagination verified against >10 items in a category.

---

## Phase 6 — Admin Module

### Functional Requirements Covered
FR29 (partially — broadcast composition, full send lands with Phase 8), FR30–FR32.

### Entities
**`AdminUser`**
| Field | Type |
|---|---|
| id | UUID (PK) |
| name | String |
| email | String (unique) |
| passwordHash | String |
| role | enum `AdminRole { OWNER_ADMIN, STAFF }` |
| active | boolean |

### Security
**`SecurityConfig`** — Spring Security, JWT-based auth for the admin web app; `@PreAuthorize` role checks on all `/admin/**` endpoints per the role split already used in Phases 1 and 3.

### Controllers
**`AdminAuthController`** — login, token issuance.
**`AdminDashboardController`** — thin aggregation endpoints if needed (e.g., daily summary — optional extra from the requirements doc's "additional functionality" table).

### Dependencies
Depends on **Phase 1** and **Phase 3** for the data it manages; introduces the auth layer everything in Phase 8 will also use.

### Definition of Done
- Login issues a JWT with correct role claim.
- `STAFF` role blocked from catalog price/broadcast endpoints (or whatever split you choose); `OWNER_ADMIN` has full access — test both.

---

## Phase 7 — AI Conversational Layer

### Functional Requirements Covered
FR-AI1–FR-AI14.

### Entities
**`Conversation`**, **`ConversationTurn`**, **`KnowledgeDocument`**, **`KnowledgeChunk`** — fields exactly as defined in the ER model document.

### AI Service Layer (package `ai`)

**`AiOrchestrationService`**
| Method | Responsibility |
|---|---|
| `AiQueryResponse handleQuery(AiQueryRequest req)` | FR-AI4, FR-AI6 — classifies intent via `ChatClient`; routes to RAG or tool-call path |
| `OrderDto confirmAction(AiConfirmRequest req)` | FR-AI5 — executes the previously proposed action **only** on explicit confirmation, by calling `OrderService` directly (same method Phase 3 already tested) |

**Tool-exposed methods** (Spring AI `@Tool`, package `ai.tools`) — thin wrappers delegating to already-tested Phase 3 services, per FR-AI4:
```java
@Tool(description = "Cancel a customer's order if it hasn't started packing")
CancelResult cancelOrder(String orderId, String customerId);

@Tool(description = "Get current status of a customer's order")
OrderStatusResult getOrderStatus(String orderId);

@Tool(description = "Get a customer's recent order history")
List<OrderSummary> getOrderHistory(String customerId, int limit);
```

**`RagQueryService`**
| Method | Responsibility |
|---|---|
| `RagAnswer answer(String customerMessage)` | Embeds query, similarity search against `KnowledgeChunk` (pgvector), generates grounded answer via `ChatClient`, returns answer + `sourceDocId` + confidence |
| `boolean isConfident(RagAnswer answer)` | Threshold check driving FR-AI2/escalation |

**`KnowledgeIngestionService`**
| Method | Responsibility |
|---|---|
| `void ingestDocument(UUID documentId)` | FR30 — splits, embeds, stores `KnowledgeChunk`s; sets `ingestionStatus` |

**`ConversationLogService`**
| Method | Responsibility |
|---|---|
| `void logTurn(ConversationTurn turn)` | FR-AI12, FR31 |
| `Page<ConversationTurn> findEscalated(LocalDate from, LocalDate to)` | Admin review query |

### Wiring change in Phase 5
`ConversationRouter.handleInbound` — replace the "not yet supported" stub for free-text with a real call to `AiOrchestrationService.handleQuery`.

### Dependencies
Depends on **Phase 3** (tool calls), **Phase 5** (routing), **Phase 6** (knowledge-document admin endpoints).

### Definition of Done
- Informational query test: known policy question → grounded answer with correct `sourceDocId`.
- Unknown query test: returns `ESCALATE`, never a fabricated answer.
- Actionable query test: "cancel my order" → `ACTIONABLE_PENDING_CONFIRMATION` → confirm → verify it calls the **same** `OrderService.cancelOrder` path Phase 3 already tested, including the same 409 behavior if status has since moved to `PACKING`.
- Race condition test: order status changes to `PACKING` *between* the AI's proposal and the customer's confirmation — confirm step must still hit the real validation and correctly reject.

---

## Phase 8 — Broadcast + Knowledge Base Management (Full)

### Functional Requirements Covered
FR29 (completed), FR30–FR31 (admin UI side).

### Entities
**`Broadcast`**, **`BroadcastRecipient`** — as per ER model.

### Service Layer
**`BroadcastService`**
| Method | Responsibility |
|---|---|
| `Broadcast scheduleBroadcast(BroadcastRequest req)` | Filters recipients strictly by `marketingOptIn = true` at the query level, not just at send time |
| `void executeBroadcast(UUID broadcastId)` | Fans out `WhatsAppMessageSender.sendTemplateMessage` per recipient, records delivery status per `BroadcastRecipient` |

### Dependencies
Depends on **Phase 4** (`WhatsAppMessageSender`), **Phase 6** (admin auth).

### Definition of Done
- Test confirms a customer with `marketingOptIn = false` can never appear in `BroadcastRecipient` for any broadcast, under any segment type.

---

## Phase 9 — Observability, Security Hardening, Deployment

### Scope
Cross-cutting, applied once the app is functionally complete.

### Deliverables
- Micrometer + OpenTelemetry tracing wired across webhook → conversation → service → Kafka → notification (correlation ID = original WhatsApp message ID).
- Resilience4j circuit breakers around the LLM call and WhatsApp API call.
- Rate limiting (Redis token bucket) on cart/order/AI endpoints per phone number.
- Kubernetes manifests (Deployments, HPA, Ingress) per the architecture doc's namespace layout.
- Secrets wired via AWS Secrets Manager + External Secrets Operator.

### Definition of Done
- Load test confirms HPA scales `webhook-service` and `catalog-service` correctly under simulated traffic.
- Chaos test: kill the LLM endpoint mid-conversation — confirm graceful fallback to menu-only mode, no hung sessions.

---

## 10. Cross-Phase Testing Discipline

- Every phase ships with its own test suite; **no phase is "done" until its tests are green in isolation**, before the next phase begins.
- When a later phase needs to modify an earlier phase's code (e.g., Phase 4 replacing the stub `EventPublisher`), re-run that earlier phase's full test suite before proceeding — this is your regression guard against context loss.
- Keep integration tests on Testcontainers (Postgres, Redis, Kafka) so each phase's tests are fully self-contained and reproducible in any fresh agent session.

---

## 11. `PROJECT_CONTEXT.md` — Template to Maintain Throughout

```markdown
# Project Context

## Stack (locked)
Java 25, Spring Boot 4.1.1, Spring AI 2.0.1, PostgreSQL 17 + pgvector,
Redis, Kafka, WhatsApp Cloud API, AWS EKS.

## Module boundaries
(paste the package tree from Phase 0)

## Conventions
- Prices: integer minor units (paise), never float.
- All timestamps: Instant/UTC.
- Idempotency: required on all state-changing customer-facing POSTs.
- Order status transitions: single source of truth is OrderStatusTransitionValidator —
  never duplicate this logic elsewhere (including in the AI tool layer).

## Phase log
- [x] Phase 0 — Scaffolding — done, CI green
- [x] Phase 1 — Catalog — done, tests passing, entities match ER model Section 2.3
- [ ] Phase 2 — Cart — in progress: TTL not yet configured
- [ ] Phase 3 — Order
- ...

## Known open decisions
- STAFF vs OWNER_ADMIN split on price-editing: not yet decided, currently both allowed
```

Feed only this file plus the **current phase's section of this spec** to each new coding-agent session — not the entire multi-document set. That scoping is what actually prevents context and functionality loss across a build this size.

---

*Next natural step: a filled-out PHASE_1_BRIEF.md (or whichever phase you want to start with) as the literal first prompt to hand your coding agent.*
