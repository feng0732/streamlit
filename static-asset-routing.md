# Streamlit 静态资源服务链路分析

本文档深入分析 Streamlit 静态资源服务链路，涵盖**路径映射**、**访问控制**和**浏览器缓存**三个维度及其相互之间的边界划分。

---

## 1. 整体架构概览

Streamlit 的静态资源服务基于 Starlette ASGI 框架构建，由后端（Python）和前端（TypeScript/React）协同完成。请求处理链路如下：

```
浏览器请求
    │
    ▼
PathSecurityMiddleware (路径安全拦截，最先执行)
    │
    ▼
SessionMiddleware (会话管理)
    │
    ▼
SelectiveGZipMiddleware (选择性 GZip 压缩)
    │
    ▼
Starlette Routes 路由分发
    ├─ / 或 /{base_url}           → 核心前端资产 (JS/CSS/HTML)
    ├─ /_stcore/*                 → 内部 API (health, upload, stream, metrics, host-config, bidi-components)
    ├─ /media/{file_id}           → 媒体文件 (图片/音频/视频)
    ├─ /component/{name}/{path}   → 自定义组件 v1 资源
    ├─ /_stcore/bidi-components/  → 自定义组件 v2 资源
    ├─ /app/static/{path}         → App 用户静态文件
    └─ /auth/*                    → 认证路由
```

相关代码文件：
- 应用组装: [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L164-L205)
- 路由定义: [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L64-L98)

---

## 2. 路径映射

### 2.1 路由类型与 URL 结构

Streamlit 的静态资源路由可分为**五类**，每类有独立的路径前缀和处理逻辑：

| 资源类型 | 路径前缀 | 路由创建函数 | 生产/开发模式 |
|---|---|---|---|
| 核心前端资产 (JS/CSS/HTML) | `/` 或 `/{base_url}` | `create_streamlit_static_assets_routes()` | 仅生产模式 |
| 媒体文件 | `/media/{file_id}` | `create_media_routes()` | 通用 |
| 自定义组件 v1 | `/component/{name}/{path}` | `create_component_routes()` | 通用 |
| 自定义组件 v2 | `/_stcore/bidi-components/{name}/{path}` | `create_bidi_component_routes()` | 通用 |
| App 用户静态文件 | `/app/static/{path}` | `create_app_static_serving_routes()` | 需启用 `server.enableStaticServing` |
| 文件上传 | `/_stcore/upload_file/{session_id}/{file_id}` | `create_upload_routes()` | 通用 |

### 2.2 Base URL 路径前缀机制

所有路由均支持通过 `server.baseUrlPath` 配置添加统一前缀，由 `_with_base()` 函数统一处理：

```python
def _with_base(path: str, base_url: str | None = None) -> str:
    base = (
        base_url if base_url is not None else config.get_option("server.baseUrlPath")
    ) or ""
    return make_url_path(base, path)  # 拼接如 "/myapp/_stcore/health"
```

