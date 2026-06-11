# Session State 持久化机制代码走读

本文档从代码实现角度梳理 Streamlit Session State 的持久化机制，包括状态创建、会话关联、序列化保存和恢复时机。

## 1. 核心类结构

### 1.1 SessionState
[session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py)

SessionState 是状态存储的核心类，内部维护三层状态结构：

```python
@dataclass(slots=True)
class SessionState:
    _old_state: dict[str, Any] = field(default_factory=dict)           # 历史运行状态（已压缩）
    _new_session_state: dict[str, Any] = field(default_factory=dict)  # 当前运行中新设置的用户状态
    _new_widget_state: WStates = field(default_factory=WStates)       # 当前运行中前端传来的 widget 状态
    _key_id_mapper: KeyIdMapper = field(default_factory=KeyIdMapper)  # 用户 key <-> widget_id 映射
    query_params: QueryParams = field(default_factory=QueryParams)    # 查询参数绑定
```

### 1.2 SafeSessionState
[safe_session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/safe_session_state.py)

线程安全包装器，使用 `threading.RLock` 保证并发安全，所有访问 SessionState 的操作都通过它进行：

```python
class SafeSessionState:
    _state: SessionState
    _lock: threading.RLock
    _yield_callback: Callable[[], None]
```

### 1.3 WStates
[session_state.py#L138-L344](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L138-L344)

Widget 状态容器，支持两种存储形式：
- `Value`: 反序列化后的 Python 对象
- `Serialized`: Protobuf 格式的序列化数据

```python
@dataclass(frozen=True, slots=True)
class Serialized:
    value: WidgetStateProto  # Protobuf 格式

@dataclass(frozen=True, slots=True)
class Value:
    value: Any               # Python 对象

WState: TypeAlias = Value | Serialized
```

---

## 2. 状态创建流程

### 2.1 初始化时机

**AppSession 初始化时创建 SessionState**：

[app_session.py#L188-L190](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/app_session.py#L188-L190)

```python
# 在 AppSession.__init__ 中
from streamlit.runtime.state import SessionState
self._session_state = SessionState()  # 创建空的 SessionState
```

**ScriptRunner 初始化时创建 SafeSessionState 包装**：

[script_runner.py#L238-L239](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L238-L239)

```python
self._session_state = SafeSessionState(
    session_state, yield_callback=self._maybe_handle_execution_control_request
)
```

### 2.2 Widget 状态注册

当脚本执行到 widget 代码时（如 `st.slider("x", 0, 10, key="my_slider")`），会调用 `register_widget` 进行状态注册：

[session_state.py#L998-L1109](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L998-L1109)

```python
def register_widget(self, metadata: WidgetMetadata[T], user_key: str | None) -> RegisterWidgetResult[T]:
    widget_id = metadata.id
    self._set_widget_metadata(metadata)
    
    if user_key is not None:
        self._set_key_widget_mapping(widget_id, user_key)  # 建立 key <-> widget_id 映射
    
    # 首次注册时初始化值
    if widget_id not in self and (user_key is None or user_key not in self):
        deserializer = metadata.deserializer
        initial_widget_value = deepcopy(deserializer(None))
        self._new_widget_state.set_from_value(widget_id, initial_widget_value)
    
    widget_value = cast("T", self[widget_id])
    widget_value = deepcopy(widget_value)
    
    return RegisterWidgetResult(widget_value, widget_value_changed)
```

---

## 3. 会话关联机制

### 3.1 会话标识

每个 `AppSession` 有唯一的 `session_id`：

[app_session.py#L153](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/app_session.py#L153)

```python
self.id = session_id_override or str(uuid.uuid4())
```

### 3.2 会话管理器

`WebsocketSessionManager` 管理活跃会话和非活跃会话的生命周期：

[websocket_session_manager.py#L88](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L88)

```python
# 活跃会话表（内存中）
self._active_session_info_by_id: dict[str, ActiveSessionInfo] = {}
```

`ActiveSessionInfo` 关联了客户端连接和会话状态：

[session_manager.py#L84-L96](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/session_manager.py#L84-L96)

```python
@dataclass
class ActiveSessionInfo:
    client: SessionClient       # WebSocket 客户端
    session: AppSession         # 会话实例（包含 SessionState）
    script_run_count: int = 0
```

### 3.3 会话状态访问链

```
用户代码 st.session_state
    ↓
ScriptRunner._session_state (SafeSessionState)
    ↓
AppSession._session_state (SessionState)
    ↓
_old_state / _new_session_state / _new_widget_state
```

---

## 4. 序列化保存机制

### 4.1 序列化检查

使用 Python `pickle` 作为序列化标准，在脚本运行完成后检查可序列化性：

[session_state.py#L1292-L1318](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L1292-L1318)

```python
def _check_serializable(self) -> None:
    for k in self:
        try:
            pickle.dumps(self[k])  # 尝试 pickle 序列化
        except Exception as e:
            raise UnserializableSessionStateError(err_msg) from e

def maybe_check_serializable(self) -> None:
    if config.get_option("runner.enforceSerializableSessionState"):
        self._check_serializable()
```

**调用时机**：脚本执行完成后

[script_runner.py#L802](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L802)

```python
self._session_state.maybe_check_serializable()
```

### 4.2 会话断开时的持久化

当 WebSocket 连接断开时，会话被保存到 `SessionStorage`：

[websocket_session_manager.py#L170-L189](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L170-L189)

```python
def disconnect_session(self, session_id: str) -> None:
    if session_id in self._active_session_info_by_id:
        active_session_info = self._active_session_info_by_id[session_id]
        session = active_session_info.session
        
        session.request_script_stop()
        session.disconnect_file_watchers()
        session.clear_session_caches()
        
        # 保存到 SessionStorage（包含完整的 SessionState）
        self._session_storage.save(
            SessionInfo(
                client=None,
                session=session,              # 整个 AppSession 被保存
                script_run_count=active_session_info.script_run_count,
            )
        )
        del self._active_session_info_by_id[session_id]
```

### 4.3 内存存储实现

`MemorySessionStorage` 使用带 TTL 的缓存：

[memory_session_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/memory_session_storage.py)

```python
class MemorySessionStorage(SessionStorage):
    def __init__(self, maxsize: int = 128, ttl_seconds: int = 2 * 60) -> None:
        self._cache: MutableMapping[str, SessionInfo] = TTLCache(
            maxsize=maxsize, ttl=ttl_seconds  # 默认 2 分钟 TTL
        )
    
    def save(self, session_info: SessionInfo) -> None:
        self._cache[session_info.session.id] = session_info  # key = session_id
```

### 4.4 Widget 状态的 Proto 序列化

Widget 状态可以序列化为 Protobuf 格式用于前后端通信：

[session_state.py#L264-L310](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L264-L310)

```python
def get_serialized(self, k: str) -> WidgetStateProto | None:
    item = self.states.get(k)
    if item is None:
        return None
    
    if isinstance(item, Serialized):
        return item.value  # 已经是序列化格式
    
    # Value -> Serialized 转换
    metadata = self.widget_metadata.get(k)
    widget = WidgetStateProto()
    widget.id = k
    
    field = metadata.value_type
    serialized = metadata.serializer(item.value)
    
    # 根据类型设置到 proto 字段
    if is_array_value_field_name(field):
        arr = getattr(widget, field)
        arr.data.extend(serialized)
    elif field in {"json_value", "json_trigger_value"}:
        setattr(widget, field, json.dumps(serialized))
    # ... 其他类型处理
    
    return widget
```

---

## 5. 状态恢复时机

### 5.1 会话重连时的完整恢复

当客户端重新连接并提供 `existing_session_id` 时，从 `SessionStorage` 恢复完整会话：

[websocket_session_manager.py#L100-L140](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L100-L140)

```python
def connect_session(
    self,
    client: SessionClient,
    script_data: ScriptData,
    user_info: UserInfoType,
    existing_session_id: str | None = None,
    session_id_override: str | None = None,
) -> str:
    # 尝试从存储中恢复
    session_info = (
        existing_session_id
        and existing_session_id not in self._active_session_info_by_id
        and self._session_storage.get(existing_session_id)
    )
    
    if isinstance(session_info, SessionInfo):
        existing_session = session_info.session
        existing_session.register_file_watchers()  # 重新注册文件监听器
        
        # 重新加入活跃会话表
        self._active_session_info_by_id[existing_session.id] = ActiveSessionInfo(
            client,
            existing_session,
            session_info.script_run_count,
        )
        self._session_storage.delete(existing_session.id)  # 从存储移除
        return existing_session.id
    
    # 未找到则创建新会话
    session = AppSession(...)
```

### 5.2 脚本重运行前的状态压缩

每次脚本重运行前，调用 `on_script_will_rerun` 进行状态压缩：

[session_state.py#L641-L651](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L641-L651)

```python
def on_script_will_rerun(self, latest_widget_states: WidgetStatesProto) -> None:
    self._reset_triggers()               # 重置触发器状态
    self._compact_state()                # 压缩：新状态 -> 旧状态
    self.set_widgets_from_proto(latest_widget_states)  # 加载前端最新 widget 状态
    self._call_callbacks()               # 调用值变化的回调
```

**状态压缩逻辑** `_compact_state`：

[session_state.py#L448-L461](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L448-L461)

```python
def _compact_state(self) -> None:
    # 将 _new_session_state 和 _new_widget_state 合并到 _old_state
    for key_or_wid in self:
        try:
            self._old_state[key_or_wid] = self[key_or_wid]
        except KeyError:
            pass
    # 清空新状态，准备接收下一轮运行的数据
    self._new_session_state.clear()
    self._new_widget_state.clear()
```

**调用时机**：脚本运行循环开始时

[script_runner.py#L708-L711](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L708-L711)

```python
if rerun_data.widget_states is not None:
    self._session_state.on_script_will_rerun(
        rerun_data.widget_states
    )
```

### 5.3 脚本运行完成后的状态清理

脚本运行完成后，调用 `on_script_finished` 清理过期 widget：

[session_state.py#L866-L878](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L866-L878)

```python
def on_script_finished(self, widget_ids_this_run: frozenset[str]) -> None:
    self._reset_triggers()                     # 重置触发器
    self._remove_stale_widgets(widget_ids_this_run)  # 移除未在本次运行中出现的 widget
```

**调用时机**有两处：

1. **下一次运行前的清理**（处理 MPA 页面切换）：

[script_runner.py#L611](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L611)

```python
self._session_state.on_script_finished(widget_ids)
```

2. **脚本正常结束后**：

[script_runner.py#L866-L868](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L866-L868)

```python
if not premature_stop:
    self._session_state.on_script_finished(ctx.widget_ids_this_run.snapshot())
```

### 5.4 前端 Widget 状态的反序列化

从前端接收 Protobuf 状态后反序列化为 Python 对象：

[session_state.py#L151-L203](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L151-L203)

```python
def __getitem__(self, k: str) -> Any:
    wstate = self.states.get(k)
    if isinstance(wstate, Value):
        return wstate.value  # 已反序列化
    
    # Serialized -> Value 转换（懒加载）
    metadata = self.widget_metadata.get(k)
    value_field_name = wstate.value.WhichOneof("value")
    value = wstate.value.__getattribute__(value_field_name)
    
    # 特殊类型处理
    if is_array_value_field_name(value_field_name):
        value = value.data
    elif value_field_name == "json_value":
        value = json.loads(value)
    
    deserialized = metadata.deserializer(value)
    self.states[k] = Value(deserialized)  # 缓存反序列化结果
    return deserialized
```

---

## 6. 完整生命周期时序

```
客户端连接
    ↓
AppSession 初始化 → 创建 SessionState (空)
    ↓
首次脚本运行
    ↓
┌─────────────────────────────────────────────────┐
│ 脚本执行循环                                    │
│    ↓                                            │
│ on_script_will_rerun()                          │
│   ├─ _reset_triggers()                          │
│   ├─ _compact_state()    新状态 → _old_state    │
│   ├─ set_widgets_from_proto()  前端状态加载      │
│   └─ _call_callbacks()      值变化回调执行       │
│    ↓                                            │
│ 用户脚本执行 → 注册 widget → 访问/修改 state    │
│    ↓                                            │
│ maybe_check_serializable()  序列化检查          │
│    ↓                                            │
│ _on_script_finished()                           │
│   └─ on_script_finished()  清理 stale widget    │
└─────────────────────────────────────────────────┘
    ↓
WebSocket 断开
    ↓
disconnect_session() → 保存到 MemorySessionStorage (TTL 2分钟)
    ↓
客户端重连（带 existing_session_id）
    ↓
connect_session() → 从 SessionStorage 恢复完整 SessionState
    ↓
继续脚本运行循环
```

---

## 7. 关键设计要点

1. **三层状态结构**：`_old_state`（持久化）、`_new_session_state`（用户设置）、`_new_widget_state`（前端传来），通过 `_compact_state` 在每次运行前合并，保证状态一致性。

2. **懒加载反序列化**：Widget 状态从前端接收时保持 Protobuf 格式，首次访问时才反序列化为 Python 对象，提升性能。

3. **会话级持久化**：断开连接时保存整个 `AppSession`（包含 `SessionState`），重连时完整恢复，实现无缝的用户体验。

4. **Stale Widget 清理**：每次运行结束后清理未出现的 widget 状态，防止内存泄漏，同时保留查询参数绑定的 widget 值。

5. **线程安全**：`SafeSessionState` 使用 `RLock` 保护所有状态访问，支持脚本被中断后新脚本立即运行的场景。
