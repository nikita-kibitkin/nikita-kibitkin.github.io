---
title: "The OLTP to OLAP Mindset Trap: Why Your PostgreSQL SQL Will Kill ClickHouse"
date: 2026-04-17T00:00:00+02:00
draft: true
description: "Why PostgreSQL mental models break on ClickHouse, how CTE isolation around JOINs blocks global pushdown, and why explicit scan narrowing changes the cost profile by orders of magnitude."
tags:
  - clickhouse
  - postgresql
  - sql-optimization
  - olap
  - query-planner
  - data-engineering
categories:
  - Databases
  - Analytics Engineering
icon: "/images/icons/squares-accent.svg"
showToc: true
canonicalURL: "https://nikita-kibitkin.github.io/posts/oltp-to-olap-mindset-trap/"
---

You know PostgreSQL inside out. You know when the planner picks a *Nested Loop* and when it flips to a *Hash Join*. You've learned to trust the optimizer: write declarative SQL and the database figures out the rest.

Then you move your analytics to ClickHouse. You write a perfectly logical query, a `LEFT JOIN` to enrich a narrow slice of data. You run it and get `Memory limit (for query) exceeded`.

Your SQL isn't broken. The result it describes is correct. The problem is that your mental model of execution (OLTP) has collided with the unforgiving mechanics of a columnar database (OLAP). And in ClickHouse, that gap costs you gigabytes of RAM.

## The Code Trap

We have a `users` dimension, say 1,000,000 users. And a huge `events` fact table with billions of rows. These events stream in with *at-least-once* delivery semantics, meaning the same `event_id` might be written multiple times due to producer retries or network interruptions.

To handle this idempotency while keeping scans prunable by `user_id`, the table uses `ReplacingMergeTree` with an `ORDER BY` that starts with `user_id` and includes `event_id`. This forces us to use `FINAL` to get an honest deduplicated stream, and true unique event counts, before aggregation. We want to compute these exact event counts for a narrow group of **50 users** whose account status is `BANNED`.

```sql
WITH target_users AS (
    -- Narrow selection: 50 users out of 1,000,000
    SELECT user_id, country
    FROM users
    WHERE status = 'BANNED'
),
user_events AS (
    -- The trap: we aggregate across ALL billions of events
    SELECT user_id,
           count() AS total_events
    FROM events FINAL
    GROUP BY user_id
)
SELECT t.user_id,
       t.country,
       e.total_events
FROM target_users AS t
LEFT JOIN user_events AS e ON t.user_id = e.user_id;
```

In Postgres, the planner would notice that `target_users` emits only 50 rows and rewrite the query so the restriction on `user_id` reaches the scan that feeds the aggregation.

ClickHouse executes the `user_events` CTE in isolation. It reads **every one** of those billions of events, aggregates counts for **all** 1,000,000 users, loads the million rows into RAM into the hash table on the build side, and only then filters down to the 50 you actually wanted.

## Under the Hood

### 1. Query flattening vs. Isolated execution

PostgreSQL aggressively does **query flattening**. It can inline a non-materialized CTE into one execution tree, combine the join and aggregate into the same plan, and turn the outer restriction into an index-driven lookup at the scan boundary.

ClickHouse's problem here is not that `GROUP BY` is inherently opaque. While modern ClickHouse, especially with the new Analyzer, is getting smarter at reordering simple joins, it still treats strict aggregation barriers and isolated CTEs as hard boundaries. It evaluates the right side of the `JOIN`, our CTE, as an independent block. It runs the full aggregation over the fact table, materializes that result in memory, and only then applies the outer join condition. The engine will not magically flatten an aggregation-heavy query to rescue a bad scan; it expects you to explicitly bound the scan scope.

Result: the `user_events` CTE runs over **every row** of `events`, producing one aggregated row per user, 1,000,000 of them.

### 2. HashJoin materializes the right side entirely in RAM

ClickHouse's default join strategy is **parallel hash join** (or standard `HashJoin` on older releases). The mechanic is blunt:

1. Read the **right** side of the join to completion.
2. Build an in-memory hash table keyed by the join columns.
3. Stream the left side through the hash table, probing for matches.

The right side lives in RAM for the duration of the query. If the right side is an aggregate over 1,000,000 users, the hash table holds 1,000,000 entries.

#### The RAM math

If `user_events` aggregates the entire user base, ClickHouse builds a hash table with 1,000,000 keys. Add the weight of each aggregation state and per-entry overhead, and you're allocating hundreds of megabytes, sometimes gigabytes, just to hold the right side of the join.

To return 50 rows, the engine builds a million-element structure in memory and throws away 99.9% of the work at the probe stage. Line up ten analysts running variants of this report in parallel and every one of those queries hits the server's `max_memory_usage` ceiling and dies.

