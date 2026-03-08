# Event-driven PubSub System

A production event bus connecting all microservices via an async, durable publish/subscribe model. Built on gRPC (publisher) and Redis Streams (transport) with a clustered worker pool (subscriber).

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Publisher Component                      │
│  ┌─────────────────────┐      ┌─────────────────────────┐  │
│  │  gRPC Server :50051  │─────►│     Redis Streams        │  │
│  │  Health: HTTP :8080  │      │                         │  │
│  └─────────────────────┘      └────────────┬────────────┘  │
└────────────────────────────────────────────│────────────────┘
                                             │
┌────────────────────────────────────────────▼────────────────┐
│                    Subscriber Component                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Cluster Manager                     │   │
│  │             (Master Process)                         │   │
│  └──────────┬──────────────┬──────────────┬────────────┘   │
│             │              │              │                  │
│    ┌────────▼───┐  ┌───────▼───┐  ┌──────▼────┐           │
│    │  Worker 1  │  │  Worker 2  │  │  Worker N  │           │
│    │StreamWorker│  │StreamWorker│  │StreamWorker│           │
│    └────────┬───┘  └───────┬───┘  └──────┬────┘           │
│             │              │              │                  │
│    ┌────────▼──────────────▼──────────────▼────────────┐   │
│    │              EventForwarderFactory                  │   │
│    └──────────────────────┬─────────────────────────────┘   │
└───────────────────────────│─────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   Admin API           Chat API         Notification API
   :7000               :8001             :3000
```

---

## Components

### Publisher — gRPC Server

Exposes two RPC methods:

```protobuf
service EventService {
  rpc PublishEvent(PublishEventRequest) returns (PublishEventResponse);
  rpc GetPublishStatus(GetPublishStatusRequest) returns (GetPublishStatusResponse);
}
```

- Receives events from any microservice
- Writes to Redis Streams — durable, ordered, consumer-group aware
- Health check on HTTP port for orchestration readiness probes
- S2S auth via JWT; public keys pulled from AWS Secrets Manager

### Topic Registry

Centralized singleton defining all event types:

| Topic | Events |
|-------|--------|
| `UserTopic` | user.created, user.updated, user.deleted |
| `ChatTopic` | message.sent, conversation.created, reaction.added |
| `NotificationTopic` | notification.created, notification.read |
| `OrganizationTopic` | org.created, member.added, member.removed |
| `TeamTopic` | team.created, team.updated |

Type-safe event definitions prevent invalid event names at the publisher side.

### Subscriber — Clustered Worker Pool

Master process forks N worker processes (configurable). Each worker:
1. Connects to Redis Streams as a consumer group member
2. Reads events in batches
3. Routes to the correct `EventForwarder` via `EventForwarderFactory`
4. Forwards to the target microservice via HTTP
5. Acknowledges the Redis Stream message on success

```
StreamWorker (per worker process)
  └── EventForwarderFactory
        ├── AdminEventForwarder      → Admin API
        ├── AuthEventForwarder       → Auth API
        ├── ChatEventForwarder       → Chat API
        ├── NotificationEventForwarder → Notification API
        └── VertoEventForwarder      → Verto Service
```

Worker isolation means one slow or crashing worker doesn't block others. The cluster manager restarts failed workers automatically.

---

## Advanced Patterns (Designed & Documented)

### Saga Pattern
For distributed transactions spanning multiple microservices:
- Choreography-based sagas with compensating transactions
- If step N fails, steps 1..N-1 are rolled back via inverse events
- Prevents partial failures leaving data inconsistent across services

### SLA/SLO Monitoring
- Prometheus metrics on queue depth, processing latency, throughput
- Little's Law applied: `L = λ × W` (queue length = arrival rate × wait time)
- Alerts when queue depth exceeds capacity planning thresholds

### Resilience Patterns
- **Circuit Breaker** — stops forwarding to a downstream service if error rate exceeds threshold; half-open probe to detect recovery
- **Bulkhead** — separate worker pools per downstream service; one overloaded service doesn't starve others
- **Rate Limiting** — per-service forwarding rate cap
- **Backpressure** — consumer slows Redis reads when downstream queue is saturated

---

## Design Decisions

**Why gRPC for publishing vs REST?**
Strongly typed contracts via protobuf, binary transport (faster than JSON), and bidirectional streaming capability for future use. All microservices can publish without knowing how the bus works internally.

**Why Redis Streams vs Kafka?**
Redis Streams provide consumer groups, message acknowledgment, and at-least-once delivery without the operational overhead of a Kafka cluster. Appropriate scale for a small startup — can migrate later.

**Why worker cluster vs single process?**
Single process becomes a bottleneck under load and a single point of failure. Forked workers use all CPU cores and the cluster manager restarts crashed workers without manual intervention.

**Why HTTP forwarding vs direct DB writes in the subscriber?**
Each microservice owns its own data. The subscriber routing to HTTP preserves service autonomy — the notification service handles notification logic, not the subscriber.
