# Online schema migrations

> How to `ALTER` a big MySQL table in production without freezing it: metadata locks, `ALGORITHM=INSTANT`, batched backfills, and how gh-ost does it.

**What you'll learn**

- Why an `ALTER TABLE` can freeze a table even when the change itself is instant.
- What row locks an `UPDATE` takes, and why backfills go in small batches.
- The ghost table pattern, and why gh-ost reads the binlog instead of using triggers.
- How gh-ost swaps the tables safely at the end.

**Before reading**: [The WAL](../01-wal/README.md) and [MySQL vs Postgres](../02-mysql-vs-postgres/README.md).

*Diagrams for this topic are still to be drawn.*

---

## 1. The metadata lock (MDL)

Every statement that touches a table takes a **metadata lock** on it. It doesn't protect rows, it protects the table's definition. It works exactly like a Go `sync.RWMutex`:

| Go | MDL | Who takes it |
|---|---|---|
| `RLock()` | shared | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| `Lock()` | exclusive | `ALTER`, `DROP`, `RENAME`, `CREATE`/`DROP TRIGGER` |

- Many shared holders at once are fine: reads and writes don't block each other on this lock.
- The exclusive lock waits until **nobody** holds the shared one.
- While an exclusive request is waiting, **new shared requests wait too**, so DDL doesn't starve.

The trap is that a shared MDL is held **until the end of the transaction**, not the end of the statement. A transaction left open (a script with autocommit off, a GUI client in manual commit mode, a Go `tx` that is never committed or rolled back) holds it for hours.

When an `ALTER` arrives then, it waits for that transaction, and every new query on the table queues behind the `ALTER`: the table looks frozen, even though the `ALTER` itself would take milliseconds.

```sql
-- before: any long-open transactions?
SELECT trx_mysql_thread_id, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS open_for_s, trx_query
FROM information_schema.innodb_trx
ORDER BY trx_started;

-- while it's stuck: who blocks whom
SELECT * FROM sys.schema_table_lock_waits\G

-- give up fast instead of freezing everyone, then retry
SET SESSION lock_wait_timeout = 2;
ALTER TABLE orders ADD COLUMN order_type ENUM('a','b') NULL, ALGORITHM=INSTANT;
```

With many short queries, the `ALTER` only waits for the ones in flight: a pause of milliseconds. Only **long** transactions make it dangerous.

In Go, always pair a transaction with a rollback:

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil { return err }
defer tx.Rollback() // no-op after Commit
```

## 2. `ALGORITHM=INSTANT`

Instead of rewriting the table, MySQL only changes the data dictionary. Existing rows aren't touched; when an old row is read, InnoDB knows the column didn't exist yet and returns its default.

Conditions (MySQL 8.0):

- Before 8.0.29, only adding a column at the end; from 8.0.29, any position, and instant `DROP COLUMN` too.
- `NOT NULL` with a `DEFAULT` is fine; the column doesn't have to be nullable.
- No `ROW_FORMAT=COMPRESSED`, no `FULLTEXT` index, not mixed with non-instant operations in the same `ALTER`.
- From 8.0.29, at most 64 instant changes before the table has to be rebuilt.

Write `ALGORITHM=INSTANT` explicitly: if it can't be done instantly, MySQL returns an error instead of silently rebuilding the table.

## 3. Backfilling in batches

What one `UPDATE` over the whole table does in InnoDB:

- It takes an **exclusive lock on each row** as it scans, and keeps them **until commit**.
- In `REPEATABLE READ` (MySQL's default) it also locks the **gaps** between rows, which blocks inserts into the scanned ranges.
- Plain `SELECT`s keep working: they read the previous version through MVCC. `SELECT ... FOR UPDATE`, other updates and inserts into locked gaps wait.

And a single huge transaction also means a huge undo log, a very long rollback if it fails, one enormous event blocking the replicas, and purge stuck for every table.

So backfills go in small batches, each one its own transaction:

```sql
UPDATE orders
SET order_type = ...
WHERE id BETWEEN 1 AND 5000
  AND order_type IS NULL;   -- idempotent: safe to rerun
