# 上传记录恢复机制：代码事实与推测边界

## 本文目的

精确区分哪些结论是**代码直接证明**的，哪些是**基于代码逻辑的合理推测**，以及哪些**还需要更多证据**。重点澄清：

1. `close_session` 后会话存储和上传文件的状态
2. TTL 过期后的状态来源与清理时机
3. 前端重连时 widget state 从哪里来
4. DeletedFile 真实出现的场景

---

## 一、close_session 的完整链路（代码可证）

### 1.1 两条路径

`close_session` 在 [websocket_session_manager.py L204-L226](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L204-L226) 中有两条完全独立的路径：

```python
def close_session(self, session_id: str) -> None:
    # 路径一：活跃会话
    if session_id in self._active_session_info_by_id:
        active_session_info = self._active_session_info_by_id[session_id]
        del self._active_session_info_by_id[session_id]  # 从活跃字典移除
        active_session_info.session.shutdown()            # 调用 shutdown
        # ... 统计 ...
        return  # ← 直接返回，不涉及 SessionStorage

    # 路径二：存储中的会话
    session_info = self._session_storage.get(session_id)
    if session_info:
        self._session_storage.delete(session_id)  # 从存储删除
        session_info.session.shutdown()           # 调用 shutdown
        # ... 统计 ...
```

**代码可证事实：**

| 路径 | 是否从活跃字典移除 | 是否从 SessionStorage 删除 | 是否调用 shutdown |
|------|-------------------|---------------------------|-------------------|
| 活跃会话 | ✅ 是 | ❌ 否（活跃会话不在存储中） | ✅ 是 |
| 存储会话 | ❌ 否 | ✅ 是 | ✅ 是 |

### 1.2 shutdown 的行为

`shutdown` 在 [app_session.py L292-L316](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L292-L316) 中的执行顺序：

```python
def shutdown(self) -> None:
    if self._state != AppSessionState.SHUTDOWN_REQUESTED:
        # 1. 第一步就删除上传文件！
        self._uploaded_file_mgr.remove_session_files(self.id)

        # 2. 清理媒体文件引用和孤立文件
        if runtime.exists():
            rt = runtime.get_instance()
            rt.media_file_mgr.clear_session_refs(self.id)
            rt.media_file_mgr.remove_orphaned_files()

        # 3. 请求脚本停止
        self.request_script_stop()

        # 4. 标记状态
        self._state = AppSessionState.SHUTDOWN_REQUESTED

        # 5. 断开文件监听器
        self.disconnect_file_watchers()

        # 6. 清理会话缓存
        self.clear_session_caches()
```

**代码可证事实：**
- `shutdown()` 第一步就调用 `remove_session_files()`，上传文件**立即被删除**
- `shutdown()` 只会执行一次（有状态守卫 `if self._state != SHUTDOWN_REQUESTED`）
- shutdown 后的会话**不会再运行脚本**（`request_rerun` 会检查状态并直接返回）

### 1.3 __del__ 兜底

[app_session.py L205-L207](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L205-L207)：

```python
def __del__(self) -> None:
    """Ensure that we call shutdown() when an AppSession is garbage collected."""
    self.shutdown()
```

**代码可证事实：**
- AppSession 对象被 GC 时，会调用 `shutdown()`
- 这是一个**兜底机制**，确保即使会话意外丢失，文件也会被清理

---

## 二、SessionStorage 与 TTL（部分推测）

### 2.1 SessionStorage 的实现

