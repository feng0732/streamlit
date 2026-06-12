# 上传记录恢复机制：最终分析报告

## 核心问题

前端断线后，当无法连接到旧会话时，上传控件的状态是否会被带到新会话中？DeletedFile 是如何产生的？

---

## 一、完整端到端流程（代码可证）

### 1.1 流程总览

```
前端断线 → 重连 → 后端创建新会话 → 前端回传旧widget状态 →
  → 新会话注入旧状态 → 组件获取文件 → 产生DeletedFile
```

### 1.2 步骤详解

#### 步骤1：前端断线检测

**代码位置**：[WebsocketConnection.tsx L366-L371](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/WebsocketConnection.tsx#L366-L371)

```typescript
case ConnectionState.CONNECTED:
  if (event === "CONNECTION_CLOSED" || event === "CONNECTION_ERROR") {
    this.setFsmState(ConnectionState.PINGING_SERVER)  // 进入重连状态
    return
  }
```

**断连时的状态处理**：[App.tsx L908-L924](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/app/src/App.tsx#L908-L924)

```typescript
} else {
  // 从 CONNECTED 状态转到其他状态 = 正在断连
  if (this.state.connectionState === ConnectionState.CONNECTED) {
    this.hostCommunicationMgr.sendMessageToHost({
      type: "WEBSOCKET_DISCONNECTED",
      attemptingToReconnect: newState !== ConnectionState.DISCONNECTED_FOREVER,
    })
  }

  if (this.sessionInfo.isSet) {
    this.sessionInfo.disconnect()  // ✅ 保存 sessionId 到 _last
  }

  this.backendOperationClient.cleanup()
}
```

**代码可证事实**：
- 断线时调用 `sessionInfo.disconnect()`，将当前 sessionId 保存到 `_last`
- **WidgetStateManager 的状态不会被清空**，仍然保留在内存中
- 没有任何代码在断线时调用 `widgetStates.clear()`

#### 步骤2：重连成功并触发 rerun

**代码位置**：[App.tsx L862-L907](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/app/src/App.tsx#L862-L907)

```typescript
handleConnectionStateChanged = (newState: ConnectionState): void => {
  if (newState === ConnectionState.CONNECTED) {
    LOG.info("Reconnected to server.")
    
    // 决定是否触发 rerun 的条件
    const lastRunWasInterrupted = this.state.scriptRunState === ScriptRunState.RUNNING
    const wasRerunRequested = this.state.scriptRunState === ScriptRunState.RERUN_REQUESTED

    if (
      !this.sessionInfo.last ||           // 首次连接
      lastRunWasInterrupted ||             // 上次运行被中断
      wasRerunRequested ||                 // 显式请求重连
      this.state.fragmentIdsThisRun.length > 0 ||  // 使用了 fragments
      this.state.autoReruns.length > 0             // 有自动重跑
    ) {
      LOG.info("Requesting a script run.")
      // ✅ 关键：发送当前所有 widget 状态
      this.widgetMgr.sendUpdateWidgetsMessage(undefined)
      this.setState({ dialog: null })
    }
  }
}
```

**sendUpdateWidgetsMessage 的实现**：[WidgetStateManager.ts L831-L841](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts#L831-L841)

```typescript
public sendUpdateWidgetsMessage(
  fragmentId: string | undefined,
  isAutoRerun: boolean | undefined = undefined
): void {
  this.props.sendRerunBackMsg(
    this.widgetStates.createWidgetStatesMsg(),  // ✅ 创建包含所有状态的消息
    fragmentId,
    undefined,
    isAutoRerun
  )
}
```

**createWidgetStatesMsg 的实现**：[WidgetStateManager.ts L187-L191](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts#L187-L191)

```typescript
public createWidgetStatesMsg(): WidgetStates {
  const msg = new WidgetStates()
  this.widgetStates.forEach(value => msg.widgets.push(value))
  return msg
}
```

**代码可证事实**：
- 重连成功后，如果满足条件，会发送 `sendUpdateWidgetsMessage`
- 该消息包含 **WidgetStateManager 中当前所有的 widget 状态**（包括上传控件的 file_id）
- 这些状态是断线前的旧状态

#### 步骤3：后端创建新会话

**代码位置**：[websocket_session_manager.py L100-L140](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L100-L140)

```python
def connect_session(self, ...) -> str:
    # 1. 从 SessionStorage 查找旧会话
    session_info = self._session_storage.get(existing_session_id)
    
    if isinstance(session_info, SessionInfo):
        # 找到旧会话，恢复它
        existing_session = session_info.session
        # ... 恢复逻辑 ...
        return existing_session.id
    
    # 2. ❌ 没找到旧会话（TTL过期或已关闭），创建全新会话
    session = AppSession(
        id=session_id,
        # ... 其他参数 ...
    )
    # ✅ 新会话有全新的 session_id、全新的 _session_state
    # ✅ UploadedFileManager 中没有新 session_id 的任何文件记录
```

**代码可证事实**：
- 如果旧会话已过期（TTL > 2分钟）或已被关闭，后端会创建**全新的 AppSession**
- 新会话有全新的 `session_id`
- 新会话的 `_session_state` 是空的
- `UploadedFileManager` 中没有新 `session_id` 的文件记录

#### 步骤4：前端回传旧 widget 状态

**代码位置**：[App.tsx L1994-L2007](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/app/src/App.tsx#L1994-L2007)

```typescript
this.sendBackMsg(
  new BackMsg({
    rerunScript: {
      queryString,
      widgetStates,      // ✅ 旧的 widget 状态（包含 file_id）
      pageScriptHash,
      pageName,
      fragmentId,
      isAutoRerun,
      cachedMessageHashes,
      contextInfo,
    },
  })
)
```

**后端接收并处理**：[app_session.py L338-L346](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L338-L346)

```python
def handle_backmsg(self, msg: BackMsg) -> None:
    msg_type = msg.WhichOneof("type")
    if msg_type == "rerun_script":
        self._handle_rerun_script_request(msg.rerun_script)
```

**提取 widget_states**：[app_session.py L424-L462](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L424-L462)

```python
def request_rerun(self, client_state: ClientState | None) -> None:
    if client_state:
        rerun_data = RerunData(
            query_string=client_state.query_string,
            widget_states=client_state.widget_states,  # ✅ 旧的 widget 状态
            # ... 其他字段 ...
        )
```

**代码可证事实**：
- 前端发送的 `rerunScript` 消息中包含完整的 `widgetStates`
- 这些状态被封装到 `RerunData` 中传递给 ScriptRunner

#### 步骤5：新会话注入旧状态

**代码位置**：[script_runner.py L708-L711](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L708-L711)

```python
if rerun_data.widget_states is not None:
    self._session_state.on_script_will_rerun(
        rerun_data.widget_states  # ✅ 旧的 widget 状态
    )
```

**on_script_will_rerun 的实现**：[session_state.py L641-L651](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py#L641-L651)

```python
def on_script_will_rerun(self, latest_widget_states: WidgetStatesProto) -> None:
    self._reset_triggers()
    self._compact_state()
    self.set_widgets_from_proto(latest_widget_states)  # ✅ 注入旧状态
    self._call_callbacks()
```

**set_widgets_from_proto 的实现**：[session_state.py L636-L639](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py#L636-L639)

```python
def set_widgets_from_proto(self, widget_states: WidgetStatesProto) -> None:
    for state in widget_states.widgets:
        self._new_widget_state.set_widget_from_proto(state)
```

**set_widget_from_proto 的实现**：[session_state.py L236-L238](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py#L236-L238)

```python
def set_widget_from_proto(self, widget_state: WidgetStateProto) -> None:
    self[widget_state.id] = Serialized(widget_state)  # ✅ 保存为序列化形式
```

**代码可证事实**：
- 前端传来的旧 widget 状态被写入新会话的 `_new_widget_state`
- 保存形式是 `Serialized`（原始 protobuf 对象），尚未反序列化
- 这意味着**新会话的 session_state 中现在包含了旧会话的 widget 状态**

#### 步骤6：组件获取文件时产生 DeletedFile

**代码位置**：[file_uploader.py L72-L103](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L72-L103)

```python
def _get_upload_files(
    widget_value: FileUploaderStateProto | None,
) -> list[UploadedFile | DeletedFile]:
    # widget_value 包含旧的 file_id（来自前端回传的状态）
    
    file_recs_list = ctx.uploaded_file_mgr.get_files(
        session_id=ctx.session_id,  # ✅ 新会话的 session_id
        file_ids=[f.file_id for f in uploaded_file_info],
    )
    
    for f in uploaded_file_info:
        maybe_file_rec = file_recs.get(f.file_id)
        if maybe_file_rec is not None:
            collected_files.append(UploadedFile(maybe_file_rec, f.file_urls))
        else:
            # ❌ 新会话的 UploadedFileManager 中没有这个 file_id
            collected_files.append(DeletedFile(f.file_id))  # ✅ 产生 DeletedFile
```

**deserialize 时的处理**：[file_uploader.py L111-L131](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L111-L131)

```python
class FileUploaderSerde:
    @staticmethod
    def deserialize(ui_value: FileUploaderStateProto | None) -> SomeUploadedFiles:
        return _get_upload_files(ui_value)  # ✅ 调用 _get_upload_files
```

**代码可证事实**：
- 当脚本运行到 `st.file_uploader` 组件时，会调用 `register_widget`
- `register_widget` 会调用 `deserializer` 反序列化 widget state
- `deserialize` 调用 `_get_upload_files`
- `_get_upload_files` 用**新会话的 session_id** 去 `UploadedFileManager` 查找文件
- 但 `UploadedFileManager` 中只有旧 session_id 的文件记录
- **所以比对失败，产生 DeletedFile**

---

## 二、DeletedFile 产生的完整时序

### 2.1 时序图

```
时间轴：

T0: 用户上传文件
  → 前端 WidgetStateManager 保存 file_id = "abc123"
  → 后端 UploadedFileManager 保存 (session_id="old123", file_id="abc123", data)

T1: 前端断线（WebSocket 关闭）
  → sessionInfo.disconnect()，保存 _last.sessionId = "old123"
  → WidgetStateManager 状态不变，仍然有 file_id = "abc123"
  → 后端 disconnect_session，保存到 SessionStorage（TTL=2min）

T2: 2分10秒后，前端尝试重连
  → SessionStorage 中的旧会话已过期（TTL=2min）
  → 后端创建新会话，session_id = "new456"
  → 新会话的 _session_state 为空
  → UploadedFileManager 中没有 "new456" 的任何记录

T3: 前端发送 rerunScript 消息
  → 包含 widgetStates: [{id: "uploader_1", fileUploaderStateValue: {uploaded_file_info: [{file_id: "abc123"}]}}]

T4: 后端处理 rerunScript
  → 新会话的 _new_widget_state["uploader_1"] = Serialized(widget_state)

T5: 脚本运行到 st.file_uploader
  → register_widget 调用 deserialize
  → deserialize 调用 _get_upload_files
  → _get_upload_files 用 session_id="new456" 查找 file_id="abc123"
  → UploadedFileManager 返回空
  → 产生 DeletedFile("abc123")
```

### 2.2 产生条件（代码可证）

DeletedFile 产生的**必要且充分条件**：

| 条件 | 说明 | 代码依据 |
|------|------|---------|
| 1. 前端断线后重连 | WebSocket onclose/onerror 触发 | [WebsocketConnection.tsx L569-L589](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/WebsocketConnection.tsx#L569-L589) |
| 2. 旧会话已过期或关闭 | TTL > 2分钟 或 已调用 close_session | [memory_session_storage.py L41-L44](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_session_storage.py#L41-L44) |
| 3. 后端创建新会话 | connect_session 中找不到旧会话 | [websocket_session_manager.py L137-L140](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L137-L140) |
| 4. 前端回传旧 widget 状态 | sendUpdateWidgetsMessage 发送所有状态 | [WidgetStateManager.ts L831-L841](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts#L831-L841) |
| 5. 旧状态包含 file_id | 上传控件的 state 中有 uploaded_file_info | [WidgetStateManager.ts L187-L191](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts#L187-L191) |
| 6. UploadedFileManager 中无新 session_id 的记录 | 新会话还没上传过文件 | [memory_uploaded_file_manager.py L38-L40](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py#L38-L40) |

**所有 6 个条件同时满足时，必然产生 DeletedFile。**

---

## 三、旧状态回传的关键链路

### 3.1 前端状态持久化分析

**代码可证事实**：

| 存储位置 | 是否持久化 | 断线后是否保留 | 代码依据 |
|---------|-----------|---------------|---------|
| WidgetStateManager.widgetStates | ❌ 内存 Map | ✅ 保留（直到页面刷新） | [WidgetStateManager.ts L147](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts#L147) |
| SessionInfo._last | ❌ 内存变量 | ✅ 保留（disconnect 时设置） | [SessionInfo.ts L54](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/SessionInfo.ts#L54) |
| localStorage | ✅ 持久化 | ❌ 不存 widget state | 全量搜索无相关代码 |
| sessionStorage | ✅ 持久化 | ❌ 不存 widget state | 全量搜索无相关代码 |

**结论（代码可证）**：
- widget state 只存在于前端内存中
- 断线后**不会**被自动清空
- 页面刷新后会丢失

### 3.2 重连时的状态发送逻辑

**触发条件（代码可证）**：[App.tsx L888-L896](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/app/src/App.tsx#L888-L896)

```typescript
if (
  !this.sessionInfo.last ||           // 首次连接
  lastRunWasInterrupted ||             // 上次运行被中断
  wasRerunRequested ||                 // 显式请求重连
  this.state.fragmentIdsThisRun.length > 0 ||  // 使用了 fragments
  this.state.autoReruns.length > 0             // 有自动重跑
) {
  this.widgetMgr.sendUpdateWidgetsMessage(undefined)
}
```

**代码可证事实**：
- 只要满足以上任一条件，就会发送当前所有 widget 状态
- 对于断线重连场景，`lastRunWasInterrupted` 通常为 true（因为脚本运行被中断）
- 所以**几乎所有断线重连场景都会触发旧状态回传**

### 3.3 后端新会话的状态注入

**关键代码**：[session_state.py L636-L651](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py#L636-L651)

```python
def set_widgets_from_proto(self, widget_states: WidgetStatesProto) -> None:
    for state in widget_states.widgets:
        self._new_widget_state.set_widget_from_proto(state)

def on_script_will_rerun(self, latest_widget_states: WidgetStatesProto) -> None:
    self._reset_triggers()
    self._compact_state()
    self.set_widgets_from_proto(latest_widget_states)  # ✅ 注入旧状态
    self._call_callbacks()
```

**代码可证事实**：
- `set_widgets_from_proto` 不会校验状态的有效性
- 它只是简单地将前端传来的状态保存到 `_new_widget_state`
- **没有任何过滤或校验逻辑**来区分"新会话"和"旧状态"

---

## 四、代码可证 vs 推测

### 4.1 代码直接证明的结论（100% 确定）

| 结论 | 代码依据 |
|------|---------|
| 前端断线时 WidgetStateManager 的状态不会被清空 | 无 `clear()` 调用，状态保留在内存 |
| 重连成功后会发送所有 widget 状态给后端 | [App.tsx L898](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/app/src/App.tsx#L898) |
| 旧会话 TTL 过期后后端会创建全新会话 | [websocket_session_manager.py L137-L140](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L137-L140) |
| 新会话的 UploadedFileManager 中没有旧文件记录 | 双层字典结构，按 session_id 分组 |
| 前端的旧 widget 状态会被注入新会话 | [session_state.py L636-L651](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py#L636-L651) |
| DeletedFile 产生于 `_get_upload_files` 比对失败时 | [file_uploader.py L95-L103](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L95-L103) |
| DeletedFile 在 serialize 时被过滤，前端看不到 | [file_uploader.py L142-L143](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L142-L143) |
| DeletedFile 在 API 层被过滤，用户看不到 | [file_uploader.py L606-L611](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L606-L611) |
| 前端 widget state 不持久化到 localStorage | 全量搜索无相关代码 |

### 4.2 合理推测（高置信度，但代码不直接证明）

| 结论 | 推测依据 | 置信度 |
|------|---------|--------|
| 页面刷新后 widget state 会丢失 | 只有内存存储，无持久化 | 🔶 90% |
| 正常使用中 DeletedFile 几乎不可感知 | 两层过滤机制 | 🔶 85% |
| 旧状态回传是有意设计的，不是 bug | 代码链路清晰完整，有明确的触发条件 | 🔶 80% |
| 设计意图是"用户体验优先"——尽量保留用户输入 | 重连后不需要重新填写表单 | 🔶 75% |
| DeletedFile 是为了处理这种"状态跨会话传递"的边界情况 | 唯一出现的场景就是这个链路 | 🔶 80% |

### 4.3 还需要更多证据的问题

| 问题 | 为什么不确定 | 如何验证 |
|------|-------------|---------|
| 是否有其他场景也会产生 DeletedFile？ | 理论上可能有竞态，但代码中没有明确的其他路径 | 仔细检查所有调用 `_get_upload_files` 的地方 |
| 前端是否有某些场景下会清空 WidgetStateManager？ | 目前只看到 `removeInactive` 和组件卸载时的清理 | 全量搜索 `clear()` 和 `deleteState()` 调用 |
| 新会话创建时是否有重置 widget state 的机制？ | 目前没看到，但可能有其他入口 | 检查 AppSession 初始化和 `handleNewSession` 的完整逻辑 |

---

## 五、关键修正与澄清

### ❌ 之前的错误表述

1. **"关停后重连会产生 DeletedFile"**
   - 修正：关停后会话不再运行脚本，所以不会调用 `_get_upload_files`
   - **只有当新会话创建并运行脚本，但前端回传了旧状态时，才会产生 DeletedFile**

2. **"DeletedFile 几乎不会出现"**
   - 修正：**断线 + TTL 过期 + 重连** 这个场景是真实存在的，且必然产生 DeletedFile
   - 但由于两层过滤，用户和前端都感知不到

3. **"前端重连失败后状态会重置"**
   - 修正：前端状态**不会**重置，而是会被发送给新会话
   - 这正是 DeletedFile 产生的根本原因

---

## 六、DeletedFile 的三层消化机制回顾

虽然 DeletedFile 会在系统内部产生，但它对用户和前端是完全透明的：

| 层级 | 处理方式 | 代码位置 |
|------|---------|---------|
| deserialize | 只跳过类型校验，不删除 | [file_uploader.py L111-L131](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L111-L131) |
| serialize | 过滤 DeletedFile，不回传前端 | [file_uploader.py L133-L150](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L133-L150) |
| API 层 | 返回给用户前最后一次过滤 | [file_uploader.py L606-L611](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L606-L611) |

所以虽然 DeletedFile 确实会在特定场景下产生，但它是一个**内部的、优雅降级**的机制，不会对用户造成影响。

---

## 七、完整代码索引

### 前端断线与重连

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/WebsocketConnection.tsx) | L366-L371 | 断线检测，状态转换到 PINGING_SERVER |
| [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/WebsocketConnection.tsx) | L569-L589 | WebSocket onclose/onerror 处理 |
| [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/app/src/App.tsx) | L862-L924 | 连接状态变化处理，断线时保存 sessionId |
| [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/app/src/App.tsx) | L888-L898 | 重连成功后触发 rerun，发送 widget 状态 |
| [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts) | L831-L841 | `sendUpdateWidgetsMessage` 发送所有状态 |
| [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts) | L187-L191 | `createWidgetStatesMsg` 创建状态消息 |
| [SessionInfo.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/SessionInfo.ts) | L80-L88 | `disconnect()` 保存旧 sessionId |

### 后端新会话创建

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py) | L100-L140 | `connect_session` 查找或创建会话 |
| [memory_session_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_session_storage.py) | L41-L44 | TTLCache，默认 2 分钟过期 |
| [memory_uploaded_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py) | L38-L40 | 双层字典结构，按 session_id 分组 |

### 状态注入与 DeletedFile 产生

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py) | L338-L346 | `handle_backmsg` 处理 rerun_script |
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py) | L424-L462 | `request_rerun` 提取 widget_states |
| [script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py) | L708-L711 | 调用 `on_script_will_rerun` |
| [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py) | L641-L651 | `on_script_will_rerun` 注入旧状态 |
| [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py) | L636-L639 | `set_widgets_from_proto` 保存状态 |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | L72-L103 | `_get_upload_files` 产生 DeletedFile |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | L111-L131 | `deserialize` 调用 `_get_upload_files` |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | L133-L150 | `serialize` 过滤 DeletedFile |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | L606-L611 | API 层最后一次过滤 |
