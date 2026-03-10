# APSW Design and Implementation Document

APSW (Another Python SQLite Wrapper) provides an interface between Python and the SQLite database engine. This document elaborates on its design and implementation across five key areas: code layout, modules and their interfaces, the build system, integration with the Python interpreter, and performance optimizations.

## 1. Code Layout

The APSW codebase is organized in a pragmatic way to support both Python wrapper functionality and direct interactions with SQLite's C APIs. The repository structure is roughly as follows:

*   **`src/` directory**: The core C extension implementation resides here.
    *   `apsw.c`: The main entry point for the C extension. It includes initialization routines, basic definitions, and imports several other `.c` files directly using `#include` directives. This architectural choice enables access to static functions without exporting them in a header.
    *   `connection.c`: Defines the `apsw.Connection` class and its methods, which wrap `sqlite3*` database connections.
    *   `cursor.c`: Defines the `apsw.Cursor` class, which maps to SQLite prepared statements (`sqlite3_stmt*`).
    *   `blob.c`, `backup.c`, `vfs.c`, `vtable.c`: Implementation for specialized SQLite features like Blob I/O, Online Backup, Virtual File Systems, and Virtual Tables.
    *   `statementcache.c`: A dedicated implementation for caching SQLite prepared statements to boost performance.
    *   `jsonb.c`: Provides direct JSONB binary processing functionalities to bypass intermediate text encoding steps.
    *   `fts.c`: Implementations specific to Full-Text Search 5 (FTS5) tokenizers and auxiliary functions.
*   **`apsw/` directory**: Contains the Python-level wrapper components.
    *   `__init__.pyi`, `py.typed`: Type stubs and annotations for type checkers like mypy.
    *   `ext.py`: Additional Python helpers and utilities that build on top of the base C extension (e.g., `DataClassRowFactory`, `query_limit`, `format_query_table`).
    *   `sqlite_extra.json`, `sqlite_extra.py`: Metadata and extraction utilities for extended SQLite info.
*   **`tools/` directory**: Scripts for documentation generation, codebase maintenance, and tests.
*   **`setup.py` & `Makefile`**: Configuration files used by the build system.

This layout clearly separates the low-level, high-performance C integration (`src/`) from the higher-level Python interface and utilities (`apsw/`).

## 2. Modules and their Interfaces

APSW is designed to be a thin layer over the SQLite C API, exposing its capabilities fully to Python without adding unnecessary abstraction layers.

### Core C Types
*   **`apsw.Connection` (`connection.c`)**: Wraps `sqlite3`. It provides methods like `cursor()`, `backup()`, `blob_open()`, and registration interfaces for functions (`create_scalar_function`, `create_aggregate_function`, `create_window_function`), collations (`create_collation`), and hooks (`set_update_hook`, `set_commit_hook`).
*   **`apsw.Cursor` (`cursor.c`)**: Wraps `sqlite3_stmt`. It handles query execution (`execute`, `executemany`) and result fetching (`fetchone`, `fetchall`, `__next__`). The cursor interacts with the statement cache and binds parameters seamlessly using internal callbacks.
*   **`apsw.Blob` / `apsw.zeroblob` (`blob.c`)**: Exposes `sqlite3_blob` APIs for incremental reading and writing of large binary objects.
*   **`apsw.Backup` (`backup.c`)**: Exposes the `sqlite3_backup` APIs to allow copying databases while preserving ACID properties.
*   **Virtual Tables (`vtable.c`)**: Provides a bridging system (`VTModule`, `VTTable`, `VTCursor`) where SQLite calls C functions that APSW redirects to user-provided Python classes.
*   **FTS5 (`fts.c`)**: Connects to the SQLite FTS5 extension API, allowing Python code to define custom tokenizers and auxiliary functions.
*   **JSONB (`jsonb.c`)**: Offers `jsonb_encode`, `jsonb_decode`, and `jsonb_detect` at the C level, translating Python objects straight into SQLite's JSONB binary format.