`MemorySessionStorage` 在 [memory_session_storage.py L27-L77](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_session_storage.py#L27-L77) 中的核心实现：

```python
class MemorySessionStorage(SessionStorage):
    def __init__(
        self,
        maxsize: int = 128,
        ttl_seconds: int = 2 * 60,  # 2分钟
    ) -> None:
        self._cache: MutableMapping[str, SessionInfo] = TTLCache(
            maxsize=maxsize, ttl=ttl_seconds
        )

    def get(self, session_id: str) -> SessionInfo | None:
        return self._cache.get(session_id, None)

    def save(self, session_info: SessionInfo) -> None:
        self._cache[session_info.session.id] = session_info

    def delete(self, session_id: str) -> None:
        del self._cache[session_id]
```

**代码可证事实：**
- 使用 `cachetools.TTLCache` 实现
- 默认 TTL：**2分钟**
- 默认最大容量：**128个会话**
- TTL 过期后，条目从缓存字典中**自动移除**

### 2.2 TTL 过期后的清理时序

**代码可证的部分：**

| 时间点 | 事件 | 代码依据 |
|--------|------|---------|
| T=0 | 会话断开，保存到 SessionStorage | `disconnect_session` 调用 `save()` |
| T=2min | SessionInfo 从 TTLCache 移除 | TTLCache 自动过期 |
| ? | AppSession 对象被 GC | 依赖 Python GC 时机 |
| GC 时 | `__del__` → `shutdown()` → 删除上传文件 | `__del__` 方法 |

**需要推测的部分：**

> ⚠️ **以下为合理推测，代码不能直接证明**

1. **GC 时机不确定**：Python 的 GC 时机不保证，TTL 过期后，AppSession 对象可能还在内存中一段时间
2. **文件清理延迟**：在 TTL 过期到 GC 发生之间，上传文件**仍然存在**于 `MemoryUploadedFileManager` 中
3. **引用存活情况**：如果事件循环或其他地方还持有 AppSession 的引用，对象不会被 GC，文件会一直存在

### 2.3 UploadedFileManager 的全局共享

**代码可证事实：**

`MemoryUploadedFileManager` 在 Runtime 中是单例，所有会话共享：

- 初始化位置：[runtime.py L218](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/runtime.py#L218)
- 传递给 SessionManager：[runtime.py L231](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/runtime.py#L231)
- 传递给每个 AppSession：[runtime.py L594](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/runtime.py#L594)

```python
# Runtime 初始化时创建，全局唯一
self._uploaded_file_mgr = config.uploaded_file_manager
```

**代码可证事实：**
- UploadedFileManager 是**全局共享**的
- 文件数据按 `session_id` 分组存储：`dict[str, dict[str, UploadedFileRec]]`
- 只有显式调用 `remove_session_files()` 才会删除文件
- **没有自动清理机制**（没有 TTL、没有定时任务）

---

## 三、前端重连时的状态来源（代码可证）

### 3.1 sessionId 的传递

前端通过 WebSocket 子协议头传递 sessionId，在 [WebsocketConnection.tsx L531-L544](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/WebsocketConnection.tsx#L531-L544)：

```typescript
// 利用 Sec-WebSocket-Protocol 头传递 auth 和 session tokens
// 格式：["streamlit", auth_token, session_id]
const sessionTokens = await this.getSessionTokens()
this.websocket = new WebSocket(uri, ["streamlit", ...sessionTokens])
```

`getSessionTokens` 在 [WebsocketConnection.tsx L504-L515](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/WebsocketConnection.tsx#L504-L515)：

```typescript
private async getSessionTokens(): Promise<Array<string>> {
    const hostAuthToken = await this.args.claimHostAuthToken()
    const xsrfCookie = getCookie("_streamlit_xsrf")
    this.args.resetHostAuthToken()
    const lastSessionId = this.args.getLastSessionId()
    return [
      hostAuthToken ?? xsrfCookie ?? "PLACEHOLDER_AUTH_TOKEN",
      ...(lastSessionId ? [lastSessionId] : []),  // 有上一次 sessionId 就带上
    ]
}
```

**代码可证事实：**
- `lastSessionId` 来自 `SessionInfo.last`
- 初始连接时没有 lastSessionId
- 断连后重连时会带上 lastSessionId

### 3.2 后端重连处理

后端在 `_parse_subprotocols` 中解析 sessionId，在 [starlette_websocket.py L67-L88](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L67-L88)：

```python
def _parse_subprotocols(headers: Headers) -> tuple[str, str | None, str | None]:
    raw = headers.get("sec-websocket-protocol")
    # ... 解析 ...
    selected = entries[0] if entries else "streamlit"
    xsrf_token = entries[1] if len(entries) >= 2 and entries[1] else None
    existing_session = entries[2] if len(entries) >= 3 and entries[2] else None
    return selected, xsrf_token, existing_session
```

然后在 `connect_session` 中处理，在 [websocket_session_manager.py L100-L140](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L100-L140)：

```python
def connect_session(self, ...) -> str:
    # 1. 先检查是不是活跃会话
    if existing_session_id in self._active_session_info_by_id:
        # 活跃会话，直接返回（理论上不会发生）
    
    # 2. 从 SessionStorage 查找
    session_info = self._session_storage.get(existing_session_id)
    
    if isinstance(session_info, SessionInfo):
        # 3. 找到！恢复会话
        existing_session = session_info.session
        existing_session.register_file_watchers()
        
        self._active_session_info_by_id[existing_session.id] = ActiveSessionInfo(...)
        self._session_storage.delete(existing_session.id)  # 从存储移除
        return existing_session.id
    
    # 4. 没找到，创建新会话
    session = AppSession(...)
    ...
    return session.id
```

**代码可证事实：**
- 重连时从 SessionStorage 取出**整个 AppSession 对象**
- 恢复后从 SessionStorage 删除（转到活跃状态）
- 找不到就创建**全新的会话**

### 3.3 重连时 widget state 的状态

**代码可证事实：**

1. **重连成功（找到会话）**：整个 AppSession 对象被恢复，包括：
   - `_session_state`（widget state 完整保留）
   - `_client_state`（客户端状态）
   - `_uploaded_file_mgr` 是全局的，文件本来就在
   - ✅ **不会产生 DeletedFile**

2. **重连失败（没找到会话）**：创建全新会话，包括：
   - 全新的 `_session_state`（空的）
   - 全新的 `_client_state`（空的）
   - 全新的 `session_id`
   - ✅ **不会产生 DeletedFile**（因为 widget state 也是空的）

> **重要澄清**：重连失败时，前端的 WidgetStateManager 会怎样？是保留旧状态还是重置？这一点需要进一步确认。

### 3.4 前端 widget state 的生命周期

**代码可证的事实：**

1. `WidgetStateManager` 的状态存储在**内存**中的 `WidgetStateDict`（Map）
2. 搜索整个前端代码，**没有** widget state 持久化到 `localStorage` 的代码
3. `localStorage` 只存了**主题**相关的设置

**需要推测的部分：**

> ⚠️ **以下为合理推测，代码不能直接证明**

1. **页面刷新**：前端重新初始化，WidgetStateManager 也是新的，状态从后端获取
2. **重连成功**：后端会发送新的 `NewSession` 消息吗？还是前端保留旧状态？
3. **重连失败**：前端会重置 widget state 吗？还是保留旧的？

这些问题需要查看前端的连接状态机和 NewSession 消息处理逻辑才能确认。

---

## 四、DeletedFile 真实出现场景分析

### 4.1 产生条件回顾

**代码可证**：DeletedFile 产生的充要条件是：
- `file_id` 存在于 **Widget State** 中
- 但 `file_id` 不存在于 **UploadedFileManager** 中

唯一产生点：[file_uploader.py L101](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L101)

### 4.2 会产生 DeletedFile 的场景

**✅ 代码可证会产生的场景：**

1. **服务重启 + 前端未刷新**
   - 服务重启后，`MemoryUploadedFileManager` 清空
   - 如果前端 WebSocket 还能连上（比如重连成功但会话变了？），且前端保留了旧的 widget state
   - → 可能产生 DeletedFile
   - **但这需要前端在重连失败后还保留旧的 widget state**，这一点存疑

**⚠️ 推测可能产生的场景：**

1. **并发删除与读取**
   - 请求 A：删除文件
   - 请求 B：同时读取文件列表
   - 极小概率产生竞态
   - 推测依据：文件删除和状态同步不是原子操作

2. **自定义 UploadedFileManager 实现**
   - 如果替换为磁盘存储或 S3 存储
   - 文件可能因为过期、清理等原因被外部删除
   - 但 widget state 中还保留着
   - 推测依据：`UploadedFileManager` 是 Protocol，可替换

3. **会话部分清理**
   - 如果某些清理逻辑只删了文件但没清 widget state
   - 目前代码中没发现这种情况
   - 推测依据：理论上可能存在 bug 或边缘情况

**❌ 代码证明不会产生的场景：**

1. **正常断连**：文件保留，状态保留 → 不会产生
2. **正常重连**：文件和状态都恢复 → 不会产生
3. **正常关停**：会话不再运行脚本 → 不会调用 `_get_upload_files` → 不会产生
4. **主动删除文件**：前端同步更新 widget state → 不会产生
5. **新建会话**：widget state 为空 → 不会产生

### 4.3 DeletedFile 的设计意图

从代码结构反推，DeletedFile 可能是为以下目的设计的（**推测**）：

1. **容错/优雅降级**：当文件存储和 widget state 不一致时，系统不崩溃，优雅降级
2. **未来扩展**：为未来的文件自动过期、磁盘存储等功能预留机制
3. **并发安全**：处理多用户/多标签页操作同一文件的竞态
4. **多通道同步**：HTTP 删除文件和 WebSocket 状态更新可能有时间差

---

## 五、关键结论速查表

### 5.1 代码直接证明的结论

| 结论 | 代码依据 | 确定性 |
|------|---------|--------|
| `close_session` 对活跃会话不涉及 SessionStorage | [websocket_session_manager.py L204-L217](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L204-L217) | ✅ 100% |
| `shutdown()` 第一步就删除上传文件 | [app_session.py L297](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L297) | ✅ 100% |
| AppSession GC 时会调用 `shutdown()` | [app_session.py L205-L207](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L205-L207) | ✅ 100% |
| UploadedFileManager 是全局共享的 | [runtime.py L218](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/runtime.py#L218) | ✅ 100% |
| SessionStorage 默认 TTL 是 2 分钟 | [memory_session_storage.py L44](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_session_storage.py#L44) | ✅ 100% |
| 重连时 sessionId 通过 WebSocket 子协议传递 | [WebsocketConnection.tsx L544](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/WebsocketConnection.tsx#L544) | ✅ 100% |
| 重连成功会恢复整个 AppSession 对象 | [websocket_session_manager.py L126-L135](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L126-L135) | ✅ 100% |
| DeletedFile 只在系统内部传递，用户和前端都看不到 | [file_uploader.py L142-L143](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L142-L143), [file_uploader.py L606-L611](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L606-L611) | ✅ 100% |
| 前端 widget state 不持久化到 localStorage | 搜索整个前端代码无相关存储逻辑 | ✅ 99%（否定性结论难以100%确认） |

### 5.2 基于代码逻辑的合理推测

| 结论 | 推测依据 | 置信度 |
|------|---------|--------|
| TTL 过期后文件不会立即被删除，要等 GC | `__del__` 是唯一兜底，GC 时机不确定 | 🔶 80% |
| 页面刷新后 widget state 从后端重新获取 | 前端无持久化，NewSession 消息可能包含状态 | 🔶 70% |
| 正常使用中 DeletedFile 几乎不会出现 | 主要场景都不会触发不一致 | 🔶 75% |
| DeletedFile 是为容错/未来扩展设计的 | 代码中的处理方式表明是防御性设计 | 🔶 70% |
| TTL 过期+未 GC 的会话可能造成内存泄漏 | UploadedFileManager 是全局的，无自动清理 | 🔶 65% |

### 5.3 还需要更多证据的问题

| 问题 | 为什么不确定 | 如何验证 |
|------|-------------|---------|
| 前端重连失败后 widget state 会不会重置？ | 没看到相关逻辑 | 查看 ConnectionManager 和 App 的重连处理 |
| NewSession 消息中是否包含 widget state？ | 没看到完整的消息结构 | 查看 protobuf 定义和 NewSession 处理 |
| 服务重启后前端的行为是什么？ | 需要端到端验证 | 实际测试或查看连接状态机 |
| 有没有其他地方会删除上传文件？ | 可能有定时清理或其他入口 | 全量搜索 remove_file/remove_session_files |

---

## 六、修正与澄清

基于本次代码分析，对上一篇文档中的一些结论进行修正：

### ❌ 之前的不准确表述

1. **"关停后重连会产生 DeletedFile"**
   - 修正：关停后会话不会再运行脚本，所以不会调用 `_get_upload_files`，不会产生 DeletedFile
   - 只有当会话还在运行（能执行脚本）但文件已被删除时，才会产生 DeletedFile

2. **"SessionStorage TTL 过期后状态来源不明确"**
   - 修正：过期后 SessionInfo 从缓存移除，但 AppSession 对象可能还在（等 GC）
   - 上传文件不会立即被删除，要等 `__del__` 被调用

3. **"前端重连时从后端获取 widget state"**
   - 修正：重连成功时是恢复整个 AppSession 对象，其中包含 session_state
   - 但重连失败时前端会不会保留旧状态？这一点还不确定

---

## 七、代码索引

### 会话管理

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py) | L100-L140 | `connect_session` 重连逻辑 |
| [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py) | L170-L189 | `disconnect_session` 断连处理 |
| [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py) | L204-L226 | `close_session` 关停处理 |
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py) | L205-L207 | `__del__` 兜底清理 |
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py) | L292-L316 | `shutdown` 关停流程 |

### 存储相关

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| [memory_session_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_session_storage.py) | L41-L77 | MemorySessionStorage 实现 |
| [memory_uploaded_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py) | L38-L40 | 全局存储结构 |
| [runtime.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/runtime.py) | L218 | UploadedFileManager 单例 |

### 前端连接

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/WebsocketConnection.tsx) | L504-L544 | sessionId 传递 |
| [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py) | L67-L88 | 子协议解析 |
| [SessionInfo.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/SessionInfo.ts) | L46-L78 | 会话信息管理 |
| [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts) | L146-L207 | WidgetStateDict 实现 |
