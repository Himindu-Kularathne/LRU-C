# LRU-C Internals

## The problem LRU-C targets

InnoDB keeps database pages in a buffer pool. When a requested page is not in memory, the server must find a free buffer frame before it can read the page from disk.

Traditional LRU eviction can run into two serialization points:

- The LRU tail may contain dirty pages that must be flushed before reuse.
- Scanning and modifying shared LRU structures can create mutex contention.

On flash SSDs, serialized I/O leaves device parallelism unused. LRU-C tries to make more page reads and page writes proceed independently.

## Core idea

LRU-C tracks a clean page in the old part of the InnoDB LRU list:

```text
LRU head                                            LRU tail
most recently used                         least recently used
|-------------------------------------------------------------|
                         old sublist          dirty/clean tail
                                             ^
                                             LRU_oldest_clean_page
```

When a foreground thread needs a free block:

1. Use the free list if possible.
2. Otherwise, validate `LRU_oldest_clean_page`.
3. If it is still clean, unpinned, not under I/O, and still in the LRU list, evict it.
4. If it is invalid, scan the old LRU region to find another clean page.

This avoids choosing a dirty page on the critical path.

## Modified data structures

`storage/innobase/include/buf0buf.h` adds page-level fields:

```c
ibool LRU_batch_write_victim;
ibool aio_write_finished;
```

These are used to avoid treating a page selected for batch write as a reusable clean page until the asynchronous write has really finished.

The same header adds buffer-pool-level fields:

```c
buf_page_t* LRU_oldest_clean_page;
ibool reached_LRU_oldest_clean_page;
os_event_t b_event;
os_event_t f_event;
ibool batch_running;
ibool flush_running;
UT_LIST_BASE_NODE_T(buf_page_t) LRU_dirty_tail_list;
ib_mutex_t LRU_dirty_tail_list_mutex;
```

The most important field is `LRU_oldest_clean_page`. The mutex and flags coordinate LRU tail scanning and flushing.

## Initialization

`storage/innobase/buf/buf0buf.cc` initializes the added buffer pool fields during buffer pool instance creation:

- `LRU_oldest_clean_page = NULL`
- `reached_LRU_oldest_clean_page = false`
- `batch_running = false`
- `flush_running = false`
- `b_event` and `f_event` are created

On buffer pool instance teardown, the added events are freed.

## Foreground allocation path

The main path is in `storage/innobase/buf/buf0lru.cc`, inside `buf_LRU_get_free_block()`.

The flow is:

1. Enter the buffer pool mutex.
2. Try `buf_LRU_get_free_only()` to take a page from the normal free list.
3. If the free list succeeds, return that block.
4. Otherwise call `buf_oldest_clean_page_is_valid()`.
5. If valid, use `buf_LRU_free_page()` on `LRU_oldest_clean_page`.
6. If invalid, call `buf_update_oldest_clean_page(buf_pool, false)`.
7. If a limited scan fails, release the buffer pool mutex and call `buf_update_oldest_clean_page(buf_pool, true)` for a broader scan.
8. Retry allocation after freeing a clean page.

The key validation checks are:

- The pointer is not `NULL`.
- The page has `oldest_modification == 0`, meaning it is clean.
- The page is not buffer-fixed.
- The page has no pending I/O fix.
- The page is still a file page in the LRU list.
- The page is not a batch-write victim whose asynchronous write has not finished.

## Finding the next clean page

`buf_update_oldest_clean_page()` searches the old LRU region for a candidate. It usually starts near the current `LRU_oldest_clean_page` and moves backward through the list. If the pointer is missing, too close to the boundary, or a full scan is requested, it starts at the LRU tail.

A candidate must be:

- Clean.
- Unpinned.
- Not under I/O.
- In the old LRU sublist.
- In a file-backed page state.
- Not the active flush-list high-priority page.
- Not an unfinished batch-write victim.

If it finds one, it stores that page in `buf_pool->LRU_oldest_clean_page`.

## Background flushing changes

`storage/innobase/buf/buf0flu.cc` changes LRU flushing so the background thread can clean dirty pages near the LRU tail without crossing the tracked clean page.

The modified LRU batch scan:

- Starts at the LRU tail.
- Stops at `LRU_old`, `LRU_oldest_clean_page`, or when enough scanning has happened.
- Frees pages that are already clean and replaceable.
- Flushes dirty pages that are ready for LRU flush.
- Marks flushed pages as `LRU_batch_write_victim` until their async write is complete.

This is the dynamic batch write part of the design: dirty pages near the eviction area are pushed out in batches so foreground page misses are more likely to find clean victims.

## Why this can improve throughput

The foreground read path is latency-sensitive. If it must first flush a dirty victim, then the read waits behind a write. LRU-C tries to decouple those operations:

- Foreground page miss: evict an already-clean old page and submit the read.
- Background cleaner: flush dirty tail pages in batches so future victims become clean.

On SSDs, this can improve throughput because the storage device can process more independent I/O instead of receiving a serialized write-then-read sequence.

## Tradeoffs

LRU-C may evict a clean page that is not the absolute LRU tail page. That can slightly reduce hit ratio. The paper/project argument is that improved I/O parallelism can more than compensate for the reduced hit ratio on flash SSD workloads.

There is also added synchronization complexity around the LRU tail and clean-page pointer. The code adds extra state checks to avoid reusing pages that are pinned, dirty, under I/O, or in the middle of batch writing.

## Reading path checklist

To continue studying the implementation, read in this order:

1. `storage/innobase/include/buf0buf.h`: added fields on `buf_page_t` and `buf_pool_t`.
2. `storage/innobase/buf/buf0buf.cc`: initialization and cleanup.
3. `storage/innobase/include/buf0lru.h`: helper declarations.
4. `storage/innobase/buf/buf0lru.cc`: clean-page validation, clean-page search, and free-block path.
5. `storage/innobase/buf/buf0flu.cc`: LRU flushing and batch write behavior.