路径拼接工具: [url_util.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/url_util.py#L107-L127)

### 2.3 核心前端资产的 SPA Fallback

核心前端资产采用 `_StreamlitStaticFiles` 类（继承 Starlette `StaticFiles`），实现了 SPA（单页应用）路由回退：

- **404 回退**: 当静态文件不存在时（非保留路径），返回 `index.html` 以支持客户端路由
- **保留路径**: `/_stcore/health` 和 `/_stcore/host-config` 不回退，返回真实 404
- **尾部斜杠重定向**: 路径带尾部斜杠时 301 重定向到无斜杠版本（避免 mount root 无限循环）
- **双斜杠保护**: 以 `//` 开头的路径返回 400 Bad Request（防止协议相对 URL 攻击）

静态资产路由: [starlette_static_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_static_routes.py#L48-L166)

### 2.4 前端对 URL 的构建

前端通过 `DefaultStreamlitEndpoints` 类统一构建后端资源 URL：

```typescript
// 端点常量（需与后端保持同步）
const MEDIA_ENDPOINT = "/media"
const STATIC_SERVING_ENDPOINT = "/app/static/"
const UPLOAD_FILE_ENDPOINT = "/_stcore/upload_file"
const COMPONENT_ENDPOINT_BASE = "/component"
const BIDI_COMPONENT_ENDPOINT_BASE = "/_stcore/bidi-components"
```

关键逻辑：
- `buildMediaURL()`: 以 `/media` 或 `/app/static/` 开头的相对 URL 会被拼接上服务器地址
- `buildDownloadUrl()`: 支持通过 `StreamlitConfig.DOWNLOAD_ASSETS_BASE_URL` 配置 CDN 域名
- `buildComponentURL()` / `buildBidiComponentURL()`: 构建自定义组件资源 URL
- **Static Connection 模式**: 静态部署（如 S3）时，媒体 URL 会被重写为 S3 地址

前端端点实现: [DefaultStreamlitEndpoints.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts#L50-L204)
静态连接模式: [StaticConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/StaticConnection.tsx#L30-L150)

### 2.5 开发模式代理

开发模式下，Vite Dev Server 通过代理将后端请求转发到 Python 服务（默认 `localhost:8501`）：

```javascript
// vite.config.ts 代理规则
"^.*/_stcore/.*"          // 内部 API
"^(?!.*/static/media).*/media/.*"  // 媒体文件（排除 Vite 自己的 static/media）
"^.*/component/.*"        // 自定义组件
"^.*/app/static/.*"       // App 静态文件
"^.*/auth/.*"             // 认证
```

Vite 配置: [vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/app/vite.config.ts#L156-L191)

---

## 3. 访问控制

Streamlit 采用**多层防御（Swiss Cheese 模型）**的访问控制策略，每层独立提供保护，即使某层失效其他层仍能拦截攻击。

### 3.1 第一层：PathSecurityMiddleware（全局路径安全）

位置：**最外层中间件，最先执行**，对所有 HTTP 请求生效。

拦截规则（按顺序，任何一步匹配即返回 400）：

1. **UNC 路径检查** (`//`, `\\\\`): 防止 Windows UNC 路径触发 SMB 连接导致 SSRF/NTLM 哈希泄露
2. **快速路径绕过**: 已知安全的路径跳过后续检查（`/_stcore/health`, `/_stcore/script-health-check`, `/_stcore/metrics`, `/_stcore/host-config`, `/_stcore/upload_file/*`）
3. **完整路径模式检查**: 调用 `is_unsafe_path_pattern()` 检查剩余路径

路径安全中间件: [starlette_path_security_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_path_security_middleware.py#L111-L174)

### 3.2 第二层：路由处理器内的路径检查

各路由处理函数在处理文件路径前独立执行安全检查，形成纵深防御。

#### is_unsafe_path_pattern() 函数

集中化路径验证，检查以下危险模式：

| 检查项 | 示例 | 说明 |
|---|---|---|
| 空字节截断 | `file.exe\x00.jpg` | `\x00` 可被某些 API 用作字符串终止符 |
| UNC 路径 | `\\\\server\\share`, `//server/share` | Windows 网络共享，可触发 SSRF |
| Windows 盘符 | `C:\\Windows`, `D:foo` | 盘符路径可能映射到网络共享 |
| 绝对路径 | `/etc/passwd`, `\\Windows` | 以斜杠或反斜杠开头的根路径 |
| 路径穿越 | `../../../etc/passwd` | `..` 段向上跳出根目录 |

路径安全核心: [path_security.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/path_security.py#L35-L98)

#### build_safe_abspath() 函数

用于组件和 App 静态文件的安全绝对路径构建：

```python
def build_safe_abspath(component_root: str, relative_url_path: str) -> str | None:
    # 1. is_unsafe_path_pattern() 初步过滤
    if is_unsafe_path_pattern(relative_url_path):
        return None
    # 2. realpath 解析符号链接
    root_real = os.path.realpath(component_root)
    candidate = os.path.normpath(os.path.join(root_real, relative_url_path))
    candidate_real = os.path.realpath(candidate)
    # 3. commonpath 确保结果仍在根目录内
    if os.path.commonpath([root_real, candidate_real]) != root_real:
        return None
    return candidate_real
```

路径构建工具: [component_file_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/component_file_utils.py#L35-L73)

#### 静态资产路由内的安全检查

`_StreamlitStaticFiles.__call__` 在进入文件系统操作前执行检查：

```python
# 1. 双斜杠检查
if path.startswith("//"):
    return 400
# 2. 完整 unsafe pattern 检查
relative_path = path.lstrip("/")
if relative_path and is_unsafe_path_pattern(relative_path):
    return 400
```

### 3.3 CORS 跨域控制

跨域策略由 `_set_cors_headers()` 函数统一处理：

```python
def _set_cors_headers(request: Request, response: Response) -> None:
    if allow_all_cross_origin_requests():
        # 开发模式或 server.enableCORS=False → 允许所有来源 "*"
        response.headers["Access-Control-Allow-Origin"] = "*"
        return
    # 生产模式: 严格匹配 server.corsAllowedOrigins 配置
    origin = request.headers.get("Origin")
    if origin and is_allowed_origin(origin):
        response.headers["Access-Control-Allow-Origin"] = origin
```

特殊情况：
- **文件上传路由** (PUT/DELETE): 启用 XSRF 时 CORS 策略更严格，设置 `Access-Control-Allow-Credentials: true` 并 `Vary: Origin`
- **App 静态文件**: 始终返回 `Access-Control-Allow-Origin: *`（方便跨域引用）

CORS 工具: [server_util.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/server_util.py#L37-L104)
路由内 CORS: [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L179-L200)

### 3.4 XSRF 跨站请求伪造防护

采用 **Double-Submit Cookie** 模式：

1. 健康检查接口响应时设置 XSRF Cookie（`_streamlit_xsrf`）
2. Cookie 格式: `2|mask|token|timestamp`，不设 HttpOnly（JS 需读取）
3. 非安全方法（PUT/DELETE）请求时，前端从 Cookie 读取 token 放入 `X-Xsrftoken` Header
4. 后端比较 Header 值与 Cookie 值是否一致

关键属性：
- `SameSite=Lax`
- SSL 时自动加 `Secure` 标志
- `Path=/` 全站可用

XSRF 处理: [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L202-L305)
前端 XSRF 发送: [DefaultStreamlitEndpoints.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts#L382-L402)

### 3.5 其他访问控制边界

- **文件大小限制**: App 静态文件最大 200MB（`MAX_APP_STATIC_FILE_SIZE`），上传文件由 `server.maxUploadSize` 控制
- **会话校验**: 上传接口校验 `session_id` 是否为活跃会话
- **保留路由前缀**: 用户自定义路由不能以 `/_stcore/`, `/media/`, `/component/`, `/static/` 开头

配置常量: [starlette_server_config.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_server_config.py#L64-L68)

---

## 4. 浏览器缓存

Streamlit 对不同类型的静态资源采用差异化的缓存策略，在性能与新鲜度之间权衡。

### 4.1 缓存策略总览

| 资源类型 | Cache-Control 头 | 说明 |
|---|---|---|
| 核心前端带哈希资源 (JS/CSS) | `public, immutable, max-age=31536000` | 一年强缓存，文件名含内容哈希 |
| HTML / manifest.json | `no-cache` | 每次需服务器校验（304） |
| 自定义组件 HTML | `no-cache` | 组件入口文件需保持新鲜 |
| 自定义组件其他资源 | `public` | 允许缓存但无强缓存 |
| App 用户静态文件 | *(未显式设置)* | 由浏览器默认行为 + `X-Content-Type-Options: nosniff` |
| 媒体文件 (media) | *(未设置)* | 动态生成，默认不缓存 |
| 健康检查 / 主机配置 | `no-cache` | 状态接口禁用缓存 |
| 301 重定向 | `Cache-Control: no-cache` | 禁止缓存重定向 |

### 4.2 核心前端资产的缓存实现

由 `_StreamlitStaticFiles._apply_cache_headers()` 实现：

```python
_NO_CACHE_PATTERN = re.compile(r"(?:\.html$|^manifest\.json$)")

def _apply_cache_headers(self, response: Response, served_path: str) -> None:
    if response.status_code in {301, 302, 303, 304, 307, 308}:
        return  # 重定向类响应不设置缓存头

    normalized = served_path.replace("\\", "/").lstrip("./")
    cache_value = (
        "no-cache"
        if not normalized or _NO_CACHE_PATTERN.search(normalized)
        else f"public, immutable, max-age={STATIC_ASSET_CACHE_MAX_AGE_SECONDS}"
        # STATIC_ASSET_CACHE_MAX_AGE_SECONDS = 365 * 24 * 60 * 60 = 1 年
    )
    response.headers["Cache-Control"] = cache_value
```

缓存头应用: [starlette_static_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_static_routes.py#L151-L164)

构建产物配合：Vite 构建时对 JS/CSS 文件名加入内容哈希（`.[hash]`），保证文件内容变化时 URL 也变化，从而规避缓存过期问题。HTML 文件不带哈希，使用 `no-cache` 保持入口新鲜。

Vite 构建配置: [vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/app/vite.config.ts#L192-L233)

### 4.3 自定义组件缓存

- **组件入口 (index.html)**: `Cache-Control: no-cache`，确保组件更新后浏览器能立即加载新版本
- **组件其他资源 (JS/CSS/图片)**: `Cache-Control: public`，允许浏览器和中间代理缓存，但每次可能触发协商缓存

组件路由缓存: [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L786-L789)
双向组件缓存: [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L865-L868)

### 4.4 App 用户静态文件缓存

App 静态文件不设置 `Cache-Control` 头，但设置了：
- `Access-Control-Allow-Origin: *`（允许跨域引用）
- `X-Content-Type-Options: nosniff`（禁止 MIME 嗅探，提升安全性）

App 静态文件响应头: [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py#L927-L930)

### 4.5 GZip 压缩与缓存的交互

`SelectiveGZipMiddleware` 对以下情况跳过压缩：

1. **静态资产路径** (`/static/*` 或 `/`): 前端构建产物已做过最优压缩，重复压缩反而浪费 CPU
2. **音频/视频 Content-Type**: 压缩二进制媒体会破坏浏览器 Range 请求播放

压缩中间件: [starlette_gzip_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_gzip_middleware.py#L44-L144)

---

## 5. 边界划分总结

### 5.1 路径映射 vs 访问控制边界

| 维度 | 路径映射 | 访问控制 |
|---|---|---|
| 职责 | URL 到文件/资源的映射关系 | 判断请求是否允许被处理 |
| 位置 | Starlette Routes 层（内部） | Middleware 外层 + 路由处理器内部双层 |
| 失效时 | 404 Not Found | 400 Bad Request / 403 Forbidden |
| 关键机制 | `_with_base()` 前缀、`Mount` 路径、SPA fallback | `PathSecurityMiddleware`、`is_unsafe_path_pattern()`、`build_safe_abspath()`、CORS、XSRF |
| 错误含义 | 资源不存在或 URL 写错 | 攻击拦截或权限不足 |

### 5.2 访问控制 vs 浏览器缓存边界

| 维度 | 访问控制 | 浏览器缓存 |
|---|---|---|
| 作用阶段 | 请求到达时，处理前 | 响应返回时 + 后续浏览器重用 |
| 目标 | 安全：防止非法请求 | 性能：减少重复传输 |
| 相关头 | `Access-Control-Allow-*`、`Set-Cookie`、`X-Content-Type-Options` | `Cache-Control`、`ETag`、`Last-Modified` |
| 冲突情况 | 缓存的 CORS 响应可能因 Origin 不同而失效 → 用 `Vary: Origin` 解决 | |

### 5.3 路径映射 vs 浏览器缓存边界

| 维度 | 路径映射 | 浏览器缓存 |
|---|---|---|
| 核心决策 | 这个 URL 对应哪个文件 | 这个响应是否可以缓存/重用 |
| 关键设计 | 带哈希文件名（Cache Busting） | `immutable` + 长 `max-age` vs `no-cache` |
| 配合点 | URL 含内容哈希 → 可安全强缓存；HTML 无哈希 → 需 `no-cache` 保持新鲜 | |

---

## 6. 关键文件索引

| 文件 | 职责 |
|---|---|
| [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_app.py) | 应用组装、中间件顺序、路由编排 |
| [starlette_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_routes.py) | 所有业务路由定义（媒体、组件、上传、App静态等） |
| [starlette_static_routes.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_static_routes.py) | 核心前端资产路由（含SPA fallback和缓存策略） |
| [starlette_path_security_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_path_security_middleware.py) | 全局路径安全中间件 |
| [starlette_gzip_middleware.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_gzip_middleware.py) | 选择性 GZip 压缩中间件 |
| [path_security.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/path_security.py) | 路径安全检查核心算法 |
| [component_file_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/component_file_utils.py) | 安全路径构建工具 |
| [server_util.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/server_util.py) | CORS 和 XSRF 开关判断 |
| [starlette_server_config.py](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/lib/streamlit/web/server/starlette/starlette_server_config.py) | 缓存年龄、文件大小等配置常量 |
| [DefaultStreamlitEndpoints.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/DefaultStreamlitEndpoints.ts) | 前端 URL 构建与 XSRF 头注入 |
| [StaticConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/connection/src/StaticConnection.tsx) | 静态部署模式下的 S3 资源加载 |
| [vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/236-streamlit/frontend/app/vite.config.ts) | 构建产物哈希命名与开发代理配置 |
