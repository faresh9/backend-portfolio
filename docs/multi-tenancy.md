# Multi-tenant Architecture

Schema-per-tenant isolation using PostgreSQL, with dynamic schema resolution at runtime and a Redis-backed cache for fast tenant lookups.

---

## Model: Schema-per-Tenant

Each tenant gets a dedicated PostgreSQL schema (e.g., `tenant_acme`, `tenant_globex`). Tables within a schema are identical in structure but fully isolated in data.

```
postgres database
├── public (shared tables: tenants registry, admin config)
├── tenant_acme
│   ├── users
│   ├── notifications
│   ├── conversations
│   └── ...
├── tenant_globex
│   ├── users
│   ├── notifications
│   └── ...
└── tenant_initech
    └── ...
```

**vs row-level isolation (adding `tenant_id` column everywhere):**
- No risk of missing a WHERE clause leaking cross-tenant data
- Simpler queries — no tenant filter on every query
- Independent schema migrations per tenant if needed
- Natural backup isolation

---

## Schema Resolution at Runtime

`SchemaResolutionService` resolves the correct schema for each incoming request:

1. Extract tenant identifier from request (header, JWT claim, or subdomain)
2. Look up schema name in Redis cache
3. Cache miss → query admin database, populate cache
4. Set PostgreSQL `search_path` to tenant schema for the request lifecycle

```typescript
// On startup — preload all tenants into cache
await schemaResolutionService.preloadAllTenants()
schemaResolutionService.displayCacheContents()

// Per request — resolve schema
const schema = await schemaResolutionService.resolveSchema(tenantId)
// Sets: SET search_path TO tenant_acme, public
```

---

## TenantConnectionManager

Handles edge cases in schema switching under concurrent requests:

- Race condition: two simultaneous requests for the same connection could switch `search_path` mid-query
- Fix applied as a monkey-patch at module load time (before any workers start) to prevent the race
- Ensures each request's DB operations run within the correct schema context for their full lifecycle

```typescript
// Applied at server startup, before workers initialize
import { applyTenantConnectionManagerFix } from '@helpers/fixTenantConnectionManager'
applyTenantConnectionManagerFix()
```

---

## New Tenant Provisioning

Handled via a dedicated BullMQ queue (`cloneSchema.queue`):

1. New tenant registered in admin database
2. `cloneSchema` job enqueued
3. Worker clones the base schema structure to new tenant schema
4. Cache updated with new tenant entry

Async provisioning means the API responds immediately; schema creation happens in the background.

---

## Shared Context Pattern

All services share a single context object injected at startup:

```typescript
interface SharedContextI {
  redis: Redis
  adminDb: DataSource       // admin database connection
  tenantDb: DataSource      // current-tenant connection
  services: {
    schemaResolutionService: SchemaResolutionService
    notificationService: NotificationEventService
    presenceService: PresenceService
    // ...
  }
}
```

Benefits:
- Single Redis connection shared across all services (not per-service)
- Services testable by injecting mock context
- No global singletons except the context itself

---

## Design Decisions

**Why cache schema lookups in Redis?**
Schema resolution happens on every request. A DB lookup per request would add ~5–20ms of latency. Redis lookup is <1ms.

**Why preload all tenants on startup?**
Eliminates cold cache misses on the first request per tenant. Acceptable memory cost for the tenant count at this scale.

**Why a cloneSchema queue vs synchronous provisioning?**
Schema creation can take 100–500ms depending on table count. Doing it synchronously in the registration request would cause timeouts under load and poor UX.
