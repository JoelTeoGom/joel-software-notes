# System design

Design problems and real case studies, worked through with diagrams: what the constraint is, which options exist, what was chosen and what it cost.

## Learning path

01 stands alone. 02-04 are one case study, Atlassian's tenant directory on AWS, split into three topics; read them in order.

| # | Topic | You'll understand |
|---|---|---|
| 01 | [Large file ingestion: 10 GB with 512 MB of RAM](01-large-file-ingestion/README.md) | streaming, stage pipelines, bounded queues and backpressure, batching, idempotent writes, checkpoints |
| 02 | [Tenant routing](02-tenant-routing/README.md) | why stateless services need a directory, shards and DB-per-tenant, hash vs directory placement, the requirements of a lookup on every request |
| 03 | [CQRS and multi-region replication](03-cqrs-multi-region/README.md) | a write side and a read side connected by streams, Kinesis vs SNS, what CQRS buys and costs |
| 04 | [Caching at scale](04-caching-at-scale/README.md) | in-process caches with invalidation, the cache library and the p99, client-side caching, the four "not magic" lessons |

The infrastructure side of the Atlassian case (DNS, load balancers, the ingress, sidecars) is in [platform-infra](../platform-infra/README.md).

## The one idea that connects everything

**Bound everything, and separate the paths that have different needs.**

- The ingestion pipeline bounds memory with fixed buffers and bounded queues, and separates read, process and write so each scales on its own.
- The tenant directory separates the rare, must-be-correct writes from the constant, must-be-fast reads.
- The caches separate the hot path (memory) from the slow path (the database), and the tail-latency fixes are about bounding the worst case, not improving the average.

## Glossary

| Term | Meaning |
|---|---|
| Backpressure | A slow consumer slowing its producers down (via full bounded queues) instead of being flooded |
| Idempotent | Applying an operation twice has the same effect as once |
| Tenant | A customer (a company) in a multi-tenant system |
| Shard | A group of tenants (or a slice of data) served by its own resources |
| CQRS | Command Query Responsibility Segregation: separate write and read models |
| Projection | A read-optimised copy derived from the source of truth |
| Eventual consistency | Replicas may briefly disagree but converge |
| p99 | The latency 99% of requests are faster than: the tail |
| Cache stampede | Many concurrent misses for one key all hitting the backing store |
| SLA | Service level agreement: what the provider promises (not necessarily what you need) |
