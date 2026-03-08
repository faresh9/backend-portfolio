# Real-time Chat Service

Production chat microservice handling real-time messaging, delivery/seen receipts, mention notifications, reactions, and message retry logic. Designed for correctness under concurrent conditions.

---

## Architecture Overview

```
Client A                    Chat API                    Client B
   │                           │                           │
   │── send message ──────────►│                           │
   │                           │── persist to DB           │
   │                           │── enqueue delivery job    │
   │◄─ ack ────────────────────│                           │
   │                           │── socket emit ───────────►│
   │                           │                           │── delivered event
   │                           │◄──────────────────────────│
   │                           │── mark delivered in DB    │
   │                           │                           │
   │                           │◄── seen event ────────────│
   │◄─ seen receipt ───────────│                           │
```

---

## Key Systems

### Message Delivery & Seen Status

Two-phase tracking: **delivered** (message reached device) and **seen** (user opened/viewed):

- Delivery confirmation sent as a socket event from the receiving client
- Seen status uses a **new record approach** — each seen event inserts a record rather than updating, avoiding race conditions under concurrent reads
- REST API fallback for delivery confirmation when socket is unavailable

The core problem this solved: duplicate delivery events and missed seen receipts when a user had multiple active sessions.

### Socket Event Architecture

Events are organized into modular handlers registered at startup:

```
socket/
  handlers/
    message.handler.ts       — send, edit, delete
    delivery.handler.ts      — delivered, seen confirmation
    conversation.handler.ts  — join/leave rooms
    mention.handler.ts       — mention events
    reaction.handler.ts      — emoji reactions
    presence.handler.ts      — online/offline
```

Each handler is registered via `registerSocketHandlers(io, context)` — clean separation, easy to extend.

### Mention System

Mentions are extracted from message content at write time:
- Parse `@username` patterns from message body
- Resolve to user IDs
- Emit targeted socket events to mentioned users
- Store mention records for notification backfill (offline users)

Edge cases handled: self-mentions, mentions of users not in conversation, deduplication of repeated mentions.

### Reaction System

Emoji reactions stored as records (user_id + message_id + emoji):
- Toggle behavior: reacting with same emoji removes it
- Aggregated counts returned with message payload
- Real-time reaction updates broadcast to conversation room

### Retry Logic

Message delivery retries handle network failures:
- Client tracks unacknowledged messages
- Exponential backoff on resend attempts
- Deduplication on the server side prevents duplicate inserts from retried sends

---

## Performance Highlights

- **Redis presence** — online/offline status in O(1) with Redis SET; no DB queries for presence checks
- **Room-based broadcasting** — Socket.IO rooms scoped per conversation; targeted delivery, no fan-out overhead
- **Queue offloading** — heavy operations (notification dispatch, mention processing) pushed to BullMQ; socket thread stays responsive
- **Validation at boundary** — Joi validation on all incoming socket payloads; rejects malformed messages early

---

## Security

- Socket connections authenticated via JWT middleware before joining any room
- Pin-protected conversations required a separate permission fix (critical security patch applied to production)
- All message IDs validated as numeric only (prevents injection via string IDs)

---

## Design Decisions

**Why Socket.IO rooms over broadcasting?**
Broadcasting to all connections doesn't scale. Rooms scope delivery to conversation participants only — correct semantics and more efficient.

**Why new-record for seen status vs update?**
Updates under concurrent reads (user opens chat on phone and tablet simultaneously) caused race conditions and lost seen events. Insert-based approach is append-only and naturally handles concurrency.

**Why separate delivery and seen phases?**
They represent different user actions. Conflating them causes UX bugs — marking something "seen" just because it was delivered, which is wrong.
