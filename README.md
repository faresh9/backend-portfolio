# Backend Systems Portfolio

Production backend systems built as a founding engineer at an early-stage SaaS startup. Designed, architected, and shipped core infrastructure from scratch — replacing legacy systems, making key architectural decisions, and owning the full lifecycle from schema design to deployment.

**Stack:** TypeScript · Node.js · PostgreSQL · Redis · BullMQ · Socket.IO · gRPC · Docker

---

## Systems Built

| System | Description | Key Tech |
|--------|-------------|----------|
| [Notification Microservice](./docs/notification-service.md) | Multi-channel notification delivery with queueing, templating, and real-time push | BullMQ, Socket.IO, Redis, PostgreSQL |
| [Real-time Chat Service](./docs/realtime-chat.md) | Production chat with delivery/seen tracking, mentions, reactions | Socket.IO, Redis, PostgreSQL |
| [Event-driven PubSub](./docs/event-driven-pubsub.md) | gRPC publisher → Redis Streams → clustered worker fan-out | gRPC, Redis Streams, Node.js Cluster |
| [Multi-tenant Architecture](./docs/multi-tenancy.md) | Schema-per-tenant isolation with dynamic resolution and connection pooling | PostgreSQL schemas, Redis caching |

---

## Architectural Highlights

- **Multi-tenant isolation** via PostgreSQL schema-per-tenant with runtime schema resolution
- **Queue-based processing** using BullMQ with exponential backoff retry and job deduplication
- **Event-driven fan-out** via Redis Streams with a clustered subscriber worker pool
- **Real-time presence** tracked in Redis, pushed over Socket.IO
- **Service-to-service auth** using JWT with public keys from AWS Secrets Manager
- **Graceful degradation** — circuit breaker patterns, dead letter queues, retry limits

---

## Design Principles

1. **Separation of concerns** — each microservice owns its domain and database schema
2. **Async by default** — operations that can be queued, are queued
3. **Observable** — structured logging, job status tracking, delivery receipts
4. **Resilient** — retry logic with backoff at queue, HTTP, and gRPC layers
5. **Testable** — services injected via shared context; easy to mock at boundaries

---

## Docs

- [Notification Service Architecture](./docs/notification-service.md)
- [Real-time Chat Architecture](./docs/realtime-chat.md)
- [Event-driven PubSub Design](./docs/event-driven-pubsub.md)
- [Multi-tenancy Design](./docs/multi-tenancy.md)
- [Patterns & Decisions](./docs/patterns.md)
# backend-portfolio