### Python Layer (`apsw/ext.py`)
APSW also includes pure Python modules that provide enhanced features on top of the extension:
*   `DataClassRowFactory`: Modifies cursor row generation to yield `dataclasses` rather than tuples.
*   `Trace` / `ShowResourceUsage`: Context managers that intercept `sqlite3_trace_v2` output to generate rich query execution profiles and logs.
*   `query_info`: Examines a query using `EXPLAIN` and the SQLite authorizer to provide a plan and insight into table access, avoiding actual execution.
*   `make_virtual_module`: Simplifies the creation of Virtual Tables by turning a Python function generator directly into a SQLite read-only virtual table.

## 3. Build System

The APSW build system is handled by `setup.py` in combination with standard `setuptools`, and is highly configurable.

### `setup.py` and Distutils
*   **Amalgamation**: APSW is primarily designed to compile against the "SQLite amalgamation" (a single massive `sqlite3.c` file containing all SQLite code). `setup.py` looks for this file and, if found, sets the macro `APSW_USE_SQLITE_AMALGAMATION`.
*   **Automatic Fetching**: The `setup.py` script has a custom `fetch` command that automatically downloads the required SQLite source code (and relevant extensions) as a zip/tarball, verifies checksums, and places them in a `sqlite3/` directory.
*   **Extension Definition**: `apsw.__init__` is defined as a single C extension. Because `apsw.c` `#include`s the other `.c` files, `setup.py` only needs to list `src/apsw.c` in the `sources` list, simplifying the build process.
*   **Feature Toggles**: `setup.py` provides flags like `--enable-all-extensions`, `--enable=...`, and `--omit=...` which translate to `#define SQLITE_ENABLE_...` or `#define SQLITE_OMIT_...` during the compiler invocation.
*   **carray patching**: `setup.py` includes logic to patch the SQLite amalgamation (`patch_amalgamation()`) to support specific features like the `carray` extension in certain environments.

### `Makefile`
While `setup.py` handles the Python build, the `Makefile` is used extensively for developer workflows:
*   Running tests (`test`, `fulltest`, `test_debug`).
*   Building documentation using Sphinx (`docs`).
*   Running Python and C coverage tools.
*   Managing different Python versions on Windows (`compile-win`).
*   Downloading SQLite using fossil (`fossil`).

## 4. Integration with the Python Interpreter

APSW behaves as a classic Python C extension, leveraging the Python C API to glue Python and SQLite together tightly.

### Memory and Object Management
*   APSW wraps SQLite C structures (`sqlite3*`, `sqlite3_stmt*`) inside Python objects (`Connection`, `Cursor`). It manages object lifetimes, ensuring SQLite statements and connections are `sqlite3_finalize`d or `sqlite3_close`d when the Python garbage collector reclaims the wrapping object (`tp_dealloc`).
*   `PyWeakref` integration is utilized. Connections keep a list of weak references to their dependent objects (Cursors, Blobs). If a Connection is forcibly closed, it attempts to clean up its dependents gracefully.

### Exception Translation
*   When a SQLite C API returns a non-`SQLITE_OK` code, APSW's `SET_EXC` macro translates the SQLite error code into a specific APSW Python exception (e.g., `apsw.ConstraintError`, `apsw.BusyError`). It extracts the error message directly from the `sqlite3*` handle via `sqlite3_errmsg`.
*   Conversely, if a Python callback (like a custom scalar function or a virtual table method) raises a Python exception, APSW intercepts it using `PyErr_Occurred()`, captures the traceback, and reports a generic error code back to SQLite (like `SQLITE_ERROR`). APSW will later restore and raise the Python exception so it propagates correctly up the stack.

