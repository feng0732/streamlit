# Streamlit WebSocket 消息分发完整链路

> **重要澄清**：当前 Streamlit 版本已**完全移除 Tornado**，所有 Web 服务和 WebSocket 处理均基于 **Starlette（Web 框架）+ uvicorn（ASGI 服务器）**。本文档中所有提及 Tornado 的地方均为历史遗留描述，实际技术栈如下所述。

---

## 一、服务端技术栈澄清

### 1.1 Tornado 的历史与现状

**代码搜索结论**：
- 代码库中**不存在任何 Tornado 相关 import**（`import tornado` / `from tornado` 等）
- 不存在 `tornado_websocket.py` 或 `tornado_server.py` 等文件
- `config.py` 中没有 Tornado 相关配置项（如 `server.backend`）
- 路径 `lib/streamlit/web/server/` 下仅存在 `starlette/` 子目录，无 `tornado/` 目录

**结论**：Tornado 作为早期 Streamlit 使用的 Web 服务器，已被完整移除，当前代码库 100% 使用 Starlette + uvicorn 技术栈。

### 1.2 当前技术栈分层

```
┌──────────────────────────────────────────────────────────────┐
│                 Streamlit Runtime / AppSession               │
│  (WebSocket 消息分发、会话管理、脚本执行、ForwardMsg 队列)     │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                     Starlette (Web Framework)                │
│  - WebSocket 路由注册（WebSocketRoute）                       │
│  - WebSocket 连接握手（accept/receive_bytes/send_bytes）     │
│  - HTTP 路由与中间件（认证、Session、GZip、安全检查）          │
│  - WebSocket 端点处理函数（_websocket_endpoint）              │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                      uvicorn (ASGI Server)                   │
│  - TCP 套接字绑定与监听                                       │
│  - HTTP/1.1、WebSocket 协议解析（websockets-sansio 实现）    │
│  - WebSocket 心跳（ping/pong）管理                           │
│  - 事件循环调度（基于 asyncio）                               │
│  - SSL/TLS 终止                                              │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
                        操作系统 TCP/IP 栈
```

