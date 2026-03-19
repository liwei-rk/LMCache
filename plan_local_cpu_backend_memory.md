# 计划: 分析 local_cpu_backend 内存分配与初始化机制

## 目标
深入理解 LMCache 中 `local_cpu_backend` 使用的内存是如何分配并初始化的，以及如何获取内存指针用于 RDMA。

## 内存分配流程

### 1. LocalCPUBackend 初始化入口
- **文件**: `lmcache/v1/storage_backend/local_cpu_backend.py`
- **类**: `LocalCPUBackend` (第36行)
- 在 `__init__` 方法中 (第43-86行) 调用 `initialize_allocator` 方法

### 2. 内存分配器初始化
- **方法**: `LocalCPUBackend.initialize_allocator` (第253-305行)
- 根据配置 `config.enable_p2p` 选择不同的分配器:
  - **P2P模式** (`enable_p2p=True`): 使用 `PagedCpuGpuMemoryAllocator`
  - **普通模式** (`enable_p2p=False`): 使用 `MixedMemoryAllocator`

### 3. MixedMemoryAllocator 内存分配 (enable_p2p=False 模式)
- **文件**: `lmcache/v1/memory_management.py`
- **类**: `MixedMemoryAllocator` (第1491行)
- **初始化流程** (第1497-1532行):
  - 调用 `_allocate_cpu_memory(size, numa_mapping)` 分配 pinned memory 缓冲区
  - 保存为 `self.buffer` 属性
  - 创建 `TensorMemoryAllocator` 或 `PagedTensorMemoryAllocator`

### 4. 底层 CPU 内存分配函数
- **函数**: `_allocate_cpu_memory` (第300-322行)
- **分配机制**:
  - 有 NUMA mapping: 使用 `lmc_ops.alloc_pinned_numa_ptr(size, numa_id)` 进行 NUMA 感知分配
  - 无 NUMA mapping: 使用 `lmc_ops.alloc_pinned_ptr(size, 0)` 分配 pinned memory
- **返回**: torch.Tensor (uint8 类型，视图)

### 5. 底层 C++ 实现 (csrc/mem_alloc.cpp)
- **alloc_pinned_ptr** (第12-19行):
  - 使用 CUDA API `cudaHostAlloc` 分配页锁定(pinned)内存
- **alloc_pinned_numa_ptr** (第43-69行):
  - 使用 `mmap` + `mbind` 系统调用分配 NUMA 感知的内存
  - 通过 `first_touch` 函数初始化内存页
  - 使用 `cudaHostRegister` 注册为 pinned memory

---

## 获取分配的内存接口

### 接口 1: LocalCPUBackend.get_memory_allocator()

**文件**: `local_cpu_backend.py` 第594-595行

```python
def get_memory_allocator(self):
    return self.memory_allocator
```

使用方式:
```python
cpu_backend = LocalCPUBackend(config, metadata)
allocator = cpu_backend.get_memory_allocator()

# 获取内存指针
buffer_ptr = allocator.buffer.data_ptr()  # C++ 内存指针 (uintptr_t)
buffer_tensor = allocator.buffer          # torch.Tensor (uint8)
buffer_size = allocator.size             # 缓冲区大小 (字节)
```

### 接口 2: MixedMemoryAllocator.buffer 属性

**文件**: `memory_management.py` 第1506行

```python
self.buffer = _allocate_cpu_memory(size, self.numa_mapping)
```

`_allocate_cpu_memory` 返回 `torch.Tensor`，通过 `data_ptr()` 获取底层 C++ 指针:

```python
# memory_management.py 第318-320行
array_type = ctypes.c_uint8 * size
buf = array_type.from_address(ptr)
buffer = torch.frombuffer(buf, dtype=torch.uint8)
# buffer.data_ptr() 返回原始 C++ 指针
```

---

## 关键属性说明

| 属性/方法 | 说明 |
|-----------|------|
| `memory_allocator.buffer` | torch.Tensor，整个预分配的 pinned 内存缓冲区 |
| `memory_allocator.buffer.data_ptr()` | 获取底层 C++ 内存指针 (uintptr_t) |
| `memory_allocator.size` | 缓冲区总大小 (字节) |
| `memory_obj.raw_data` | 分配的内存切片 (torch.Tensor) |
| `memory_obj.raw_data.data_ptr()` | 分配块的起始指针 |
| `memory_obj.get_size()` | 分配块的大小 (字节) |

---

## 核心组件关系图

```
LocalCPUBackend.__init__()
    └── initialize_allocator()
            ├── enable_p2p=True  → PagedCpuGpuMemoryAllocator
            │                           └── init_cpu_memory_allocator()
            └── enable_p2p=False → MixedMemoryAllocator
                                        ├── self.buffer = _allocate_cpu_memory()
                                        │       └── lmc_ops.alloc_pinned_ptr()  → cudaHostAlloc
                                        └── self.pin_allocator = TensorMemoryAllocator()
                                                └── allocate() → TensorMemoryObj.raw_data

RDMA 使用:
    ├── 整个缓冲区: memory_allocator.buffer.data_ptr()
    └── 单个分配: memory_obj.raw_data.data_ptr()
```

---

## 待执行任务
- [x] 1. 阅读 LocalCPUBackend 类完整实现
- [x] 2. 阅读 MixedMemoryAllocator 类完整实现  
- [x] 3. 确认底层 lmc_ops 函数调用 (csrc/mem_alloc.cpp)
- [x] 4. 分析如何获取内存指针用于 RDMA
