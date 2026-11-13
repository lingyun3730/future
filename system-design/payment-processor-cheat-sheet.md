# Payment Processor – System Design Cheat Sheet

Short version for speaking. Full details: `payment-processor-design-prep.md`.

## One-liner
> "Money must never be lost or duplicated. So every request is idempotent, the result of a bank call can be unknown, and recovery plus reconciliation make sure we always end in a correct state."

## 1. Clarify
- Which flows: authorize, capture, refund? Partial refunds?
- Scale: transactions per second, peak?
- Sync API: what is the latency budget? How do merchants get late results (webhook)?
- Multiple banks / acquirers? Multi-currency?
- Correctness vs availability: if unsure, which one wins? (Correctness.)

## 2. API (4 endpoints)
```
POST /v1/payments/authorizations        Idempotency-Key header
POST /v1/payments/{id}/captures         one full capture per authorization
POST /v1/payments/{id}/refunds          partial and multiple refunds
GET  /v1/payments/{id}
```
- `{id}` = `processorTransactionId`, created by us at authorization.
- Declined → `402` with an id. Result not known yet → `202` (final result by webhook / GET).

## 3. Three core tables
| Table | Role |
|---|---|
| `transaction` | Current state + counters (authorized, captured, refunded, refund_pending). The lock point. |
| `operation` | One row per call: type, status, bank idempotency key, merchant idempotency key, request hash, response. `UNIQUE(merchant_id, type, idempotency_key)`. |
| `webhook_outbox` | Events for the webhook worker. Only when the merchant did not get a final sync answer. |

- `operation.status`: PENDING / UNKNOWN / SUCCESS / FAILED (updated in place).
- `transaction.status`: AUTHORIZING → AUTHORIZED → CAPTURING → CAPTURED, plus DECLINED, EXPIRED.
- The PENDING `operation` row is itself the recovery task. No extra queue.

## 4. Request flow (authorize)
```
Merchant -> Gateway (auth, rate limit)
         -> Payment Service
              1. Validate
              2. DB transaction: INSERT operation (PENDING, unique key)
                                 + INSERT/UPDATE transaction
                 (duplicate key? -> return the stored response)
              3. Call bank via Bank Adapter (no DB transaction held)
              4. DB transaction: update operation + transaction (+ outbox if async)
              5. Return result
```
Say: "Insert the operation first. The unique key decides who wins, no check-then-insert."

## 5. Key points (the part to remember)
1. **Idempotency:** unique key on `(merchant, type, Idempotency-Key)`. Same key + same body → same response. Same key + different body → error.
2. **Timeout = unknown, never failed.** Mark UNKNOWN, recovery asks the bank.
3. **Bank idempotency:** `operationId` is sent to the bank as its idempotency key. Our DB key protects only us, not the bank.
4. **No DB transaction during the bank call.** It holds locks and connections for too long.
5. **Recovery worker:** scan old PENDING / UNKNOWN rows, claim with `SKIP LOCKED` or a lease, query the bank first, then retry with the same bank key.
6. **Don't switch to another bank when the result is unknown.** You could charge twice.
7. **Reconciliation** with bank files is the final safety net.

## 6. Refund concurrency
One conditional UPDATE on the transaction row, inside the same DB transaction as the PENDING refund operation:
```sql
UPDATE transaction
SET refund_pending_amount = refund_pending_amount + :amt
WHERE id = :id AND status = 'CAPTURED'
  AND captured_amount - refunded_amount - refund_pending_amount >= :amt;
-- 0 rows updated -> roll back, return "refund too large"
```
- UPDATE takes a row lock until commit, so concurrent refunds queue up and re-check the latest value.
- Don't do plain `SELECT` then `UPDATE`.
- Bank success → `refunded += amt, pending -= amt`. Bank failure → `pending -= amt`.

## 7. Bank Adapter
A translation layer per bank, with one unified interface: `authorize / capture / refund / queryStatus`.
- Maps bank responses to three outcomes: **success / definite failure / unknown**.
- Owns timeout, connection pool, circuit breaker.
- Does not route and does not decide retries. The payment service decides.

## 8. Webhooks
- Sync success: no outbox row, merchant already has the answer.
- Async or late result: update status + insert outbox in **one DB transaction**.
- Webhook worker polls the outbox `(status, next_attempt_at)`, retries with backoff + jitter.
- At-least-once → merchants dedupe by event id. (See `webhook-system-design-prep.md`.)

## 9. Scale
- Shard by `processorTransactionId` (shard id is inside the id). **Never one merchant per shard.**
- Authorization routes by `hash(merchant_id, idempotency_key)`, so duplicates meet in the same place.
- Per-merchant rate limits at the gateway (see `rate-limiter-system-design-prep.md`).
- Merchant queries: from a read store fed by CDC, not the primary DB.
- Whale merchant: dedicated shard group.

## One-minute answer
"I clarify flows, scale and correctness first. The API has authorize, capture, refund and get, with an idempotency key. In the DB I keep three tables: transaction for current state and the lock point, operation for every call with a unique idempotency key, and a webhook outbox. For each request, I insert the operation as PENDING with the unique key and update the transaction in one DB transaction, then call the bank outside any transaction. A timeout means unknown, never failed, so a recovery worker asks the bank and retries with the same bank idempotency key. Refunds use a conditional update on the transaction row so concurrent refunds can't exceed the captured amount. I shard by transaction id, and reconciliation is the final safety net."

## Follow-ups
| Question | Short answer |
|---|---|
| Duplicate request while the first is in flight? | Unique key conflict → return the stored (or PENDING/202) response. |
| Bank timed out? | UNKNOWN → recovery queries the bank, then retries with the same key. |
| Two concurrent refunds? | Conditional UPDATE with a row lock; the second sees the new value. |
| Why not retry on another bank? | The first call may have succeeded → double charge. |
| Why no distributed transaction? | Banks don't support it. Use idempotency + recovery + reconciliation. |
| Capture after authorization expired? | Check expiry at capture time, return error. |
| Webhook lost? | Retries + merchant can GET. |
| Exactly-once? | No. At-least-once + idempotency = effectively once. |
| Where do things go wrong? | Unknown results, double refunds, lost events → covered by recovery and reconciliation. |

## Remember
- Correctness over availability.
- Timeout = unknown, never failed.
- Unique key decides; no check-then-insert.
- No DB transaction during the bank call.
- Same bank key on every retry.
- Reconciliation is the final safety net.
