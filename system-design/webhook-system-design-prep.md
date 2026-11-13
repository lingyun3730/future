# Webhook System Design (Interview Cheat Sheet)

**Question:** Design a webhook system for a payments platform. When a payment changes status (for example, captured), notify the merchant's server by HTTP.

Keep it simple. Say the one-liner first, then walk through the 6 steps.

> **One-liner:** At-least-once delivery. Save the event in the same transaction as the status change. Workers send it with a signature, retry with backoff, and isolate slow merchants.

---

## 1. Clarify (2 minutes)

Ask these:

- How many events per second? (1,000 average, 5,000 peak)
- Delivery guarantee and ordering? (at-least-once, no global order)
- How long do we retry? (a few days)
- Do merchants register URLs and pick event types? (yes)
- Signature? Delivery history and replay? (yes)
- Latency? (1–2 seconds normally)

Don't ask: which programming language.

---

## 2. The 6 steps

```text
1. Guarantee     at-least-once, merchant dedupes by event ID
2. Create event  same DB transaction as the status change
3. Deliver       workers send a signed POST, short timeout
4. Retry         exponential backoff + jitter, limited window
5. Isolate       per-merchant limits, so a slow merchant can't hurt others
6. Operate       history, replay, monitoring
```

### Step 1: Guarantee

If the merchant times out, we can't tell if it missed the event or if the response got lost. So we resend.

> "Delivery is at least once. Every event has a stable event ID, so the merchant deduplicates, and the effect happens once."

Exactly-once is not possible over a network.

### Step 2: Create the event

Problem: the payment becomes CAPTURED, then our service crashes before we create the event.

Fix: in **one transaction**, update the payment, insert the event, and insert one pending delivery for each endpoint that subscribes to this event type.

```text
BEGIN
  update payment status
  insert webhook_event            (event ID, payment ID, type, version, payload)
  insert webhook_delivery x N     (one per subscribed endpoint, status = PENDING)
COMMIT
```

> "In the same transaction as the status update, we save the event, and one pending delivery per endpoint. If the status is saved, the notification is saved."

Why not call the merchant directly, or publish to Kafka after the commit? Because the two writes can't be one transaction, so a crash in between loses the event.

### Step 3: Deliver

Stateless workers:

```text
loop:
  claim due deliveries   (status = PENDING and next_attempt_at <= now, SKIP LOCKED)
  send POST with signature, short timeout (5-10 seconds)
  2xx      -> mark SENT
  failure  -> retry later (step 4)
```

Signature: HMAC of **timestamp + body**, with a per-merchant secret, in a header. The merchant checks the signature and rejects old timestamps (so an old request can't be replayed). HTTPS only.

> "We sign the timestamp and the body with a per-merchant secret. The merchant verifies it and rejects old timestamps."

### Step 4: Retry

- Exponential backoff with jitter (so merchants aren't hit all at once when they recover).
- Retry for a limited window (for example 3 days), then mark FAILED and alert.
- The merchant can replay failed events.

### Step 5: Isolate

- Short timeout, so a slow merchant doesn't hold workers.
- Limit concurrency per merchant (or use separate queues).
- Circuit breaker: after many failures, pause that endpoint and tell the merchant.

> "A slow merchant can't take all the workers, because we limit concurrency per merchant."

### Step 6: Operate

- **Ordering:** no global order. Each event has an event ID and a version. The merchant compares versions or calls `GET /payments/{id}` for the latest state. Keep the payload small.
- **Scale:** add more workers; split by shard or merchant. At very large scale, use change data capture into a queue instead of polling.
- **Monitoring:** backlog size, age of the oldest event, success rate, per merchant.
- **Merchant tools:** delivery history, manual replay.

---

## 3. Two tables

```text
webhook_event      what happened (one row per event, never changes)
  event_id, payment_id, type, version, payload, created_at

webhook_delivery   who we send it to, and how it is going (one row per endpoint)
  delivery_id, event_id, endpoint_id, status, attempts, next_attempt_at
```

Why two tables? One event can go to many endpoints, and each endpoint retries on its own. The payload is stored once.

(There is also a small `webhook_endpoint` table: merchant, URL, secret, subscribed event types.)

If a merchant had thousands of endpoints, insert only the event, and let a fan-out worker create the deliveries.

---

## 4. Draw this

```text
Payment Service --(one transaction)--> DB: payment + event + deliveries
                                           |
                                    Delivery Workers (claim due deliveries)
                                           |  signed POST
                                       Merchant servers
```

---

## 5. One-minute answer

> "We guarantee at-least-once delivery. When a payment changes status, we save the event and one pending delivery per endpoint in the same database transaction, so we never lose a notification. Stateless workers claim due deliveries and send a signed request, with an HMAC over the timestamp and the body, and a short timeout. If it fails, we retry with exponential backoff and jitter, for a limited window. We limit concurrency per merchant, so one slow merchant can't affect the others. Events can arrive out of order, so each has an event ID and a version, and merchants can query the payment for the latest state. We scale by adding workers, and we monitor the backlog and the age of the oldest event. Merchants can see delivery history and replay failed events."

---

## 6. Likely follow-ups

| Question | Short answer |
| --- | --- |
| Exactly-once? | Not possible. At-least-once plus merchant dedupe by event ID. |
| Why not send directly or use Kafka after commit? | Two writes can't be one transaction. A crash in between loses the event. |
| Worker crashes after sending, before marking SENT? | It resends. That is why the merchant dedupes. |
| Merchant down for days? | Backoff, retry for the window, then FAILED and alert. Replay later. |
| One slow merchant? | Short timeout, per-merchant concurrency limit, circuit breaker. |
| Out-of-order events? | No global order. Event ID, version, and GET for the latest state. |
| How to secure it? | HMAC over timestamp and body, per-merchant secret, HTTPS, secret rotation. |
| How to scale? | More stateless workers, SKIP LOCKED, shard, change data capture at large scale. |
| What do you monitor? | Backlog, oldest event age, success rate, by merchant. |
| Thin or fat payload? | Thin, plus GET for the latest state. Avoids stale data and leaking sensitive data. |

---

## 7. Remember

**Five words:** Guarantee, Create, Deliver, Retry, Isolate.

**Common mistakes:**

- Promising exactly-once.
- Sending right after the commit instead of saving the event in the transaction.
- Retrying forever with a long timeout.
- One shared queue that blocks on a slow merchant.
- Relying only on HTTPS or an IP allowlist, with no signature.
- Forgetting history and replay for merchants.

**Link to your experience** (check it is accurate before you use it): the framework you use claims tasks with a lease and heartbeat, and a crashed worker's task is taken by another one. You added exponential backoff and a kill API for retry loops. Don't say you designed the framework; say you used it heavily and improved it.
