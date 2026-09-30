# Implementation Plan: Process Deduction Endpoint

Sep 25, 2026 · @Ahmed

## 1. Summary

The endpoint makes one deduction's SELL_OUT expense events match its current rule and the store's current sales. The main decisions:

- **Apply only the difference; an upsert alone can't remove stale rows.** Compute the correct set in SQL, then remove stale rows, update changed rows and insert missing ones.
- **An advisory lock per store,** so calls for the same deduction, or for overlapping fees, run one at a time.
- **An analyzed temp table** holds the correct set, so the sales join runs once and later queries are planned on real row counts.
- **One transaction per call,** so a failure leaves the old events untouched and a retry is safe.
- **Constant app memory at 100,000+ rows.** No sales or event rows reach Node (section 9).

## 2. The problem behind the brief

The cost is one multiplication; the hard part is keeping stored events correct as inputs change:

- **Rules get edited.** If an end date moves from 31 to 15 January, the events for 16 to 31 January must go. An upsert would leave them, silently overstating the vendor's costs.
- **Calls get repeated.** Retries and double clicks must never duplicate events (Rule 4).
- **The table is shared.** `vendor.expense_events` also holds chargebacks and other fees; the endpoint may only touch its own rows.
- **The request carries two IDs,** so the deduction must belong to the store.

Requirement: after every call, events equal what the current rule and sales imply.

## 3. Starting point: the repo and the assumptions

I started from the existing repo:

- `aggregator-job` is Express and TypeScript with Sequelize and Winston. The endpoint is a TODO in vendors.routes.ts; I'd move the logic into a testable service module. The Zod schema, error handler and route response don't match the brief yet (section 10).
- `vendor.product_sales` is a hypertable on `report_date` with only a `(store_id)` index besides the default time index.
- `vendor.expense_events` is a hypertable on `expense_date` with primary key `(store_id, source_system_id, expense_date, expense_type, expense_subtype)`. V1.3 added a nullable `deduction_id`, outside the key and without a foreign key, indexed as `(store_id, deduction_id)`. It's how a deduction finds its own events; chargebacks have none.
- `deduction_value` is DECIMAL(13,2), a fraction (0.10 for 10%). IDs are BIGINT.

**The primary key relies on a per-product `source_system_id`.** With no `product_sku` or `deduction_id` in the key, events get one row per product per day only because they use the product's `source_system_id` (such as `MERCH001_SKU-1-001`). Chargebacks use the store's `SYS001`; SELL_OUT events doing the same would put every product on a given day under one key. The missing `deduction_id` causes the overlap problem in section 7.

**The date rule deviates from the brief.** Rule 5 says `report_date` between the effective dates, inclusive. Those columns carry a time of day, so I compare UTC calendar days, both ends inclusive. The catch: a "1 to 31 January" deduction entered at midnight +08:00 is stored as 31 December 16:00 to 30 January 16:00 UTC, so it gains 31 December and loses 31 January. The alternative is the literal `ps.report_date BETWEEN d.effective_date_from AND d.effective_date_to`, which is right when end dates are stored as end of day.

The first five assumptions are open questions I'd confirm with the team; the rest are cheap to change:

| Assumption | If wrong |
| --- | --- |
| Days are UTC calendar days (above) | Use the literal `BETWEEN`, or the marketplace's time zone |
| cost = ROUND(`shipped_revenue` × `deduction_value`, 2), returns not netted; a negative correction gives a negative cost | Use `shipped_revenue_total`, or subtract returns; floor cost at 0 if negatives aren't allowed |
| Events carry the product's `product_sku` and `source_system_id` | `seller_sku` can replace `product_sku`; `source_system_id` must stay per product |
| Same-type deductions never overlap for a store and dates | Section 12 has the migration |
| A non-NET_RECEIPTS deduction returns 400, even with SELL_OUT events left from before a basis change | Remove those events first |
| One event per sales row, dated with its `report_date`, including $0 sales | Aggregate, or skip zero rows |
| `expense_status` = CONFIRMED, as in the seed | Use ESTIMATED |
| Existing SELL_OUT events carry their `deduction_id`, and no other system writes SELL_OUT events | Q5 blocks untagged rows; backfill first. An outside writer would also need the store lock |
| IDs are digit strings within BIGINT range, which rejects the brief's placeholder "store123" | Look IDs up by an external key |
| Callers are internal (no auth); inactive stores still process; deductions are deactivated, never hard-deleted or moved to another store (either would orphan their events) | Add auth; return 400 for inactive stores; add a foreign key with ON DELETE CASCADE, and clear events on a store move |

