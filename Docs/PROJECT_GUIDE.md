# LRU-C Project Guide

## What this project is

This repository is a MySQL Server 5.6.26 source tree with a focused InnoDB buffer pool modification called LRU-C. The upstream MySQL codebase is large, but the project-specific changes are intentionally small and are concentrated in the InnoDB buffer manager.

LRU-C stands for "least-recently-used clean". The idea is to keep track of an old clean page in the InnoDB LRU list and use that page as an eviction victim when a foreground thread needs a free buffer frame. By preferring clean victims, the page miss path can avoid waiting for a dirty page write before reading the requested page.

## Where the custom code lives

The README states that the LRU-C changes are only in these files:

| File | Role |
| --- | --- |
| `storage/innobase/buf/buf0lru.cc` | LRU list eviction path and oldest-clean-page selection logic |
| `storage/innobase/buf/buf0flu.cc` | Flush-list and LRU-flush behavior, including batch flushing near the LRU tail |
| `storage/innobase/buf/buf0buf.cc` | Buffer pool initialization and cleanup for new LRU-C state |
| `storage/innobase/include/buf0lru.h` | Declarations for LRU-C helper functions |
| `storage/innobase/include/buf0buf.h` | Extra fields in buffer page and buffer pool structures |

Most project-specific comments are marked with `FOR LRU-C`, `For LRU-C`, or `lbh`.

## Main runtime components

When MySQL starts, InnoDB initializes one or more buffer pool instances. This patched tree adds extra state to each buffer pool instance:

- `LRU_oldest_clean_page`: pointer to the current old clean page candidate.
- `reached_LRU_oldest_clean_page`: marker used by flushing code to avoid flushing past the selected clean page.
- `LRU_dirty_tail_list_mutex`: extra mutex used around LRU tail scanning and dirty-tail related logic.
- `b_event`, `f_event`, `batch_running`, and `flush_running`: coordination fields added for batch and flush behavior.

Each buffer page also gets:

- `LRU_batch_write_victim`: marks pages selected by LRU-C batch flushing.
- `aio_write_finished`: tracks whether the asynchronous write for that batch victim has completed.

## High-level behavior

In regular MySQL/InnoDB, when a query needs a page that is not already in memory, InnoDB must find a free buffer frame. If the LRU tail contains dirty pages, foreground work may indirectly wait for writes or repeatedly scan the LRU list.

LRU-C changes that flow:

1. A page miss needs a free block.
2. InnoDB first tries the normal free list.
3. If the free list is empty, LRU-C checks whether `LRU_oldest_clean_page` is still valid.
4. If valid, it frees that clean page immediately and retries allocation.
5. If invalid or missing, it scans the old part of the LRU list to find a new old clean page.
6. Background flushing tries to keep dirty pages near the LRU tail moving toward clean state, but stops before crossing the tracked clean page.

The intended effect is more parallel I/O on SSDs: reads for cache misses can proceed without serializing behind single dirty-page writes, while dirty pages are flushed in larger batches in the background.

## Repository layout

This is mostly a standard MySQL source tree:

| Directory | Purpose |
| --- | --- |
| `sql/` | SQL layer, parser/executor/server core |
| `client/` | MySQL command-line clients such as `mysql`, `mysqldump`, and admin tools |
| `storage/innobase/` | InnoDB storage engine, including the LRU-C changes |
| `storage/` | Other storage engines such as MyISAM, CSV, MEMORY/HEAP, Archive |
| `mysys/`, `strings/`, `vio/` | MySQL portability, string, and I/O support libraries |
| `mysql-test/` | MySQL regression test suite |
| `unittest/` | Unit tests |
| `scripts/` | Generated/install scripts such as `mysql_install_db` |
| `support-files/` | Example configs and service scripts |
| `Docs/` | Legacy MySQL documentation files |
| `docs/` | Human-readable project notes added for this repository |

## What this project is not

This is not a standalone LRU simulator and it does not expose a separate LRU-C command. You build and run it as a MySQL server. The LRU-C behavior happens inside InnoDB when the server is running real database workloads.

