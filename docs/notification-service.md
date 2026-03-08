# Notification Microservice

A production notification service built to replace a legacy monolithic notification system. Handles multi-channel delivery (in-app, push, SMS, email) with queueing, templating, preference management, and real-time socket delivery.

---

## Architecture Overview

```
                    ┌─────────────────────────┐
                    │     Incoming Events      │
                    │  (PubSub / REST / gRPC)  │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │      BullMQ Queue        │
                    │  (userNotification.queue) │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │     Notification         │
                    │       Worker             │
                    └──┬──────────────────┬───┘
                       │                  │
          ┌────────────▼──┐        ┌──────▼──────────┐
          │ Channel Router │        │  Event Logger   │
          └──┬─────────┬──┘        └─────────────────┘
             │         │
    ┌─────────▼─┐  ┌───▼────────┐
    │ Push/SMS/ │  │ Socket.IO  │
    │   Email   │  │  Real-time │
    │ Providers │  │   Push     │
    └───────────┘  └────────────┘
```

---

## Key Components

### Queue Layer — BullMQ
- All notification operations go through a BullMQ queue (`userNotification.queue`)
- Jobs are enqueued with UUID job IDs for deduplication
- Automatic retry with exponential backoff (configurable attempts + delay)
- Job retention: last 10 completed, last 5 failed — for debugging without memory bloat

```typescript
export const enqueueNotificationJob = async (
  jobType: string,
  payload: NotificationJobPayload,
  opts: Partial<NotificationJobOptions> = {},
) => {
  return notificationQueue.add(jobType, payload, {
    attempts: WORKER_JOB_DEFAULTS.ATTEMPTS,
    backoff: WORKER_JOB_DEFAULTS.BACKOFF,
    jobId: uuidv4(),
    removeOnComplete: { count: 10 },
    removeOnFail: { count: 5 },
    ...opts,
  })
}
```

### Services Layer
Each domain has a dedicated service class, injected via a shared context object:

| Service | Responsibility |
|---------|---------------|
| `NotificationEventService` | Core notification CRUD and dispatch |
| `NotificationTemplateService` | Template management with version history |
| `NotificationTemplateVersionService` | Template versioning and rollback |
| `NotificationPreferenceService` | Per-user channel preferences |
| `NotificationDeliveryService` | Delivery receipt tracking |
| `NotificationProviderService` | Provider abstraction (FCM, Bluenet SMS, etc.) |
| `DeviceTokenService` | FCM/APNs device token lifecycle |
| `PresenceService` | Real-time user online/offline state in Redis |
| `UserNotificationService` | User-facing notification state (read/unread) |
| `SchemaResolutionService` | Tenant schema lookup and caching |

### Real-time Delivery — Socket.IO Gateway
- Singleton `SocketGateway` wraps the Socket.IO server
- Waits for Redis readiness before initializing (prevents race conditions at startup)
- Presence tracked in Redis; users marked online/offline on connect/disconnect
- Auth middleware validates JWT before allowing socket connection

```typescript
export class SocketGateway {
  static getInstance(): SocketGateway { ... }

  public async init(server: http.Server, context: SharedContextI): Promise<void> {
    // Wait for Redis before accepting connections
    await new Promise<void>((resolve) => {
      if (context.redis.status === 'ready') return resolve()
      context.redis.once('ready', resolve)
    })
    // ...
  }
}
```

### Multi-channel Provider Abstraction
SMS provider follows an interface pattern, allowing swapping providers without touching business logic:

```typescript
interface SmsProvider {
  sendSms(params: { to: string; body: string }): Promise<void>
}

// Providers: BluenetProvider | TwilioProvider
// Selected at runtime via environment config
```

Retry logic with exponential backoff handles transient provider failures.

---

## Multi-tenancy

- Each tenant gets an isolated **PostgreSQL schema**
- `SchemaResolutionService` resolves the correct schema at request time
- Schema cache is preloaded on startup for fast lookups
- A `TenantConnectionManager` monkey-patch handles edge cases in schema switching during concurrent requests

```typescript
// On startup
await SharedContext.services.schemaResolutionService.preloadAllTenants()
```

---

## Additional Queues

| Queue | Purpose |
|-------|---------|
| `userNotification.queue` | Main notification processing |
| `resetSequence.queue` | Resets PostgreSQL sequences after bulk operations |
| `cloneSchema.queue` | Clones tenant schema for new tenant provisioning |

---

## Design Decisions

**Why BullMQ over direct DB writes?**
Notifications can spike. Queuing decouples the ingestion rate from processing rate, enables retry without client involvement, and gives visibility into job states.

**Why Socket.IO over WebSockets directly?**
Rooms, namespaces, and built-in reconnection logic. Real-time delivery is a secondary channel — queue is the source of truth.

**Why schema-per-tenant vs row-level isolation?**
Hard data isolation, simpler queries (no tenant_id filter everywhere), and independent schema migrations per tenant.
