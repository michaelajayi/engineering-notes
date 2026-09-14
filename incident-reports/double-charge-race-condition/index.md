---
title: The Order That Got Charged Twice - A Race Condition Post-Mortem
published: "false"
tags:
concepts: "[race-condition, check-then-act, optimistic-concurrency, atomic-updates, prisma, postgresql-isolation]"
---
## The Symptom ##
Over the past 48 hours, the support team has received 6 tickets from customers who were charged twice for a single order. Refunds have been manually issued, but engineering hasn’t identified the cause. 

Customer `cus_8841` placed order `ord_20394` for `$87.50`, four seconds apart. 

## Begin Investigation. ##
The first logical instinct is to query the database, our source of truth in this case. 

### Database Query Result (run against `orders` and `payments` table for `ord_20394` ) ###


**orders**

| id        | status | amount | customer_id | updated_at              |
| --------- | ------ | ------ | ----------- | ----------------------- |
| ord_20394 | paid   | 87.50  | cus_8841    | 2026-09-09 14:22:31.812 |

**payments**

| id         | order_id  | amount | status    | provider_charge_id | created_at              |
| ---------- | --------- | ------ | --------- | ------------------ | ----------------------- |
| pay_9a31f2 | ord_20394 | 87.50  | succeeded | ch_7f0a3c          | 2026-09-09 14:22:27.501 |
| pay_9a31f9 | ord_20394 | 87.50  | succeeded | ch_7f0a41          | 2026-09-09 14:22:31.809 |

### Facts From The Evidence. ###
- there is indeed a single order recorded for that customer. 
- two distinct payment records, `r1` and `r2` was recorded for the single order, four seconds apart. 
- the payment records for both `r1` and `r2` resolved successfully. 

### Tracing the Execution Flow. ###
To make sense of this, I went straight to the call site of the payment provider. 

```typescript
async processPayment(orderId: string): Promise<PaymentResult> {
  const order = await this.ordersRepo.findById(orderId);

  if (!order) throw new NotFoundException(...);
  if (order.status !== 'pending') throw new ConflictException(...);

  const charge = await this.paymentProvider.charge({
    amount: order.amount,
    customerId: order.customerId,
  });

  if (charge.status !== 'succeeded') throw new PaymentFailedException(...);

  await this.paymentsRepo.create({ orderId: order.id, ...  });
  await this.ordersRepo.updateStatus(order.id, 'paid');

  return { orderId: order.id, chargeId: charge.id, status: 'paid' };
}
```

Tracing the path of execution from the code, the call to the payment provider for charge is guarded conditionally by a state column, `status`. In other words, an `order` must be in a `pending` state before a charge is allowed. 

**Logical assumption based on `processPayment`**
In both instances of the distinct double charges:
- `r1` and `r2` passed the conditional state check, `state='pending'` then proceeded to charge. 
-  `ch_7f0a3c` and `ch_7f0a41` from the payment provider resolved successfully. 
- `r1` and `r2`  independently created payment records and updated order, `status='paid'`.

To clarify these assumption and trace the actual execution path within `r1` and `r2` lifecycle, one piece of evidence is crucial, the application log at the http layer. This log contains a detailed p, sufficient to draw a timeline. 

**Request Logs**
```
14:22:27.410  req_id=r1-88c2  POST /orders/ord_20394/pay
14:22:27.418  req_id=r2-88c9  POST /orders/ord_20394/pay

14:22:27.421  r1  findById -> status=pending
14:22:27.427  r2  findById -> status=pending

14:22:27.428  r1  charge() called
14:22:27.431  r2  charge() called

14:22:27.501  r1  charge succeeded (ch_7f0a3c)
14:22:27.510  r1  updateStatus -> paid
14:22:27.512  r1  200 OK (102ms)

14:22:29.431  r2  charge attempt 1 failed: ECONNABORTED, retrying
14:22:31.802  r2  charge attempt 2 succeeded (ch_7f0a41)
14:22:31.809  r2  updateStatus -> paid
14:22:31.812  r2  200 OK (4394ms)
``` 

**The Race (Visualised)**
``` mermaid
sequenceDiagram
    participant R1 as r1-88c2
    participant DB as orders (status)
    participant P as Payment Provider
    participant R2 as r2-88c9

	R1->>DB: findById → pending (14:22:27.421)
    R2->>DB: findById → pending (14:22:27.427)
    Note over R1,R2: both read pending before either writes

    R1->>P: charge() (14:22:27.428)
    R2->>P: charge() (14:22:27.431)

    P-->>R1: succeeded, ch_7f0a3c (14:22:27.501)
    R1->>DB: updateStatus → paid (14:22:27.510)

    P--xR2: timeout, ECONNABORTED (14:22:29.431)
    R2->>P: charge() retry
    P-->>R2: succeeded, ch_7f0a41 (14:22:31.802)
    R2->>DB: updateStatus → paid (14:22:31.809)
    Note over R2,DB: no check catches this - order already paid
``` 

