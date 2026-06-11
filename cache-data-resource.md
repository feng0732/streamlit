# Streamlit cache_data 与 cache_resource 代码协作深度解析

## 一、整体架构概览

Streamlit 提供两种缓存装饰器，共享同一套核心缓存框架但在存储层和对象语义上有本质区别：

```
                     ┌──────────────────────────────────┐
                     │    CachedFunc (公共调用包装层)     │
                     │  - __call__ 统一入口               │
                     │  - _get_or_create_cached_value    │
                     │  - _handle_cache_hit / _miss      │
                     └───────────────┬──────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
          ┌─────────▼──────┐ ┌──────▼──────────┐     │
          │ CachedFuncInfo │ │ Cache (抽象基类) │     │
          │  (函数元信息)   │ │  - read_result  │     │
          └────────┬───────┘ │  - write_result │     │
                   │         └───────┬─────────┘     │
                   │                 │               │
          ┌────────▼────────┐ ┌─────▼──────────┐   ┌▼───────────────┐
          │CachedDataFunc   │ │  DataCache     │   │ ResourceCache  │
          │CachedResourceFnc│ │  (pickle存储)  │   │ (内存TTLCache)  │
          └─────────────────┘ └────┬───────────┘   └┬───────────────┘
                                   │                │
                          ┌────────▼────────┐  ┌───▼──────────────┐
                          │ CacheStorage    │  │ TTLCleanupCache  │
                          │ (协议层可插拔)   │  │ (LRU + TTL + on_release)
                          └────────┬────────┘  └──────────────────┘
                                   │
                          ┌────────▼────────┐
                          │InMemoryCacheWrpr│  ← 内存 + 持久化双层
                          │LocalDiskStorage  │
                          └─────────────────┘
```

核心源码文件：
- 公共调用层：[cache_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_utils.py)
- cache_data 实现：[cache_data_api.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_data_api.py)
- cache_resource 实现：[cache_resource_api.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_resource_api.py)
- TTL 清理缓存：[ttl_cleanup_cache.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/ttl_cleanup_cache.py)
- 消息重放：[cached_message_replay.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cached_message_replay.py)
- 存储协议：[cache_storage_protocol.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/cache_storage_protocol.py)

---

## 二、缓存 Key 计算机制（命中判定的前提）

### 2.1 两级 Key 体系

Streamlit 使用**两级 Key** 来唯一标识缓存条目：

| Key 层级 | 计算函数 | 作用 | 变化触发条件 |
|---------|---------|------|-------------|
| **function_key** | `_make_function_key()` | 标识一个被装饰函数的整个缓存空间 | 函数源码、模块名、限定名变化 |
| **value_key** | `_make_value_key()` | 标识函数某组参数的具体结果 | 参数值变化 |

### 2.2 function_key 计算详解