### Python Callbacks from C
APSW makes heavy use of `PyObject_Vectorcall` (for performance) when SQLite needs to invoke user-defined Python functions (e.g., UDFs, collations, authorizers, virtual tables).
*   **Virtual Tables**: The `apswvtab` struct intercepts SQLite callbacks (`xBestIndex`, `xFilter`, `xNext`, `xColumn`) and routes them to the user's Python methods using `PyObject_VectorcallMethod`.
*   **Functions**: `cbdispatch_func`, `cbdispatch_step`, and `cbdispatch_final` wrap Python callables provided for SQLite scalar and aggregate functions, performing necessary type conversions on parameters (`convert_value_to_pyobject`) and results (`set_context_result`).

### Async Support
APSW supports asynchronous execution (`as_async`, `async_run`). Since SQLite's native C API is strictly synchronous and blocking, APSW handles async via an `AsyncConnectionController` which spawns a background worker thread. Cursors yield iterators that manage `AIter_On`, `AIter_Exception`, and `AIter_End` states to yield control back to the Python async event loop when fetching rows.

### GIL Management
APSW correctly manages the Global Interpreter Lock (GIL):
*   `Py_BEGIN_ALLOW_THREADS` and `Py_END_ALLOW_THREADS` are used around blocking SQLite calls like `sqlite3_step`, `sqlite3_prepare_v3`, and `sqlite3_open_v2` to allow other Python threads to execute concurrently.
*   `PyGILState_Ensure()` and `PyGILState_Release()` are used whenever SQLite invokes an APSW callback (like a trace hook, authorizer, or UDF), ensuring it is safe to interact with Python objects.

## 5. Performance Optimizations

APSW prioritizes minimal overhead and maximum performance, implementing several architectural optimizations.

### Statement Cache
Preparing a SQL statement (`sqlite3_prepare_v3`) is a CPU-intensive operation. APSW mitigates this by maintaining a built-in statement cache (`src/statementcache.c`).
*   Instead of a Python dictionary, the cache uses a custom C-level hash table implementation based on the DJB2 algorithm.
*   When a user calls `Cursor.execute()`, APSW hashes the query string and looks up a cached `APSWStatement` containing the compiled `sqlite3_stmt*`.
*   If found, the statement bindings are cleared (`sqlite3_clear_bindings`), the statement is reset (`sqlite3_reset`), and it is immediately reused.
*   This entirely avoids parsing and compiling overhead for repeated queries, while keeping the implementation invisible to the user.

### Type Conversion Fast Paths
Data is constantly converted between Python and SQLite types.
*   `convert_column_to_pyobject` utilizes `sqlite3_column_type` to immediately branch into the correct `PyLong_FromLongLong`, `PyFloat_FromDouble`, or `PyUnicode_FromStringAndSize` calls without generic overhead.
*   `set_context_result` does the reverse. It uses direct type checks (`PyLong_Check`, `PyUnicode_Check`) instead of duck typing, mapping standard Python primitives immediately to `sqlite3_result_int64`, `sqlite3_result_text64`, etc.

### Vectorcall
Wherever possible, APSW uses the `FASTCALL` and `Vectorcall` conventions (`PyObject_Vectorcall`, `PyObject_VectorcallMethod`) introduced in modern CPython versions. This avoids creating intermediate Python tuple objects when passing arguments to callbacks or functions, reducing memory allocations and garbage collection pressure.

### Direct JSONB Conversion
The `src/jsonb.c` file implements direct serialization and deserialization between Python objects (dicts, lists, int, strings) and SQLite's internal JSONB binary format.
*   Without this, a user would convert a dict to a JSON string using Python's `json` module, pass it to SQLite, and SQLite would parse that string into JSONB.
*   APSW's implementation skips the string generation entirely, writing binary JSONB tags directly. As noted in the APSW docs, this reduces the total encoding time by a factor of ~10x compared to the string intermediate method.

### Minimal Object Allocation
In areas like execution traces and custom hooks, APSW tries to reuse objects or return `None` rather than building extensive object hierarchies. Cursors do not maintain large lists of data unless explicitly using `fetchall()`; `fetchone()` and `__next__` fetch data incrementally (`sqlite3_step`) and build the Python tuple strictly on demand.