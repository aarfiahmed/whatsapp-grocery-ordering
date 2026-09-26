# WhatsApp Grocery Ordering System — OpenAPI 3.1 Specification

This is the OpenAPI spec generated from the API contracts, wired against the ER model entities (Customer, Category, Item, Cart, Order, OrderItem, OrderStatusHistory, KnowledgeDocument, etc.). Save the YAML block below as `openapi.yaml` to load directly into Swagger UI, Redoc, or generate a Spring client/server stub via `openapi-generator`.

```yaml
openapi: 3.1.0
info:
  title: WhatsApp Grocery Ordering System API
  version: 1.0.0
  description: >
    REST API backing a WhatsApp-based grocery ordering service.
    Covers catalog browsing, cart management, order lifecycle,
    admin operations, and AI-assisted conversation handling.
servers:
  - url: https://api.internal.yourgrocery.com/v1
    description: Internal API (invoked by Conversation Service / Admin Web App)

tags:
  - name: Catalog
  - name: Cart
  - name: Orders
  - name: Admin
  - name: AI
  - name: Webhook

security:
  - bearerAuth: []

paths:

  # ---------------- WEBHOOK ----------------
  /webhook/whatsapp:
    get:
      tags: [Webhook]
      summary: WhatsApp webhook verification handshake
      security: []
      parameters:
        - name: hub.mode
          in: query
          required: true
          schema: { type: string }
        - name: hub.verify_token
          in: query
          required: true
          schema: { type: string }
        - name: hub.challenge
          in: query
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Echoes back hub.challenge as plain text
          content:
            text/plain:
              schema: { type: string }
    post:
      tags: [Webhook]
      summary: Inbound WhatsApp message/event notification
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/WhatsAppWebhookPayload'
      responses:
        '200':
          description: Acknowledged (processing happens asynchronously via Kafka)

  # ---------------- CATALOG ----------------
  /catalog/categories:
    get:
      tags: [Catalog]
      summary: List all active categories
      responses:
        '200':
          description: List of categories
          content:
            application/json:
              schema:
                type: object
                properties:
                  categories:
                    type: array
                    items: { $ref: '#/components/schemas/Category' }

  /catalog/categories/{categoryId}/items:
    get:
      tags: [Catalog]
      summary: List items within a category
      parameters:
        - name: categoryId
          in: path
          required: true
          schema: { type: string }
        - name: page
          in: query
          schema: { type: integer, default: 0 }
        - name: size
          in: query
          schema: { type: integer, default: 20 }
      responses:
        '200':
          description: Paged list of items
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ItemPage' }
        '404':
          $ref: '#/components/responses/NotFound'

  /catalog/items/search:
    get:
      tags: [Catalog]
      summary: Search items by name
      parameters:
        - name: q
          in: query
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Matching items
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ItemPage' }

  # ---------------- CART ----------------
  /cart/{customerId}:
    get:
      tags: [Cart]
      summary: Get current cart for a customer
      parameters:
        - name: customerId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Current cart
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Cart' }
    delete:
      tags: [Cart]
      summary: Clear the cart
      parameters:
        - name: customerId
          in: path
          required: true
          schema: { type: string }
      responses:
        '204':
          description: Cart cleared

  /cart/{customerId}/items:
    post:
      tags: [Cart]
      summary: Add or update an item quantity in the cart (qty 0 removes it)
      parameters:
        - name: customerId
          in: path
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [itemId, qty]
              properties:
                itemId: { type: string }
                qty: { type: integer, minimum: 0 }
      responses:
        '200':
          description: Updated cart
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Cart' }
        '422':
          description: Insufficient stock
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  # ---------------- ORDERS ----------------
  /orders:
    post:
      tags: [Orders]
      summary: Place a new order from the customer's cart
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/PlaceOrderRequest' }
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '200':
          description: Duplicate idempotencyKey — returns the original order
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '422':
          $ref: '#/components/responses/BusinessRuleViolation'
    get:
      tags: [Orders]
      summary: Get order history for a customer
      parameters:
        - name: customerId
          in: query
          required: true
          schema: { type: string }
        - name: page
          in: query
          schema: { type: integer, default: 0 }
        - name: size
          in: query
          schema: { type: integer, default: 10 }
      responses:
        '200':
          description: Paged order history
          content:
            application/json:
              schema: { $ref: '#/components/schemas/OrderPage' }

  /orders/{orderId}:
    get:
      tags: [Orders]
      summary: Get order detail including status history
      parameters:
        - name: orderId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Order detail
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '404':
          $ref: '#/components/responses/NotFound'

  /orders/{orderId}/cancel:
    post:
      tags: [Orders]
      summary: Cancel an order (allowed only before packing starts)
      parameters:
        - name: orderId
          in: path
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [cancelledBy]
              properties:
                cancelledBy: { type: string, enum: [CUSTOMER, ADMIN, AI] }
                reason: { type: string }
      responses:
        '200':
          description: Order cancelled
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '409':
          description: Order not cancellable in its current status
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /orders/{orderId}/reorder:
    post:
      tags: [Orders]
      summary: Recreate a cart from a previous order
      parameters:
        - name: orderId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: New cart populated from the referenced order
          content:
            application/json:
              schema:
                type: object
                properties:
                  cart: { $ref: '#/components/schemas/Cart' }
                  unavailableItems:
                    type: array
                    items: { $ref: '#/components/schemas/UnavailableItem' }

  # ---------------- AI (internal) ----------------
  /internal/ai/query:
    post:
      tags: [AI]
      summary: Classify and respond to a free-text customer message
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/AiQueryRequest' }
      responses:
        '200':
          description: AI response (informational, actionable-pending, or escalate)
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AiQueryResponse' }

  /internal/ai/confirm:
    post:
      tags: [AI]
      summary: Confirm or reject a pending AI-proposed action
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/AiConfirmRequest' }
      responses:
        '200':
          description: Result of the confirmed action
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }

  # ---------------- ADMIN ----------------
  /admin/catalog/items:
    post:
      tags: [Admin]
      summary: Create a new catalog item
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ItemUpsertRequest' }
      responses:
        '201':
          description: Item created
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Item' }

  /admin/catalog/items/{itemId}:
    put:
      tags: [Admin]
      summary: Update an existing catalog item
      parameters:
        - name: itemId
          in: path
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ItemUpsertRequest' }
      responses:
        '200':
          description: Item updated
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Item' }

  /admin/catalog/items/{itemId}/stock:
    patch:
      tags: [Admin]
      summary: Update stock/availability for an item (triggers out-of-stock notifications)
      parameters:
        - name: itemId
          in: path
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                inStock: { type: boolean }
                reason: { type: string }
      responses:
        '200':
          description: Stock updated
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Item' }

  /admin/orders:
    get:
      tags: [Admin]
      summary: List orders for the admin queue
      parameters:
        - name: status
          in: query
          schema: { type: string }
          description: Comma-separated list, e.g. PLACED,CONFIRMED
        - name: sort
          in: query
          schema: { type: string, default: placedAt,asc }
      responses:
        '200':
          description: Order queue
          content:
            application/json:
              schema: { $ref: '#/components/schemas/OrderPage' }

  /admin/orders/{orderId}/status:
    patch:
      tags: [Admin]
      summary: Advance an order to the next lifecycle status
      parameters:
        - name: orderId
          in: path
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [status, updatedBy]
              properties:
                status:
                  type: string
                  enum: [CONFIRMED, PACKING, PACKED, OUT_FOR_DELIVERY, READY_FOR_PICKUP, DELIVERED]
                updatedBy: { type: string }
      responses:
        '200':
          description: Order status updated
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '422':
          description: Invalid status transition
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /admin/broadcasts:
    post:
      tags: [Admin]
      summary: Schedule a marketing broadcast to opted-in customers
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/BroadcastRequest' }
      responses:
        '201':
          description: Broadcast scheduled
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Broadcast' }

  /admin/knowledge-documents:
    post:
      tags: [Admin]
      summary: Upload or update a policy/FAQ document feeding the RAG knowledge base
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/KnowledgeDocumentRequest' }
      responses:
        '201':
          description: Document created, async re-ingestion triggered
          content:
            application/json:
              schema: { $ref: '#/components/schemas/KnowledgeDocument' }

  /admin/ai-conversations:
    get:
      tags: [Admin]
      summary: Review AI-handled conversations, especially escalations
      parameters:
        - name: escalated
          in: query
          schema: { type: boolean }
        - name: from
          in: query
          schema: { type: string, format: date }
        - name: to
          in: query
          schema: { type: string, format: date }
      responses:
        '200':
          description: Matching conversation turns
          content:
            application/json:
              schema:
                type: object
                properties:
                  conversations:
                    type: array
                    items: { $ref: '#/components/schemas/ConversationTurn' }

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  responses:
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
    BusinessRuleViolation:
      description: Business rule violation (e.g., insufficient stock)
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }

  schemas:

    Error:
      type: object
      properties:
        error:
          type: object
          properties:
            code: { type: string, example: ORDER_NOT_CANCELLABLE }
            message: { type: string }
            details: { type: object, additionalProperties: true }
            timestamp: { type: string, format: date-time }

    Category:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        imageUrl: { type: string }
        itemCount: { type: integer }

    Item:
      type: object
      properties:
        id: { type: string }
        categoryId: { type: string }
        name: { type: string }
        unit: { type: string, enum: [kg, litre, pack, piece] }
        price: { type: integer, description: "Minor units (paise)" }
        stockQty: { type: integer }
        inStock: { type: boolean }
        imageUrl: { type: string }
        updatedAt: { type: string, format: date-time }

    ItemUpsertRequest:
      type: object
      required: [categoryId, name, unit, price]
      properties:
        categoryId: { type: string }
        name: { type: string }
        unit: { type: string }
        price: { type: integer }
        stockQty: { type: integer }
        imageUrl: { type: string }
        inStock: { type: boolean }

    ItemPage:
      type: object
      properties:
        items:
          type: array
          items: { $ref: '#/components/schemas/Item' }
        page: { type: integer }
        totalPages: { type: integer }

    UnavailableItem:
      type: object
      properties:
        itemId: { type: string }
        name: { type: string }
        reason: { type: string }

    CartItem:
      type: object
      properties:
        itemId: { type: string }
        name: { type: string }
        qty: { type: integer }
        unitPrice: { type: integer }
        lineTotal: { type: integer }

    Cart:
      type: object
      properties:
        customerId: { type: string }
        items:
          type: array
          items: { $ref: '#/components/schemas/CartItem' }
        subtotal: { type: integer }
        updatedAt: { type: string, format: date-time }

    Fulfillment:
      type: object
      properties:
        type: { type: string, enum: [DELIVERY, PICKUP] }
        address: { type: string }
        slot: { type: string, description: "ISO 8601 interval" }

    PlaceOrderRequest:
      type: object
      required: [customerId, items, idempotencyKey]
      properties:
        customerId: { type: string }
        items:
          type: array
          items:
            type: object
            properties:
              itemId: { type: string }
              qty: { type: integer }
              unitPrice: { type: integer }
        fulfillment: { $ref: '#/components/schemas/Fulfillment' }
        note: { type: string }
        idempotencyKey: { type: string }

    OrderItem:
      type: object
      properties:
        itemId: { type: string }
        itemNameSnapshot: { type: string }
        qty: { type: integer }
        unitPriceSnapshot: { type: integer }
        lineTotal: { type: integer }

    OrderStatusHistoryEntry:
      type: object
      properties:
        status: { type: string }
        at: { type: string, format: date-time }
        changedBy: { type: string, enum: [CUSTOMER, ADMIN, AI, SYSTEM] }

    Order:
      type: object
      properties:
        orderId: { type: string }
        customerId: { type: string }
        status:
          type: string
          enum: [PLACED, CONFIRMED, PACKING, PACKED, OUT_FOR_DELIVERY, READY_FOR_PICKUP, DELIVERED, CANCELLED]
        items:
          type: array
          items: { $ref: '#/components/schemas/OrderItem' }
        subtotal: { type: integer }
        statusHistory:
          type: array
          items: { $ref: '#/components/schemas/OrderStatusHistoryEntry' }
        placedAt: { type: string, format: date-time }

    OrderPage:
      type: object
      properties:
        orders:
          type: array
          items: { $ref: '#/components/schemas/Order' }
        page: { type: integer }
        totalPages: { type: integer }

    AiQueryRequest:
      type: object
      required: [customerId, message]
      properties:
        customerId: { type: string }
        message: { type: string }
        sessionContext:
          type: object
          properties:
            activeOrderId: { type: string }

    AiQueryResponse:
      type: object
      properties:
        type:
          type: string
          enum: [INFORMATIONAL, ACTIONABLE_PENDING_CONFIRMATION, ESCALATE]
        answer: { type: string }
        sourceDocId: { type: string }
        confidence: { type: number, format: float }
        intent: { type: string }
        targetOrderId: { type: string }
        confirmationPrompt: { type: string }
        reason: { type: string }
        suggestedAction: { type: string }

    AiConfirmRequest:
      type: object
      required: [customerId, intent, confirmed]
      properties:
        customerId: { type: string }
        intent: { type: string }
        targetOrderId: { type: string }
        confirmed: { type: boolean }

    ConversationTurn:
      type: object
      properties:
        customerId: { type: string }
        inputMessage: { type: string }
        resolvedIntent: { type: string }
        outcome: { type: string }
        escalated: { type: boolean }
        occurredAt: { type: string, format: date-time }

    BroadcastRequest:
      type: object
      required: [title, templateName, segment]
      properties:
        title: { type: string }
        templateName: { type: string }
        segment:
          type: object
          properties:
            type: { type: string, enum: [OPTED_IN_ALL, OPTED_IN_BY_CATEGORY] }
        scheduledAt: { type: string, format: date-time }

    Broadcast:
      type: object
      properties:
        id: { type: string }
        title: { type: string }
        status: { type: string, enum: [SCHEDULED, SENDING, COMPLETED, FAILED] }
        scheduledAt: { type: string, format: date-time }

    KnowledgeDocumentRequest:
      type: object
      required: [title, content, category]
      properties:
        title: { type: string }
        content: { type: string }
        category: { type: string }

    KnowledgeDocument:
      type: object
      properties:
        id: { type: string }
        title: { type: string }
        category: { type: string }
        version: { type: string }
        ingestionStatus: { type: string, enum: [PENDING, COMPLETED, FAILED] }
        updatedAt: { type: string, format: date-time }

    WhatsAppWebhookPayload:
      type: object
      description: Mirrors Meta's WhatsApp Cloud API webhook schema — passed through largely as-is.
      properties:
        object: { type: string }
        entry:
          type: array
          items:
            type: object
            additionalProperties: true
```

---

## Notes on using this spec

- **Entity alignment**: every schema here maps 1:1 to an entity or projection from the ER model — `Order`/`OrderItem`/`OrderStatusHistoryEntry` mirror `Order`/`OrderItem`/`OrderStatusHistory`; `Item.price` and `OrderItem.unitPriceSnapshot` both stay in minor units (paise) per the ER model's design note.
- **Generating server stubs**: `openapi-generator-cli generate -i openapi.yaml -g spring -o ./generated` will scaffold Spring MVC controller interfaces + DTOs matching Spring Boot 4.1 — use `--additional-properties=useSpringBoot3=true,java17=true` (bump to your actual Java 25 target once the generator's template catalog supports it explicitly).
- **Internal-only paths** (`/internal/ai/*`, `/webhook/*`) should not be exposed through the public API gateway/Ingress — restrict via network policy even though they're documented here for completeness.
- **Validate the spec** with `swagger-cli validate openapi.yaml` before wiring it into CI, especially after any ER model change — a field rename in Postgres should be a deliberate matching edit here, not silently drift.

---

*Next natural step: a Flyway/Liquibase migration script to stand up the ER model schema in Postgres, or generated Spring Boot controller/DTO stubs from this spec. Let me know which to draft next.*
