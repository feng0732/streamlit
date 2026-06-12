# 上传记录缺失的代码事实对齐与恢复机制

## 本文目的

精确对齐代码事实，澄清三个问题：
1. **真正会出现缺失记录的场景**有哪些？
2. **DeletedFile 在序列化和返回值链路里怎样被消化**？
3. **断连和关停分别会不会把这类状态带出来**？

---

## 一、DeletedFile 产生的准确代码路径

### 1.1 产生的唯一点：`_get_upload_files`

DeletedFile 只在 [file_uploader.py L72-L103](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L72-L103) 的 `_get_upload_files` 函数中产生：

```python
def _get_upload_files(
    widget_value: FileUploaderStateProto | None,
) -> list[UploadedFile | DeletedFile]:
    # ... 空值检查 ...

    # 1. 从 widget state 中提取所有 file_id
    uploaded_file_info = widget_value.uploaded_file_info

    # 2. 批量从文件管理器查询实际存在的文件
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
            collected_files.append(UploadedFile(maybe_file_rec, f.file_urls))
        else:
            # ↓↓↓ DeletedFile 唯一产生点 ↓↓↓
            collected_files.append(DeletedFile(f.file_id))

    return collected_files
```

**产生条件**：`file_id` 存在于 **Widget State**，但不存在于 **UploadedFileManager**。

### 1.2 真正会触发的场景

基于代码路径，只有以下场景会真正产生 DeletedFile：

| 场景 | 触发方式 | Widget State | UploadedFileManager | 结果 |
|------|---------|-------------|---------------------|------|
| **服务重启** | 进程重启 | 保留在浏览器 sessionStorage | 清空（内存） | ✅ 产生 DeletedFile |
| **会话关停** | 调用 `shutdown()` | 可能保留在 SessionStorage | 调用 `remove_session_files()` 清空 | ✅ 产生 DeletedFile |
| **主动删除文件** | 前端调用 DELETE 接口 | 下次 sync 时更新 | `remove_file()` 删除 | ❌ 不会产生（同步更新） |
| **断连** | WebSocket 断开 | 保留 | 保留（不删除） | ❌ 不会产生 |
| **SessionStorage 过期** | 默认 TTL 2分钟后 | 可能保留在前端 | 会话数据被清理 | ✅ 可能产生 |
| **并发删除** | 一个请求删除，另一个读取 | 存在 | 已删除 | ✅ 极小概率产生 |

> **重要澄清**：断连不会产生 DeletedFile！因为 `disconnect_session` 只保存会话，不删除文件。只有 `shutdown()` 才会调用 `remove_session_files()`。

---

## 二、DeletedFile 在序列化链路中的消化

DeletedFile 产生后，会经过三层处理，逐步被过滤掉。这是一个"幽灵文件"从产生到消失的完整旅程。

### 2.1 第一层：deserialize（只跳过校验，不删除）

在 [FileUploaderSerde.deserialize](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L111-L131) 中：

```python
def deserialize(self, ui_value: FileUploaderStateProto | None) -> SomeUploadedFiles:
    # 1. 调用 _get_upload_files，可能产生 DeletedFile
    upload_files = _get_upload_files(ui_value)

    # 2. 只跳过 DeletedFile 的类型校验，不删除！
    for file in upload_files:
        if isinstance(file, DeletedFile):
            continue  # ← 跳过，不移除
        if self.allowed_types:
            enforce_filename_restriction(file.name, self.allowed_types)

    # 3. 直接返回，包含 DeletedFile！
    return upload_files if is_multiple_or_directory else upload_files[0]
```

> **关键点**：deserialize 返回值中**仍然包含 DeletedFile**！它只是被跳过了类型校验。

### 2.2 第二层：serialize（主动过滤，不回传前端）

在 [FileUploaderSerde.serialize](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L133-L150) 中：

```python
def serialize(self, files: SomeUploadedFiles) -> FileUploaderStateProto:
    state_proto = FileUploaderStateProto()

    if not files:
        return state_proto
    if not isinstance(files, list):
        files = [files]

    for f in files:
        if isinstance(f, DeletedFile):
            continue  # ← 直接跳过，不写入 proto
        # ... 写入正常文件信息 ...

    return state_proto
```

> **关键点**：序列化时 DeletedFile 被完全过滤，**不会回传给前端**。这意味着前端永远看不到 DeletedFile。

### 2.3 第三层：组件 API 返回值（最终过滤，用户看不到）

在各组件的 Python API 层，返回给用户前会做最后一次过滤：

