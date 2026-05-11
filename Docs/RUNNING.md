# How to Build and Run LRU-C MySQL

## Quick summary

Build this repository the same way you build MySQL 5.6 from source. After installation, initialize a MySQL data directory, start `mysqld`, connect with the `mysql` client, and run an InnoDB workload. The LRU-C code runs automatically inside InnoDB; there is no separate switch or command in this tree.

## Prerequisites

Use a Linux environment for the least friction. MySQL 5.6.26 is old, so modern compilers and OpenSSL versions may need compatibility fixes. A VM or container with an older Ubuntu/CentOS generation is usually easier than building directly on a new macOS system.

Typical packages:

```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake bison perl libncurses5-dev
```

Optional but useful:

```bash
sudo apt-get install -y libaio-dev git
```

## Build from source

Run the build out of tree so generated files do not clutter the source directory:

```bash
cd /path/to/LRU-C
mkdir -p build
cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DWITH_INNOBASE_STORAGE_ENGINE=1 \
  -DWITH_PARTITION_STORAGE_ENGINE=1
make -j"$(nproc)"
```

For a debug build:

```bash
cmake .. -DWITH_DEBUG=1
make -j"$(nproc)"
```

## Install to a local prefix

Installing to a local directory keeps this patched server separate from any system MySQL installation:

```bash
cd /path/to/LRU-C/build
cmake .. -DCMAKE_INSTALL_PREFIX="$HOME/lru-c-mysql"
make -j"$(nproc)"
make install
```

After install, the important binaries are usually here:

```text
$HOME/lru-c-mysql/bin/mysqld
$HOME/lru-c-mysql/bin/mysql
$HOME/lru-c-mysql/scripts/mysql_install_db
```

## Initialize a data directory

MySQL 5.6 uses `mysql_install_db`; it does not use the newer `mysqld --initialize` flow.

```bash
export BASE="$HOME/lru-c-mysql"
export DATA="$HOME/lru-c-mysql-data"

mkdir -p "$DATA"
"$BASE/scripts/mysql_install_db" \
  --basedir="$BASE" \
  --datadir="$DATA" \
  --user="$(whoami)"
```

## Start the server

Use a private port and socket so this does not collide with another MySQL server:

```bash
"$BASE/bin/mysqld_safe" \
  --basedir="$BASE" \
  --datadir="$DATA" \
  --port=3307 \
  --socket="$DATA/mysql.sock" \
  --pid-file="$DATA/mysql.pid" \
  --innodb-buffer-pool-size=1G \
  --innodb-buffer-pool-instances=1 &
```

Connect:

```bash
"$BASE/bin/mysql" \
  --socket="$DATA/mysql.sock" \
  -uroot
```

Stop:

```bash
"$BASE/bin/mysqladmin" \
  --socket="$DATA/mysql.sock" \
  -uroot shutdown
```

## Run a workload

The original README points to TPC-C at MySQL as the intended benchmark workload. Any InnoDB workload that causes buffer pool misses and dirty-page pressure can exercise the LRU-C path.

A minimal smoke test:

```sql
CREATE DATABASE lruc_test;
USE lruc_test;

CREATE TABLE t (
  id INT PRIMARY KEY AUTO_INCREMENT,
  payload VARCHAR(200)
) ENGINE=InnoDB;

INSERT INTO t(payload) VALUES ('one'), ('two'), ('three');
SELECT * FROM t;
```

For meaningful LRU-C behavior, use a dataset larger than the InnoDB buffer pool and a write-heavy or mixed read/write workload. TPC-C is a good fit because it creates dirty pages, cache misses, and concurrent access.

## How to know it is running this code

This tree includes a visible debug print in `buf_LRU_get_free_block()`:

```text
free block from free list
```

You may see that in the server error output when a block is taken from the normal free list. Most LRU-C debug prints are commented out, so absence of extra LRU-C messages does not mean the patch is inactive.

The practical confirmation is:

1. Build this source tree, not a system MySQL package.
2. Start the installed `mysqld` from your chosen prefix.
3. Use InnoDB tables.
4. Run a workload large enough to put pressure on the buffer pool.

## Common problems

If CMake fails on a modern system, try an older Linux image or install compatibility packages. This codebase is from 2015 and assumes the MySQL 5.6 era toolchain.

If `mysql_install_db` fails, check that you ran `make install` and that `--basedir` points to the install prefix, not the source tree.

If the client cannot connect, make sure the client uses the same `--socket` and `--port` values that the server was started with.

If another MySQL server is already running, keep this one isolated with a separate port, socket, and data directory.