**关键文件与代码证据**：
- 服务器启动入口：[server.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/server.py#L51-L95)
  ```python
  # Server.__init__: 不直接创建 Tornado，而是在 start() 中创建 UvicornServer
  self._starlette_server: UvicornServer | None = None
  ```

- UvicornServer 包装：[starlette_server.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_server.py#L291-L485)
  ```python
  class UvicornServer:
      async def start(self) -> None:
          app = create_starlette_app(self._runtime)
          self._server = uvicorn.Server(uvicorn_config)
          # 手动绑定 socket，通过 startup(sockets=[socket]) 启动
  ```

- Starlette 应用创建：[starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L208-L245)
  ```python
  def create_starlette_app(runtime: Runtime) -> Starlette:
      routes = create_streamlit_routes(runtime)  # 包含 WebSocket 路由
      middleware = create_streamlit_middleware()
      return Starlette(routes=routes, middleware=middleware, lifespan=_lifespan)
  ```

- WebSocket 路由注册：[starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L527-L555)
  ```python
  def create_websocket_routes(runtime: Runtime, base_url: str | None) -> list[BaseRoute]:
      return [WebSocketRoute(path, create_websocket_handler(runtime))]
  ```

### 1.3 各环节框架归属对照表

| 处理环节 | 负责组件 | 框架归属 |
|---------|---------|---------|
| **连接接收**（TCP 握手、协议解析） | uvicorn | ASGI 服务器 |
| **WebSocket 握手**（Upgrade、Origin 校验） | `_websocket_endpoint` | Starlette |
| **消息接收**（二进制帧接收） | `websocket.receive_bytes()` | Starlette (封装 uvicorn) |
| **消息解码**（Protobuf ParseFromString） | Streamlit 自定义代码 | 业务逻辑层 |
| **消息分发**（Runtime -> AppSession） | `Runtime.handle_backmsg()` | 业务逻辑层 |
| **消息入队**（ForwardMsgQueue） | `AppSession._enqueue_forward_msg()` | 业务逻辑层 |
| **消息广播**（Runtime flush loop） | `Runtime._loop_coroutine()` | 业务逻辑层 |
| **消息编码**（Protobuf SerializeToString） | `serialize_forward_msg()` | 业务逻辑层 |
| **消息发送**（二进制帧发送） | `StarletteSessionClient.write_forward_msg()` | Starlette 桥接层 |

### 1.4 服务端启动完整调用链

```
streamlit run 命令
    │
    ▼
[cli.py]  main() -> _main_run()
    │
    ▼
[bootstrap.py]  run()  [L362-L428]
    ├─ 设置 config._server_mode = "starlette-managed"
    └─ 创建 Server(main_script_path, is_hello)
        │
        ▼
[server.py]  Server.__init__  [L51-L76]
    └─ 创建 Runtime（包含 WebSocketSessionManager）
        │
        ▼
[bootstrap.py]  run_server() 协程  [L384-L400]
    └─ await server.start()
        │
        ▼
[server.py]  Server.start()  [L84-L95]
    └─ self._starlette_server = UvicornServer(self._runtime)
       await self._starlette_server.start()
           │
           ▼
[starlette_server.py]  UvicornServer.start()  [L329-L473]
    ├─ create_starlette_app(runtime)  ← 注册所有路由（含 WebSocket）
    ├─ 手动绑定 TCP socket（支持端口重试、IPv6 双栈）
    ├─ 创建 uvicorn.Config（配置 ws_protocol, ws_max_size, ping_interval 等）
    ├─ 创建 uvicorn.Server
    └─ 启动后台任务 serve_with_signal()
           │
           ├─ await server.startup(sockets=[socket])  ← uvicorn 启动
           └─ await server.main_loop()  ← 进入 uvicorn 事件循环
```

---

## 二、总体架构概览

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
│                     uvicorn (ASGI Server)                           │
│  TCP 监听 / TLS / WebSocket 协议帧解析 / 心跳管理                    │
└───────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Starlette (Web Framework)                        │
│  ┌────────────────────┐  ┌───────────────┐  ┌───────────────────┐  │
│  │_websocket_endpoint │  │StarletteSess..│  │ WebsocketSession..│  │
│  │ (accept/receive)   │─▶│ write_forward..│─▶│ connect/disconnect │  │
│  │ receive_bytes      │  │  _sender task  │  │                   │  │
│  └────────────────────┘  └───────────────┘  └───────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
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

## 三、连接建立链路

### 3.1 前端连接发起

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

### 3.2 后端连接接受（Starlette 层）

**关键文件：**
- [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L139-L141)
- [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L359-L524)

**路由注册过程**：
1. `create_starlette_app(runtime)` 调用 `create_streamlit_routes(runtime)`
2. `create_streamlit_routes()` 调用 `create_websocket_routes(runtime, base_url)`
3. `create_websocket_routes()` 创建 `WebSocketRoute`，路径为 `/{base_url}/_stcore/stream`
4. `create_websocket_handler(runtime)` 返回闭包 `_websocket_endpoint` 作为处理函数

**`_websocket_endpoint` 处理流程（Starlette 层）**：
1. **Origin 校验**：`_is_origin_allowed()` 防止跨站 WebSocket 劫持
2. **Subprotocol 解析**：`_parse_subprotocols()` 从 `Sec-WebSocket-Protocol` 提取 (streamlit, xsrf_token, existing_session_id)
3. **接受连接**：`websocket.accept(subprotocol=subprotocol)` —— Starlette 封装 uvicorn 完成 WebSocket 握手
4. **创建 StarletteSessionClient**：包装 WebSocket，提供同步写入接口
5. **用户认证**：XSRF token 验证 + Cookie 解析 + 可信 Header 提取
6. **注册会话**：`runtime.connect_session()` → `WebsocketSessionManager.connect_session()`
7. **进入消息循环**：`while True: await websocket.receive_bytes()` —— Starlette 封装 uvicorn 接收二进制帧

### 3.3 会话管理器

**关键文件：**
- [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L59-L169)

`WebsocketSessionManager.connect_session()` 处理：
- 若 `existing_session_id` 有效且不在活跃列表中，从 `SessionStorage` 恢复会话（重连场景）
- 否则创建新的 `AppSession`，分配唯一 session ID
- 记录到 `_active_session_info_by_id` 字典

---

## 四、消息编解码（Protobuf）

> **框架归属说明**：Protobuf 编解码属于 Streamlit 业务逻辑层，与底层 Web 框架（Starlette/uvicorn）无关。但二进制数据的收发由 Starlette 封装 uvicorn 完成。

### 4.1 消息协议定义

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

### 4.2 后端编码（业务逻辑层）

**关键文件：**
- [runtime_util.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/runtime_util.py#L72-L96)

`serialize_forward_msg(msg)` 流程：
1. 调用 `msg.SerializeToString()` 序列化为二进制
2. 检查是否超过 `server.maxMessageSize`（默认 200MB）
3. 若超限，替换为 `MessageSizeError` 异常消息
4. 返回最终二进制 bytes → 后续由 Starlette 层的 `send_bytes()` 发送

### 4.3 后端解码（Starlette 层 + 业务逻辑层）

**关键文件：**
- [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L457-L480)

```
uvicorn 接收 WebSocket 二进制帧 → Starlette websocket.receive_bytes() 返回 bytes
  ▼
BackMsg.ParseFromString(data)  # 业务逻辑层：从二进制解析 BackMsg
  ▼
back_msg.WhichOneof("type")  # 确定具体消息类型
```

> **框架边界**：`receive_bytes()` 是 Starlette 对 uvicorn WebSocket 协议解析的封装，`ParseFromString()` 是纯业务逻辑。

### 4.4 前端编码

**关键文件：**
- [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/connection/src/WebsocketConnection.tsx#L665-L679)

```typescript
sendMessage(obj: IBackMsg): void {
  const msg = BackMsg.create(obj)
  const buffer = BackMsg.encode(msg).finish()  // Protobuf 编码
  const encodedMessage = new Uint8Array(
    buffer.buffer as ArrayBuffer,
    buffer.byteOffset,
    buffer.byteLength
  )
  this.websocket.send(encodedMessage)  // 浏览器原生 WebSocket API 发送
}
```

### 4.5 前端解码 & 消息缓存

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

## 五、后端消息广播分发（ForwardMsg 下行链路）

> **框架归属说明**：消息产生、入队、flush 调度均属于 Streamlit Runtime 业务逻辑层，最终通过 Starlette 桥接层发送到 WebSocket。

### 5.1 消息产生：从 DeltaGenerator 到 ForwardMsg

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

### 5.2 ScriptRunContext：Hash 计算与缓存引用

**关键文件：**
- [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L312-L327)
- [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L468-L479)

`ScriptRunContext.enqueue(msg)`：
1. 设置 `msg.metadata.active_script_hash`（用于多页面追踪）
2. `populate_hash_if_needed(msg)`：计算消息内容 hash（用于去重缓存）
3. 若消息可缓存且 hash 在 `cached_message_hashes`（前端已收到的消息集合）中：
   - 创建引用消息 `create_reference_msg(msg)`，只发送 hash，不发实际内容
4. 调用 `self._enqueue(msg)` → 实际绑定到 `AppSession._enqueue_forward_msg`

### 5.3 AppSession：入队 ForwardMsgQueue

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

### 5.4 Runtime 主循环：Flush & Dispatch

**关键文件：**
- [runtime.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/runtime.py#L624-L699)

Runtime 的 `_loop_coroutine()` 是消息分发的心脏（运行在 uvicorn 事件循环中）：

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

### 5.5 StarletteSessionClient：业务逻辑 → Starlette 桥接

**关键文件：**
- [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L265-L357)

> **设计背景**：Runtime 与 WebSocket 连接同属一个 asyncio 事件循环，但为了抽象解耦，通过 `SessionClient` 接口隔离。`StarletteSessionClient` 是该接口的 Starlette 实现。

```
Runtime._send_message()  [业务逻辑层]
  ▼
StarletteSessionClient.write_forward_msg(msg)  [Starlette 桥接层]
  ├─ serialize_forward_msg(msg) → bytes  [编码]
  └─ self._send_queue.put_nowait(payload)  # asyncio.Queue（有界）
         │
         ▼  后台 _sender() 协程（同事件循环）
      await self._websocket.send_bytes(payload)  [Starlette → uvicorn]
```

- `_send_queue` 限制最大积压量（`WEBSOCKET_MAX_SEND_QUEUE_SIZE`），防止客户端消费过慢导致内存泄漏
- 队列满或 WebSocket 断开时抛出 `SessionClientDisconnectedError`，触发会话断开

### 5.6 ScriptRunner 事件 → ForwardMsg

除了 UI Delta，ScriptRunner 执行状态变化也会产生 ForwardMsg：

**关键文件：**
- [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/app_session.py#L569-L767)

`AppSession._on_scriptrunner_event()` 从脚本线程通过 `call_soon_threadsafe` 切到 uvicorn 事件循环线程：

| ScriptRunnerEvent | 产生的 ForwardMsg |
|-------------------|-------------------|
| `SCRIPT_STARTED` | `new_session`（配置/主题/页面）+ `session_status_changed` |
| `SCRIPT_STOPPED_WITH_SUCCESS` | `script_finished(FINISHED_SUCCESSFULLY)` + `session_status_changed` |
| `SCRIPT_STOPPED_WITH_COMPILE_ERROR` | `script_finished(FINISHED_WITH_COMPILE_ERROR)` + `session_event(exception)` |
| `SCRIPT_STOPPED_FOR_RERUN` | `script_finished(FINISHED_EARLY_FOR_RERUN)` |
| `FRAGMENT_STOPPED_WITH_SUCCESS` | `script_finished(FINISHED_FRAGMENT_RUN_SUCCESSFULLY)` |
| `ENQUEUE_FORWARD_MSG` | 直接透传传入的 ForwardMsg |

---

## 六、前端消息消费链路

### 6.1 ConnectionManager 到 App

**关键文件：**
- [ConnectionManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/connection/src/ConnectionManager.ts#L89-L336)
- [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/frontend/app/src/App.tsx#L968-L1042)

```
WebsocketConnection.handleMessage()  [浏览器 WebSocket onmessage 事件]
  ├─ ForwardMsg.decode()     // Protobuf 解码
  ├─ ForwardMsgCache 处理    // 缓存 / ref_hash 解引用
  └─ 保序队列 onMessage()
         ▼
ConnectionManager.props.onMessage
         ▼
App.handleMessage(msgProto)
```

### 6.2 App.handleMessage 消息分发

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

### 6.3 Delta 渲染树更新

`handleDeltaMsg` 将 Delta 应用到 `AppRoot`（不可变渲染树），最终触发 React 重渲染：
- `delta_path` 定位 UI 树中的节点位置
- 支持 `add_block`（新增容器）、`new_element`（新增元素）、`new_transient`（临时元素如 spinner）

---

## 七、前端→后端 BackMsg 上行链路

### 7.1 触发来源

常见触发场景：
- 用户点击按钮 / 修改组件值 → `WidgetStateManager.sendRerunBackMsg()`
- 用户点击 Stop → `App.stopScript()`
- 文件上传 → `FileUploadClient` 请求 URL
- 心跳定时发送 → `ConnectionManager.onHeartbeatSent()`

### 7.2 编码与发送

```
WidgetStateManager / App 等
  ▼
BackMsg.create({ rerun_script: { widget_states, page_script_hash, ... } })
  ▼
ConnectionManager.sendMessage()
  ▼
WebsocketConnection.sendMessage()
  ├─ BackMsg.encode(msg).finish()  // Protobuf 编码
  └─ websocket.send(Uint8Array)    // 浏览器原生 WebSocket API 发送
```

### 7.3 后端接收与处理

**关键文件：**
- [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L455-L510)
- [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/runtime/app_session.py#L338-L376)

```
uvicorn 解析 WebSocket 帧 → Starlette websocket.receive_bytes()  [Starlette 层]
  ▼
BackMsg.ParseFromString(data)  [业务逻辑层]
  ▼
back_msg.WhichOneof("type")  // 确定类型
  ▼
runtime.handle_backmsg(session_id, back_msg)  [业务逻辑层]
  ▼
AppSession.handle_backmsg(msg)  [业务逻辑层]
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

## 八、完整时序图（以用户点击按钮触发重跑为例）

> **框架标注**：每条消息链路都明确标注经过的框架层。

```
  Browser (Frontend)         uvicorn/Starlette (Backend)       Runtime (Backend)
        │                            │                               │
        │  用户点击 st.button        │                               │
        │                            │                               │
        │ WidgetStateManager         │                               │
        │   sendRerunBackMsg()       │                               │
        │   BackMsg Protobuf 编码    │                               │
        │                            │                               │
        │── WebSocket binary ───────▶│                               │
        │                            │                               │
        │                            │ uvicorn 解析 WebSocket 帧     │
        │                            │ websocket.receive_bytes()     │
        │                            │ BackMsg.ParseFromString()     │
        │                            │ runtime.handle_backmsg() ────▶│
        │                            │                               │ AppSession.handle_backmsg()
        │                            │                               │ request_rerun()
        │                            │                               │ ScriptRunner 启动（新线程）
        │                            │                               │
        │                            │                               │ 执行用户脚本
        │                            │                               │ st.write(...) → DeltaGenerator
        │                            │                               │ ScriptRunContext.enqueue
        │                            │                               │ AppSession._enqueue_forward_msg
        │                            │                               │ ForwardMsgQueue.enqueue
        │                            │                               │
        │                            │     Runtime._loop_coroutine   │
        │                            │     flush_browser_queue() ◀───│
        │                            │     _send_message()            │
        │                            │                               │
        │◀── WebSocket binary ──────│  StarletteSessionClient       │
        │                            │    write_forward_msg()        │
        │                            │    serialize + send_bytes()   │
        │                            │                               │
        │ WebsocketConnection        │                               │
        │   handleMessage()          │                               │
        │   ForwardMsg.decode()      │                               │
        │   ForwardMsgCache          │                               │
        │   保序派发                 │                               │
        │                            │                               │
        │ App.handleMessage()        │                               │
        │   dispatchProto            │                               │
        │   Delta 更新 AppRoot        │                               │
        │                            │                               │
        │ React 重渲染 UI            │                               │
        ▼                            ▼                               ▼
```

---

## 九、关键设计要点

1. **全双工二进制协议**：严格使用 WebSocket binary frame + Protobuf，拒绝 text frame
2. **Subprotocol 复用**：利用 `Sec-WebSocket-Protocol` 传递 XSRF token 和 session ID（浏览器 API 限制）
3. **同步/异步桥接**：`StarletteSessionClient` 使用 asyncio.Queue + 后台 sender task 连接 Runtime 的同步调用与 WebSocket 的异步发送
4. **消息保序**：前端 `messageQueue` + `lastDispatchedMessageIndex` 保证即使异步解码也按接收顺序派发
5. **增量去重**：
   - 后端 `ForwardMsgQueue` 合并同 `delta_path` 的冗余 Delta
   - 前后端 `ref_hash` 机制避免重复发送相同内容
6. **背压保护**：发送队列有界，客户端消费过慢会触发断开
7. **线程安全**：ScriptRunner 线程通过 `event_loop.call_soon_threadsafe()` 切回 uvicorn 事件循环操作 AppSession
8. **会话生命周期**：区分 active（WebSocket 已连接）和 inactive（断开暂存于 SessionStorage），支持短时间重连恢复
9. **框架分层清晰**：
   - **uvicorn 层**：TCP、TLS、WebSocket 协议帧解析、心跳管理、事件循环
   - **Starlette 层**：路由、中间件、WebSocket 握手、Origin 校验、`receive_bytes`/`send_bytes` API
   - **业务逻辑层**：Protobuf 编解码、消息分发、会话管理、脚本执行、ForwardMsg 队列

---

## 十、st.App 与外部 ASGI 宿主的 WebSocket 处理边界

### 10.1 四种运行模式

`config._server_mode` 标识 Streamlit 当前的运行模式，决定了 Runtime 生命周期由谁管理：

**关键文件**：[config.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/config.py#L65-L73)

| `_server_mode` | 设置位置 | 含义 | 谁管理 Runtime |
|-----------------|---------|------|---------------|
| `"starlette-managed"` | [bootstrap.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/bootstrap.py#L379) `run()` | `streamlit run app.py`（传统脚本模式） | `Server` → `UvicornServer` |
| `"starlette-app"` | [bootstrap.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/bootstrap.py#L349) `run_asgi_app()` | `streamlit run app.py`（检测到 `st.App` 实例） | `UvicornRunner` + `st.App._combined_lifespan` |
| `"asgi-server"` | [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L605-L607) | `uvicorn app:app`（独立运行） | `st.App._combined_lifespan` |
| `"asgi-mounted"` | [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L604) / [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L765) | 挂载到 FastAPI/Django 等 | 宿主框架 lifespan 或 auto-start |

### 10.2 默认启动流程（传统脚本模式）

```
streamlit run app.py
    │
    ├─ 脚本中无 st.App 实例
    │
    ▼
[bootstrap.py]  run()
    ├─ config._server_mode = "starlette-managed"
    ├─ 创建 Server(main_script_path, is_hello)
    │     └─ Server.__init__: 创建 Runtime（含 WebSocketSessionManager）
    └─ asyncio.run(run_server())
          ├─ await server.start()
          │     └─ UvicornServer(runtime).start()
          │           ├─ create_starlette_app(runtime)  ← lifespan 管理 Runtime
          │           └─ uvicorn.Server.main_loop()
          ├─ _on_server_start(server)  ← 打印 URL、打开浏览器
          ├─ _set_up_signal_handler(server)  ← SIGTERM/SIGINT
          └─ await server.stopped

Runtime 生命周期:
  startup → create_starlette_app 内部 lifespan → await runtime.start()
  shutdown → create_starlette_app 内部 lifespan → runtime.stop()
```

**关键点**：Runtime 的 start/stop 由 `create_starlette_app()` 内部的 `_lifespan` 管理，不需要 `st.App` 类参与。

### 10.3 st.App 独立运行（streamlit run 检测到 st.App 实例）

```
streamlit run app.py  （app.py 中有 app = st.App("main.py")）
    │
    ▼
[cli.py]  检测到 st.App 实例 → 调用 bootstrap.run_asgi_app()
    │
    ▼
[bootstrap.py]  run_asgi_app()
    ├─ config._server_mode = "starlette-app"
    └─ UvicornRunner(app_import_string).run()  ← 阻塞式 uvicorn
          │
          ▼
st.App.__call__()  ← ASGI 入口
    └─ _build_starlette_app()
          ├─ create_streamlit_routes(runtime)  ← 注册 WebSocket 路由
          ├─ create_streamlit_middleware()
          └─ lifespan = _combined_lifespan  ← 内部管理 Runtime start/stop

Runtime 生命周期:
  startup → _combined_lifespan → await runtime.start()
  shutdown → _combined_lifespan finally → runtime.stop()
```

### 10.4 st.App 独立运行（外部 uvicorn 直接启动）

```
uvicorn app:app  （app = st.App("main.py")）
    │
    ▼
st.App.__call__()  ← ASGI 入口
    ├─ _build_starlette_app()
    │     ├─ lifespan = _combined_lifespan  ← 内部管理
    │     └─ _external_lifespan = False
    └─ await starlette_app(scope, receive, send)

_combined_lifespan 内部:
  ├─ config._server_mode is None → 设为 "asgi-server"
  ├─ prepare_streamlit_environment()
  └─ await self._runtime.start() / self._runtime.stop()
```

### 10.5 st.App 挂载到外部框架（FastAPI/Django 等）

这是最复杂的场景，涉及 **lifespan 传递问题** 和 **Runtime 自动启动** 两个关键边界。

#### 场景 A：正确使用 `app.lifespan()`

```python
from fastapi import FastAPI
import streamlit as st

streamlit_app = st.App("dashboard.py")
app = FastAPI(lifespan=streamlit_app.lifespan())
app.mount("/dashboard", streamlit_app)
```

```
FastAPI 启动 → lifespan 协议触发
    │
    ▼
st.App._combined_lifespan()  ← 由 FastAPI 的 lifespan 调用
    ├─ _external_lifespan = True  ← lifespan() 方法设置
    ├─ config._server_mode → "asgi-mounted"
    ├─ prepare_streamlit_environment()
    ├─ await self._runtime.start()
    ├─ 执行用户的 user_lifespan（如有）
    └─ yield → FastAPI 运行 → finally: runtime.stop()

WebSocket 请求到达:
  FastAPI → Mount("/dashboard") → st.App.__call__()
    └─ runtime 已经启动 → 直接转发到 _starlette_app
```

**关键文件**：[starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L485-L515)

> **注意**：`_build_starlette_app()` 中，当 `_external_lifespan = True` 时，内部 Starlette 应用的 lifespan 设为 `None`，因为生命周期由宿主框架管理。

#### 场景 B：直接 mount，未使用 `app.lifespan()`

```python
from fastapi import FastAPI
import streamlit as st

streamlit_app = st.App("dashboard.py")
app = FastAPI()
app.mount("/dashboard", streamlit_app)  # 没有传 lifespan
```

**问题**：ASGI 规范中，`lifespan` 事件只发送给根应用（FastAPI），**不会传递给 mounted 子应用**。因此 Streamlit 的 Runtime 不会通过 lifespan 启动。

**解决方案**：[st.App.__call__()](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L691-L726) 中的 **auto-start 机制**：

```python
async def __call__(self, scope, receive, send):
    if self._starlette_app is None:
        self._starlette_app = self._build_starlette_app()

    # 关键：检测 Runtime 未启动，在第一个请求到达时自动启动
    if (
        scope["type"] in {"http", "websocket"}
        and self._runtime is not None
        and self._runtime.state == RuntimeState.INITIAL
    ):
        async with self._startup_lock:  # 防止并发启动
            if self._runtime.state == RuntimeState.INITIAL and not self._auto_started:
                await self._auto_start_runtime()

    await self._starlette_app(scope, receive, send)
```

**auto-start 的行为**：
1. 设置 `config._server_mode = "asgi-mounted"`
2. 调用 `prepare_streamlit_environment()`
3. 调用 `await self._runtime.start()`
4. 注册 `atexit` 清理回调
5. 如果用户提供了 `lifespan` 但没用 `app.lifespan()`，输出警告

### 10.6 WebSocket 处理在各运行模式下的差异

| 环节 | `starlette-managed` | `starlette-app` | `asgi-server` | `asgi-mounted` |
|------|---------------------|-----------------|---------------|----------------|
| **WebSocket 路由注册** | `create_starlette_app()` 内部 | `st.App._build_starlette_app()` | 同左 | 同左 |
| **WebSocket 端点处理** | `_websocket_endpoint` 闭包 | 同左 | 同左 | 同左 |
| **Runtime 生命周期** | Starlette lifespan | `_combined_lifespan` | `_combined_lifespan` | 宿主 lifespan 或 auto-start |
| **SessionMiddleware** | `create_streamlit_middleware()` | 同左 | 同左 | 同左 |
| **Cookie 路径** | `/` | `/` | `/` | 可能带 basePath 前缀 |
| **Origin 校验** | `_is_origin_allowed()` | 同左 | 同左 | 同左（需宿主正确代理 Host/Origin） |
| **XSRF 校验** | `_parse_subprotocols()` | 同左 | 同左 | 同左 |
| **Base URL 前缀** | `config.server.baseUrlPath` | 同左 | 同左 | 取决于 mount 路径 |

### 10.7 挂载场景下 WebSocket 的注意事项

1. **路径映射**：挂载到 `/dashboard` 时，WebSocket 路由为 `/dashboard/_stcore/stream`。Streamlit 内部路由通过 `base_url` 配置自动处理，但宿主框架必须正确转发 WebSocket Upgrade 请求到子应用。

2. **Lifespan 不传递**：ASGI 规范中 `lifespan` scope 只发给根应用。`st.App` 通过 `__call__` 中的 auto-start 机制补偿，但用户应优先使用 `app.lifespan()` 显式传递。

3. **Cookie 路径冲突**：挂载到非根路径时，认证 Cookie 需要设置正确的 `path`。[starlette_auth_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_auth_routes.py#L252-L260) 中 `_delete_cookie_at_current_and_legacy_paths()` 负责清理旧 Tornado 遗留的根路径 Cookie。

4. **反向代理**：在宿主框架前面再加 Nginx 等反向代理时，需确保：
   - WebSocket Upgrade 头正确传递
   - `Host` / `Origin` 头不被代理覆盖
   - `X-Forwarded-For` / `X-Forwarded-Proto` 正确设置

---

## 十一、Tornado 历史命名残留分析

### 11.1 代码库中的 Tornado 残留

全局搜索 `tornado` 关键字，仅出现在以下位置（均为注释或兼容性代码，无实际 Tornado 依赖）：

#### (1) Cookie 签名兼容 — `websocket_mask()` 函数

**关键文件**：[starlette_app_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app_utils.py#L92-L117)

```python
def websocket_mask(mask: bytes, data: bytes) -> bytes:
    """Mask or unmask data for WebSocket transmission per RFC 6455."""
```

**历史背景**：Tornado 使用 `tornado.web.websocket_mask` 对 XSRF token 做 XOR 掩码。Starlette 迁移后，这个函数被复制到 `starlette_app_utils.py` 中保持相同的掩码算法，确保新旧 Cookie 格式兼容。

**使用场景**：
- `generate_xsrf_token_string()` 生成 V2 XSRF token 时对 token 做 XOR 掩码
- `decode_xsrf_token_string()` 解码 V2 token 时做反向 XOR

#### (2) XSRF Token V1 兼容

**关键文件**：[starlette_app_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_app_utils.py#L261-L272)

```python
# V1 tokens:
# TODO(lukasmasuch): This is likely unused in Streamlit since only V2 tokens
# are used. We might be able to just remove this part.
token = binascii.a2b_hex(value.encode("ascii"))
```

V1 token 是 Tornado 时代的格式（无掩码、无时间戳），当前仍保留解码逻辑以兼容可能的旧 Cookie，但注释标注可能移除。

#### (3) 认证 Cookie 路径迁移

**关键文件**：[starlette_auth_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_auth_routes.py#L262-L304)

```python
def _delete_legacy_root_auth_cookies(response: Response) -> None:
    """Delete legacy root-path auth cookies when auth is scoped to a base path."""
    # Legacy Tornado auth cookies were never chunked, so deleting the base
    # cookie name at the root path is sufficient here.

def _clear_auth_cookie(response: Response, request: Request) -> None:
    """Clear the auth cookies...
    Also delete legacy root-path auth cookies left behind by the
    pre-Starlette Tornado auth implementation.
    """
```

**历史背景**：Tornado 时代认证 Cookie 统一设置在根路径 `/`。Starlette 迁移后，当配置了 `server.baseUrlPath` 时 Cookie 路径变为 `/basePath/`。为避免用户从 Tornado 版本升级后残留旧 Cookie 无法清除，Starlette 版本在设置新 Cookie 时会同时删除根路径的旧 Cookie。

#### (4) 静态文件符号链接 — `follow_symlink`

**关键文件**：[starlette_static_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/lib/streamlit/web/server/starlette/starlette_static_routes.py#L66-L69)

```python
# follow_symlink=True restores Tornado parity for Bazel/Nix-style deployments
# where the static directory may contain or be a symlink.
super().__init__(directory=directory, html=True, follow_symlink=True)
```

Tornado 默认跟随符号链接，而 Starlette 的 `StaticFiles` 默认不跟随。此处显式开启以保持行为一致。

#### (5) Issue 引用中的 Tornado

**关键文件**：[product-spec.md](file:///d:/fz/0601/solo-dogfeeding/code/215-streamlit/specs/2025-12-23-st-app/product-spec.md#L55-L65)

st.App 设计文档中引用了旧 Tornado 时代的 Issue（如 #8661 "Expose Tornado instance"、#9916 "Tornado HTTPServer extra arguments"）。这些 Issue 的需求已被 st.App 的 ASGI 方案替代，不再需要暴露 Tornado 实例。

### 11.2 Cookie 签名机制的 Tornado → Starlette 迁移对照

| 机制 | Tornado 时代 | Starlette 时代 |
|------|-------------|---------------|
| **Cookie 签名** | `tornado.web.create_signed_value()` | `itsdangerous.URLSafeTimedSerializer` |
| **Cookie 验签** | `tornado.web.decode_signed_value()` | `itsdangerous` 反序列化 |
| **XSRF 生成** | `tornado.web.XSRFTokenHandler` | `generate_xsrf_token_string()` + `websocket_mask()` |
| **XSRF 验证** | `tornado.web.check_xsrf_cookie()` | `validate_xsrf_token()` + `decode_xsrf_token_string()` |
| **掩码算法** | `tornado.websocket.websocket_mask` | 本地复制的 `websocket_mask()` |
| **Cookie 分块** | 不支持 | `set_cookie_with_chunks()` / `get_cookie_with_chunks()` |
| **Cookie 路径** | 始终 `/` | 基于 `server.baseUrlPath` 动态计算 |

> **核心原因**：Tornado 有自带的签名 Cookie API（`create_signed_value` / `decode_signed_value`），Starlette 没有。迁移时选用 `itsdangerous` 库重新实现，但保留了相同的 XOR 掩码 XSRF token 格式（V2）以保证浏览器中旧 Cookie 仍可验证。

### 11.3 为什么不完全清除 Tornado 残留

1. **Cookie 兼容性**：用户从 Tornado 版本升级后，浏览器中可能仍有旧格式的认证 Cookie 和 XSRF token。如果移除 V1 解码逻辑，这些用户会被强制登出。
2. **XSRF 掩码格式锁定**：`websocket_mask()` 生成的 V2 token 格式已在前端和后端之间形成协议约定，改变格式会导致所有活跃 WebSocket 连接的 XSRF 验证失败。
3. **渐进清理**：代码中的 `TODO` 注释表明这些残留计划逐步移除，但需要等待一个合理的时机（如大版本升级）。
