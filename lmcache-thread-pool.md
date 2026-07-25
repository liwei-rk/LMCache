# remotebackend与storage_backend关系

**Session ID:** ses_0675a5f13ffekQEunl4dpgdQbV
**Created:** 7/25/2026, 9:40:19 AM
**Updated:** 7/25/2026, 11:19:45 AM

---

## User

remotebackend和storage_backend有什么关系

---

## Assistant (Build · GLM-5.1 · 181.2s)

_Thinking:_

The user is asking about the relationship between `remotebackend` and `storage_backend` in the LMCache codebase. Let me explore the codebase to understand these concepts.

**Tool: task**

**Input:**
```json
{
  "description": "Explore remotebackend and storage_backend relationship",
  "prompt": "In the LMCache codebase at /home/models/src/LMCache, I need to understand the relationship between \"remotebackend\" and \"storage_backend\". \n\nPlease do the following:\n1. Search for files and classes related to \"remotebackend\" or \"RemoteBackend\" \n2. Search for files and classes related to \"storage_backend\" or \"StorageBackend\"\n3. Read key files that define these concepts, especially in lmcache/v1/ directory\n4. Look at how they relate to each other - inheritance, composition, delegation, etc.\n5. Check the cache_engine.py to see how they are used together\n\nReturn a concise summary of:\n- What is RemoteBackend and what does it do?\n- What is StorageBackend and what does it do?\n- How are they related (inheritance, composition, etc.)?\n- How are they used together in the cache engine?",
  "subagent_type": "explore"
}
```

**Output:**
```
<task id="ses_0675a3191ffeGVCuZBX6Pg8Myv" state="completed">
<task_result>
Now I have all the information needed. Here is the concise summary:

---

## Summary: RemoteBackend vs. StorageBackend Relationship

### What is StorageBackend (StorageBackendInterface)?

**Definition:** `StorageBackendInterface` (defined in `/home/models/src/LMCache/lmcache/v1/storage_backend/abstract_backend.py`) is the **abstract base class** that defines the contract for all storage backends in the v1 engine. It is not a concrete class -- it is an interface/ABC.

**What it does:** It declares the universal API every storage backend must implement:
- `contains()` / `batched_contains()` -- check if a key exists
- `batched_submit_put_task()` / `async_batched_submit_put_task()` -- store KV cache data
- `get_blocking()` / `batched_get_blocking()` / `get_non_blocking()` -- retrieve KV cache data
- `pin()` / `unpin()` -- pin/unpin to prevent eviction
- `remove()` -- delete a cached entry
- `get_allocator_backend()` -- return the backend responsible for memory allocation
- `close()` -- teardown

Two specialized sub-interfaces extend it:
- **`AllocatorBackendInterface`** -- adds `allocate()`, `batched_allocate()`, `initialize_allocator()`, `get_memory_allocator()` for backends that own memory pools (e.g., `LocalCPUBackend`, `PDBackend`)
- **`StoragePluginInterface`** -- adds config/metadata/loop/local_cpu_backend fields for dynamically-loaded plugin backends

**Concrete implementations include:** `LocalCPUBackend`, `LocalDiskBackend`, `RemoteBackend`, `P2PBackend`, `GdsBackend`, `PDBackend`, `AuditBackend`, `MaruBackend`, `NixlStorageBackend`, and any user-supplied plugins.

### What is RemoteBackend?

**Definition:** `RemoteBackend` (defined in `/home/models/src/LMCache/lmcache/v1/storage_backend/remote_backend.py`, line 27) is a **concrete class** that **inherits from `StorageBackendInterface`**.

**What it does:** It manages KV cache storage on a **remote service** (Redis, LM server, Mooncake, filesystem server, etc.). Its key responsibilities:
- **Serialization/deserialization**: Compresses data before sending to remote; decompresses on retrieval (via `CreateSerde`)
- **Network I/O delegation**: Uses a `RemoteConnector` (created via `CreateConnector`) for actual network operations (put, get, exists, remove)
- **Async operations**: All remote I/O is async, dispatched on a dedicated event loop via `asyncio.run_coroutine_threadsafe()`
- **Connection management**: Handles connection initialization, reconnection with cooldown, and graceful failure (returns None/False on errors instead of crashing)
- **MLA worker_id normalization**: In MLA multi-worker mode, remaps keys so worker_id=0 is used for remote storage (so all workers share the same remote cache)
- **Memory allocation delegation**: `get_allocator_backend()` returns its held `LocalCPUBackend` reference, because `RemoteBackend` itself does not manage memory -- it relies on the local CPU allocator for deserialized objects

### How are they related?

The relationship is **inheritance**: `RemoteBackend` **extends** `StorageBackendInterface`. There is no separate "StorageBackend" concrete class -- `StorageBackendInterface` is the abstract parent, and `RemoteBackend` is one of several concrete children.

```
StorageBackendInterface (ABC)           [abstract_backend.py]
    |
    |-- AllocatorBackendInterface       [adds allocate/batched_allocate]
    |       |-- LocalCPUBackend
    |       |-- PDBackend / PDBackendAsync
    |
    |-- StoragePluginInterface          [adds config/metadata/loop fields]
    |
    |-- RemoteBackend                   [remote_backend.py]
    |-- LocalDiskBackend                [local_disk_backend.py]
    |-- P2PBackend                      [p2p_backend.py]
    |-- GdsBackend                      [gds_backend.py]
    |-- AuditBackend (decorator)        [audit_backend.py - wraps any StorageBackendInterface]
    |-- MaruBackend                     [maru_backend.py]
    |-- NixlStorageBackend              [nixl_storage_backend.py]
```

Additionally, `RemoteBackend` has a **composition** relationship with `LocalCPUBackend`: it holds a `local_cpu_backend` reference for memory allocation during `get_blocking()` / `batched_get_blocking()`. This is because `RemoteBackend` does not inherit from `AllocatorBackendInterface` -- it cannot allocate memory itself, so it delegates that to the local CPU backend.

### How are they used together in the cache engine?

The orchestration chain is:

```
LMCacheEngine (cache_engine.py)
    |
    |-- StorageManager (storage_manager.py)
            |
            |-- storage_backends: OrderedDict[str, StorageBackendInterface]
                    |-- "LocalCPUBackend"   -> LocalCPUBackend instance
                    |-- "LocalDiskBackend"   -> LocalDiskBackend instance
                    |-- "RemoteBackend"      -> RemoteBackend instance
                    |-- "P2PBackend"         -> P2PBackend instance
                    |-- ... (others as configured)
```

**Key coordination patterns in StorageManager:**

1. **Tiered prefix-based lookup**: `StorageManager.contains()` and `batched_contains()` iterate through backends in order. If a prefix of keys is found in one backend, the remaining suffix keys are searched in the next backend. This means `RemoteBackend` acts as a "fallback tier" -- local CPU is checked first, then local disk, then remote.

2. **Write-back on retrieval**: When `StorageManager.get()` or `batched_get()` retrieves data from `RemoteBackend` (or any non-local backend), it automatically writes the result back into `LocalCPUBackend` via `submit_put_task()` or `batched_submit_put_task()`. This creates a local hot cache of recently-used remote data.

3. **Multi-backend put**: `StorageManager.batched_put()` stores data in *all* configured backends simultaneously. It allocates memory in each backend's allocator (copying data as needed), then calls `batched_submit_put_task()` on each.

4. **Creation**: `CreateStorageBackends()` (in `__init__.py`) instantiates all backends from config. `RemoteBackend` is created when `config.remote_url` is set (legacy) or `config.remote_storage_plugins` is set (new plugin approach). It requires `LocalCPUBackend` to already exist.

5. **Health monitoring**: `RemoteBackendHealthCheck` (in `health_monitor/checks/`) monitors `RemoteBackend` health. On failure, it can trigger "bypass mode" in `StorageManager`, which skips the `RemoteBackend` during contains/put/get operations.

6. **Audit wrapping**: If `audit_backend_enabled` is configured, `CreateStorageBackends()` wraps every non-`LocalCPUBackend` with `AuditBackend`, which is a **decorator pattern** that logs all operations. So `RemoteBackend` might actually be wrapped as `AuditBackend(real_backend=RemoteBackend(...))`.

In short: **`StorageBackendInterface` is the abstract contract; `RemoteBackend` is a concrete implementation of that contract for remote/network storage; and `StorageManager` orchestrates multiple such implementations (including `RemoteBackend`) as a tiered caching hierarchy.**
</task_result>
</task>
```

---

## Assistant (Build · GLM-5.1 · 14.4s)

_Thinking:_

The task agent has provided a very comprehensive answer. Let me summarize the key points concisely for the user.

**`StorageBackendInterface`**（在 `lmcache/v1/storage_backend/abstract_backend.py`）是所有存储后端的**抽象基类/接口**，定义了 `contains`、`get`、`put`、`pin` 等通用 API。

**`RemoteBackend`**（在 `lmcache/v1/storage_backend/remote_backend.py:27`）是 `StorageBackendInterface` 的一个**具体子类实现**，专门负责远程存储（Redis、LM server、Mooncake 等）。它通过 `RemoteConnector` 委托网络 I/O，通过 `LocalCPUBackend` 委托内存分配。

继承关系：

```
StorageBackendInterface (抽象基类)
  ├── AllocatorBackendInterface → LocalCPUBackend, PDBackend ...
  ├── RemoteBackend           ← 远程存储实现
  ├── LocalDiskBackend        ← 本地磁盘实现
  ├── P2PBackend, GdsBackend, MaruBackend, NixlStorageBackend ...
  └── AuditBackend (装饰器，可包装任意 StorageBackendInterface)
```

在 `StorageManager` 中，多个后端（包括 `RemoteBackend`）以 `OrderedDict` 组织成**分层缓存**：查找时依次遍历 local CPU → local disk → remote；从 remote 取到数据后会写回 local CPU（热点缓存）。

---

## User

IO线程池是怎么分配的？给出相关示例

---

## Assistant (Build · GLM-5.1 · 228.1s)

**Tool: task**

**Input:**
```json
{
  "description": "Explore IO thread pool allocation",
  "prompt": "In the LMCache codebase at /home/models/src/LMCache, I need to understand how IO thread pools are allocated and used. \n\nPlease search for:\n1. Thread pool related code - search for \"ThreadPoolExecutor\", \"thread_pool\", \"io_thread\", \"event_loop\", \"asyncio\" in the lmcache/v1/ directory\n2. How thread pools are created, configured, and assigned to different backends\n3. The relationship between thread pools and storage backends (especially RemoteBackend, LocalDiskBackend)\n4. Any configuration parameters that control thread pool sizes\n5. Read the relevant source files thoroughly\n\nReturn:\n- A clear explanation of how IO thread pools are allocated\n- Which components use thread pools and how\n- Configuration parameters for thread pool sizing\n- Concrete code examples showing thread pool creation and usage (with file paths and line numbers)",
  "subagent_type": "explore"
}
```

**Output:**
```
<task id="ses_0674eef13ffewBLU7TXX196xSU" state="completed">
<task_result>
Now I have a thorough understanding of the entire thread pool architecture. Here is my comprehensive analysis:

---

## How IO Thread Pools Are Allocated and Used in LMCache

### 1. The Central Event Loop (StorageManager)

The **StorageManager** at `/home/models/src/LMCache/lmcache/v1/storage_backend/storage_manager.py` (lines 233-240) creates a single central `asyncio` event loop that runs on a dedicated daemon thread. This loop is the primary dispatch mechanism for most storage backends:

```python
# Lines 233-240
self.loop = asyncio.new_event_loop()
self.thread = threading.Thread(
    target=start_loop_in_thread_with_exceptions,
    args=(self.loop,),
    name="storage-manager-event-loop",
)
self.thread.start()
```

When `CreateStorageBackends()` (lines 1275-1286) creates backends, it passes `self.loop` to each one. Most backends submit async work onto this shared loop using `asyncio.run_coroutine_threadsafe()`. This is the key architectural pattern: **one shared event loop, many backends dispatching onto it**.

---

### 2. Thread Pool Types and Their Components

There are **6 distinct thread pool/executor patterns** in the codebase:

| Pattern | Class | Where Used | Default Workers | Config Key |
|---------|-------|------------|----------------|------------|
| Priority Queue + ThreadPool | `AsyncPQThreadPoolExecutor` | LocalDiskBackend | 4 | (hardcoded TODO) |
| Priority Queue + asyncio | `AsyncPQExecutor` | ValkeyConnector, S3Connector | 4 | (hardcoded) |
| Plain `ThreadPoolExecutor` | `concurrent.futures.ThreadPoolExecutor` | GdsBackend, HF3fsConnector, MQ Server, L2 Adapters, DaxBackend, AsyncSerdeProcessor | varies | varies |
| AWS CRT EventLoopGroup | `awscrt.io.EventLoopGroup` | S3Connector, S3L2Adapter | 64 | `s3_num_io_threads` |
| AffinityThreadPool | Custom per-identity hashing | MQ Server (GPU handlers) | 1 | `max_gpu_workers` |
| Dedicated asyncio loop | `asyncio.new_event_loop()` | PDBackendAsync, S3L2Adapter, FSL2Adapter, NixlStoreL2Adapter | N/A (coroutine-based) | N/A |

---

### 3. Component-by-Component Thread Pool Analysis

#### 3.1 LocalDiskBackend

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/local_disk_backend.py`

