# Patterns & Engineering Decisions

Recurring patterns applied across these systems, with the reasoning behind each.

---

## Queue-first for Side Effects

Any operation that triggers a side effect (notification, email, schema clone, sequence reset) goes through a queue — never called inline in the request path.

**Why:** Decouples ingestion rate from processing rate. Enables retry without client involvement. Provides visibility into job state. Prevents request timeouts from slow downstream calls.

---

## Singleton Gateways

Socket.IO server and gRPC server are exposed as singletons:

```typescript
static getInstance(): SocketGateway {
  if (!SocketGateway.instance) {
    SocketGateway.instance = new SocketGateway()
  }
  return SocketGateway.instance
}
```

**Why:** These are stateful, resource-heavy objects. Creating multiple instances causes port conflicts, duplicate event registrations, and memory waste.

---

## Shared Context Injection

Services are not global singletons — they're injected via a shared context object created at startup.

```typescript
// Bad
const service = NotificationService.getInstance()

// Used instead
const { services } = SharedContext
services.notificationService.create(...)
```

**Why:** Makes dependencies explicit. Enables testing by swapping the context. Avoids circular import hell in large service graphs.

---

## Interface-based Provider Pattern

External providers (SMS, push, email) implement a common interface:

```typescript
interface SmsProvider {
  sendSms(params: SmsParams): Promise<SmsResult>
}
```

**Why:** Swap providers without touching business logic. Tested in isolation with mock implementations. Runtime selection via environment config.

---

## Append-only for Concurrent State

Seen receipts and delivery confirmations use insert-based records instead of UPDATE statements.

**Why:** Concurrent sessions (same user on phone + desktop) caused race conditions with UPDATE. Appending is inherently concurrent-safe. Aggregation happens at read time.

---

## Validation at System Boundaries

Joi validation applied on all incoming socket payloads and HTTP request bodies — not inside service methods.

**Why:** Service methods trust their inputs. Validation at the boundary catches bad data once, before it propagates. Inner validation would mean duplicating checks throughout the call stack.

---

## Exponential Backoff Retry

BullMQ jobs and HTTP forwarders both use exponential backoff:

```typescript
backoff: {
  type: 'exponential',
  delay: 1000,  // 1s, 2s, 4s, 8s...
}
```

**Why:** Immediate retries on transient failures hammer the downstream service. Backoff gives it time to recover. Jitter (random offset) prevents thundering herd after an outage.

---

## Readiness Gates

Services don't start accepting traffic until dependencies are ready:

```typescript
// Wait for Redis before accepting socket connections
await new Promise<void>((resolve) => {
  if (context.redis.status === 'ready') return resolve()
  context.redis.once('ready', resolve)
})
```

**Why:** Accepting requests before Redis/DB is ready causes a burst of failures that look like bugs. Readiness gates make startup deterministic.

---

## Cluster-based Parallelism

CPU-bound or IO-bound worker tasks run in forked worker processes via Node.js cluster:

```
Master ──► Worker 1
       ──► Worker 2
       ──► Worker N (one per CPU core)
```

**Why:** Node.js is single-threaded. A single process bottlenecks at one core. Clustering multiplies throughput horizontally on the same machine, with the master restarting crashed workers.

---

## Monkey-patch at Module Load Time

Critical fixes applied before any other module initializes:

```typescript
// server.ts — top of file, before imports that trigger the bug
import { applyTenantConnectionManagerFix } from '@helpers/fixTenantConnectionManager'
applyTenantConnectionManagerFix()
```

**Why:** When a race condition lives in a third-party or internal library, patching at module load time is the safest way to guarantee the fix is in place before any code that triggers the race runs. Documents the problem clearly at the entry point.
