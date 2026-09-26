# WhatsApp Grocery Ordering System — API Contract Definitions

## 0. Conventions

- Base URL (internal APIs): `https://api.internal.yourgrocery.com/v1`
- Auth: customer-facing endpoints are invoked internally by the Conversation Service (not exposed to the internet); Admin APIs require `Authorization: Bearer <JWT>` with role claim (`OWNER_ADMIN` / `STAFF`).
- Content type: `application/json` everywhere except the WhatsApp webhook, which follows Meta's schema.
- All timestamps: ISO 8601 UTC (`2026-09-12T10:15:30Z`).
- All monetary values: integer minor units (paise) to avoid float rounding — e.g., ₹149.50 → `14950`.

### 0.1 Standard error envelope
```json
{
  "error": {
    "code": "ORDER_NOT_CANCELLABLE",
    "message": "Order cannot be cancelled once packing has started.",
    "details": { "orderId": "ORD1234", "currentStatus": "PACKING" },
    "timestamp": "2026-09-12T10:15:30Z"
  }
}
```

| HTTP Status | Usage |
|---|---|
| 400 | Validation error (bad payload) |
| 401 / 403 | Auth failure / insufficient role |
| 404 | Resource not found |
| 409 | Conflict (e.g., cancelling a packed order, duplicate order submission) |
| 422 | Business rule violation (stock insufficient, invalid status transition) |
| 500 | Unexpected server error |

---

## 1. WhatsApp Webhook Contract (Meta → Us)

### 1.1 Webhook verification (GET) — one-time setup
```
GET /webhook/whatsapp?hub.mode=subscribe&hub.verify_token=<token>&hub.challenge=<challenge>
```
Response: return `hub.challenge` as plain text, HTTP 200, if `hub.verify_token` matches configured value.

### 1.2 Inbound message event (POST)
```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "WABA_ID",
      "changes": [
        {
          "value": {
            "messaging_product": "whatsapp",
            "metadata": { "display_phone_number": "919999999999", "phone_number_id": "PHONE_ID" },
            "contacts": [ { "profile": { "name": "Aarfi" }, "wa_id": "919812345678" } ],
            "messages": [
              {
                "from": "919812345678",
                "id": "wamid.HBgL...",
                "timestamp": "1757670930",
                "type": "text",
                "text": { "body": "Cancel my order" }
              }
            ]
          },
          "field": "messages"
        }
      ]
    }
  ]
}
```

Other `type` values we must handle: `interactive` (button_reply / list_reply), `image`, `location`. Example interactive reply:
```json
{
  "type": "interactive",
  "interactive": {
    "type": "list_reply",
    "list_reply": { "id": "cat_dairy", "title": "Dairy" }
  }
}
```

Our webhook controller must respond `HTTP 200` within a few seconds regardless of downstream processing outcome — publish to Kafka internally and return immediately.

---

## 2. Outbound WhatsApp Message Contracts (Us → Meta)

### 2.1 Send interactive list (category browse)
```json
POST https://graph.facebook.com/v20.0/{PHONE_ID}/messages
{
  "messaging_product": "whatsapp",
  "to": "919812345678",
  "type": "interactive",
  "interactive": {
    "type": "list",
    "header": { "type": "text", "text": "Shop Categories" },
    "body": { "text": "Choose a category to browse" },
    "action": {
      "button": "View Categories",
      "sections": [
        {
          "title": "Categories",
          "rows": [
            { "id": "cat_veg", "title": "Vegetables", "description": "Fresh daily" },
            { "id": "cat_dairy", "title": "Dairy" }
          ]
        }
      ]
    }
  }
}
```

### 2.2 Send order status update (Utility Template, outside service window)
```json
{
  "messaging_product": "whatsapp",
  "to": "919812345678",
  "type": "template",
  "template": {
    "name": "order_status_update",
    "language": { "code": "en" },
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "ORD1234" },
          { "type": "text", "text": "Packed" }
        ]
      }
    ]
  }
}
```

### 2.3 Send free-form service reply (inside 24h window)
```json
{
  "messaging_product": "whatsapp",
  "to": "919812345678",
  "type": "text",
  "text": { "body": "Your order #ORD1234 has been packed and is ready for pickup." }
}
```

---

## 3. Catalog API

### 3.1 List categories
```
GET /catalog/categories
```
```json
{
  "categories": [
    { "id": "cat_veg", "name": "Vegetables", "itemCount": 42, "imageUrl": "https://cdn.../veg.jpg" }
  ]
}
```

### 3.2 List items in a category
```
GET /catalog/categories/{categoryId}/items?page=0&size=20
```
```json
{
  "items": [
    {
      "id": "item_101",
      "name": "Tomato",
      "unit": "kg",
      "price": 3500,
      "imageUrl": "https://cdn.../tomato.jpg",
      "inStock": true,
      "stockQty": 40
    }
  ],
  "page": 0,
  "totalPages": 3
}
```