The `LocalDiskWorker` (lines 39-96) creates an `AsyncPQThreadPoolExecutor` with **4 max_workers** (hardcoded, with a TODO comment to make it configurable):

```python
# Line 48
self.executor = AsyncPQThreadPoolExecutor(loop, max_workers=4)
```

The `AsyncPQThreadPoolExecutor` (defined in `/home/models/src/LMCache/lmcache/v1/storage_backend/job_executor/pq_executor.py`, lines 133-235) is a priority-queue-based executor that runs **sync** functions via `asyncio.to_thread()`. Tasks have priority ordering:
- `prefetch`: priority 0 (highest)
- `delete`: priority 1
- `put`: priority 2 (lowest)

The `loop` parameter is the shared StorageManager event loop. Tasks are submitted via `self.disk_worker.submit_task()` (lines 52-74), which wraps the actual sync I/O function.

#### 3.2 GdsBackend (GPU Direct Storage)

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/gds_backend.py`

Creates its own `ThreadPoolExecutor` specifically for **parallel batched reads** (lines 264-274):

```python
# Lines 264-274
self.use_thread_pool = self.use_gds
if self.use_thread_pool:
    thread_count = _DEFAULT_THREAD_COUNT  # = 4
    if config.extra_config is not None:
        thread_count = config.extra_config.get(
            "gds_io_threads", _DEFAULT_THREAD_COUNT
        )
    self._thread_pool = ThreadPoolExecutor(
        max_workers=thread_count, thread_name_prefix="gds-io"
    )
```

**Config parameter:** `gds_io_threads` in `extra_config` (default: 4)

The thread pool is used for `_batched_get_blocking_by_thread_pool_impl()` (lines 880-929), which calls `self._thread_pool.map()` to parallelize GDS reads. For writes, it uses `asyncio.to_thread()` on the shared StorageManager loop instead.

Shutdown: line 1225-1226 calls `self._thread_pool.shutdown(wait=True)`.

#### 3.3 RemoteBackend

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/remote_backend.py`

**RemoteBackend itself does NOT own any thread pool.** It delegates all I/O to its `RemoteConnector` and submits coroutines onto the shared StorageManager event loop via `asyncio.run_coroutine_threadsafe()`:

```python
# Line 268-269 (put)
future = asyncio.run_coroutine_threadsafe(
    self.connection.put(key, compressed_memory_obj), self.loop
)

# Line 363 (get_blocking)
future = asyncio.run_coroutine_threadsafe(self.connection.get(key), self.loop)
```

The thread pool work happens inside the **connector** (see below).

#### 3.4 Connectors (I/O Layer beneath RemoteBackend)

##### ValkeyConnector

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/valkey_connector.py`

Two-layer threading architecture:
1. **`_ThreadWorkerPool`** (lines 70-258): A `ThreadPoolExecutor(max_workers=num_workers)` where each thread has its own GLIDE sync client via `threading.local()`:

```python
# Lines 120-122
self._executor = ThreadPoolExecutor(
    max_workers=num_workers,
    thread_name_prefix="valkey",
)
```

**Config parameter:** `valkey_num_workers` or `valkey_sync_num_workers` in `extra_config` (default: 8). See `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/valkey_adapter.py` lines 54-58.

2. **`AsyncPQExecutor`** (line 326): Priority scheduling on the shared StorageManager event loop with priorities: PEEK(1) > PREFETCH(2) > GET(3) > PUT(4).

```python
# Line 326
self._pq_executor = AsyncPQExecutor(loop)
```

Operations flow: `public_method -> pq_executor.submit_job -> _internal_method -> _pool.submit_* -> ThreadPoolExecutor` (actual I/O on worker thread).

##### S3Connector

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/s3_connector.py`

Uses AWS CRT's native **EventLoopGroup** (not a Python thread pool):

```python
# Lines 103-105
event_loop_group = io.EventLoopGroup(s3_num_io_threads)
host_resolver = io.DefaultHostResolver(event_loop_group)
client_bootstrap = io.ClientBootstrap(event_loop_group, host_resolver)
```

**Config parameter:** `s3_num_io_threads` in `extra_config` (default: 64). See `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/s3_adapter.py` line 32.

Also uses `AsyncPQExecutor(loop)` (line 167) for priority scheduling.

##### HF3fsConnector

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/hf3fs_connector.py`

Creates its own `ThreadPoolExecutor` for sync I/O dispatch:

```python
# Lines 110-112
self.io_executor = ThreadPoolExecutor(
    max_workers=io_thread_num, thread_name_prefix="HF3fsConnector_executor"
)
```

**Config parameter:** `hf3fs_io_thread_num` in `extra_config` (default: 4, valid range 2-16). See `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/hf3fs_adapter.py` line 76.

Operations use `self.loop.run_in_executor(self.io_executor, ...)` (line 144, 218-219):

```python
# Line 144
return await self.loop.run_in_executor(self.io_executor, self.sync_get, key)
```

#### 3.5 L2 Adapters (Distributed Storage Layer)

##### RawBlockL2Adapter

**File:** `/home/models/src/LMCache/lmcache/v1/distributed/l2_adapters/raw_block_l2_adapter.py`

3 separate dedicated `ThreadPoolExecutor`s (lines 336-347):

```python
self._store_pool = ThreadPoolExecutor(
    max_workers=config.num_store_workers,
    thread_name_prefix="rawblk-store",
)
self._lookup_pool = ThreadPoolExecutor(
    max_workers=config.num_lookup_workers,
    thread_name_prefix="rawblk-lookup",
)
self._load_pool = ThreadPoolExecutor(
    max_workers=config.num_load_workers,
    thread_name_prefix="rawblk-load",
)
```

**Config parameters** (from `RawBlockL2AdapterConfig`, lines 89-91):
- `num_store_workers`: default 2
- `num_lookup_workers`: default 1
- `num_load_workers`: default 4

##### DaxL2Adapter

**File:** `/home/models/src/LMCache/lmcache/v1/distributed/l2_adapters/dax_l2_adapter.py`

Same 3-pool pattern (lines 191-202):

```python
self._store_executor = ThreadPoolExecutor(
    max_workers=config.num_store_workers,
    thread_name_prefix="dax-l2-store",
)
self._lookup_executor = ThreadPoolExecutor(
    max_workers=config.num_lookup_workers,
    thread_name_prefix="dax-l2-lookup",
)
self._load_executor = ThreadPoolExecutor(
    max_workers=config.num_load_workers,
    thread_name_prefix="dax-l2-load",
)
```

**Config parameters** (from `DaxL2AdapterConfig`, lines 55-57):
- `num_store_workers`: default 1
- `num_lookup_workers`: default 1
- `num_load_workers`: default `min(4, os.cpu_count() or 1)`

##### S3L2Adapter

**File:** `/home/models/src/LMCache/lmcache/v1/distributed/l2_adapters/s3_l2_adapter.py`

Uses AWS CRT EventLoopGroup (line 350) **plus** its own asyncio event loop on a daemon thread (lines 419-421):

```python
event_loop_group = io.EventLoopGroup(config.s3_num_io_threads)
self._loop = asyncio.new_event_loop()
self._loop_thread = threading.Thread(target=self._run_event_loop, daemon=True)
```

**Config parameter:** `s3_num_io_threads` (default: 64)

##### FSL2Adapter / NixlStoreL2Adapter / MockL2Adapter

Each creates its own dedicated asyncio event loop running on a daemon thread, using `asyncio.run_coroutine_threadsafe()` to dispatch operations. No Python `ThreadPoolExecutor`; async I/O happens natively or via CRT.

#### 3.6 DaxBackend (Plugin)

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/plugins/dax_backend.py`

Two `ThreadPoolExecutor`s (lines 210-217):

```python
self._restore_executor = ThreadPoolExecutor(
    max_workers=self._restore_workers,
    thread_name_prefix="dax-restore",
)
self._restore_dispatch_executor = ThreadPoolExecutor(
    max_workers=1,
    thread_name_prefix="dax-restore-dispatch",
)
```

**Config parameters:**
- `dax.restore_workers` in `extra_config` (default: `min(8, os.cpu_count() or 1)`)
- `dax.restore_max_regions` (default: same as restore_workers)

#### 3.7 AsyncSerdeProcessor

**File:** `/home/models/src/LMCache/lmcache/v1/distributed/serde/async_processor.py`

Single `ThreadPoolExecutor` for serialization/deserialization (line 70):

```python
self._pool = ThreadPoolExecutor(max_workers=max_workers)
```

**Config parameter:** `max_workers` in `SerdeConfig` kwargs (default: 1, since CPU-bound serialization typically releases the GIL only partially).

#### 3.8 MessageQueueServer (Multiprocess)

**File:** `/home/models/src/LMCache/lmcache/v1/multiprocess/mq.py`

Two pools are created (lines 639-714), assigned in `/home/models/src/LMCache/lmcache/v1/multiprocess/server.py` (lines 222-229):

```python
# server.py lines 222-229
if affinity_types:
    server.add_affinity_thread_pool(
        affinity_types, max_workers=mp_config.max_gpu_workers
    )
if normal_types:
    server.add_normal_thread_pool(
        normal_types, max_workers=mp_config.max_cpu_workers
    )
```

- **AffinityThreadPool** for GPU-bound handlers (STORE/RETRIEVE): routes requests from the same zmq identity to the same thread. Configurable via `max_gpu_workers`.
- **ThreadPoolExecutor** for CPU handlers (LOOKUP, END_SESSION): standard thread pool. Configurable via `max_cpu_workers`.

**Config parameters** (from `/home/models/src/LMCache/lmcache/v1/multiprocess/config.py`):
- `max_workers`: base number (default 1), CLI `--max-workers`
- `max_gpu_workers`: GPU pool size (default: inherits from `max_workers`), CLI `--max-gpu-workers`
- `max_cpu_workers`: CPU pool size (default: inherits from `max_workers`), CLI `--max-cpu-workers`

#### 3.9 PDBackendAsync (Prefill/Decode Disaggregation)

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/pd_backend_async.py`

Creates **two separate asyncio event loops** (not thread pools), one for sender and one for receiver (lines 350-366):

```python
if self.pd_config.role == "sender":
    self._sender_loop = asyncio.new_event_loop()
    self._sender_thread = threading.Thread(
        target=self._sender_loop.run_forever, daemon=True, name="pd-sender-async"
    )
elif self.pd_config.role == "receiver":
    self._recv_loop = asyncio.new_event_loop()
    self._recv_thread = threading.Thread(
        target=self._recv_loop.run_forever, daemon=True, name="pd-receiver-async"
    )
```

No `ThreadPoolExecutor`; all I/O is async via the transfer channel.

---

### 4. Complete Configuration Parameter Reference

| Parameter | Config Source | Default | Valid Range | Component |
|-----------|--------------|---------|-------------|-----------|
| `gds_io_threads` | `extra_config` dict | 4 | any positive int | GdsBackend thread pool |
| `s3_num_io_threads` | `extra_config` dict | 64 | any positive int | S3Connector CRT EventLoopGroup / S3L2Adapter CRT |
| `hf3fs_io_thread_num` | `extra_config` dict | 4 | [2, 16] | HF3fsConnector ThreadPoolExecutor |
| `valkey_num_workers` / `valkey_sync_num_workers` | `extra_config` dict | 8 | any positive int | ValkeyConnector `_ThreadWorkerPool` |
| `dax.restore_workers` | `extra_config` dict | min(8, cpu_count) | any positive int | DaxBackend (plugin) restore executor |
| `dax.restore_max_regions` | `extra_config` dict | same as restore_workers | any positive int | DaxBackend (plugin) dispatch |
| `max_workers` | CLI `--max-workers` | 1 | any positive int | MPServerConfig base pool size |
| `max_gpu_workers` | CLI `--max-gpu-workers` | inherits max_workers | any positive int | MQ Server affinity pool (STORE/RETRIEVE) |
| `max_cpu_workers` | CLI `--max-cpu-workers` | inherits max_workers | any positive int | MQ Server normal pool (LOOKUP, etc.) |
| `num_store_workers` | L2 adapter JSON | 2 (raw_block) / 1 (dax) | any positive int | RawBlockL2/DaxL2 adapter store pool |
| `num_lookup_workers` | L2 adapter JSON | 1 (both) | any positive int | RawBlockL2/DaxL2 adapter lookup pool |
| `num_load_workers` | L2 adapter JSON | 4 (raw_block) / min(4,cpu_count) (dax) | any positive int | RawBlockL2/DaxL2 adapter load pool |
| `max_workers` (serde) | `SerdeConfig` kwargs | 1 | any positive int | AsyncSerdeProcessor thread pool |
| `blocking_timeout_secs` | `LMCacheEngineConfig` | 10 | any positive int | Timeout for blocking remote operations |

**Note:** The LocalDiskBackend's `AsyncPQThreadPoolExecutor` with 4 workers is **hardcoded** (line 48) with a TODO comment to make it configurable.

---

### 5. Architecture Summary Diagram

```
StorageManager
  |
  |-- self.loop (asyncio event loop on daemon thread "storage-manager-event-loop")
  |     |
  |     |-- LocalDiskBackend.disk_worker.executor (AsyncPQThreadPoolExecutor, 4 workers)
  |     |     Uses asyncio.to_thread() for sync disk I/O
  |     |
  |     |-- RemoteBackend.connection (varies by connector)
  |     |     |
  |     |     |-- ValkeyConnector
  |     |     |     |-- _ThreadWorkerPool (ThreadPoolExecutor, num_workers threads)
  |     |     |     |-- AsyncPQExecutor (priority scheduling on shared loop)
  |     |     |
  |     |     |-- S3Connector
  |     |     |     |-- io.EventLoopGroup (CRT native, s3_num_io_threads)
  |     |     |     |-- AsyncPQExecutor (priority scheduling on shared loop)
  |     |     |
  |     |     |-- HF3fsConnector
  |     |     |     |-- io_executor (ThreadPoolExecutor, hf3fs_io_thread_num threads)
  |     |
  |     |-- GdsBackend
  |     |     |-- self._thread_pool (ThreadPoolExecutor, gds_io_threads)
  |     |     |-- asyncio.to_thread() for async writes on shared loop
  |
  |-- PDBackendAsync (separate loops)
  |     |-- _sender_loop / _recv_loop (own asyncio loops on daemon threads)
  |
  L2 Adapters (standalone, separate from StorageManager)
  |-- RawBlockL2Adapter: 3 ThreadPoolExecutors (store/lookup/load)
  |-- DaxL2Adapter: 3 ThreadPoolExecutors (store/lookup/load)
  |-- S3L2Adapter: CRT EventLoopGroup + own asyncio loop
  |-- FSL2Adapter: own asyncio loop
  |
  Multiprocess Server (standalone)
  |-- MessageQueueServer
  |     |-- AffinityThreadPool (max_gpu_workers, for GPU-bound handlers)
  |     |-- ThreadPoolExecutor (max_cpu_workers, for CPU handlers)
  |
  DaxBackend (plugin, standalone)
  |-- _restore_executor (ThreadPoolExecutor, restore_workers)
  |-- _restore_dispatch_executor (ThreadPoolExecutor, 1 worker)
  |
  AsyncSerdeProcessor
  |-- ThreadPoolExecutor (max_workers=1 default)
