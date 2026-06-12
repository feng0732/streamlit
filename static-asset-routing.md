# Streamlit 静态资源服务链路深度分析

本文档深入分析 Streamlit 静态资源服务链路的**代码实现细节**，重点讲解：
1. **资源分流数量** — 有多少类静态资源，各路由如何创建和分流
2. **核心静态资源的独立挂载** — 为何使用 `Mount` 独立挂载，具体如何实现
3. **上传路由跳过全量路径校验的原因** — 为何 `/_stcore/upload_file/*` 可以安全绕过 `is_unsafe_path_pattern()`
4. **静态部署时外部静态源改写** — Static Connection 模式下如何将资源 URL 改写为外部静态源（如 S3）

---

## 1. 资源分流数量：6 类静态资源路由的代码编排

Streamlit 的静态资源在代码层面被拆分为 **6 类独立路由**，由 `create_streamlit_routes()` 函数统一编排，通过多次调用不同的 `create_*_routes()` 工厂函数将各类路由添加到同一个路由列表中。

### 1.1 路由编排代码

在 [create_streamlit_routes()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L101-L161) 中可以清晰看到分流过程：

```python
def create_streamlit_routes(runtime: Runtime) -> list[BaseRoute]:
    routes: list[Any] = []

    # 1. 健康检查
    routes.extend(create_health_routes(runtime, base_url))
    # 2. 指标
    routes.extend(create_metrics_routes(runtime, base_url))
    # 3. 主机配置
    routes.extend(create_host_config_routes(base_url))
    # 4. 媒体文件（st.image/st.audio/st.video/st.download_button）
    routes.extend(create_media_routes(media_storage, base_url))
    # 5. 文件上传
    routes.extend(create_upload_routes(runtime, upload_mgr, base_url))
    # 6. 自定义组件 v1
    routes.extend(create_component_routes(component_registry, base_url))
    # 7. 自定义组件 v2（双向组件）
    routes.extend(create_bidi_component_routes(bidi_component_manager, base_url))
    # 8. WebSocket
    routes.extend(create_websocket_routes(runtime, base_url))
    # 9. 认证路由
    routes.extend(create_auth_routes(base_url))
    # 10. App 用户静态文件（需启用 server.enableStaticServing）
    if config.get_option("server.enableStaticServing"):
        routes.extend(create_app_static_serving_routes(main_script_path, base_url))
    # 11. 脚本健康检查（可选）
    if config.get_option("server.scriptHealthCheckEnabled"):
        routes.extend(create_script_health_routes(runtime, base_url))
    # 12. 核心前端资产（仅生产模式）
    if not dev_mode:
        routes.extend(create_streamlit_static_assets_routes(base_url=base_url))

    return routes
```

其中**静态资源相关**的有 **6 类**：

| 序号 | 资源类型 | 创建函数 | 路径前缀 | 处理的资源 |
|---|---|---|---|---|
| 1 | 核心前端资产 | `create_streamlit_static_assets_routes()` | `/` 或 `/{base_url}` | JS/CSS/HTML/字体等前端构建产物 |
| 2 | 媒体文件 | `create_media_routes()` | `/media/{file_id}` | st.image / st.audio / st.video / st.download_button |
| 3 | 自定义组件 v1 | `create_component_routes()` | `/component/{name}/{path}` | 第三方自定义组件的 HTML/JS/CSS |
| 4 | 自定义组件 v2 | `create_bidi_component_routes()` | `/_stcore/bidi-components/{name}/{path}` | 新一代双向组件资源 |
| 5 | App 用户静态文件 | `create_app_static_serving_routes()` | `/app/static/{path}` | 应用目录 `static/` 下的用户文件 |
| 6 | 文件上传 | `create_upload_routes()` | `/_stcore/upload_file/{session_id}/{file_id}` | st.file_uploader 上传的文件（PUT/DELETE） |

### 1.2 为什么要拆成多个工厂函数

这种"多函数分流"的设计有三个好处：
1. **条件启用**：App 静态文件需启用配置、核心资产仅生产模式、脚本健康检查可选 — 按需组装
2. **依赖注入**：每个工厂函数接收不同的运行时依赖（`media_storage`、`upload_mgr`、`component_registry`），职责单一
3. **独立测试**：每个路由类型可单独测试，不依赖其他路由