源码位置：[cache_utils.py#L540-L578](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_utils.py#L540-L578)

```python
def _make_function_key(cache_type: CacheType, func: Callable) -> str:
    func_hasher = util.create_fast_hasher()
    
    # 1. 包含模块名 + 限定名（支持嵌套函数但会有哈希共享问题）
    update_hash((func.__module__, func.__qualname__), ...)
    
    # 2. 包含源码，取不到则回退到字节码
    try:
        source_code = inspect.getsource(func)
    except (OSError, TypeError):
        source_code = func.__code__.co_code
    update_hash(source_code, ...)
    
    return func_hasher.hexdigest()
```

**设计要点**：
- 同一模块下两个源码相同的函数会产生**不同**的 function_key（因为 `__qualname__` 不同）
- 嵌套函数在不同运行中**可能共享**缓存（见 issue #11157）
- 源码微小变化（如添加注释）会导致整个缓存空间重建

### 2.3 value_key 计算详解

源码位置：[cache_utils.py#L473-L537](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_utils.py#L473-L537)

```python
def _make_value_key(cache_type, func, func_args, func_kwargs, hash_funcs) -> str:
    arg_pairs = []
    # 位置参数 → (参数名, 参数值)，*args/**kwargs 的位置参数名为 None
    for arg_idx in range(len(func_args)):
        arg_name = _get_positional_arg_name(func, arg_idx)
        arg_pairs.append((arg_name, func_args[arg_idx]))
    for kw_name, kw_val in func_kwargs.items():
        arg_pairs.append((kw_name, kw_val))
    
    args_hasher = util.create_fast_hasher()
    for arg_name, arg_value in arg_pairs:
        # 下划线前缀参数跳过哈希：_conn, _db 等
        if arg_name is not None and arg_name.startswith("_"):
            continue
        
        # 参数名哈希（不使用用户 hash_funcs）
        update_hash(arg_name, hasher=args_hasher, ...)
        # 参数值哈希（评估用户自定义 hash_funcs）
        update_hash(arg_value, hasher=args_hasher, hash_funcs=hash_funcs, ...)
    
    return args_hasher.hexdigest()
```

**设计要点**：
- `_` 前缀参数**完全不参与** value_key 计算（是排除不可哈希对象的逃生舱）
- 参数名和参数值**分别**调用两次 `update_hash`，确保 `foo(a=1, b=2)` ≠ `foo(b=1, a=2)`
- 位置参数的参数名解析：通过 `inspect.Parameter` 映射位置索引到形参名，*args/**kwargs 的位置参数名为 None 但仍会被哈希

---

## 三、缓存命中判定流程

### 3.1 调用链总览

每次调用被装饰函数的完整流程（`CachedFunc.__call__` → `_get_or_create_cached_value`）：

源码位置：[cache_utils.py#L268-L327](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_utils.py#L268-L327)

```
CachedFunc(*args, **kwargs)
    │
    ├─► 构建 spinner 消息（仅最外层缓存函数显示，避免嵌套缓存的 proto 风暴）
    │
    └─► _get_or_create_cached_value(args, kwargs, spinner_msg)
            │
            ├─► 1. 实时获取 function_cache（因为参数可能变化导致重建）
            │     cache = info.get_function_cache(function_key)
            │
            ├─► 2. 计算 value_key
            │     value_key = _make_value_key(...)
            │
            ├─► 3. 第一次无锁尝试读取（快速路径）
            │     try:
            │         cached_result = cache.read_result(value_key)
            │         return _handle_cache_hit(cached_result)  ← 命中！
            │     except CacheKeyNotFoundError:
            │         pass  ← 未命中，进入慢路径
            │
            └─► 4. 显示 spinner + 进入 _handle_cache_miss
```

### 3.2 Cache Miss 的双重检查锁（Double-Checked Locking）

源码位置：[cache_utils.py#L339-L412](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_utils.py#L339-L412)

```python
def _handle_cache_miss(cache, value_key, func_args, func_kwargs):
    with cache.compute_value_lock(value_key):  # 按 value_key 粒度加锁
        # 第二次读取：另一个线程可能已经算完了
        try:
            cached_result = cache.read_result(value_key)
            return _handle_cache_hit(cached_result)  # 锁后再命中
        except CacheKeyNotFoundError:
            pass
        
        # 进入消息捕获上下文，记录期间所有 st 调用
        with info.cached_message_replay_ctx.calling_cached_function(func):
            computed_value = func(*func_args, **func_kwargs)
        
        # 提取期间捕获的 st 消息列表
        messages = info.cached_message_replay_ctx._most_recent_messages
        
        # 写入缓存（value + messages + main_id + sidebar_id）
        cache.write_result(value_key, computed_value, messages)
        return computed_value
```

**锁的粒度设计**：
- `Cache._value_locks: dict[str, Lock]` — 每个 value_key 独立一把锁
- 用 `_value_locks_lock` 保护这个 dict 的创建
- 好处：不同参数的计算互不阻塞，只有相同参数的并发调用才会排队

### 3.3 Cache Hit 的消息重放

源码位置：[cached_message_replay.py#L245-L290](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cached_message_replay.py#L245-L290)

命中后不仅返回 value，还会重放缓存的所有 `st.*` 调用：

```python
def replay_cached_messages(result, cache_type, cached_func):
    # result.main_id / sidebar_id 记录了当时 main/sidebar 的 DG id
    # 将旧 id 映射到本次 script run 的 DG 实例
    returned_dgs = {
        result.main_id: st._main,
        result.sidebar_id: st.sidebar,
    }
    
    for msg in result.messages:  # 按原始调用顺序重放
        if isinstance(msg, ElementMsgData):
            dg = returned_dgs[msg.id_of_dg_called_on]
            maybe_dg = dg._enqueue(msg.delta_type, msg.message, ...)
            if isinstance(maybe_dg, DeltaGenerator):
                returned_dgs[msg.returned_dgs_id] = maybe_dg  # 追踪新产生的 DG
        elif isinstance(msg, BlockMsgData):
            dg = returned_dgs[msg.id_of_dg_called_on]
            new_dg = dg._block(msg.message)
            returned_dgs[msg.returned_dgs_id] = new_dg
```

**消息捕获机制**（写入时）：
- `CachedMessageReplayContext` 是 `threading.local` 子类，线程安全
- 进入缓存函数前 `push` 一个空列表和空的 seen_dg_set
- 每个 `st.*` 调用通过 `save_element_message/save_block_message` 追加到栈顶的所有列表（支持嵌套缓存）
- 退出时 `pop` 得到本次函数的完整消息序列

**隔离边界**：
- `CacheReplayClosureError` — 如果重放时发现某个 `msg.id_of_dg_called_on` 不在本次运行的 DG 映射中，抛出闭包越界错误

---

## 四、缓存失效判定机制

### 4.1 两种失效维度对比

| 失效机制 | cache_data | cache_resource |
|---------|-----------|---------------|
| **TTL** | ✅ 可配置（存储层 + 内存包装层双重） | ✅ 可配置（TTLCache 原生） |
| **max_entries** | ✅ 可配置（LRU 淘汰） | ✅ 可配置（LRU 淘汰） |
| **函数源码变化** | ✅ 整个 function_cache 重建 | ✅ 整个 function_cache 重建 |
| **validate 函数** | ❌ 不支持 | ✅ 每次读取时调用 |
| **手动 clear()** | ✅ 全部/按参数 | ✅ 全部/按参数 |
| **session 断开** | scope="session" 时自动清理 | scope="session" 时自动清理 + on_release |
| **持久化重启** | persist="disk" 时保留 | ❌ 重启丢失 |

### 4.2 cache_data 的参数变更检测

源码位置：[cache_data_api.py#L174-L261](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_data_api.py#L174-L261)

```python
def get_cache(self, key, persist, max_entries, ttl, display_name, scope):
    session_id = None if scope == "global" else get_session_id_or_throw()
    
    with self._caches_lock:
        session_caches = self._function_caches.get(session_id)
        cache = session_caches.get(key) if session_caches else None
        
        # 只有 4 个参数完全一致才复用旧缓存
        if (cache is not None
            and cache.ttl_seconds == ttl_seconds
            and cache.max_entries == max_entries
            and cache.persist == persist):
            return cache
        
        # 参数变化：关闭旧存储 + 创建新缓存
        if cache is not None:
            cache.storage.close()
        cache = DataCache(key=key, storage=new_storage, ...)
        self._function_caches[session_id][key] = cache
        return cache
```

**缓存变更会触发重建的 3 个参数**：
1. `ttl`（转换为秒数后比较）
2. `max_entries`
3. `persist`

注意：`show_spinner`、`hash_funcs` 变化**不会**触发缓存重建，但 `hash_funcs` 会影响 value_key 计算（间接导致不同 key）。

### 4.3 cache_resource 的参数变更检测

源码位置：[cache_resource_api.py#L103-L163](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_resource_api.py#L103-L163)

```python
def get_cache(self, key, display_name, max_entries, ttl, validate, on_release, scope):
    # 相比 cache_data，多出 validate 函数的比较
    if (cache is not None
        and cache.ttl_seconds == ttl_seconds
        and cache.max_entries == max_entries
        and _equal_validate_funcs(cache.validate, validate)):
        return cache
```

`_equal_validate_funcs` 的实现很有意思：
```python
def _equal_validate_funcs(a, b):
    # 不比较字节码，只判断"是否同时为 None 或同时非 None"
    return (a is None and b is None) or (a is not None and b is not None)
```

**设计权衡**：为了性能，不做函数字节码对比。这意味着修改 validate 函数的逻辑但保留非 None，缓存**不会重建**，但每次读取会调用新的 validate。

### 4.4 TTL 失效的实现差异

**cache_resource（TTLCleanupCache）**：
- 基于 `cachetools.TTLCache`，访问时惰性过期（get/set 时检查）
- `expire()` 返回过期列表，逐个触发 `on_release` 钩子
- 源码：[ttl_cleanup_cache.py#L71-L76](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/ttl_cleanup_cache.py#L71-L76)

**cache_data（InMemoryCacheStorageWrapper）**：
- 内存层同样是 `TTLCache`，支持 TTL + LRU
- 持久化层（如 LocalDisk）依赖文件系统 mtime/单独的 TTL 索引文件
- 读取顺序：先查内存 → 未命中查磁盘 → 从磁盘加载后回填内存

### 4.5 validate 钩子（仅 cache_resource）

源码位置：[cache_resource_api.py#L679-L695](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_resource_api.py#L679-L695)

```python
def read_result(self, key):
    with self._mem_cache_lock:
        if key not in self._mem_cache:
            raise CacheKeyNotFoundError()
        
        result = self._mem_cache[key]
        
        # validate 失败：删除 + 抛错（上层走 miss 流程重算）
        if self.validate is not None and not self.validate(result.value):
            del self._mem_cache[key]  # 注意：这里用 del 不会触发 on_release
            raise CacheKeyNotFoundError()
        
        return result
```

**重要细节**：validate 失败的删除用的是 `del`，而不是 `safe_del`，所以不会触发 `on_release`。如果需要在 validate 失败时清理资源，需要在 validate 函数内部自行处理。

### 4.6 on_release 钩子触发时机

| 触发方式 | 是否调用 on_release | 说明 |
|---------|-------------------|------|
| LRU 淘汰（超 max_entries） | ✅ | 通过 `popitem()` |
| TTL 过期（访问时触发） | ✅ | 通过 `expire()` 逐个调用 |
| `cache.clear()`（全部清理） | ✅ | 通过循环 `popitem()` |
| `cache.clear(key)`（单条删除） | ✅ | ResourceCache 用 `safe_del` |
| validate 失败 | ❌ | 直接 `del`，绕过钩子 |
| 缓存参数变更重建 | ❌ | DataCache 只 `storage.close()`，ResourceCache 直接替换 dict |
| session 断开清理 | ✅ | `clear_session()` 调用 `cache.clear()` → `popitem()` |

---

## 五、对象复用和隔离边界

### 5.1 语义核心差异：复制 vs 共享

| 维度 | cache_data | cache_resource |
|-----|-----------|---------------|
| **返回值语义** | 返回**深拷贝**（pickle 序列化/反序列化） | 返回**同一对象引用**（单例语义） |
| **可变性** | 调用者修改返回值**不影响**缓存原值 | 调用者修改返回值**直接影响**缓存 |
| **存储形态** | `pickle.dumps(entry)` → bytes → 存储层 | 内存中直接持有 `CachedResult(value, ...)` 对象 |
| **线程安全要求** | 低（返回拷贝，无共享可变状态） | 高（全局 scope 时多线程共享同一对象） |

**cache_data 的复制流程**（DataCache.read_result/write_result）：
```python
# 写入：pickle.dumps 序列化
def write_result(self, key, value, messages):
    entry = CachedResult(value, messages, main_id, sidebar_id)
    pickled_entry = pickle.dumps(entry)  # 深拷贝发生点
    self.storage.set(key, pickled_entry)

# 读取：pickle.loads 反序列化（产生新对象）
def read_result(self, key):
    pickled_entry = self.storage.get(key)
    entry = pickle.loads(pickled_entry)  # 新对象产生点
    return entry
```

### 5.2 Scope 隔离边界

两者都支持 `scope="global"`（默认）和 `scope="session"`：

```
DataCaches._function_caches / ResourceCaches._function_caches
    │
    ├─ Key: None (global scope)
    │     └─ { function_key_1: Cache, function_key_2: Cache, ... }
    │        ↑ 所有用户、所有会话共享
    │
    ├─ Key: "session_id_abc123" (session scope)
    │     └─ { function_key_1: Cache, ... }
    │        ↑ 仅该会话可见，断开时清理
    │
    └─ Key: "session_id_xyz789"
          └─ { ... }
```

**session 清理触发**：
- 调用点：`clear_session_cache(session_id)`，通常在会话断开时由 Runtime 调用
- 行为：从 `_function_caches` 中移除该 session_id → 逐个 `cache.clear()` → `storage.close()`

**get_session_id_or_throw 的保护**：
- 当 `scope="session"` 但不在 app 执行线程（如后台线程）访问时，抛出 `StreamlitAPIException`
- 防止会话级缓存被错误的上下文访问

### 5.3 线程安全设计

**每个缓存层级的锁保护**：

| 层级 | 锁机制 | 保护内容 |
|-----|--------|---------|
| `DataCaches._function_caches` | `_caches_lock` (Lock) | 缓存字典的新增/删除/重建 |
| `ResourceCaches._function_caches` | `_caches_lock` (Lock) | 同上 |
| `Cache._value_locks` | `_value_locks_lock` (Lock) | value_key 级锁字典的创建 |
| `Cache.compute_value_lock(key)` | 每个 key 独立 Lock | 相同参数的并发计算去重 |
| `ResourceCache._mem_cache` | `_mem_cache_lock` (Lock) | TTLCache 的读写 |
| `InMemoryCacheStorageWrapper._mem_cache` | `_mem_cache_lock` (Lock) | 内存层 TTLCache 的读写 |
| `CachedMessageReplayContext` | `threading.local` 基类 | 每个线程独立的消息栈 |

**并发场景示例（cache_resource, global scope）**：

```
线程A（用户1）: 调用 get_conn(url="db1")
    ├─ compute_value_lock("hash_db1") 获取锁
    ├─ 读缓存：miss
    └─ 实际创建数据库连接...

线程B（用户2）: 同时调用 get_conn(url="db1")
    ├─ compute_value_lock("hash_db1") 阻塞等待...
    └─ A释放后：二次检查 → 命中 → 返回同一连接对象引用  ← 对象复用！

线程C（用户3）: 调用 get_conn(url="db2")
    └─ compute_value_lock("hash_db2") 不同锁，完全不阻塞
```

### 5.4 嵌套缓存协作

源码位置：[cache_utils.py#L316-L324](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/cache_utils.py#L316-L324)

```python
# 判断是否在已嵌套的缓存函数中
is_nested_cache_function = in_cached_function.get()

# 内层缓存函数自动禁用 spinner，避免大量 proto 消息拖慢应用
spinner_or_no_context = (
    main_dg.spinner(...) if (spinner_message and not is_nested_cache_function)
    else contextlib.nullcontext()
)
```

`in_cached_function` 是一个 `contextvars.ContextVar`，在 `calling_cached_function` 上下文管理器中设置：
- 进入最外层缓存函数：设为 `True`，并记录 `nested_call=False`
- 进入内层缓存函数：`nested_call=True`
- 退出最外层：重置为 `False`
- 退出内层：不重置（外层仍需要 True 状态）

**消息栈的嵌套捕获**：
```python
# calling_cached_function 中：
self._cached_message_stack.append([])  # 每层 push 自己的消息列表
# 期间的 st 调用会追加到 stack 中**所有**列表（外层能看到内层消息）
# 退出时：
self._most_recent_messages = self._cached_message_stack.pop()  # 只取本层消息
```

---

## 六、cache_data 持久化缓存的失效边界深度解析

> 本章节深入 disk 模式下 TTL 为什么不生效、淘汰发生在哪一层、磁盘层和内存层各自负责什么。

### 6.1 三层存储架构全景

`@st.cache_data(persist="disk")` 实际上是**三层结构**，每层职责完全不同：

```
┌───────────────────────────────────────────────────────────┐
│                      DataCache                             │
│  （pickle 序列化/反序列化 + value 锁 + message 重放）      │
└─────────────────────┬─────────────────────────────────────┘
                      │
┌─────────────────────▼─────────────────────────────────────┐
│              InMemoryCacheStorageWrapper                  │
│  ┌────────────────────┐      ┌────────────────────────┐  │
│  │  内存层 (TTLCache) │◄────►│  持久化层               │  │
│  │  - TTL 过期 ✅     │      │  (LocalDiskCacheStorage)│  │
│  │  - LRU 淘汰 ✅     │      │  - 纯文件读写 ❌TTL     │  │
│  │  - max_entries ✅  │      │  - 无淘汰 ❌max_entries │  │
│  └────────────────────┘      └────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

各层源码位置：
- 包装层：[in_memory_cache_storage_wrapper.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/in_memory_cache_storage_wrapper.py)
- 磁盘层：[local_disk_cache_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/local_disk_cache_storage.py)
- 管理器配置：[cache_storage_manager_config.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/web/cache_storage_manager_config.py)

### 6.2 读路径：内存 miss → 磁盘命中 → 回填内存

源码位置：[in_memory_cache_storage_wrapper.py#L91-L101](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/in_memory_cache_storage_wrapper.py#L91-L101)

```python
def get(self, key):
    try:
        entry_bytes = self._read_from_mem_cache(key)  # 先查内存
    except CacheStorageKeyNotFoundError:
        entry_bytes = self._persist_storage.get(key)  # 内存 miss，查磁盘
        self._write_to_mem_cache(key, entry_bytes)   # 磁盘命中，回填内存
    return entry_bytes
```

**关键细节**：
- 磁盘层的 `get` 方法**没有任何 TTL 检查**，文件存在就直接读
- 磁盘层保存的 `_ttl_seconds` 和 `_max_entries` 属性**完全不使用**，只是摆设
- 回填到内存层时，内存层会按照自己的 TTL 重新计时（从回填时刻开始算 TTL）

### 6.3 写路径：内存+磁盘双写，不同步淘汰

源码位置：[in_memory_cache_storage_wrapper.py#L103-L106](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/in_memory_cache_storage_wrapper.py#L103-L106)

```python
def set(self, key, value):
    self._write_to_mem_cache(key, value)    # 写内存（受 LRU+TTL 约束）
    self._persist_storage.set(key, value)   # 写磁盘（无任何约束，永久保存）
```

**两层淘汰的不对称性**：

| 操作 | 内存层 (TTLCache) | 磁盘层 (LocalDisk) |
|-----|------------------|-------------------|
| **写入** | 受 max_entries 限制，超了 LRU 淘汰 | 无限制，直接写文件 |
| **TTL 过期** | 访问时惰性检查，过期直接删除 | 完全不检查，文件永久保留 |
| **LRU 淘汰** | 超 max_entries 时自动淘汰最旧 | 完全不淘汰，文件只增不减 |
| **手动 delete(key)** | 删除内存 key | **同时**删除磁盘文件 |
| **手动 clear()** | 清空内存 | **同时**清空磁盘所有该函数的文件 |

### 6.4 为什么 disk 模式下 TTL "不生效"

这是最容易混淆的点。**准确说法是：TTL 只在内存层生效，在磁盘层完全不生效。**

表现出来的现象：
1. 设置 `ttl=60` 后，60 秒内再次调用 → 内存命中（很快）
2. 60 秒后再次调用 → 内存 TTL 过期 → 去磁盘读 → 磁盘有文件 → 回填内存 → **返回旧数据**（用户以为 TTL 没生效）
3. 重启应用后调用 → 内存空 → 磁盘有文件 → 回填内存 → **返回很久之前的旧数据**

**官方其实有警告，但很容易被忽略**：

源码位置：[local_disk_cache_storage.py#L105-L115](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/local_disk_cache_storage.py#L105-L115)

```python
def check_context(self, context):
    if (context.persist == "disk"
        and context.ttl_seconds is not None
        and not math.isinf(context.ttl_seconds)):
        _LOGGER.warning(
            "The cached function '%s' has a TTL that will be ignored. "
            "Persistent cached functions currently don't support TTL.",
            context.function_display_name,
        )
```

这是 `check_context` 在装饰器创建时调用的，会在日志中打一条 warning，但不会抛异常，也不会阻止你设置 TTL。

### 6.5 max_entries 的真实作用范围

`max_entries` 同样只在内存层生效，磁盘层是无限增长的。

**一个反直觉的场景**：

```python
@st.cache_data(persist="disk", max_entries=3)
def load_data(id):
    # 假设 id 可以取 1~100 共 100 种
    ...
```

- 内存里最多保留 3 个结果，超过就 LRU 淘汰
- 磁盘上会有 100 个 `.memo` 文件，**越来越多，永不删除**
- 每次访问新 id 都会先查内存 → miss → 查磁盘 → 命中就回填内存
- 只有调用 `load_data.clear()` 才会一次性删除磁盘上所有该函数的文件

**磁盘文件的命名规则**：

源码位置：[local_disk_cache_storage.py#L208-L213](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/local_disk_cache_storage.py#L208-L213)

```python
def _get_cache_file_path(self, value_key):
    cache_dir = get_cache_folder_path()  # ~/.streamlit/cache/
    return os.path.join(
        cache_dir, f"{self.function_key}-{value_key}.memo"
    )
```

每个被缓存的 value 对应独立文件，文件名 = `{function_key}-{value_key}.memo`。

### 6.6 磁盘层的实际能力边界

LocalDiskCacheStorage 的 `get`/`set`/`delete`/`clear` 实现极其简单：

| 方法 | 实际行为 | 是否检查 TTL | 是否检查 max_entries |
|-----|---------|-------------|---------------------|
| `get(key)` | 读文件，不存在抛错 | ❌ | ❌ |
| `set(key,value)` | 写文件，失败清理空文件 | ❌ | ❌ |
| `delete(key)` | `os.remove` 文件，不存在静默 | ❌ | ❌ |
| `clear()` | 遍历目录删除所有以 function_key 开头的 .memo 文件 | ❌ | ❌ |
| `close()` | 空实现，什么也不做 | - | - |

源码为证：[local_disk_cache_storage.py#L137-L207](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/local_disk_cache_storage.py#L137-L207) 中完全没有 TTL 或 LRU 相关的逻辑代码。

### 6.7 非 persist 模式的存储层

当 `persist=None`（默认）时，使用的是 `MemoryCacheStorageManager`，其持久化层是 `DummyCacheStorage`：

源码位置：[dummy_cache_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/dummy_cache_storage.py)

```python
class DummyCacheStorage(CacheStorage):
    def get(self, key):
        raise CacheStorageKeyNotFoundError("Key not found in dummy cache")
    def set(self, key, value):
        pass
    def delete(self, key):
        pass
    def clear(self):
        pass
```

也就是说：
- 非 disk 模式下，`InMemoryCacheStorageWrapper` 仍然存在，但 `_persist_storage` 是个永远 miss 的空壳
- 此时 TTL 和 max_entries **完全生效**，因为只有内存层在工作
- 内存层就是真实的唯一存储，没有磁盘回源

### 6.8 持久化缓存的失效边界总结

```
┌────────────────────────────────────────────────────────────┐
│               失效 / 淘汰发生的层级                         │
├─────────────┬──────────────────┬───────────────────────────┤
│   机制       │  内存层           │  磁盘层                    │
├─────────────┼──────────────────┼───────────────────────────┤
│ TTL 过期     │  ✅ 访问时惰性检查 │  ❌ 永久保留，永不检查      │
│ LRU 淘汰     │  ✅ 超 max_entries  │  ❌ 只增不减，无限增长    │
│              │      自动淘汰最旧  │                           │
│ 手动 clear() │  ✅ 全部清空      │  ✅ 全部清空（遍历删除）    │
│ 单条 delete  │  ✅ 删除指定 key  │  ✅ 删除对应文件           │
│ 重启进程     │  ✅ 全部丢失      │  ✅ 全部保留（持久化本义）  │
│ 源码变更     │  新 function_key  │  新 function_key           │
│              │  → 新缓存空间     │  → 旧文件变孤儿，永不清理   │
└─────────────┴──────────────────┴───────────────────────────┘
```

**特别注意**：函数源码变更导致 function_key 变化后，磁盘上旧 function_key 对应的所有 `.memo` 文件会变成**孤儿文件**，永远不会被自动清理。只有手动调用 `st.cache_data.clear()`（会调用 `clear_all()` 直接 `shutil.rmtree` 整个 cache 目录）才会清理掉。

源码位置：[local_disk_cache_storage.py#L100-L103](file:///d:/fz/0601/solo-dogfeeding/code/216-streamlit/lib/streamlit/runtime/caching/storage/local_disk_cache_storage.py#L100-L103)

```python
def clear_all(self):
    cache_path = get_cache_folder_path()
    if os.path.isdir(cache_path):
        shutil.rmtree(cache_path)  # 暴力删除整个缓存目录
```

---

## 七、cache_data vs cache_resource 协作对比总结

### 7.1 决策速查表

| 使用场景 | 推荐装饰器 | 原因 |
|---------|-----------|------|
| DataFrame 转换、SQL 查询结果 | `@st.cache_data` | 返回拷贝，用户互不干扰；支持落盘持久化 |
| 数据库连接池、ML 模型 | `@st.cache_resource` | 单例复用，避免重复初始化；支持 on_release 清理 |
| 大语言模型实例 | `@st.cache_resource` | 重量级对象，不可 pickle；需要 validate 检查连接健康 |
| 用户专属数据 | `@st.cache_data(scope="session")` | 会话级隔离，断开自动清理 |
| 每个用户独立的数据库连接 | `@st.cache_resource(scope="session", on_release=...)` | 避免多用户共享连接状态，断开自动关闭 |
| 可变对象且调用者需要修改 | `@st.cache_data` | 返回副本，修改安全 |
| 不可序列化对象 | `@st.cache_resource` | 不经过 pickle，直接内存引用 |

### 7.2 存储架构差异图

```
cache_data:
┌─────────────────────────────────────────────────────┐
│                   DataCache                         │
│  ┌───────────────────────────────────────────────┐  │
│  │         InMemoryCacheStorageWrapper           │  │
│  │  ┌──────────────┐    ┌────────────────────┐  │  │
│  │  │  Mem Layer   │    │  Persist Layer      │  │  │
│  │  │ (TTLCache)   │───►│ (LocalDisk/Redis)   │  │  │
│  │  │ LRU+TTL淘汰  │    │ pickle bytes 文件   │  │  │
│  │  └──────┬───────┘    └────────────────────┘  │  │
│  └─────────┼─────────────────────────────────────┘  │
│            │ 每次 read/write 都 pickle 序列化       │
│            ▼                                        │
│    返回值 = 深拷贝（独立对象）                       │
└─────────────────────────────────────────────────────┘

cache_resource:
┌─────────────────────────────────────────────────────┐
│               ResourceCache                         │
│  ┌───────────────────────────────────────────────┐  │
│  │          TTLCleanupCache (内存 only)          │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │ {value_key: CachedResult(value, msgs)}  │  │  │
│  │  │  LRU + TTL 淘汰 → 触发 on_release       │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
│            │ 直接返回 .value 引用                   │
│            ▼                                        │
│    返回值 = 同一对象（所有调用者共享）                │
└─────────────────────────────────────────────────────┘
```

### 7.3 易混淆点澄清

1. **「hash_funcs 变化会重建缓存吗？」**  
   ❌ 不会。hash_funcs 只参与 value_key 计算，不参与 function_cache 参数比较。但会导致新的 value_key（如果 hash 结果变化），所以效果类似"部分失效"。

2. **「cache_resource 的 on_release 保证被调用吗？」**  
   ❌ 不保证。进程强杀时不会调用；validate 失败时也不会调用；缓存参数变更导致旧缓存丢弃时也不会调用。只适合做"尽力而为"的清理。

3. **「相同函数，先装 cache_data 再换 cache_resource，会复用缓存吗？」**  
   ❌ 不会。function_key 包含 `cache_type`（DATA vs RESOURCE），两者的命名空间完全独立。

4. **「scope="session" 的缓存，用户刷新页面会丢吗？」**  
   通常不会丢，刷新会复用同一个 session_id。但网络不稳定导致 WebSocket 重连可能会产生新 session，此时旧 session 的缓存会被清理。

5. **「嵌套缓存函数中，内层缓存 miss 但外层命中，内层的 st 消息会显示吗？」**  
   ✅ 会。因为缓存的是完整 CachedResult（包含所有消息），重放时按顺序全部重放，不管这些消息是本层还是内层函数产生的。

6. **「persist=\"disk\" 时，TTL 还有用吗？」**  
   ❌ 磁盘层完全没用。TTL 只在内存层（InMemoryCacheStorageWrapper 的 TTLCache）生效，磁盘上的文件永久保留永不检查 TTL。内存 TTL 过期后会从磁盘重新读回来（旧数据），给人的感觉就是 "TTL 没生效"。官方会在日志里打 warning 但不会报错。

7. **「persist=\"disk\" + max_entries，磁盘上的文件也会限制数量吗？」**  
   ❌ 不会。max_entries 只限制内存层，磁盘文件只增不减无限增长。只有调用 `clear()` 或 `st.cache_data.clear()` 才会删除磁盘文件。

8. **「函数源码改了，磁盘上的旧缓存文件会自动清理吗？」**  
   ❌ 不会。源码变更会产生新的 function_key，旧 function_key 对应的所有 .memo 文件变成孤儿文件永远留在磁盘上。只有 `st.cache_data.clear()`（它会 rmtree 整个 cache 目录）才能清理掉。

---

## 八、核心协作代码路径速查

| 功能 | cache_data | cache_resource |
|-----|-----------|---------------|
| **入口装饰器类** | `CacheDataAPI._decorator` | `CacheResourceAPI._decorator` |
| **函数信息类** | `CachedDataFuncInfo` | `CachedResourceFuncInfo` |
| **缓存管理器（单例）** | `_data_caches: DataCaches` | `_resource_caches: ResourceCaches` |
| **单函数缓存类** | `DataCache` | `ResourceCache` |
| **底层存储** | `CacheStorage`（可插拔协议） | `TTLCleanupCache`（内存专用） |
| **read_result 核心** | pickle.loads + 存储协议 get | TTLCache 查找 + validate 钩子 |
| **write_result 核心** | pickle.dumps + 存储协议 set | TTLCache 直接赋值 |
| **消息重放上下文** | `CACHE_DATA_MESSAGE_REPLAY_CTX` | `CACHE_RESOURCE_MESSAGE_REPLAY_CTX` |
| **额外特性** | persist 落盘、存储层可扩展 | validate 健康检查、on_release 清理 |