```

---

### 6. Key Design Patterns

1. **Shared event loop pattern**: The StorageManager's `self.loop` is the central dispatch point. Backends that need async operations use `asyncio.run_coroutine_threadsafe(coro, self.loop)` to submit work from synchronous caller threads onto this loop.

2. **Connector-owned thread pools**: Remote connectors (Valkey, HF3fs) own their own `ThreadPoolExecutor` because their I/O libraries (GLIDE, HF3FS) are synchronous and need true parallel threads. The connector submits sync I/O calls to its pool, then wraps results back into the shared asyncio loop.

3. **Priority queue executor**: Both `AsyncPQExecutor` (async coroutines) and `AsyncPQThreadPoolExecutor` (sync functions via `asyncio.to_thread()`) enforce priority ordering so that latency-sensitive reads (PEEK/PREFETCH) are not blocked behind bulk writes (PUT).

4. **Per-operation-type pools**: L2 adapters (RawBlock, Dax) use **3 separate pools** for store, lookup, and load operations. This isolates latency-sensitive loads from bulk stores.

5. **Affinity routing**: The MQ server's `AffinityThreadPool` hashes zmq client identity to a fixed thread, ensuring that GPU operations from the same client always run on the same thread (avoiding cross-thread GPU synchronization overhead).
</task_result>
</task>
```

---

## Assistant (Build · GLM-5.1 · 27.1s)

_Thinking:_

The agent has provided a very comprehensive answer. Let me summarize it concisely for the user, focusing on the key patterns and providing relevant code examples.

## IO线程池分配机制

LMCache 中有**6种线程池/执行器模式**，核心架构如下：

### 1. 共享事件循环（最核心）

`StorageManager` 在 `storage_manager.py:233-240` 创建一个共享 asyncio 事件循环，所有后端通过 `asyncio.run_coroutine_threadsafe()` 向其提交任务：

```python
self.loop = asyncio.new_event_loop()
self.thread = threading.Thread(
    target=start_loop_in_thread_with_exceptions,
    args=(self.loop,),
    name="storage-manager-event-loop",
)
self.thread.start()
```

`CreateStorageBackends()` 创建各后端时，把 `self.loop` 传给每个后端。

### 2. 各组件的线程池示例

| 组件 | 线程池类型 | 默认大小 | 配置参数 |
|------|-----------|---------|---------|
| LocalDiskBackend | `AsyncPQThreadPoolExecutor` | 4 | 硬编码（TODO） |
| GdsBackend | `ThreadPoolExecutor` | 4 | `gds_io_threads` |
| ValkeyConnector | `ThreadPoolExecutor` | 8 | `valkey_num_workers` |
| S3Connector | `awscrt.io.EventLoopGroup` | 64 | `s3_num_io_threads` |
| HF3fsConnector | `ThreadPoolExecutor` | 4 | `hf3fs_io_thread_num` |
| RawBlockL2Adapter | 3个独立 `ThreadPoolExecutor` | 2/1/4 | `num_store/lookup/load_workers` |
| MQ Server | `AffinityThreadPool` + `ThreadPoolExecutor` | CLI配置 | `max_gpu/cpu_workers` |

**LocalDiskBackend** (`local_disk_backend.py:48`)：
```python
self.executor = AsyncPQThreadPoolExecutor(loop, max_workers=4)
```
优先级队列：prefetch(0) > delete(1) > put(2)

**GdsBackend** (`gds_backend.py:264-274`)：
```python
thread_count = config.extra_config.get("gds_io_threads", 4)
self._thread_pool = ThreadPoolExecutor(
    max_workers=thread_count, thread_name_prefix="gds-io"
)
```

**ValkeyConnector** (`valkey_connector.py:120`) — 双层架构：
```python
self._executor = ThreadPoolExecutor(
    max_workers=num_workers, thread_name_prefix="valkey"
)
```
外层 `AsyncPQExecutor` 做优先级调度，内层 `ThreadPoolExecutor` 执行同步 GLIDE I/O。

**RawBlockL2Adapter** (`raw_block_l2_adapter.py:336-347`) — 每类操作独立池：
```python
self._store_pool = ThreadPoolExecutor(max_workers=config.num_store_workers, ...)
self._lookup_pool = ThreadPoolExecutor(max_workers=config.num_lookup_workers, ...)
self._load_pool = ThreadPoolExecutor(max_workers=config.num_load_workers, ...)
```
隔离延迟敏感的 load 和批量 store。

**MQ Server** (`server.py:222-229`) — GPU亲和线程池：
```python
server.add_affinity_thread_pool(affinity_types, max_workers=mp_config.max_gpu_workers)
server.add_normal_thread_pool(normal_types, max_workers=mp_config.max_cpu_workers)
```
`AffinityThreadPool` 按 zmq identity hash 分配固定线程，避免跨线程 GPU 同步开销。

### 3. 关键设计模式

- **RemoteBackend 本身不拥有线程池**，通过 `asyncio.run_coroutine_threadsafe(coro, self.loop)` 提交到共享循环，I/O 由其 Connector 的线程池执行
- **优先级队列**：`AsyncPQExecutor`/`AsyncPQThreadPoolExecutor` 确保 peek/prefetch 优先于 put
- **按操作类型隔离**：L2 adapter 用 3 个独立池，避免 store 堆塞影响 load 延迟
- **GPU亲和路由**：MQ server 的 AffinityThreadPool 保证同一客户端的 GPU 操作在同一线程

---

## User

RemoteBackend用的是什么connecotr? connector的又是是什么

---

## Assistant (Build · GLM-5.1 · 156.3s)

**Tool: task**

**Input:**
```json
{
  "description": "Explore RemoteBackend connector system",
  "prompt": "In the LMCache codebase at /home/models/src/LMCache, I need to understand:\n\n1. What connectors does RemoteBackend use? How is a connector chosen/created?\n2. What is the role/purpose of a connector in the architecture?\n3. How does CreateConnector work? What connector types exist?\n\nPlease:\n1. Read lmcache/v1/storage_backend/remote_backend.py - how it creates and uses connectors\n2. Read the connector creation factory (likely in lmcache/v1/storage_backend/connector/__init__.py or similar)\n3. List all connector implementations (files in the connector directory)\n4. Read the base/abstract connector interface\n5. Read at least 2-3 concrete connector implementations to understand the pattern\n\nReturn:\n- How RemoteBackend selects/creates its connector\n- The connector interface/API (what methods connectors must implement)\n- A complete list of all connector types\n- The role of connectors in the architecture (why this abstraction exists)\n- Key code snippets with file paths and line numbers",
  "subagent_type": "explore"
}
```

**Output:**
```
<task id="ses_06745393cffe4HvmArxeU7IS3q" state="completed">
<task_result>
Now I have a complete picture. Here is the comprehensive analysis:

---

## 1. How RemoteBackend Selects/Creates Its Connector

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/remote_backend.py`

RemoteBackend creates its connector in `init_connection()` (lines 118-160). There are two modes:

**Legacy URL mode** (lines 50-54): Uses `config.remote_url` directly (e.g., `redis://localhost:6379`).

**Plugin mode** (lines 44-47): Constructs a virtual URL `plugin://{plugin_name}` when `plugin_name` is provided.

The key call is at **lines 144-151**:

```python
self.connection = CreateConnector(
    url,
    self.loop,
    self.local_cpu_backend,
    self.config,
    self.metadata,
    plugin_name=self.plugin_name,
)
```

RemoteBackend then delegates virtually all remote operations to `self.connection`:
- **`submit_put_task`** (line 268): `self.connection.put(key, compressed_memory_obj)`
- **`get_blocking`** (line 363): `self.connection.get(key)`
- **`contains`** (lines 175-181): `self.connection.exists_sync(key)` or `self.connection.exists(key)`
- **`batched_contains`** (line 203): `self.connection.batched_contains(keys)`
- **`batched_get_blocking`** (line 419): `self.connection.batched_get(keys)`
- **`batched_submit_put_task`** (line 330): `self.connection.batched_put(keys, compressed_memory_objs)`
- **`batched_async_contains`** (line 531): `self.connection.batched_async_contains(...)`
- **`batched_get_non_blocking`** (line 569): `self.connection.batched_get_non_blocking(...)`
- **`remove`** (line 600): `self.connection.remove_sync(key)`
- **`close`** (line 618): `self.connection.close()`

---

## 2. How CreateConnector Works (The Factory)

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/__init__.py`

`CreateConnector` (lines 358-423) is the top-level factory function:

```python
def CreateConnector(...) -> InstrumentedRemoteConnector:
    # Validates URL has a scheme (line 415)
    if "://" not in url:
        raise ValueError(f"Invalid remote url {url}: missing scheme")

    # Creates a ConnectorManager which auto-discovers adapters (line 418)
    manager = ConnectorManager(url, loop, local_cpu_backend, config, metadata, plugin_name)
    connector = manager.create_connector()

    # Wraps in InstrumentedRemoteConnector for observability (line 423)
    return InstrumentedRemoteConnector(connector)
```

**ConnectorManager** (lines 214-355) orchestrates the selection:

1. **`__init__`** (lines 222-242): Creates a `ConnectorContext` holding all creation parameters, then runs two discovery phases:
   - **`_remote_adapters_builtin_launcher`** (lines 244-258): Uses `discover_subclasses()` from `lmcache/v1/utils/subclass_discovery.py` to auto-find all `ConnectorAdapter` subclasses in the connector package. The filter `lambda name: not name.startswith("_") and name.endswith("_adapter")` ensures only `*_adapter.py` modules are scanned. Each discovered class is instantiated with no arguments.
   - **`_remote_adapters_plugin_launcher`** (lines 260-346): Loads dynamic plugins from `config.remote_storage_plugins`. For each plugin name, it reads `module_path` and `class_name` from `config.extra_config`, dynamically imports the class, and wraps it as either a `ConnectorAdapter` instance or a `DynamicConnectorAdapter` (if it's a raw `RemoteConnector` subclass).

2. **`create_connector()`** (lines 348-355): Iterates through `self.adapters`, finds the first one where `adapter.can_parse(url)` returns `True`, then calls `adapter.create_connector(context)`:

```python
def create_connector(self) -> RemoteConnector:
    for adapter in self.adapters:
        if adapter.can_parse(self.context.url):
            connector = adapter.create_connector(self.context)
            return connector
    raise ValueError(f"No adapter found for URL: {self.context.url}")
```

---

## 3. The Adapter Pattern (ConnectorAdapter)

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/__init__.py`, lines 157-172

```python
class ConnectorAdapter(ABC):
    def __init__(self, schema: str = "") -> None:
        self.schema = schema

    def can_parse(self, url: str) -> bool:
        return self.schema != "" and url.startswith(self.schema)

    @abstractmethod
    def create_connector(self, context: ConnectorContext) -> RemoteConnector:
        pass
```

Each adapter:
- **Declares a URL schema** (e.g., `"redis://", "lm://", "infinistore://"`)
- **Overrides `can_parse()`** for more complex matching (e.g., Redis adapter also matches `rediss://` and `unix://`; FS/Mooncakestore/HF3fs adapters also match `plugin://` URLs)
- **Implements `create_connector()`** which parses the URL, extracts config, and instantiates the concrete `RemoteConnector` subclass

The adapter separates **URL routing/config extraction** from **protocol implementation**. Adding a new backend requires two files: `*_adapter.py` (routing) and `*_connector.py` (protocol).