## 4. The API contract

The request is `POST /api/vendors/deductions` with `{ "storeId": "1", "deductionId": "4" }`. The acceptance criteria ask for counts, so the response adds them. It also fixes the brief's "succesfully" typo; if clients match that string exactly, keep the original:

```json
{ "status": "success", "message": "Deductions processed successfully",
  "deductionsProcessed": 1, "expenseEventsRemoved": 0,
  "expenseEventsUpdated": 0, "expenseEventsCreated": 500, "expenseEventsTotal": 500 }
```

Errors use the brief's codes and shape plus one field, `retryable`:

| Failure | Response | retryable |
| --- | --- | --- |
| Missing, non-numeric or out-of-range ID | 400 INVALID_REQUEST, before any database work | false |
| Basis other than NET_RECEIPTS, or an end date before the start date | 400 INVALID_REQUEST | false |
| Store or deduction not found, or the deduction belongs to another store (same message) | 404 NOT_FOUND | false |
| Q4 or Q5 finds rows this deduction doesn't own | 500 PROCESSING_FAILED, naming the other deduction or the untagged rows | false |
| Statement timeout | 500 PROCESSING_FAILED | false, since the same work would time out again |
| Lock timeout, no free pooled connection, deadlock, lost connection | 500 PROCESSING_FAILED | true |

**Idempotency.** With the same rule and sales, any number of calls leave the same events; a repeat, including a retry after a lost response, reports 0 created and the same total. A stored-response idempotency key would be wrong: the body doesn't change when the deduction is edited, so a replay would skip a needed recalculation.

## 5. How a request is processed

Queries are numbered in the order they run.

```text
Validate IDs (Zod) ──────────────────── invalid ──► 400
   │
   ▼  one transaction from here to COMMIT
Lock store (Q1)
   │
Load store + deduction (Q2) ────────── not found ──► 404
   │
Check basis and dates ────── bad basis or dates ──► 400
   │
Compute new_events (Q3) + ANALYZE
   │
Ownership checks (Q4, Q5) ───────────── conflict ──► 500, rolled back
   │
Remove / update / insert (Q6 to Q8)
   │
COMMIT ─────────────────────────────── any error ──► 500, rolled back
   │
200 with counts
```

1. **Validate** the body with the existing Zod middleware, tightened to digit strings within BIGINT range.
2. **Open a transaction** and set, with SET LOCAL, a `lock_timeout` of 10s and a `statement_timeout` of 60s. Sequelize already sets each connection to UTC.
3. **Lock the store (Q1).** A second call for the same store waits here.
4. **Load the store and deduction (Q2)** after the lock, so the newest rule is read.
5. **Compute the correct set (Q3)** into a temp table, empty if the deduction is inactive.
6. **Check ownership (Q4, Q5, section 7).**
7. **Apply the difference (Q6 to Q8).** For an inactive deduction the empty set means every event is removed, as Rule 3 requires.
8. **Commit** and return the counts.

## 6. Database reads

**Q2 loads the store and deduction in one round trip** through the `store_id` unique index and the deduction's primary key, and turns the effective dates into a day range:

```sql
SELECT s.store_id,
       d.deduction_id, d.deduction_type, d.deduction_basis,
       d.deduction_value, d.vendor_code, d.is_active,
       CAST(d.effective_date_from AS date)     AS from_date,
       CAST(d.effective_date_to AS date) + 1   AS to_date_exclusive
FROM generic.store s
LEFT JOIN vendor.deductions d
       ON d.deduction_id = :deductionId AND d.store_id = s.store_id
WHERE s.store_id = :storeId;
```

No row means "Store not found"; no deduction means "Deduction not found". The dates stay 'YYYY-MM-DD' strings in Node, never JavaScript Dates, so the Node time zone can't shift them.

**Q3 builds the correct set.** The dates go in as fixed values, so TimescaleDB skips out-of-range chunks at planning time:

