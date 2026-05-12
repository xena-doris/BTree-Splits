# Experiment 2: Effect of Cache Size on SQLite B-Tree Splits and Performance

---

# Overview

## Objective

The purpose of this experiment is to analyze how changing SQLite’s **cache size** affects:

* B-Tree split behavior
* Insert performance
* Update performance
* Page access efficiency
* Overall database execution time

SQLite internally maintains a **page cache** that stores recently accessed database pages in memory.

When SQLite performs operations such as:

* INSERT
* UPDATE
* DELETE
* B-Tree balancing

it repeatedly accesses database pages.

If these pages are already cached in memory:

* Disk I/O is reduced
* Page fetches become faster
* B-Tree balancing becomes more efficient

This experiment helps demonstrate how memory allocation through cache size impacts SQLite’s internal B-Tree operations.

---

# What is SQLite Cache Size?

SQLite uses a page cache to store database pages temporarily in RAM.

The cache contains:

* Recently used B-Tree pages
* Internal nodes
* Leaf nodes
* Overflow pages

The cache size is controlled using:

```sql
PRAGMA cache_size = N;
```

Where `N` represents the number of pages SQLite can cache.

---

# Why Cache Size Matters

## Small Cache Size

A small cache means:

* Pages are evicted frequently
* SQLite must reload pages from disk often
* B-Tree operations require more disk reads
* Updates become slower

Result:

* Increased I/O overhead
* Slower execution
* Higher page replacement activity

---

## Large Cache Size

A large cache means:

* More B-Tree pages remain in memory
* SQLite reuses pages efficiently
* Disk access is reduced
* Updates and inserts execute faster

Result:

* Better locality
* Reduced page fetching cost
* Improved performance

---

# Experiment Setup

## Table Schema

```sql
CREATE TABLE t(
    id INTEGER PRIMARY KEY,
    data BLOB
);
```

---

# Workload Used

## Insert Workload

50,000 rows inserted using recursive SQL generation.

Each row contained:

* Random integer key
* 1000-byte blob

```sql
WITH RECURSIVE cnt(x) AS (
    SELECT 1
    UNION ALL
    SELECT x+1 FROM cnt WHERE x < 50000
)
INSERT INTO t(id, data)
SELECT abs(random()), randomblob(1000) FROM cnt;
```

This workload heavily stresses:

* B-Tree insertions
* Page allocation
* Node balancing
* Cache utilization

---

## Update Workload

20,000 random rows were updated.

```sql
UPDATE t
SET data = randomblob(1000)
WHERE id IN (
    SELECT id FROM t ORDER BY random() LIMIT 20000
);
```

This workload stresses:

* Random page access
* Cache reuse efficiency
* B-Tree traversal performance

---

# Part 1 — Small Cache Size (50 Pages)

## Purpose

This experiment analyzes SQLite behavior under extremely constrained cache memory.

Configuration used:

```sql
PRAGMA cache_size = 50;
```

---

# Screenshot

### Cache Size = 50

![Cache Size 50](cache%20size%2050.png)

---

# Observed Results

## Insert Phase

| Metric             | Value        |
| ------------------ | ------------ |
| Pages After Insert | 14535        |
| Total Splits       | 71920        |
| Insert Runtime     | 1.298775 sec |

---

## Update Phase

| Metric             | Value        |
| ------------------ | ------------ |
| Pages After Update | 14535        |
| Total Splits       | 8506         |
| Update Runtime     | 2.082996 sec |

---

# Analysis

With only 50 cached pages:

* SQLite constantly evicted pages from memory
* Frequently accessed B-Tree nodes had to be reloaded repeatedly
* Random updates became very expensive

---

## Key Observations

### 1. Slowest Update Performance

Update runtime:

```text
2.08 seconds
```

This was the slowest among all configurations.

Reason:

* Random updates required repeated page fetches
* Cache misses became extremely frequent

---

### 2. Very High Split Activity

Insert split count:

```text
71920
```

Update split count:

```text
8506
```

Although cache size does not directly determine splits, poor cache locality increases:

* Page movement overhead
* Balancing operations
* Internal node accesses

---

## Interpretation

Small cache sizes cause:

* Heavy disk I/O
* Poor locality
* Slow random access performance
* Increased memory pressure

---

# Part 2 — Medium Cache Size (2000 Pages)

## Purpose

This experiment evaluates SQLite under a moderate cache configuration.

Configuration used:

```sql
PRAGMA cache_size = 2000;
```

---

# Screenshot

### Cache Size = 2000

![Cache Size 2000](Cache%20Size%202000.png)

---

# Observed Results

## Insert Phase

| Metric             | Value        |
| ------------------ | ------------ |
| Pages After Insert | 14549        |
| Total Splits       | 71612        |
| Insert Runtime     | 1.212883 sec |

---

## Update Phase

| Metric             | Value        |
| ------------------ | ------------ |
| Pages After Update | 14549        |
| Total Splits       | 8488         |
| Update Runtime     | 0.682186 sec |

---

# Analysis

Increasing cache size significantly improved performance.

---

## Key Observations

### 1. Massive Improvement in Update Speed

Update runtime improved from:

```text
2.08 sec → 0.68 sec
```

This demonstrates:

* Better page reuse
* Reduced disk reads
* Improved cache locality

---

### 2. Slight Reduction in Splits

Insert splits:

```text
71920 → 71612
```

Update splits:

```text
8506 → 8488
```

The reduction is small because:

* Cache size does not alter page capacity
* Physical B-Tree structure remains unchanged

---

## Interpretation

A moderate cache size provides:

* Better in-memory page reuse
* Faster random updates
* Lower page fetch overhead

This configuration already delivers major performance gains.

---

# Part 3 — Large Cache Size (10000 Pages)

## Purpose

This experiment investigates the effect of allocating a very large page cache.

Configuration used:

```sql
PRAGMA cache_size = 10000;
```

---

# Screenshot

### Cache Size = 10000

![Cache Size 10000](Cache%20Size%2010000.png)

---

# Observed Results

## Insert Phase

| Metric             | Value        |
| ------------------ | ------------ |
| Pages After Insert | 14452        |
| Total Splits       | 71508        |
| Insert Runtime     | 1.042348 sec |

---

## Update Phase

| Metric             | Value        |
| ------------------ | ------------ |
| Pages After Update | 14452        |
| Total Splits       | 8144         |
| Update Runtime     | 0.679072 sec |

---

# Analysis

This configuration produced the best overall performance.

---

## Key Observations

### 1. Fastest Insert Runtime

Insert runtime improved to:

```text
1.04 seconds
```

This indicates:

* Efficient page reuse
* Minimal disk fetches
* Better B-Tree traversal efficiency

---

### 2. Fastest Update Runtime

Update runtime:

```text
0.67 seconds
```

Compared to small cache:

```text
2.08 sec → 0.67 sec
```

This is a major improvement.

---

### 3. Lowest Split Count

Insert splits:

```text
71508
```

Update splits:

```text
8144
```

This is the lowest among all configurations.

---

## Interpretation

Large caches improve:

* Page locality
* B-Tree traversal speed
* Internal node reuse
* Update efficiency

SQLite spends less time loading pages and more time executing operations.

---

# Comparative Analysis

| Cache Size | Insert Runtime | Update Runtime | Insert Splits | Update Splits |
| ---------- | -------------- | -------------- | ------------- | ------------- |
| 50         | 1.298775 sec   | 2.082996 sec   | 71920         | 8506          |
| 2000       | 1.212883 sec   | 0.682186 sec   | 71612         | 8488          |
| 10000      | 1.042348 sec   | 0.679072 sec   | 71508         | 8144          |

---

# Main Observation

Increasing cache size dramatically improved execution time, especially for updates.

However:

* Split counts changed only slightly
* Total pages remained nearly identical

This confirms that:

> Cache size mainly affects memory efficiency and I/O behavior, not physical B-Tree capacity.

---

# Key Findings

## Small Cache Sizes

Cause:

* Frequent page eviction
* Higher disk I/O
* Poor update performance

---

## Large Cache Sizes

Provide:

* Faster page reuse
* Better locality
* Improved update performance
* Lower page fetch overhead

---

# Important Insight

The biggest improvement was observed during random updates.

Reason:

* Random updates repeatedly access scattered B-Tree pages
* Larger caches keep more pages in memory
* Disk reloads are minimized

This strongly demonstrates the importance of cache locality in database systems.

---

# Final Conclusion

This experiment proves that SQLite cache size has a major impact on database execution performance.

The results demonstrate:

* Larger caches significantly improve INSERT and UPDATE speed
* Cache size has only minor impact on actual B-Tree split counts
* Performance gains come primarily from reduced disk I/O and better page reuse