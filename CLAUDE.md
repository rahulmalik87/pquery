# CLAUDE.md — AI Assistant Guide for pquery

## Project Overview

pquery (parallel query) is a GPLv2-licensed, multi-threaded MySQL stress testing tool written in C++11. It generates random or sequential SQL workloads to find bugs in MySQL, Percona Server, MariaDB, and WebScaleSQL. Version 3.0 adds dynamic SQL generation with DDL/DML operations, table encryption, tablespace management, and server variable manipulation during test runs.

## Repository Structure

```
pquery/
├── CMakeLists.txt                  # Top-level CMake build config
├── cmake/                          # CMake modules
│   ├── PQCPack.cmake               # CPack packaging (RPM, DEB, TGZ)
│   ├── PQFindMySQLFork.cmake       # MySQL fork detection and library finding
│   ├── PQFindRevision.cmake        # Git revision extraction
│   └── PQSetupCompiler.cmake       # Compiler flags, deps (OpenSSL, pthreads, zlib)
├── src/
│   ├── CMakeLists.txt              # Source build config
│   ├── pquery.cpp                  # Main entry point, INI config parsing
│   ├── pquery.hpp                  # Main header
│   ├── common.hpp                  # Global option system (Option class, opt_int/opt_bool/opt_string macros)
│   ├── help.cpp                    # CLI option definitions and help text
│   ├── node.cpp / node.hpp         # Node (MySQL server) connection and worker management
│   ├── thread.cpp                  # Worker thread execution logic
│   ├── random_test.cpp             # Core random SQL generation engine (~2500 lines)
│   ├── random_test.hpp             # Table/Column/Index data structures
│   └── third_party/
│       ├── inih++/                 # INI file parser (compiled library)
│       └── rapidjson/              # JSON serialization (header-only)
├── doc/CONTRIBUTORS
├── README.md
├── COPYING                         # GPLv2 license
├── clean-tree.sh                   # Git reset/clean (destructive)
└── format.sh                       # Code formatter (bcpp -i 2)
```

## Build System

**Tool:** CMake >= 2.6 with Make.

### Build Commands

```bash
# Configure (pick ONE MySQL fork flag)
cmake . -DPERCONASERVER=ON -DBASEDIR=/path/to/percona-server
# or: -DMYSQL=ON, -DMARIADB=ON, -DWEBSCALESQL=ON

# Build
make -j

# Install (optional)
sudo make install
```

### CMake Options

| Option | Default | Description |
|---|---|---|
| `PERCONASERVER` | OFF | Build for Percona Server / PXC |
| `MYSQL` | OFF | Build for Oracle MySQL |
| `MARIADB` | OFF | Build for MariaDB |
| `WEBSCALESQL` | OFF | Build for WebScaleSQL |
| `BASEDIR` | — | Path to MySQL installation for auto-detection |
| `MYSQL_INCLUDE_DIR` | — | Manual header path override |
| `MYSQL_LIBRARY` | — | Manual library path override |
| `STATIC_LIB` | ON | Link MySQL client library statically |
| `STRICT_FLAGS` | ON | Enable `-Wall -Werror -Wextra -pedantic-errors` |
| `STRICT_CPU` | OFF | CPU-specific optimization (`-march=native`) |
| `ASAN` | OFF | AddressSanitizer (GCC >= 4.8) |
| `CMAKE_BUILD_TYPE` | Release | Release / Debug / RelWithDebInfo / MinSizeRel |

### Output Binaries

Named `pquery3-{ps,ms,md,ws}` depending on the selected fork. Found in `src/` after building.

### System Dependencies

- C++ compiler >= 4.7 with C++11 support
- CMake >= 2.6
- MySQL/Percona/MariaDB development files
- OpenSSL and libcrypto
- pthreads
- zlib
- dl, rt (Linux)

## Code Conventions

### Formatting

- **Indentation:** 2 spaces (enforced via `format.sh` using `bcpp -i 2`)
- **Formatter:** Run `./format.sh` from repo root to reformat all `.cpp`, `.hpp`, `.c`, `.h` files

### Naming

- **Classes/Structs:** PascalCase (`Column`, `Table`, `Node`, `Thd1`)
- **Functions:** snake_case (`create_worker()`, `set_seed()`)
- **Member variables:** snake_case with trailing underscore (`type_`, `columns_`, `name_`)
- **Enums/Constants:** UPPER_CASE (`COLUMN_TYPES`, `TABLE_TYPES`, `MIN_SEED_SIZE`)

### Header Guards

Use `#ifndef __FILENAME_HPP__` pattern.

### General Patterns

- C++11 standard (`-std=gnu++11`)
- STL containers: `std::vector`, `std::string`, `std::map`
- Thread-safe counters: `std::atomic<unsigned long>`
- Threading: `std::thread` with `std::thread::join()`
- Random generation: `std::mt19937`, `std::uniform_distribution`
- JSON serialization via RapidJSON templates
- INI configuration parsing via inih++

## Architecture

### Key Components

1. **Option System** (`common.hpp`): Global `options` vector of `Option*`. Supports INT, STRING, BOOL types. Access via `opt_int()`, `opt_bool()`, `opt_string()` macros.

2. **Node** (`node.cpp/hpp`): Represents a MySQL server connection. Manages worker thread creation and lifecycle.

3. **Worker Threads** (`thread.cpp`): Each thread maintains an independent MySQL connection and executes queries (random or from SQL file).

4. **Random SQL Generator** (`random_test.cpp/hpp`): Core engine generating DDL (CREATE/ALTER/DROP/TRUNCATE) and DML (INSERT/UPDATE/DELETE/SELECT) with configurable probabilities for each operation type.

5. **Table Metadata** (`random_test.hpp`): In-memory schema tracking with JSON serialization. Tracks columns (INT, CHAR, VARCHAR, FLOAT, DOUBLE, BOOL, BLOB, GENERATED), indexes, partitions, and encryption state.

### Execution Flow

1. `pquery.cpp` — Parse CLI args or INI config file
2. Create `Node` objects for each configured server
3. Each node spawns worker threads
4. Workers execute random or file-based SQL against their MySQL connection
5. Results logged to per-thread log files

### Logging

- General log: `<logdir>/<node>_general.log`
- Per-thread SQL: `<logdir>/<node>_thread-<id>.sql`
- Client output: `<logdir>/<node>_thread-<id>.out`
- Metadata: JSON format

## Testing

There is no automated test suite. Testing is done via manual execution against a running MySQL server instance. The `test` database must exist before starting runs.

### Basic Test Run

```bash
./src/pquery3-ps --tables 30 --records 200 --threads 10 --seconds 200 \
  --socket /tmp/mysql.sock --logdir=$PWD/log -k
```

Key flags: `--test-connection` to verify connectivity, `-k` for dynamic load generation, `-A` to log all SQL statements.

## Compiler Warnings

`STRICT_FLAGS=ON` (default) enables `-Wall -Werror -Wextra -pedantic-errors -Wmissing-declarations`. All code must compile warning-free.

## CI/CD

No automated CI/CD pipeline exists. Builds and testing are manual.

## Important Notes

- `clean-tree.sh` runs `git reset` and `git clean` — it destroys local changes
- Only one MySQL fork flag can be active per build
- MariaDB builds use shared linking (no static client library provided)
- The binary name changed from `pquery2-*` to `pquery3-*` in v3.0
