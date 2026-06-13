# Streamlit 静态资源服务链路深度分析

本文档深入分析 Streamlit 静态资源服务链路的**代码实现细节**，重点讲解：
1. **资源分流数量** — 有多少类静态资源，各路由如何创建和分流
2. **核心静态资源的独立挂载** — 为何使用 `Mount` 独立挂载，具体如何实现
3. **上传路由跳过全量路径校验的原因** — 为何 `/_stcore/upload_file/*` 可以安全绕过 `is_unsafe_path_pattern()`
4. **静态部署时外部静态源改写** — Static Connection 模式下如何将资源 URL 改写为外部静态源（如 S3）
5. **dev/prod 静态资源入口差异** — 开发态与生产态在后端路由、CORS 策略、前端代理上的不同
6. **静态资源连接状态切换时序** — ConnectionState 状态机的完整转移路径和触发条件
7. **dev/prod 资源入口分流情况** — 代码层面如何判断开发模式并分流路由
8. **上传路由依赖的安全边界** — 上传路由的五层安全防御体系（除路径校验外的其他防线）

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
| 2 | 媒体文件 | `create_media_routes()` | `/media/{file_id:path}` | st.image / st.audio / st.video / st.download_button |
| 3 | 自定义组件 v1 | `create_component_routes()` | `/component/{name}/{path:path}` | 第三方自定义组件的 HTML/JS/CSS |
| 4 | 自定义组件 v2 | `create_bidi_component_routes()` | `/_stcore/bidi-components/{component_name}/{path:path}` | 新一代双向组件资源 |
| 5 | App 用户静态文件 | `create_app_static_serving_routes()` | `/app/static/{path:path}` | 应用目录 `static/` 下的用户文件 |
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
    staticAppId: string,
    onConnectionStateChange: OnConnectionStateChange,
    onMessage: OnMessage,
    onConnectionError: (message: ErrorDetails) => void,
    endpoints: StreamlitEndpoints
): Promise<void> {
    onConnectionStateChange(ConnectionState.STATIC_CONNECTING)

    // Step 1: await — 等待静态配置 URL 返回
    const staticConfigUrl = await getStaticConfig()
    endpoints.setStaticConfigUrl(staticConfigUrl)

    // Step 2: 没有 await！fire-and-forget
    // eslint-disable-next-line @typescript-eslint/no-floating-promises
    dispatchAppForwardMessages(
        staticAppId,
        staticConfigUrl,
        onMessage,
        onConnectionError
    )

    // Step 3: 立即切换为 STATIC_CONNECTED，不等待 protobuf 加载完成
    onConnectionStateChange(ConnectionState.STATIC_CONNECTED)
}
```

**关键发现**：`dispatchAppForwardMessages()` 是一个返回 `Promise<void>` 的异步函数，但在调用时**没有 `await`**。eslint 注释 `@typescript-eslint/no-floating-promises` 也确认了这是有意为之的 fire-and-forget 调用。这意味着：

- `STATIC_CONNECTED` 状态在 `dispatchAppForwardMessages` 调用发起后**立即**切换
- protobuf 消息的加载和解码在后台异步进行
- 消息分发完成时，状态已经是 `STATIC_CONNECTED`

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

## 5. dev/prod 静态资源入口差异

### 5.1 开发态 vs 生产态的核心差异矩阵

| 维度 | 开发态 (developmentMode=True) | 生产态 (developmentMode=False) |
|---|---|---|
| 核心前端资产来源 | Vite Dev Server 提供（默认端口 3000） | 后端 Python 通过 `Mount` 挂载 `static/` 目录 |
| 路由存在性 | `create_streamlit_static_assets_routes()` 不添加 | `create_streamlit_static_assets_routes()` 被调用，挂载到 `/` |
| CORS 策略 | 允许所有跨域请求（`*`），因为 Vite 和后端端口不同 | 根据 `server.enableCORS` 和 `server.corsAllowedOrigins` 严格控制 |
| Host Config | 自动添加 `http://localhost` 到 `allowedOrigins` | 仅使用配置的 `client.allowedOrigins` |
| 前端代理 | Vite dev server 配置代理规则，将 `/_stcore/*`、`/media/*` 等转发到后端 | 无代理，浏览器直接请求同一域名 |
| XSRF 开关 | `server.enableXsrfProtection` 默认 True，开发态下仍执行除非显式关闭 | 同左 |

### 5.2 开发态判断的代码位置

开发态由 `global.developmentMode` 配置项控制，在三处关键代码中影响路由分流：

#### 5.2.1 后端路由分流（最关键）

在 [create_streamlit_routes()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L125-L149)：

```python
def create_streamlit_routes(runtime: Runtime) -> list[BaseRoute]:
    routes: list[Any] = []

    # ... 其他路由始终添加 ...

    # 10. App 用户静态文件（需启用配置，与 dev_mode 无关）
    if config.get_option("server.enableStaticServing"):
        routes.extend(create_app_static_serving_routes(main_script_path, base_url))

    # 11. 脚本健康检查（可选）
    if config.get_option("server.scriptHealthCheckEnabled"):
        routes.extend(create_script_health_routes(runtime, base_url))

    # 12. 核心前端资产：仅生产模式添加！
    dev_mode = bool(config.get_option("global.developmentMode"))
    if not dev_mode:
        routes.extend(create_streamlit_static_assets_routes(base_url=base_url))

    return routes
```

