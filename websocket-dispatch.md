# Streamlit WebSocket 消息分发完整链路

本文档梳理 Streamlit 中基于 Tornado/Starlette 的 WebSocket 消息分发处理路径，涵盖连接建立、消息编解码、广播分发和前端消费四大环节。

---

## 一、总体架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Frontend Browser                            │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────────────┐   │
│  │     App      │──▶│ConnectionManager│──▶│  WebsocketConnection  │   │
│  │ handleMessage│◀──│   onMessage    │◀──│  handleMessage/cache  │   │
│  └──────────────┘   └────────────────┘   └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ WebSocket (binary protobuf)
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Starlette Server                              │
│  ┌────────────────────┐  ┌───────────────┐  ┌───────────────────┐  │
│  │_websocket_endpoint │  │StarletteSess..│  │ WebsocketSession..│  │
│  │ (accept/receive)   │─▶│ write_forward..│─▶│ connect/disconnect │  │
│  │ receive_bytes      │  │  _sender task  │  │                   │  │
│  └────────────────────┘  └───────────────┘  └───────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Runtime / AppSession                         │
│  ┌──────────────────┐   ┌───────────────────┐  ┌─────────────────┐  │
│  │ Runtime._loop_.. │   │ AppSession        │  │ ForwardMsgQueue │  │
│  │ (flush & dispatch)│◀──│_enqueue_forward..│─▶│ enqueue/flush   │  │
│  └──────────────────┘   └───────────────────┘  └─────────────────┘  │
│           │                                                         │
│           ▼                                                         │
│  ┌────────────────────────────┐    ┌────────────────────────────┐   │
│  │   ScriptRunContext.enqueue │    │  DeltaGenerator._enqueue    │   │
│  │   (hash/cache ref)         │◀───│  (st.write/st.button/...)   │   │
│  └────────────────────────────┘    └────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、连接建立链路

### 2.1 前端连接发起

前端从 `App` 组件启动，经由 `ConnectionManager` 到 `WebsocketConnection` 建立 WebSocket。

