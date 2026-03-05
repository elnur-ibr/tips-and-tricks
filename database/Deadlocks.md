# Deadlocks in Databases

A **deadlock** occurs when two or more transactions are waiting for each other to release locks, creating a cycle where **none of them can proceed**.

---

## How Does a Deadlock Happen?

Imagine two transactions running at the same time:

```
Transaction A                    Transaction B
─────────────                    ─────────────
1. Lock Row #1 ✅                1. Lock Row #2 ✅
2. Try to lock Row #2 ⏳ wait    2. Try to lock Row #1 ⏳ wait
   (held by B)                      (held by A)

💥 DEADLOCK — both waiting forever
```

Neither can continue because each holds a lock the other needs.

---

## Real-World Example in Laravel

### ❌ Deadlock-Prone Code

Two requests hit your app simultaneously — one transfers money from Account A→B, another from B→A:

```php
<?php

// Request 1: Transfer from Account A to Account B
DB::transaction(function () {
    $accountA = Account::lockForUpdate()->find(1); // Locks row 1
    sleep(1); // Simulates slow processing
    $accountB = Account::lockForUpdate()->find(2); // Tries to lock row 2 — WAITS

    $accountA->decrement('balance', 100);
    $accountB->increment('balance', 100);
});

// Request 2 (simultaneous): Transfer from Account B to Account A
DB::transaction(function () {
    $accountB = Account::lockForUpdate()->find(2); // Locks row 2
    sleep(1);
    $accountA = Account::lockForUpdate()->find(1); // Tries to lock row 1 — WAITS

    $accountB->decrement('balance', 50);
    $accountA->increment('balance', 50);
});

// 💥 DEADLOCK — Request 1 holds row 1, waits for row 2
//               Request 2 holds row 2, waits for row 1
```

### ✅ Fix: Always Lock in the Same Order

```php
<?php

// ALWAYS lock accounts in ascending ID order, regardless of transfer direction
DB::transaction(function () {
    $fromId = 2;
    $toId = 1;

    // Sort IDs so we always lock the lower ID first
    $ids = collect([$fromId, $toId])->sort()->values();

    $accounts = Account::lockForUpdate()
        ->whereIn('id', $ids)
        ->orderBy('id')
        ->get()
        ->keyBy('id');

    $accounts[$fromId]->decrement('balance', 50);
    $accounts[$toId]->increment('balance', 50);
});
```

Now both transactions lock rows in the **same order** (ID 1 first, then ID 2) — no cycle, no deadlock.

---

## Common Causes of Deadlocks

### 1. Inconsistent Lock Ordering
The most common cause. Different transactions lock the same rows but in **different order**.

### 2. Long-Running Transactions
The longer a transaction holds locks, the higher the chance another transaction will conflict.

```php
<?php

// ❌ Bad — holds locks while doing slow work
DB::transaction(function () {
    $order = Order::lockForUpdate()->find(1);
    $this->callExternalPaymentApi($order); // Slow HTTP call while holding lock!
    $order->update(['status' => 'paid']);
});

// ✅ Good — do slow work outside the transaction
$order = Order::find(1);
$paymentResult = $this->callExternalPaymentApi($order); // No lock held

DB::transaction(function () use ($order, $paymentResult) {
    $order = Order::lockForUpdate()->find($order->id); // Lock only when needed
    $order->update(['status' => $paymentResult->status]);
});
```

### 3. Missing Indexes
Without indexes, the database locks **entire table scans** instead of specific rows, dramatically increasing deadlock chances.

```sql
-- ❌ Without index: locks many rows during scan
UPDATE invoices SET status = 'paid' WHERE company_id = 5 AND invoice_number = 'INV-001';

-- ✅ With composite index: locks only the matching row
CREATE INDEX idx_invoices_company_number ON invoices(company_id, invoice_number);
```

### 4. Gap Locks (InnoDB-Specific)
InnoDB uses **gap locks** to prevent phantom reads in `REPEATABLE READ` isolation. These lock ranges of non-existent rows and can cause unexpected deadlocks.

```sql
-- Transaction A: locks gap for non-existent rows where status = 'pending'
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE;

-- Transaction B: tries to INSERT into the same gap
INSERT INTO orders (status) VALUES ('pending');
-- 💥 Blocked by Transaction A's gap lock
```

---

## How Databases Handle Deadlocks

Most databases (MySQL/InnoDB, PostgreSQL) have **automatic deadlock detection**:

1. The database detects the cycle
2. It picks a **"victim"** transaction (usually the one with least work done)
3. It **rolls back** the victim and throws an error
4. The other transaction continues

### MySQL Error
```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

### PostgreSQL Error
```
ERROR: deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5678; blocked by process 9012.
```

---

## How to Handle Deadlocks in Laravel

### Retry the Transaction

```php
<?php

// Laravel's retry helper — attempts the transaction up to 3 times
DB::transaction(function () {
    $account = Account::lockForUpdate()->find(1);
    $account->decrement('balance', 100);
}, attempts: 3); // Retries up to 3 times on deadlock
```

### Manual Retry with Logging

```php
<?php

use Illuminate\Database\DeadlockException;

$maxRetries = 3;

for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
    try {
        DB::transaction(function () {
            // Your transactional logic here
        });
        break; // Success — exit the loop
    } catch (DeadlockException $e) {
        if ($attempt === $maxRetries) {
            throw $e; // Give up after max retries
        }
        Log::warning("Deadlock detected, retrying (attempt {$attempt}/{$maxRetries})");
        usleep(100_000 * $attempt); // Back off: 100ms, 200ms, 300ms
    }
}
```

---

## Prevention Checklist

| Strategy | How |
|:---------|:----|
| **Consistent lock ordering** | Always lock rows in the same order (e.g., by ascending ID) |
| **Keep transactions short** | Do slow work (API calls, file I/O) outside transactions |
| **Add proper indexes** | Prevent full table scans that lock too many rows |
| **Use appropriate isolation level** | Consider `READ COMMITTED` instead of `REPEATABLE READ` to reduce gap locks |
| **Retry on deadlock** | Use `DB::transaction($callback, attempts: 3)` |
| **Avoid user interaction in transactions** | Never wait for user input while holding locks |

---

## Debugging Deadlocks in MySQL

```sql
-- Show the most recent deadlock details
SHOW ENGINE INNODB STATUS;

-- Look for the "LATEST DETECTED DEADLOCK" section
-- It shows exactly which transactions and queries caused the deadlock
```

This output tells you:
- Which transactions were involved
- Which locks each was holding
- Which locks each was waiting for
- Which transaction was rolled back as the victim