PostgreSQL on the same task: a Nested Loop with an index seek on `events(user_id)` 50 times, O(1) memory per iteration. Microseconds. Megabytes.

Same SQL. Same result. Several orders of magnitude apart in resource cost.

### 3. `FINAL` on `ReplacingMergeTree` amplifies everything else

`ReplacingMergeTree` does not remove duplicates at write time. To get the latest logical state, `FINAL` forces **merge-on-read**: for every key, the engine reads active part versions, compares them, keeps the latest, and only then produces the deduplicated stream.

That changes the cost profile completely. A normal wide scan is often I/O-bound: the engine streams compressed columns from disk and aggregates them at gigabytes per second. `FINAL` turns the same scan CPU-bound because every row now participates in key comparison and version resolution. On billions of rows without a sort-key filter, this is exactly what makes the full scan fatal.

In fairness: since 22.x, and especially after improvements in 23.x, `FINAL` reads are parallelized. ClickHouse runs merge-on-read across parts concurrently and knows how to skip parts that have nothing to deduplicate. That genuinely lowered `FINAL`'s cost on the happy path. But executing a bad idea in parallel doesn't turn it into a good idea: if your `FINAL` still scans the whole fact table because there's no sort-key `WHERE`, you're just burning more cores and more disk bandwidth at once. CPU and I/O quotas saturate just as reliably as they did in single-threaded mode.

Without a `WHERE` on the sort-key prefix, `FINAL` scans every granule of every active part. With a narrow `WHERE`, it only touches the parts where matching rows could live. Reading billions of rows with `FINAL` and no sort-key pruning routinely costs 10x to 100x more CPU than the same scan without deduplication.

Stacked on top of the full CTE scan and a hash table holding 1,000,000 keys, you end up with a query that reads billions of rows, deduplicates them all, aggregates them all, and builds a giant hash table, to produce a 50-row result.

## The Fix

One line, inside the CTE:

```sql
WITH target_users AS (
    SELECT user_id, country
    FROM users
    WHERE status = 'BANNED'
),
user_events AS (
    SELECT user_id,
           count() AS total_events
    FROM events FINAL
    -- Physically narrow the scan scope; the whole story
    WHERE user_id IN (SELECT user_id FROM target_users)
    GROUP BY user_id
)
SELECT t.user_id,
       t.country,
       e.total_events
FROM target_users AS t
LEFT JOIN user_events AS e ON t.user_id = e.user_id;
```

Same result. Several orders of magnitude cheaper. Here's why.

## From Declarations to Physical Execution

### Data skipping via the sparse primary index

`events` is sorted by an `ORDER BY` whose leading column is `user_id`. ClickHouse keeps a sparse primary index, one entry per granule (8,192 rows by default), recording the min/max of sort-key columns. When a query contains `WHERE user_id IN (...)`, the engine:

1. Materializes the `IN` set as a `HashSet<UInt64>`.
2. Walks the primary index and keeps only granules whose min/max range intersects the set.
3. Reads only those granules from disk, and only the columns the query needs.

For a narrow `IN` over billions of rows, this routinely cuts the read volume by four to six orders of magnitude. It's the single most important optimization in ClickHouse, and it requires an explicit `WHERE`, the optimizer won't derive it for you.

**Crucial Caveat:** For this skipping mechanic to work effectively, the filtered column, `user_id`, must be a **leading column** in the table's `ORDER BY` tuple. If your primary key is `(timestamp, user_id)`, filtering by `user_id` alone will not allow the sparse index to prune granules efficiently across a wide time range, and you will degrade back to a heavy scan. Physical declaration means aligning your filters with your physical sort order.

### `FINAL` scope collapses

`FINAL`'s merge-on-read now runs only on the surviving granules, the handful of parts where the target users' data actually lives. No deduplication across the rest of the fact table.

### HashJoin build side becomes trivial

`user_events` now contains 50 rows instead of 1,000,000. The hash table fits in L1 cache. Concurrent queries stop fighting over memory.

### Why `IN`, not an `INNER JOIN`?

It's tempting to fix the memory issue by replacing the outer `LEFT JOIN` with an `INNER JOIN target_users` directly inside the CTE. Semantically, it works. Physically, it's unreliable.

Modern ClickHouse does have Runtime Join Filters, like Bloom-filter prefiltering, which attempt to use the right-side join keys to discard rows early. However, this optimization happens in memory *during* the execution phase. If conditions aren't perfect, ClickHouse falls back to its default behavior: physically reading billions of rows, pushing them through decompression and CPU, building the tiny hash table for the 50 users, and throwing away 99.9% of the stream at the probe stage.

