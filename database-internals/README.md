# Database internals

How a database turns `COMMIT;` into data that survives crashes, how it ships changes to other machines, how it lays rows out on disk, and how to change a table without stopping production. Mostly Postgres and MySQL/InnoDB, compared side by side.

## Learning path

Read them in order: each one builds on the previous.

| # | Topic | You'll understand |
|---|---|---|
| 01 | [The WAL](01-wal/README.md) | why `COMMIT` is fast and durable: bump pointer, insertion locks, group commit, `fsync`, checkpoints |
| 02 | [MySQL vs Postgres](02-mysql-vs-postgres/README.md) | how each one streams changes to replicas, and how each stores rows: clustered index vs heap, MVCC |
| 03 | [Sync vs async replication](03-sync-vs-async-replication/README.md) | what replication is for, what sync replication really buys you, and how a bank would lay it out |
| 04 | [CDC and Kafka](04-cdc-and-kafka/README.md) | when to put Kafka in a CDC pipeline, and how topics, partitions and offsets work |
| 05 | [Online schema migrations](05-online-schema-migrations/README.md) | how to `ALTER` a big table in production: metadata locks, `INSTANT`, batched backfills, gh-ost |

## The one idea that connects everything

Almost every topic here is the same pattern: **an append-only log plus someone reading it from a position**.

- The WAL is a log that crash recovery replays from the last checkpoint.
- A replica is a reader of that log that never stops replaying it.
- A CDC tool is a reader that turns it into events.
- Kafka is that same log, offered as a service, with many readers.
- gh-ost is a reader of the binlog that replays one table's changes onto a copy.

## Glossary

| Term | Meaning |
|---|---|
| WAL | Write-ahead log: every change is logged (and fsynced) before the data pages are written |
| LSN | Log sequence number: a byte position in the WAL |
| Dirty page | A data page modified in RAM that isn't on disk yet |
| Checkpoint | The point up to which all dirty pages have been written; older WAL isn't needed for recovery |
| Binlog | MySQL's server-level logical log, used for replication and point-in-time recovery |
| Redo log | InnoDB's physical log, its equivalent of the WAL |
| Replication slot | Postgres' record of how far a consumer has read; keeps the WAL it still needs |
| RPO | Recovery point objective: how much committed work you can lose in a failover |
| CDC | Change data capture: turning a database's log into a stream of row changes |
| DDL / DML | Statements that change structure (`ALTER`) / data (`INSERT`, `UPDATE`, `DELETE`) |
| MDL | Metadata lock: the lock protecting a table's definition |