**关键结论**：开发模式下，后端不提供核心前端资产路由，完全由 Vite Dev Server 负责。

#### 5.2.2 CORS 策略差异

在 [allow_all_cross_origin_requests()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/server_util.py#L41-L50)：

```python
def allow_all_cross_origin_requests() -> bool:
    """True if cross-origin requests from any origin are allowed.

    We allow ALL cross-origin requests when CORS protection has been disabled
    with server.enableCORS=False or when in dev mode (where the Vite dev server
    and backend use different ports, counting as two origins).
    """
    return not config.get_option("server.enableCORS") or config.get_option(
        "global.developmentMode"
    )
```

开发模式下，Vite Dev Server（默认 3000 端口）和 Python 后端（默认 8501 端口）是两个不同的源，浏览器会对跨域请求执行预检，因此需要放开 CORS 限制。

#### 5.2.3 Host Config 差异

在 [create_host_config_routes()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L456-L462)：

```python
async def _host_config_endpoint(request: Request) -> JSONResponse:
    allowed: list[str] = list(config.get_option("client.allowedOrigins"))
    if (
        config.get_option("global.developmentMode")
        and "http://localhost" not in allowed
    ):
        allowed.append("http://localhost")  # 开发模式自动添加 localhost
```

### 5.3 前端 Vite 代理配置（开发态特有）

开发模式下，Vite Dev Server 配置了代理规则，将非前端资源的请求转发到 Python 后端（默认 `http://localhost:8501`）：

```javascript
// [vite.config.ts#L156-L191]
// DEV_SERVER_BACKEND_URL 默认 "http://localhost:8501"
// DEV_SERVER_PORT 默认 3000（通过 VITE_PORT 或 PORT 环境变量覆盖）

server: {
    open: true,
    port: DEV_SERVER_PORT,  // 默认 3000
    host: true,
    proxy: {
        // 1. 内部 API 全部代理（含 WebSocket）
        "^.*/_stcore/.*": {
            target: DEV_SERVER_BACKEND_URL,
            changeOrigin: true,
            ws: true,  // WebSocket 也代理
        },
        // 2. 媒体文件代理（用负向前瞻排除 Vite 的 /static/media/ 字体目录）
        "^(?!.*/static/media).*/media/.*": {
            target: DEV_SERVER_BACKEND_URL,
            changeOrigin: true,
        },
        // 3. 自定义组件代理
        "^.*/component/.*": {
            target: DEV_SERVER_BACKEND_URL,
            changeOrigin: true,
        },
        // 4. App 静态文件代理
        "^.*/app/static/.*": {
            target: DEV_SERVER_BACKEND_URL,
            changeOrigin: true,
        },
        // 5. 认证路由代理
        "^.*/auth/.*": {
            target: DEV_SERVER_BACKEND_URL,
            changeOrigin: true,
        },
        // 6. OAuth 回调代理
        "^.*/oauth2callback": {
            target: DEV_SERVER_BACKEND_URL,
            changeOrigin: true,
        },
    },
}
```

代理规则使用正则表达式，巧妙地将 Vite 自己的 `/static/media/` 路径排除（负向前瞻 `(?!.*/static/media)`），避免字体等 Vite 管理的资源被误转发到后端。

### 5.4 入口切换逻辑

前端入口通过 `ConnectionManager` 在运行时动态判断连接类型，不直接区分 dev/prod。但由于：
- 开发态：`window.location` 指向 Vite 端口（默认 3000），代理透明转发
- 生产态：`window.location` 指向后端服务端口，直接请求后端

因此前端代码不需要做显式的 dev/prod 判断，完全由后端的配置和 Vite 的代理机制透明处理。

---

## 6. 静态资源连接状态切换时序

### 6.1 ConnectionState 状态枚举

前端连接状态由 [ConnectionState](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/ConnectionState.ts#L17-L25) 枚举定义，共 7 个状态：

```typescript
export enum ConnectionState {
  CONNECTED = "CONNECTED",              // 已连接到后端 WebSocket
  DISCONNECTED_FOREVER = "DISCONNECTED_FOREVER",  // 永久断开（无法重连）
  INITIAL = "INITIAL",                  // 初始状态，未连接
  PINGING_SERVER = "PINGING_SERVER",    // 正在 ping 服务器检测可用性
  CONNECTING = "CONNECTING",            // 正在建立 WebSocket 连接
  STATIC_CONNECTING = "STATIC_CONNECTING",  // 静态模式正在加载
  STATIC_CONNECTED = "STATIC_CONNECTED",     // 静态模式加载完成
}
```

### 6.2 连接类型选择：WebSocket vs Static

[ConnectionManager.connect()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/ConnectionManager.ts#L181-L216) 是连接入口，根据 URL 参数 `staticAppId` 选择连接类型：

```typescript
private async connect(): Promise<void> {
    const staticAppId = this.checkStaticConnection()  // 检查 URL ?staticAppId=xxx

    if (staticAppId) {
        // 静态连接分支
        establishStaticConnection(
            staticAppId,
            this.setConnectionState,
            this.props.onMessage,
            this.props.onConnectionError,
            this.props.endpoints
        )
        this.websocketConnection = null  // 静态模式不需要 WebSocket
    } else {
        // WebSocket 连接分支
        try {
            this.websocketConnection = await this.connectToRunningServer()
        } catch (e) {
            this.setConnectionState(ConnectionState.DISCONNECTED_FOREVER, {
                message: err.message,
            })
        }
    }
}
```

### 6.3 WebSocket 连接时序（正常模式）

[WebsocketConnection](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/WebsocketConnection.tsx#L330-L399) 使用**有限状态机（FSM）**管理连接生命周期。

#### 6.3.1 标准时序（非 Bypass 模式）

```
INITIAL
   │ INITIALIZED 事件
   ▼
PINGING_SERVER
   │ doInitPings() 循环 ping /_stcore/health
   │ 直到服务器返回 200，同时获取 /_stcore/host-config
   │ SERVER_PING_SUCCEEDED 事件
   ▼
CONNECTING
   │ 尝试建立 WebSocket 连接
   │ CONNECTION_SUCCEEDED 事件
   ▼
CONNECTED  ←───────────────────┐
   │                            │
   │ CONNECTION_CLOSED 或      │ 重连成功
   │ CONNECTION_ERROR 事件     │ CONNECTION_SUCCEEDED
   ▼                            │
PINGING_SERVER  ───────────────┘
   │ SERVER_PING_SUCCEEDED
   ▼
CONNECTING
```

#### 6.3.2 Bypass 模式时序（优化延迟）

如果 `isHostConfigBypassEnabled()` 返回 true（通过 `window.__streamlit` 预配置了足够的信息），可以并行执行 ping 和 WebSocket 连接，降低首屏延迟：

```
INITIAL
   │ INITIALIZED 事件（enableBypass=true）
   ├──────────────────────────────────┐
   │ CONNECTING 状态转移               │ pingServerInBackground()
   │ WebSocket 连接尝试                 │ 同时在后台执行 ping
   ▼                                  ▼
CONNECTED                      后台 ping 完成
   │ (WebSocket 先就绪)               │ (不触发状态转移)
   │ CONNECTION_CLOSED
   ▼
PINGING_SERVER  →  CONNECTING  →  CONNECTED
```

Bypass 模式的判断条件在 [isHostConfigBypassEnabled()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/utils.ts#L39-L59)：

```typescript
export function isHostConfigBypassEnabled(): boolean {
    const initialHostConfig = StreamlitConfig.HOST_CONFIG
    if (!initialHostConfig) return false

    const { allowedOrigins, useExternalAuthToken } = initialHostConfig
    return (
        Boolean(StreamlitConfig.BACKEND_BASE_URL) &&          // 后端地址已知
        isValidAllowedOrigins(allowedOrigins) &&              // 允许的来源有效
        typeof useExternalAuthToken === "boolean"             // 外部认证 token 标志位已设置
    )
}
```

#### 6.3.3 重连时序

当连接意外断开时（如网络波动），FSM 会自动进入 PINGING_SERVER 状态尝试重连。额外支持**心跳超时重连**：

```typescript
// [ConnectionManager.ts#L231-L251]
public onHeartbeatSent(ackTimeoutMilliseconds: number): void {
    this.heartbeatAckTimeoutId = setTimeout(() => {
        if (this.isConnected()) {  // 只有 CONNECTED 状态下才重连
            this.reconnect()       // 关闭当前连接，进入 PINGING_SERVER
        }
    }, ackTimeoutMilliseconds)
}
```

### 6.4 静态连接时序

静态连接的状态转移需要注意一个关键细节：`dispatchAppForwardMessages()` 是 fire-and-forget 调用。

```
INITIAL
   │ connect() 检测到 staticAppId
   ▼
STATIC_CONNECTING
   │ await getStaticConfig()     ← 等待 S3 配置地址返回
   │ setStaticConfigUrl()        ← 设置到 endpoints
   │ dispatchAppForwardMessages()← 发起但**不等待**（fire-and-forget）
   │                                 protobuf 在后台异步加载
   ▼
STATIC_CONNECTED                 ← dispatch 调用后立即切换
                                    此时 protobuf 消息可能尚未解码完成
```

时序代码在 [establishStaticConnection()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/StaticConnection.tsx#L125-L150)：

```typescript
export async function establishStaticConnection(...): Promise<void> {
    onConnectionStateChange(ConnectionState.STATIC_CONNECTING)

    // Step 1: await — 确实等待配置获取
    const staticConfigUrl = await getStaticConfig()
    endpoints.setStaticConfigUrl(staticConfigUrl)

    // Step 2: 没有 await — fire-and-forget
    // eslint-disable-next-line @typescript-eslint/no-floating-promises
    dispatchAppForwardMessages(staticAppId, staticConfigUrl, onMessage, onConnectionError)

    // Step 3: 立即切换，不等待 Step 2 完成
    onConnectionStateChange(ConnectionState.STATIC_CONNECTED)
}
```

这种设计的原因是：`onMessage` 回调会触发 React 状态更新和组件渲染，这些操作本身就是在事件循环中异步排队的。将状态先切为 `STATIC_CONNECTED`，可以让 UI 层在收到第一条消息时就已经处于"已连接"状态，避免渲染时序问题。

### 6.5 状态转移的安全边界

状态机的设计严格防止非法转移，[WebsocketConnection.stepFsm()](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/WebsocketConnection.tsx#L330-L399) 中只允许预定义的转移：

```typescript
switch (this.state) {
  case ConnectionState.INITIAL:
    if (event === "INITIALIZED") { ... }  // 唯一合法事件
    break
  case ConnectionState.CONNECTING:
    if (event === "CONNECTION_SUCCEEDED") { ... }
    if (event === "CONNECTION_TIMED_OUT" ||
        event === "CONNECTION_ERROR" ||
        event === "CONNECTION_CLOSED") { ... }
    break
  // ... 其他状态的合法转移
  default:
    throw new Error("Unsupported state transition")
}
```

---

## 7. dev/prod 资源入口分流情况

### 7.1 开发模式的代码分流路径

开发模式下，`global.developmentMode=True`，核心前端资产由 Vite 提供，后端不提供。完整的分流链路：

```
浏览器访问 http://localhost:3000 (Vite Dev Server)
    │
    ├─ /index.html               → Vite 直接提供（HTML 入口）
    ├─ /src/...                   → Vite 直接提供（带 HMR 热更新）
    ├─ /@id/... / /node_modules/  → Vite 直接提供（依赖预构建）
    ├─ /static/media/...          → Vite 直接提供（字体等构建产物）
    │
    └─ 以下路径由 Vite 代理转发到 http://localhost:8501（Python 后端）：
        /_stcore/health            → 后端健康检查
        /_stcore/host-config       → 后端主机配置
        /_stcore/stream            → WebSocket 连接（ws: true 代理）
        /_stcore/upload_file/{session_id}/{file_id}  → 文件上传（PUT/DELETE）
        /_stcore/metrics           → 指标采集
        /_stcore/bidi-components/{component_name}/{path:path}  → 双向组件
        /media/{file_id:path}      → 媒体文件（排除 /static/media/）
        /component/{name}/{path:path}  → 自定义组件
        /app/static/{path:path}    → App 用户静态文件
        /auth/...                  → 认证路由
        /oauth2callback            → OAuth 回调
```

### 7.2 生产模式的代码分流路径

生产模式下，`global.developmentMode=False`，所有资源统一由 Python 后端提供。路由通过 `_with_base()` 函数统一处理 `server.baseUrlPath` 前缀：

```python
# [starlette_routes.py#L67-L89] 路由常量定义
BASE_ROUTE_CORE: Final = "_stcore"
BASE_ROUTE_MEDIA: Final = "media"                            # 没有 _stcore 前缀！
BASE_ROUTE_COMPONENT: Final = "component"                    # 没有 _stcore 前缀！
BASE_ROUTE_APP_STATIC_SERVING: Final = "app/static"          # 没有 _stcore 前缀！
BASE_ROUTE_BIDI_COMPONENTS: Final = "_stcore/bidi-components"  # 有 _stcore 前缀
BASE_ROUTE_UPLOAD_FILE: Final = "_stcore/upload_file"          # 有 _stcore 前缀

_ROUTE_MEDIA: Final = f"{BASE_ROUTE_MEDIA}/{{file_id:path}}"   # path 类型允许包含斜杠
```

**生产模式完整链路图**（路径精确对应代码常量）：

```
浏览器访问 https://app.example.com
    │
    ├─ /                       → Mount 挂载的 _StreamlitStaticFiles
    │   ├─ /static/js/xxx.js   → 带哈希，immutable 缓存
    │   └─ /index.html         → SPA fallback, no-cache
    │
    ├─ /_stcore/*              → Starlette Route 精确匹配（BASE_ROUTE_CORE 下）
    │   ├─ /_stcore/health
    │   ├─ /_stcore/host-config
    │   ├─ /_stcore/metrics
    │   ├─ /_stcore/stream (WebSocket)
    │   ├─ /_stcore/bidi-components/{component_name}/{path:path}
    │   └─ /_stcore/upload_file/{session_id}/{file_id}  → PUT/DELETE
    │
    ├─ /media/{file_id:path}   → MediaFileManager (独立路由，不在 _stcore 下！)
    ├─ /component/{name}/{path:path}  → ComponentRegistry (自定义组件 v1，不在 _stcore 下！)
    └─ /app/static/{path:path} → 用户 static/ 目录 (需启用 server.enableStaticServing，不在 _stcore 下！)
```

**关键事实**：`media`、`component`、`app/static` 这三类路由**不在 `/_stcore` 前缀下**，它们的 `BASE_ROUTE_*` 常量直接就是 `"media"`、`"component"`、`"app/static"`，没有 `_stcore/` 前缀。只有 `bidi-components` 和 `upload_file` 在 `_stcore/` 下。

### 7.3 分流判断的代码位置

核心分流逻辑位于 [starlette_app.py#L125-L149](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L125-L149)：

```python
dev_mode = bool(config.get_option("global.developmentMode"))

# 核心前端资产：仅生产模式添加
if not dev_mode:
    routes.extend(create_streamlit_static_assets_routes(base_url=base_url))
```

这个简单的 `if` 判断是整个 dev/prod 分流的**唯一控制开关**，它直接决定了：
- 是否将 `_StreamlitStaticFiles` 挂载到根路径
- 404 请求是否会回退到 `index.html`
- JS/CSS/HTML 等核心资产是否由 Python 提供

### 7.4 与 Base URL 的交互

当配置了 `server.baseUrlPath`（如 `"/myapp"`）时，分流仍然生效，只是所有路由都加上了前缀：

```python
# 生产模式下挂载点变为 "/myapp"
mount_path = make_url_path(base_url or "", "").rstrip("/") or "/"
# 结果：Mount("/myapp", app=static_assets)

# 上传路径变为 "/myapp/_stcore/upload_file/*"
safe_path_prefix = make_url_path(base_url_path, _SAFE_ROUTE_UPLOAD_PREFIX)
```

### 7.5 构建产物的配合

Vite 构建时会在 `output` 配置中为文件名添加内容哈希，配合生产模式的 `immutable + max-age=1年` 缓存策略：

```javascript
// [vite.config.ts#L192-L233]
build: {
    rollupOptions: {
        output: {
            manualChunks: {
                react: ["react", "react-dom"],
                // ... 其他 chunk 配置
            },
        },
    },
}
```

Vite 自动为 JS/CSS 文件名添加 `.[hash]` 后缀（如 `index.abc123.js`），确保内容变化时 URL 变化，从而安全使用长时缓存。

---

## 8. 上传路由依赖的安全边界

上传路由虽然跳过了 `is_unsafe_path_pattern()` 的全量路径校验，但依赖**五层独立的安全边界**确保安全性。每层边界独立工作，某层失效不影响其他层。

### 8.1 五层安全防御体系

| 层级 | 防御机制 | 代码位置 | 检查时机 |
|---|---|---|---|
| **L1** | UNC 双斜杠检查 | [starlette_path_security_middleware.py#L137-L139](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_path_security_middleware.py#L137-L139) | 中间件最外层，所有请求 |
| **L2** | XSRF 跨站请求伪造防护 | [starlette_routes.py#L598-L634](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L598-L634) | PUT/DELETE 处理函数入口 |
| **L3** | 活跃会话校验 | [starlette_routes.py#L664-L665](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L664-L665) | PUT 入口 |
| **L4** | 文件大小限制 | [starlette_routes.py#L667-L700](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L667-L700) | PUT 内容读取前后 |
| **L5** | CORS 跨域策略 | [starlette_routes.py#L636-L649](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L636-L649) | 所有响应设置 Header |

### 8.2 L1: UNC 双斜杠检查（所有请求强制执行）

即使是上传路由快速路径，UNC 检查仍然执行：

```python
# [starlette_path_security_middleware.py#L137-L139]
# Step 1: UNC 双斜杠检查（对所有请求强制执行，包括上传路径）
if path.startswith(("//", "\\\\")):
    return 400
```

防止 Windows UNC 路径攻击（如 `//server/share/file`），这类攻击可能触发 SMB 连接导致 NTLM 哈希泄露或 SSRF。

### 8.3 L2: XSRF 跨站请求伪造防护

上传是**非安全方法**（PUT/DELETE），必须通过 XSRF 检查。[`_check_xsrf()`](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L598-L634) 在 PUT 和 DELETE 入口第一行执行：

```python
def _check_xsrf(request: Request) -> None:
    """Check XSRF token for non-safe requests."""
    if not is_xsrf_enabled():
        return

    # 比较 Header 中的 X-Xsrftoken 和 Cookie 中的 xsrf token
    xsrf_header = request.headers.get("X-Xsrftoken")
    xsrf_cookie = request.cookies.get(XSRF_COOKIE_NAME)

    if not validate_xsrf_token(xsrf_header, xsrf_cookie):
        raise HTTPException(status_code=403, detail="XSRF token missing or invalid")

async def _upload_put(request: Request) -> Response:
    _check_xsrf(request)  # 第一行就执行
    session_id = request.path_params["session_id"]
    # ...

async def _upload_delete(request: Request) -> Response:
    _check_xsrf(request)  // 第一行就执行
    session_id = request.path_params["session_id"]
    # ...
```

XSRF 采用 Double-Submit Cookie 模式：
1. Cookie 值格式：`2|mask|token|timestamp`
2. JavaScript 从 Cookie 读取值，放入 `X-Xsrftoken` Header
3. 后端比较两者是否一致

XSRF Cookie 由健康检查接口设置（`_ensure_xsrf_cookie`），不设 `HttpOnly` 标志（JS 需要读取）。

### 8.4 L3: 活跃会话校验

PUT 请求需要验证 `session_id` 是活跃会话：

```python
# [starlette_routes.py#L664-L665]
session_id = request.path_params["session_id"]
if not runtime.is_active_session(session_id):
    raise HTTPException(status_code=400, detail="Invalid session_id")
```

这确保了：
- 攻击者不能构造任意 `session_id` 上传文件
- 只有真实运行的应用才能接收上传
- 会话过期后自动拒绝上传

注意：DELETE 请求**不做**会话校验，因为文件上传完成后可能会话已过期但仍需清理资源。

### 8.5 L4: 文件大小限制（双重检查）

采用**双重检查**确保不超过 `server.maxUploadSize` 限制（单位 MB）：

```python
# [starlette_routes.py#L667-L700]
max_size_bytes = config.get_option("server.maxUploadSize") * 1024 * 1024

# 1. Header 快速失败（Content-Length 存在时）
content_length = request.headers.get("content-length")
if content_length:
    if int(content_length) > max_size_bytes:
        raise HTTPException(status_code=413, detail="File too large")

# 2. 实际读取后二次检查（防止客户端伪造 Content-Length）
data = await upload.read()
if len(data) > max_size_bytes:
    raise HTTPException(status_code=413, detail="File too large")
```

### 8.6 L5: CORS 跨域策略（XSRF 启用时更严格）

当 XSRF 启用时，上传路由的 CORS 策略比普通路由更严格：

```python
# [starlette_routes.py#L636-L649]
async def _set_upload_headers(request: Request, response: Response) -> None:
    response.headers["Access-Control-Allow-Methods"] = "PUT, OPTIONS, DELETE"
    if is_xsrf_enabled():
        # 严格模式：仅允许特定 Origin，允许凭证，设置 Vary
        response.headers["Access-Control-Allow-Origin"] = get_url(
            config.get_option("browser.serverAddress")
        )
        response.headers["Access-Control-Allow-Headers"] = "X-Xsrftoken, Content-Type"
        response.headers["Vary"] = "Origin"
        response.headers["Access-Control-Allow-Credentials"] = "true"
    else:
        # 宽松模式：使用通用 CORS 策略
        await _set_cors_headers(request, response)
```

严格模式的关键点：
- `Access-Control-Allow-Credentials: true` 允许跨域请求携带 Cookie（XSRF Cookie 需要）
- `Vary: Origin` 确保不同 Origin 的响应不被缓存混淆
- 只允许 `X-Xsrftoken` 和 `Content-Type` 两个请求头

### 8.7 前端配合的安全措施：三层调用链 + XSRF 注入

前端上传不是直接调用 endpoints，而是经过**三层封装**：UI 组件层 → 业务封装层（FileUploadClient）→ 协议层（csrfRequest + axios）。

#### 8.7.1 调用链路全景

```
UI 组件层（4 处调用）：
  ├─ FileUploader.tsx     （st.file_uploader）
  ├─ ChatInput.tsx        （st.chat_input 带文件）
  ├─ CameraInput.tsx      （st.camera_input）
  └─ createFileUploadHandler.ts / uploadFiles.ts
        │
        ▼ 调用 uploadClient.uploadFile()
业务封装层：FileUploadClient
  ├─ uploadFile()         （追踪表单 pending 计数）
  │      └─ endpoints.uploadFileUploaderFile()
  ├─ deleteFile()         （调用 endpoints.deleteFileAtURL）
  └─ fetchFileURLs()      （通过 WebSocket 请求上传/删除 URL）
        │
        ▼
协议层：DefaultStreamlitEndpoints
  ├─ uploadFileUploaderFile()  （构造 FormData → csrfRequest）
  ├─ deleteFileAtURL()         （构造 payload → csrfRequest）
  └─ csrfRequest()             （核心：注入 X-Xsrftoken + withCredentials → axios）
```

#### 8.7.2 业务封装层（FileUploadClient）

[FileUploadClient](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/lib/src/FileUploadClient.ts#L47-L214) 负责追踪表单的 pending 请求计数，让 UI 组件知道表单是否还有正在上传的文件：

```typescript
// [FileUploadClient.ts#L102-L132]
public async uploadFile(
    widget: WidgetInfo,         // 组件的 id 和 formId
    fileUploadUrl: string,
    file: File,
    onUploadProgress?: ...,
    signal?: AbortSignal
): Promise<void> {
    // 先把表单 pending 计数 +1
    this.offsetPendingRequestCount(widget.formId, 1)
    return this.endpoints
      .uploadFileUploaderFile(   // 调用协议层
          fileUploadUrl,
          file,
          this.sessionInfo.current.sessionId,
          onUploadProgress,
          signal
      )
      .finally(() => this.offsetPendingRequestCount(widget.formId, -1))
}

public deleteFile(fileUrl: string): Promise<void> {
    return this.endpoints.deleteFileAtURL
        ? this.endpoints.deleteFileAtURL(fileUrl, this.sessionInfo.current.sessionId)
        : Promise.resolve()
}
```

`fetchFileURLs()` 方法用于先通过 WebSocket 向后端请求上传/删除 URL（避免前端硬编码路径）：

```typescript
// [FileUploadClient.ts#L144-L156]
public fetchFileURLs(files: File[]): Promise<IFileURLs[]> {
    const resolver = Promise.withResolvers<IFileURLs[]>()
    const requestId = uuidv4()
    this.pendingFileURLsRequests.set(requestId, resolver)
    this.requestFileURLs(requestId, files)  // 发送 BackMsg 到 WebSocket
    return resolver.promise
}
```

#### 8.7.3 协议层（DefaultStreamlitEndpoints）

协议层负责构造 HTTP 请求并注入 XSRF。

**上传函数真实代码**（变量名精确对应）：

```typescript
// [DefaultStreamlitEndpoints.ts#L269-L310]
public async uploadFileUploaderFile(
    fileUploadUrl: string,
    file: File,
    _sessionId: string,          // 注意：形参是 _sessionId（未使用，FileUploadClient 传入了 sessionId）
    onUploadProgress?: (progressEvent: AxiosProgressEvent) => void,
    signal?: AbortSignal
): Promise<void> {
    const form = new FormData()
    const { name, webkitRelativePath } = file
    // 目录上传时使用 webkitRelativePath 保留目录结构
    const fileName = webkitRelativePath || name
    form.append(name, file, fileName)

    const headers: Record<string, string> = this.getAdditionalHeaders()
    const uploadUrl = this.buildFileUploadURL(fileUploadUrl)

    try {
        await this.csrfRequest<number>(uploadUrl, {
            signal,
            method: "PUT",
            data: form,
            responseType: "text",
            headers,
            onUploadProgress,
        })
    } catch (error: unknown) {
        // 失败时发送 ClientError 到 Host
        const message = error instanceof Error ? error.message : "Unknown Error"
        this.sendClientErrorToHost("File Uploader", "Error uploading file", message, uploadUrl)
        throw error
    }
}
```

**删除函数真实代码**：

```typescript
// [DefaultStreamlitEndpoints.ts#L327-L355]
public async deleteFileAtURL(
    fileUrl: string,
    sessionId: string            // DELETE 的 sessionId 放在请求体中（非路径参数）
): Promise<void> {
    const headers: Record<string, string> = this.getAdditionalHeaders()
    const deleteUrl = this.buildFileUploadURL(fileUrl)

    await this.csrfRequest<number>(deleteUrl, {
        method: "DELETE",
        data: { sessionId },     // sessionId 在 body 中，不在 URL 路径
        headers,
    })
}
```

注意两处代码事实：
1. 上传 URL 中的 `session_id` 是路径参数（`/_stcore/upload_file/{session_id}/{file_id}`），由后端生成并通过 WebSocket 下发给前端
2. DELETE 的 `sessionId` 额外放在请求体中（后端 DELETE 处理器不校验会话）

#### 8.7.4 csrfRequest：XSRF 核心封装

**`csrfRequest()` 是核心封装**（[DefaultStreamlitEndpoints.ts#L382-L402](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts#L382-L402)）：

```typescript
private async csrfRequest<T = unknown, R = AxiosResponse<T>>(
    url: string,
    params: AxiosRequestConfig
): Promise<R> {
    params.url = url

    if (this.csrfEnabled) {
        // 1. 从 Cookie 读取 _streamlit_xsrf 的值
        const xsrfCookie = getCookie("_streamlit_xsrf")  // document.cookie 正则匹配
        if (notNullOrUndefined(xsrfCookie)) {
            // 2. 注入 X-Xsrftoken Header（Cookie 名和 Header 名需与后端匹配）
            params.headers = {
                "X-Xsrftoken": xsrfCookie,
                ...params.headers,
            }
            // 3. 启用跨域凭证发送（axios 语法，等价于 fetch 的 credentials: "include"）
            params.withCredentials = true
        }
    }

    // 4. 动态 import axios，避免在首屏 bundle 中引入
    const { default: axios } = await import("axios")
    return axios.request<T, R>(params)
}
```

关键细节：
- **HTTP 客户端**：使用 **axios**（非 `fetch`），通过动态 `import("axios")` 按需加载
- **XSRF Cookie 名**：`_streamlit_xsrf`（与后端 `XSRF_COOKIE_NAME` 一致）
- **Header 名**：`X-Xsrftoken`（与后端 `request.headers.get("X-Xsrftoken")` 一致）
- **凭证模式**：`withCredentials: true`（axios 语法，等价于 `fetch` 的 `credentials: "include"`，**不是** `"same-origin"`）
- **`csrfEnabled` 标志**：在 [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/app/src/App.tsx#L432-L434) 中硬编码为 `true`
- **`getCookie()` 实现**：通过 `document.cookie` 正则匹配读取（[browser/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/utils/src/browser/index.ts#L20-L23)）
- **额外 Headers**：如果 `fileUploadClientConfig` 存在（外部上传服务场景），会合并额外的 headers
- **错误上报**：上传失败时调用 `sendClientErrorToHost()` 将错误信息通过 WebSocket 上报

### 8.8 安全边界总结

上传路由的安全设计体现了"纵深防御"思想：
- 即使快速路径跳过了路径校验，UNC 检查仍在（L1）
- 即使 UNC 检查被绕过，XSRF 仍阻止跨站伪造（L2）
- 即使 XSRF 被绕过，会话校验仍阻止非法 session_id（L3）
- 即使会话校验被绕过，文件大小限制仍防止内存耗尽（L4）
- 即使以上都失效，CORS 策略仍限制跨域访问（L5）

---

## 9. 整体架构回顾：路径映射、访问控制、缓存的边界

### 9.1 路径映射 × 访问控制 × 缓存 的交互矩阵

| 资源类型 | 路径映射方式 | 访问控制 | 缓存策略 |
|---|---|---|---|
| 核心前端资产 | `Mount` 独立挂载到 `/`，继承 StaticFiles | `_StreamlitStaticFiles.__call__` 内双斜杠 + `is_unsafe_path_pattern` 检查 | HTML: `no-cache`；带哈希资源: `immutable + max-age=1年` |
| 媒体文件 | `Route` 精确匹配 `/media/{file_id:path}` | CORS + `Content-Disposition` 处理 | *(未设置)* |
| 自定义组件 v1/v2 | `Route` 匹配 `/component/{name}/{path:path}` 和 `/_stcore/bidi-components/{component_name}/{path:path}` | `build_safe_abspath` 做路径规范化 + 根目录校验 | HTML: `no-cache`；其他: `public` |
| App 用户静态文件 | `Route` 匹配 `/app/static/{path:path}` | `build_safe_abspath` + 文件大小限制 200MB + `X-Content-Type-Options: nosniff` | *(未设置)* |
| 文件上传 | `Route` 匹配 `/_stcore/upload_file/{session_id}/{file_id}` | 快速路径跳过 `is_unsafe_path_pattern` + 五层安全边界（UNC+XSRF+会话+大小+CORS），前端通过 axios `csrfRequest()` 自动注入 `X-Xsrftoken` 头并设置 `withCredentials: true` | *(未设置)* |

### 9.2 关键安全边界的代码定位

| 安全机制 | 所在文件 | 关键函数 |
|---|---|---|
| 全局路径安全（第一层） | [starlette_path_security_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_path_security_middleware.py) | `PathSecurityMiddleware.__call__` |
| 路径模式检测（核心算法） | [path_security.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/path_security.py) | `is_unsafe_path_pattern` |
| 路径规范化与根校验（第二层） | [component_file_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/component_file_utils.py) | `build_safe_abspath` |
| CORS 跨域控制 | [server_util.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/server_util.py) | `allow_all_cross_origin_requests`, `is_allowed_origin` |
| XSRF 防护 | [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py) | `_ensure_xsrf_cookie`, `_check_xsrf` |
| 选择性 GZip | [starlette_gzip_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_gzip_middleware.py) | `SelectiveGZipMiddleware.__call__` |
| 前端 URL 构建 | [DefaultStreamlitEndpoints.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts) | `buildMediaURL`, `buildStaticUrl`, `buildDownloadUrl`, `uploadFileUploaderFile` |
| 静态部署模式 | [StaticConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/StaticConnection.tsx) | `establishStaticConnection`, `getStaticConfig` |
| 连接状态管理 | [ConnectionManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/ConnectionManager.ts) | `connect`, `setConnectionState` |
| WebSocket 状态机 | [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/WebsocketConnection.tsx) | `stepFsm`, `setFsmState` |

---

## 10. 关键文件索引

| 文件 | 职责 |
|---|---|
| [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_app.py) | 应用组装、中间件顺序、6类路由分流编排、dev/prod 分流开关 |
| [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py) | 各业务路由工厂函数、XSRF 防护、上传五层安全边界 |
| [starlette_static_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_static_routes.py) | 核心前端资产的独立 Mount 挂载、_StreamlitStaticFiles 自定义处理器 |
| [starlette_path_security_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_path_security_middleware.py) | 全局路径安全中间件、快速路径绕过逻辑、UNC 检查 |
| [path_security.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/path_security.py) | is_unsafe_path_pattern 核心算法（UNC、盘符、穿越检测） |
| [component_file_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/component_file_utils.py) | build_safe_abspath 安全路径构建 |
| [memory_uploaded_file_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/runtime/memory_uploaded_file_manager.py) | 上传文件仅做字典存储的证据（无文件系统操作） |
| [starlette_gzip_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_gzip_middleware.py) | 选择性 GZip 压缩（跳过静态路径和音视频） |
| [server_util.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/server_util.py) | CORS 策略（dev 模式放开）、XSRF 开关 |
| [starlette_server_config.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_server_config.py) | STATIC_ASSET_CACHE_MAX_AGE_SECONDS 等配置常量 |
| [DefaultStreamlitEndpoints.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts) | 前端 URL 构建、静态部署 URL 改写、XSRF Header 注入 |
| [StaticConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/StaticConnection.tsx) | 静态部署模式建立、S3 配置获取、Protobuf 消息加载 |
| [ConnectionManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/ConnectionManager.ts) | 连接类型选择（WebSocket vs Static）、心跳超时重连 |
| [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/WebsocketConnection.tsx) | WebSocket 状态机、Bypass 模式并行连接 |
| [ConnectionState.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/ConnectionState.ts) | 7 种连接状态枚举定义 |
| [DoInitPings.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DoInitPings.tsx) | 初始服务器 ping 循环、host-config 获取 |
| [config/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/utils/src/config/index.ts) | StreamlitConfig 捕获与冻结（含 DOWNLOAD_ASSETS_BASE_URL） |
| [utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/utils.ts) | Host Config Bypass 判断、URL 解析工具 |
| [vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/app/vite.config.ts) | 构建产物哈希命名（支持强缓存）、开发代理配置 |
