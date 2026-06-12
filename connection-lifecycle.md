# Streamlit 数据库连接入口生命周期分析

## 概览

Streamlit 的数据库连接系统由三层协作构成：**配置解析层**（从 secrets.toml / kwargs / 环境变量读取连接参数）、**连接缓存层**（基于 `st.cache_resource` 的全局/会话级缓存）、**用户调用层**（`st.connection()` 工厂函数及连接对象上的操作方法）。三者之间的关系如下：

```
用户脚本 st.connection(name, type, ttl, ...)
        │
        ▼
┌─────────────────────────────────────────────────┐
│  connection_factory()                           │
│  1. 解析连接类型 (type 参数 / secrets / 内置映射) │
│  2. _create_connection() → cache_resource 包装   │
│  3. 返回缓存的连接实例或新建实例                   │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│  BaseConnection.__init__()                      │
│  1. 读取 _secrets (从 secrets.toml 的 connections 段)│
│  2. 计算 _config_section_hash (用于检测 secrets 变更)│
│  3. 注册 secrets 变更监听器                       │
│  4. 调用 _connect(**kwargs) 创建底层连接          │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│  具体连接类._connect()                           │
│  SQLConnection → SQLAlchemy Engine               │
│  SnowflakeConnection → snowflake.connector       │
│  SnowflakeCallersRightsConnection → OAuth token  │
│  SnowparkConnection → Snowpark Session           │
└─────────────────────────────────────────────────┘
```

---

## 一、配置解析层

### 1.1 连接类型的确定

