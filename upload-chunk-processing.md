# 文件上传分片处理代码链路分析

## 概述

Streamlit 的文件上传机制采用 **整体上传 + 内存存储** 的架构。虽然不是传统意义上的"分片上传"（将大文件拆成多个 chunk 分别上传），但整个流程可以从四个阶段来理解：**分片接收**、**临时存储**、**进度同步**、**合并校验**。

整体流程如下：

```
前端选择文件 → WebSocket 请求上传URL → 获取上传URL → HTTP PUT 上传文件
    ↓
后端接收文件 → 内存存储 → 前端状态同步 → Widget 状态更新 → Python 脚本可用
```

---

## 一、分片接收：文件数据传输链路

### 1.1 前端发起上传请求

文件上传的起点是用户在 [FileUploader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx) 组件中选择文件。核心入口是 `dropHandler` 函数（第 448-539 行）。

**流程步骤：**

1. **获取上传 URL**：通过 WebSocket 通道向后端请求上传地址
   - 调用 [FileUploadClient.fetchFileURLs()](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/FileUploadClient.ts#L144-L156)
   - 生成唯一 requestId（uuidv4），将 Promise resolver 存入 `pendingFileURLsRequests` 映射
   - 通过 `requestFileURLs` 回调发送 WebSocket 消息

2. **后端生成上传 URL**：在 [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L971-L989) 的 `_handle_file_urls_request` 方法中
   - 调用 `uploaded_file_mgr.get_upload_urls()` 生成上传/删除 URL
   - 每个文件分配一个唯一的 `file_id`（uuid4）
   - URL 格式：`/_stcore/upload_file/{session_id}/{file_id}`

3. **前端执行文件上传**：拿到 URL 后调用 `uploadFile` 函数
   - 创建 `UploadFileInfo` 对象，状态设为 `uploading`
   - 调用 [FileUploadClient.uploadFile()](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/FileUploadClient.ts#L102-L119)
   - 最终由 [DefaultStreamlitEndpoints.uploadFileUploaderFile()](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts#L269-L310) 执行 HTTP PUT 请求

### 1.2 后端接收文件数据

后端通过 Starlette 路由处理上传请求，定义在 [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py) 的 `_upload_put` 函数（第 656-714 行）。

**接收流程：**

```python
# 1. XSRF 校验
_check_xsrf(request)

# 2. Session 校验
session_id = request.path_params["session_id"]
if not runtime.is_active_session(session_id):
    raise HTTPException(status_code=400, detail="Invalid session_id")

# 3. 快速大小校验（通过 Content-Length header）
content_length = request.headers.get("content-length")
if content_length and int(content_length) > max_size_bytes:
    raise HTTPException(status_code=413, detail="File too large")

# 4. 解析表单数据，读取文件内容
form = await request.form()
upload = uploads[0]
data = await upload.read()

# 5. 实际大小校验
if len(data) > max_size_bytes:
    raise HTTPException(status_code=413, detail="File too large")
```

> **注意**：当前实现是将整个文件读入内存后再校验大小。代码注释中标注了 TODO：建议改用流式方式，在读取过程中超出大小就立即拒绝。

---

## 二、临时存储：内存文件管理

### 2.1 存储架构

文件上传完成后，存储在 [MemoryUploadedFileManager](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py) 中。

**数据结构：**

```python
class MemoryUploadedFileManager(UploadedFileManager):
    # 双层字典：session_id -> file_id -> UploadedFileRec
    file_storage: dict[str, dict[str, UploadedFileRec]]
```

[UploadedFileRec](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/uploaded_file_manager.py#L30-L36) 是一个不可变的命名元组：

```python
class UploadedFileRec(NamedTuple):
    file_id: str      # 文件唯一标识
    name: str         # 文件名
    type: str         # MIME 类型
    data: bytes       # 文件原始字节数据
```

### 2.2 存储操作

| 操作 | 方法 | 说明 |
|------|------|------|
| 添加文件 | [add_file()](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py#L81-L97) | 按 session_id 分组存储 |
| 获取文件 | [get_files()](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py#L46-L72) | 根据 file_ids 批量查询 |
| 删除文件 | [remove_file()](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py#L99-L102) | 删除单个文件 |
| 清理会话 | [remove_session_files()](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py#L74-L76) | 会话结束时清理所有文件 |

### 2.3 存储特点

- **纯内存存储**：文件数据以 bytes 形式保存在内存中，没有落盘
- **会话隔离**：每个 session 的文件独立存储，互不干扰
- **线程安全**：使用 `defaultdict` 和字典操作，文档标注可安全地从多线程调用
- **生命周期**：与会话绑定，会话结束时自动清理

---

## 三、进度同步：上传状态管理

### 3.1 前端进度跟踪

文件上传进度通过 axios 的 `onUploadProgress` 回调获取，但不同组件的使用方式不同：

| 组件 | 是否使用进度回调 | 代码位置 |
|------|-----------------|----------|
| FileUploader | ❌ 未使用（传 `undefined`） | [FileUploader.tsx L392](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx#L392) |
| CameraInput | ✅ 使用，显示上传进度条 | [CameraInput.tsx L367-L393](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/CameraInput/CameraInput.tsx#L367-L393) |
| AudioInput | ❌ 未使用 | 通过 `uploadFiles` 工具函数 |

**进度更新机制（以 CameraInput 为例）：**

```typescript
const onUploadProgress = useCallback(
  (progressEvent: AxiosProgressEvent, fileId: number) => {
    const file = getFile(fileId)
    if (isNullOrUndefined(file) || file.status.type !== "uploading") {
      return
    }
    const progress = progressEvent.total
      ? (progressEvent.loaded / progressEvent.total) * 100
      : 0
    updateFile(
      fileId,
      file.setStatus({
        type: "uploading",
        abortController: (file.status as UploadingStatus).abortController,
        progress,
      })
    )
  },
  [getFile, updateFile]
)
```

### 3.2 文件状态流转

[UploadFileInfo](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/shared/UploadedFile/UploadFileInfo.ts) 定义了三种状态：

```
uploading → uploaded
     ↓
   error
```

- **uploading**：上传中，包含 `abortController` 和 `progress`
- **uploaded**：已上传，包含 `fileId` 和 `fileUrls`
- **error**：上传失败，包含 `errorMessage`

### 3.3 Widget 状态同步

上传状态通过 [WidgetStateManager](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts) 同步到后端。

**同步时机**（在 [FileUploader.tsx L279-L296](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx#L279-L296)）：

```typescript
useEffect(() => {
  // 只有当状态为 "ready"（没有文件在上传/删除）时才同步
  if (status !== "ready") {
    return
  }
  // 将已上传的文件信息同步到 Widget 状态
  const newWidgetValue = toWidgetState(files)
  widgetMgr.setFileUploaderStateValue(element, newWidgetValue, { fromUi: true }, fragmentId)
}, [status, files, widgetMgr, element, fragmentId])
```

> **关键设计**：上传过程中不同步 Widget 状态，只有当所有文件都上传完成（status === "ready"）时，才一次性将所有已上传文件的信息同步给后端。

### 3.4 状态同步机制

`WidgetStateManager.setFileUploaderStateValue()` 会触发：
1. 更新本地 widget state
2. 调用 `onWidgetValueChanged()` 通知状态变化
3. 通过 WebSocket 将新状态发送给后端
4. 后端触发脚本重运行（rerun）

---

## 四、合并校验：完整性与安全性校验

### 4.1 前端校验

**文件类型校验**：
- 在 [FileHelper.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/util/FileHelper.ts) 的 `isFileTypeAllowed` 函数中实现
- 支持扩展名、MIME 类型、MIME 通配符、类别快捷方式（image/audio/video/text）

**文件大小校验**：
- 前端通过 `maxUploadSizeMb` 属性限制
- react-dropzone 库负责大小检查

### 4.2 后端校验

**XSRF 防护**（[starlette_routes.py L622-L634](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L622-L634)）：
- 对比请求头 `X-Xsrftoken` 和 Cookie 中的值
- 采用 double-submit cookie 模式

**Session 校验**：
- 验证 URL 中的 `session_id` 是否为活跃会话
- 防止越权访问其他会话的文件

**文件大小双重校验**：
- 快速校验：读取 body 前检查 `Content-Length` header
- 实际校验：读取完整个文件后检查实际字节数

**文件名校验**（[file_uploader_utils.py L118-L157](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/lib/file_uploader_utils.py#L118-L157)）：
- 禁止 null 字节（防止绕过文件类型检查）
- 仅当配置了纯扩展名类型时才做服务端校验
- MIME 类型信任前端过滤结果

### 4.3 反序列化校验

在 [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) 的 `FileUploaderSerde.deserialize` 中：

1. 根据 widget state 中的 file_ids 从 `uploaded_file_mgr` 取出文件记录
2. 对比 widget state 中的文件列表和实际存储的文件
3. 找不到的文件标记为 `DeletedFile`（用于处理已过期/被清理的文件）
4. 对每个找到的文件再次做文件名类型校验

---

## 五、协作流程全景

### 5.1 完整时序图

```
用户           FileUploader    FileUploadClient    WebSocket     AppSession    UploadManager
 |                |                 |                |              |              |
 |--选择文件------>|                |                |              |              |
 |                |--fetchFileURLs->|                |              |              |
 |                |                 |--requestFileURLs(BackMsg)-->|              |
 |                |                 |                |--handle---->|              |
 |                |                 |                |              |--get_upload_urls()-->|
 |                |                 |                |              |<--返回URLs---|
 |                |                 |<---ForwardMsg(file_urls_response)---------|
 |                |<--resolve Promise---|           |              |              |
 |                |--uploadFile()--->|                |             |              |
 |                |                 |--HTTP PUT /_stcore/upload_file/{session}/{file_id}-->|
 |                |                 |                |              |  接收文件数据  |
 |                |                 |                |              |  XSRF校验     |
 |                |                 |                |              |  大小校验     |
 |                |                 |                |              |--add_file()-->|
 |                |                 |<----- 204 No Content -------|              |
 |                |<--完成回调------|                |              |              |
 |  更新UI状态    |                |                |              |              |
 |                |--setFileUploaderStateValue---->|              |              |
 |                |                 |--WebSocket 发送 widget state -->|         |
 |                |                 |                |              |  触发rerun   |
 |                |                 |                |              |--deserialize->|
 |                |                 |                |              |<--返回文件对象|
```

### 5.2 关键协作点

**1. URL 请求-响应配对**
- 前端：`pendingFileURLsRequests` Map 存储 requestId -> Promise resolver
- 后端：通过 `response_id` 回传对应关系
- 机制：WebSocket 全双工通信下的请求-响应配对模式

**2. 上传中状态的本地管理**
- 上传中的文件只存在于前端组件的本地 state
- 不会同步到 WidgetStateManager，也不会触发脚本重运行
- 上传完成/失败后才更新 Widget 状态

**3. 文件数据与文件元数据分离**
- 文件数据：通过 HTTP PUT 上传，存储在 `MemoryUploadedFileManager`
- 文件元数据：通过 WebSocket 传输，存储在 Widget state 中
- 关联纽带：`file_id`

**4. 表单提交时的状态一致性**
- `FileUploadClient` 维护 `formsWithPendingRequests` Map
- 跟踪每个 formId 的待处理上传请求数
- 上传进行中时阻止表单提交（防止提交不完整的文件）

---

## 六、代码索引

### 前端核心文件

| 文件 | 核心职责 | 关键函数/类 |
|------|---------|------------|
| [FileUploader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/widgets/FileUploader/FileUploader.tsx) | 文件上传组件 | `dropHandler`, `uploadFile`, `onUploadComplete` |
| [FileUploadClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/FileUploadClient.ts) | 上传客户端 | `fetchFileURLs`, `uploadFile`, `onFileURLsResponse` |
| [uploadFiles.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/util/uploadFiles.ts) | 批量上传工具 | `uploadFiles` |
| [DefaultStreamlitEndpoints.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts) | HTTP 请求实现 | `uploadFileUploaderFile`, `csrfRequest` |
| [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/WidgetStateManager.ts) | Widget 状态管理 | `setFileUploaderStateValue`, `getFileUploaderStateValue` |
| [UploadFileInfo.ts](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/frontend/lib/src/components/shared/UploadedFile/UploadFileInfo.ts) | 文件状态模型 | `UploadFileInfo`, `FileStatus` |

### 后端核心文件

| 文件 | 核心职责 | 关键函数/类 |
|------|---------|------------|
| [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py) | HTTP 上传路由 | `_upload_put`, `create_upload_routes` |
| [memory_uploaded_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py) | 内存文件存储 | `MemoryUploadedFileManager`, `add_file`, `get_files` |
| [uploaded_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/uploaded_file_manager.py) | 存储协议定义 | `UploadedFileManager`, `UploadedFileRec`, `UploadedFile` |
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py) | 会话管理 | `_handle_file_urls_request` |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | Widget 后端逻辑 | `FileUploaderSerde`, `_get_upload_files` |
| [file_uploader_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/lib/file_uploader_utils.py) | 上传工具函数 | `enforce_filename_restriction`, `normalize_upload_file_type` |

---

## 七、设计特点与注意事项

1. **整体上传而非分片上传**：整个文件一次性 PUT 上传，不适用于超大文件场景。最大上传限制由 `server.maxUploadSize` 配置（默认 200MB）。

2. **内存存储而非磁盘存储**：文件全部保存在内存中，重启或会话结束后丢失。适合临时文件使用场景。

3. **双轨通信**：文件数据走 HTTP PUT，控制信令走 WebSocket。两者通过 `file_id` 关联。

4. **延迟状态同步**：上传过程中不触发 Widget 状态同步，避免频繁的脚本重运行。所有文件上传完成后才同步一次。

5. **扩展性设计**：`UploadedFileManager` 是 Protocol 类型，可以替换为其他存储实现（如磁盘存储、S3 等）。