---

## 4. The Connector Interface (RemoteConnector)

**File:** `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/base_connector.py`

### Required abstract methods (must implement):

| Method | Signature | Purpose |
|--------|-----------|---------|
| `exists` | `async def exists(key: CacheEngineKey) -> bool` | Check if key exists (async) |
| `exists_sync` | `def exists_sync(key: CacheEngineKey) -> bool` | Check if key exists (sync) |
| `get` | `async def get(key: CacheEngineKey) -> Optional[MemoryObj]` | Retrieve value by key |
| `put` | `async def put(key: CacheEngineKey, memory_obj: MemoryObj)` | Store value by key |
| `list` | `async def list() -> List[str]` | List all keys |
| `close` | `async def close()` | Close connection |

### Optional methods with default implementations:

| Method | Default | Purpose |
|--------|---------|---------|
| `support_ping()` / `ping()` | Returns `False` / raises NotImplementedError | Health check |
| `support_batched_get()` / `batched_get()` | Returns `False` / raises NotImplementedError | Batch retrieval |
| `support_batched_put()` / `batched_put()` | Returns `False` / raises NotImplementedError | Batch storage |
| `support_batched_async_contains()` / `batched_async_contains()` | Returns `True` / default implementation via `asyncio.gather` of individual `exists` | Batch existence check (returns prefix count) |
| `support_batched_get_non_blocking()` / `batched_get_non_blocking()` | Returns `True` / default implementation via `asyncio.gather` of individual `get` | Non-blocking batch retrieval |
| `support_batched_contains()` / `batched_contains()` | Returns `False` / raises NotImplementedError | Synchronous batched contains |
| `remove_sync()` | Raises NotImplementedError | Remove a key |
| `reshape_partial_chunk()` | Provided in base class (lines 86-138) | Handle partial (non-full) chunk reshaping |
| `post_init()` | No-op in base class | Post-initialization hook |

### Base class `__init__` (lines 36-83) computes common metadata:
- `save_chunk_meta`, `meta_shapes`, `meta_dtypes`, `meta_fmt`, `full_chunk_size_bytes`, `single_token_size`, `remote_metadata_bytes`

---

## 5. Complete List of All Connector Types

| # | URL Schema | Adapter Class | Connector Class | File | Description |
|---|-----------|--------------|-----------------|------|-------------|
| 1 | `resp://` | `RESPConnectorAdapter` | `RESPConnector` | `redis_adapter.py` / `redis_connector.py` | Native RESP protocol client (zero-copy via RESPClient) |
| 2 | `redis://`, `rediss://`, `unix://` | `RedisConnectorAdapter` | `RedisConnector` | `redis_adapter.py` / `redis_connector.py` | Redis via redis-py async client |
| 3 | `redis-sentinel://` | `RedisSentinelConnectorAdapter` | `RedisSentinelConnector` | `redis_adapter.py` / `redis_connector.py` | Redis Sentinel for HA |
| 4 | `redis-cluster://` | `RedisClusterConnectorAdapter` | `RedisClusterConnector` | `redis_adapter.py` / `redis_connector.py` | Redis Cluster mode |
| 5 | `lm://` | `LMServerConnectorAdapter` | `LMCServerConnector` | `lm_adapter.py` / `lm_connector.py` | LMCache built-in TCP server (socket-based) |
| 6 | `infinistore://` | `InfinistoreConnectorAdapter` | `InfinistoreConnector` | `infinistore_adapter.py` / `infinistore_connector.py` | RDMA via infinistore library |
| 7 | `mooncakestore://` / `plugin://mooncakestore` | `MooncakestoreConnectorAdapter` | `MooncakestoreConnector` | `mooncakestore_adapter.py` / `mooncakestore_connector.py` | Mooncake Transfer engine |
| 8 | `valkey://` | `ValkeyConnectorAdapter` | `ValkeyConnector` | `valkey_adapter.py` / `valkey_connector.py` | Valkey via GLIDE sync client (standalone or cluster) |
| 9 | `fs://` / `plugin://fs` | `FsConnectorAdapter` | `FSConnector` | `fs_adapter.py` / `fs_connector.py` | Local filesystem storage |
| 10 | `s3://` | `S3ConnectorAdapter` | `S3Connector` | `s3_adapter.py` / `s3_connector.py` | Amazon S3 storage |
| 11 | `eic://` | `EICConnectorAdapter` | `EICConnector` | `eic_adapter.py` / `eic_connector.py` | EIC (External Inference Cache) |
| 12 | `hf3fs://` / `plugin://hf3fs` | `HF3fsConnectorAdapter` | `HF3fsConnector` | `hf3fs_adapter.py` / `hf3fs_connector.py` | 3FS Usrbio interfaces (Samsung) |
| 13 | `plugin://hfbucket` | `HFBucketConnectorAdapter` | `HFBucketConnector` | `hfbucket_adapter.py` / `hfbucket_connector.py` | Hugging Face model buckets |
| 14 | `sagemaker-hyperpod://` | `SageMakerHyperPodConnectorAdapter` | `SageMakerHyperPodConnector` | `sagemaker_hyperpod_adapter.py` / `sagemaker_hyperpod_connector.py` | AWS SageMaker HyperPod distributed storage |
| 15 | `blackhole://` | `BlackholeConnectorAdapter` | `BlackholeConnector` | `blackhole_adapter.py` / `blackhole_connector.py` | Testing/development (discards all data) |
| 16 | `mock://` | `MockConnectorAdapter` | `MockConnector` | `mock_adapter.py` / `mock_connector.py` | Testing with simulated latency/throughput/capacity |
| 17 | `audit://` | `AuditConnectorAdapter` | `AuditConnector` | `audit_adapter.py` / `audit_connector.py` | Debugging wrapper that audits/checksums all operations |
| 18 | `external://` | `ExternalConnectorAdapter` | Dynamically loaded | `external_adapter.py` | Dynamic external connector (deprecated, use plugin framework) |
| 19 | `plugin://{type}` | `DynamicConnectorAdapter` | User-provided class | `__init__.py` lines 174-211 | Dynamic plugin loaded from config |

---

## 6. The Role of Connectors in the Architecture

Connectors serve as the **protocol abstraction layer** between RemoteBackend and the actual remote storage system. This design exists for several reasons:

### a) Separation of concerns
RemoteBackend handles **application-level logic**: serialization/deserialization, task management (put_tasks set), MLA worker-id remapping, timeout handling, error recovery, and observability. It delegates **wire-level protocol** entirely to the connector. The connector only needs to know how to send/receive raw bytes to a specific backend.

### b) Pluggable storage backends
Different deployments need different remote storage: Redis for shared caches, RDMA (Infinistore/Mooncakestore) for low-latency inter-node transfer, S3 for durable cloud storage, filesystem for local shared disks, etc. The connector abstraction allows swapping backends by changing a single URL string in config, with no code changes to RemoteBackend.

### c) Two-layer adapter pattern
The architecture uses a **two-layer factory**:
- **ConnectorAdapter** (`*_adapter.py`): Handles URL scheme matching, URL parsing, and config extraction. This is the routing layer.
- **RemoteConnector** (`*_connector.py`): Implements the actual protocol (Redis commands, RDMA operations, socket I/O, S3 API calls, etc.). This is the implementation layer.

This separation means adding a new backend requires no changes to existing code -- just add two new files that the auto-discovery mechanism (`discover_subclasses`) will pick up automatically.

### d) Observability wrapping
The final `InstrumentedRemoteConnector` wrapper (`instrumented_connector.py`) adds timing, byte-count, and stats monitoring around any connector, without modifying the connector itself. This is a **decorator pattern** that keeps metrics separate from protocol logic.

### e) Graceful degradation
Connectors expose capability flags (`support_batched_get()`, `support_batched_put()`, etc.). RemoteBackend checks these before calling batched methods (e.g., line 299: `if self.connection.support_batched_put()`), falling back to individual operations when a connector lacks batch support. This allows simple connectors to implement only the basic interface while advanced connectors (like RESP/Redis) offer optimized batch operations.

### Key architectural flow diagram:

```
LMCacheEngine
    |
    v
RemoteBackend (application logic: serialization, task tracking, MLA remapping, timeouts)
    |
    v
CreateConnector() factory --> ConnectorManager --> auto-discover adapters --> match URL schema
    |
    v
InstrumentedRemoteConnector (metrics/timing wrapper)
    |
    v
Concrete RemoteConnector (protocol implementation: Redis, RDMA, S3, FS, etc.)
```
</task_result>
</task>
```

---

## Assistant (Build · GLM-5.1 · 22.8s)

### RemoteBackend 的 Connector 选择机制

`RemoteBackend` 通过 `CreateConnector()` 工厂函数根据 URL scheme 自动选择 connector（`remote_backend.py:144-151`）：

```python
self.connection = CreateConnector(
    url, self.loop, self.local_cpu_backend, self.config,
    self.metadata, plugin_name=self.plugin_name,
)
```

**两种模式：**
- **Legacy URL**：直接用 `config.remote_url`，如 `redis://localhost:6379`
- **Plugin 模式**：构造 `plugin://{plugin_name}` URL

`CreateConnector`（`connector/__init__.py:358-423`）内部用 `ConnectorManager` 自动发现所有 `ConnectorAdapter` 子类，找到第一个 `can_parse(url)` 为 True 的适配器，创建对应 connector，最后包装为 `InstrumentedRemoteConnector`（监控装饰器）。

---

### Connector 的角色

Connector 是 **协议抽象层**，将 RemoteBackend 的应用逻辑（序列化、MLA worker_id 重映射、超时、错误恢复）与底层存储协议（Redis命令、RDMA操作、S3 API 等）分离：

```
RemoteBackend (应用逻辑) → InstrumentedRemoteConnector (监控) → Concrete RemoteConnector (协议实现)
```

Connector 接口（`base_connector.py`）定义了：
- **必须实现**：`exists`, `get`, `put`, `list`, `close`
- **可选实现**：`batched_get`, `batched_put`, `batched_contains`, `remove_sync`, `ping` 等（通过 `support_*()` 方法声明能力，RemoteBackend 会检查并降级）

---

### 全部 Connector 类型

| URL Schema | Connector | 说明 |
|-----------|-----------|------|
| `redis://`, `rediss://`, `unix://` | `RedisConnector` | Redis异步客户端 |
| `redis-sentinel://` | `RedisSentinelConnector` | Redis Sentinel HA |
| `redis-cluster://` | `RedisClusterConnector` | Redis Cluster |
| `resp://` | `RESPConnector` | RESP协议零拷贝 |
| `valkey://` | `ValkeyConnector` | Valkey (GLIDE客户端) |
| `lm://` | `LMCServerConnector` | LMCache内置TCP服务 |
| `infinistore://` | `InfinistoreConnector` | RDMA |
| `mooncakestore://` | `MooncakestoreConnector` | Mooncake传输引擎 |
| `s3://` | `S3Connector` | Amazon S3 |
| `fs://` | `FSConnector` | 本地文件系统 |
| `hf3fs://` | `HF3fsConnector` | 3FS Usrbio |
| `hfbucket://` | `HFBucketConnector` | HF模型桶 |
| `sagemaker-hyperpod://` | `SageMakerHyperPodConnector` | AWS HyperPod |
| `eic://` | `EICConnector` | 外部推理缓存 |
| `blackhole://` | `BlackholeConnector` | 测试（丢弃数据） |
| `mock://` | `MockConnector` | 测试（模拟延迟） |
| `audit://` | `AuditConnector` | 调试审计 |
| `plugin://{type}` | `DynamicConnectorAdapter` | 动态插件 |

**两层适配器模式**：每个后端分 `*_adapter.py`（URL路由+配置提取）和 `*_connector.py`（协议实现），新增后端只需添加这两个文件，自动发现机制会自动注册。

---

## User

RemoteConnector是怎么创建IO线程池的？多大的空间

---

## Assistant (Build · GLM-5.1 · 222.4s)

**Tool: task**

**Input:**
```json
{
  "description": "Explore RemoteConnector thread pool creation",
  "prompt": "In the LMCache codebase at /home/models/src/LMCache, I need to understand how each concrete RemoteConnector creates and uses IO thread pools, and their sizes.\n\nPlease read the following connector files thoroughly, focusing on thread pool / executor creation, event loop creation, and their configuration:\n\n1. lmcache/v1/storage_backend/connector/redis_connector.py - look for ThreadPoolExecutor, event loop, thread creation\n2. lmcache/v1/storage_backend/connector/valkey_connector.py - look for _ThreadWorkerPool, ThreadPoolExecutor\n3. lmcache/v1/storage_backend/connector/lm_connector.py - look for threading, event loop\n4. lmcache/v1/storage_backend/connector/infinistore_connector.py - look for threading, event loop\n5. lmcache/v1/storage_backend/connector/mooncakestore_connector.py - look for threading\n6. lmcache/v1/storage_backend/connector/s3_connector.py - look for EventLoopGroup, ThreadPool\n7. lmcache/v1/storage_backend/connector/fs_connector.py - look for threading\n8. lmcache/v1/storage_backend/connector/hf3fs_connector.py - look for ThreadPoolExecutor\n9. lmcache/v1/storage_backend/connector/resp_connector.py - look for threading\n10. lmcache/v1/storage_backend/connector/eic_connector.py - look for threading\n\nAlso read:\n- The base_connector.py for any shared thread pool infrastructure\n- The adapter files for configuration parameters that control thread pool sizes\n\nFor each connector, report:\n- Does it create its own thread pool / event loop, or use the shared StorageManager loop?\n- What type of thread pool/executor (ThreadPoolExecutor, asyncio loop, CRT EventLoopGroup, etc)?\n- What is the default size / max_workers?\n- What configuration parameter controls the size?\n- Exact file path and line numbers for thread pool creation code\n\nBe thorough - read each file completely.",
  "subagent_type": "explore"
}
```

