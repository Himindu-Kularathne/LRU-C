# MySQL Benchmarking Guide

This document describes the benchmarking setup, workload generation process, metric collection methods, and analysis steps for evaluating MySQL performance using `sysbench`, MySQL Performance Schema, and Linux monitoring tools.

---

# Benchmark Plan

## Objectives

- Measure MySQL throughput and latency
- Evaluate InnoDB buffer pool efficiency
- Analyze CPU, memory, and disk I/O behavior
- Compare performance under different concurrency levels
- Collect reproducible benchmark results

---

# Benchmark Workflow

1. Prepare environment and install required tools
2. Create reproducible benchmark dataset
3. Warm up MySQL buffer pool
4. Run workload benchmarks
5. Collect MySQL internal metrics
6. Collect OS-level system metrics
7. Analyze results and compare runs

---

# Environment Preparation

## Install Required Tools

Example for Debian/Ubuntu systems:

```bash
sudo apt update

sudo apt install -y \
    sysbench \
    sysstat \
    iotop \
    linux-tools-common \
    linux-tools-$(uname -r) \
    percona-toolkit \
    mysql-client
```

---

# MySQL Benchmark Dataset Preparation

## Create Benchmark Database

```sql
CREATE DATABASE sbtest;
EXIT;
```

---

## Prepare Dataset Using Sysbench

The following command creates:

- 10 tables
- 100,000 rows per table

Adjust dataset size depending on the MySQL InnoDB buffer pool size.

```bash
sysbench oltp_read_write \
    --db-driver=mysql \
    --mysql-host=127.0.0.1 \
    --mysql-port=3307 \
    --mysql-user=root \
    --mysql-password="" \
    --tables=10 \
    --table-size=100000 \
    prepare
```

---

# Buffer Pool Warm-Up

Before running actual benchmarks, warm up the InnoDB buffer pool to minimize cold-cache effects.

## Capture Initial Buffer Pool Statistics

```bash
mysql -uroot -pYOURPW \
    -e "SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_%';" \
    > /tmp/before.txt
```

---

## Run Warm-Up Workload

Example: 5-minute read-only warm-up

```bash
sysbench oltp_read_only \
    --db-driver=mysql \
    --mysql-host=127.0.0.1 \
    --mysql-user=root \
    --mysql-password=YOURPW \
    --tables=10 \
    --table-size=100000 \
    --time=300 \
    --threads=16 \
    run
```

---

## Capture Final Buffer Pool Statistics

```bash
mysql -uroot -pYOURPW \
    -e "SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_%';" \
    > /tmp/after.txt
```

---

## Compare Results

```bash
cat /tmp/before.txt /tmp/after.txt
```

A high cache hit ratio (close to 100%) indicates that the working dataset is mostly cached in memory.

---

# Benchmark Execution

## OLTP Read/Write Benchmark

Example benchmark configuration:

- Duration: 5 minutes
- Threads: 32
- Mixed read/write workload

```bash
sysbench oltp_read_write \
    --db-driver=mysql \
    --mysql-host=127.0.0.1 \
    --mysql-port=3307 \
    --mysql-user=root \
    --mysql-password=YOURPW \
    --tables=10 \
    --table-size=100000 \
    --time=300 \
    --threads=32 \
    --report-interval=10 \
    run
```

---

# Concurrency Sweep

Run the benchmark multiple times using different thread counts:

```text
1, 4, 8, 16, 32, 64
```

Repeat each run multiple times to obtain stable averages and latency percentiles.

---

# Benchmark Metrics

Sysbench automatically reports:

- Transactions per second (TPS)
- Queries per second (QPS)
- Average latency
- 95th percentile latency
- 99th percentile latency

---

# InnoDB Buffer Pool Hit Ratio

## Collect Metrics

```bash
mysql -uroot -pYOURPW -e "
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';
"
```

---

## Hit Ratio Formula

```text
hit_ratio = 1 - (
    Innodb_buffer_pool_reads /
    Innodb_buffer_pool_read_requests
)
```

Use metric deltas between benchmark start and end for accurate measurements.

---

# MySQL Internal Metrics

## Transaction Metrics

```sql
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Com_commit';
SHOW GLOBAL STATUS LIKE 'Com_rollback';
```

---

## InnoDB Metrics

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_%buffer_pool_%';
```

```sql
SHOW ENGINE INNODB STATUS\G
```

---

# Performance Schema Metrics

Performance Schema can be used to analyze:

- Statement latency
- Wait events
- Table I/O statistics
- Query digest summaries

Useful tables:

```sql
performance_schema.events_statements_summary_by_digest
```

```sql
performance_schema.table_io_waits_summary_by_table
```

---

# OS-Level Monitoring

## Disk I/O Monitoring

### iostat

```bash
iostat -xm 1
```

### iotop

```bash
sudo iotop
```

---

## CPU and Memory Monitoring

### vmstat

```bash
vmstat 1
```

### pidstat

```bash
pidstat -dur 1
```

---

# Result Analysis

## Analyze:

- Throughput scalability
- Latency under concurrency
- CPU saturation
- Disk bottlenecks
- Cache efficiency
- Query hotspots

---

# Optional Advanced Analysis

## Query Digest Analysis

Using Percona Toolkit:

```bash
pt-query-digest mysql-slow.log
```

---

# Benchmarking Recommendations

- Disable unrelated background services
- Use consistent dataset sizes
- Repeat runs multiple times
- Record MySQL configuration for reproducibility
- Keep benchmark duration sufficiently long
- Monitor thermal throttling and system load

---

# Cleanup

To remove benchmark tables:

```bash
sysbench oltp_read_write \
    --db-driver=mysql \
    --mysql-host=127.0.0.1 \
    --mysql-port=3307 \
    --mysql-user=root \
    --mysql-password=YOURPW \
    --tables=10 \
    cleanup
```

---

# References

- Sysbench Documentation
- MySQL Performance Schema Documentation
- Percona Toolkit Documentation
- Linux Performance Monitoring Tools