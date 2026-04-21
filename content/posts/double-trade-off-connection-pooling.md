---
title: "The Double Trade-off of Connection Pooling: HikariCP, PostgreSQL, and the Price of PgBouncer"
date: 2026-04-19T00:00:00+02:00
draft: true
description: "Why Spring Boot often wants a large HikariCP pool, why PostgreSQL degrades under the same numbers, and which second trade-off PgBouncer introduces in transaction mode."
tags:
  - java
  - spring-boot
  - postgresql
  - hikaricp
  - pgbouncer
  - performance
  - connection-pooling
categories:
  - Backend Engineering
  - Databases
icon: "/images/icons/squares-accent.svg"
showToc: true
canonicalURL: "https://nikita-kibitkin.github.io/posts/double-trade-off-connection-pooling-hikaricp-postgresql-pgbouncer-en/"
---

Connection pooling is usually debated at the wrong abstraction level. The application team sees `SQLTransientConnectionException` and raises `maximumPoolSize`. The database team looks at `pg_stat_activity`, sees hundreds of backend processes, and asks for a smaller pool. Both sides are correct at the same time. Both sides can also make the system worse if they optimize only for their own layer.

The problem is that the application and PostgreSQL have different optima. The first trade-off sits between convenient JVM-side concurrency and actual database throughput. The second appears when we try to resolve that conflict with PgBouncer.

## Act 1. The Requirements Paradox: the application wants 200, the database wants 20

For a Spring Boot service, a large HikariCP pool often looks rational.

The reason is simple: the application does not measure pure **query execution time**. It measures **connection hold time**. A connection can stay occupied much longer than the database actually spends executing SQL.

A typical synchronous request flow in enterprise Java looks like this:

1. A thread enters an `@Transactional` service.
2. It executes a `SELECT` or `UPDATE`.
3. It calls an external HTTP API.
4. It runs more business logic.
5. It commits the transaction.

If a transaction lives for 180 ms while the total SQL time inside it is only 12 ms, HikariCP cares about the full 180 ms. The connection is occupied for that entire interval. At 1000 RPS, **Little's Law** already puts you at roughly 180 concurrently held connections:

```text
Concurrency ≈ Throughput × Hold Time
Concurrency ≈ 1000 req/s × 0.180 s = 180 connections
```

From that angle, `maximumPoolSize = 200` does not look irrational. It is just an attempt to avoid `getConnection()` timeouts and to keep request threads from turning into a queue in front of HikariCP.

PostgreSQL sees a different picture. It does not care how many request threads exist in the JVM or how much time they spend waiting on external HTTP. It cares about how many backend processes are simultaneously competing for CPU, shared memory, the lock manager, and I/O.

If the same request hits the database for only 12 ms out of 180 ms, average concurrency on the database side is much closer to this:

```text
Database concurrency ≈ 1000 req/s × 0.012 s = 12 active queries
```

That is the paradox:

- the application may need a pool around 150-200 to survive high **connection hold time**;
- PostgreSQL under the same traffic may reach maximum throughput at only 12-20 truly active backend processes.

The same workload produces two different truths. At the application layer, a large pool looks like protection against timeouts. At the database layer, the same pool becomes oversubscription.

## Act 2. The Physics of the Bottleneck: why the database does not scale linearly

At this point it helps to leave folklore behind and use **Neil Gunther's Universal Scalability Law (USL)**. In simplified form, the law looks like this:

```text
X(N) = N / (1 + α(N - 1) + βN(N - 1))
```

Where:

- `α` models **contention**: competition for a shared resource;
- `β` models **coherency**: the cost of coordination and consistency between concurrent workers.

If `α = 0` and `β = 0`, throughput would grow linearly. Real databases do not behave like that.

### What `contention` means in PostgreSQL

PostgreSQL does not have a single global bottleneck, but it does have a set of shared structures that backend processes compete for on hot paths:

- `shared_buffers` and the related LWLock/SpinLock structures;
- lock tables;
- `ProcArray`;
- shared metadata for WAL and snapshot visibility.

Once the number of active backends grows too high, they start spending time not on useful work but on waiting for access to shared structures. That is the practical form of **latch contention**: queues around short but very hot critical sections.

