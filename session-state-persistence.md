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

### 4.1 序列化检查详解

序列化检查分为两个层面：**预检机制**和**实际保存时的隐式序列化**。

#### 4.1.1 预检机制：maybe_check_serializable

使用 Python `pickle` 作为序列化标准，**仅在配置开启时执行可序列化性验证**，不做实际持久化：

[session_state.py#L1292-L1318](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L1292-L1318)

```python
def _check_serializable(self) -> None:
    """验证 session_state 中所有值是否可 pickle 序列化。"""
    for k in self:
        try:
            pickle.dumps(self[k])  # 仅尝试序列化，结果被丢弃
        except Exception as e:
            err_msg = (
                f"Cannot serialize the value (of type `{type(self[k])}`) of '{k}' in "
                "st.session_state. Streamlit has been configured to use "
                "[pickle](https://docs.python.org/3/library/pickle.html) to "
                "serialize session_state values..."
            )
            raise UnserializableSessionStateError(err_msg) from e

def maybe_check_serializable(self) -> None:
    """仅当 runner.enforceSerializableSessionState 配置为 True 时执行检查。"""
    if config.get_option("runner.enforceSerializableSessionState"):
        self._check_serializable()
```

**关键点**：
- 这是一个**防御性检查**，不是实际的持久化操作
- 配置开关 `runner.enforceSerializableSessionState` 默认为关闭
- 检查遍历 SessionState 的所有 key（包括 `_old_state`、`_new_session_state`、`_new_widget_state`）
- 通过 `self[k]` 触发完整查找链，确保 widget state 也被检查

#### 4.1.2 调用时机：脚本执行完成后

[script_runner.py#L798-L804](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L798-L804)

```python
# 用户脚本执行完毕后
self._fragment_storage.clear(
    new_fragment_ids=ctx.new_fragment_ids.snapshot()
)

self._session_state.maybe_check_serializable()  # ← 在此调用
# check for control requests, e.g. rerun requests have arrived
self._maybe_handle_execution_control_request()
```

**时间线位置**：在 `exec_func_with_error_handling(code_to_exec, ctx)` 执行用户脚本之后，在 `_on_script_finished` 之前。

### 4.2 会话断开时的实际保存过程

当 WebSocket 连接断开时，整个 `AppSession` 对象（包含 `SessionState`）被保存到 `SessionStorage`。

#### 4.2.1 断开触发点

[starlette_websocket.py#L455-L460](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L455-L460)

```python
while True:
    try:
        data = await websocket.receive_bytes()
    except WebSocketDisconnect:
        break  # ← 捕获断开异常，跳出消息循环
```

连接断开后的清理：

[starlette_websocket.py#L515-L525](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L515-L525)

```python
finally:
    if session_id is not None:
        try:
            runtime.disconnect_session(session_id)  # ← 触发会话保存
        except Exception:
            _LOGGER.exception("Error disconnecting session")
```

#### 4.2.2 保存执行流程

[runtime.py#L485-L507](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/runtime.py#L485-L507) → [websocket_session_manager.py#L170-L189](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L170-L189)

```python
def disconnect_session(self, session_id: str) -> None:
    if session_id in self._active_session_info_by_id:
        active_session_info = self._active_session_info_by_id[session_id]
        session = active_session_info.session

        # 步骤1: 停止正在运行的脚本
        session.request_script_stop()

        # 步骤2: 断开文件监听器（释放监听资源）
        session.disconnect_file_watchers()
        # [app_session.py#L248-L261] 关闭 LocalSourcesWatcher, config listener, secrets listener

        # 步骤3: 清除会话级缓存（释放内存）
        session.clear_session_caches()
        # [app_session.py#L263-L270] clear_session_data_cache + clear_session_resource_cache

        # 步骤4: 保存到 SessionStorage（核心！整个 AppSession 对象被存入）
        self._session_storage.save(
            SessionInfo(
                client=None,                    # 断开后 client 置空
                session=session,                 # 包含完整 SessionState 的引用
                script_run_count=active_session_info.script_run_count,
            )
        )

        # 步骤5: 从活跃表移除
        del self._active_session_info_by_id[session_id]
```

#### 4.2.3 实际存储：内存引用而非序列化

[memory_session_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/memory_session_storage.py)

```python
class MemorySessionStorage(SessionStorage):
    def __init__(self, maxsize: int = 128, ttl_seconds: int = 2 * 60) -> None:
        # 使用 TTLCache，默认最多存 128 个会话，TTL 2 分钟
        self._cache: MutableMapping[str, SessionInfo] = TTLCache(
            maxsize=maxsize, ttl=ttl_seconds
        )

    def save(self, session_info: SessionInfo) -> None:
        # key = session_id, value = SessionInfo 对象的直接引用
        # ⚠️ 注意：这里没有做 pickle 序列化，只是 Python 对象的内存引用
        self._cache[session_info.session.id] = session_info

    def get(self, session_id: str) -> SessionInfo | None:
        return self._cache.get(session_id, None)
```

**关键理解**：
- `MemorySessionStorage` 存储的是 Python 对象的**直接内存引用**，不是序列化后的字节
- `maybe_check_serializable` 的 pickle 检查是为**未来可能的持久化实现**做预检（如写入磁盘或跨进程传输）
- 2 分钟 TTL 后会话对象会被自动清理，Python GC 回收内存

---

## 4B. 两条恢复路径的深度对比

### 4B.1 路径一：普通重跑（用户交互触发）

当用户在页面上操作 widget（如拖动滑块、点击按钮）时触发。

#### 完整调用链

```
前端 widget 交互
    ↓ WebSocket 发送 BackMsg
    ↓ [starlette_websocket.py#L471-L480] 解析 BackMsg protobuf
    ↓ [runtime.py#L509-L533] runtime.handle_backmsg()
    ↓ [app_session.py#L338-L371] AppSession.handle_backmsg()
    ↓   msg_type == "rerun_script"
    ↓ [app_session.py#L916-L928] _handle_rerun_script_request()
    ↓ [app_session.py#L406-L488] request_rerun(client_state)
    ↓
    ├─ 分支A: 复用现有 ScriptRunner
    │   [app_session.py#L481] self._scriptrunner.request_rerun(rerun_data)
    │   ↓
    │   ScriptRunner._run_script_loop 下一次迭代
    │
    └─ 分支B: 新建 ScriptRunner (fastReruns 或首次运行)
        [app_session.py#L488] self._create_scriptrunner(rerun_data)
        ↓
        [script_runner.py#L174-L249] 新建 ScriptRunner
        ↓   session_state = AppSession._session_state (同一个对象引用!)
        ↓   包装为 SafeSessionState
        ↓
        ScriptRunner.start() → _run_script_loop()
```

#### 状态恢复的连接点：on_script_will_rerun

不论分支 A 还是 B，最终都会进入 `_run_script_loop`，在每次脚本执行前调用：

[script_runner.py#L707-L711](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L707-L711)

```python
# Run callbacks for widgets whose values have changed.
if rerun_data.widget_states is not None:
    self._session_state.on_script_will_rerun(
        rerun_data.widget_states
    )
```

[session_state.py#L641-L651](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L641-L651)

```python
def on_script_will_rerun(self, latest_widget_states: WidgetStatesProto) -> None:
    """Called by ScriptRunner before its script re-runs."""
    # 1. 重置触发器（按钮点击状态等临时状态）
    self._reset_triggers()

    # 2. ★ 状态压缩：上一轮的新状态 → _old_state（持久化层）
    self._compact_state()

    # 3. 加载前端传来的最新 widget 状态（Protobuf 格式，存入 _new_widget_state）
    self.set_widgets_from_proto(latest_widget_states)

    # 4. 调用值变化的 widget 回调
    self._call_callbacks()
```

#### _compact_state 的详细过程

[session_state.py#L448-L461](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L448-L461)

```python
def _compact_state(self) -> None:
    # 遍历所有 key（_old_state ∪ _new_widget_state ∪ _new_session_state）
    for key_or_wid in self:
        try:
            # self[key_or_wid] 触发完整查找链：
            #   _new_session_state → _new_widget_state → _old_state
            # 对于 widget_state，会触发懒加载反序列化
            self._old_state[key_or_wid] = self[key_or_wid]
        except KeyError:
            # widget 缺少 metadata 时忽略（如重连后的残留状态）
            pass

    # 清空，准备接收新一轮数据
    self._new_session_state.clear()
    self._new_widget_state.clear()
```

**状态流转**：
```
运行前:
  _old_state:       {slider_1: 5, "counter": 3}        ← 历史值
  _new_widget_state: {}                                   ← 空
  _new_session_state: {}                                  ← 空

用户交互（拖动滑块到 8）→ 前端发送 widget_states

on_script_will_rerun:
  _compact_state() → _old_state 保持不变（没有新值可合并）
  set_widgets_from_proto({slider_1: 8}) → _new_widget_state: {slider_1: Serialized(8)}

脚本执行中:
  用户代码 st.slider(..., key="slider_1")
    → register_widget() → self[widget_id]
    → WStates.__getitem__ → 反序列化 Serialized(8) → Value(8)
  用户代码 st.session_state.counter = 4
    → SessionState.__setitem__ → _new_session_state: {"counter": 4}

运行结束 → 下次 on_script_will_rerun:
  _compact_state():
    _old_state["slider_1"] = 8   ← 从 _new_widget_state 合并
    _old_state["counter"] = 4    ← 从 _new_session_state 合并
    _new_widget_state.clear()
    _new_session_state.clear()
```

---

### 4B.2 路径二：断线重连（WebSocket 重连）

当用户网络短暂中断或浏览器标签页休眠后恢复时触发。

#### 完整调用链

```
WebSocket 连接断开
    ↓ [starlette_websocket.py] WebSocketDisconnect → runtime.disconnect_session()
    ↓ [websocket_session_manager.py#L170-L189] 保存到 MemorySessionStorage
    ↓ SessionInfo(AppSession) 存入 TTLCache (TTL 2分钟)
    ↓
    ⏳ 时间窗口：2 分钟内用户重新连接
    ↓
前端发起新 WebSocket 连接
    ↓ Sec-WebSocket-Protocol 头携带 existing_session_id
    ↓   格式: "streamlit, <xsrf_token>, <session_id>"
    ↓
[starlette_websocket.py#L64-L88] _parse_subprotocols()
    ↓ 解析 headers["sec-websocket-protocol"]
    ↓ 返回 (subprotocol, xsrf_token, existing_session_id)
    ↓
[starlette_websocket.py#L449-L453] runtime.connect_session()
    ↓ existing_session_id=existing_session_id
    ↓
[runtime.py#L376-L437] Runtime.connect_session()
    ↓
[websocket_session_manager.py#L100-L168] WebsocketSessionManager.connect_session()
    ↓
    ├─ 检查 existing_session_id 是否在 _active_session_info_by_id 中
    │   （如果是，说明会话还活跃，忽略重连请求，创建新会话）
    │
    └─ 从 SessionStorage 查找
        session_info = self._session_storage.get(existing_session_id)
        ↓
        找到 → 恢复会话 ★
        未找到 → 创建新 AppSession
```

#### 恢复会话的关键步骤

[websocket_session_manager.py#L126-L140](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L126-L140)

```python
if isinstance(session_info, SessionInfo):
    existing_session = session_info.session  # 取出保存的 AppSession

    # ★ 恢复步骤1: 重新注册文件监听器（之前被 disconnect_file_watchers 关闭了）
    existing_session.register_file_watchers()
    # [app_session.py#L223-L246] 重新创建 LocalSourcesWatcher, config listener, secrets listener

    # ★ 恢复步骤2: 重新加入活跃会话表，关联新的 WebSocket client
    self._active_session_info_by_id[existing_session.id] = ActiveSessionInfo(
        client,                  # 新 WebSocket 连接
        existing_session,        # 原来的 AppSession（含原来的 SessionState！）
        session_info.script_run_count,
    )

    # ★ 恢复步骤3: 从存储中删除（避免重复使用）
    self._session_storage.delete(existing_session.id)

    with self._stats_lock:
        self._reconnect_count += 1
        self._session_connect_times[existing_session.id] = time.monotonic()
    return existing_session.id
```

**SessionState 的连续性**：由于 `MemorySessionStorage` 保存的是对象引用，恢复后 `existing_session._session_state` 就是断开前的同一个 `SessionState` 对象，所有 `_old_state`、`_key_id_mapper`、`query_params` 等数据完整保留。

#### 重连后首次脚本运行的连接点

重连后，前端会立即发送 `rerun_script` BackMsg，携带最新的 widget 状态：

```
重连成功 → 返回 session_id 给前端
    ↓
前端发送 BackMsg { rerun_script: ClientState { widget_states: {...} } }
    ↓
AppSession.handle_backmsg() → request_rerun()
    ↓
_create_scriptrunner(rerun_data)
    ↓   session_state = existing_session._session_state  ← 原来的对象！
    ↓
ScriptRunner._run_script_loop()
    ↓
on_script_will_rerun(latest_widget_states)  ← 与普通重跑完全相同的入口！
    ├─ _reset_triggers()
    ├─ _compact_state()        ← 合并断开前留下的状态
    ├─ set_widgets_from_proto(latest_widget_states)  ← 加载前端最新值
    └─ _call_callbacks()
    ↓
用户脚本执行 → 一切恢复正常
```

---

### 4B.3 两条路径的对比总结

| 维度 | 普通重跑 | 断线重连 |
|------|---------|---------|
| **触发方式** | 用户交互 widget → BackMsg.rerun_script | WebSocket 断开 → 重新连接 → existing_session_id |
| **AppSession** | 复用同一个实例 | 从 SessionStorage 恢复原来的实例 |
| **SessionState** | 始终是同一个对象 | 恢复后也是原来的对象（内存引用未变） |
| **ScriptRunner** | 复用或新建 | 必定新建（原 ScriptRunner 线程已终止） |
| **状态恢复入口** | `on_script_will_rerun()` | 同左，也是 `on_script_will_rerun()` |
| **_old_state 内容** | 上一次压缩的结果 | 断开前最后一次压缩的结果 |
| **_new_widget_state** | 空 → 加载前端最新状态 | 可能残留断开前的值 → _compact_state 合并 → 加载前端最新 |
| **额外操作** | 无 | 重新注册文件监听器、重新关联 client |
| **时间窗口** | 即时 | 需在 TTLCache TTL（默认 2 分钟）内重连 |

**共同连接点**：两条路径最终都汇聚到 `on_script_will_rerun()` 这个统一的状态恢复入口，保证了状态处理逻辑的一致性。

---

### 4B.4 连接点深度剖析：前端如何保存旧会话标识

断线重连的第一步是前端在重连时告诉服务端"我之前有个会话"。这涉及 session_id 在前端的存储和跨连接传递。

#### 前端 SessionInfo 的 last/current 双缓冲机制

[SessionInfo.ts](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/frontend/lib/src/SessionInfo.ts)

```typescript
export class SessionInfo {
  private _current?: Props    // 当前活跃会话
  private _last?: Props       // 上一次会话（断开后保留）

  public setCurrent(props?: Props): void {
    // 新会话到来时，旧 _current 被拷贝到 _last
    this._last = notNullOrUndefined(this._current)
      ? { ...this._current }
      : undefined
    this._current = notNullOrUndefined(props) ? { ...props } : undefined
  }

  public disconnect(): void {
    if (this._current) {
      // 断开时，标记 isConnected=false，但数据仍保留在 _last
      this.setCurrent({
        ...this._current,
        isConnected: false,
      })
    }
  }
}
```

**关键**：`sessionId` 存储在 `SessionInfo.current.sessionId` 中，断开后调用 `disconnect()` 会将 `isConnected` 设为 `false` 并将其移到 `_last`，但 `sessionId` 仍然保留。

[App.tsx#L565](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/frontend/app/src/App.tsx#L565)

```typescript
// ConnectionManager 构造时传入 getLastSessionId 回调
getLastSessionId: () => this.sessionInfo.last?.sessionId,
```

#### session_id 如何嵌入 WebSocket 连接请求

[WebsocketConnection.tsx#L504-L515](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/frontend/connection/src/WebsocketConnection.tsx#L504-L515)

```typescript
private async getSessionTokens(): Promise<Array<string>> {
  const hostAuthToken = await this.args.claimHostAuthToken()
  const xsrfCookie = getCookie("_streamlit_xsrf")
  this.args.resetHostAuthToken()

  // ★ 核心：从 SessionInfo.last 取出上一次的 sessionId
  const lastSessionId = this.args.getLastSessionId()

  return [
    // 第一个元素：认证 token 或 XSRF cookie 或占位符
    hostAuthToken ?? xsrfCookie ?? "PLACEHOLDER_AUTH_TOKEN",
    // 第二个元素（可选）：上一次的 sessionId，仅重连时存在
    ...(lastSessionId ? [lastSessionId] : []),
  ]
}
```

[WebsocketConnection.tsx#L543-L544](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/frontend/connection/src/WebsocketConnection.tsx#L543-L544)

```typescript
// 将 session tokens 嵌入 Sec-WebSocket-Protocol 头
const sessionTokens = await this.getSessionTokens()
this.websocket = new WebSocket(uri, ["streamlit", ...sessionTokens])
```

**请求格式**：`Sec-WebSocket-Protocol: streamlit, <auth_token>, <session_id>`

- 首次连接：`["streamlit", "PLACEHOLDER_AUTH_TOKEN"]` → 无第三个元素
- 重连时：`["streamlit", "auth_token", "old_session_id"]` → 第三个元素是旧 session_id

#### 后端如何解析

[starlette_websocket.py#L64-L88](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L64-L88)

```python
def _parse_subprotocols(headers: Headers) -> tuple[str|None, str|None, str|None]:
    raw = headers.get("sec-websocket-protocol")
    entries = [value.strip() for value in raw.split(",")]
    selected = entries[0] if entries and entries[0] else None       # "streamlit"
    xsrf_token = entries[1] if len(entries) >= 2 and entries[1] else None  # auth
    existing_session = entries[2] if len(entries) >= 3 and entries[2] else None  # ★
    return selected, xsrf_token, existing_session
```

**完整数据流**：

```
SessionInfo.last.sessionId (前端内存)
    ↓ getLastSessionId() 回调
    ↓ WebsocketConnection.getSessionTokens()
    ↓ WebSocket(uri, ["streamlit", token, sessionId])
    ↓ HTTP Sec-WebSocket-Protocol 头
    ↓ _parse_subprotocols(headers)
    ↓ existing_session_id
    ↓ runtime.connect_session(existing_session_id=...)
    ↓ SessionStorage.get(existing_session_id)
    ↓ 恢复 AppSession（含 SessionState）
```

---

### 4B.5 连接点深度剖析：重连后前端如何补发第一次重跑

重连建立后，前端必须主动向服务端发送第一次 rerun 请求，否则服务端不知道该重新执行脚本。

#### 触发入口：handleConnectionStateChanged

[App.tsx#L862-L903](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/frontend/app/src/App.tsx#L862-L903)

```typescript
handleConnectionStateChanged = (newState: ConnectionState): void => {
  if (newState === ConnectionState.CONNECTED) {
    // ★ 重连成功后的核心判断逻辑

    // 我们在以下情况请求 rerun：
    //   1. 这是首次建立连接（没有 last session）
    //   2. 上次脚本运行被断开中断（scriptRunState === RUNNING）
    //   3. host 显式请求了重连（scriptRunState === RERUN_REQUESTED）
    //   4. 脚本使用了 fragments（需要重新初始化）

    const lastRunWasInterrupted =
      this.state.scriptRunState === ScriptRunState.RUNNING
    const wasRerunRequested =
      this.state.scriptRunState === ScriptRunState.RERUN_REQUESTED

    if (
      !this.sessionInfo.last ||             // 条件1: 首次连接
      lastRunWasInterrupted ||               // 条件2: 运行被中断
      wasRerunRequested ||                   // 条件3: 显式请求重连
      this.state.fragmentIdsThisRun.length > 0 ||  // 条件4a: 有 fragment
      this.state.autoReruns.length > 0       // 条件4b: 有自动 rerun
    ) {
      LOG.info("Requesting a script run.")
      // ★ 发送 rerun：携带当前所有 widget 状态
      this.widgetMgr.sendUpdateWidgetsMessage(undefined)
      this.setState({ dialog: null })
    } else {
      // 无需 rerun，仅关闭连接错误弹窗
      this.setState({ dialog: null })
    }
  } else {
    // 断开连接时
    if (this.sessionInfo.isSet) {
      this.sessionInfo.disconnect()  // ← 保存 sessionId 到 _last
    }
  }
}
```

#### sendUpdateWidgetsMessage 携带的状态

[WidgetStateManager.ts#L831-L841](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/frontend/lib/src/WidgetStateManager.ts#L831-L841)

```typescript
public sendUpdateWidgetsMessage(
  fragmentId: string | undefined,
  isAutoRerun: boolean | undefined = undefined
): void {
  // ★ 收集当前所有 widget 状态，构造 BackMsg.rerun_script
  this.props.sendRerunBackMsg(
    this.widgetStates.createWidgetStatesMsg(),  // 所有 widget 的当前值
    fragmentId,
    undefined,    // pageScriptHash
    isAutoRerun
  )
}
```

[App.tsx#L1994-L2007](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/frontend/app/src/App.tsx#L1994-L2007)

```typescript
this.sendBackMsg(
  new BackMsg({
    rerunScript: {
      queryString,             // 当前 URL 查询参数
      widgetStates,            // ★ 前端所有 widget 的当前值（Protobuf）
      pageScriptHash,          // 当前页面哈希
      pageName,                // 页面名称
      fragmentId,              // fragment ID（如有）
      isAutoRerun,             // 是否自动 rerun
      cachedMessageHashes,     // 消息缓存哈希
      contextInfo,             // 时区、语言等上下文
    },
  })
)
```

#### 完整补发时序

```
WebSocket 断开
    ↓ sessionInfo.disconnect() → _last = { sessionId: "abc", isConnected: false }
    ↓
    ⏳ 网络恢复
    ↓
WebsocketConnection 重新建立连接
    ↓ Sec-WebSocket-Protocol: "streamlit, token, abc"  ← 传递旧 sessionId
    ↓ 服务端恢复 AppSession → 返回 NewSession 消息
    ↓
前端收到 NewSession → handleNewSession()
    ↓ sessionInfo.setCurrent() → _current = 新会话, _last = 旧会话
    ↓
ConnectionManager 状态变为 CONNECTED
    ↓ handleConnectionStateChanged(CONNECTED)
    ↓ 判断条件 → 需要补发 rerun
    ↓ widgetMgr.sendUpdateWidgetsMessage(undefined)
    ↓ sendRerunBackMsg(widgetStates=createWidgetStatesMsg())
    ↓
BackMsg { rerunScript: { widgetStates: {...}, queryString: "...", ... } }
    ↓
服务端 AppSession.handle_backmsg()
    ↓ _handle_rerun_script_request()
    ↓ request_rerun(client_state)  ← client_state 包含前端所有 widget 状态
    ↓
ScriptRunner._run_script_loop()
    ↓ on_script_will_rerun(latest_widget_states)  ← 恢复点！
```

**关键理解**：
- 前端重连后自动补发 rerun 的决策逻辑在 `handleConnectionStateChanged` 中
- 补发的 rerun 消息携带**前端当前所有 widget 状态**，确保服务端状态与前端一致
- 如果不需要 rerun（如只是短暂断开且脚本已完成），前端只关闭错误弹窗

---

### 4B.6 连接点深度剖析：源码热更新时复用上一份 ClientState

当用户修改了 Python 源码并开启了 `runOnSave` 时，Streamlit 会自动触发重跑。关键在于重跑时复用了上一份 `_client_state`，其中包含上一次的所有 widget 值。

#### 热更新触发入口

[app_session.py#L538-L550](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/app_session.py#L538-L550)

```python
def _on_source_file_changed(self, filepath: str | None = None) -> None:
    """源文件变化时调用。清除缓存并根据设置触发 rerun。"""
    self._script_cache.clear()  # 清除脚本字节码缓存

    if filepath is not None and not self._should_rerun_on_file_change(filepath):
        return  # 变化的文件不是当前页面，跳过

    if self._run_on_save:
        # ★ 核心：传入 self._client_state 复用上一次的完整客户端状态
        self.request_rerun(self._client_state)
    else:
        # 未开启 runOnSave，仅通知前端文件已变化
        self._enqueue_forward_msg(self._create_file_change_message())
```

#### _client_state 是什么

[app_session.py#L174](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/app_session.py#L174)

```python
# 在 AppSession.__init__ 中初始化
self._client_state = ClientState()  # 空 ClientState
```

`_client_state` 是一个 Protobuf 对象，包含以下关键字段：

```
ClientState {
  widget_states: WidgetStates    ← 前端所有 widget 的当前值
  query_string: string           ← 当前 URL 查询参数
  page_script_hash: string       ← 当前页面脚本哈希
  page_name: string              ← 当前页面名称
  fragment_id: string            ← fragment ID（如有）
  is_auto_rerun: bool            ← 是否自动 rerun
  cached_message_hashes: ...     ← 消息缓存哈希
  context_info: ContextInfo      ← 时区、语言等
}
```

#### _client_state 的更新时机

`_client_state` 在两个地方被更新：

1. **前端发送 rerun 请求时**，`request_rerun(client_state)` 更新 `_client_state` 的部分字段：

[app_session.py#L450-L451](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/app_session.py#L450-L451)

```python
if client_state.HasField("context_info"):
    self._client_state.context_info.CopyFrom(client_state.context_info)
```

2. **ScriptRunner SHUTDOWN 事件时**，完整替换 `_client_state`：

[app_session.py#L752](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/app_session.py#L752)

```python
elif event == ScriptRunnerEvent.SHUTDOWN:
    # ...
    self._client_state = client_state  # ★ 完整替换为最新的 client_state
```

#### 热更新路径与普通重跑的对比

| 维度 | 普通重跑 | 源码热更新 |
|------|---------|-----------|
| **触发** | 前端 `BackMsg.rerun_script` | 服务端 `_on_source_file_changed` |
| **ClientState 来源** | 前端实时构造（含最新 widget 值） | 服务端缓存的 `self._client_state` |
| **Widget 状态** | 前端当前所有 widget 值（经过用户交互） | **上一次前端发来的** widget 值 |
| **request_rerun 入参** | `client_state`（来自前端） | `self._client_state`（来自缓存） |
| **后续路径** | `on_script_will_rerun()` | 同左 |
| **_compact_state 行为** | 相同 | 相同 |

#### 热更新时的完整数据流

```
源码文件变化
    ↓ LocalSourcesWatcher 检测到变化
    ↓ _on_source_file_changed(filepath)
    ↓ _script_cache.clear()
    ↓ _should_rerun_on_file_change(filepath) → True
    ↓ self._run_on_save → True
    ↓
request_rerun(self._client_state)  ★ 复用缓存的 ClientState
    ↓
    ├─ 分支A: 复用现有 ScriptRunner
    │   request_rerun(rerun_data)
    │   ↓ rerun_data = RerunData(
    │       widget_states=self._client_state.widget_states,  ← 缓存的 widget 值
    │       query_string=self._client_state.query_string,    ← 缓存的 URL 参数
    │       page_script_hash=self._client_state.page_script_hash,
    │       ...
    │   )
    │
    └─ 分支B: 新建 ScriptRunner
        _create_scriptrunner(initial_rerun_data=rerun_data)
        ↓ 同样使用缓存的 widget_states

    ↓
ScriptRunner._run_script_loop()
    ↓
on_script_will_rerun(rerun_data.widget_states)  ← 用缓存的 widget 值恢复状态
    ├─ _compact_state()       合并旧状态
    ├─ set_widgets_from_proto() 加载缓存的 widget 值
    └─ _call_callbacks()      执行值变化回调
    ↓
用户脚本执行（使用上一次的 widget 状态 + 新代码）
```

**关键理解**：
- 热更新复用的 `_client_state` 是**上一次前端发来的 widget 状态快照**
- 如果用户在两次前端交互之间修改了源码，热更新会使用**最近一次前端发送的** widget 值，而非实时值
- 这意味着用户修改源码后自动 rerun 时，所有 widget 保持上次的值，用户不会丢失交互状态
- 如果未开启 `runOnSave`，前端仅收到一个 `script_changed_on_disk` 通知，由用户决定是否手动 rerun

---

### 4.3 Widget 状态的 Proto 序列化（前后端通信）

Widget 状态的 Protobuf 序列化用于**前后端通信**，与会话级持久化是不同层面的机制：

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
    elif field == "file_uploader_state_value":
        widget.file_uploader_state_value.CopyFrom(serialized)
    # ... 其他类型处理

    return widget
```

---

## 5. 状态恢复时机（补充：运行后清理与反序列化）

### 5.1 脚本运行完成后的状态清理

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

### 5.2 前端 Widget 状态的反序列化（懒加载）

从前端接收 Protobuf 状态后反序列化为 Python 对象，采用**懒加载**策略：首次访问时才反序列化并缓存结果。

[session_state.py#L151-L203](file:///d:/fz/0601/solo-dogfeeding/code/214-streamlit/lib/streamlit/runtime/state/session_state.py#L151-L203)

```python
def __getitem__(self, k: str) -> Any:
    wstate = self.states.get(k)
    if isinstance(wstate, Value):
        return wstate.value  # 已反序列化，直接返回

    # Serialized(Protobuf) -> Value(Python对象) 懒加载转换
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

## 6. 完整生命周期时序（含两条路径）

```
                           新客户端连接
                                ↓
                    AppSession 初始化 → 创建 SessionState (空)
                                ↓
                          首次脚本运行
                                ↓
        ┌───────────────────────────────────────────────────────┐
        │                                                       │
        │            ═══ 路径A: 普通重跑 ═══                    │
        │                                                       │
        │  用户交互 widget → BackMsg.rerun_script              │
        │        ↓                                              │
        │  AppSession.request_rerun(client_state)              │
        │        ↓                                              │
        │  ScriptRunner._run_script_loop()                     │
        │        ↓                                              │
        │  on_script_will_rerun(latest_widget_states) ★        │
        │    ├─ _reset_triggers()                               │
        │    ├─ _compact_state()     新状态 → _old_state       │
        │    ├─ set_widgets_from_proto() 前端最新状态加载        │
        │    └─ _call_callbacks()       值变化回调执行          │
        │        ↓                                              │
        │  用户脚本执行 → 注册 widget → 访问/修改 state        │
        │        ↓                                              │
        │  maybe_check_serializable()  序列化预检              │
        │        ↓                                              │
        │  _on_script_finished()                                │
        │    └─ on_script_finished()  清理 stale widget        │
        │                                                       │
        └───────────────────────────────────────────────────────┘
                                ↓
                    WebSocket 连接断开
                                ↓
                    runtime.disconnect_session()
                                ↓
        ┌───────────────────────────────────────────────────────┐
        │  websocket_session_manager.disconnect_session()       │
        │    ├─ session.request_script_stop()                   │
        │    ├─ session.disconnect_file_watchers()              │
        │    ├─ session.clear_session_caches()                  │
        │    └─ _session_storage.save(SessionInfo)              │
        │         ↓ TTLCache, key=session_id, TTL=2分钟         │
        │         内存引用（非序列化）                           │
        └───────────────────────────────────────────────────────┘
                                ↓
                    ⏳ 时间窗口：2 分钟内
                                ↓
                           客户端重连
                                ↓
        ┌───────────────────────────────────────────────────────┐
        │                                                       │
        │            ═══ 路径B: 断线重连 ═══                    │
        │                                                       │
        │  Sec-WebSocket-Protocol: "streamlit, token, session_id"
        │        ↓                                              │
        │  _parse_subprotocols() → existing_session_id         │
        │        ↓                                              │
        │  runtime.connect_session(existing_session_id=...)    │
        │        ↓                                              │
        │  websocket_session_manager.connect_session()         │
        │    └─ session_info = _session_storage.get(id)        │
        │         ↓ 找到？                                      │
        │         ├─ 是 → 恢复会话 ★                            │
        │         │    ├─ existing_session.register_file_watchers()
        │         │    ├─ _active_session_info_by_id[id] =     │
        │         │    │    ActiveSessionInfo(新client, 原AppSession)
        │         │    └─ _session_storage.delete(id)           │
        │         └─ 否 → 创建新 AppSession + SessionState      │
        │                                                       │
        └───────────────────────────────────────────────────────┘
                                ↓
                   前端发送 rerun_script BackMsg
                                ↓
                   进入路径A → on_script_will_rerun() ★
                   （两条路径汇聚到同一个状态恢复入口）
```

**关键汇聚点 `★`**：
- `on_script_will_rerun()` 是两条路径共同的状态恢复入口
- 断线重连后，SessionState 对象本身已完整恢复（通过内存引用）
- `_compact_state()` 将断开前残留的 `_new_widget_state`/`_new_session_state` 合并到 `_old_state`
- `set_widgets_from_proto()` 用前端最新 widget 状态覆盖，保证前后端一致

---

## 7. 关键设计要点

1. **三层状态结构**：`_old_state`（持久化）、`_new_session_state`（用户设置）、`_new_widget_state`（前端传来），通过 `_compact_state` 在每次运行前合并，保证状态一致性。

2. **懒加载反序列化**：Widget 状态从前端接收时保持 Protobuf 格式，首次访问时才反序列化为 Python 对象，提升性能。

3. **会话级持久化**：断开连接时保存整个 `AppSession`（包含 `SessionState`），重连时完整恢复，实现无缝的用户体验。

4. **Stale Widget 清理**：每次运行结束后清理未出现的 widget 状态，防止内存泄漏，同时保留查询参数绑定的 widget 值。

5. **线程安全**：`SafeSessionState` 使用 `RLock` 保护所有状态访问，支持脚本被中断后新脚本立即运行的场景。
