# CLAUDE.md — pquery Codebase Guide

This file provides guidance for AI assistants working in this repository.

## Project Overview

**pquery** is an open-source (GPLv2), multi-threaded MySQL stress testing tool written in C++11. It supports multiple MySQL flavors (Oracle MySQL, Percona Server, MariaDB, WebScaleSQL, Percona XtraDB Cluster) and has two main modes of operation:

- **Static mode**: Executes SQL from an input file sequentially or shuffled
- **Dynamic mode** (`-k` flag): Generates random DDL/DML operations programmatically at runtime

The current active version is **v3.0**, which produces binaries named `pquery3-{flavor}`.

## Repository Structure

```
pquery/
├── CMakeLists.txt              # Root CMake entry point
├── cmake/
│   ├── PQCPack.cmake           # CPack packaging (RPM/DEB/TGZ)
│   ├── PQFindMySQLFork.cmake   # MySQL flavor detection and library linking
│   ├── PQFindRevision.cmake    # Git revision embedding
│   └── PQSetupCompiler.cmake   # Compiler flags, arch detection, ASAN, etc.
├── src/
│   ├── CMakeLists.txt          # Source build rules, binary naming
│   ├── pquery.cpp              # Main entry point: option parsing, thread dispatch
│   ├── pquery.hpp              # Top-level declarations
│   ├── common.hpp              # Option struct, typedefs, global declarations
│   ├── help.cpp                # All CLI option definitions and help text
│   ├── node.cpp / node.hpp     # Node class: connection management, query execution
│   ├── thread.cpp              # Worker thread implementation
│   ├── random_test.cpp         # Core: dynamic SQL generation (~2,500 lines)
│   ├── random_test.hpp         # Table, Column, Index, Thd1 struct definitions
│   ├── pquery.cfg              # Example multi-node configuration file
│   ├── pquery.sql              # Example static SQL input file
│   └── grammer.sql             # SQL grammar templates
│   └── third_party/
│       ├── inih++/             # INI configuration parser (bundled)
│       └── rapidjson/          # JSON parser for metadata persistence (bundled)
├── doc/
│   └── CONTRIBUTORS
├── clean-tree.sh               # git reset --hard && git clean -xfd && git pull
├── format.sh                   # Code formatter using bcpp
└── README.md
```

## Building

### Prerequisites

- CMake >= 2.6
- C++ compiler with C++11 support (gcc >= 4.7)
- MySQL/Percona Server/MariaDB development headers and libraries
- OpenSSL, ZLIB development files

### CMake Flags

| Flag | Default | Description |
|------|---------|-------------|
| `-DPERCONASERVER=ON` | OFF | Build for Percona Server / PXC → `pquery3-ps` |
| `-DMYSQL=ON` | OFF | Build for Oracle MySQL → `pquery3-ms` |
| `-DMARIADB=ON` | OFF | Build for MariaDB → `pquery3-md` |
| `-DWEBSCALESQL=ON` | OFF | Build for WebScaleSQL → `pquery3-ws` |
| `-DPERCONACLUSTER=ON` | OFF | Build for Percona XtraDB Cluster → `pquery3-pxc` |
| `-DBASEDIR=<path>` | — | Path to MySQL binary tarball (auto-detects libs) |
| `-DSTATIC_LIB=ON` | ON | Statically link MySQL client library |
| `-DSTRICT_FLAGS=ON` | ON | Enable `-Wall -Werror -Wextra -pedantic-errors` |
| `-DSTRICT_CPU=ON` | OFF | Optimize for current CPU (`-march=native`) |
| `-DASAN=ON` | OFF | Enable AddressSanitizer |
| `-DCMAKE_BUILD_TYPE=Debug` | Release | Build type |

### Typical Build Commands

```bash
# Build for Percona Server using a tarball
cmake . -DPERCONASERVER=ON -DBASEDIR=/path/to/percona-server-basedir
make -j

# Build with debug info and ASAN
cmake . -DPERCONASERVER=ON -DBASEDIR=/path/to/basedir -DCMAKE_BUILD_TYPE=Debug -DASAN=ON
make -j

# Clean build environment (WARNING: discards local changes)
./clean-tree.sh
```

The resulting binary lands in `./src/pquery3-{flavor}`.

## Running pquery

### Dynamic load (v3.0 primary mode)

```bash
# Basic dynamic run
./src/pquery3-ps --tables 30 --logdir=/tmp/pq_log --records 200 \
  --threads 10 --seconds 200 --socket /tmp/mysql.sock -k

# DDL-only run
./src/pquery3-ps --tables 30 --logdir=/tmp/pq_log --records 200 \
  --threads 10 --seconds 200 --socket /tmp/mysql.sock -k \
  --no-insert --no-update --no-delete --no-select

# Temporary tables only
./src/pquery3-ps ... -k --only-temp-tables

# Dynamic server variable toggling
./src/pquery3-ps ... -k --mso=innodb_redo_log_encrypt=ON=OFF
```

### Static mode (SQL file replay)

```bash
./src/pquery3-ps --infile=pquery.sql --threads=4 --queries-per-thread=10000 \
  --socket=/tmp/mysql.sock --logdir=/tmp/log
```

### Configuration file mode (multi-node)

```bash
./src/pquery3-ps --config=pquery.cfg
```

## Key Source Files and Architecture

### `common.hpp`
Defines `Option` and `Server_Option` structs and the global `Opx *options` vector. Every CLI/config option is represented as an `Option*` in this vector, indexed by `Option::Opt` enum values. Convenience macros for option access:

```cpp
opt_int(TABLES)            // get integer option
opt_bool(NO_ENCRYPTION)    // get bool option
opt_string(ENGINE)         // get string option
opt_int_set(TABLES, 10)    // set integer option
```

### `help.cpp`
The single source of truth for all option definitions. Each option is created with `newOption()` and configured with `.help`, `.default_int`, `.default_bool`, etc. When adding a new option, add its enum to `Option::Opt` in `common.hpp` and register it in `help.cpp`.

### `random_test.hpp` / `random_test.cpp`
Core of the dynamic load generator. Key data structures:

- **`Column`** (and subclasses `Blob_Column`, `Generated_Column`): Represents a table column with type, constraints, and serialization
- **`Index`** / **`Ind_col`**: Index definitions with per-column desc support
- **`Table`** (and subclasses `Partition_table`, `Temporary_table`): Table abstraction with methods for all DDL/DML operations
- **`Thd1`**: Per-thread context holding the MySQL connection, log streams, and query counters

Global table list is shared across threads with per-table `std::mutex table_mutex` for concurrent DDL safety.

### `node.cpp` / `node.hpp`
The `Node` class manages the lifecycle of worker threads. It reads configuration (from INI file or CLI), establishes MySQL connections via `tryConnect()`, spawns `std::thread` workers, and writes summary logs.

### `thread.cpp`
Worker thread body. Calls `run_some_query(thd)` from `random_test.cpp` in a loop until the time limit (`--seconds`) or query limit (`--queries-per-thread`) is reached.

## Code Conventions

### Naming
- **Classes/Structs**: PascalCase (`Node`, `Column`, `Table`, `Thd1`)
- **Member variables**: `snake_case_` with trailing underscore for most data members (e.g., `name_`, `columns_`, `type_`)
- **Free functions**: `snake_case` (e.g., `execute_sql`, `load_metadata`, `save_metadata_to_file`)
- **Public methods on structs**: PascalCase (e.g., `Table::InsertRandomRow`, `Table::DropColumn`)
- **Macros**: `UPPER_SNAKE_CASE` (`PQVERSION`, `FORK`, `MAX_SEED_SIZE`)
- **Enum values**: `UPPER_SNAKE_CASE` (e.g., `Option::TABLES`, `Column::VARCHAR`)

### Header Guards
Use the pattern `__FILENAME_HPP__`:
```cpp
#ifndef __COMMON_HPP__
#define __COMMON_HPP__
// ...
#endif
```

### Thread Safety
- Use `std::atomic<unsigned long long>` for shared counters (`performed_queries_total`, `failed_queries_total`, `Option::total_queries`, `Option::success_queries`)
- Use `std::mutex` (`Table::table_mutex`, `Column::mutex`) for DDL operations that modify table structure
- Thread-local state is passed via `Thd1*` pointer

### Error Handling
- File operation failures: check return codes
- Configuration errors: throw `std::runtime_error`
- MySQL errors: log via `mysql_error()` and increment failure counters; do not abort
- System errors: use `strerror(errno)`

### Logging
- All SQL logs go to `{logdir}/{nodename}_general.log` (controlled by `-A`)
- Per-thread SQL logs: `{logdir}/{nodename}_thread-{id}.sql`
- DDL operations are always logged regardless of `-A`

### SQL Generation Pattern
Frequency-weighted random selection: each `Option` that represents an SQL operation has a probability integer (e.g., `--add-index=100`). `sum_of_all_options()` computes the total weight, then `pick_some_option()` selects one proportionally. Operations are implemented as methods on `Table` (e.g., `Table::AddIndex(thd)`).

## Adding New Features

### Adding a new SQL operation

1. Add enum value to `Option::Opt` in `common.hpp`
2. Register option in `help.cpp` via `newOption(...)` with `.setSQL()` and optionally `.setDDL()`
3. Add the operation method to `Table` in `random_test.hpp`
4. Implement the method in `random_test.cpp`
5. Wire it up in `run_some_query()` in `random_test.cpp` (the main dispatch switch/if chain)

### Adding a new column type

1. Add value to `Column::COLUMN_TYPES` enum in `random_test.hpp`
2. Update `col_type_to_string()` and `col_type()` in `random_test.cpp`
3. Add random value generation in `rand_value()` or a subclass
4. Update `Serialize()` template if metadata persistence is needed

## Metadata / State Persistence

pquery v3.0 uses RapidJSON to serialize table schema metadata to disk (via `save_metadata_to_file()`) and reload it (`load_metadata()`). This allows resuming a run without recreating schema from scratch. The metadata path is controlled by `--metadata-path`.

## Known Build Issues

- **WebScaleSQL**: May fail with `my_stacktrace.h: No such file or directory`. Fix: copy `my_stacktrace.h` from the WebScaleSQL source `include/` dir to `$BASEDIR/include/`.
- **Exit code 4 / invalid opcode**: Binary was compiled with `STRICT_CPU=ON` for a different CPU. Recompile locally without that flag.
- **MariaDB**: No static client library available in standard packages; `STATIC_LIB` is automatically disabled.

## Development Workflow

```bash
# Format all source files (requires bcpp)
./format.sh

# Clean working tree (WARNING: destroys local changes)
./clean-tree.sh

# Create a package
cpack -G RPM   # or DEB, TGZ, STGZ
```

There is no automated test suite. Testing is done by running pquery against a live MySQL instance and checking for crashes, assertion failures, or unexpected behavior in the MySQL error log.

## Dependencies (Bundled)

- **`src/third_party/inih++/`**: INI file parser (C++ wrapper around inih)
- **`src/third_party/rapidjson/`**: JSON parsing/serialization (header-only)

Do not upgrade or modify these vendored libraries without careful testing.