**The Trace (Execution Flow)**
- `r1` reads order, `pending`.
- `r2` reads order, `pending` (8 milliseconds later, before `r1` writes anything). 
- `r1` and `r2` hold their in-memory copy of `status='pending'` within their independent execution contexts (each unaware of the other one). 
- both proceed to `charge().` 
- `r1` finishes first, marked order paid. 
- `r2` (still mid-flight) retries its timed-out charge and resolved successfully (has no way of knowing `r1` already wrote paid). 
- `r2` finishes, resolved successfully, marked order paid again (no-op overwrite, since nothing enforces that the row was still pending at the write time either). 

**The Underlying Principle**
A plain `SELECT` (the mechanism behind `findById`) is a snapshot read. It returns the row's value at the time of read. It does not guarantee that the value will be the same by the time you come back to act on it. It also doesn't prevent other transactions from reads and writes.

This is a classic case of check-then-act race condition. A bug pattern where a condition is read in one step, and acted on in a separate step. There's no promise the condition still holds by the time of the act. 

**The Fix.**
There are multiple design choices to address this check-and-act problem. Two common ones are pessimistic lock and optimistic lock. I took the optimistic concurrency control via a state column approach. 

**The Trade-Off.**
- optimistic concurrency is cheap to implement,
- no waiting or lock held during the call to `charge()`; the losing request finds out immediately (`count === 0`), fails fast and return early rather than blocking,
- doesn't hold database connection, this matters under load and across retrying http calls. 

**Implementing Fix.**

``` typescript
async claimForPayment(id: string): Promise<{ claimed: boolean }> {
  const result = await this.prisma.order.updateMany({
    where: { id, status: 'pending' },
    data: { status: 'processing' },
  });
  return { claimed: result.count === 1 };
}
``` 

`updateMany()` lets Postgres evaluate the WHERE conditions (`id=x`, `status='pending'`). This operation combines the conditional check and the update into a single atomic write. It doesn't throw on zero matches. Rather, it returns the count matching how many rows was touched/modified.

**Applying The Fix.**
``` typescript
async processPayment(orderId: string): Promise<PaymentResult> {
  const order = await this.ordersRepo.findById(orderId);

  if (!order) {
    throw new NotFoundException(`Order ${orderId} not found`);
  }

  const { claimed } = await this.ordersRepo.claimForPayment(orderId);

  if (!claimed) {
    throw new ConflictException(
      `Order ${orderId} is not in a payable state`,
    );
  }

  let charge: ChargeResult;

  try {
    charge = await this.paymentProvider.charge({ ... });
  } catch (err) {
    // network failure - charge() never resolved to a value
    await this.paymentsRepo.create({ orderId: order.id, status: 'failed', failureReason: err.message, ... });
    await this.ordersRepo.updateStatus(order.id, 'pending');
    throw err;
  }

  if (charge.status !== 'succeeded') {
    // business decline - charge() resolved, but not successfully
    await this.paymentsRepo.create({ orderId: order.id, status: 'failed', failureReason: charge.failureReason, ... });
    await this.ordersRepo.updateStatus(order.id, 'pending');
    throw new PaymentFailedException(charge.failureReason);
  }

  await this.paymentsRepo.create({ orderId: order.id, status: 'succeeded', ... });
  await this.ordersRepo.updateStatus(order.id, 'paid');

  return { orderId: order.id, chargeId: charge.id, status: 'paid' };
}
```

After the first caller flips the state, the row no longer satisfy the condition. The second concurrent caller's `updateMany()` on the same row matches zero count. That is the stop signal that prevents charge from being called subsequently.

After a charge (`state='processing`') resolves to failed, it is important to rollback from processing state to pending state. Without this, a failed charge would leave the order, permanently stuck in processing and unpayable. 

**Conclusion**
The logs confirmed two distinct requests from the same client IP. The evidence wasn't sufficient enough to establish why two results came in concurrently from the client. Some likely possibilities:
- double-click with no submit-button disabling,
- a client-side retry implementation,
- a proxy-level retry, etcetera. 

No frontend code, browser log, or session recording was part of this investigation. Clients can send concurrent requests for the same business intent. The backend's job is not to prevent these requests. Rather, to make it safe for concurrent requests to arrive and be processed. Ensuring only one can ever win the check-then-act race.  