**Output:**
```
<task id="ses_0671069bcffebc5PZC1E6uffa3" state="completed">
<task_result>
Now I have all the information needed. Let me compile the comprehensive report.

---

## Comprehensive Thread Pool / Event Loop Analysis for LMCache Remote Connectors

### Shared Infrastructure

**StorageManager Event Loop** (`/home/models/src/LMCache/lmcache/v1/storage_backend/storage_manager.py`)
- **Line 233**: `self.loop = asyncio.new_event_loop()` -- creates a **dedicated asyncio event loop**
- **Lines 235-240**: Creates a **dedicated OS thread** (`threading.Thread(target=start_loop_in_thread_with_exceptions, args=(self.loop,), name="storage-manager-event-loop")`) to run this loop
- This `loop` is passed to every connector via `ConnectorContext.loop`

**AsyncPQExecutor** (`/home/models/src/LMCache/lmcache/v1/storage_backend/job_executor/pq_executor.py`)
- **Line 18**: `def __init__(self, loop: asyncio.AbstractEventLoop, max_workers: int = 4)` -- default `max_workers=4`
- **Lines 37-39**: Spawns `max_workers` coroutine-based workers on the provided event loop via `asyncio.run_coroutine_threadsafe(self._worker(), loop)`
- These are **not OS threads** -- they are asyncio coroutine workers running on the shared event loop
- They form a priority queue that dispatches async operations with PEEK > PREFETCH > GET > PUT ordering

**C++ ConnectorBase** (`/home/models/src/LMCache/csrc/storage_backends/connector_base.h`)
- **Line 49**: `ConnectorBase(int num_workers)` -- creates `num_workers` **native C++ std::thread** workers
- **Lines 292-298** (`start_workers()`): Spawns `num_workers` threads: `workers_.emplace_back([this]() { this->worker_loop(); });`
- Each worker creates its own `WorkerConn` (TCP connection) at line 529: `ConnectionType conn = create_connection();`
- Workers pull from a shared `std::queue<Request>` with `std::condition_variable` synchronization
- Supports per-operation dedicated "lanes" via `WorkerPoolConfig.per_op_workers` (lines 27-30)

**ConnectorClientBase** (`/home/models/src/LMCache/lmcache/v1/storage_backend/native_clients/connector_client_base.py`)
- **Line 28**: `self.loop.add_reader(self._fd, self._on_ready)` -- registers an eventfd reader on the shared asyncio loop for completion delivery

---

### 1. RESPConnector (redis_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/redis_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| C++ worker pool | Native C++ std::thread | `csrc/storage_backends/connector_base.h:49,292-298` (indirectly via `LMCacheRedisClient`) | 8 | `extra_config["resp_num_threads"]` |
| AsyncPQExecutor | Asyncio coroutine workers on shared loop | `redis_connector.py:63` | 4 (default) | Hard-coded default in `AsyncPQExecutor.__init__` |

**Key details**:
- **Line 43-62**: `RESPConnector.__init__` accepts `num_threads: int = 8`, creates `RESPClient(host, port, num_threads, loop, username, password)`
- **RESPClient** (`/home/models/src/LMCache/lmcache/v1/storage_backend/native_clients/resp_client.py`, line 34): passes `num_workers` to `LMCacheRedisClient(host, port, num_workers, username, password)` -- the C++ extension
- **C++ LMCacheRedisClient** (`/home/models/src/LMCache/csrc/storage_backends/redis/pybind.cpp`, line 10): takes `num_workers` as its 3rd constructor argument; inherits from `ConnectorBase<WorkerConn>` which creates `num_workers` native threads
- **Line 63**: `self.pq_executor = AsyncPQExecutor(loop)` -- uses shared StorageManager loop with default `max_workers=4`
- **Adapter** (`/home/models/src/LMCache/lmcache/v1/storage_backend/connector/redis_adapter.py`, line 44): `self.resp_num_threads = int(extra_config.get("resp_num_threads", 8))`

**Does it use the shared StorageManager loop?**: Yes (for AsyncPQExecutor). The C++ threads are independent native threads.

---

### 2. RedisConnector (redis_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/redis_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| AsyncPQExecutor | Asyncio coroutine workers on shared loop | `redis_connector.py:263` | 4 | Hard-coded default |
| redis.ConnectionPool | Redis-managed connection pool | `redis_connector.py:256-258` | 150 | Hard-coded `max_connections=150` |

**Key details**:
- **Lines 253-258**: `self.max_connections = 150`, `self.sem = asyncio.Semaphore(self.max_connections)`, `self.pool = redis.ConnectionPool.from_url(url, max_connections=self.max_connections)`
- **Line 260**: `self.loop = loop` (shared StorageManager loop)
- **Line 263**: `self.pq_executor = AsyncPQExecutor(loop)`
- The `redis.ConnectionPool` manages connections internally but is not a thread pool per se -- it provides async connections that run on the event loop.

**Does it use the shared StorageManager loop?**: Yes. No separate thread pool.

---

### 3. RedisClusterConnector (redis_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/redis_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| AsyncPQExecutor | Asyncio coroutine workers on shared loop | `redis_connector.py:641` | 4 | Hard-coded default |
| RedisCluster connection pool | Redis-managed cluster connections | `redis_connector.py:627-637` | 150 | Hard-coded `max_connections=150` |

**Key details**:
- **Lines 626-637**: `self.max_connections = 150`, `self.sem = asyncio.Semaphore(self.max_connections)`, `self.cluster = RedisCluster(..., max_connections=self.max_connections)`
- **Line 638**: `self.loop = loop`
- **Line 641**: `self.pq_executor = AsyncPQExecutor(loop)`
- Nearly identical pattern to RedisConnector.

**Does it use the shared StorageManager loop?**: Yes. No separate thread pool.

---

### 4. RedisSentinelConnector (redis_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/redis_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| None | No thread pool or executor created | N/A | N/A | N/A |

**Key details**:
- **Lines 469-509**: Uses **synchronous** `redis.Sentinel` client (not async) -- `self.master` and `self.slave` are sync Redis objects
- **Line 509**: `self.local_cpu_backend = local_cpu_backend` -- no loop or executor stored
- All operations (`exists`, `get`, `put`) are synchronous calls to the sync redis client

**Does it use the shared StorageManager loop?**: No -- it doesn't use async operations at all.

---

### 5. ValkeyConnector (valkey_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/valkey_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| `_ThreadWorkerPool._executor` | Python `ThreadPoolExecutor` | `valkey_connector.py:120-123` | `num_workers` (default 8) | `extra_config["valkey_num_workers"]` or `extra_config["valkey_sync_num_workers"]` |
| AsyncPQExecutor | Asyncio coroutine workers on shared loop | `valkey_connector.py:326` | 4 | Hard-coded default |

**Key details**:
- **Lines 70-135**: `_ThreadWorkerPool.__init__` creates `ThreadPoolExecutor(max_workers=num_workers, thread_name_prefix="valkey")` at **lines 120-123**
- **Lines 124-127**: Warm-up phase creates a GLIDE sync client on each thread via `self._executor.submit(self._get_client)`
- **Line 117**: `self._local = threading.local()` for per-thread GLIDE client instances
- **Lines 290-325**: `ValkeyConnector.__init__` accepts `num_workers: int = 8`, creates `_ThreadWorkerPool(host, port, num_workers, ...)` at line 314 and `AsyncPQExecutor(loop)` at line 326
- Operations are submitted as `Future` objects to the ThreadPoolExecutor, then wrapped via `asyncio.wrap_future()` to run on the shared loop
- **Adapter** (`/home/models/src/LMCache/lmcache/v1/storage_backend/connector/valkey_adapter.py`, lines 54-58): `num_workers = int(extra_config.get("valkey_num_workers", extra_config.get("valkey_sync_num_workers", 8)))`

**Does it use the shared StorageManager loop?**: Yes (for AsyncPQExecutor and `asyncio.wrap_future`). The ThreadPoolExecutor is a **separate, dedicated Python thread pool**.

---

### 6. LMCServerConnector (lm_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/lm_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| None | No thread pool or executor created | N/A | N/A | N/A |

**Key details**:
- **Lines 29-54**: `__init__` creates a raw `socket.socket` and stores the shared `loop`
- **Line 54**: `self.async_socket_lock = asyncio.Lock()` -- uses the shared event loop for async serialization
- **Lines 124,136**: Uses `loop.sock_sendall()` for async socket I/O (transport-based API)
- **Lines 146,162**: Uses synchronous `self.client_socket.sendall()` and `self.client_socket.recv()` within `async with self.async_socket_lock` blocks
- No ThreadPoolExecutor, no AsyncPQExecutor, no dedicated threads

**Does it use the shared StorageManager loop?**: Yes. Operations run directly on it with `asyncio.Lock` for serialization.

---

### 7. InfinistoreConnector (infinistore_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/infinistore_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| Default Python executor | (implicit) via `loop.run_in_executor(None, blocking_io)` | `infinistore_connector.py:85` | Python default (min(32, os.cpu_count()+4)) | N/A |
| asyncio.Queue | Asyncio queue on shared loop | `infinistore_connector.py:64-65` | 16 (maxsize) | Hard-coded `MAX_BUFFER_CNT=16` |

**Key details**:
- **Line 85**: `await self.loop.run_in_executor(None, blocking_io)` in `exists()` -- uses the **default Python ThreadPoolExecutor** (no dedicated executor)
- **Lines 64-65**: `self.send_queue: asyncio.Queue[int] = asyncio.Queue(maxsize=MAX_BUFFER_CNT)` and `self.recv_queue: asyncio.Queue[int] = asyncio.Queue(maxsize=MAX_BUFFER_CNT)` -- buffer management queues on the shared loop, not thread pools
- **Lines 96,159**: `await self.rdma_conn.rdma_read_cache_async(...)` and `await self.rdma_conn.rdma_write_cache_async(...)` -- async RDMA operations handled by the infinistore library internally
- No AsyncPQExecutor, no dedicated ThreadPoolExecutor

**Does it use the shared StorageManager loop?**: Yes. RDMA async ops and `run_in_executor(None)` use the loop's default executor.

---

### 8. MooncakestoreConnector (mooncakestore_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/mooncakestore_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| Default Python executor | (implicit) via `asyncio.to_thread()` | Lines 569,613,732,766,804 | Python default (min(32, os.cpu_count()+4)) | N/A |

**Key details**:
- **Line 569**: `await asyncio.to_thread(self.store.batch_get_into, key_strs, buffer_ptrs, buffer_sizes)` -- in `_batch_get_into`
- **Line 613**: `await asyncio.to_thread(self.store.batch_get_buffer, key_strs)` -- in `_batch_get_buffer`
- **Lines 732-738**: `await asyncio.to_thread(self.store.batch_put_from, ...)` -- in `_batched_put_zero_copy`
- **Lines 766-773**: `await asyncio.to_thread(self.store.put_from, ...)` -- in `_put_without_metadata`
- **Lines 804-808**: `await asyncio.to_thread(self.store.put_parts, ...)` -- in `_put_with_metadata`
- `asyncio.to_thread()` submits blocking functions to the event loop's **default ThreadPoolExecutor**
- No AsyncPQExecutor, no dedicated ThreadPoolExecutor created by the connector

**Does it use the shared StorageManager loop?**: Yes. All `asyncio.to_thread` calls use the loop's default executor.

---

### 9. S3Connector (s3_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/s3_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| `io.EventLoopGroup` | AWS CRT native event loop group (C threads) | `s3_connector.py:103` | `s3_num_io_threads` (default 64) | `extra_config["s3_num_io_threads"]` |
| AsyncPQExecutor | Asyncio coroutine workers on shared loop | `s3_connector.py:167` | 4 | Hard-coded default |

**Key details**:
- **Line 103**: `event_loop_group = io.EventLoopGroup(s3_num_io_threads)` -- creates AWS CRT native event loop group with `s3_num_io_threads` threads
- **Line 104**: `host_resolver = io.DefaultHostResolver(event_loop_group)`
- **Line 105**: `client_bootstrap = io.ClientBootstrap(event_loop_group, host_resolver)` -- CRT client bootstrap uses the event loop group for all I/O
- **Lines 108-118**: Credentials provider also uses the `client_bootstrap`
- **Lines 146-153**: `self.s3_client = s3.S3Client(bootstrap=client_bootstrap, ...)` -- all S3 requests use CRT threads
- **Line 167**: `self.pq_executor = AsyncPQExecutor(loop)`
- **Adapter** (`/home/models/src/LMCache/lmcache/v1/storage_backend/connector/s3_adapter.py`, line 32): `self.s3_num_io_threads = int(extra_config.get("s3_num_io_threads", 64))`

**Does it use the shared StorageManager loop?**: Yes (for AsyncPQExecutor and `asyncio.wrap_future`). The CRT EventLoopGroup is a **separate native thread pool** managed by AWS CRT.

---

### 10. FSConnector (fs_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/fs_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| Default Python executor | (implicit) via `loop.run_in_executor(None, ...)` | Lines 229,355 | Python default (min(32, os.cpu_count()+4)) | N/A |
| aiofiles internal thread pool | aiofiles-managed | Used throughout | Managed by aiofiles | N/A |

**Key details**:
- **Lines 228-231**: O_DIRECT reads use `await self.loop.run_in_executor(None, self._get_with_odirect, file_path)` -- default executor
- **Lines 355-357**: O_DIRECT writes use `await self.loop.run_in_executor(None, self._put_with_odirect, temp_path, buffer)` -- default executor
- All other I/O uses `aiofiles` (async file I/O library) which manages its own internal thread pool
- No AsyncPQExecutor, no dedicated ThreadPoolExecutor

**Does it use the shared StorageManager loop?**: Yes. Uses `aiofiles` and default executor.

---

### 11. HF3fsConnector (hf3fs_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/hf3fs_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| `self.io_executor` | Python `ThreadPoolExecutor` | `hf3fs_connector.py:110-112` | `io_thread_num` (default 4) | `extra_config["hf3fs_io_thread_num"]` |
| Hf3fsClient per-thread instances | `threading.local()` singleton per thread | `hf3fs_connector.py:304-321` | N/A (per-thread) | N/A |

**Key details**:
- **Lines 110-112**: `self.io_executor = ThreadPoolExecutor(max_workers=io_thread_num, thread_name_prefix="HF3fsConnector_executor")` -- **dedicated ThreadPoolExecutor**
- **Lines 144**: `await self.loop.run_in_executor(self.io_executor, self.sync_get, key)` -- uses its own executor, **not** the default one
- **Lines 218-219**: `await self.loop.run_in_executor(self.io_executor, self.sync_put, key, memory_obj)` -- same
- **Lines 304-321**: `Hf3fsClient` uses `threading.local()` for singleton-per-thread pattern -- each thread in the executor gets its own `Hf3fsClient` instance
- **Lines 126-130**: `_close()` shuts down `self.io_executor.shutdown(wait=True)`
- **Adapter** (`/home/models/src/LMCache/lmcache/v1/storage_backend/connector/hf3fs_adapter.py`, line 76): `self.hf3fs_io_thread_num = self._get_positive_int("hf3fs_io_thread_num", 4)`
- **Validation** (hf3fs_adapter.py, line 181): `if not 2 <= self.hf3fs_io_thread_num <= 16`

**Does it use the shared StorageManager loop?**: Yes (for `run_in_executor` coordination). The `io_executor` is a **separate dedicated thread pool**.

---

### 12. EICConnector (eic_connector.py)

**File**: `/home/models/src/LMCache/lmcache/v1/storage_backend/connector/eic_connector.py`

**Thread pool / event loop creation**:
| Component | Type | Location | Default Size | Config Parameter |
|-----------|------|----------|-------------|-----------------|
| AsyncPQExecutor | Asyncio coroutine workers on shared loop | `eic_connector.py:129` | 4 | Hard-coded default |
| eic.Client internal threads | Native C++ threads (managed by eic library) | `eic_connector.py:184` | `eic_thread_num` (default 6) | `config.get("eic_thread_num", 6)` from YAML config |

**Key details**:
- **Line 129**: `self.pq_executor = AsyncPQExecutor(loop)` -- on shared loop with default `max_workers=4`
- **Lines 161-162**: `eic_thread_num = config.get("eic_thread_num", 6)` -- read from LMCACHE_CONFIG_FILE YAML, but **not passed to any Python thread pool**. This value likely configures the native `eic.Client` C++ library's internal threading
- **Line 184**: `self.connection = eic.Client()` -- the native C++ client manages its own internal threads
- The `AsyncPQExecutor` dispatches async operations on the shared event loop with priority scheduling

**Does it use the shared StorageManager loop?**: Yes (for AsyncPQExecutor). The `eic.Client` manages its own native threads internally.

---

### Summary Table

| Connector | Own Thread Pool? | Own Event Loop? | Type | Default Size | Config Parameter | Shared Loop? | File & Line |
|-----------|:-:|:-:|------|:-:|------|:-:|------------|
| **RESPConnector** | Yes (C++) | No | C++ `std::thread` (via ConnectorBase) | 8 | `extra_config["resp_num_threads"]` | Yes (AsyncPQ) | redis_connector.py:62, csrc/connector_base.h:292 |
| **RedisConnector** | No | No | AsyncPQExecutor (asyncio coroutines) | 4 | Hard-coded | Yes | redis_connector.py:263 |
| **RedisClusterConn** | No | No | AsyncPQExecutor (asyncio coroutines) | 4 | Hard-coded | Yes | redis_connector.py:641 |
| **RedisSentinelConn** | No | No | None (synchronous) | N/A | N/A | No | redis_connector.py:469-509 |
| **ValkeyConnector** | Yes | No | `ThreadPoolExecutor` + AsyncPQExecutor | 8 | `extra_config["valkey_num_workers"]` or `valkey_sync_num_workers` | Yes | valkey_connector.py:120-123, 326 |
| **LMCServerConn** | No | No | `asyncio.Lock` (direct on loop) | N/A | N/A | Yes | lm_connector.py:54 |
| **InfinistoreConn** | No (uses default) | No | `loop.run_in_executor(None, ...)` | Python default | N/A | Yes | infinistore_connector.py:85 |
| **MooncakestoreConn** | No (uses default) | No | `asyncio.to_thread()` | Python default | N/A | Yes | mooncakestore_connector.py:569,613,732,766,804 |
| **S3Connector** | Yes (CRT) | No | AWS CRT `EventLoopGroup` + AsyncPQExecutor | 64 | `extra_config["s3_num_io_threads"]` | Yes | s3_connector.py:103, 167 |
| **FSConnector** | No (uses default) | No | `loop.run_in_executor(None, ...)` + aiofiles | Python default | N/A | Yes | fs_connector.py:229,355 |
| **HF3fsConnector** | Yes | No | `ThreadPoolExecutor` | 4 | `extra_config["hf3fs_io_thread_num"]` | Yes | hf3fs_connector.py:110-112 |
| **EICConnector** | No (native eic) | No | AsyncPQExecutor + native `eic.Client` | 4 (AsyncPQ) / 6 (eic native) | `eic_thread_num` in YAML | Yes | eic_connector.py:129, 161-162 |

### AsyncPQExecutor Usage Summary

The `AsyncPQExecutor` (with default `max_workers=4`) is used by these connectors for priority-queue scheduling of async coroutines on the shared StorageManager event loop:
- RESPConnector (redis_connector.py:63)
- RedisConnector (redis_connector.py:263)
- RedisClusterConnector (redis_connector.py:641)
- ValkeyConnector (valkey_connector.py:326)
- S3Connector (s3_connector.py:167)
- EICConnector (eic_connector.py:129)

These connectors do NOT use AsyncPQExecutor:
- RedisSentinelConnector (synchronous)
- LMCServerConnector (uses `asyncio.Lock`)
- InfinistoreConnector (uses `run_in_executor`)
- MooncakestoreConnector (uses `asyncio.to_thread`)
- FSConnector (uses aiofiles / `run_in_executor`)
- HF3fsConnector (uses dedicated `ThreadPoolExecutor`)

### Connectors Using Default Python Executor

Three connectors rely on the **implicit default Python ThreadPoolExecutor** (via `loop.run_in_executor(None, ...)` or `asyncio.to_thread()`) whose default size is `min(32, os.cpu_count() + 4)`:
- InfinistoreConnector
- MooncakestoreConnector
- FSConnector

### Connectors with Dedicated Thread Pools

Four connectors create their own dedicated thread pools:
- **RESPConnector**: C++ `std::thread` pool via `ConnectorBase(num_workers)` -- size controlled by `resp_num_threads`
- **ValkeyConnector**: Python `ThreadPoolExecutor` via `_ThreadWorkerPool` -- size controlled by `valkey_num_workers`
- **S3Connector**: AWS CRT `EventLoopGroup` -- size controlled by `s3_num_io_threads`
- **HF3fsConnector**: Python `ThreadPoolExecutor` via `self.io_executor` -- size controlled by `hf3fs_io_thread_num`
</task_result>
</task>
```