**FileUploader** ([file_uploader.py L606-L611](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py#L606-L611))：
```python
widget_state = register_widget(...)

if isinstance(widget_state.value, DeletedFile):
    return None
if isinstance(widget_state.value, list):
    return [f for f in widget_state.value if not isinstance(f, DeletedFile)]

return widget_state.value
```

**CameraInput** ([camera_input.py L284-L286](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/camera_input.py#L284-L286))：
```python
if isinstance(camera_input_state.value, DeletedFile):
    return None
return camera_input_state.value
```

**AudioInput** ([audio_input.py L337-L339](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/audio_input.py#L337-L339))：
```python
if isinstance(audio_input_state.value, DeletedFile):
    return None
return audio_input_state.value
```

**ChatInput** ([chat.py L378-L381](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/chat.py#L378-L381))：
```python
uploaded_files = _pop_upload_files(ui_value.file_uploader_state)
for file in uploaded_files:
    if self.allowed_types and not isinstance(file, DeletedFile):
        enforce_filename_restriction(file.name, self.allowed_types)
```

> **关键点**：API 返回值中 DeletedFile 被完全过滤，**用户脚本永远看不到 DeletedFile**。

### 2.4 完整消化链路图

```
产生：_get_upload_files() →  list[UploadedFile | DeletedFile]
          ↓
  deserialize() → 跳过类型校验，保留 DeletedFile
          ↓
  widget_state.value → 可能包含 DeletedFile
          ↓
  组件 API 层 → 过滤 DeletedFile，返回给用户
          ↓
  serialize() → 过滤 DeletedFile，不回传前端
```

DeletedFile 只在系统内部传递，对外部（前端和用户脚本）完全透明。

---

## 三、断连 vs 关停：状态影响对比

### 3.1 断连（disconnect_session）流程

**代码位置**：[websocket_session_manager.py L170-L189](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L170-L189)

```python
def disconnect_session(self, session_id: str) -> None:
    if session_id in self._active_session_info_by_id:
        active_session_info = self._active_session_info_by_id[session_id]
        session = active_session_info.session

        session.request_script_stop()           # 停止脚本
        session.disconnect_file_watchers()      # 断开文件监听器
        session.clear_session_caches()          # 清理缓存

        # ↓↓↓ 保存到 SessionStorage，不删除文件！↓↓↓
        self._session_storage.save(
            SessionInfo(
                client=None,
                session=session,
                script_run_count=active_session_info.script_run_count,
            )
        )
        del self._active_session_info_by_id[session_id]
```

**断连后的状态**：

| 状态 | 处理方式 | 结果 |
|------|---------|------|
| Widget State | 保存在 SessionStorage 中 | ✅ 保留 |
| UploadedFileManager | **不调用 remove_session_files** | ✅ 保留 |
| 会话对象 | 保存到 TTLCache（默认 TTL 2分钟） | ✅ 可恢复 |

**重连后的恢复**：重连时从 SessionStorage 取出会话，文件和状态都在，**不会产生 DeletedFile**。

### 3.2 关停（shutdown）流程

**代码位置**：[app_session.py L290-L316](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py#L290-L316)

```python
def shutdown(self) -> None:
    if self._state != AppSessionState.SHUTDOWN_REQUESTED:
        # ↓↓↓ 第一步就删除所有上传文件！↓↓↓
        self._uploaded_file_mgr.remove_session_files(self.id)

        if runtime.exists():
            rt = runtime.get_instance()
            rt.media_file_mgr.clear_session_refs(self.id)
            rt.media_file_mgr.remove_orphaned_files()

        self.request_script_stop()
        self._state = AppSessionState.SHUTDOWN_REQUESTED
        self.disconnect_file_watchers()
        self.clear_session_caches()
```

**关停后的状态**：

| 状态 | 处理方式 | 结果 |
|------|---------|------|
| Widget State | 可能还在 SessionStorage 中（取决于 TTL） | ⚠️ 可能保留 |
| UploadedFileManager | 调用 `remove_session_files()` | ❌ 删除 |
| 会话对象 | 调用 `close_session()` 后删除 | ❌ 删除 |

**关停后重连**：如果 SessionStorage 还没过期（2分钟内），Widget State 中还有 file_id，但文件已被删除，**会产生 DeletedFile**。

### 3.3 关键差异对比表

| 操作 | 文件数据 | Widget State | 是否产生 DeletedFile |
|------|---------|-------------|----------------------|
| **断连** | 保留（内存） | 保存（SessionStorage） | ❌ 不会 |
| **关停** | 删除（remove_session_files） | 可能残留（取决于 TTL） | ✅ 会（如果重连） |
| **服务重启** | 清空（进程重启） | 保留（浏览器） | ✅ 会 |
| **主动删除文件** | 删除 | 同步更新 | ❌ 不会 |

### 3.4 SessionStorage 的 TTL 机制

**代码位置**：[memory_session_storage.py L41-L77](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_session_storage.py#L41-L77)

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
```

**TTL 触发的场景**：
- 断连后 2 分钟内没有重连 → SessionStorage 中的会话被清理
- 如果 2 分钟后用户重新连接 → 新建会话，但前端可能还保留着旧的 file_id
- 此时如果用户刷新页面，前端的 widget state 与后端的空存储比对 → **产生 DeletedFile**

---

## 四、DeletedFile 在 session_state 中的生命周期

### 4.1 Widget 值的存储形式

在 SessionState 中，Widget 值有两种存储形式：

| 形式 | 类型 | 说明 |
|------|------|------|
| `Serialized` | 序列化的 protobuf | 从前端收到，或初始状态 |
| `Value` | 反序列化后的 Python 对象 | 访问后缓存 |

**代码位置**：[session_state.py L151-L203](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py#L151-L203)

```python
def __getitem__(self, k: str) -> Any:
    wstate = self.states.get(k)
    if wstate is None:
        raise KeyError(k)

    if isinstance(wstate, Value):
        return wstate.value

    # 反序列化
    metadata = self.widget_metadata.get(k)
    # ... 提取 proto 值 ...
    deserialized = metadata.deserializer(value)  # ← 这里产生 DeletedFile
    self.states[k] = Value(deserialized)  # ← 缓存为 Value 形式
    return deserialized
```

### 4.2 DeletedFile 如何被缓存

当第一次访问 file_uploader 的值时：
1. 调用 `deserializer` → `_get_upload_files` → 可能产生 DeletedFile
2. 结果被缓存为 `Value` 形式存储在 `_new_widget_state` 中
3. **DeletedFile 就驻留在 session_state 中了**

### 4.3 什么时候被清理

**脚本运行结束时**，`on_script_finished` → `_remove_stale_widgets` 会清理不活跃的 widget：

**代码位置**：[session_state.py L906-L953](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py#L906-L953)

```python
def _remove_stale_widgets(self, active_widget_ids: frozenset[str]) -> None:
    # 清理 _new_widget_state 中的不活跃 widget
    self._new_widget_state.remove_stale_widgets(
        active_widget_ids,
        ctx.fragment_ids_this_run,
    )

    # 清理 _old_state 中的不活跃 widget
    self._old_state = {
        k: v
        for k, v in self._old_state.items()
        if not _is_stale_widget(...)
    }
```

**清理条件**：widget 不在本次脚本运行的 `active_widget_ids` 中。

> **注意**：只要用户的脚本还在调用 `st.file_uploader(...)`，这个 widget 就会被标记为 active，其状态（包括 DeletedFile）就会一直保留。只有当脚本中删除了这个 widget，它的状态才会在下一次脚本运行结束时被清理。

---

## 五、用户能感知到的表现

由于 DeletedFile 在 API 层和序列化层都被过滤了，用户**永远不会直接看到 DeletedFile**。但可以通过以下表现间接感知：

| 表现 | 场景 | 说明 |
|------|------|------|
| **文件突然消失** | 服务重启后刷新页面 | 上传的文件列表变空了，没有报错 |
| **session_state 中值为 None** | 单文件模式下文件缺失 | `st.session_state["my_uploader"]` 返回 None |
| **session_state 中列表变短** | 多文件模式下部分文件缺失 | 列表长度比预期短，缺失的被过滤了 |
| **前端显示正常** | 任何场景 | serialize 时已过滤，前端看不到缺失 |

---

## 六、代码索引

### DeletedFile 相关

| 文件 | 代码位置 | 说明 |
|------|---------|------|
| [uploaded_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/uploaded_file_manager.py) | L47-L58 | DeletedFile 定义 |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | L72-L103 | `_get_upload_files` 产生 DeletedFile |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | L114-L116 | deserialize 跳过类型校验 |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | L142-L143 | serialize 过滤 |
| [file_uploader.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/elements/widgets/file_uploader.py) | L606-L611 | API 层最终过滤 |

### 会话管理相关

| 文件 | 代码位置 | 说明 |
|------|---------|------|
| [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py) | L170-L189 | `disconnect_session` 断连处理 |
| [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/websocket_session_manager.py) | L204-L213 | `close_session` 关停处理 |
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/app_session.py) | L290-L316 | `shutdown` 关停清理 |
| [memory_session_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/memory_session_storage.py) | L41-L77 | TTL 缓存实现 |
| [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py) | L151-L203 | Widget 值反序列化缓存 |
| [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/221-streamlit/lib/streamlit/runtime/state/session_state.py) | L906-L953 | 不活跃 widget 清理 |

---

## 七、关键结论速查表

| 问题 | 答案 |
|------|------|
| **断连会产生 DeletedFile 吗？** | ❌ 不会。文件保留在内存中。 |
| **关停会产生 DeletedFile 吗？** | ✅ 会。如果 SessionStorage 还没过期，重连后 Widget state 中的 file_id 找不到对应文件。 |
| **用户能拿到 DeletedFile 吗？** | ❌ 不能。API 层已过滤。 |
| **前端能看到 DeletedFile 吗？** | ❌ 不能。serialize 时已过滤。 |
| **DeletedFile 会在 session_state 中吗？** | ✅ 会。以 `Value([..., DeletedFile(id), ...])` 形式缓存。 |
| **DeletedFile 什么时候消失？** | 脚本中移除 widget 后，下一次 `_remove_stale_widgets` 清理。 |
| **主动删除文件会产生 DeletedFile 吗？** | ❌ 不会。前端同步更新 Widget state。 |
| **服务重启会产生 DeletedFile 吗？** | ✅ 会。内存清空，浏览器还保留 file_id。 |