### 3.3 Search items
```
GET /catalog/items/search?q=milk
```
Same response shape as 3.2.

### 3.4 Admin: create/update item
```
POST /admin/catalog/items
PUT  /admin/catalog/items/{itemId}
```
```json
{
  "categoryId": "cat_dairy",
  "name": "Toned Milk 1L",
  "unit": "pack",
  "price": 6800,
  "stockQty": 100,
  "imageUrl": "https://cdn.../milk.jpg",
  "inStock": true
}
```

### 3.5 Admin: update stock / availability
```
PATCH /admin/catalog/items/{itemId}/stock
```
```json
{ "inStock": false, "reason": "OUT_OF_STOCK" }
```
Triggers `item.out_of_stock` Kafka event on `inStock: false`.

---

## 4. Cart API

### 4.1 Get cart
```
GET /cart/{customerId}
```
```json
{
  "customerId": "919812345678",
  "items": [
    { "itemId": "item_101", "name": "Tomato", "qty": 2, "unitPrice": 3500, "lineTotal": 7000 }
  ],
  "subtotal": 7000,
  "updatedAt": "2026-09-12T09:00:00Z"
}
```

### 4.2 Add/update item in cart
```
POST /cart/{customerId}/items
```
```json
{ "itemId": "item_101", "qty": 2 }
```
Response: updated cart (same shape as 4.1). `qty: 0` removes the item.

### 4.3 Clear cart
```
DELETE /cart/{customerId}
```

---

## 5. Order API

### 5.1 Place order
```
POST /orders
```
Request:
```json
{
  "customerId": "919812345678",
  "items": [ { "itemId": "item_101", "qty": 2, "unitPrice": 3500 } ],
  "fulfillment": { "type": "DELIVERY", "address": "12 Sector 62, Noida", "slot": "2026-09-12T18:00:00Z/2026-09-12T20:00:00Z" },
  "note": "Ring the bell twice",
  "idempotencyKey": "c7f1e2a0-..."
}
```
Response (201):
```json
{
  "orderId": "ORD1234",
  "status": "PLACED",
  "items": [ { "itemId": "item_101", "name": "Tomato", "qty": 2, "unitPrice": 3500, "lineTotal": 7000 } ],
  "subtotal": 7000,
  "placedAt": "2026-09-12T09:05:00Z"
}
```
`idempotencyKey` required — retried webhook deliveries must not create duplicate orders (NFR1). Duplicate key within 24h returns the original order (200, not 201).

### 5.2 Get order
```
GET /orders/{orderId}
```
```json
{
  "orderId": "ORD1234",
  "customerId": "919812345678",
  "status": "PACKING",
  "items": [ ... ],
  "subtotal": 7000,
  "statusHistory": [
    { "status": "PLACED", "at": "2026-09-12T09:05:00Z" },
    { "status": "CONFIRMED", "at": "2026-09-12T09:07:00Z" },
    { "status": "PACKING", "at": "2026-09-12T09:20:00Z" }
  ]
}
```

### 5.3 Update order status (admin)
```
PATCH /admin/orders/{orderId}/status
```
```json
{ "status": "PACKED", "updatedBy": "staff_002" }
```
Business rule enforced server-side: only forward transitions allowed in the defined sequence; invalid transition → `422 INVALID_STATUS_TRANSITION`.

### 5.4 Cancel order
```
POST /orders/{orderId}/cancel
```
```json
{ "cancelledBy": "CUSTOMER", "reason": "Changed my mind" }
```
- Allowed only while status is `PLACED` or `CONFIRMED`.
- Attempting to cancel a `PACKING`/`PACKED` order → `409 ORDER_NOT_CANCELLABLE` (see error envelope example in Section 0.1).

### 5.5 Order history
```
GET /orders?customerId=919812345678&page=0&size=10
```
```json
{
  "orders": [
    { "orderId": "ORD1230", "status": "DELIVERED", "subtotal": 12000, "placedAt": "2026-09-01T10:00:00Z" }
  ],
  "page": 0,
  "totalPages": 2
}
```

### 5.6 Reorder
```
POST /orders/{orderId}/reorder
```
Response: a new cart pre-populated from the referenced order's items (unavailable items flagged, not silently dropped):
```json
{
  "cart": { "customerId": "919812345678", "items": [ ... ], "subtotal": 7000 },
  "unavailableItems": [ { "itemId": "item_205", "name": "Capsicum", "reason": "OUT_OF_STOCK" } ]
}
```

---

## 6. AI Orchestration — Internal Contract

These aren't public REST endpoints; they're the internal contract between Conversation Service and AI Orchestration Service (in-process method call or internal HTTP call within the cluster).