---

## 2. 核心静态资源的独立挂载：Mount 的设计与实现

### 2.1 为何要独立挂载

核心前端资产（JS/CSS/HTML）与其他路由有本质区别：
- **路径根占用**：需要挂载在 `/`（或 `/{base_url}`）根路径，而非特定前缀
- **SPA 回退**：404 时需返回 `index.html` 支持客户端路由
- **独立的缓存策略**：需根据文件名区分 `no-cache`（HTML）和 `immutable + max-age=1年`（带哈希的 JS/CSS）
- **独立的安全检查**：双斜杠保护、尾部斜杠重定向等

因此 Streamlit 没有将其作为普通 `Route` 添加，而是使用 Starlette 的 `Mount` 组件**独立挂载**了一个完整的 ASGI 应用。

### 2.2 挂载实现代码

在 [create_streamlit_static_assets_routes()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_static_routes.py#L169-L191) 中：

```python
def create_streamlit_static_assets_routes(base_url: str | None) -> list[BaseRoute]:
    """Create the static assets mount for serving Streamlit's core assets."""
    from starlette.routing import Mount

    static_dir = file_util.get_static_dir()
    if not os.path.isdir(static_dir):
        return []

    # 创建自定义 StaticFiles 处理器（继承 Starlette StaticFiles）
    static_assets = create_streamlit_static_handler(
        directory=static_dir, base_url=base_url
    )

    # 构建挂载路径（去掉尾部斜杠）
    mount_path = make_url_path(base_url or "", "").rstrip("/") or "/"

    # 关键：使用 Mount 将整个 static_assets 应用挂载到根路径
    return [
        Mount(
            mount_path,          # 挂载点："/" 或 "/myapp"
            app=static_assets,   # 被挂载的 ASGI 应用：_StreamlitStaticFiles
            name="static-assets",
        )
    ]
```

### 2.3 挂载点路径的处理细节

注释中特别提到尾部斜杠的处理：
> Strip trailing slash from the path because Starlette's Mount with a trailing
> slash (e.g., "/myapp/") won't match requests without it (e.g., "/myapp").
> Mount without trailing slash handles both cases by redirecting "/myapp" to
> "/myapp/". Use "/" as fallback for root path.

这是一个 Starlette 的行为适配：带尾部斜杠的 Mount 无法匹配不带斜杠的请求，而不带斜杠的 Mount 可以通过重定向兼容两种情况。

### 2.4 自定义 StaticFiles 处理器

`create_streamlit_static_handler()` 返回的 `_StreamlitStaticFiles` 类继承了 Starlette 的 `StaticFiles`，但重写了三个关键方法：

| 重写方法 | 新增功能 |
|---|---|
| `__call__` | 双斜杠保护 → `is_unsafe_path_pattern` 检查 → 尾部斜杠重定向 |
| `get_response` | 404 回退到 `index.html`（排除保留路径 `_stcore/health`、`_stcore/host-config`）→ 应用缓存头 |
| `_apply_cache_headers` | 根据文件名判断是 HTML/manifest（`no-cache`）还是带哈希资源（`immutable + max-age=31536000`） |

独立挂载处理器: [starlette_static_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_static_routes.py#L48-L166)

---

## 3. 上传路由跳过全量路径校验的原因

### 3.1 快速路径的设计

在 [PathSecurityMiddleware](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_path_security_middleware.py#L11-L55) 的开头注释明确说明了快速路径的设计原则：

```
Fast-Path Optimization
----------------------
For performance, certain known-safe routes skip the is_unsafe_path_pattern()
validation. This is a **performance optimization only**, not a security boundary.

Why upload_file/ is safe to skip:
The /_stcore/upload_file/{session_id}/{file_id} route uses session_id and file_id
as opaque dictionary keys in MemoryUploadedFileManager, never as filesystem paths.
Even a malicious session_id like "../../../etc/passwd" is just a failed dict lookup,
not a path traversal - the values are never passed to open() or os.path functions.
```

### 3.2 快速路径的判定代码

```python
# 安全路径列表
safe_exact_paths = frozenset({
    make_url_path(base_url_path, ROUTE_HEALTH),           # /_stcore/health
    make_url_path(base_url_path, ROUTE_SCRIPT_HEALTH),    # /_stcore/script-health-check
    make_url_path(base_url_path, ROUTE_METRICS),          # /_stcore/metrics
    make_url_path(base_url_path, ROUTE_HOST_CONFIG),      # /_stcore/host-config
})

# 上传路径前缀（带尾部斜杠避免匹配到其他路由）
_SAFE_ROUTE_UPLOAD_PREFIX = f"{BASE_ROUTE_UPLOAD_FILE}/"  # "_stcore/upload_file/"
safe_path_prefix = make_url_path(base_url_path, _SAFE_ROUTE_UPLOAD_PREFIX)

# 中间件中的快速路径判断
if path in self._safe_exact_paths or path.startswith(self._safe_path_prefix):
    await self.app(scope, receive, send)  # 直接放行，不执行 is_unsafe_path_pattern
    return
```

### 3.3 为什么上传路径是安全的：完整证据链

上传路由的 URL 格式是 `/_stcore/upload_file/{session_id}/{file_id}`，其中 `session_id` 和 `file_id` 只用作**字典键**，**永不作为文件系统路径**。完整证据链：

#### 证据 1：MemoryUploadedFileManager 只做字典查找

在 [MemoryUploadedFileManager](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py#L33-L118) 中：

```python
class MemoryUploadedFileManager(UploadedFileManager):
    def __init__(self, upload_endpoint: str) -> None:
        # file_storage 是一个嵌套字典：{session_id: {file_id: UploadedFileRec}}
        self.file_storage: dict[str, dict[str, UploadedFileRec]] = defaultdict(dict)

    def add_file(self, session_id: str, file: UploadedFileRec) -> None:
        # session_id 和 file_id 只用作字典键
        self.file_storage[session_id][file.file_id] = file

    def remove_file(self, session_id: str, file_id: str) -> None:
        # 同样只做字典操作，无任何 os.path 调用
        session_storage = self.file_storage[session_id]
        session_storage.pop(file_id, None)
```

#### 证据 2：上传路由处理器只做字典查找

在 [create_upload_routes()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L593-L745) 的 `_upload_put` 和 `_upload_delete` 中：

```python
async def _upload_put(request: Request) -> Response:
    session_id = request.path_params["session_id"]
    file_id = request.path_params["file_id"]

    # 1. session_id 仅用于活跃会话校验（字典键）
    if not runtime.is_active_session(session_id):
        raise HTTPException(status_code=400, detail="Invalid session_id")

    # 2. 读取上传的文件数据（file_id 用于存储时的字典键）
    upload_mgr.add_file(
        session_id=session_id,
        file=UploadedFileRec(
            file_id=file_id,   # 只作为存储键
            name=upload.filename or "",
            type=upload.content_type or "application/octet-stream",
            data=data,         # 实际文件内容存在内存中
        ),
    )

async def _upload_delete(request: Request) -> Response:
    session_id = request.path_params["session_id"]
    file_id = request.path_params["file_id"]

    # 同样是字典操作
    upload_mgr.remove_file(session_id=session_id, file_id=file_id)
```

**全程没有任何 `os.path` 调用，没有任何 `open()` 调用。** 即使 `session_id` 是 `../../../etc/passwd`，也只会导致：
```python
self.file_storage["../../../etc/passwd"][file_id] = file  # 安全的字典键
```
这只是一个奇怪的字典键名，不会造成任何文件系统操作。

#### 证据 3：URL 生成时使用 UUID，不可预测

URL 由后端生成，前端无法控制 `session_id` 和 `file_id` 的值：

```python
# [memory_uploaded_file_manager.py#L104-L118]
def get_upload_urls(self, session_id: str, file_names: Sequence[str]) -> list[UploadFileUrlInfo]:
    result = []
    for _ in file_names:
        file_id = str(uuid.uuid4())  # file_id 是随机 UUID
        result.append(
            UploadFileUrlInfo(
                file_id=file_id,
                upload_url=f"{self.endpoint}/{session_id}/{file_id}",
                delete_url=f"{self.endpoint}/{session_id}/{file_id}",
            )
        )
    return result
```

`session_id` 也是由后端生成的会话标识，不是用户可控的路径字符串。

### 3.4 检查顺序的安全性

中间件的检查顺序至关重要，**UNC 双斜杠检查始终在快速路径之前**：

```python
# [starlette_path_security_middleware.py#L137-L174]
async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
    # Step 1: UNC 双斜杠检查（对所有请求强制执行，包括上传路径）
    if path.startswith(("//", "\\\\")):
        return 400

    # Step 2: 快速路径绕过（仅当 UNC 检查通过后才执行）
    if path in self._safe_exact_paths or path.startswith(self._safe_path_prefix):
        await self.app(scope, receive, send)
        return

    # Step 3: 全量 is_unsafe_path_pattern 检查
    if relative_path and is_unsafe_path_pattern(relative_path):
        return 400
```

这意味着即使是上传路径，`//server/share` 这类 UNC 攻击仍然会被第一步拦截。快速路径**只跳过** `is_unsafe_path_pattern()` 的**路径穿越/盘符/绝对路径**检查，因为这些攻击对上传路由无效。

---

## 4. 静态部署时外部静态源改写

### 4.1 什么是 Static Connection 模式

Streamlit 支持**静态部署**（Static App）模式：将应用的全部状态序列化为 protobuf 文件，上传到 S3 等静态存储，前端直接从 S3 加载资源，不需要连接后端 WebSocket。这种模式下，所有媒体资源 URL 都需要被改写为 S3 地址。

### 4.2 静态连接建立流程

在 [establishStaticConnection()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/StaticConnection.tsx#L125-L150) 中：

```typescript
export async function establishStaticConnection(
    staticAppId: string,                      // 从 URL 参数 ?staticAppId=xxx 获取
    onConnectionStateChange: OnConnectionStateChange,
    onMessage: OnMessage,
    onConnectionError: (message: ErrorDetails) => void,
    endpoints: StreamlitEndpoints
): Promise<void> {
    onConnectionStateChange(ConnectionState.STATIC_CONNECTING)

    // Step 1: 获取静态配置 URL（S3 bucket 地址）
    const staticConfigUrl = await getStaticConfig()
    endpoints.setStaticConfigUrl(staticConfigUrl)  // 保存到 endpoints，供后续 URL 构建使用

    // Step 2: 从 S3 加载序列化的 protobuf 消息并渲染
    dispatchAppForwardMessages(staticAppId, staticConfigUrl, onMessage, onConnectionError)

    onConnectionStateChange(ConnectionState.STATIC_CONNECTED)
}
```

### 4.3 静态配置 URL 的获取与缓存

[getStaticConfig()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/StaticConnection.tsx#L36-L71) 函数负责获取静态资源的根地址：

```typescript
// 静态配置的固定地址（存放 static_url 指向实际的 S3 bucket）
const STATIC_ASSET_CONFIG = "https://data.streamlit.io/static.json"

export async function getStaticConfig(): Promise<string> {
    // 1. 优先从 localStorage 读取缓存
    const isLocalStoreAvailable = localStorageAvailable()
    if (isLocalStoreAvailable) {
        const cachedStaticConfigUrl = window.localStorage.getItem("stStaticAssetUrl")
        if (cachedStaticConfigUrl) {
            return cachedStaticConfigUrl
        }
    }

    // 2. 从固定地址获取配置
    const response = await fetch(STATIC_ASSET_CONFIG, { signal: AbortSignal.timeout(5000) })
    const config = await response.json()
    staticConfigUrl = config.static_url  // 例如："https://static.streamlit.io"

    // 3. 缓存到 localStorage
    if (isLocalStoreAvailable && staticConfigUrl) {
        window.localStorage.setItem("stStaticAssetUrl", staticConfigUrl)
    }

    return staticConfigUrl
}
```

### 4.4 静态部署下的 URL 改写逻辑

当 `staticConfigUrl` 被设置后，[DefaultStreamlitEndpoints.buildMediaURL()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts#L187-L204) 会将相对媒体 URL 改写为外部静态源地址：

```typescript
// 端点常量
const MEDIA_ENDPOINT = "/media"
const STATIC_SERVING_ENDPOINT = "/app/static/"

public buildMediaURL(url: string): string {
    // 关键判断：如果 staticConfigUrl 已设置（静态部署模式）
    if (this.staticConfigUrl) {
        // 对 /media 和 /app/static/ 开头的 URL 都进行改写
        if (url.startsWith(MEDIA_ENDPOINT) || url.startsWith(STATIC_SERVING_ENDPOINT)) {
            return this.buildStaticUrl(url)  // 改写为 S3 地址
        }
    }

    // 正常模式：拼接后端服务器地址
    if (url.startsWith(MEDIA_ENDPOINT) || url.startsWith(STATIC_SERVING_ENDPOINT)) {
        return buildHttpUri(this.requireServerUri(), url)
    }
    return url
}

// 实际改写逻辑
private buildStaticUrl(file: string): string {
    // 从 URL 参数获取 staticAppId
    const queryParams = new URLSearchParams(document.location.search)
    const staticAppId = queryParams.get("staticAppId")

    // 构造成：https://static.streamlit.io/{staticAppId}/media/xxx.png
    return `${this.staticConfigUrl}/${staticAppId}${file}`
}
```

### 4.5 Protobuf 消息加载

应用的实际内容通过序列化的 protobuf 加载：

```typescript
// [StaticConnection.tsx#L76-L96]
export async function getProtoResponse(
    staticAppId: string,
    staticConfigUrl: string
): Promise<null | ArrayBuffer> {
    // 从 S3 加载序列化的 protobuf 消息：https://static.streamlit.io/{appId}/protos.pb
    const path = `${staticConfigUrl}/${staticAppId}/protos.pb`
    const response = await fetch(path, { signal: AbortSignal.timeout(5000) })
    return response.arrayBuffer()
}

// [StaticConnection.tsx#L100-L123]
export async function dispatchAppForwardMessages(...) {
    const arrayBuffer = await getProtoResponse(staticAppId, staticConfigUrl)
    const forwardMsgList = ForwardMsgList.decode(new Uint8Array(arrayBuffer))

    // 将 protobuf 消息逐一分发到 App.tsx 的 handleMessage，实现无后端渲染
    forwardMsgList.messages.forEach(msg => {
        onMessage(msg as ForwardMsg)
    })
}
```

### 4.6 下载 URL 的额外改写

除了媒体 URL，下载 URL 也支持通过 `StreamlitConfig.DOWNLOAD_ASSETS_BASE_URL` 配置 CDN 域名：

```typescript
// [DefaultStreamlitEndpoints.ts#L213-L223]
public buildDownloadUrl(url: string): string {
    if (!url.startsWith(MEDIA_ENDPOINT)) {
        return url
    }

    // 如果配置了 DOWNLOAD_ASSETS_BASE_URL（通过 window.__streamlit 注入）
    const downloadAssetBaseUrl = StreamlitConfig.DOWNLOAD_ASSETS_BASE_URL
    return downloadAssetBaseUrl
        ? buildHttpUri(parseUriIntoBaseParts(downloadAssetBaseUrl), url)
        : buildHttpUri(this.requireServerUri(), url)
}
```

`StreamlitConfig` 在模块加载时从 `window.__streamlit` 捕获并冻结：

```typescript
// [config/index.ts#L225-L255]
const capturedConfig = (() => {
    const windowConfig = window.__streamlit
    if (!windowConfig) return undefined
    const cloned = deepClone(windowConfig)
    return deepFreeze(cloned)  // 深度冻结，防止运行时篡改
})()

export const StreamlitConfig = {
    get DOWNLOAD_ASSETS_BASE_URL(): string | undefined {
        return capturedConfig?.DOWNLOAD_ASSETS_BASE_URL
    },
    // ... 其他配置项
} as const
```

---

## 5. 整体架构回顾：路径映射、访问控制、缓存的边界

### 5.1 路径映射 × 访问控制 × 缓存 的交互矩阵

| 资源类型 | 路径映射方式 | 访问控制 | 缓存策略 |
|---|---|---|---|
| 核心前端资产 | `Mount` 独立挂载到 `/`，继承 StaticFiles | `_StreamlitStaticFiles.__call__` 内双斜杠 + `is_unsafe_path_pattern` 检查 | HTML: `no-cache`；带哈希资源: `immutable + max-age=1年` |
| 媒体文件 | `Route` 精确匹配 `/media/{file_id}` | CORS + `Content-Disposition` 处理 | *(未设置)* |
| 自定义组件 v1/v2 | `Route` 匹配 `/component/{name}/{path:path}` | `build_safe_abspath` 做路径规范化 + 根目录校验 | HTML: `no-cache`；其他: `public` |
| App 用户静态文件 | `Route` 匹配 `/app/static/{path:path}` | `build_safe_abspath` + 文件大小限制 200MB + `X-Content-Type-Options: nosniff` | *(未设置)* |
| 文件上传 | `Route` 匹配 `/_stcore/upload_file/{session_id}/{file_id}` | 快速路径跳过 `is_unsafe_path_pattern`（因为是字典键）+ XSRF + 会话校验 | *(未设置)* |

### 5.2 关键安全边界的代码定位

| 安全机制 | 所在文件 | 关键函数 |
|---|---|---|
| 全局路径安全（第一层） | [starlette_path_security_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_path_security_middleware.py) | `PathSecurityMiddleware.__call__` |
| 路径模式检测（核心算法） | [path_security.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/path_security.py) | `is_unsafe_path_pattern` |
| 路径规范化与根校验（第二层） | [component_file_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/component_file_utils.py) | `build_safe_abspath` |
| CORS 跨域控制 | [server_util.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/server_util.py) | `allow_all_cross_origin_requests`, `is_allowed_origin` |
| XSRF 防护 | [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py) | `_ensure_xsrf_cookie`, `_check_xsrf` |
| 选择性 GZip | [starlette_gzip_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_gzip_middleware.py) | `SelectiveGZipMiddleware.__call__` |
| 前端 URL 构建 | [DefaultStreamlitEndpoints.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts) | `buildMediaURL`, `buildStaticUrl`, `buildDownloadUrl` |
| 静态部署模式 | [StaticConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/StaticConnection.tsx) | `establishStaticConnection`, `getStaticConfig` |

---

## 6. 关键文件索引

| 文件 | 职责 |
|---|---|
| [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_app.py) | 应用组装、中间件顺序、6类路由分流编排 |
| [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py) | 各业务路由工厂函数（媒体、组件、上传、App静态等） |
| [starlette_static_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_static_routes.py) | 核心前端资产的独立 Mount 挂载、_StreamlitStaticFiles 自定义处理器 |
| [starlette_path_security_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_path_security_middleware.py) | 全局路径安全中间件、快速路径绕过逻辑 |
| [path_security.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/path_security.py) | is_unsafe_path_pattern 核心算法（UNC、盘符、穿越检测） |
| [component_file_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/component_file_utils.py) | build_safe_abspath 安全路径构建 |
| [memory_uploaded_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py) | 上传文件仅做字典存储的证据（无文件系统操作） |
| [starlette_gzip_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_gzip_middleware.py) | 选择性 GZip 压缩（跳过静态路径和音视频） |
| [server_util.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/server_util.py) | CORS 策略、XSRF 开关 |
| [starlette_server_config.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_server_config.py) | STATIC_ASSET_CACHE_MAX_AGE_SECONDS 等配置常量 |
| [DefaultStreamlitEndpoints.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts) | 前端 URL 构建、静态部署 URL 改写、XSRF 头注入 |
| [StaticConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/StaticConnection.tsx) | 静态部署模式建立、S3 配置获取、Protobuf 消息加载 |
| [config/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/utils/src/config/index.ts) | StreamlitConfig 捕获与冻结（含 DOWNLOAD_ASSETS_BASE_URL） |
| [vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/app/vite.config.ts) | 构建产物哈希命名（支持强缓存）、开发代理 |