---

## Assistant (Build · GLM-5.1 · 22.8s)

各 Connector 的 IO 线程池创建方式如下：

### 三种模式

**1. 自建线程池（4个 Connector）**

| Connector | 线程池类型 | 默认大小 | 配置参数 | 位置 |
|-----------|----------|---------|---------|------|
| RESPConnector | C++ `std::thread`（ConnectorBase） | 8 | `extra_config["resp_num_threads"]` | `redis_connector.py:43` → `csrc/connector_base.h:292` |
| ValkeyConnector | `ThreadPoolExecutor`（_ThreadWorkerPool） | 8 | `extra_config["valkey_num_workers"]` | `valkey_connector.py:120-123` |
| S3Connector | AWS CRT `EventLoopGroup`（C线程） | 64 | `extra_config["s3_num_io_threads"]` | `s3_connector.py:103` |
| HF3fsConnector | `ThreadPoolExecutor` | 4 | `extra_config["hf3fs_io_thread_num"]`（范围2-16） | `hf3fs_connector.py:110-112` |

示例 — **ValkeyConnector** (`valkey_connector.py:120-127`)：
```python
self._executor = ThreadPoolExecutor(
    max_workers=num_workers, thread_name_prefix="valkey"
)
self._executor.submit(self._get_client)  # 每个线程预热一个GLIDE client
```

示例 — **HF3fsConnector** (`hf3fs_connector.py:110-112`)：
```python
self.io_executor = ThreadPoolExecutor(
    max_workers=io_thread_num, thread_name_prefix="HF3fsConnector_executor"
)
# 使用: await self.loop.run_in_executor(self.io_executor, self.sync_get, key)
```

**2. 共享 StorageManager 事件循环 + AsyncPQExecutor（6个 Connector）**

`AsyncPQExecutor` 在共享 loop 上跑 `max_workers=4` 个协程 worker，优先级：PEEK > PREFETCH > GET > PUT。

使用的 Connector：RESPConnector、RedisConnector、RedisClusterConnector、ValkeyConnector、S3Connector、EICConnector

示例 — **RedisConnector** (`redis_connector.py:263`)：
```python
self.pq_executor = AsyncPQExecutor(loop)
# 连接池: self.pool = redis.ConnectionPool.from_url(url, max_connections=150)
```

**3. 默认 Python ThreadPoolExecutor（3个 Connector）**

通过 `loop.run_in_executor(None, ...)` 或 `asyncio.to_thread()` 使用 Python 默认线程池（大小 `min(32, os.cpu_count()+4)`）。

| Connector | 方式 | 位置 |
|-----------|------|------|
| InfinistoreConnector | `loop.run_in_executor(None, blocking_io)` | `infinistore_connector.py:85` |
| MooncakestoreConnector | `asyncio.to_thread()` | `mooncakestore_connector.py:569,613,732` |
| FSConnector | `loop.run_in_executor(None, ...)` + aiofiles | `fs_connector.py:229,355` |

**无线程池**：RedisSentinelConnector（纯同步）、LMCServerConnector（asyncio.Lock 直接在共享 loop 上跑）

---

### 关键：ValkeyConnector 双层架构

最复杂的是 **ValkeyConnector**，采用双层设计：

```python
# 外层: AsyncPQExecutor 优先级调度（共享loop，4个协程worker）
self.pq_executor = AsyncPQExecutor(loop)          # valkey_connector.py:326

# 内层: ThreadPoolExecutor 实际I/O（自建，8个OS线程）
self._executor = ThreadPoolExecutor(max_workers=num_workers)  # valkey_connector.py:120
```

调用链：`public_method → pq_executor.submit_job → _internal_method → _executor.submit → ThreadPoolExecutor(OS线程执行GLIDE同步IO)`，结果通过 `asyncio.wrap_future()` 回到共享 loop。

---

## User

AsyncPQThreadPoolExecutor是1个小城，4个协程吗？

---

## Assistant (Build · GLM-5.1 · 5.5s)

_Thinking:_

The user is asking whether AsyncPQThreadPoolExecutor is 1 thread with 4 coroutines. Let me check the actual implementation.

**Tool: read**

**Input:**
```json
{
  "filePath": "/home/models/src/LMCache/lmcache/v1/storage_backend/job_executor/pq_executor.py"
}
```