`WHERE x IN (subquery)` relies on a much stronger physical contract: **Data Skipping**.

1. ClickHouse evaluates the subquery first and builds a constant `Set`.
2. It passes this Set down to the storage layer.
3. *Before* reading any data, it checks the table's sparse Primary Index. If a granule's min/max range doesn't intersect with the Set, it completely skips reading that granule from disk.

A `JOIN` is an opportunistic filter during execution. An `IN` clause is a deterministic storage-level prune.

### Cluster caveat: `IN` vs `GLOBAL IN`

Everything above works perfectly on a single local shard. But in most production setups, `events` is a `Distributed` table on top of `*_local` replicas across N shards, and that hides a second trap.

The correct form depends on your data distribution strategy.

If `users` and `events` are strictly **colocated**, meaning they are distributed by the same hashing key, for example `user_id`, a plain `IN` is **usually the ideal choice**. The query typically executes locally on each shard, matching local users to local events.

But if `users` is not distributed by the same key, or lives only on the initiator node, the plain `IN` fails or amplifies work. In this case, you must use:

```sql
WHERE user_id GLOBAL IN (SELECT user_id FROM target_users)
```

`GLOBAL IN` changes the semantics: it runs the subquery *once* on the initiator, serializes the result into a temporary table, and broadcasts that table to every shard as a constant.

**The Refined Rule:** If your tables are not colocated by the exact same shard key, any subquery restriction across a `Distributed` table must use `GLOBAL IN`. Getting this wrong costs you either empty results or N-fold work amplification across the cluster.

## The Rule of Thumb

> Every `FROM {large_table}` inside a CTE or subquery must have a `WHERE` clause that narrows it to the data the outer query actually needs. In an aggregation-heavy isolated CTE, do not expect the optimizer to derive that `WHERE` for you. Leaving a large-table scan unbounded behind an aggregation barrier is a bug.

That's the mindset shift. Here it is as a table, the single piece of this article worth saving:

| Axis | PostgreSQL (OLTP) | ClickHouse (OLAP) |
| --- | --- | --- |
| Role of `WHERE` inside a subquery | hint to the planner | **physical declaration of scan scope** |
| Default JOIN strategy | cost-based: Nested Loop / Merge / Hash | **Parallel hash join** (by default), build side materialized in RAM |
| Predicate pushdown | yes | **Limited (CTE & JOIN barriers block global pushdown)** |
| Preferred JOIN key | anything a B-tree index covers | **Matters less than build-side size; Right table must fit in RAM.** |
| Row deduplication semantics | MVCC, transparent | **For ReplacingMergeTree: FINAL at read time (costs 10x-100x CPU vs scan)** |
| Selectivity estimation | histograms, `pg_statistic` | sparse primary index + MinMax |
| Cost of one redundant input row | nanoseconds (B-tree seek) | **Reading and decompressing at least one full granule (8192 rows)** |

The underlying shift: in PostgreSQL you describe **what** to retrieve and the optimizer decides **how**. In ClickHouse you often still need to describe **what and how to physically read**. Modern releases do reorder joins and apply more rewrites than they used to, but they still won't reliably flatten aggregation-heavy subqueries or rescue an unbounded scan hidden behind an isolated CTE. On a columnar engine, one lazy line of SQL is still measured in gigabytes.

## Takeaway

- ClickHouse does not globally push outer filters through an isolated CTE on the right side of a `JOIN`.
- `HashJoin` materializes the right side of the join entirely in memory.
- `FINAL` without a sort-key `WHERE` scans the whole fact table, parallelism doesn't save you.
- In this query shape, the fix is to push the filter **inside** the CTE, ideally as `WHERE x IN (SELECT ... FROM narrow_cte)`.
- On `Distributed` tables, use `GLOBAL IN` when the restricting subquery is not colocated with the fact table by the same shard key.

You don't need to stop writing joins. You need to stop expecting the optimizer to reliably rescue an unbounded analytical scan.

## Appendix: LinkedIn Hook

Ready-to-post teaser for when this article goes live:

```text
Moving from PostgreSQL to ClickHouse? Your SQL muscle memory might just take down production.

In classic OLTP (Postgres), we lean on the optimizer. It flattens the query and picks efficient index scans. In ClickHouse, a harmless-looking LEFT JOIN against an aggregated CTE with no explicit inner filters can force a full aggregation scan, a massive build side, and eat gigabytes of RAM.

I just wrote up the OLTP to OLAP mindset trap: why isolated CTE execution in ClickHouse changes the plan shape, how ReplacingMergeTree + FINAL multiplies the damage, and the one physical rule that matters whenever a narrow dimension drives a large fact-table scan.

Link in the comments.
```