```sql
CREATE TEMP TABLE new_events ON COMMIT DROP AS
SELECT ps.store_id, p.source_system_id, p.product_sku,
       ps.report_date                                  AS expense_date,
       :deductionType                                  AS expense_subtype,
       :vendorCode                                     AS vendor_code,
       ROUND(ps.shipped_revenue * :deductionValue, 2)  AS cost
FROM vendor.product_sales ps
JOIN generic.product p ON p.product_id = ps.product_id
WHERE ps.store_id = :storeId
  AND ps.report_date >= :fromDate
  AND ps.report_date <  :toDateExclusive
  AND :isActive;
```

Values go in as Sequelize replacements, which escape each value (`CREATE TEMP TABLE … AS` doesn't accept bind parameters). `deduction_value` stays the string Postgres returned (`"0.10"`), so the math happens in `numeric` and Scenario 1's cost is exactly 1,000.00.

`ANALYZE new_events;` follows Q3, because Postgres never collects statistics for temp tables. The temp table costs one extra write; in return the sales join runs once and later queries are planned on real row counts.

**One index is missing.** I'd add `(store_id, report_date)` on sales, built chunk by chunk. Until then chunk exclusion limits Q3 to in-range chunks, and `(store_id)` filters within them.

## 7. Concurrency and ownership

**Q1 locks the store:** `SELECT pg_advisory_xact_lock(hashtextextended('deduction-processing:' || :storeId, 0));`. Advisory lock keys are shared across the database, so a bare `pg_advisory_xact_lock(1)` could collide with another feature; the prefixed hash gives this feature its own 64-bit key space. I lock the store rather than the deduction because two same-type deductions of one store compete for the same rows. With a lock per deduction they would run in parallel, and neither would see the other's rows before commit. The cost is that a store's deductions run one at a time, which is fine for a handful per store.

**Q4 checks for overlapping fees.** Without `deduction_id` in the key (section 3), two active same-type deductions covering the same days need the same rows. Q4 fails the second one before any write:

```sql
SELECT e.deduction_id, count(*)
FROM new_events n
JOIN vendor.expense_events e
  ON  e.store_id = n.store_id
  AND e.source_system_id = n.source_system_id
  AND e.expense_date = n.expense_date
  AND e.expense_type = 'SELL_OUT'
  AND e.expense_subtype = n.expense_subtype
WHERE e.deduction_id IS DISTINCT FROM :deductionId
  AND e.expense_date >= :fromDate AND e.expense_date < :toDateExclusive
GROUP BY e.deduction_id;
```

Any row fails the call and names the other deduction. `IS DISTINCT FROM` treats NULL as "someone else", so colliding untagged rows are caught too.

**Q5 checks for untagged events.** Q4 sees nothing when the new set is empty, so Q5 covers inactive deductions as well:

```sql
SELECT count(*)
FROM vendor.expense_events
WHERE store_id = :storeId AND deduction_id IS NULL
  AND expense_type = 'SELL_OUT' AND expense_subtype = :deductionType
  AND expense_date >= :fromDate AND expense_date < :toDateExclusive;
```

A nonzero count fails the call instead of reporting a misleading `removed: 0`.

## 8. Database writes

Q6 to Q8 touch only this deduction's SELL_OUT rows. Q6 and Q7 find them through the `(store_id, deduction_id)` index.

**Q6 removes rows that no longer apply,** such as days outside a shortened range:

```sql
DELETE FROM vendor.expense_events e
WHERE e.store_id = :storeId AND e.deduction_id = :deductionId
  AND e.expense_type = 'SELL_OUT'
  AND NOT EXISTS (SELECT 1 FROM new_events n
                  WHERE n.source_system_id = e.source_system_id
                    AND n.expense_date     = e.expense_date
                    AND n.expense_subtype  = e.expense_subtype);
```

**Q7 updates rows whose cost or details changed:**

```sql
UPDATE vendor.expense_events e
SET cost = n.cost, vendor_code = n.vendor_code,
    product_sku = n.product_sku, updated_at = now()
FROM new_events n
WHERE e.store_id = :storeId AND e.deduction_id = :deductionId
  AND e.expense_type = 'SELL_OUT'
  AND e.expense_date >= :fromDate AND e.expense_date < :toDateExclusive
  AND e.source_system_id = n.source_system_id
  AND e.expense_date = n.expense_date
  AND e.expense_subtype = n.expense_subtype
  AND (e.cost, e.vendor_code, e.product_sku)
      IS DISTINCT FROM (n.cost, n.vendor_code, n.product_sku);
```

**Q8 inserts missing rows.** `ON CONFLICT DO NOTHING` skips existing rows. This is safe only because the store lock (Q1) and Q4 guarantee that any existing row belongs to this deduction. Q4 only sees committed rows, so a row an outside writer commits after Q4 would be skipped silently; section 3 assumes no such writer. I avoid a `NOT EXISTS` probe here because on a fresh, unanalyzed `expense_events` it can rescan the growing table for every inserted row:

```sql
INSERT INTO vendor.expense_events
  (store_id, source_system_id, product_sku, expense_date, vendor_code,
   expense_type, expense_subtype, cost, expense_status, deduction_id,
   created_at, updated_at)
SELECT n.store_id, n.source_system_id, n.product_sku, n.expense_date,
       n.vendor_code, 'SELL_OUT', n.expense_subtype, n.cost, 'CONFIRMED',
       :deductionId, now(), now()
FROM new_events n
ON CONFLICT DO NOTHING;
```

**Date bounds.** Q7 repeats Q3's bounds, so TimescaleDB skips other chunks. Q6 can't, because it must find leftovers from an older, wider range, so it probes the `(store_id, deduction_id)` index in every chunk.

**Counts.** Each write is wrapped in a CTE that counts in Postgres, so one number per statement reaches Node. The total is Q3's row count:

```typescript
await sequelize.transaction(async (t) => {
  const opts = { replacements, transaction: t, type: QueryTypes.SELECT };
  // deleteSql, updateSql, insertSql = Q6 to Q8 without the trailing semicolon.
  const count = async (sql: string) => {
    const [row] = await sequelize.query<{ n: number }>(
      `WITH x AS (${sql} RETURNING 1) SELECT count(*)::int AS n FROM x`, opts);
    return row.n;
  };
  return {
    expenseEventsRemoved: await count(deleteSql),
    expenseEventsUpdated: await count(updateSql),
    expenseEventsCreated: await count(insertSql),
  };
});
```

**Transaction boundaries.** Validation runs first; Q1 to Q8 and the ANALYZE run in one managed transaction, and the response is sent after commit. Any failure rolls everything back. Readers see the old events until commit, then all the new ones at once.

**Batch size.** No app-level batching. Rows never reach Node, and each write is one set-based statement. Work that outgrows one transaction goes to the background worker in section 12.

## 9. Performance at 100,000+ rows

App memory and round trips don't grow with the range. The read is bounded by the date range, and the writes by what changed.

**Where one transaction stops working.** The brief's 100,000 rows are expected to finish inside one atomic transaction, roughly 1 to 3s per 100,000-row write on a similar setup; the 100,000-row test confirms this. A deduction with millions of changed rows could pass the 60s `statement_timeout`. It then rolls back cleanly and returns a non-retryable 500, with nothing half-written. That size is outside the brief (section 12).

| Change | Sales rows read | Removed | Updated | Inserted |
| --- | --- | --- | --- | --- |
| First run, 100,000 sales | 100,000 | 0 | 0 | 100,000 |
| Repeat call | 100,000 | 0 | 0 | 0 |
| Rate changed | 100,000 | 0 | 100,000 | 0 |
| End date 10 days earlier | the shorter range | the removed days | 0 | 0 |
| One sale corrected | 100,000 | 0 | 1 | 0 |
| Deactivated | 0 | 100,000 | 0 | 0 |

Every active call reads its full range even when it writes nothing, which is why the section 6 index matters. I'd validate on a fresh database with 100,000+ seeded sales and `EXPLAIN (ANALYZE, BUFFERS)`: Q3 reads only in-range chunks, Q6 and Q7 hash-join against the temp table, and Q8 stays fast while `expense_events` has no statistics.

## 10. Error handling and logging

A rewritten error handler maps typed errors to the section 4 table. The current one returns `{ error, stack }` and lacks Express's fourth `next` argument, so Express never uses it; the validation middleware's `{ error: "Validation Error", details }` also moves to the brief's shape. Retryable failures are recognized by Postgres error code and by Sequelize's connection-acquire timeout. Callers retry with backoff only when `retryable` is true.

Each step logs one Winston line with the request, store and deduction IDs: request received, validation result, lock acquired (with wait time), deduction loaded, size of the new set, ownership checks passed, rows removed, updated and created, commit or rollback, and total duration. Revenue isn't logged.

## 11. Testing

Most logic is SQL, so tests run against a separate TimescaleDB database. The repo has no tests yet (jest.config.js points at a missing tests/setup.ts), so I'd add them with the installed supertest. The seed has no inactive or FIXED_AMOUNT deduction, so tests insert their own data:

- Both scenarios in section 13; a repeat call writes nothing; a shorter range, new rate or corrected sale changes only the affected rows.
- Sales on the first and last effective days are included, including the +08:00 case from section 3.
- SELL_IN events (Rule 2), chargebacks and other deductions' events are never changed.
- Another store's deduction returns 404; FIXED_AMOUNT, a 30-digit ID and an end date before the start date return 400; an overlapping deduction or untagged row fails with `retryable: false` and writes nothing.
- Two concurrent calls on one store wait on the store lock: for one deduction they end with one correct set, and for two overlapping deductions the second fails at Q4. A failure forced after Q6 leaves the original events intact.
- A 100,000-row first run and rate change each finish in one transaction within the 60s timeout, starting from a fresh database.

## 12. Future considerations

- **Keep events fresh when sales change.** After loading or correcting sales, the ETL calls the endpoint for each active deduction on the changed store and dates, one store's deductions in sequence so they don't queue on the store lock. A nightly run is the safety net.
- **Let overlapping same-type fees coexist,** if the business confirms they're real. Replace the primary key with a unique index on the same columns plus `deduction_id`, with NULLS NOT DISTINCT so rows without a deduction stay unique. Q4 is then unnecessary and Q6 to Q8 don't change. First audit other writers for `ON CONFLICT` targets on the old key and confirm TimescaleDB supports the index. If overlaps aren't real, reject them when deductions are created.
- **FIXED_AMOUNT and SELL_IN** reuse the same flow; only Q3 changes.
- **Very large deductions:** when a range holds millions of sales rows, I'd hand the work to a background worker. The endpoint returns 202 with a job ID.
- **Smaller items:** a run log for audit (it would also let Q6 target the previous range), a currency check, and wider precision for rates like 7.5%.
- **Not proposed:** a saga (one database is enough), event sourcing, or caching the rule (a stale rule gives wrong costs).

## 13. Worked examples: the brief's two scenarios

The deduction IDs and fee name are illustrative; the seed only has deductions 1 to 6.

**Scenario 1: active 10% fee, 500 sales.** Input: deduction 10, a NET_RECEIPTS MARKETING_FEE of 0.10, effective 1 to 31 January 2026, with 500 sales of $10,000 shipped revenue each. Key step: Q3 builds 500 rows at ROUND(10,000 × 0.10, 2) = 1,000.00; nothing is stored yet, so Q8 inserts all 500 tagged with `deduction_id` 10. Response: removed 0, updated 0, created 500, total 500. A repeat returns created 0, total 500.

**Scenario 2: inactive 15% fee, 200 existing events.** Input: deduction 11, a deactivated 15% NET_RECEIPTS fee with 200 SELL_OUT events tagged `deduction_id` 11. Key step: `:isActive` is false, so Q3's set is empty and no sales are read; Q6 removes all 200, and chargebacks and other deductions are untouched. Response: removed 200, created 0, total 0. The deduction row stays, which is the soft delete Rule 3 describes.

## Appendix: hardening

Useful in production, not needed for the brief:

- **Retry-After on lock timeouts,** with the message "store is being processed, retry shortly".
- **A whole-request deadline** (for example 120s, capping the 30s pool wait, 10s lock wait and the statements) that cancels the running statement and rolls back. The caller's HTTP timeout sits above it, so nothing commits after the caller gives up.
- **A concurrency cap of 5 runs across instances,** to bound database load and keep pool connections free for the rest of the service. Across instances that needs an advisory-lock semaphore: each run tries `pg_try_advisory_xact_lock(hashtextextended('deduction-slot:' || n, 0))` for n = 1 to 5 and returns 500 with `retryable: true` if no slot is free.