**Output:**
```
<path>/home/models/src/LMCache/lmcache/v1/storage_backend/job_executor/pq_executor.py</path>
<type>file</type>
<content>
1: # SPDX-License-Identifier: Apache-2.0
2: # Standard
3: from asyncio import CancelledError
4: from concurrent.futures import Future
5: from typing import Any, Awaitable, Callable
6: import asyncio
7: import itertools
8: 
9: # First Party
10: from lmcache.v1.storage_backend.job_executor.base_executor import BaseJobExecutor
11: 
12: _SENTINEL = object()
13: 
14: 
15: class AsyncPQExecutor(BaseJobExecutor):
16:     """Execute async functions with a priority queue and using event loop"""
17: 
18:     def __init__(self, loop: asyncio.AbstractEventLoop, max_workers: int = 4):
19:         max_size = 0  # infinite
20:         self.loop = loop
21:         # (priority, time_id, fn, args, kwargs, done_future)
22:         self._queue: asyncio.PriorityQueue[
23:             tuple[
24:                 int,
25:                 int,
26:                 Callable[..., Awaitable[Any]] | object,
27:                 Any,
28:                 dict[str, Any],
29:                 asyncio.Future[Any] | None,
30:             ]
31:         ] = asyncio.PriorityQueue(maxsize=max_size)
32:         # counter breaks ties for equal priority items (FCFS)
33:         self._counter = itertools.count()
34:         self.max_workers = max_workers
35:         # we don't use asyncio.create_task so that PQ executor can be invoked
36:         # from sync code
37:         self._workers: list[Future] = [
38:             asyncio.run_coroutine_threadsafe(self._worker(), loop)
39:             for _ in range(max_workers)
40:         ]
41:         self._closed = False
42: 
43:     async def submit_job(
44:         self,
45:         fn: Callable[..., Awaitable[Any]],
46:         *args: Any,
47:         **kwargs: Any,
48:     ) -> Any:
49:         # Assign highest priority by default
50:         priority = kwargs.pop("priority", 0)
51:         done: asyncio.Future[Any] = self.loop.create_future()
52:         await self._queue.put((priority, next(self._counter), fn, args, kwargs, done))
53:         return await done
54: 
55:     async def _worker(self):
56:         while True:
57:             try:
58:                 item = await self._queue.get()
59:             except GeneratorExit:
60:                 # GeneratorExit means the generator/coroutine is being closed,
61:                 # likely during interpreter shutdown.
62:                 # Just exit without calling task_done()
63:                 break
64:             except CancelledError:
65:                 # Task cancellation, we should exit cleanly
66:                 break
67:             except Exception:
68:                 # Any other exception, break the loop
69:                 break
70: 
71:             # Detect sentinel both as raw object and wrapped tuple
72:             if item is _SENTINEL or (
73:                 isinstance(item, tuple) and len(item) >= 3 and item[2] is _SENTINEL
74:             ):
75:                 self._queue.task_done()
76:                 break
77: 
78:             _, _, fn, args, kwargs, done = item
79:             try:
80:                 result = await fn(*args, **kwargs)
81:                 done.set_result(result)
82:             except Exception as e:
83:                 done.set_exception(e)
84:             finally:
85:                 # decrement task count
86:                 # join needs to wait until task count is zero
87:                 self._queue.task_done()
88: 
89:     async def shutdown_async(self, wait: bool = True) -> None:
90:         """Async shutdown — safe to call from a coroutine on the event loop.
91: 
92:         Args:
93:             wait: If True, drain the queue before shutting down.
94:         """
95:         if self._closed:
96:             return
97:         self._closed = True
98: 
99:         # Enqueue comparable sentinel tuples with the highest priority value so
100:         # that outstanding work drains before shutdown signals are consumed.
101:         # Use a very large integer to satisfy the typed queue's expected int priority
102:         sentinel_priority = 2**31 - 1
103: 
104:         # Push sentinel for each worker
105:         for _ in range(self.max_workers):
106:             await self._queue.put(
107:                 (sentinel_priority, next(self._counter), _SENTINEL, None, {}, None)
108:             )
109: 
110:         # Cancel all worker futures to ensure they stop
111:         # Even when wait=False, we should cancel workers to prevent hanging coroutines
112:         for fut in self._workers:
113:             if not fut.done():
114:                 fut.cancel()
115: 
116:         if wait:
117:             # Let workers drain tasks and exit gracefully
118:             await self._queue.join()
119:             # Wait for all workers to complete (with cancellation handling)
120:             await asyncio.gather(
121:                 *[asyncio.wrap_future(fut, loop=self.loop) for fut in self._workers],
122:                 return_exceptions=True,
123:             )
124: 
125:     def shutdown(self, wait: bool = True) -> None:
126:         """Sync shutdown — blocks the calling thread until shutdown is complete."""
127:         future = asyncio.run_coroutine_threadsafe(self.shutdown_async(wait), self.loop)
128:         if wait:
129:             # Propagate exceptions if any
130:             future.result()
131: 
132: 
133: class AsyncPQThreadPoolExecutor(AsyncPQExecutor):
134:     """Execute sync functions with a priority queue and using threadpool"""
135: 
136:     def __init__(self, loop: asyncio.AbstractEventLoop, max_workers: int = 4):
137:         max_size = 0  # infinite
138:         self.loop = loop
139:         self._queue: asyncio.PriorityQueue[
140:             tuple[
141:                 int,
142:                 int,
143:                 Callable[..., Any] | object,
144:                 Any,
145:                 dict[str, Any],
146:                 asyncio.Future[Any] | None,
147:             ]
148:         ] = asyncio.PriorityQueue(maxsize=max_size)
149:         # counter breaks ties for equal priority items (FCFS)
150:         self._counter = itertools.count()
151:         self.max_workers = max_workers
152:         self._workers = []
153:         for _ in range(max_workers):
154:             self._workers.append(
155:                 asyncio.run_coroutine_threadsafe(self._worker(), self.loop)
156:             )
157:         self._closed = False
158: 
159:     async def _worker(self):
160:         while True:
161:             try:
162:                 item = await self._queue.get()
163:             except GeneratorExit:
164:                 # GeneratorExit means the generator/coroutine is being closed,
165:                 # likely during interpreter shutdown.
166:                 # Just exit without calling task_done()
167:                 break
168:             except CancelledError:
169:                 # Task cancellation, we should exit cleanly
170:                 break
171:             except Exception:
172:                 # Any other exception, break the loop
173:                 break
174: 
175:             # Detect sentinel both as raw object and wrapped tuple
176:             if item is _SENTINEL or (
177:                 isinstance(item, tuple) and len(item) >= 3 and item[2] is _SENTINEL
178:             ):
179:                 self._queue.task_done()
180:                 break
181: 
182:             _, _, fn, args, kwargs, done = item
183:             try:
184:                 result = await asyncio.to_thread(fn, *args, **kwargs)
185:                 done.set_result(result)
186:             except Exception as e:
187:                 done.set_exception(e)
188:             finally:
189:                 # decrement task count
190:                 # join needs to wait until task count is zero
191:                 self._queue.task_done()
192: 
193:     async def shutdown_async(self, wait: bool = True) -> None:
194:         """Async shutdown — safe to call from a coroutine on the event loop.
195: 
196:         Args:
197:             wait: If True, drain the queue before shutting down.
198:         """
199:         if self._closed:
200:             return
201:         self._closed = True
202:         # Enqueue comparable sentinel tuples with the highest priority value so
203:         # that outstanding work drains before shutdown signals are consumed.
204:         # Use a very large integer to satisfy the typed queue's expected int priority
205:         sentinel_priority = 2**31 - 1
206:         for _ in range(self.max_workers):
207:             await self._queue.put(
208:                 (sentinel_priority, next(self._counter), _SENTINEL, None, {}, None)
209:             )
210: 
211:         if wait:
212:             # Let workers drain tasks and exit gracefully
213:             await self._queue.join()
214:             # Cancel all worker futures after queue is drained
215:             for fut in self._workers:
216:                 if not fut.done():
217:                     fut.cancel()
218:             # Wait for all workers to complete (with cancellation handling)
219:             await asyncio.gather(
220:                 *[asyncio.wrap_future(fut, loop=self.loop) for fut in self._workers],
221:                 return_exceptions=True,
222:             )
223:         else:
224:             # When wait=False, just cancel workers immediately
225:             # This may cause RuntimeError if loop is closed, but that's expected
226:             for fut in self._workers:
227:                 if not fut.done():
228:                     fut.cancel()
229: 
230:     def shutdown(self, wait: bool = True) -> None:
231:         """Sync shutdown — blocks the calling thread until shutdown is complete."""
232:         future = asyncio.run_coroutine_threadsafe(self.shutdown_async(wait), self.loop)
233:         if wait:
234:             # Propagate exceptions if any
235:             future.result()

(End of file - total 235 lines)
</content>
```

---

## Assistant (Build · GLM-5.1 · 13.4s)

不是。`AsyncPQThreadPoolExecutor` 是 **4个协程worker** 在**1个共享事件循环线程**上运行，每个协程通过 `asyncio.to_thread()` 将同步I/O调度到 **Python默认ThreadPoolExecutor**（大小 `min(32, os.cpu_count()+4)`）。

关键代码在 `pq_executor.py:153-156` 和 `184`：

```python
# 创建4个协程worker，提交到共享loop的1个OS线程上
for _ in range(max_workers):
    self._workers.append(
        asyncio.run_coroutine_threadsafe(self._worker(), self.loop)
    )

# 每个协程worker拿到任务后，把同步函数丢给默认线程池
async def _worker(self):
    ...
    result = await asyncio.to_thread(fn, *args, **kwargs)  # 调度到默认ThreadPoolExecutor
```

所以实际架构是：

```
1个OS线程 (storage-manager-event-loop)
  └── 4个协程worker（从PriorityQueue取任务）
       └── 每个worker通过 asyncio.to_thread() 把同步I/O丢给默认ThreadPoolExecutor
            └── 默认ThreadPoolExecutor: min(32, cpu_count+4) 个OS线程执行实际I/O
```

**4个协程**控制的是**并发调度粒度**（最多4个任务同时处于"正在执行"状态），而非4个OS线程。实际的磁盘I/O由Python默认线程池的OS线程完成。

与 `AsyncPQExecutor`（父类）的区别仅在第184行：父类直接 `await fn(*args, **kwargs)`（fn是异步函数），而 `AsyncPQThreadPoolExecutor` 用 `await asyncio.to_thread(fn, *args, **kwargs)` 把**同步**函数桥接到线程池。

---

## User

ThreadPoolExecutor中是多少个线程

---

## Assistant (Build · GLM-5.1 · 6.6s)

**Tool: grep**

**Input:**
```json
{
  "pattern": "asyncio\\.to_thread",
  "include": "*.py",
  "path": "/home/models/src/LMCache/lmcache/v1/storage_backend/job_executor"
}
```

**Output:**
```
Found 1 matches
/home/models/src/LMCache/lmcache/v1/storage_backend/job_executor/pq_executor.py:
  Line 184:                 result = await asyncio.to_thread(fn, *args, **kwargs)

```

**Tool: bash**

**Input:**
```json
{
  "command": "python3 -c \"import asyncio; print(asyncio.events._DEFAULT_EXECUTOR_MAX_WORKERS if hasattr(asyncio.events, '_DEFAULT_EXECUTOR_MAX_WORKERS') else 'N/A')\" 2>/dev/null; python3 -c \"import os; print('min(32, os.cpu_count()+4) =', min(32, os.cpu_count()+4))\" 2>/dev/null"
}
```

**Output:**
```
N/A
min(32, os.cpu_count()+4) = 32

```

---

## Assistant (Build · GLM-5.1 · 7.4s)

Python 默认 `ThreadPoolExecutor` 大小为 **`min(32, os.cpu_count() + 4)`**。

在当前机器上 `os.cpu_count()` 很大，所以上限为 **32个OS线程**。

`asyncio.to_thread()` 使用的就是这个默认线程池——除非 `loop.set_default_executor()` 被调用覆盖，LMCache 代码中没有做这个覆盖，所以就是 **32线程**。

但注意：这32个线程是**所有使用 `asyncio.to_thread()` 的地方共享的**（LocalDiskBackend、InfinistoreConnector、MooncakestoreConnector、FSConnector 等），不是每个组件独占。而 ValkeyConnector、HF3fsConnector 等自建了独立线程池，不走这个默认池。

---

## User

4个协程，32个线程，但是最多也就4个worker同时执行？读写最多并发是4

---

## Assistant (Build · GLM-5.1 · 6.4s)

对。4个协程worker控制了**并发上限**，最多4个任务同时在"执行中"状态。32个默认线程池线程只是资源池，但受协程worker数限制，实际同时做I/O的只有4个。

所以 LocalDiskBackend 的读写并发上限就是 **4**。如果要调大并发，需要改 `AsyncPQThreadPoolExecutor` 的 `max_workers` 参数——目前是硬编码的（`local_disk_backend.py:48` 有 TODO 注释说要做成可配置的）。

---

## User

那为啥要用协程呢？我都有32个线程了

---

## Assistant (Build · GLM-5.1 · 7.8s)

因为需要**优先级调度**。

`asyncio.PriorityQueue` 是协程层面的优先级队列——32个线程池线程本身无法做优先级排序，它们只是无序地并行执行任务。

用4个协程worker从优先级队列取任务，保证 **prefetch(优先级0) 优先于 put(优先级2)** 执行。如果直接用32个线程并发取任务，低优先级的put可能占满线程池，高优先级的prefetch就被饿死了。

协程worker是**调度器**，线程池是**执行器**。调度器控制"谁先跑"，执行器提供"跑的场地"。4个并发上限确保优先级逻辑生效，32个线程池确保有足够OS线程承接被调度出的同步I/O。

---

## User

读操作优先级是多少？prefetch又是什么操作

---

## Assistant (Build · GLM-5.1 · 3.8s)

**Tool: grep**

**Input:**
```json
{
  "pattern": "priority|PREFETCH|prefetch",
  "include": "*.py",
  "path": "/home/models/src/LMCache/lmcache/v1/storage_backend"
}
```