### What `coherency` means in PostgreSQL

The second part of the problem is less visible, but just as expensive. PostgreSQL still uses a **process-per-connection** model. Every database connection is a separate OS process with its own memory footprint, scheduler participation, and context switches.

On 8 physical cores, 200 runnable backend processes do not give you “25x more parallelism.” They give you:

- constant **context switches**;
- worse cache locality;
- more scheduler bookkeeping;
- extra inter-core traffic around shared state.

So as connection count rises, you pay not only queueing overhead but also a coordination tax. That is exactly what USL describes.

### Why can the JVM handle 200 threads while the database cannot?

At this point, developers often ask a reasonable question: if a web server such as Tomcat or Undertow can comfortably hold thread pools in the 200-500 range, why can't the database do the same? The difference lies in the architecture of memory isolation.

In the JVM, each request-handling thread mostly works with its own local thread stack and with independent heap objects. At the application level, those threads only rarely compete on truly global locks. Context switching between JVM request threads is relatively cheap, and a large share of those threads spend most of their lifetime simply sleeping on network I/O.

In PostgreSQL, the picture is the opposite. First, each connection is a heavyweight OS process. Second, and more importantly, all 200 of those processes are forced to keep touching the same **shared global state**. They look up pages in the common `shared_buffers`, acquire locks on the same financial ledger rows or account balances, and line up to flush commits through the same Write-Ahead Log (WAL).

An application running on the JVM scales across threads because its requests are physically *isolated*. PostgreSQL degrades as connection count rises because its processes aggressively *share state*.

### Why a smaller pool is usually faster than a bigger one

Brett Wooldridge, the creator of HikariCP, points to a practical **rule of thumb** for the database side:

Rule of thumb (HikariCP):
> `Pool ≈ (2 × N_cores) + N_spindles`

For a server with 8 physical cores, that gives:

```text
Pool ≈ (2 × 8) + 4 = 20
```

Even if you assume `N_spindles = 0` for a well-cached dataset or an SSD-heavy system, the order of magnitude is still around 16, not 200.

That is the uncomfortable conclusion: from PostgreSQL's point of view, the “ideal” pool for maximum throughput often lives in the 16-32 range. Not hundreds. Not thousands. A small number, close to the actual compute capacity of the node.

## Act 3. The Illusion of a Free Solution: PgBouncer as connection multiplexing

At this point, PgBouncer usually enters the discussion.

The idea looks elegant: separate **client-side concurrency** from **server-side concurrency**.

The topology looks like this:

```text
Spring Boot threads
    -> HikariCP (maximumPoolSize = 200)
        -> PgBouncer (pool_mode = transaction)
            -> PostgreSQL (default_pool_size = 20)
```

On the application side, you can keep the large pool:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 200
      connection-timeout: 250