-- commit, short pause, next range
```

Use PK ranges (not `LIMIT`/`OFFSET`) so InnoDB walks the clustered index, and pause between batches so replicas keep up.

## 4. The ghost table pattern

For changes that can't be instant, every online schema change tool does the same thing:

1. Create an empty **ghost table** with the new schema. Altering an empty table is instant.
2. **Copy** the existing rows in chunks, by PK range.
3. Meanwhile, **propagate** every ongoing `INSERT`/`UPDATE`/`DELETE` to the ghost table.
4. When both are in sync, **swap** them (cut-over).

The difference between tools is step 3.

## 5. Triggers vs the binlog

`pt-online-schema-change` and similar tools use **triggers**: SQL stored in the database that runs inside the same transaction as each write and mirrors it into the ghost table. Problems:

- Every write pays for an interpreted trigger and a second write, in the critical path.
- Each transaction now locks rows in **two** tables, competing with the chunk copier: lock contention, even deadlocks.
- Creating and dropping triggers needs an exclusive MDL: the queue problem from section 1.
- **Not pausable**: you can stop the copier, but not the triggers. Removing them loses changes.
- Can't be tested realistically on a replica.

**gh-ost** connects to MySQL **as a fake replica** and reads the binlog (in row format) instead. Changes are applied to the ghost table asynchronously, by gh-ost alone, over one connection. Your transactions touch one table again, and gh-ost can truly pause.

Why the old tools didn't do this: until MySQL 5.7.7 (2015) the default binlog format was **statement-based**, which doesn't say which rows changed. And reading the binlog means implementing the replication protocol and its binary format, while a trigger is a few lines of SQL.

## 6. How the copy and the binlog converge

The copy and the binlog events can touch the same row in any order. Two rules make it always converge:

- The **copy** uses `INSERT IGNORE`: if the row is already there (it came from the binlog), keep it.
- **Binlog events** always win: inserts and updates are applied as `REPLACE INTO` with the full row image; a delete of a row that hasn't been copied is a no-op.

The binlog is applied in order and carries complete rows, so once gh-ost catches up with the present, every row matches its latest change.

The copy stops at the `MAX(id)` noted at the start. Every row inserted after that arrives through the binlog.

## 7. The cut-over

The swap is the delicate moment: no write may slip through unapplied. gh-ost does it like this:

1. Create a **sentry table** `_orders_del` and `LOCK TABLES orders WRITE`. App writes now wait (they don't fail).
2. From another connection, issue `RENAME TABLE orders TO _orders_del, _orders_gho TO orders`. It blocks on the lock.
3. Write a marker into its changelog table and wait until it comes back through the binlog: everything before the lock has been applied.
4. Drop the sentry and unlock. MySQL runs the pending `RENAME` before the waiting writes, so they land on the new table.

If gh-ost dies halfway, the lock is released and the `RENAME` fails because `_orders_del` still exists: the original table stays untouched.

---

## Key takeaways

- An `ALTER` is dangerous because of the **MDL queue**, not because of its duration. Hunt long transactions, use a low `lock_wait_timeout` and retry.
- `ALGORITHM=INSTANT` + nullable or defaulted column + batched backfill by PK range is the cheap path.
- A big single `UPDATE` locks every row (and gaps) until commit; reads still work through MVCC.
- Online schema change = ghost table + chunked copy + change propagation + atomic swap.
- gh-ost replaces triggers with binlog streaming, so the migration is pausable, testable and out of the write path.

## Interview cheat sheet

**Q: You add a column to a 2M-row table and it freezes for 15 minutes. Why?**
A: Probably not the `ALTER` itself: it waited for the metadata lock behind a long-open transaction, and every new query queued behind it. Find it in `innodb_trx`, and run DDL with a low `lock_wait_timeout` and retries.

**Q: How do you add and fill a column safely?**
A: Add it with `ALGORITHM=INSTANT` (nullable or with a default), then backfill in small batches by PK range, each in its own transaction and idempotent.

**Q: Why does gh-ost read the binlog instead of using triggers?**
A: Triggers run inside every app transaction: extra latency, lock contention, and they can't be paused. Reading the binlog moves the work out of the write path.

**Q: How does gh-ost swap the tables without losing writes?**
A: It locks the original table, issues an atomic `RENAME`, waits until its own marker comes back through the binlog to know it has applied everything, then unlocks. A sentry table makes the `RENAME` fail safely if gh-ost dies.

## Further reading

- [gh-ost: GitHub's online schema migration tool for MySQL](https://github.blog/news-insights/company-news/gh-ost-github-s-online-migration-tool-for-mysql/)
- [gh-ost on GitHub](https://github.com/github/gh-ost)
- [MySQL docs: Online DDL operations](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)
