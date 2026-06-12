# Streamlit 媒体资源处理流程：图片加载 → 资源编码 → 浏览器展示

本文档梳理 Streamlit 中媒体资源（图片、音频、视频）从用户调用 API 到浏览器最终渲染的完整链路。

---

## 一、总览流程图

```
用户调用 API
    │
    ▼
┌──────────────────────────────────────────────────────┐
│  Python 后端 - 数据采集与格式归一化                     │
│  • st.image() / st.audio() / st.video()               │
│  • 支持: URL / 文件路径 / PIL / BytesIO / numpy / SVG  │
│  • 统一转换为 bytes 或直接透传 URL                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  Python 后端 - MediaFileManager 注册                   │
│  • 计算内容 hash 生成 file_id                           │
│  • 存储到 MemoryMediaFileStorage (内存字典)             │
│  • 返回相对 URL: /media/{file_id}.{ext}                │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  Protobuf 序列化 + WebSocket 传输                       │
│  • ImageList / Audio / Video proto 消息                 │
│  • 仅携带 URL + 元数据 (caption/start_time/loop 等)     │
│  • 不携带二进制内容本身                                  │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  前端 React 组件                                       │
│  • DefaultStreamlitEndpoints.buildMediaURL()           │
│  • 将 /media/... 相对路径补全为完整 HTTP URL            │
│  • <img>/<audio>/<video>/<iframe> 发起 GET 请求         │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  Starlette HTTP 服务 - /media/{file_id}                │
│  • 根据 file_id 从 MemoryMediaFileStorage 取出 bytes    │
│  • 设置 Content-Type / Accept-Ranges / CORS            │
│  • 支持 HTTP Range 请求 (视频流媒体)                     │
└──────────────────────────────────────────────────────┘
```

---

## 二、后端：Python 侧数据采集与编码

### 2.1 API 入口