**Output:**
```
Found 100 matches (more matches available)
/home/models/src/LMCache/lmcache/v1/storage_backend/connector/valkey_connector.py:
  Line 21: - Priority scheduling via ``AsyncPQExecutor`` (PEEK > PREFETCH > GET > PUT)

  Line 23:   matching the priority scheme used by ``RESPConnector``.

  Line 60:     Lower numeric value = higher priority.  Matches the scheme used by

  Line 65:     PREFETCH = auto()

  Line 263:     per-thread clients, ThreadPoolExecutor, and priority scheduling.

  Line 269:     Operations are dispatched through an ``AsyncPQExecutor`` with priority

  Line 270:     levels (PEEK > PREFETCH > GET > PUT) so that latency-sensitive lookups

  Line 337:             self._exists, key=key, priority=Priorities.PEEK

  Line 379:             self._get, key=key, priority=Priorities.GET

  Line 399:             self._put, key=key, memory_obj=memory_obj, priority=Priorities.PUT

  Line 430:             priority=Priorities.PUT,

  Line 516:             self._batched_get, keys=keys, priority=Priorities.GET

  Line 573:             priority=Priorities.PREFETCH,

  Line 616:             priority=Priorities.PREFETCH,


/home/models/src/LMCache/lmcache/v1/storage_backend/maru_backend.py:
  Line 524:     # Async lookup API (used by StorageManager.async_lookup_and_prefetch)


/home/models/src/LMCache/lmcache/v1/storage_backend/job_executor/base_executor.py:
  Line 20:         :param kwargs: The keyword arguments to pass to the function (e.g., priority).


/home/models/src/LMCache/lmcache/v1/storage_backend/job_executor/pq_executor.py:
  Line 16:     """Execute async functions with a priority queue and using event loop"""

  Line 21:         # (priority, time_id, fn, args, kwargs, done_future)

  Line 32:         # counter breaks ties for equal priority items (FCFS)

  Line 49:         # Assign highest priority by default

  Line 50:         priority = kwargs.pop("priority", 0)

  Line 52:         await self._queue.put((priority, next(self._counter), fn, args, kwargs, done))

  Line 99:         # Enqueue comparable sentinel tuples with the highest priority value so

  Line 101:         # Use a very large integer to satisfy the typed queue's expected int priority

  Line 102:         sentinel_priority = 2**31 - 1

  Line 107:                 (sentinel_priority, next(self._counter), _SENTINEL, None, {}, None)

  Line 134:     """Execute sync functions with a priority queue and using threadpool"""

  Line 149:         # counter breaks ties for equal priority items (FCFS)

  Line 202:         # Enqueue comparable sentinel tuples with the highest priority value so

  Line 204:         # Use a very large integer to satisfy the typed queue's expected int priority

  Line 205:         sentinel_priority = 2**31 - 1

  Line 208:                 (sentinel_priority, next(self._counter), _SENTINEL, None, {}, None)


/home/models/src/LMCache/lmcache/v1/storage_backend/connector/sagemaker_hyperpod_connector.py:
  Line 40:     """Priority levels for job execution in the priority queue."""

  Line 42:     LEASE = 0  # Highest priority - lease acquisition/release

  Line 43:     PREFETCH = auto()  # Medium priority - prefetching data

  Line 44:     PUT = auto()  # Lower priority - storing data

  Line 410:         """Submit lease acquisition to priority executor."""

  Line 414:             priority=Priorities.LEASE,

  Line 744:         """Store data to ai-toolkit (queued with priority)."""

  Line 749:             priority=Priorities.PUT,

  Line 898:         """Check existence of multiple keys (queued with priority)."""

  Line 904:             priority=Priorities.LEASE,

  Line 921:         """Prefetch multiple keys (queued with priority)."""

  Line 926:             priority=Priorities.PREFETCH,

  Line 962:         # Shutdown priority queue executor


/home/models/src/LMCache/lmcache/v1/storage_backend/gds_backend.py:
  Line 717:     def submit_prefetch_task(

  Line 737:         # TODO(Jiayi): Need to modify this when prefetch interface is determined.

  Line 865:         # TODO: Using a dummy wrapper around prefetch for now.

  Line 866:         if not self.submit_prefetch_task(key):


/home/models/src/LMCache/lmcache/v1/storage_backend/connector/mock_connector.py:
  Line 24:     PREFETCH = auto()

  Line 211:             self._exists, key=key, priority=Priorities.PEEK

  Line 237:             self._get, key=key, priority=Priorities.GET

  Line 247:             self._put, key=key, memory_obj=memory_obj, priority=Priorities.PUT

  Line 291:             self._batched_get, keys=keys, priority=Priorities.GET

  Line 322:             priority=Priorities.PEEK,

  Line 336:             self._batched_get, keys=keys, priority=Priorities.PREFETCH


/home/models/src/LMCache/lmcache/v1/storage_backend/connector/redis_connector.py:
  Line 33:     PREFETCH = auto()

  Line 70:             self._exists, key=key, priority=Priorities.PEEK

  Line 93:             self._get, key=key, priority=Priorities.GET

  Line 105:             self._put, key=key, memory_obj=memory_obj, priority=Priorities.PUT

  Line 130:             priority=Priorities.PUT,

  Line 161:             self._batched_get, keys=keys, priority=Priorities.GET

  Line 209:         # prefetch priority

  Line 211:             self._batched_async_contains, keys=keys, priority=Priorities.PREFETCH

  Line 222:         # prefetch priority

  Line 224:             self._batched_get, keys=keys, priority=Priorities.PREFETCH

  Line 271:             self._exists, key=key, priority=Priorities.PEEK

  Line 336:             self._get, key=key, priority=Priorities.GET

  Line 360:             priority=Priorities.PUT,

  Line 383:             self._put, key=key, memory_obj=memory_obj, priority=Priorities.PUT

  Line 424:             priority=Priorities.PEEK,

  Line 448:             priority=Priorities.PREFETCH,

  Line 649:             self._exists, key=key, priority=Priorities.PEEK

  Line 715:             self._get, key=key, priority=Priorities.GET

  Line 739:             priority=Priorities.PUT,

  Line 762:             self._put, key=key, memory_obj=memory_obj, priority=Priorities.PUT

  Line 803:             priority=Priorities.PEEK,

  Line 827:             priority=Priorities.PREFETCH,


/home/models/src/LMCache/lmcache/v1/storage_backend/connector/eic_connector.py:
  Line 30:     PREFETCH = auto()

  Line 289:             self._exists, key=key, priority=Priorities.PEEK

  Line 431:             self._get, key=key, priority=Priorities.GET

  Line 618:             self._put, key=key, memory_obj=memory_obj, priority=Priorities.PUT

  Line 635:             priority=Priorities.PUT,

  Line 693:             priority=Priorities.PEEK,

  Line 716:             priority=Priorities.GET,

  Line 744:             priority=Priorities.PREFETCH,


/home/models/src/LMCache/lmcache/v1/storage_backend/connector/s3_connector.py:
  Line 27:     PREFETCH = auto()

  Line 585:             priority=Priorities.PUT,

  Line 627:             priority=Priorities.PEEK,

  Line 649:             priority=Priorities.PREFETCH,


/home/models/src/LMCache/lmcache/v1/storage_backend/local_disk_backend.py:
  Line 38: # TODO(Jiayi): handle cases where cache is repetitvely prefetched.

  Line 44:         self.prefetch_lock = threading.Lock()

  Line 45:         self.prefetch_tasks: dict[CacheEngineKey, Future] = {}

  Line 59:         if task_type == "prefetch":

  Line 60:             priority = 0

  Line 61:             # self.insert_prefetch_task(kwargs["key"], None)

  Line 63:             priority = 1

  Line 65:             priority = 2

  Line 72:             priority=priority,

  Line 482:             # NOTE(Jiayi): Currently, we consider prefetch as cache hit.


(Results truncated. Consider using a more specific path or pattern.)
```

---

## Assistant (Build · GLM-5.1 · 6.0s)

**Tool: read**

**Input:**
```json
{
  "filePath": "/home/models/src/LMCache/lmcache/v1/storage_backend/local_disk_backend.py",
  "offset": 38,
  "limit": 40
}
```

**Output:**
```
<path>/home/models/src/LMCache/lmcache/v1/storage_backend/local_disk_backend.py</path>
<type>file</type>
<content>
38: # TODO(Jiayi): handle cases where cache is repetitvely prefetched.
39: class LocalDiskWorker:
40:     def __init__(self, loop: asyncio.AbstractEventLoop) -> None:
41:         self.put_lock = threading.Lock()
42:         self.put_tasks: List[CacheEngineKey] = []
43: 
44:         self.prefetch_lock = threading.Lock()
45:         self.prefetch_tasks: dict[CacheEngineKey, Future] = {}
46: 
47:         # TODO(Jiayi): make executor and its parameters configurable
48:         self.executor = AsyncPQThreadPoolExecutor(loop, max_workers=4)
49:         self.loop = loop
50:         self._closed = False
51: 
52:     async def submit_task(
53:         self,
54:         task_type: str,
55:         task: Callable,
56:         *args,
57:         **kwargs,
58:     ) -> Any:
59:         if task_type == "prefetch":
60:             priority = 0
61:             # self.insert_prefetch_task(kwargs["key"], None)
62:         elif task_type == "delete":
63:             priority = 1
64:         elif task_type == "put":
65:             priority = 2
66:         else:
67:             raise ValueError(f"Unknown task type: {task_type}")
68: 
69:         return await self.executor.submit_job(
70:             task,
71:             *args,
72:             priority=priority,
73:             **kwargs,
74:         )
75: 
76:     def remove_put_task(self, key: CacheEngineKey):
77:         with self.put_lock:

(Showing lines 38-77 of 673. Use offset=78 to continue.)
</content>
```

**Tool: read**

**Input:**
```json
{
  "filePath": "/home/models/src/LMCache/lmcache/v1/storage_backend/connector/redis_connector.py",
  "offset": 30,
  "limit": 10
}
```

**Output:**
```
<path>/home/models/src/LMCache/lmcache/v1/storage_backend/connector/redis_connector.py</path>
<type>file</type>
<content>
30: 
31: class Priorities(IntEnum):
32:     PEEK = auto()
33:     PREFETCH = auto()
34:     GET = auto()
35:     PUT = auto()
36: 
37: 
38: class RESPConnector(RemoteConnector):
39:     """

(Showing lines 30-39 of 828. Use offset=40 to continue.)
</content>
```

---

## Assistant (Build · GLM-5.1 · 9.2s)

优先级数值（越小越高）：

| 操作 | 优先级值 | 含义 |
|------|---------|------|
| PEEK | 1 | 检查key是否存在（exists） |
| PREFETCH | 2 | 预取——提前从远端把数据拉到本地 |
| GET | 3 | 同步读取 |
| PUT | 4 | 写入 |

LocalDiskBackend 的优先级略有不同：prefetch=0, delete=1, put=2，没有PEEK/GET区分。

**prefetch** 是"预取"操作——当 StorageManager 在 `contains()` 中发现数据在远端（如 RemoteBackend）存在时，不等用户主动请求读取，就**提前把数据从远端拉到本地磁盘**。这样后续 GET 时数据已在本地，减少延迟。prefetch 优先级高于 GET，是因为预取是**投机性的优化操作**，如果延迟执行就失去了预取的意义。

---

## User

如果是协程的话，那这里这里实际上并发的IO数，可以超过32线程，比如64 worker 可以同时提交到线程池中

---

## Assistant (Build · GLM-5.1 · 8.5s)

不完全是。`max_workers=64` 会让64个协程worker同时从队列取任务，但 `asyncio.to_thread()` 最终都提交到**同一个默认ThreadPoolExecutor（32线程）**。

所以64个worker同时提交64个任务到线程池时：
- **调度层面**：64个协程worker都在 `await asyncio.to_thread(...)` 状态
- **执行层面**：只有32个OS线程在真正做I/O，剩下32个任务在ThreadPoolExecutor的内部队列排队等待线程

实际上并发IO数**不会超过32**，超过的部分只是在协程层面"等待线程池分配线程"，不是真正的并发I/O。

所以4个worker × 32线程池 = 最多4个并发I/O，是合理的。64个worker也只是让32个线程更忙，不会产生更多并发。如果真想增大并发IO，应该直接用独立的 `ThreadPoolExecutor(max_workers=N)` 替代 `asyncio.to_thread()`，而不是加协程worker。

---

## User

那这里的协程不像是真正的协程

---

## Assistant (Build · GLM-5.1 · 10.7s)

对。这里的协程worker**不干任何实际计算**，只是从优先级队列取任务、然后 `await asyncio.to_thread()` 把任务交给线程池、等待结果回来。

它本质上是一个**带优先级的任务分发器**，不是真正的协程（做IO、做计算）。真正的IO都在OS线程里完成。

所以更准确的理解是：

```
优先级分发器（4个协程worker） → ThreadPoolExecutor（32个OS线程做实际IO）
```

协程在这里的唯一价值就是**让 `asyncio.PriorityQueue` 能生效**——因为只有协程才能 `await` 队列的 `get()`，实现优先级排序的等待-取出-调度流程。如果用纯线程池，`PriorityQueue` 就没法用 `await` 语义，优先级调度就做不了。

---