### 6.1 Classify + respond
```
POST /internal/ai/query
```
```json
{
  "customerId": "919812345678",
  "message": "Cancel my order",
  "sessionContext": { "activeOrderId": "ORD1234" }
}
```
Response — informational case:
```json
{
  "type": "INFORMATIONAL",
  "answer": "Orders can be cancelled free of charge before packing begins. Once packing starts, cancellation isn't available.",
  "sourceDocId": "policy_cancellation_v3",
  "confidence": 0.91
}
```
Response — actionable case, pending confirmation:
```json
{
  "type": "ACTIONABLE_PENDING_CONFIRMATION",
  "intent": "CANCEL_ORDER",
  "targetOrderId": "ORD1234",
  "confirmationPrompt": "Cancel order #ORD1234? Reply YES to confirm."
}
```
Response — low confidence / escalate:
```json
{
  "type": "ESCALATE",
  "reason": "LOW_CONFIDENCE",
  "suggestedAction": "OFFER_HUMAN_HANDOFF"
}
```

### 6.2 Confirm pending action
```
POST /internal/ai/confirm
```
```json
{ "customerId": "919812345678", "intent": "CANCEL_ORDER", "targetOrderId": "ORD1234", "confirmed": true }
```
On `confirmed: true`, AI Orchestration calls Order Service's `POST /orders/{orderId}/cancel` (Section 5.4) — same code path as a manual cancel, no special AI-only logic.

### 6.3 `@Tool`-exposed function signatures (Spring AI)
```java
@Tool(description = "Cancel a customer's order if it hasn't started packing")
CancelResult cancelOrder(String orderId, String customerId);

@Tool(description = "Get current status of a customer's order")
OrderStatusResult getOrderStatus(String orderId);

@Tool(description = "Get a customer's recent order history")
List<OrderSummary> getOrderHistory(String customerId, int limit);
```

---

## 7. Notification Event Schemas (Kafka)

### 7.1 `order.placed`
```json
{
  "eventId": "evt_9001",
  "eventType": "order.placed",
  "orderId": "ORD1234",
  "customerId": "919812345678",
  "occurredAt": "2026-09-12T09:05:00Z"
}
```

### 7.2 `order.status.changed`
```json
{
  "eventId": "evt_9002",
  "eventType": "order.status.changed",
  "orderId": "ORD1234",
  "customerId": "919812345678",
  "previousStatus": "CONFIRMED",
  "newStatus": "PACKING",
  "occurredAt": "2026-09-12T09:20:00Z"
}
```

### 7.3 `item.out_of_stock`
```json
{
  "eventId": "evt_9003",
  "eventType": "item.out_of_stock",
  "itemId": "item_205",
  "itemName": "Capsicum",
  "affectedCustomerIds": ["919812345678", "919800011122"],
  "occurredAt": "2026-09-12T09:30:00Z"
}
```

### 7.4 `ai.conversation.logged`
```json
{
  "eventId": "evt_9004",
  "eventType": "ai.conversation.logged",
  "customerId": "919812345678",
  "inputMessage": "Cancel my order",
  "resolvedIntent": "CANCEL_ORDER",
  "outcome": "COMPLETED",
  "escalated": false,
  "occurredAt": "2026-09-12T09:05:10Z"
}
```

---

## 8. Admin API

### 8.1 New order queue
```
GET /admin/orders?status=PLACED,CONFIRMED&sort=placedAt,asc
```

### 8.2 Broadcast (marketing)
```
POST /admin/broadcasts
```
```json
{
  "title": "Fresh Mangoes Arrived!",
  "templateName": "new_arrival_promo",
  "segment": { "type": "OPTED_IN_ALL" },
  "scheduledAt": "2026-09-13T09:00:00Z"
}
```
Requires every recipient to have `marketingOptIn: true` on their customer record — enforced server-side, not just at UI level.

### 8.3 Knowledge base document upload
```
POST /admin/knowledge-documents
```
```json
{
  "title": "Order Cancellation Policy",
  "content": "Orders can be cancelled free of charge before packing begins...",
  "category": "POLICY"
}
```
Triggers async re-embedding into pgvector; response includes `documentId` and `ingestionStatus: PENDING`.

### 8.4 AI conversation review log
```
GET /admin/ai-conversations?escalated=true&from=2026-09-01&to=2026-09-12
```
```json
{
  "conversations": [
    {
      "customerId": "919812345678",
      "inputMessage": "can i get discount on bulk order",
      "outcome": "ESCALATED",
      "reason": "NO_KB_MATCH",
      "occurredAt": "2026-09-10T14:02:00Z"
    }
  ]
}
```
This is your running list of real customer questions the knowledge base doesn't cover yet.

---

*Next natural step: full ER data model (entity/field/relationship diagram) to back these contracts, or OpenAPI/Swagger spec generation for the REST endpoints above. Let me know which to draft next.*
