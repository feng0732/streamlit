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
| `st.image()` | `ImageMixin.image()` | [image.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/elements/image.py#L48-L242) |
| `st.audio()` | `MediaMixin.audio()` | [media.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/elements/media.py#L70-L226) |
| `st.video()` | `MediaMixin.video()` | [media.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/elements/media.py#L227-L408) |

### 2.2 图片格式归一化：`image_to_url()`

核心逻辑在 [image_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/elements/lib/image_utils.py#L235-L346)。

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
  [media.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/elements/media.py#L742-L762)
- **视频 YouTube URL** → `_reshape_youtube_url()` 正则提取 video_id，重写为 embed 链接
  [media.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/elements/media.py#L415-L446)
- **字幕文件** → `process_subtitle_data()` 同样注册到 MediaFileManager

### 2.4 MediaFileManager：注册与生成 URL

核心类在 [media_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/runtime/media_file_manager.py#L82-L409)。

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

[memory_media_file_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/runtime/memory_media_file_storage.py)

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

## 三、传输：Protobuf + WebSocket

### 3.1 Protobuf 定义

**Image**：[Image.proto](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/proto/streamlit/proto/Image.proto)
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

**Audio**：[Audio.proto](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/proto/streamlit/proto/Audio.proto)
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

**Video**：[Video.proto](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/proto/streamlit/proto/Video.proto)
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

### 3.2 序列化与入队

以 image 为例，[image.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/elements/image.py#L216-L237)：
```python
image_list_proto = ImageListProto()
marshall_images(..., image_list_proto, ...)   # 填充 proto 中的 url / caption
return self.dg._enqueue("imgs", image_list_proto, layout_config=layout_config)
```

`_enqueue` 将 proto 序列化后通过 WebSocket 的 `ForwardMsg` 推送到前端。

---

## 四、服务端：Starlette HTTP 媒体路由

### 4.1 路由注册

[starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L488-L590)

```python
BASE_ROUTE_MEDIA = "media"
_ROUTE_MEDIA = f"{BASE_ROUTE_MEDIA}/{{file_id:path}}"
# 注册 GET / HEAD / OPTIONS 三种方法
```

### 4.2 `_media_endpoint` 请求处理

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

## 五、前端：React 组件渲染

### 5.1 URL 补全：DefaultStreamlitEndpoints

[DefaultStreamlitEndpoints.ts](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts#L187-L204)

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

### 5.2 组件渲染

#### 图片组件 [ImageList.tsx](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/frontend/lib/src/components/elements/ImageList/ImageList.tsx)

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

#### 音频组件 [Audio.tsx](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/frontend/lib/src/components/elements/Audio/Audio.tsx)

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

#### 视频组件 [Video.tsx](file:///d:/fz/0601/solo-dogfeeding/code/229-streamlit/frontend/lib/src/components/elements/Video/Video.tsx)

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

## 六、衔接点总结

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

## 七、关键设计决策

1. **内容寻址去重**：file_id = hash(content + mimetype + filename)，相同内容的媒体文件在服务端只存一份，跨 session 共享。
2. **二进制与控制信令分离**：protobuf 只传 URL 和元数据，大体积二进制走独立 HTTP 通道，避免 WebSocket 消息过大。
3. **Session 引用计数清理**：`_files_by_session_and_coord` 追踪每个 session 在用哪些 file；脚本重跑或 session 断开后，`remove_orphaned_files()` 回收孤儿文件。
4. **部分哈希优化**：>1 MiB 文件只取头尾中段各 64 KiB 计算指纹，权衡碰撞概率与性能。
5. **HTTP Range 支持**：视频无需完整下载即可拖动进度条播放，响应 `206 Partial Content`。
6. **SVG 内联为 data URI**：避免额外 HTTP 请求，并解决 SVG xmlns 缺失导致浏览器不渲染的问题。
