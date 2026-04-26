---
title: "Why PostgreSQL Gets Correlated Predicates Wrong and Falls into Nested Loop"
date: 2026-04-22T00:00:00+02:00
draft: false
description: "How the Column Independence Assumption underestimates row counts on correlated business data and pushes PostgreSQL from Hash Join into CPU-expensive Nested Loop."
tags:
  - postgresql
  - query-planner
  - cardinality-estimation
  - sql-optimization
  - performance
  - database-internals
categories:
  - Databases
  - Backend Engineering
icon: "/images/icons/squares-accent.svg"
showToc: true
canonicalURL: "" 

---

TL;DR: this article walks through a scenario where business-driven cross-column correlation breaks the PostgreSQL planner's math. Due to a cardinality estimation error, the database funnels millions of rows into a Nested Loop, while the presence of an index merely creates an illusion of efficiency, pushing the optimizer even further toward the wrong plan.
Code for emulating and locally reproducing the problem is available in the [GitHub repository](https://github.com/nikita-kibitkin/postgres-mcv-demo).


By default, PostgreSQL's mathematical model treats columns as independent. In practice, business data frequently contains cross-column correlation. Here, statistics diverge from reality: the cardinality estimate is underestimated at the base relation level, and this underestimation propagates up the tree (error propagation).

As a result, the database may pick a Nested Loop where the actual data volume already demands a Hash Join. Moreover, the presence of a suitable index can even make things worse, leading to excessive random reads and CPU-amplification.

Below we'll walk through one specific scenario. It's deliberately simplified for clarity and somewhat exaggerated to reproduce on small synthetic-experiment scales, but the mechanism itself is typical for production.

## Problem

We have two tables of 10M rows each.

```sql
CREATE TABLE transactions (
    id bigserial primary key,
    transfer_type text not null,
    route_type text not null
);

CREATE INDEX idx_transactions_type_route
    ON transactions (transfer_type, route_type);

CREATE TABLE compliance_docs (
    transaction_id bigint not null,
    notes text not null
);

CREATE UNIQUE INDEX idx_compliance_docs_txid
    ON compliance_docs (transaction_id);
```

The `transactions` table here is a base relation with two statistically dependent predicates. We `JOIN transactions` with `compliance_docs`. For simplicity, we assume a perfect `1:1` between `transactions` and `compliance_docs`: one row in `compliance_docs` per `transaction_id`. I deliberately avoid a fan-out join for a clean experiment.

The business rule is as follows:

- 10% of all transfers have `transfer_type = 'SWIFT'`;
- 10% of rows in the table have `route_type = 'CROSS_BORDER'`;
- all `SWIFT` transfers go via `route_type = 'CROSS_BORDER'`.

In a real payment system, not every `CROSS_BORDER` transfer implies `SWIFT`. There are `SEPA`, local gateway routes, and other channels. But in this synthetic slice the proportions are deliberately made equal so that the math of the error stays transparent: the planner sees `10% × 10% = 1%`, whereas the actual intersection is still `10%`.

In other words, the data contains strong correlation:

```text
P(route_type = 'CROSS_BORDER' | transfer_type = 'SWIFT') = 1
```

The query:

```sql
SELECT t.id, d.notes
FROM transactions t
JOIN compliance_docs d
  ON t.id = d.transaction_id
WHERE t.transfer_type = 'SWIFT'
  AND t.route_type = 'CROSS_BORDER';
```

## Act 1. Problem Physics: planner math

By default, the PostgreSQL planner operates under the **Column Independence Assumption**. If `WHERE` contains two predicates on two different columns of the same table, it estimates their joint selectivity as the product of individual probabilities:

```text
P(A ∩ B) = P(A) × P(B)
```

For the planner in our example:

```text
A := transfer_type = 'SWIFT'
B := route_type = 'CROSS_BORDER'
```

If ordinary per-column statistics show that:

```text
P(A) = 0.1
P(B) = 0.1
```

then the estimate will be:

```text
P(A ∩ B) = 0.1 × 0.1 = 0.01
```

On a 10M-row table this gives:

```text
E-Rows = 10,000,000 × 0.01 = 100,000
```

But the real business semantics are different. For `SWIFT`, the route-type condition doesn't just "often coincide". It follows functionally from the first condition:

```text
P(B | A) = 1
P(A ∩ B) = P(A) × P(B | A) = 0.1 × 1 = 0.1
```

So the actual cardinality is:

```text
A-Rows = 10,000,000 × 0.1 = 1,000,000
```

The planner's error in this toy example is `10x`. In production it easily grows to `100x+` if one condition is rare, the second is also rare, and together they are almost deterministic.

This is already unpleasant with just two predicates. But as the number of dependent predicates grows, the error grows exponentially. Every new `AND` that the planner treats as independent from previous conditions widens the estimate gap once more. If two correlated columns give an error on the order of `10x`, three columns under the same logic easily give `100x`, and four — `1000x`. That's why the most dangerous queries in such schemas are not short OLTP filters, but long compliance or analytics predicates, where with each additional condition the planner widens the gap between the assumed base relation size and reality.

A pseudo-`EXPLAIN` at this step looks like this:

```text
Bitmap Heap Scan on transactions t
  Recheck Cond: ((transfer_type = 'SWIFT') AND (route_type = 'CROSS_BORDER'))
  ->  Bitmap Index Scan on idx_transactions_type_route
        Index Cond: ((transfer_type = 'SWIFT') AND (route_type = 'CROSS_BORDER'))
  rows=100000  -- E-Rows
```

In reality `EXPLAIN ANALYZE` will show something conceptually like this:

```text
Bitmap Heap Scan on transactions t
  ...
  rows=100000 loops=1      -- estimate
  actual rows=1000000      -- reality
```

That alone is enough to trigger error propagation and break the cost model computation.

## Act 2. Under-the-Hood Mechanics: why Nested Loop appears

The choice between `Nested Loop` and `Hash Join` is sensitive to the size of the outer side. In our case the outer side is precisely that filtered base relation (transactions).

If the planner believes about `100k` rows will remain after the filter, the picture it sees is:

1. Quickly fetch outer rows from `transactions` via `(transfer_type, route_type)`.
2. For each fetched row, do a cheap index lookup in `compliance_docs(transaction_id)`.
3. Since the outer side is "small", the cost of `100k` B-Tree probes looks acceptable.

The plan on paper is roughly:

```text
Nested Loop
  -> Bitmap Heap Scan on transactions t
       rows=100000
  -> Index Scan using idx_compliance_docs_txid on compliance_docs d
       Index Cond: (transaction_id = t.id)
```

But the outer side is actually not `100k`, it's `1,000,000`. The error in the base relation estimate propagates further and turns a sensible `100k` nested lookup into `1M` separate index probes.

An important caveat: the node reading `transactions` isn't required to be specifically `Index Scan`. For 10% of the table, the planner may well choose `Bitmap Index Scan + Bitmap Heap Scan`, and on very wide rows or different `cost settings` even `Seq Scan`. The problem isn't the specific scan node. The problem is that the join stage receives an order of magnitude more outer rows than expected.

### What "1 million index lookups" means physically

This isn't one big linear operation. It's a million separate traversals of the B-Tree:

1. descend from root to leaf;
2. search for the required `transaction_id`;
3. read the corresponding TID;
4. jump to the heap tuple or the index-only path, if you're lucky;
5. repeat this again.

If the working set fits in memory, the query often becomes **CPU-bound**: too many short, poorly predicted operations, too many buffer pin/unpin, too much `LWLock` and latch activity around hot pages and metadata in `shared_buffers`. If the working set doesn't fit in memory, random I/O is added on top of that.

That's exactly why an "index" plan can be orders of magnitude worse than a sequential read. Plans with a linear profile (`Seq Scan` + `Hash Join`) pay for large linear work once. `Nested Loop` — a little, but a million times.

A pseudo-`EXPLAIN ANALYZE` for the bad case:

```text
Nested Loop  (cost=...)
  rows=100000
  actual rows=1000000
  -> Bitmap Heap Scan on transactions t
       rows=100000
       actual rows=1000000
  -> Index Scan using idx_compliance_docs_txid on compliance_docs d
       rows=1
       loops=1000000
```

Note `loops=1000000`.

If the planner had known the real outer-side size in advance, the plan economics would be different:

- one `Hash Join`;
- one sequential scan of the large side;
- one build/probe phase instead of a million B-Tree traversals.

But the incorrect cardinality estimate led to the choice of a suboptimal join algorithm.

## Act 3. False Engineering Intuitions

### Myth 1. "Just add an index"

In this scenario the indexes are already in place:

- a composite index on `transactions(transfer_type, route_type)`;
- an index on `compliance_docs(transaction_id)`.

Moreover, it's precisely the presence of these indexes that mathematically justifies the choice of `Nested Loop`:

- Estimate (`E-Rows = 100k`): the planner calculates that the cost of `100,000` point lookups (`Index Probes`) is lower than the overhead of allocating memory and building a hash table. Thanks to the low base cost of index traversal, `Nested Loop` ends up with a smaller total cost.

- Execution (`A-Rows = 1M`): the `10x` estimate error leads to a proportional growth in loop iterations. The engine does `1,000,000` `B-Tree` traversals instead of `100,000`, which is already more expensive than `Hash Join` in resource terms.

Thus, the index doesn't create the problem, but its low estimated cost masks the planner's mathematical error and leads to the choice of an inefficient execution plan.

### Myth 2. "We load-tested this, it looked fine"

A typical synthetic dataset is often generated via something like:

```sql
transfer_type = CASE WHEN random() < 0.1 THEN 'SWIFT' ELSE 'SEPA' END
route_type = CASE WHEN random() < 0.1 THEN 'CROSS_BORDER' ELSE 'DOMESTIC' END
```

This kind of generation makes the columns statistically independent. That is, the tests artificially satisfy the planner's assumption:

```text
P(A ∩ B) ≈ P(A) × P(B)
```

Then `E-Rows` will indeed be close to `A-Rows`, the planner will pick the "right" plan, and the tests will report everything as stable.

After rollout to production, the real business rule appears:

```text
IF transfer_type = 'SWIFT'
THEN route_type = 'CROSS_BORDER'
```

And the planner starts to systematically underestimate the outer side. Put differently, the load-testing environment validated not the real workload, but an artificial random data model.

That's how problems like this slip into production: the test volume is realistic, but the real-world data correlations are not.

## Act 4. Solution: extended statistics via MCV

The fix looks like this:

```sql
CREATE STATISTICS tx_correlation (mcv)
ON transfer_type, route_type
FROM transactions;

ANALYZE transactions;
```

What does `mcv` do?

Ordinary column statistics store popular values separately per column. Extended statistics of type **MCV (Most Common Values)** store a compact list of the most frequent value combinations and their frequencies. This is not a full 2D matrix over all possible pairs. It's metadata specifically about those value pairs that turned out to be frequent enough to make it into the multivariate statistics.

In our example's terms, the planner stops looking only at:

```text
P(transfer_type = 'SWIFT')
P(route_type = 'CROSS_BORDER')
```

in isolation and starts to know that the pair itself

```text
('SWIFT', 'CROSS_BORDER') -> 0.1
```

occurs noticeably more often than the product of marginal probabilities predicts. In other words, the **Independence Assumption** is lifted not for the whole table in general, but only for those combinations that actually made it into the MCV list.

After `ANALYZE` the estimate changes conceptually like this:

```text
Bitmap Heap Scan on transactions t
  ...
  rows=1000000
```

One important precision here: `MCV` does not fix `join selectivity` directly. It fixes the estimate for the `transactions` base relation, lifting the false assumption of independence between columns within a single table.

Next, the planner simply recomputes the join economics. With `1,000,000` outer rows, `Nested Loop` no longer looks profitable. Under that estimate, `Hash Join` or another plan with a more linear profile often becomes cheaper.

There's one more practical caveat here. `MCV` only helps if the required combination actually made it into the statistics object. In our toy example this is almost guaranteed: the pair `('SWIFT', 'CROSS_BORDER')` covers 10% of the table and is sufficiently "hot". But in real-world schemas with weaker correlation or a large number of competing hot combinations you may need to raise the `STATISTICS` target, otherwise `ANALYZE` simply won't store a sufficiently detailed MCV list.


It's important to correctly scope the effect. Extended statistics in PostgreSQL are not used directly for **join selectivity estimation**. They are used for more accurate estimation of **base relation predicates**. But in this scenario that's enough, because it was precisely the error in the size of the filtered `transactions` that was breaking the join strategy choice.

## Trade-off and decision rule

`MCV` shouldn't be attached to every pair of columns "just in case".

Extended statistics come at a cost:

- `ANALYZE` becomes heavier;
- planning gets a bit more work to do;
- system catalogs `pg_statistic_ext` and `pg_statistic_ext_data` grow.

That cost is usually small compared to the gain from a corrected plan, but it isn't zero.

The practical rule is simple:

- the columns must have strong correlation or near-functional dependency;
- these columns must actually be used together in `WHERE`;
- misestimation must negatively affect the physical plan shape, usually through the join algorithm choice;
- only then does `CREATE STATISTICS ... (mcv)` pay off.

Typical candidates:

- `status + type`;
- `country + city`;
- `payment_provider + payment_method`;

If the columns are rarely used together in predicates, or the cardinality error doesn't affect the physical plan, MCV will just be extra noise in the metadata.

## Historical context

`CREATE STATISTICS` appeared before PostgreSQL 12, but **multi-column MCV statistics** specifically were added in PostgreSQL 12.

- before PostgreSQL 12, engineers had no built-in MCV mechanism for frequencies of specific multi-column combinations;
- in some cases they used expression indexes, materialized helper columns, or other workarounds;
- these workarounds carried a write penalty, consumed RAM and disk, and added operational complexity;
- metadata-level MCV has no such write penalty, because it doesn't alter the row write path — it only improves the planner's statistical model after `ANALYZE`.

## Measured behavior on real hardware

On a Linux machine fairly close to a typical server profile, this synthetic case manifests exactly as the article predicts. On a laptop with `16 GB RAM`, `AMD Ryzen 5 4500U`, `SSD`, `Debian 13`. PostgreSQL runs in `docker-compose` with a `1 GB` memory limit — so that the working set doesn't fit entirely in cache and the I/O profile stays realistic. On `10M`-row tables, the baseline scenario (averaged over 5 runs) produced:

```text
before MCV:
Nested Loop
rows=99734
actual rows=1000000
Execution Time: ~120000 ms

after MCV:
Hash Join
rows=985001
actual rows=1000000
Execution Time: ~40000 ms
```

This is the canonical outcome: a `10x` underestimate on the base relation, `Nested Loop -> Hash Join` after `MCV`, and an already clear wall-clock win for the more linear plan.

On very fast laptop-class hardware the picture can be less clean. On a MacBook with Apple Silicon the planner bug remains visible: `E-Rows` are underestimated, the join strategy changes after `MCV`, but the improvement from switching to `Hash Join` may be less noticeable, or even lose in wall-clock time to `Nested Loop`. This doesn't refute the mechanism. It's a reminder that the result depends not only on the abstract math of the planner cost model, but also on the physics of specific hardware: CPU cache locality, the state of the page cache, and the disk subsystem.

There's also a more practical caveat here. For latency benchmarking of databases and brokers, macOS — including Apple Silicon — shouldn't be considered a reference environment. Docker Desktop runs inside a Linux VM, and results depend more heavily on the hypervisor, the page cache, and host storage behavior than they do on native Linux. Canonical measurements are better taken on Linux hardware close to production. In this context, a Mac is useful for local diagnostics, plan-shape verification, and quick mechanism validation, but not as a definitive source of latency conclusions.

## Conclusion

In this class of problems, the solution is not an index, but a correct statistical model of the data.

As long as the planner treats correlated predicates as independent, it underestimates `Estimated Rows` at the base relation level, which causes it to underestimate the outer-side size and pick `Nested Loop` where the execution physics already demands algorithms with linear complexity (e.g., Hash Join). Indexes only make this wrong choice even more compelling in the cost model.

If the business data contains stable multi-column correlations and the cardinality estimate error changes the join strategy, then what needs fixing is not the `JOIN` and not the index, but the selectivity model itself, via `CREATE STATISTICS ... (mcv)`.

The full code for table initialization, synthetic skewed data generation, and the SQL queries for reproduction are available in the [GitHub repository](https://github.com/nikita-kibitkin/postgres-mcv-demo).