| 用户 API | 实现类/方法 | 文件 |
|---|---|---|
| `st.image()` | `ImageMixin.image()` | [image.py](lib/streamlit/elements/image.py#L48-L242) |
| `st.audio()` | `MediaMixin.audio()` | [media.py](lib/streamlit/elements/media.py#L70-L226) |
| `st.video()` | `MediaMixin.video()` | [media.py](lib/streamlit/elements/media.py#L227-L408) |

### 2.2 图片格式归一化：`image_to_url()`

核心逻辑在 [image_utils.py](lib/streamlit/elements/lib/image_utils.py#L235-L346)。

**分支处理顺序：**

1. **URL / 相对静态 URL** — 直接返回原字符串
   - 判断条件：`url_util.is_url()` 或 `url_util.is_relative_static_url()`
   - 协议白名单：`http`, `https`, `data`

2. **本地 SVG 文件 / SVG 字符串** — 转 base64 data URI
   ```python
   image_b64_encoded = base64.b64encode(image.encode("utf-8")).decode("utf-8")
   return f"data:image/svg+xml;base64,{image_b64_encoded}"
   ```

3. **本地文件路径** — 读取 bytes

4. **PIL Image** → `_pil_to_bytes()` 按 JPEG/PNG/GIF 保存到 BytesIO

5. **BytesIO** → `getvalue()` 取 bytes

6. **Numpy ndarray**：
   - `_clip_image()`: clamp 像素值到 [0,255] 或 [0.0, 1.0]
   - `channels="BGR"` 时翻转通道顺序
   - `_np_array_to_bytes()`: PIL 转 JPEG/PNG bytes

7. **原始 bytes** — 直接使用

**图片尺寸与格式统一：** `_ensure_image_size_and_format()`
- 若图片宽度超过目标宽度（`MAXIMUM_CONTENT_WIDTH = 2 * 730 = 1460px` 或用户指定 width），等比缩放到目标宽度，quality=90
- 若实际格式与目标格式不一致，重新编码

### 2.3 音视频格式归一化

- **音频 Numpy 数组** → `_make_wav()` 封装为 PCM WAV bytes
  [media.py](lib/streamlit/elements/media.py#L742-L762)
- **视频 YouTube URL** → `_reshape_youtube_url()` 正则提取 video_id，重写为 embed 链接
  [media.py](lib/streamlit/elements/media.py#L415-L446)
- **字幕文件** → `process_subtitle_data()` 同样注册到 MediaFileManager

### 2.4 MediaFileManager：注册与生成 URL

核心类在 [media_file_manager.py](lib/streamlit/runtime/media_file_manager.py#L82-L409)。

**关键数据结构：**
```python
self._files_by_id: dict[str, MediaFileMetadata]
self._files_by_session_and_coord: dict[session_id, dict[coordinates, file_id]]
```

**`add(path_or_data, mimetype, coordinates)` 流程：**
1. 通过 `_get_session_id()` 获取当前 AppSession 的 session_id
2. 调用 `storage.load_and_get_id()` 存储并拿到 file_id
3. 建立两层索引：`session_id → coordinates → file_id`
4. 返回 `storage.get_url(file_id)`

**coordinates 的作用**：格式如 `"1.(3.-14).5-0"`，是元素在 Delta 树中的路径 + 在列表中的索引。用来在同一位置元素被替换时，旧的 file_id 可以被判定为 orphan 并清理。

### 2.5 MemoryMediaFileStorage：内存存储实现

[memory_media_file_storage.py](lib/streamlit/runtime/memory_media_file_storage.py)

**`load_and_get_id()`：**
- 若是文件路径字符串，先读成 bytes
- 调用 `_calculate_file_id()` 生成稳定 ID：
  ```
  hash( content_length + content + mimetype + filename )
  ```
- 大文件 (>1 MiB) 使用**部分哈希**优化：只取 head(64KiB) + middle(64KiB) + tail(64KiB)，提速 50-100x
- 存到 `self._files_by_id[file_id] = MemoryFile(content, mimetype, kind, filename)`
- 因为 file_id 是内容哈希，**相同内容不会重复存储**

**`get_url(file_id)`：**
```python
extension = get_extension_for_mimetype(media_file.mimetype)
return f"{self._media_endpoint}/{file_id}{extension}"
# 例如: /media/abc123def456.jpg
```

---

## 三、MediaFileManager 与 MediaFileStorage 职责边界深度解析

### 3.1 接口抽象层：MediaFileStorage 协议

[media_file_storage.py](lib/streamlit/runtime/media_file_storage.py#L42-L143) 定义了纯抽象接口（Protocol），定义存储层必须实现的三个核心能力：

```python
class MediaFileStorage(Protocol):
    def load_and_get_id(self, path_or_data, mimetype, kind, filename) -> str:
        # 输入：原始数据或文件路径 + 元数据
        # 输出：全局唯一的 file_id
        # 职责：读取/存储内容，计算内容哈希，去重

    def get_url(self, file_id: str) -> str:
        # 输入：file_id
        # 输出：相对 URL 路径（如 /media/abc123.jpg）
        # 职责：根据 mimetype 拼接扩展名，构造可被路由识别的 URL

    def delete_file(self, file_id: str) -> None:
        # 输入：file_id
        # 输出：无
        # 职责：物理删除内容，释放资源
```

> **设计意图**：通过 Protocol 抽象，存储层可以有多种实现（内存、S3、本地磁盘等），Manager 层不感知具体存储介质。

### 3.2 维护的状态对比

| 状态 | MediaFileManager | MediaFileStorage (Memory) |
|---|---|---|
| **二进制内容** | ❌ 不持有 | ✅ `_files_by_id: dict[str, MemoryFile]` |
| **内容元数据** | ✅ `_file_metadata: dict[str, MediaFileMetadata]` (kind, is_marked_for_delete) | ✅ 内嵌于 `MemoryFile` (mimetype, kind, filename) |
| **Session 引用关系** | ✅ `_files_by_session_and_coord: dict[session_id, dict[coord, file_id]]` | ❌ 不感知 |
| **延迟执行逻辑** | ✅ `_deferred_callables: dict[str, DeferredCallableEntry]` | ❌ 不感知 |
| **线程安全** | ✅ `_lock: threading.Lock` | ❌ 不做同步（由 Manager 保证） |
| **URL 前缀配置** | ❌ 不持有 | ✅ `_media_endpoint: str` (如 "/media") |
| **统计信息** | ❌ 不提供 | ✅ 实现 `StatsProvider` 接口，提供内存使用统计 |

**核心结论**：
- **Storage 层** 维护「内容本身」：bytes 存哪里、怎么存、怎么取、怎么算 ID
- **Manager 层** 维护「生命周期」：谁在用、用在哪、什么时候删、并发安全

### 3.3 媒体 URL 生成流程（写路径）

调用链：`image_to_url()` → `MediaFileManager.add()` → `Storage.load_and_get_id()` → `Storage.get_url()`

```
用户数据 (bytes/str/PIL/numpy)
        │
        ▼
image_to_url() — 格式归一化为 bytes + mimetype
        │
        ▼
MediaFileManager.add(data, mimetype, coordinates)
  1. _get_session_id() → "session_abc123"
  2. self._lock.acquire()
  3. storage.load_and_get_id(data, mimetype, kind, filename)
     ├─ 若 data 是文件路径 → _read_file() 读为 bytes
     ├─ _calculate_file_id(bytes, mimetype, filename) → "file_xyz789"
     └─ 若 ID 不存在 → _files_by_id["file_xyz789"] = MemoryFile(bytes, ...)
  4. _file_metadata["file_xyz789"] = MediaFileMetadata(kind=MEDIA)
  5. _files_by_session_and_coord["session_abc123"][coordinates] = "file_xyz789"
  6. storage.get_url("file_xyz789")
     ├─ get_file("file_xyz789") → MemoryFile
     ├─ get_extension_for_mimetype("image/jpeg") → ".jpg"
     └─ return "/media/file_xyz789.jpg"
  7. self._lock.release()
        │
        ▼
返回: "/media/file_xyz789.jpg"
```

**关键分层点**：
- Manager 不知道 file_id 怎么算的，也不知道 URL 怎么拼的
- Storage 不知道 session_id 是什么，也不知道 coordinates 有什么用
- 两者通过 `file_id` 这个唯一标识符解耦

### 3.4 媒体 URL 读取流程（读路径 / HTTP 层）

调用链：`GET /media/file_xyz789.jpg` → `Starlette _media_endpoint` → `Storage.get_file()`

```
HTTP Request: GET /media/file_xyz789.jpg
        │
        ▼
Starlette 路由匹配 {file_id:path} = "file_xyz789.jpg"
        │
        ▼
media_storage.get_file("file_xyz789.jpg")
  1. os.path.splitext("file_xyz789.jpg")[0] → "file_xyz789"
  2. _files_by_id["file_xyz789"] → MemoryFile(content=bytes, mimetype="image/jpeg", ...)
        │
        ▼
返回 Response:
  body = bytes
  Content-Type = "image/jpeg"
  Accept-Ranges = "bytes"
```

**关键设计**：
- URL 中的扩展名 `.jpg` 是**给浏览器看的**，Storage 实际通过去掉扩展名的部分 `file_xyz789` 来索引
- HTTP 层**绕过 Manager 直接访问 Storage**，因为读取路径不需要生命周期管理
- 这也解释了为什么 Storage 必须独立存在：它同时被 Manager（写路径）和 Starlette 路由（读路径）调用

### 3.5 引用计数与垃圾回收

`remove_orphaned_files()` 是 Manager 层的核心职责，完全基于自己维护的状态判断，Storage 不参与决策：

```python
def _get_inactive_file_ids(self) -> set[str]:
    # 所有已知的 file_id
    file_ids = set(self._file_metadata.keys())
    # 减去所有 session 正在引用的 file_id
    for session_file_ids_by_coord in self._files_by_session_and_coord.values():
        file_ids.difference_update(session_file_ids_by_coord.values())
    return file_ids  # 剩下的就是孤儿文件

def remove_orphaned_files(self) -> None:
    with self._lock:
        for file_id in self._get_inactive_file_ids():
            file = self._file_metadata[file_id]
            if file.kind == MEDIA or file.is_marked_for_delete:
                self._storage.delete_file(file_id)  # 通知 Storage 物理删除
                del self._file_metadata[file_id]    # 清理自己的元数据
```

### 3.6 边界模糊点澄清

**Q: 为什么 `MediaFileMetadata` 由 Manager 维护，而不是存到 Storage 里？**

A: `MediaFileMetadata` 的 `is_marked_for_delete` 字段是生命周期状态（"这个文件被标记了，下次清理时再删"），不是内容属性。Storage 只关心"内容是什么"，不关心"用户还想不想留着它"。

**Q: 为什么 `MediaFileManager.add()` 调用 `storage.get_url()` 而不是自己拼 URL？**

A: URL 格式是存储层的内部约定。如果未来换成 S3 存储，URL 可能变成 `https://bucket.s3.amazonaws.com/file_xyz789.jpg`，Manager 层代码不需要改动。

**Q: 为什么读路径不经过 Manager？**

A: 读路径是高频的浏览器 HTTP 请求，不需要生命周期追踪、不需要线程锁、不需要 session 关联。直接访问 Storage 性能更好，职责也更清晰。

---

## 四、传输：Protobuf + WebSocket

### 4.1 Protobuf 定义

**Image**：[Image.proto](proto/streamlit/proto/Image.proto)
```protobuf
message Image {
  string url = 3;       // /media/... 或外部 URL 或 data: URI
  string caption = 2;
}
message ImageList {
  repeated Image imgs = 1;
  string link = 3;      // 单图时的点击跳转链接
}
```

**Audio**：[Audio.proto](proto/streamlit/proto/Audio.proto)
```protobuf
message Audio {
  string url = 5;
  int32 start_time = 3;
  int32 end_time = 6;
  bool loop = 7;
  bool autoplay = 8;
  string id = 9;
  optional streamlit.WidthConfig width_config = 10;
}
```

**Video**：[Video.proto](proto/streamlit/proto/Video.proto)
```protobuf
message SubtitleTrack {
  string label = 1;
  string url = 2;
}
message Video {
  string url = 6;
  enum Type { NATIVE = 1; YOUTUBE_IFRAME = 2; }
  Type type = 5;
  repeated SubtitleTrack subtitles = 7;
  int32 start_time = 3;
  int32 end_time = 8;
  bool loop = 9;
  bool autoplay = 10;
  bool muted = 11;
  // ...
}
```

> **关键点**：protobuf 消息中只携带 `url`（相对路径或绝对地址），**从不携带二进制内容**。真正的 bytes 通过 HTTP `/media/...` 单独获取。

### 4.2 序列化与入队

以 image 为例，[image.py](lib/streamlit/elements/image.py#L216-L237)：
```python
image_list_proto = ImageListProto()
marshall_images(..., image_list_proto, ...)   # 填充 proto 中的 url / caption
return self.dg._enqueue("imgs", image_list_proto, layout_config=layout_config)
```

`_enqueue` 将 proto 序列化后通过 WebSocket 的 `ForwardMsg` 推送到前端。

---

## 五、服务端：Starlette HTTP 媒体路由

### 5.1 路由注册

[starlette_routes.py](lib/streamlit/web/server/starlette/starlette_routes.py#L488-L590)

```python
BASE_ROUTE_MEDIA = "media"
_ROUTE_MEDIA = f"{BASE_ROUTE_MEDIA}/{{file_id:path}}"
# 注册 GET / HEAD / OPTIONS 三种方法
```

### 5.2 `_media_endpoint` 请求处理

```
Request: GET /media/abc123def.jpg
         Range: bytes=0-99999
```

1. **从存储取文件**：`media_storage.get_file(file_id)`
   - file_id 是去掉扩展名后的部分（`os.path.splitext(filename)[0]`）

2. **设置响应头**：
   - `DOWNLOADABLE` 种类：加 `Content-Disposition: attachment; filename="..."`
   - `Accept-Ranges: bytes`（支持视频分段加载）
   - CORS 头

3. **Range 请求处理**（视频流式播放关键）：
   - 解析 `Range: bytes=start-end`
   - 返回 206 Partial Content
   - 设置 `Content-Range` / `Content-Length`

4. **返回**：`Response(content, media_type=mimetype, ...)`

---

## 六、前端：React 组件渲染

### 6.1 URL 补全：DefaultStreamlitEndpoints

[DefaultStreamlitEndpoints.ts](frontend/connection/src/DefaultStreamlitEndpoints.ts#L187-L204)

```typescript
public buildMediaURL(url: string): string {
  // 静态部署模式：从 S3 等静态资源地址获取
  if (this.staticConfigUrl) {
    if (url.startsWith(MEDIA_ENDPOINT) || url.startsWith(STATIC_SERVING_ENDPOINT)) {
      return this.buildStaticUrl(url)
    }
  }
  // 动态模式：拼接后端服务器地址
  if (url.startsWith(MEDIA_ENDPOINT) || url.startsWith(STATIC_SERVING_ENDPOINT)) {
    return buildHttpUri(this.requireServerUri(), url)
  }
  // 已是完整 URL（http/https/data），原样返回
  return url
}
```

例如：`/media/abc123.jpg` → `http://localhost:8501/media/abc123.jpg`

### 6.2 组件渲染

#### 图片组件 [ImageList.tsx](frontend/lib/src/components/elements/ImageList/ImageList.tsx)

```tsx
<img
  style={imgStyle}
  src={buildMediaURL(image.url)}   // 核心：触发浏览器 GET /media/...
  alt={itemKey}
  onError={handleImageError}
  crossOrigin={crossOrigin}
/>
```
- caption 通过 `StreamlitMarkdown` 渲染在图片下方
- 单图可包裹 `<StyledImageLink>` 实现点击跳转

#### 音频组件 [Audio.tsx](frontend/lib/src/components/elements/Audio/Audio.tsx)

```tsx
const uri = endpoints.buildMediaURL(element.url)
return <StyledAudio
  ref={audioRef}
  controls
  autoPlay={autoplay && !preventAutoplay}
  src={uri}
  crossOrigin={crossOrigin}
/>
```
- `useEffect` 监听 `loadedmetadata` 后设置 `currentTime = startTime`
- `timeupdate` 事件中实现 `end_time` 截断和 `loop` 逻辑（因为原生 HTML5 audio 不支持 end_time）

#### 视频组件 [Video.tsx](frontend/lib/src/components/elements/Video/Video.tsx)

两种分支：

1. **YouTube**（`type === YOUTUBE_IFRAME`）：
   ```tsx
   <StyledVideoIframe src={getYoutubeSrc(url)} allowFullScreen />
   ```
   - 将 start/end/loop/autoplay/muted 转为 YouTube embed URL 的 query 参数

2. **原生视频**（`type === NATIVE`）：
   ```tsx
   <StyledVideo
     ref={videoRef}
     controls
     muted={muted}
     autoPlay={autoplay && !preventAutoplay}
     src={endpoints.buildMediaURL(url)}
     crossOrigin={crossOrigin}
   >
     {subtitles?.map(subtitle => (
       <track kind="captions"
         src={endpoints.buildMediaURL(`${subtitle.url}`)}
         label={`${subtitle.label}`}
         default={idx === 0}
       />
     ))}
   </StyledVideo>
   ```
   - 字幕的 URL 同样走 `buildMediaURL()`，因为字幕文件也注册在 MediaFileManager 中
   - 因为 `<track>` 没有 onerror 事件，字幕加载失败通过 `fetch()` 主动探测

---

## 七、衔接点总结

| 衔接环节 | 位置 | 关键数据 |
|---|---|---|
| **后端 bytes → URL** | `MediaFileManager.add()` → `MemoryMediaFileStorage.get_url()` | bytes + mimetype → `/media/{hash}.{ext}` |
| **URL → Protobuf** | `marshall_images()` / `marshall_audio()` / `marshall_video()` | `proto.url = file_url` |
| **Protobuf → WebSocket** | `DeltaGenerator._enqueue()` | ForwardMsg binary frame |
| **WebSocket → 前端组件** | App render tree 根据 delta type 路由到 ImageList/Audio/Video | props.element 是反序列化后的 proto 对象 |
| **相对 URL → 完整 URL** | `DefaultStreamlitEndpoints.buildMediaURL()` | `/media/...` → `http://host:port/media/...` |
| **HTTP GET → bytes 返回** | Starlette `_media_endpoint` | 从 `_files_by_id[hash]` 取出 bytes 流式返回 |
| **bytes → 浏览器渲染** | 原生 `<img>` / `<audio>` / `<video>` 标签 | 由浏览器根据 Content-Type 解码展示 |

---

## 八、关键设计决策

1. **内容寻址去重**：file_id = hash(content + mimetype + filename)，相同内容的媒体文件在服务端只存一份，跨 session 共享。
2. **二进制与控制信令分离**：protobuf 只传 URL 和元数据，大体积二进制走独立 HTTP 通道，避免 WebSocket 消息过大。
3. **Session 引用计数清理**：`_files_by_session_and_coord` 追踪每个 session 在用哪些 file；脚本重跑或 session 断开后，`remove_orphaned_files()` 回收孤儿文件。
4. **部分哈希优化**：>1 MiB 文件只取头尾中段各 64 KiB 计算指纹，权衡碰撞概率与性能。
5. **HTTP Range 支持**：视频无需完整下载即可拖动进度条播放，响应 `206 Partial Content`。
6. **SVG 内联为 data URI**：避免额外 HTTP 请求，并解决 SVG xmlns 缺失导致浏览器不渲染的问题。
7. **Manager 与 Storage 分层**：通过 Protocol 抽象解耦，Storage 管"内容"（存哪里、怎么取），Manager 管"生命周期"（谁在用、何时删），两者通过 file_id 唯一标识符协作。