```

On the PgBouncer side, you limit the number of physical backend connections:

```ini
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
reserve_pool_size = 5
```

What happens next:

- HikariCP holds up to 200 client connections to PgBouncer;
- PgBouncer exposes only 20 server connections to PostgreSQL;
- each short transaction gets one backend while it runs, and that backend is returned to the pool after `COMMIT` or `ROLLBACK`.

From Spring Boot's perspective, this removes the first form of scarcity: `getConnection()` stops being blocked directly by the PostgreSQL backend-process limit. The application now sees “many cheap connections” in front of the proxy and is less likely to hit acquisition timeouts in HikariCP.

From PostgreSQL's perspective, this is also beneficial: the database sees not 200 processes, but roughly 20. That keeps backend concurrency near the throughput plateau instead of constant competition for shared state.

On paper, it looks like a clean win:

- the application appears to have 200 connections;
- PostgreSQL actually executes the workload through 20 processes;
- the system stops paying for database-side oversubscription.

But the mechanism should not be idealized.

PgBouncer in `transaction` mode does not make long transactions cheap. If you hold an `@Transactional` block open for 300 ms and call external HTTP inside it, the backend remains occupied for the full 300 ms. The proxy cannot cut an open transaction in half and release the server connection in the middle. It solves the conflict between client connection count and backend process count. It does not justify wide transaction boundaries.

## Act 4. The Price of Magic: the second trade-off

PgBouncer resolves the first conflict, but creates the second one. The price is no longer paid in process count. It is paid in session affinity and protocol semantics.

### 1. Prepared Statements stop being free

In `transaction` mode, the next query from a client is not guaranteed to land on the same PostgreSQL backend process as the previous one. That breaks the naive expectation that a server-side prepared statement created “on the connection” will still exist on the next call.

Historically, that led to a simple conclusion: **transaction pooling is incompatible with server-side prepared statements**.

Today the picture is slightly more nuanced. Modern PgBouncer versions can support **protocol-level prepared statements** if `max_prepared_statements > 0` is enabled. But that is not free restoration of session affinity. It is extra logic inside the proxy itself:

- PgBouncer has to inspect and rewrite protocol messages;
- the proxy keeps an LRU cache of prepared statements on server connections;
- CPU and memory overhead rise on the PgBouncer side;
- SQL-level `PREPARE/EXECUTE` still do not become a real replacement for session mode.

That is why a predictable baseline for a Java stack in `transaction` pooling often looks like this:

```text
prepareThreshold=0
```

In `pgjdbc`, that means the driver stops promoting frequently executed `PreparedStatement` objects into server-side prepared statements. That is not the same as fully disabling the client-side statement cache, but the stable server-side plan cache on a backend process disappears from the picture. If a team wants the most literal behavior possible, it may also set `preparedStatementCacheQueries=0` and `preparedStatementCacheSizeMiB=0`.

In practice, this is usually configured directly in the JDBC URL, for example:

```text
jdbc:postgresql://host:5432/db?prepareThreshold=0
```

That is a separate cost. On OLTP traffic with short repetitive queries, it can be measurable in CPU terms.

### 2. The extra network hop never disappears

PgBouncer is one more process and one more TCP hop between the application and PostgreSQL.

If the proxy runs on the same host, the added latency can stay in the hundreds of microseconds. If it is a separate instance in the same availability zone, an additional `0.2-1.0 ms` per round trip is easy to get. For a query that itself takes `2-3 ms`, that is already a double-digit percentage overhead. For a `50 ms` query, it is almost noise.

So there is always a latency tax. The only question is whether it is smaller than the benefit from capping backend concurrency.

### 3. Session state becomes unsafe by default

Once the hard binding between client and backend process disappears, the reliability of session-level state disappears with it.

In `transaction` mode, you cannot safely rely on things like:

- plain `SET statement_timeout = ...` as a long-lived session setting;
- temporary tables with semantics that outlive one transaction;
- `LISTEN` and other scenarios that require session affinity;
- session-level advisory locks;
- cursors that live longer than the transaction scope.

Even when some parameters can be tracked explicitly by the proxy, that is no longer “transparent passthrough.” It is a new area of configuration, testing, and production debugging.

The practical conclusion is blunt but useful: PgBouncer in `transaction` mode works well only with a disciplined workload, where transactions are short, session state is minimal, and the application does not expect too much from any specific backend connection.

## Conclusion

There is no silver bullet here.

A large HikariCP pool can be logical for the application, because the JVM lives in terms of **connection hold time**. The same size is usually wrong for PostgreSQL, because the database lives in terms of **active backend concurrency**. PgBouncer lets you separate those two worlds and often gives the right shape of compromise: many client connections at the top, few physical backend processes at the bottom.

But the proxy does not provide magic for free. You pay for it with loss of session affinity, complications around prepared statements, an extra hop, and stricter discipline around transactions.

The Staff-level job here is not to find the “perfect” pool size. It is to choose consciously which limitation the system will pay for: PostgreSQL oversubscription, queuing in the proxy, extra parse/plan CPU, or loss of session-level features. Architecture almost never offers free solutions. It offers different forms of controlled damage.

## References

- [HikariCP Wiki: About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Neil J. Gunther: A General Theory of Computational Scalability Based on Rational Functions](https://arxiv.org/abs/0808.1431)
- [PgBouncer Features](https://www.pgbouncer.org/features.html)
- [PgBouncer Config: `max_prepared_statements`](https://www.pgbouncer.org/config)
- [pgjdbc Connection Properties: `prepareThreshold`](https://jdbc.postgresql.org/documentation/use/)
