# Hybrid LRU-C Experiment Results

## Goal

The goal of this experiment was to compare the original LRU-C buffer
replacement behavior with the hybrid LRU-C improvement.

The hybrid improvement tries to choose eviction victims using more than one
signal:

- Page cleanliness
- Recency of access
- Whether the page is in the old part of the LRU list
- Whether the page can be evicted without waiting for write I/O

The expected benefit is to improve buffer pool hit ratio while still keeping
the LRU-C advantage of preferring clean pages for eviction.

## Experiment Setup

Both versions were tested with the same workload and the same small buffer
pool configuration.

Configuration:

- MySQL/InnoDB 5.6 based LRU-C codebase
- Docker Ubuntu 20.04 test environment
- Buffer pool size: `5M`
- Buffer pool instances: `1`
- `innodb_old_blocks_time=1000`
- Table engine: InnoDB
- Workload: insert rows, update part of the table, then run full scans,
  secondary index scans, and range queries

The small buffer pool was intentional. It creates buffer pressure so eviction
logic is exercised during the test.

## Workload

The test created an InnoDB table with:

- An auto-increment primary key
- A secondary index on `k`
- A large `payload` column

The workload inserted rows by repeatedly doubling the table, updated one
quarter of the rows, then ran repeated primary index scans, secondary index
scans, and range queries.

This workload was chosen because it creates a mix of:

- Clean pages
- Dirty pages
- Recently used pages
- Older pages near the LRU tail
- Pressure to find free buffer blocks

That mix is useful for comparing pure LRU-C and hybrid victim selection.

## Results

| Metric | Pure LRU-C | Hybrid LRU-C | Change |
|---|---:|---:|---:|
| Runtime | 4 sec | 3 sec | 25.00% faster |
| Buffer pool dirty pages | 99 | 55 | 44.44% lower |
| Buffer pool pages flushed | 600 | 664 | 10.67% higher |
| Buffer pool read requests | 26,445 | 26,517 | Similar workload size |
| Buffer pool physical reads | 348 | 311 | 10.63% lower |
| Data writes | 829 | 867 | 4.58% higher |
| Pages written | 737 | 781 | 5.97% higher |
| Hit ratio | 98.6841% | 98.8272% | +0.1431 percentage points |
| Shutdown result | `shutdown_rc=0` | `shutdown_rc=0` | Clean shutdown in both |

Hybrid LRU-C internal stats:

```text
candidates_scanned=131328
clean_victims=1938
dirty_flushed=156
hot_clean_protected=0
fallback_to_lruc=0
no_candidate=114
```

## Interpretation

The hybrid version improved the hit ratio from `98.6841%` to `98.8272%`.
That is a small absolute improvement, but it happened under a short pressure
test with a very small buffer pool. The important supporting metric is that
physical reads dropped from `348` to `311`.

That means the hybrid version needed fewer disk reads for almost the same
number of logical buffer pool read requests. In simple terms, more requested
pages were already in memory.

The hybrid version also selected many clean victims:

```text
clean_victims=1938
```

This shows that the hybrid eviction path was actively used. It did not only
fall back to the old LRU-C behavior:

```text
fallback_to_lruc=0
```

The hybrid version flushed more pages and wrote slightly more pages:

```text
pages_flushed: 600 -> 664
pages_written: 737 -> 781
```

This is acceptable for this experiment because the goal is not only to reduce
I/O count. The goal is to keep useful pages in memory while still maintaining
enough clean pages for eviction. The higher write activity suggests the hybrid
system is doing more background cleaning work, which helps it find clean
eviction victims later.

## Why Hybrid Performs Better Here

Pure LRU-C mainly depends on the clean-page position/pointer and LRU-C clean
page behavior. Under pressure, this can choose a clean page that is not
necessarily the best page to remove from the buffer pool.

Hybrid LRU-C improves this by scanning candidates and scoring them. A page is
a better eviction victim when it is:

- Clean
- In the old part of the LRU list
- Less recently/frequently useful
- Not pinned
- Not already involved in I/O

Because of that, hybrid LRU-C can avoid removing some useful clean pages and
prefer colder clean pages instead. That explains the lower physical reads and
slightly higher hit ratio.

## Conclusion

The hybrid LRU-C improvement is considerable because it improves the main
target metric, buffer pool hit ratio, while preserving the LRU-C design goal
of clean-page eviction.

Summary:

- Hit ratio improved from `98.6841%` to `98.8272%`
- Physical reads reduced from `348` to `311`
- Runtime improved from `4 sec` to `3 sec`
- Hybrid victim selection was actively used
- Both versions shut down cleanly

The result supports the idea that hybrid victim selection can improve hit
ratio without losing the clean-page eviction advantage of LRU-C.

## Notes

This was a short pressure test, so the numbers should be treated as an initial
validation, not a final benchmark. For stronger evidence, the same test should
be repeated multiple times and averaged. A larger benchmark should also be run
with different buffer pool sizes and different read/write mixes.