**关键文件：**
- [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/app/src/App.tsx#L564-L570)
- [ConnectionManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/connection/src/ConnectionManager.ts#L89-L109)
- [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/connection/src/WebsocketConnection.tsx#L517-L590)

**状态机：**
```
INITIAL ──INITIALIZED──▶ PINGING_SERVER ──SERVER_PING_SUCCEEDED──▶ CONNECTING
                                                           │
                                               CONNECTION_SUCCEEDED
                                                           ▼
                                                        CONNECTED
```

连接建立细节：
1. `ConnectionManager` 构造时调用 `connect()`，根据是否为静态应用决定连接类型
2. `WebsocketConnection` 使用 FSM（有限状态机）管理连接状态
3. 默认先通过 HTTP ping 检测服务端健康，再进入 `CONNECTING`
4. `connectToWebSocket()` 中创建原生 WebSocket，路径为 `_stcore/stream`
5. **利用 `Sec-WebSocket-Protocol` 头部传递认证令牌和 session ID**（浏览器不允许自定义 HTTP header）

```typescript
// WebsocketConnection.tsx L543-L545
const sessionTokens = await this.getSessionTokens()
this.websocket = new WebSocket(uri, ["streamlit", ...sessionTokens])
this.websocket.binaryType = "arraybuffer"
```

### 2.2 后端连接接受

**关键文件：**
- [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L62)
- [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L359-L555)

路由注册：
- `create_websocket_routes()` 在 Starlette 应用中注册 WebSocket 路由
- 路径为 `/{base_url}/_stcore/stream`

`_websocket_endpoint` 处理流程：
1. **Origin 校验**：`_is_origin_allowed()` 防止跨站 WebSocket 劫持
2. **Subprotocol 解析**：`_parse_subprotocols()` 从 `Sec-WebSocket-Protocol` 提取 (streamlit, xsrf_token, existing_session_id)
3. **接受连接**：`websocket.accept(subprotocol=subprotocol)`
4. **创建 StarletteSessionClient**：包装 WebSocket，提供同步写入接口
5. **用户认证**：XSRF token 验证 + Cookie 解析 + 可信 Header 提取
6. **注册会话**：`runtime.connect_session()` → `WebsocketSessionManager.connect_session()`
7. **进入消息循环**：`while True: websocket.receive_bytes()`

### 2.3 会话管理器

**关键文件：**
- [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L59-L169)

`WebsocketSessionManager.connect_session()` 处理：
- 若 `existing_session_id` 有效且不在活跃列表中，从 `SessionStorage` 恢复会话（重连场景）
- 否则创建新的 `AppSession`，分配唯一 session ID
- 记录到 `_active_session_info_by_id` 字典

---

## 三、消息编解码（Protobuf）

### 3.1 消息协议定义

**关键文件：**
- [ForwardMsg.proto](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/proto/streamlit/proto/ForwardMsg.proto#L36-L160)
- [BackMsg.proto](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/proto/streamlit/proto/BackMsg.proto#L23-L87)

**ForwardMsg（后端→前端）** 消息类型：
| 类型 | 用途 |
|------|------|
| `new_session` | 会话初始化，携带配置、主题、页面信息 |
| `delta` | UI 增量更新（new_element/add_block/new_transient 等） |
| `script_finished` | 脚本执行结束状态 |
| `session_status_changed` | 运行状态变化（run_on_save、script_is_running） |
| `session_event` | 编译异常、文件变更等事件 |
| `page_config_changed` | 页面配置（标题、布局、favicon） |
| `ref_hash` | 引用已缓存消息（减少重复传输） |
| `heartbeat_ack` | 心跳响应 |
| `backend_operation_response` | 后端操作响应（延迟文件下载等） |

**BackMsg（前端→后端）** 消息类型：
| 类型 | 用途 |
|------|------|
| `rerun_script` | 请求重新运行脚本（携带 widget 状态、页面 hash） |
| `stop_script` | 停止当前脚本执行 |
| `clear_cache` | 清空缓存 |
| `app_heartbeat` | 心跳保活 |
| `file_urls_request` | 文件上传/下载 URL 请求 |
| `backend_operation_request` | 无脚本重跑的后端操作 |

### 3.2 后端编码

**关键文件：**
- [runtime_util.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/runtime_util.py#L72-L96)

`serialize_forward_msg(msg)` 流程：
1. 调用 `msg.SerializeToString()` 序列化为二进制
2. 检查是否超过 `server.maxMessageSize`（默认 200MB）
3. 若超限，替换为 `MessageSizeError` 异常消息
4. 返回最终二进制 bytes

### 3.3 后端解码

**关键文件：**
- [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L471-L480)

```python
back_msg = BackMsg()
back_msg.ParseFromString(data)  # 从二进制解析 BackMsg
msg_type = back_msg.WhichOneof("type")  # 确定具体消息类型
```

### 3.4 前端编码

**关键文件：**
- [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/connection/src/WebsocketConnection.tsx#L665-L679)

```typescript
sendMessage(obj: IBackMsg): void {
  const msg = BackMsg.create(obj)
  const buffer = BackMsg.encode(msg).finish()
  const encodedMessage = new Uint8Array(
    buffer.buffer as ArrayBuffer,
    buffer.byteOffset,
    buffer.byteLength
  )
  this.websocket.send(encodedMessage)
}
```

### 3.5 前端解码 & 消息缓存

**关键文件：**
- [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/connection/src/WebsocketConnection.tsx#L699-L723)
- [ForwardMessageCache.ts](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/connection/src/ForwardMessageCache.ts#L49-L217)

```typescript
private async handleMessage(data: ArrayBuffer): Promise<void> {
  const messageIndex = this.nextMessageIndex++
  const encodedMsg = new Uint8Array(data)
  const msg = ForwardMsg.decode(encodedMsg)   // Protobuf 解码

  // 经过 ForwardMsgCache 处理（缓存 / 解引用 ref_hash）
  this.messageQueue[messageIndex] = await this.cache.processMessagePayload(msg, encodedMsg)

  // 按顺序派发（保证消息有序性）
  while (this.lastDispatchedMessageIndex + 1 in this.messageQueue) {
    const dispatchMessageIndex = this.lastDispatchedMessageIndex + 1
    this.args.onMessage(this.messageQueue[dispatchMessageIndex])
    delete this.messageQueue[dispatchMessageIndex]
    this.lastDispatchedMessageIndex = dispatchMessageIndex
  }
}
```

ForwardMsgCache 工作原理：
- 若 `msg.metadata.cacheable` 为 true 且有 hash，则缓存该消息
- 若收到 `ref_hash` 类型消息，从缓存中取出原消息并替换 metadata
- 通过 `scriptRunCount` 管理缓存过期，超过最大年龄的消息被清理

---

## 四、后端消息广播分发（ForwardMsg 下行链路）

### 4.1 消息产生：从 DeltaGenerator 到 ForwardMsg

**关键文件：**
- [delta_generator.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/delta_generator.py#L473-L572)

当用户代码调用 `st.write("hello")`、`st.button("Click")` 等 API 时：

```
st.button(...)
  ▼
ButtonMixin.button()  # 各 Mixin 实现
  ▼
self._enqueue("button", button_proto)  # DeltaGenerator._enqueue
  ▼
构造 ForwardMsg():
  - msg.delta.new_element.button = button_proto
  - msg.metadata.delta_path = cursor.delta_path  # UI 树位置
  ▼
_enqueue_message(msg)  # from scriptrunner_utils
```

### 4.2 ScriptRunContext：Hash 计算与缓存引用

**关键文件：**
- [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L312-L327)
- [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L468-L479)

`ScriptRunContext.enqueue(msg)`：
1. 设置 `msg.metadata.active_script_hash`（用于多页面追踪）
2. `populate_hash_if_needed(msg)`：计算消息内容 hash（用于去重缓存）
3. 若消息可缓存且 hash 在 `cached_message_hashes`（前端已收到的消息集合）中：
   - 创建引用消息 `create_reference_msg(msg)`，只发送 hash，不发实际内容
4. 调用 `self._enqueue(msg)` → 实际绑定到 `AppSession._enqueue_forward_msg`

### 4.3 AppSession：入队 ForwardMsgQueue

**关键文件：**
- [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/app_session.py#L318-L337)
- [forward_msg_queue.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/forward_msg_queue.py#L25-L264)

`AppSession._enqueue_forward_msg(msg)`：
1. 设置 `debug_last_backmsg_id`（测试用）
2. `self._browser_queue.enqueue(msg)` → `ForwardMsgQueue.enqueue()`
3. 触发 `message_enqueued_callback()` → `Runtime._enqueued_some_message()` 设置 `need_send_data` Event

**ForwardMsgQueue 消息合并优化：**
- 维护 `_delta_index_map: {(delta_path) -> queue_index}`
- 对可组合消息（非 `new_transient` 的 Delta）：
  - 若同路径已有消息在队列中，尝试 `_maybe_compose_delta_msgs()` 合并
  - 如 `st.empty()` → `st.markdown()` 同路径时，跳过前者直接用后者
- 特殊规则：`add_block` 类型永不合并（避免子节点引用失效）

### 4.4 Runtime 主循环：Flush & Dispatch

**关键文件：**
- [runtime.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/runtime.py#L624-L699)

Runtime 的 `_loop_coroutine()` 是消息分发的心脏：

```python
while not async_objs.must_stop.is_set():
    if self._state == ONE_OR_MORE_SESSIONS_CONNECTED:
        async_objs.need_send_data.clear()

        # 遍历所有活跃会话
        for active_session_info in self._session_mgr.list_active_sessions():
            # ① 取出该会话队列中所有待发消息
            msg_list = active_session_info.session.flush_browser_queue()
            for msg in msg_list:
                try:
                    # ② 发送到该会话对应的客户端
                    self._send_message(active_session_info, msg)
                except SessionClientDisconnectedError:
                    self._session_mgr.disconnect_session(...)
                await asyncio.sleep(0)  # 让出事件循环

        await asyncio.sleep(MESSAGE_FLUSH_INTERVAL_SECS)  # 0.001 秒

    # 等待新消息到来或停止信号
    await asyncio.wait([must_stop.wait(), need_send_data.wait()], ...)
```

### 4.5 StarletteSessionClient：同步→异步桥接

**关键文件：**
- [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L265-L357)

由于 Runtime 运行在 asyncio 事件循环中，而消息产生可能来自脚本线程，需要桥接：

```
Runtime._send_message()
  ▼
StarletteSessionClient.write_forward_msg(msg)
  ├─ serialize_forward_msg(msg) → bytes
  └─ self._send_queue.put_nowait(payload)  # asyncio.Queue（有界）
         │
         ▼  后台 _sender() 协程
      await self._websocket.send_bytes(payload)
```

- `_send_queue` 限制最大积压量（`WEBSOCKET_MAX_SEND_QUEUE_SIZE`），防止客户端消费过慢导致内存泄漏
- 队列满或 WebSocket 断开时抛出 `SessionClientDisconnectedError`，触发会话断开

### 4.6 ScriptRunner 事件 → ForwardMsg

除了 UI Delta，ScriptRunner 执行状态变化也会产生 ForwardMsg：

**关键文件：**
- [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/app_session.py#L569-L767)

`AppSession._on_scriptrunner_event()` 从脚本线程通过 `call_soon_threadsafe` 切到事件循环线程：

| ScriptRunnerEvent | 产生的 ForwardMsg |
|-------------------|-------------------|
| `SCRIPT_STARTED` | `new_session`（配置/主题/页面）+ `session_status_changed` |
| `SCRIPT_STOPPED_WITH_SUCCESS` | `script_finished(FINISHED_SUCCESSFULLY)` + `session_status_changed` |
| `SCRIPT_STOPPED_WITH_COMPILE_ERROR` | `script_finished(FINISHED_WITH_COMPILE_ERROR)` + `session_event(exception)` |
| `SCRIPT_STOPPED_FOR_RERUN` | `script_finished(FINISHED_EARLY_FOR_RERUN)` |
| `FRAGMENT_STOPPED_WITH_SUCCESS` | `script_finished(FINISHED_FRAGMENT_RUN_SUCCESSFULLY)` |
| `ENQUEUE_FORWARD_MSG` | 直接透传传入的 ForwardMsg |

---

## 五、前端消息消费链路

### 5.1 ConnectionManager 到 App

**关键文件：**
- [ConnectionManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/connection/src/ConnectionManager.ts#L89-L336)
- [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/app/src/App.tsx#L968-L1042)

```
WebsocketConnection.handleMessage()
  ├─ ForwardMsg.decode()     // Protobuf 解码
  ├─ ForwardMsgCache 处理    // 缓存 / ref_hash 解引用
  └─ 保序队列 onMessage()
         ▼
ConnectionManager.props.onMessage
         ▼
App.handleMessage(msgProto)
```

### 5.2 App.handleMessage 消息分发

**关键文件：**
- [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/app/src/App.tsx#L968-L1042)

`dispatchProto` 根据 `ForwardMsg.type` 分发到对应处理器：

| ForwardMsg 类型 | 处理函数 | 作用 |
|-----------------|----------|------|
| `newSession` | `handleNewSession` | 初始化 AppRoot、配置、主题、页面列表 |
| `delta` | `handleDeltaMsg` | 更新 AppRoot 渲染树（核心 UI 更新） |
| `scriptFinished` | `handleScriptFinished` | 更新运行状态、清理缓存、触发完成回调 |
| `sessionStatusChanged` | `handleSessionStatusChanged` | 更新 script_is_running、run_on_save |
| `sessionEvent` | `handleSessionEvent` | 处理脚本编译异常、文件变更提示 |
| `pageConfigChanged` | `handlePageConfigChanged` | 标题、favicon、布局、菜单项 |
| `pageInfoChanged` | `handlePageInfoChanged` | 更新 URL query params |
| `heartbeatAck` | `handleHeartbeatAck` | 清除心跳超时定时器 |
| `backendOperationResponse` | `backendOperationClient.onResponse` | 延迟文件下载等后端操作 |
| `authRedirect` | - | 重定向到认证 URL |

### 5.3 Delta 渲染树更新

`handleDeltaMsg` 将 Delta 应用到 `AppRoot`（不可变渲染树），最终触发 React 重渲染：
- `delta_path` 定位 UI 树中的节点位置
- 支持 `add_block`（新增容器）、`new_element`（新增元素）、`new_transient`（临时元素如 spinner）

---

## 六、前端→后端 BackMsg 上行链路

### 6.1 触发来源

常见触发场景：
- 用户点击按钮 / 修改组件值 → `WidgetStateManager.sendRerunBackMsg()`
- 用户点击 Stop → `App.stopScript()`
- 文件上传 → `FileUploadClient` 请求 URL
- 心跳定时发送 → `ConnectionManager.onHeartbeatSent()`

### 6.2 编码与发送

```
WidgetStateManager / App 等
  ▼
BackMsg.create({ rerun_script: { widget_states, page_script_hash, ... } })
  ▼
ConnectionManager.sendMessage()
  ▼
WebsocketConnection.sendMessage()
  ├─ BackMsg.encode(msg).finish()  // Protobuf 编码
  └─ websocket.send(Uint8Array)    // 二进制发送
```

### 6.3 后端接收与处理

**关键文件：**
- [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L455-L510)
- [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/app_session.py#L338-L376)

```
websocket.receive_bytes()
  ▼
BackMsg.ParseFromString(data)
  ▼
back_msg.WhichOneof("type")  // 确定类型
  ▼
runtime.handle_backmsg(session_id, back_msg)
  ▼
AppSession.handle_backmsg(msg)
```

| BackMsg 类型 | AppSession 处理函数 | 效果 |
|--------------|---------------------|------|
| `rerun_script` | `_handle_rerun_script_request` → `request_rerun` | 创建/重启 ScriptRunner，使用新的 widget 状态执行脚本 |
| `stop_script` | `_handle_stop_script_request` → `request_script_stop` | 中断当前脚本执行 |
| `clear_cache` | `_handle_clear_cache_request` | 清空 data/resource 缓存 |
| `app_heartbeat` | `_handle_app_heartbeat_request` | 回发 `heartbeat_ack` ForwardMsg |
| `file_urls_request` | `_handle_file_urls_request` | 生成上传/下载 URL 并回发 |
| `backend_operation_request` | `_handle_backend_operation_request` (async task) | 延迟文件下载等无需重跑脚本的操作 |
| `set_run_on_save` | `_handle_set_run_on_save_request` | 修改自动运行配置 |
| `load_git_info` | `_handle_git_information_request` | 加载 Git 信息 |

---

## 七、完整时序图（以用户点击按钮触发重跑为例）

```
  Browser (Frontend)                          Starlette/Runtime (Backend)
        │                                               │
        │  用户点击 st.button                           │
        │                                               │
        │ WidgetStateManager.sendRerunBackMsg()         │
        │   └─ BackMsg{rerun_script: ClientState}       │
        │                                               │
        │──── WebSocket binary (BackMsg) ──────────────▶│
        │                                               │
        │                                   _websocket_endpoint.receive_bytes()
        │                                   BackMsg.ParseFromString()
        │                                   runtime.handle_backmsg()
        │                                   AppSession.handle_backmsg()
        │                                     └─ request_rerun(client_state)
        │                                        ScriptRunner 启动（新线程）
        │                                               │
        │                                               │  执行用户脚本
        │                                               │  st.write(...) → DeltaGenerator._enqueue
        │                                               │  enqueue_message → ScriptRunContext.enqueue
        │                                               │    (hash 计算 / ref_hash 优化)
        │                                               │  AppSession._enqueue_forward_msg()
        │                                               │    ForwardMsgQueue.enqueue()
        │                                               │    (消息合并优化)
        │                                               │  ScriptRunner 事件 → _on_scriptrunner_event
        │                                               │
        │                                   Runtime._loop_coroutine():
        │                                     flush_browser_queue()
        │                                     for msg: _send_message()
        │                                               │
        │◀──── WebSocket binary (ForwardMsg) ──────────│
        │                                               │
        │ WebsocketConnection.handleMessage()          │
        │   ForwardMsg.decode()                        │
        │   ForwardMsgCache (ref_hash 解引用)          │
        │   保序队列 → onMessage                       │
        │                                               │
        │ App.handleMessage():                          │
        │   ├─ newSession → 初始化配置                  │
        │   ├─ delta → AppRoot 增量更新                 │
        │   └─ script_finished → 标记运行结束           │
        │                                               │
        │ React 重渲染 UI                               │
        ▼                                               ▼
```

---

## 八、关键设计要点

1. **全双工二进制协议**：严格使用 WebSocket binary frame + Protobuf，拒绝 text frame
2. **Subprotocol 复用**：利用 `Sec-WebSocket-Protocol` 传递 XSRF token 和 session ID（浏览器 API 限制）
3. **同步/异步桥接**：`StarletteSessionClient` 使用 asyncio.Queue + 后台 sender task 连接 Runtime 的同步调用与 WebSocket 的异步发送
4. **消息保序**：前端 `messageQueue` + `lastDispatchedMessageIndex` 保证即使异步解码也按接收顺序派发
5. **增量去重**：
   - 后端 `ForwardMsgQueue` 合并同 `delta_path` 的冗余 Delta
   - 前后端 `ref_hash` 机制避免重复发送相同内容
6. **背压保护**：发送队列有界，客户端消费过慢会触发断开
7. **线程安全**：ScriptRunner 线程通过 `event_loop.call_soon_threadsafe()` 切回事件循环操作 AppSession
8. **会话生命周期**：区分 active（WebSocket 已连接）和 inactive（断开暂存于 SessionStorage），支持短时间重连恢复
