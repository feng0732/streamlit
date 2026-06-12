# 文件上传边界情况与设计取舍分析

## 概述

本文是 [upload-chunk-processing.md](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/upload-chunk-processing.md) 的补充，聚焦于四个容易被忽略但至关重要的边界场景：

1. **删除与取消操作后的状态回收**
2. **会话生命周期中的文件清理**
3. **上传记录缺失时的应急降级**
4. **双通道传输与延迟同步的设计取舍**

---

## 一、删除与取消操作的状态回收

### 1.1 删除操作的完整链路

删除操作在 [FileUploader.tsx L421-L442](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx#L421-L442) 的 `deleteFile` 函数中实现，需要区分三种状态：

```typescript
const deleteFile = useCallback(
  (fileId: number): void => {
    if (disabled) return

    const file = getFile(fileId)
    if (isNullOrUndefined(file)) return

    // 情况1：上传中 → 先取消HTTP请求
    if (file.status.type === "uploading") {
      file.status.abortController.abort()
    }

    // 情况2：已上传 → 调用后端删除接口
    if (file.status.type === "uploaded" && file.status.fileUrls.deleteUrl) {
      void uploadClient.deleteFile(file.status.fileUrls.deleteUrl)
    }

    // 情况3：从本地状态移除（无论哪种状态都要做）
    removeFile(fileId)
  },
  [disabled, getFile, removeFile, uploadClient]
)
```

### 1.2 取消上传的状态流转

**取消触发点**：
- 用户点击已上传文件的删除按钮（对 uploading 状态的文件）
- 组件卸载时的隐式清理
- 重新上传同名文件时（先删后传）

**取消后的状态回收**：

```
发起取消 → AbortController.abort() → HTTP 请求中断 → Promise reject(AbortError)
                                                              ↓
                                                      错误回调中判断：
                                                      是 AbortError？→ 静默处理
                                                              ↓
                                                      不是？→ 设置 error 状态
```

关键代码在 [FileUploader.tsx L396-L410](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx#L396-L410)：

```typescript
.catch(err => {
  if (!(err instanceof DOMException && err.name === "AbortError")) {
    // 非取消导致的错误才显示错误状态
    updateFile(...)
  }
})
```

### 1.3 表单维度的状态同步

上传请求计数通过 `FileUploadClient.formsWithPendingRequests` Map 维护，影响表单提交按钮的可用性。

**计数变更时机**：
- 上传开始：`offsetPendingRequestCount(formId, +1)`
- 上传结束（成功/失败/取消）：`offsetPendingRequestCount(formId, -1)` —— 在 `.finally()` 中

**表单提交按钮禁用逻辑**（[FormSubmitButton.tsx L49-L58](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/Form/FormSubmitButton.tsx#L49-L58)）：

```typescript
const hasInProgressUpload = formsData.formsWithUploads.has(formId)
const isDisabled = disabled || hasInProgressUpload
```

> **设计意图**：防止用户在文件上传过程中提交表单，导致后端拿到不完整的文件列表。

---

## 二、会话清理机制

### 2.1 会话生命周期中的三个清理层级

| 层级 | 触发时机 | 清理内容 | 代码位置 |
|------|---------|---------|---------|
| **断开连接** | WebSocket 断开 | 停止脚本、断开文件监听器、清理缓存、保存会话到存储 | [websocket_session_manager.py L170-L189](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L170-L189) |
| **关闭会话** | 显式调用 close_session | 调用 session.shutdown()，彻底清理 | [websocket_session_manager.py L204-L213](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L204-L213) |
| **会话关停** | shutdown() 方法 | 移除上传文件、清理媒体文件引用、停止脚本 | [app_session.py L290-L316](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L290-L316) |

### 2.2 上传文件的清理时机

**关键发现**：上传文件在**会话关停**时才被清理，**断开连接**时不清理。

```
WebSocket 断开 → disconnect_session()
                    ↓
            保存 SessionInfo 到 SessionStorage
            （上传文件保留在内存中，支持重连）

显式关闭 → close_session()
                ↓
        session.shutdown()
                ↓
        uploaded_file_mgr.remove_session_files(session_id)
        （真正删除所有上传文件）
```

**shutdown 中的清理顺序**（[app_session.py L292-L316](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L292-L316)）：

```python
def shutdown(self) -> None:
    if self._state != AppSessionState.SHUTDOWN_REQUESTED:
        # 1. 清理上传文件
        self._uploaded_file_mgr.remove_session_files(self.id)

        # 2. 清理媒体文件引用和孤立文件
        if runtime.exists():
            rt = runtime.get_instance()
            rt.media_file_mgr.clear_session_refs(self.id)
            rt.media_file_mgr.remove_orphaned_files()

        # 3. 请求脚本停止
        self.request_script_stop()

        # 4. 标记为已请求关停
        self._state = AppSessionState.SHUTDOWN_REQUESTED

        # 5. 断开文件监听器
        self.disconnect_file_watchers()

        # 6. 清理会话缓存
        self.clear_session_caches()
```

### 2.3 断开 vs 关闭的设计考量

**为什么断开时不删除上传文件？**

- **重连支持**：用户网络波动导致 WebSocket 短暂断开，重连后上传的文件仍然可用
- **会话持久化**：`SessionStorage` 协议允许会话在断开后保留一段时间，等待重连
- **用户体验**：避免因为网络抖动导致用户上传的文件全部丢失

**什么时候真正删除？**

- 会话显式关闭（如用户点击"Stop"、页面关闭后超时）
- 会话存储的清理策略触发（不同 SessionStorage 实现有不同策略）

---

## 三、上传记录缺失的应急处理

### 3.1 DeletedFile：幽灵文件占位符

当 Widget 状态中引用的文件 ID 在文件管理器中找不到时，系统不会直接报错，而是用 `DeletedFile` 占位。

**DeletedFile 定义**（[uploaded_file_manager.py L47-L58](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/uploaded_file_manager.py#L47-L58)）：

```python
class DeletedFile(NamedTuple):
    """Represents a deleted file in deserialized values for st.file_uploader and
    st.camera_input.

    Return this from deserialize (so they can be used in session_state),
    when widget value contains file record that is missing from the storage.
    DeleteFile instances filtered out before return final value to the user
    in script, or before sending to frontend.
    """
    file_id: str
```

### 3.2 缺失检测与降级流程

在 [file_uploader.py L72-L103](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L72-L103) 的 `_get_upload_files` 函数中：

```python
def _get_upload_files(widget_value):
    # 1. 从 widget state 中提取所有 file_id
    uploaded_file_info = widget_value.uploaded_file_info

    # 2. 批量从文件管理器查询
    file_recs_list = ctx.uploaded_file_mgr.get_files(
        session_id=ctx.session_id,
        file_ids=[f.file_id for f in uploaded_file_info],
    )
    file_recs = {f.file_id: f for f in file_recs_list}

    # 3. 逐个比对，找不到的用 DeletedFile 占位
    collected_files: list[UploadedFile | DeletedFile] = []
    for f in uploaded_file_info:
        maybe_file_rec = file_recs.get(f.file_id)
        if maybe_file_rec is not None:
            uploaded_file = UploadedFile(maybe_file_rec, f.file_urls)
            collected_files.append(uploaded_file)
        else:
            collected_files.append(DeletedFile(f.file_id))  # 占位！

    return collected_files
```

### 3.3 DeletedFile 的三层过滤

DeletedFile 就像幽灵一样在系统中传递，但在三个关键节点被过滤掉：

| 过滤层 | 位置 | 说明 |
|-------|------|------|
| **序列化过滤** | [FileUploaderSerde.serialize](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L133-L149) | 发回前端的状态中不包含 DeletedFile |
| **类型校验过滤** | [FileUploaderSerde.deserialize](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L114-L116) | 对 DeletedFile 跳过文件名类型校验 |
| **用户返回值过滤** | 文件上传组件的 Python API 层 | 返回给用户脚本的列表中不包含 DeletedFile |

> **设计意图**：DeletedFile 是一个内部的"容忍标记"，它让系统在面对不一致状态时能够优雅降级，而不是直接崩溃。Widget state 和实际文件存储可能因为各种原因不同步（如服务重启、会话过期、并发修改），DeletedFile 提供了一个安全的中间状态。

### 3.4 可能导致记录缺失的场景

1. **服务重启**：内存中的 UploadedFileManager 清空，但浏览器的 Widget state 还在
2. **会话过期**：session 被清理后用户重新连接，但 widget 状态被保留
3. **并发删除**：一个请求删除文件，另一个请求正在读取
4. **URL 过期**：上传 URL 有有效期，但 file_id 已经写入 widget state

---

## 四、双通道传输与延迟同步的设计取舍

### 4.1 为什么是双通道（HTTP + WebSocket）？

**现状**：
- 文件数据 → HTTP PUT 请求
- 控制信令 & 状态同步 → WebSocket

**如果只用 WebSocket 上传文件，会有什么问题？**

| 问题 | 说明 |
|------|------|
| **帧大小限制** | WebSocket 单帧有大小限制，大文件需要分片，增加复杂度 |
| **Head-of-Line Blocking** | WebSocket 是有序的，大文件传输会阻塞其他控制消息 |
| **浏览器原生支持** | HTTP 有原生的 `XMLHttpRequest`/`fetch`，支持进度事件、取消、超时 |
| **服务器处理** | HTTP 文件上传是成熟模式，有现成的中间件和优化 |
| **流量控制** | HTTP 可以利用 TCP 流控，WebSocket 需要自己实现 |

**为什么不用纯 HTTP 轮询状态？**

- **延迟**：WebSocket 实时推送状态变化，轮询有延迟
- **效率**：频繁轮询浪费资源，WebSocket 是双向的
- **一致性**：其他 Widget 状态都走 WebSocket，文件上传也应该保持一致

**最终选择的权衡**：

> 大文件走 HTTP（利用成熟的传输能力），小状态走 WebSocket（利用实时双向通信）。两者通过 `file_id` 关联，形成一个"宽带传数据、窄带传信令"的经典架构。

### 4.2 延迟同步设计：为什么上传中不更新 Widget 状态？

**现状**：上传过程中，文件状态只存在于前端组件本地，不触发 Widget 状态同步，不触发脚本重运行。

**如果上传过程中同步状态，会怎样？**

1. **频繁重运行**：每上传一个文件（甚至每个进度更新）都触发一次脚本重运行，性能极差
2. **不一致状态**：脚本运行时，可能文件还没上传完，读到不完整的数据
3. **并发问题**：上传和脚本执行并发，可能产生竞态条件

**延迟同步的好处**：

| 好处 | 说明 |
|------|------|
| **性能优化** | 避免 N 次不必要的脚本重运行（N = 文件数 × 进度更新次数） |
| **状态一致性** | 脚本运行时，所有文件都已经上传完成，状态是确定的 |
| **用户可控** | 用户明确感知"上传中"和"上传完成"两个阶段 |
| **实现简单** | 不需要处理部分上传的中间状态 |

**延迟同步的代价**：

| 代价 | 说明 |
|------|------|
| **表单提交阻塞** | 上传完成前不能提交表单，用户需要等待 |
| **状态可见性** | 上传中的文件对 Python 脚本不可见（虽然本来也用不了） |
| **刷新丢失** | 上传过程中刷新页面，所有进度丢失 |

### 4.3 状态的三层存储模型

整个文件上传系统中，文件状态实际上存储在三个地方：

```
┌─────────────────────────────────────────────────────┐
│  第一层：前端组件本地状态（React useState）          │
│  - 包含 uploading / error / uploaded 三种状态       │
│  - 上传过程中的进度信息                             │
│  - 最实时，但刷新就丢                               │
└──────────────────────┬──────────────────────────────┘
                       │ 上传完成 + status === "ready"
                       │ 才同步到下一层
                       ↓
┌─────────────────────────────────────────────────────┐
│  第二层：WidgetStateManager + WebSocket             │
│  - 只包含已上传完成的文件元数据                     │
│  - 持久化在前端，刷新后从后端恢复                   │
│  - 触发脚本重运行                                   │
└──────────────────────┬──────────────────────────────┘
                       │ 脚本运行时反序列化
                       ↓
┌─────────────────────────────────────────────────────┐
│  第三层：MemoryUploadedFileManager（后端内存）      │
│  - 存储文件的实际字节数据                           │
│  - 与会话生命周期绑定                               │
│  - 通过 HTTP PUT 直接写入                           │
└─────────────────────────────────────────────────────┘
```

**设计要点**：
- 每一层都有不同的生命周期和一致性保证
- 状态只能从下层向上层同步，不能反向
- 上层状态可能因为各种原因与下层不一致（所以需要 DeletedFile 机制）

---

## 五、其他边界情况

### 5.1 flushSync 的必要性

在 [FileUploader.tsx L172-L182](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx#L172-L182) 中使用了 `flushSync`：

```typescript
const setFilesImmediate = useCallback((updater: FilesUpdater): void => {
  /* Using flushSync here because we need the state to be immediately updated
   * before any subsequent file upload operations occur. Without this, React
   * can defer the commit and our upload callbacks (completion or abort) may
   * run while filesRef.current still points to the previous state. Those
   * callbacks rely on filesRef.current to locate the in-flight upload, so
   * deferring the update would cause them to no-op.
   */
  flushSync(() => {
    setFiles(updater)
  })
}, [setFiles])
```

**问题场景**：
1. 用户触发上传 → React 异步更新 state → 发起 HTTP 请求
2. 请求很快完成（或很快失败/取消）→ 回调执行
3. 此时 React 的 state 更新还没 commit → `filesRef.current` 还是旧值
4. 回调找不到对应的上传条目 → 静默失败

**解决方案**：用 `flushSync` 强制同步提交状态更新，确保回调执行时能找到正确的文件条目。

### 5.2 同名文件重新上传

在 [FileUploader.tsx L487-L496](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx#L487-L496) 中：

```typescript
if (existingFile) {
  setForceUpdatingStatus(true)
  try {
    deleteFile(existingFile.id)
  } finally {
    setForceUpdatingStatus(false)
  }
}
```

**设计意图**：当用户上传与已有文件同名的新文件时，先删除旧的再上传新的。`setForceUpdatingStatus(true)` 是为了在删除+上传这个短暂的窗口期内，保持 "updating" 状态，避免状态闪烁。

### 5.3 WebSocket 重连后的上传状态

WebSocket 断开重连时，App 组件的 [handleConnectionStateChanged](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/app/src/App.tsx#L862-L944) 会：

1. 如果之前是上传中状态 → 没有显式处理，上传中的文件状态保留在前端
2. 如果上传请求还在进行 → HTTP 请求不受 WebSocket 断开影响，继续上传
3. 如果上传请求已完成但还没同步 Widget 状态 → 重连后会继续同步

> **注意**：HTTP 上传通道和 WebSocket 控制通道是独立的。WebSocket 断开不影响正在进行的 HTTP 上传。这是双通道架构的一个重要特性——控制通道故障不影响数据通道传输。

---

## 六、设计取舍总结表

| 设计决策 | 收益 | 代价 | 适用场景 |
|---------|------|------|---------|
| **双通道传输**（HTTP+WebSocket） | 利用 HTTP 成熟的大文件传输能力 + WebSocket 实时信令 | 架构复杂，需要处理两个通道的状态同步 | 大文件 + 实时状态更新 |
| **延迟状态同步** | 避免频繁重运行，保证状态一致性 | 上传中无法提交表单，刷新丢失进度 | 交互式数据应用，文件作为输入 |
| **内存存储** | 实现简单，速度快，自动清理 | 不能持久化，重启丢失，占用内存 | 临时文件、单次会话使用 |
| **DeletedFile 占位** | 优雅降级，不一致时不崩溃 | 增加了系统复杂度，需要多层过滤 | 分布式/并发系统中容忍状态不一致 |
| **断开不删文件** | 支持重连，网络波动不丢文件 | 内存占用时间更长 | WebSocket 不稳定的网络环境 |
| **表单上传计数** | 防止提交不完整的表单 | 增加了跨组件状态同步的复杂度 | 表单内包含文件上传的场景 |
| **AbortController 取消** | 原生支持，能真正中断 TCP 连接 | 需要正确处理 AbortError 分支 | 用户主动取消上传 |

---

## 七、代码索引

### 边界情况相关

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| [FileUploader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx) | `deleteFile` L421-L442 | 删除操作实现 |
| [FileUploader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx) | `setFilesImmediate` L172-L182 | flushSync 强制同步 |
| [FileUploadClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/FileUploadClient.ts) | `offsetPendingRequestCount` L184-L188 | 表单上传计数 |
| [FormSubmitButton.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/Form/FormSubmitButton.tsx) | `hasInProgressUpload` L49 | 表单禁用逻辑 |
| [uploaded_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/uploaded_file_manager.py) | `DeletedFile` L47-L58 | 缺失文件占位符 |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | `_get_upload_files` L72-L103 | 缺失检测逻辑 |

### 会话清理相关

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py) | `shutdown` L290-L316 | 会话关停清理 |
| [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py) | `disconnect_session` L170-L189 | WebSocket 断开处理 |
| [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py) | `close_session` L204-L213 | 会话关闭处理 |
| [memory_uploaded_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py) | `remove_session_files` L74-L76 | 会话文件清理 |