[connection_factory](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/connection_factory.py#L244-L498) 是 `st.connection()` 的底层实现，它按优先级依次确定连接类型：

| 优先级 | 来源 | 示例 |
|--------|------|------|
| 1 | `type` 关键字参数（类引用） | `st.connection("x", type=SQLConnection)` |
| 2 | `type` 关键字参数（字符串） | `st.connection("x", type="sql")` 或 `type="streamlit.connections.SQLConnection"` |
| 3 | `name` 匹配内置连接名 | `st.connection("sql")` → 自动推断为 SQLConnection |
| 4 | `secrets.toml` 中的 `type` 字段 | `[connections.my_conn] type = "sql"` |

关键代码路径（[connection_factory.py#L436-L474](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/connection_factory.py#L436-L474)）：

```python
# 支持 env: 前缀从环境变量读取连接名
if name.startswith(_USE_ENV_PREFIX):
    envvar_name = name.removeprefix(_USE_ENV_PREFIX)
    name = os.environ[envvar_name]

connection_class = type
if connection_class is None:
    if name in _FIRST_PARTY_CONNECTIONS:
        connection_class = _get_first_party_connection(name)
    else:
        # 必须在 secrets.toml 中定义 type
        secrets_singleton.load_if_toml_exists()
        connection_class = secrets_singleton["connections"][name]["type"]

if isinstance(connection_class, str):
    if "." in connection_class:
        # 动态导入: "streamlit.connections.SQLConnection"
        parts = connection_class.split(".")
        classname = parts.pop()
        connection_module = importlib.import_module(".".join(parts))
        connection_class = getattr(connection_module, classname)
    else:
        connection_class = _get_first_party_connection(connection_class)
```

内置连接类型映射（[connection_factory.py#L42-L47](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/connection_factory.py#L42-L47)）：

```python
_FIRST_PARTY_CONNECTIONS = {
    "snowflake": SnowflakeConnection,
    "snowflake-callers-rights": SnowflakeCallersRightsConnection,
    "snowpark": SnowparkConnection,
    "sql": SQLConnection,
}
```

### 1.2 连接参数的来源与合并

各连接类的 `_connect()` 方法负责合并配置参数。参数来源有三种，合并优先级为 **kwargs > secrets.toml > 其他配置文件**。

#### SQLConnection 的参数合并

[SQLConnection._connect()](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/sql_connection.py#L180-L222) 使用 `ChainMap` 实现参数合并：

```python
conn_param_kwargs = extract_from_dict(_ALL_CONNECTION_PARAMS, kwargs)
conn_params = ChainMap(conn_param_kwargs, self._secrets.to_dict())
# ChainMap: kwargs 覆盖 secrets
```

支持的参数集合（[sql_connection.py#L41-L51](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/sql_connection.py#L41-L51)）：`url`, `driver`, `dialect`, `username`, `password`, `host`, `port`, `database`, `query`。

两种 URL 构建方式：
- 有 `url` → 直接 `make_url(url)`
- 无 `url` → 必须提供 `dialect`, `username`, `host`，通过 `URL.create()` 构建

`create_engine_kwargs` 也支持通过 secrets 传入。

#### SnowflakeConnection 的参数合并

[SnowflakeConnection._connect()](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/snowflake_connection.py#L625-L690) 按运行环境分级处理：

1. **SiS (Streamlit in Snowflake) 环境**：直接获取活跃 session 的连接，忽略所有 kwargs
2. **有 secrets 配置**：`{**st_secrets, **kwargs}` 合并
3. **name=="snowflake" 且无 secrets**：使用 Snowflake 默认连接配置
4. **有 connection_name 且无 kwargs**：使用 Snowflake connections.toml 中的命名连接
5. **仅有 kwargs**：直接传递给 `snowflake.connector.connect()`

#### SnowflakeCallersRightsConnection 的参数获取

[SnowflakeCallersRightsConnection._connect()](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/snowflake_connection.py#L777-L787) 读取以下三部分凭据：

| 凭据 | 来源 |
|------|------|
| `account`, `host`, `database`, `schema` | 环境变量 `SNOWFLAKE_ACCOUNT` 等 |
| login token | 文件 `/snowflake/session/token` |
| user token | 请求头 `Sf-Context-Current-User-Token` |

合并策略：`**{**params, **kwargs}` — 用户 kwargs 可覆盖自动参数。

#### SnowparkConnection 的参数合并（已废弃）

[SnowparkConnection._connect()](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/snowpark_connection.py#L82-L108) 使用三层 ChainMap：

```python
conn_params = ChainMap(
    kwargs,                                        # 最高优先级
    self._secrets.to_dict(),                       # 中等优先级
    load_from_snowsql_config_file(self._connection_name),  # 最低优先级
)
```

### 1.3 Secrets 的读取与监听

[BaseConnection._secrets](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/base_connection.py#L111-L128) 属性从 `secrets_singleton` 中读取 `[connections.<connection_name>]` 段：

```python
@property
def _secrets(self) -> AttrDict:
    connections_section = None
    if secrets_singleton.load_if_toml_exists():
        connections_section = secrets_singleton.get("connections")
    if connections_section is None or type(connections_section) is not AttrDict:
        return AttrDict({})
    return connections_section.get(self._connection_name, AttrDict({}))
```

[secrets_singleton](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/secrets.py#L642) 是全局单例，其特点：
- 懒加载：首次访问时才解析 TOML 文件
- 文件监听：安装 watcher 监听 secrets.toml 变更
- 变更信号：通过 `blinker.Signal` 广播变更事件
- 线程安全：使用 `threading.RLock` 保护

---

## 二、连接缓存层

### 2.1 cache_resource 包装机制

[_create_connection()](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/connection_factory.py#L66-L119) 是连接缓存的核心。它将 `BaseConnection` 子类的实例化包装在 `@st.cache_resource` 中：

```python
def _create_connection(name, connection_class, max_entries=None, ttl=None, **kwargs):
    def __create_connection(name, connection_class, **kwargs):
        return connection_class(connection_name=name, **kwargs)

    # 修改 __qualname__ 避免不同 ttl/max_entries 重置缓存
    __create_connection.__qualname__ = (
        f"{__create_connection.__qualname__}_{ttl_str}_{max_entries}"
    )

    scope = connection_class.scope()

    def on_release_wrapped(connection):
        connection.close()

    __create_connection = cache_resource(
        max_entries=max_entries,
        show_spinner="Running `st.connection(...)`.",
        ttl=ttl,
        scope=scope,
        on_release=on_release_wrapped,
    )(__create_connection)

    return __create_connection(name, connection_class, **kwargs)
```

关键设计决策：

1. **`__qualname__` 修改**：Streamlit 的 `cache_resource` 默认用函数的 `__qualname__` 作为缓存键的一部分。如果 `ttl` 或 `max_entries` 不同，不修改 `__qualname__` 会导致缓存被重置。通过在 `__qualname__` 后附加 ttl 和 max_entries，使得不同的缓存参数对应不同的缓存空间。

2. **`on_release` 钩子**：当缓存条目被移除时（TTL 过期、max_entries 淘汰、session 断开），调用 `connection.close()` 释放资源。

3. **`scope` 参数**：由连接类的 `scope()` 类方法决定，默认 `"global"`。

### 2.2 全局缓存 vs 会话缓存

[ResourceCaches](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/caching/cache_resource_api.py#L91-L163) 管理所有 `cache_resource` 缓存实例：

```python
class ResourceCaches:
    def __init__(self):
        self._caches_lock = threading.Lock()
        # Map of session IDs to map of function keys to caches.
        self._function_caches: dict[str | None, dict[str, ResourceCache]] = {}
```

| scope 值 | session_id | 生命周期 | 适用场景 |
|----------|-----------|---------|---------|
| `"global"` | `None` | 进程生命周期，跨所有用户共享 | SQLConnection, SnowflakeConnection, SnowparkConnection |
| `"session"` | 当前会话 ID | 会话断开时清除 | SnowflakeCallersRightsConnection |

获取缓存时的逻辑（[cache_resource_api.py#L103-L163](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/caching/cache_resource_api.py#L103-L163)）：

```python
def get_cache(self, key, display_name, max_entries, ttl, validate, on_release, scope="global"):
    if scope == "global":
        session_id = None
    else:
        session_id = get_session_id_or_throw()  # 从线程上下文获取

    with self._caches_lock:
        session_caches = self._function_caches.get(session_id)
        # ... 查找或创建缓存
```

### 2.3 会话断开时的缓存清理

当用户会话断开时，[AppSession.clear_session_caches()](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/app_session.py#L263-L270) 被调用：

```python
def clear_session_caches(self) -> None:
    caching.clear_session_data_cache(self.id)
    caching.clear_session_resource_cache(self.id)
```

[ResourceCaches.clear_session()](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/caching/cache_resource_api.py#L165-L175) 的实现：

```python
def clear_session(self, session_id):
    with self._caches_lock:
        session_caches = self._function_caches.get(session_id)
        if session_caches is not None:
            del self._function_caches[session_id]
    if session_caches is not None:
        for cache in session_caches.values():
            cache.clear()  # 触发 on_release → connection.close()
```

这意味着会话断开时，所有 `scope="session"` 的连接都会被关闭。

### 2.4 缓存键的构成

`cache_resource` 的缓存键由以下部分组成：
- 函数的 `__qualname__`（包含 ttl/max_entries 后缀）
- 函数参数的哈希值（`name`, `connection_class`, 其他 kwargs）

对于相同的 `name` + `connection_class` + `kwargs` 组合，只会创建一个连接实例。不同 `ttl`/`max_entries` 会创建不同的缓存空间（因为 `__qualname__` 不同），但不会导致旧缓存被清除。

---

## 三、用户调用层

### 3.1 st.connection() 入口

用户通过 `st.connection()` 创建连接，该函数在 [\_\_init\_\_.py#L293](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/__init__.py#L293) 导出：

```python
from streamlit.runtime.connection_factory import connection_factory as _connection
connection = _connection
```

调用流程：

```
st.connection("sql")
  → connection_factory(name="sql", type=None)
    → _get_first_party_connection("sql") → SQLConnection
    → _create_connection("sql", SQLConnection)
      → cache_resource 包装的 __create_connection("sql", SQLConnection)
        → SQLConnection.__init__("sql")
          → self._secrets 读取 [connections.sql]
          → self._connect() 创建 SQLAlchemy Engine
```

### 3.2 连接实例的惰性重建

[BaseConnection._instance](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/base_connection.py#L153-L159) 属性实现惰性重建：

```python
@property
def _instance(self) -> RawConnectionT:
    if self._raw_instance is None:
        self._raw_instance = self._connect(**self._kwargs)
    return self._raw_instance
```

当 `_raw_instance` 为 `None` 时，下次访问 `_instance` 会自动调用 `_connect()` 重建。这发生在以下场景：
- 手动调用 `reset()`
- secrets 变更触发 `_on_secrets_changed()`
- query 方法的 retry 逻辑中调用 `self.reset()`

### 3.3 Secrets 变更响应

[BaseConnection._on_secrets_changed()](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/base_connection.py#L97-L109) 响应 secrets 文件变更：

```python
def _on_secrets_changed(self, _):
    new_hash = calc_hash(json.dumps(self._secrets.to_dict()))
    if new_hash != self._config_section_hash:
        self._config_section_hash = new_hash
        self.reset()  # _raw_instance = None
```

注意：这里只重置底层连接对象（`_raw_instance = None`），不会替换缓存中的 `BaseConnection` 实例本身。连接实例的下次使用会通过 `_instance` 属性惰性重建底层连接。

### 3.4 query() 方法的双层缓存

连接的 `query()` 方法使用了**双层缓存**设计：

1. **连接级缓存**（`cache_resource`）：`st.connection()` 返回的连接对象本身被缓存
2. **查询级缓存**（`cache_data`）：`query()` 的结果被缓存

以 [SQLConnection.query()](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/sql_connection.py#L224-L358) 为例：

```python
def query(self, sql, *, ttl=None, **kwargs):
    @retry(after=lambda _: self.reset(), stop=stop_after_attempt(3), ...)
    def _query(instance_id, sql, index_col=None, ...):
        instance = self._instance.connect()
        return pd.read_sql(text(sql), instance, ...)

    # 修改 __qualname__ 避免不同 ttl 重置缓存，并按连接名隔离
    _query.__qualname__ = f"{_query.__qualname__}_{self._connection_name}_{ttl_str}"
    _query = cache_data(show_spinner=..., ttl=ttl)(_query)

    return _query(self._connection_instance_id, sql, ...)
```

关键设计：
- `_connection_instance_id`（UUID）作为 `_query` 的参数，确保每个连接实例有独立的查询缓存
- `__qualname__` 被修改以包含连接名和 TTL，实现连接隔离和 TTL 隔离
- retry 机制在数据库错误时自动 `reset()` 连接并重试（最多 3 次）

### 3.5 连接的 close() 生命周期

各连接类的 `close()` 行为：

| 连接类 | close() 行为 |
|--------|-------------|
| BaseConnection | 空操作（no-op） |
| SnowflakeConnection / BaseSnowflakeConnection | 调用底层 `_raw_instance.close()` 并置 None |
| SQLConnection | 继承默认空操作（SQLAlchemy Engine 自身管理连接池） |
| SnowparkConnection | 继承默认空操作 |

`close()` 的调用时机：
- `cache_resource` 的 `on_release` 钩子触发时（TTL 过期 / max_entries 淘汰 / session 断开）
- 全局缓存的 `clear_all()` 时（`st.cache_resource.clear()`）

---

## 四、完整生命周期时序图

### 4.1 首次创建连接

```
用户脚本                     connection_factory            _create_connection          BaseConnection            cache_resource
   │                              │                            │                          │                         │
   │ st.connection("sql")        │                            │                          │                         │
   │─────────────────────────────>│                            │                          │                         │
   │                              │ 解析 type=None             │                          │                         │
   │                              │ name="sql"→SQLConnection   │                          │                         │
   │                              │───────────────────────────>│                          │                         │
   │                              │                            │ cache_resource 装饰       │                         │
   │                              │                            │──────────────────────────────────────────────────>│
   │                              │                            │                          │    缓存未命中            │
   │                              │                            │                          │<───────────────────────│
   │                              │                            │                          │  调用 __create_connection│
   │                              │                            │                          │────────────────────────>│
   │                              │                            │                          │  __init__("sql")        │
   │                              │                            │                          │  读取 _secrets          │
   │                              │                            │                          │  _connect() → Engine    │
   │                              │                            │                          │  注册 secrets 监听器     │
   │                              │                            │                          │────────────────────────>│
   │                              │                            │                          │    写入缓存              │
   │<──────────────────────────────────────────────────────────────────────────────────────────────────────────│
   │  返回 SQLConnection 实例    │                            │                          │                         │
```

### 4.2 再次调用（缓存命中）

```
用户脚本                     connection_factory            _create_connection          cache_resource
   │                              │                            │                         │
   │ st.connection("sql")        │                            │                         │
   │─────────────────────────────>│                            │                         │
   │                              │───────────────────────────>│                         │
   │                              │                            │ cache_resource 查找      │
   │                              │                            │────────────────────────>│
   │                              │                            │    缓存命中              │
   │                              │                            │<────────────────────────│
   │<─────────────────────────────────────────────────────────────────────────────────│
   │  返回同一个 SQLConnection   │                            │                         │
```

### 4.3 Secrets 变更 → 连接重置

```
secrets.toml 变更         secrets_singleton           BaseConnection          _instance 属性
   │                          │                          │                      │
   │ 文件修改                  │                          │                      │
   │─────────────────────────>│ _on_secrets_changed()    │                      │
   │                          │ 重载 secrets              │                      │
   │                          │ file_change_listener.send()                     │
   │                          │─────────────────────────>│ _on_secrets_changed()│
   │                          │                          │ 比较 hash            │
   │                          │                          │ hash 不同 → reset()  │
   │                          │                          │ _raw_instance = None │
   │                          │                          │                      │
   │   用户下次访问 conn.query()                       │                      │
   │                          │                          │ 访问 _instance        │
   │                          │                          │─────────────────────>│
   │                          │                          │ _raw_instance is None│
   │                          │                          │ 重新 _connect()      │
   │                          │                          │ 新的底层连接对象       │
```

### 4.4 会话断开（session-scoped 连接）

```
WebSocket 断开          AppSession              ResourceCaches           ResourceCache          BaseConnection
   │                       │                        │                       │                      │
   │                       │ clear_session_caches() │                       │                      │
   │──────────────────────>│                        │                       │                      │
   │                       │ clear_session(id)      │                       │                      │
   │                       │───────────────────────>│                       │                      │
   │                       │                        │ 删除 session_caches   │                      │
   │                       │                        │──────────────────────>│                      │
   │                       │                        │                       │ cache.clear()        │
   │                       │                        │                       │ on_release(conn)     │
   │                       │                        │                       │─────────────────────>│
   │                       │                        │                       │                      │ conn.close()
   │                       │                        │                       │                      │ _raw_instance.close()
```

---

## 五、核心文件索引

| 文件 | 职责 |
|------|------|
| [connection_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/connection_factory.py) | `st.connection()` 工厂函数，类型解析，`cache_resource` 包装 |
| [base_connection.py](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/base_connection.py) | 抽象基类，secrets 读取，惰性重建，secrets 变更监听 |
| [sql_connection.py](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/sql_connection.py) | SQLAlchemy Engine 连接，query() 双层缓存+重试 |
| [snowflake_connection.py](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/snowflake_connection.py) | Snowflake 连接族（含 CallersRights），环境自适应 |
| [snowpark_connection.py](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/connections/snowpark_connection.py) | Snowpark Session 连接（已废弃） |
| [cache_resource_api.py](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/caching/cache_resource_api.py) | `cache_resource` 实现，全局/会话缓存管理，`on_release` 钩子 |
| [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/secrets.py) | Secrets 单例，TOML 解析，文件监听，变更信号 |
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/228-streamlit/lib/streamlit/runtime/app_session.py) | 会话生命周期管理，断开时清理缓存 |

---

## 六、设计要点总结

1. **连接实例是单例**：同一 `(name, type, kwargs)` 组合在全局/会话范围内只创建一次，由 `cache_resource` 保证。

2. **底层连接可惰性重建**：`BaseConnection` 区分"连接包装对象"（缓存的）和"底层连接对象"（可 reset 的）。Secrets 变更或 retry 只重置底层连接，不替换缓存中的包装对象。

3. **`__qualname__` 修改是关键的缓存隔离技巧**：避免不同 `ttl`/`max_entries` 参数互相冲刷缓存，同时为 `query()` 实现连接级隔离。

4. **Scope 决定连接的可见性和生命周期**：`"global"` 连接跨会话共享，`"session"` 连接随会话创建和销毁。`SnowflakeCallersRightsConnection` 是唯一的内置 session-scoped 连接。

5. **`on_release` 钩子确保资源释放**：缓存条目移除时自动调用 `close()`，避免连接泄漏。但全局资源的 `on_release` 不保证在应用关闭时调用。

6. **Secrets 变更的细粒度响应**：仅当本连接对应的 secrets 段发生变更时才重置，通过 `_config_section_hash` 比较，避免无关变更触发不必要的重连。
