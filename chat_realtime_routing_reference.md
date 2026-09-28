# Chat Real-Time Routing — Working Architecture

The delivery layer of the chat system, designed correctly: targeted routing via
Redis registry + per-server pub/sub channels (NOT broadcast, NOT topic-per-server).

**Diagram:** see `chat_realtime_routing.svg`.

---

## The three stores (each a distinct job — don't conflate them)

| Store | Job | Lifetime |
|-------|-----|----------|
| **messages DB** (+ S3 for files) | permanent history — "recent 50", multi-device sync | forever |
| **Redis registry** | who is connected to which gateway (`userId → gatewayId`) | while connected |
| **pending store** (Redis list / DB table, per user) | hold messages for OFFLINE users until reconnect | until delivered |

Plus the transport: **Redis Pub/Sub channel per gateway** (`channel:gw-5`) — the
cross-server routing hop.

---

## The flow

```
A sends on WebSocket → Gateway 1 (holds A's connection)
  1. PERSIST message to messages DB          ← always, for history/sync
  2. look up recipient B in Redis registry   → "B is on gw-5"
  3. is B online?
       ONLINE  → PUBLISH message to Redis channel:gw-5
                 → Gateway 5 (subscribed to its own channel) receives it
                 → pushes down B's WebSocket
                 → B acks → mark DELIVERED ; B reads → mark READ
       OFFLINE → write to B's pending store (per-user)
                 → send push notification (APNs/FCM)
  4. On reconnect: the gateway B lands on drains B's pending store
       SELECT ... WHERE user_id = B ORDER BY created_at → push each → delete on ack
```

---

## Why THIS topology (the key decisions)

**1. Targeted routing, not broadcast.**
- WRONG: each server subscribes to every other server's topic → every message hits
  every server → N-2 servers discard it → O(N) waste per message. At 100 servers,
  120K msg/s becomes ~12M wasted deliveries/s. Doesn't scale.
- RIGHT: each gateway subscribes ONLY to its own channel `channel:gw-{id}`. The sender
  looks up the recipient's gateway in the **Redis registry** and publishes ONLY to that
  channel. O(1) per message, no filtering. The registry is what turns broadcast into
  targeted delivery.

**2. Redis Pub/Sub for the routing hop, not Kafka.**
- The routing copy is ephemeral (the message is already durably in the messages DB),
  and needs LOW LATENCY. Redis pub/sub is faster and lighter than Kafka for this.
- Channels are ephemeral/self-managing as servers autoscale — a Kafka-topic-per-server
  would need dynamic create/destroy and orphans topics on crash.
- Kafka earns its place for DURABLE pipelines (persistence, analytics, cross-region
  replay) — not the real-time push.

**3. Offline → pending store, never back onto the bus.**
- Republishing an offline message to the same channel/topic = infinite loop
  (consume → still offline → republish → ...). Instead, park it in a per-user pending
  store (durable), and it rests there until reconnect. Transport vs storage.

**4. Persist ALWAYS (not just offline).**
- Online messages must be stored too — history, multi-device sync, and if the real-time
  push fails you haven't lost the message. Persist first, deliver second.

---

## Consumer / delivery logic (per gateway)

```java
// Gateway subscribes to ONLY its own channel: channel:gw-{thisServerId}
onMessage(message) {
    String recipient = message.getRecipientId();
    WebSocketSession session = localConnections.get(recipient);

    if (session != null && session.isOpen()) {
        session.send(message);          // push down the live WS
        // client ack → mark DELIVERED ; client read → mark READ
    } else {
        // recipient dropped between registry lookup and here → park it
        pendingStore.save(recipient, message);
        pushNotifier.notify(recipient);
    }
}
```

Sender side:
```java
persist(message);                                  // messages DB — always
String gw = registry.lookup(message.getRecipientId());
if (gw != null) {
    redis.publish("channel:" + gw, message);       // targeted → only that gateway
} else {
    pendingStore.save(message.getRecipientId(), message);  // offline → park
    pushNotifier.notify(message.getRecipientId());
}
```

---

## Guarantees to state
- **Ordering:** messages in a conversation stay ordered — key the persist/transport by
  `conversationId` so one conversation's messages don't scatter/reorder.
- **At-least-once + dedup:** delivery can retry → client dedups on `message_id` (idempotent).
- **Receipts:** SENT (persisted) → DELIVERED (acked by client) → READ (client reports read).

---

## pending_messages schema
```
pending_messages
  id PK, user_id FK, message_id FK, payload/ref, created_at
  → index (user_id, created_at)   "drain a user's backlog in order on reconnect"
```
(Or a Redis list per user: RPUSH pending:{user} ... / LRANGE on reconnect / DEL after.)
