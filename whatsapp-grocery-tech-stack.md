# WhatsApp Grocery Ordering System — Technical Stack (Latest, as of Sep 2026)

## 1. Core Runtime & Framework

| Layer | Choice | Version (current as of Sep 2026) | Notes |
|---|---|---|---|
| Language | Java | **25 LTS** | Latest LTS, released Sep 2025, supported until 2032. Brings structured concurrency, further pattern-matching improvements, generational ZGC/Shenandoah gains — all useful for a chat-latency-sensitive service. |
| Framework | Spring Boot | **4.1.1** (Aug 21, 2026) | Built on Spring Framework 7.0.9, Jakarta EE 11 baseline. Supports Java 17–26, so you're safe running Java 25. |
| Build tool | Maven or Gradle | Gradle 8.x/9.x if you want faster incremental builds; Maven fine too given your existing familiarity | Either works with Boot 4.1 |
| API style | Spring WebFlux (reactive) or Spring MVC (virtual threads) | **Recommendation: Spring MVC + Java virtual threads** (`spring.threads.virtual.enabled=true`) | Virtual threads (stable since Java 21) give you reactive-like scalability for I/O-bound WhatsApp webhook traffic without WebFlux's steeper learning curve — a better fit than reactive here given the workload is mostly "wait on I/O, not heavy compute."

## 2. AI / RAG Layer

| Component | Choice | Version | Notes |
|---|---|---|---|
| AI framework | **Spring AI** | **2.0.1** (GA since June 12, 2026, patched Aug 21, 2026) | Now a full application platform, not just an integration library: Advisors API for composable tool-call/RAG/retry loops, native MCP (Model Context Protocol) support via MCP Java SDK 2.0.0. |
| LLM provider | OpenAI (cloud) + Ollama (local/self-hosted) — dual setup, matches what you already use | Use OpenAI for production-quality responses; Ollama for local dev/testing or cost-sensitive fallback | Spring AI's portable `ChatClient` abstraction lets you swap providers via config, not code |
| Vector store (RAG) | **PostgreSQL + pgvector** | `spring-ai-starter-vector-store-pgvector` | Best fit for you: you likely already run Postgres for catalog/orders, so no new infra to operate. Spring AI supports 12+ vector stores (Chroma, Qdrant, Redis, Milvus, etc.) if you outgrow pgvector, but for a single-shop RAG over policy documents, pgvector is more than sufficient and keeps ops simple. |
| Tool/function calling | Spring AI's "Spring beans as tools" pattern (`@Tool` methods) | Built into 2.0 | This is how `cancel_order`, `check_order_status`, `get_order_history` get exposed to the LLM as callable functions — the model picks the function + arguments, your existing Order Service executes it deterministically. |
| RAG pipeline | Spring AI ETL pipeline (DocumentReader → Splitter → Embedding → VectorStore) | Built into 2.0 | Feed it your cancellation/delivery/policy documents; re-run ingestion whenever admin edits a policy doc (FR30 from the requirements doc). |

## 3. Messaging & Integration Layer

| Component | Choice | Notes |
|---|---|---|
| WhatsApp channel | **WhatsApp Cloud API** (direct with Meta) | On-Premises API was sunset Oct 23, 2025 — Cloud API is the only path forward now. |
| Webhook handling | Spring Boot REST controller + async processing | Validate Meta's webhook signature, push event to Kafka immediately, return 200 fast — don't process synchronously inside the webhook call. |
| Event backbone | **Apache Kafka** | For `order.placed`, `order.status.changed`, `item.out_of_stock`, `ai.conversation.logged` — decouples notification sending from order processing and doubles as your audit trail. |
| Session/cart state | **Redis** | Fast, ephemeral, matches WhatsApp's turn-by-turn chat nature; TTL-based cart expiry is a natural fit. |

## 4. Data Layer

| Component | Choice | Notes |
|---|---|---|
| Primary database | **PostgreSQL 17.x** | Catalog, orders, order history, customers — also hosts pgvector for RAG, one less moving part. |
| Object storage | **AWS S3** | Item images, and the source policy documents that feed the RAG index. |
| Cache | Redis (shared with session state, or a separate instance for isolation) | Frequently-read catalog data. |

## 5. Infrastructure & Deployment

| Component | Choice | Notes |
|---|---|---|
| Containers | Docker, base image `eclipse-temurin:25-jre-alpine` | Matches Java 25 LTS. |
| Orchestration | **Kubernetes (EKS)** | Consistent with your existing stack. |
| CI/CD | GitHub Actions (or your existing pipeline) | Build → test → image push → deploy to EKS. |
| Observability | Spring Boot Actuator + Micrometer + OpenTelemetry | Spring Boot 4.1 improved OTel support out of the box — trace a request from webhook → intent classification → tool call → Kafka → notification, end to end. |
| Secrets | AWS Secrets Manager / Kubernetes Secrets | WhatsApp access tokens, OpenAI API keys — never in config files. |

## 6. Admin Web App

| Component | Choice | Notes |
|---|---|---|
| Backend | Same Spring Boot 4.1 service (separate module/context, or a dedicated internal service) | Order queue, catalog CRUD, knowledge-base document management, AI-conversation review log. |
| Frontend | React (or your team's preferred SPA framework) | Simple internal tool — doesn't need to be fancy, needs to be fast for staff during busy hours. |
| Auth | Spring Security with role-based access (Owner/Admin vs Staff) | Matches FR32 from the requirements doc. |

## 7. Why this stack, specifically

- **Java 25 + Spring Boot 4.1** is the current LTS-aligned pairing — you get long-term support (until 2032 for the JDK) instead of chasing a non-LTS release.
- **Spring AI 2.0** reaching GA in June 2026 is what makes "AI in Java, not Python" genuinely production-viable now — tool-calling is a first-class advisor-chain concept, not a bolt-on, which matches exactly the "AI classifies, backend executes" split from your requirements doc.
- **pgvector over a dedicated vector DB** (Qdrant/Milvus/Pinecone) is the pragmatic call for your scale: one less service to operate, one less thing to secure, and Postgres already needs to be there for orders.
- **Virtual threads over WebFlux**: your workload (webhook in → maybe an LLM call → maybe a DB call → webhook reply) is I/O-heavy but not throughput-extreme for a single shop; virtual threads give you most of the scalability benefit with normal blocking-style code, which is easier to reason about and debug than reactive streams.

---

## 8. Suggested `pom.xml` starter dependencies (illustrative, not exhaustive)

```xml
<properties>
    <java.version>25</java.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-ollama</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

Check the Spring AI BOM version alignment against 2.0.1 when you scaffold the project — Spring AI's milestone/RC cadence moved fast through early 2026, so pin exact versions rather than using `+`/latest ranges.

---

*Next natural step: entity/data model for Order, OrderItem, Item, Category, Customer, Conversation, Intent, KnowledgeDocument — or a package/module structure for the Spring Boot codebase itself. Let me know which you want next.*
