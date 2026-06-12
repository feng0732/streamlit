# Streamlit CLI 启动流程分析

本文档详细分析 Streamlit 命令行启动流程，涵盖 **参数解析**、**脚本装载** 和 **服务启动** 三个核心阶段的调用顺序与交互关系。

---

## 一、总体调用流程图

```
streamlit run app.py [--args]
        │
        ▼
┌─────────────────────────────────┐
│  __main__.py                    │
│  main(prog_name="streamlit")    │
└───────────────┬─────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────────────────┐
│  web/cli.py                                                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 1. Click 参数解析 (main_run 命令)                       │  │
│  │    - @click.group main()                               │  │
│  │    - @main.command("run") main_run()                   │  │
│  │    - @configurator_options 动态注入配置选项            │  │
│  └───────────────────────────┬────────────────────────────┘  │
│                              │                               │
│  ┌───────────────────────────▼────────────────────────────┐  │
│  │ 2. 目标解析与验证                                       │  │
│  │    - URL 远程脚本下载 (_download_remote)                │  │
│  │    - 目录自动补全 streamlit_app.py                      │  │
│  │    - 扩展名校验 (_check_extension_or_raise)             │  │
│  └───────────────────────────┬────────────────────────────┘  │
│                              │                               │
│  ┌───────────────────────────▼────────────────────────────┐  │
│  │ 3. _main_run() 核心调度                                 │  │
│  │    - 设置 config._main_script_path                      │  │
│  │    - bootstrap.load_config_options()                    │  │
│  │    - check_credentials()                                │  │
│  │    - discover_asgi_app() → 分支决策                     │  │
│  └───────────────────────────┬────────────────────────────┘  │
└──────────────────────────────┼───────────────────────────────┘
                               │
          ┌────────────────────┴────────────────────┐
          │                                         │
          ▼                                         ▼
┌─────────────────────────┐             ┌───────────────────────────────┐
│  ASGI 模式 (st.App)     │             │  传统 Streamlit 模式          │
│  bootstrap.run_asgi_app()│             │  bootstrap.run()              │
└────────────┬────────────┘             └───────────────┬───────────────┘
             │                                           │
             ▼                                           ▼
┌─────────────────────────┐             ┌───────────────────────────────┐
│  UvicornRunner.run()    │             │  Server(main_script_path)     │
│  (同步阻塞)             │             │  __init__ 创建 Runtime        │
└────────────┬────────────┘             └───────────────┬───────────────┘
             │                                           │
             │                                           ▼
             │                              ┌───────────────────────────────┐
             │                              │  Server.start()                │
             │                              │  └─ UvicornServer.start()     │
             │                              │     └─ asyncio 后台任务        │
             │                              └───────────────┬───────────────┘
             │                                            │
             ▼                                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  uvicorn.Server (Socket 监听中, 等待浏览器连接)                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ starlette_app.create_starlette_app(runtime)                      │   │
│  │   ├─ create_streamlit_routes()  路由注册                         │   │
│  │   ├─ create_streamlit_middleware()  中间件                       │   │
│  │   └─ lifespan → runtime.start()  Runtime 启动                    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ Runtime.start()                                                   │   │
│  │   ├─ 创建 AsyncObjects (Event/Future)                            │   │
│  │   ├─ 启动 _loop_coroutine_task 协程                              │   │
│  │   └─ 等待 started 信号 → NO_SESSIONS_CONNECTED 状态                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────┬───────────────────────────┘
                                              │
                        ┌─────────────────────┴─────────────────────┐
                        │                                           │
                        ▼                                           │
              [浏览器] 打开 http://localhost:8501                   │
                        │                                           │
                        ▼                                           │
              HTTP GET / → 返回 index.html + JS 资源                 │
                        │                                           │
                        ▼                                           │
              WebSocket 握手 /_stcore/stream                        │
                        │                                           │
                        ▼                                           │
              _websocket_endpoint()                                  │
                ├─ Origin 验证                                       │
                ├─ StarletteSessionClient (send 队列 + sender Task)  │
                ├─ runtime.connect_session() → AppSession 创建      │
                └─ has_connection.set() → 唤醒 Runtime 主循环        │
                        │                                           │
                        ▼                                           │
              前端 handleConnectionStateChanged(CONNECTED)           │
                └─ sendUpdateWidgetsMessage(undefined)               │
                   └─ sendRerunBackMsg() → BackMsg(rerun_script)   │
                        │                                           │
                        ▼                                           │
              后端 receive_bytes() → ParseFromString → handle_backmsg│
                └─ AppSession.request_rerun()                       │
                   └─ _create_scriptrunner() → start() 新线程       │
                        │                                           │
                        ▼                                           │
              ScriptRunner.scriptThread (独立线程!)                  │
                ├─ 创建 ScriptRunContext (threading.local)          │
                ├─ _run_script()                                     │
                │  ├─ 发送 SCRIPT_STARTED → NewSession              │
                │  ├─ 编译并 exec(bytecode, module.__dict__)        │
                │  └─ 逐行执行用户脚本 → 发送 DeltaMsg              │
                └─ 发送 SCRIPT_STOPPED_WITH_SUCCESS                 │
```

---

## 二、阶段一：参数解析

### 2.1 入口文件

**文件**: [lib/streamlit/__main__.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/__main__.py#L15-L20)

```python
from streamlit.web.cli import main

if __name__ == "__main__":
    main(prog_name="streamlit")
```

入口非常简洁，直接导入并调用 `web.cli.main()`。`prog_name="streamlit"` 确保无论是直接执行 `streamlit` 还是 `python -m streamlit`，命令行字符串都保持一致。

### 2.2 Click 命令组

**文件**: [lib/streamlit/web/cli.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/cli.py#L130-L151)

```python
@click.group(context_settings={"auto_envvar_prefix": "STREAMLIT"})
@click.option("--log_level", show_default=True, type=click.Choice(LOG_LEVELS))
@click.version_option(prog_name="Streamlit")
def main(log_level: str = "info") -> None:
    ...
```

使用 `click.group` 构建命令组，自动支持 `STREAMLIT_` 前缀的环境变量。

### 2.3 `run` 子命令与动态配置注入

**文件**: [lib/streamlit/web/cli.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/cli.py#L198-L251)

```python
@main.command("run")
@configurator_options          # <-- 动态注入所有配置项
@click.argument("target", default="streamlit_app.py", envvar="STREAMLIT_RUN_TARGET")
@click.argument("args", nargs=-1)
def main_run(target: str, args: list[str] | None = None, **kwargs: Any) -> None:
    ...
```

#### `@configurator_options` 装饰器

**文件**: [lib/streamlit/web/cli.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/cli.py#L86-L113)

这是参数解析的核心机制：

1. 遍历 `_config._config_options_template` 中的所有配置项
2. 通过 `_convert_config_option_to_click_option()` 将配置项转为 Click 选项
3. 敏感配置项（如密码）使用 `_make_sensitive_option_callback` 禁止通过 CLI 设置
4. 非敏感配置项自动暴露环境变量（`show_envvar=True`）

这样，所有 `streamlit config show` 中列出的配置项都可以通过 CLI 标志传递，例如：
```bash
streamlit run app.py --server.port 8502 --server.headless true
```

### 2.4 目标解析分支

`main_run()` 根据 `target` 参数分三条路径处理：

| 目标类型 | 处理逻辑 | 关键函数 |
|---------|---------|---------|
| **URL** (http/https) | 下载到临时目录后执行 | `_download_remote()` |
| **目录** | 自动拼接 `streamlit_app.py` | `path /= "streamlit_app.py"` |
| **文件** | 直接使用 | `_check_extension_or_raise()` |

---

## 三、阶段二：脚本装载与分支决策

### 3.1 `_main_run()` 调度中心

**文件**: [lib/streamlit/web/cli.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/cli.py#L287-L325)

执行顺序：

```
1. main_script_path = os.path.abspath(file)
2. _config._main_script_path = main_script_path   # 全局配置锚点
3. bootstrap.load_config_options(flag_options)     # 加载配置文件 + CLI 覆盖
4. check_credentials()                              # 验证激活凭据
5. discover_asgi_app(Path(main_script_path))        # AST 扫描，分支决策
   ├─ is_asgi_app=True  → bootstrap.run_asgi_app()
   └─ is_asgi_app=False → bootstrap.run()
```

### 3.2 `load_config_options()` 配置合并

**文件**: [lib/streamlit/web/bootstrap.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/bootstrap.py#L268-L294)

```python
def load_config_options(flag_options: dict[str, Any]) -> None:
    options_from_flags = {
        name.replace("_", "."): val
        for name, val in flag_options.items()
        if val is not None and val != ()
    }
    config.get_config_options(force_reparse=True, options_from_flags=options_from_flags)
```

优先级链（从低到高）：
```
默认值 → ~/.streamlit/config.toml → 项目级 config.toml → CLI flags
```

### 3.3 ASGI App 发现机制 (AST 静态分析)

**文件**: [lib/streamlit/web/server/app_discovery.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/app_discovery.py#L316-L422)

这是脚本装载的关键步骤，**无需执行代码**即可判断脚本类型：

```
discover_asgi_app(path)
    │
    ├─ 读取源文件文本
    │
    ├─ _find_asgi_app_assignments(source)
    │   ├─ ast.parse(source)  解析 AST
    │   ├─ _extract_imports(tree)  建立 {local_name: full_module_path} 映射
    │   │   └─ 处理 import, from...import, as 别名
    │   │
    │   └─ 遍历所有 AST 节点
    │       ├─ ast.Assign / ast.AnnAssign
    │       └─ _is_asgi_app_call(node.value, imports)
    │           ├─ _get_call_name_parts() 提取调用链 (st.App → ("st","App"))
    │           └─ _resolve_call_to_module_path() 解析为完整路径
    │               └─ 匹配 _KNOWN_ASGI_APP_CLASSES:
    │                   ├─ streamlit.App
    │                   ├─ fastapi.FastAPI
    │                   └─ starlette.applications.Starlette
    │
    └─ 优先选择 "app" / "streamlit_app" 变量名
        └─ 返回 import_string (如 "myapp:app")
```

**设计考量**: 使用 AST 而非 `import` 执行，避免用户脚本的副作用（如自动启动服务），同时防止恶意代码在检测阶段执行。

---

## 四、阶段三：服务启动（两条路径）

### 路径 A：传统 Streamlit 模式 (`bootstrap.run()`)

**文件**: [lib/streamlit/web/bootstrap.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/bootstrap.py#L362-L428)

#### Step 1: 环境准备

```python
_fix_sys_path(main_script_path)       # 脚本目录加入 sys.path
_fix_sys_argv(main_script_path, args) # 重写 sys.argv 为用户脚本视角
_install_config_watchers(flag_options)# 监听 config.toml 变更
config._server_mode = "starlette-managed"
```

#### Step 2: 创建 Server 实例

**文件**: [lib/streamlit/web/server/server.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/server.py#L51-L75)

```python
server = Server(main_script_path, is_hello)
# __init__ 内部:
#   - 创建 MemoryMediaFileStorage
#   - 创建 MemoryUploadedFileManager
#   - 创建 Runtime(RuntimeConfig(...))  <-- Runtime 实例化但未启动
```

#### Step 3: 异步启动

```python
async def run_server():
    await server.start()           # 启动服务
    _on_server_start(server)       # 打印URL、打开浏览器
    _set_up_signal_handler(server) # 注册 SIGINT/SIGTERM
    await server.stopped           # 阻塞等待退出

# 事件循环处理
if running_in_event_loop:
    asyncio.create_task(main())    # 已有循环（如测试环境），创建任务
else:
    _maybe_install_uvloop(...)     # 非 Windows 尝试安装 uvloop
    asyncio.run(main())            # 标准 CLI 路径
```

#### Step 4: `Server.start()` → `UvicornServer.start()`

**文件**: [lib/streamlit/web/server/server.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/server.py#L84-L94)
**文件**: [lib/streamlit/web/server/starlette/starlette_server.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/starlette/starlette_server.py#L291-L485)

```
Server.start()
  └─ UvicornServer(self._runtime).start()
      ├─ create_starlette_app(self._runtime)  # 构建 Starlette 应用
      ├─ 端口绑定循环 (最多 MAX_PORT_SEARCH_RETRIES 次)
      │   ├─ _bind_server_socket() 预绑定 socket
      │   │   └─ 优先尝试 IPv6 双栈 "::"，失败回退 IPv4
      │   ├─ port=0 时读取 OS 分配的实际端口
      │   └─ uvicorn.Config(app, host, port, ...)
      │
      ├─ uvicorn.Server(uvicorn_config)
      ├─ serve_with_signal() 作为后台 Task 运行
      │   ├─ server_config.load() + lifespan_class 初始化
      │   ├─ await server.startup(sockets=[socket])  # <-- 触发 lifespan
      │   └─ await server.main_loop()
      │
      └─ await startup_complete.wait()  # 返回给调用者
```

#### Step 5: Starlette Lifespan → Runtime 启动

**文件**: [lib/streamlit/web/server/starlette/starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L208-L245)

```python
@asynccontextmanager
async def _lifespan(_app: Starlette):
    _set_anyio_thread_limiter()
    await runtime.start()   # <-- Runtime 真正启动
    yield
    runtime.stop()          # <-- 关闭时调用
```

#### Step 6: `Runtime.start()`

**文件**: [lib/streamlit/runtime/runtime.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/runtime.py#L317-L344)

```python
async def start(self):
    async_objs = AsyncObjects(
        eventloop=asyncio.get_running_loop(),
        must_stop=asyncio.Event(),
        has_connection=asyncio.Event(),
        need_send_data=asyncio.Event(),
        started=asyncio.Future(),
        stopped=asyncio.Future(),
    )
    self._async_objs = async_objs

    # 启动主循环协程（处理 session 消息、缓存等）
    self._loop_coroutine_task = asyncio.create_task(
        self._loop_coroutine(), name="Runtime.loop_coroutine"
    )

    await async_objs.started  # 等待 _loop_coroutine 标记就绪
```

启动完成后，`_on_server_start()` 被调用：
- `prepare_streamlit_environment()` 初始化 MIME 类型、PyDeck API Key、secrets.toml
- `_print_url()` 打印访问地址
- `report_watchdog_availability()` 报告文件监听能力
- `maybe_open_browser()` 根据配置自动打开浏览器

---

### 路径 B：ASGI 模式 (`bootstrap.run_asgi_app()`)

**文件**: [lib/streamlit/web/bootstrap.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/bootstrap.py#L315-L359)

```python
def run_asgi_app(main_script_path, app_import_string, args, flag_options):
    _fix_sys_path(main_script_path)
    _fix_sys_argv(main_script_path, args)
    _install_config_watchers(flag_options)
    config._server_mode = "starlette-app"
    report_watchdog_availability()

    runner = UvicornRunner(app_import_string)  # "module:app" 格式
    runner.run()  # 同步阻塞直到退出
```

#### `UvicornRunner.run()`

**文件**: [lib/streamlit/web/server/starlette/starlette_server.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/starlette/starlette_server.py#L488-L615)

与 `UvicornServer` 的关键区别：

| 特性 | UvicornServer (传统) | UvicornRunner (ASGI) |
|------|---------------------|---------------------|
| 调用方式 | `await start()` 异步 | `run()` 同步阻塞 |
| 事件循环 | 嵌入已有 loop | `server.run()` 内部自建 |
| 信号处理 | 外部 `_set_up_signal_handler` | uvicorn 内部处理 |
| App 参数 | Starlette 实例 | import 字符串 `"module:app"` |
| Runtime 启动 | `create_starlette_app` 的 lifespan | `st.App._combined_lifespan` |

当 uvicorn 导入并实例化 `st.App` 时，流程进入 [starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/starlette/starlette_app.py#L387-L689) 的 `App.__init__()` → `_build_starlette_app()` → `_combined_lifespan()` → `Runtime.start()`。

---

## 四、浏览器建连与脚本执行时序

服务启动完成后，用户在浏览器中打开页面并不会立即执行脚本。脚本执行是由浏览器的 WebSocket 连接和前端主动发送的 rerun 请求驱动的。本章节详细梳理从浏览器建连到脚本首次执行、以及后续重跑的完整时序。

### 4.1 完整时序图

```
服务启动 (Runtime.start())
        │
        ▼  Runtime 进入 NO_SESSIONS_CONNECTED 状态
        ▼  阻塞等待 has_connection Event
        │
        │                        [浏览器]
        │                          │
        │                          ▼  打开 http://localhost:8501
        │                          │
        │  HTTP GET /              │
        │◀─────────────────────────┤  返回 index.html
        │                          │
        │                          ▼  加载前端 JS 资源
        │                          │
        │  WebSocket 握手 /_stcore/stream
        │◀─────────────────────────┤
        │  Sec-WebSocket-Protocol: streamlit
        │  Origin: http://localhost:8501
        │                          │
        ▼  _is_origin_allowed() 验证通过
        ▼  _parse_subprotocols() 提取 xsrf_token / existing_session_id
        ▼  await websocket.accept()
        ▼  StarletteSessionClient 实例化 (带 send 队列 + 后台 sender Task)
        ▼  user_info 从 cookies/trusted headers 解析
        ▼  runtime.connect_session() 被调用
        │                          │
┌───────┴──────────────────────────────────────────┐
│  Step 1: 会话建立 (connect_session)              │
│──────────────────────────────────────────────────│
│  位置: runtime.py connect_session()              │
│        websocket_session_manager.py              │
│                                                  │
│  runtime.connect_session(                        │
│    client=StarletteSessionClient,                │
│    user_info=user_info,                          │
│    existing_session_id=...                       │
│  )                                               │
│    │                                             │
│    ├─ 尝试从 SessionStorage 恢复已有会话          │
│    │  (仅在断线重连场景)                         │
│    │                                             │
│    └─ 新建 AppSession                             │
│        ├─ 生成唯一 session_id (UUID)              │
│        ├─ ForwardMsgQueue (浏览器消息队列)        │
│        ├─ PagesManager (页面管理)                 │
│        ├─ SessionState (会话状态)                │
│        ├─ FragmentStorage (片段缓存)             │
│        ├─ LocalSourcesWatcher (文件监听)         │
│        └─ BackendOperationDispatcher (文件请求)  │
│                                                  │
│  _session_mgr._active_session_info_by_id[...]     │
│    = ActiveSessionInfo(client, session)          │
│                                                  │
│  Runtime 状态:                                    │
│  NO_SESSIONS_CONNECTED → ONE_OR_MORE_SESSIONS    │
│  has_connection.set()                            │
│    │                                             │
│    ▼  Runtime._loop_coroutine() 被唤醒            │
│       开始轮询 flush_browser_queue()              │
└──────────────────────────────────────────────────┘
        │                          │
        │                          ▼  前端: handleConnectionStateChanged(CONNECTED)
        │                          │
        │                          ▼  满足首次连接条件
        │                          │
┌───────┴──────────────────────────────────────────┐
│  Step 2: 前端触发首次 rerun 请求                 │
│──────────────────────────────────────────────────│
│  位置: frontend/app/src/App.tsx                  │
│        handleConnectionStateChanged() [L862-L898]│
│                                                  │
│  触发条件 (满足任一):                            │
│  1. !sessionInfo.last         (首次连接)         │
│  2. lastRunWasInterrupted    (上次被中断)        │
│  3. wasRerunRequested        (重跑被请求)        │
│  4. 使用了 fragments / auto-reruns                │
│                                                  │
│  this.widgetMgr.sendUpdateWidgetsMessage(undefined)
│    │                                             │
│    ▼  WidgetStateManager.sendUpdateWidgetsMessage │
│       [frontend/lib/src/WidgetStateManager.ts L831]
│       this.props.sendRerunBackMsg(               │
│         createWidgetStatesMsg(),                 │
│         fragmentId=undefined,                    │
│         isAutoRerun=undefined                    │
│       )                                          │
│    │                                             │
│    ▼  App.sendRerunBackMsg() [L1917]             │
│       构建 BackMsg:                              │
│       ├─ rerunScript: {                          │
│       │   queryString: "...",                    │
│       │   widgetStates: {},                      │
│       │   pageScriptHash: "",                    │
│       │   pageName: "",                          │
│       │   fragmentId: "",                        │
│       │   cachedMessageHashes: [],               │
│       │   contextInfo: {                         │
│       │     timezone, locale, url,               │
│       │     isEmbedded, colorScheme              │
│       │   }                                      │
│       └─ }                                       │
│                                                  │
│  connectionManager.sendMessage(BackMsg)          │
│  WebSocket 二进制帧发送                          │
└──────────────────────────────────────────────────┘
        │                          │
        ▼  _websocket_endpoint: websocket.receive_bytes()
        │  BackMsg.ParseFromString(data)
        │  msg_type = "rerun_script"
        │
┌───────┴──────────────────────────────────────────┐
│  Step 3: 后端路由到 AppSession                   │
│──────────────────────────────────────────────────│
│  位置: starlette_websocket.py [L510]             │
│        runtime.handle_backmsg()                  │
│                                                  │
│  runtime.handle_backmsg(session_id, back_msg)    │
│    │                                             │
│    ├─ 查找 session_info =                         │
│    │    _session_mgr.get_active_session_info(id) │
│    │                                             │
│    └─ session_info.session.handle_backmsg(msg)   │
│                                                  │
│  AppSession.handle_backmsg()                     │
│  [app_session.py L338-L371]                      │
│    │                                             │
│    ├─ msg_type = "rerun_script"                  │
│    └─ _handle_rerun_script_request(msg.rerun_script)
│       → this.request_rerun(client_state)         │
└──────────────────────────────────────────────────┘
        │
        ▼
┌───────┴──────────────────────────────────────────┐
│  Step 4: request_rerun 与 ScriptRunner 创建      │
│──────────────────────────────────────────────────│
│  位置: app_session.py [L406-L488]                │
│                                                  │
│  AppSession.request_rerun(client_state):         │
│                                                  │
│  1. 解析 RerunData:                              │
│     ├─ query_string                               │
│     ├─ widget_states (表单值、slider 值等)       │
│     ├─ page_script_hash / page_name              │
│     ├─ fragment_id (片段运行时)                  │
│     ├─ cached_message_hashes (消息去重)          │
│     └─ context_info (客户端环境)                 │
│                                                  │
│  2. 已有 ScriptRunner 时的处理:                   │
│     ├─ fastReruns=True 且非片段运行:             │
│     │   → request_stop() 终止当前运行             │
│     │   → self._scriptrunner = None              │
│     │                                             │
│     └─ 否则:                                      │
│        → _scriptrunner.request_rerun(rerun_data) │
│        → 成功则直接返回                           │
│                                                  │
│  3. 无 ScriptRunner 或 fastReruns 时:             │
│     → _create_scriptrunner(rerun_data)           │
│                                                  │
└──────────────────────────────────────────────────┘
        │
        ▼
┌───────┴──────────────────────────────────────────┐
│  Step 5: ScriptRunner 线程启动                   │
│──────────────────────────────────────────────────│
│  位置: app_session.py [L502-L518]                │
│        script_runner.py [L333-L346]              │
│                                                  │
│  _create_scriptrunner(initial_rerun_data):       │
│                                                  │
│  1. 实例化 ScriptRunner:                          │
│     ├─ session_id                                │
│     ├─ main_script_path                          │
│     ├─ SafeSessionState (带 yield 回调)          │
│     ├─ ScriptRequests (请求队列)                 │
│     │   └─ 预存 initial_rerun_data               │
│     ├─ on_event Signal (事件回调)                │
│     └─ 关联 PagesManager、FragmentStorage 等      │
│                                                  │
│  2. 事件绑定:                                     │
│     _scriptrunner.on_event.connect(              │
│       _on_scriptrunner_event                     │
│     )                                            │
│                                                  │
│  3. 启动线程:                                    │
│     _scriptrunner.start()                        │
│       → threading.Thread(                        │
│           target=_run_script_thread,             │
│           name="ScriptRunner.scriptThread"       │
│         ).start()                                │
│                                                  │
└──────────────────────────────────────────────────┘
        │
        ▼  ScriptRunner.scriptThread (独立线程!)
        │
┌───────┴──────────────────────────────────────────┐
│  Step 6: 脚本线程主循环                          │
│──────────────────────────────────────────────────│
│  位置: script_runner.py [L378-L435]              │
│                                                  │
│  _run_script_thread():                           │
│                                                  │
│  1. 创建 ScriptRunContext (线程本地存储):         │
│     ├─ session_id                                │
│     ├─ _enqueue 回调 (发送 ForwardMsg)           │
│     ├─ script_requests (请求队列)                │
│     ├─ session_state                             │
│     ├─ uploaded_file_mgr                         │
│     ├─ user_info                                 │
│     └─ pages_manager                             │
│                                                  │
│  2. 绑定上下文到线程:                            │
│     add_script_run_ctx(thread, ctx)              │
│     (通过 threading.local 存储)                  │
│                                                  │
│  3. 主循环:                                      │
│     request = _requests.on_scriptrunner_ready()  │
│     → 取出预存的 RERUN 请求                      │
│                                                  │
│     while request.type == RERUN:                 │
│         _run_script(request.rerun_data)          │
│         request = _requests.on_scriptrunner_ready()
│                                                  │
│  4. SHUTDOWN 时发送保存 client_state              │
└──────────────────────────────────────────────────┘
        │
        ▼
┌───────┴──────────────────────────────────────────┐
│  Step 7: _run_script 实际执行用户代码             │
│──────────────────────────────────────────────────│
│  位置: script_runner.py                          │
│                                                  │
│  _run_script(rerun_data):                        │
│                                                  │
│  1. 发送 SCRIPT_STARTED 事件:                    │
│     on_event.send(SCRIPT_STARTED, ...)           │
│     → _on_scriptrunner_event                     │
│       → call_soon_threadsafe 调度到事件循环线程   │
│         → _handle_scriptrunner_event_on_event_loop
│           → 发送 NewSession ForwardMsg           │
│           → 包含 config, theme, pages 等元数据   │
│           → 浏览器显示 "Running..." 状态         │
│                                                  │
│  2. 准备执行环境:                                │
│     ├─ _set_execing_flag(True)                   │
│     ├─ _install_tracer() (覆盖 sys.settrace)     │
│     ├─ local_sources_watcher.on_script_run()     │
│     └─ pages_manager.reset()                     │
│                                                  │
│  3. 编译并执行:                                  │
│     ├─ get_command_line_from_page_name()         │
│     ├─ _script_cache.load_bytecode(script_path)  │
│     │  (带源代码哈希的缓存机制)                   │
│     ├─ module = _new_module()                    │
│     │  (__file__, __name__, __dict__)            │
│     ├─ sys.modules[module_name] = module         │
│     └─ exec(bytecode, module.__dict__)           │
│        → 逐行执行用户脚本                        │
│        → 遇到 st.write/st.button 等时调用组件API │
│        → 组件通过 ScriptRunContext 发送 DeltaMsg │
│        → Delta 入队 ForwardMsgQueue              │
│                                                  │
│  4. Runtime 刷新循环:                            │
│     _loop_coroutine 每 MESSAGE_FLUSH_INTERVAL_SECS
│     调用 session.flush_browser_queue()            │
│     → 取出 ForwardMsg → client.write_forward_msg()
│     → StarletteSessionClient.send_queue → websocket
│     → 浏览器实时渲染 Delta 变化                   │
│                                                  │
│  5. 执行完成:                                    │
│     ├─ 发送 SCRIPT_STOPPED_WITH_SUCCESS 事件     │
│     ├─ on_scriptrunner_event 发送 script_finished
│     ├─ 浏览器显示 "Running..." → 完成状态         │
│     └─ 更新 LocalSourcesWatcher 监听列表          │
└──────────────────────────────────────────────────┘
```

### 4.2 关键时间节点详解

#### 节点 A: Runtime 启动后但浏览器未连接

**文件**: [runtime.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/runtime.py#L624-L668)

```python
# Runtime._loop_coroutine() 中的 NO_SESSIONS_CONNECTED 分支
while not async_objs.must_stop.is_set():
    if self._state == RuntimeState.NO_SESSIONS_CONNECTED:
        # 阻塞等待:
        #   - must_stop (进程退出信号)
        #   - has_connection (有浏览器连接)
        done_tasks, pending_tasks = await asyncio.wait(
            (
                asyncio.create_task(async_objs.must_stop.wait()),
                asyncio.create_task(async_objs.has_connection.wait()),
            ),
            return_when=asyncio.FIRST_COMPLETED,
            timeout=_SIGNAL_CHECK_INTERVAL,  # Windows 信号处理
        )
```

**注意**: 此时用户脚本完全没有被加载或执行。

#### 节点 B: connect_session 完成

**文件**: [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L100-L168)

```python
session = AppSession(
    script_data=ScriptData(main_script_path, is_hello),
    uploaded_file_manager=...,
    script_cache=...,
    message_enqueued_callback=_enqueued_some_message,
    user_info=user_info,
)
self._active_session_info_by_id[session.id] = ActiveSessionInfo(client, session)
```

此时 AppSession 创建完成，但仍然**没有**执行任何用户代码。ScriptRunner 尚未创建。

#### 节点 C: 前端主动发送 rerun_script

**文件**: [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/frontend/app/src/App.tsx#L862-L898)

```typescript
if (newState === ConnectionState.CONNECTED) {
  if (!this.sessionInfo.last ||   // 首次连接
      lastRunWasInterrupted ||
      wasRerunRequested ||
      fragmentIdsThisRun.length > 0) {
    this.widgetMgr.sendUpdateWidgetsMessage(undefined)
    // 触发 sendRerunBackMsg → WebSocket 发送 BackMsg
  }
}
```

这是**整个流程的关键触发点**：脚本执行不是由后端主动发起，而是由前端在连接建立后主动请求的。

#### 节点 D: ScriptRunner 线程启动

**文件**: [script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L333-L346)

```python
def start(self) -> None:
    self._script_thread = threading.Thread(
        target=self._run_script_thread,
        name="ScriptRunner.scriptThread",
    )
    self._script_thread.start()
```

**关键设计考量**:
- 用户脚本在**独立线程**中执行，而非事件循环线程
- 这避免了脚本执行阻塞 WebSocket 连接和其他会话
- 通过 `call_soon_threadsafe` 实现跨线程通信

---

### 4.3 脚本重跑机制 (Rerun)

脚本在以下场景会被重新执行（均有直接代码证据）：

| 触发源 | 触发路径 | 关键代码 |
|--------|---------|---------|
| **用户交互** (按钮/滑块/复选框等) | 前端 WidgetStateManager 状态变化 → sendUpdateWidgetsMessage() → sendRerunBackMsg() → WebSocket BackMsg → handle_backmsg → request_rerun | [WidgetStateManager.ts L831-L841](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/frontend/lib/src/WidgetStateManager.ts#L831-L841) |
| **st.rerun()** | 调用 `ctx.script_requests.request_rerun()` 入队 → 通过 `st.empty()` 强制触发 yield point → `_maybe_handle_execution_control_request` 取出请求 → 抛出 `RerunException` 中断当前执行 → 下一轮循环取出 RERUN 重新执行 | [execution_control.py L140-L192](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/commands/execution_control.py#L140-L192) |
| **文件变更** | LocalSourcesWatcher 检测到源文件/依赖文件变化 → `_on_source_file_changed()` → `request_rerun()` 入队 | [app_session.py L538-L548](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/app_session.py#L538-L548) |
| **页面切换** (`st.switch_page`) | 设置新的 page_script_hash 和 page_name → `request_rerun()` 入队 → yield point 触发中断重跑 | [execution_control.py L194-L260](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/commands/execution_control.py#L194-L260) |
| **手动 Rerun 按钮** | 前端 MainMenu → rerunScript → sendRerunBackMsg() → WebSocket BackMsg | [MainMenu.tsx](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/frontend/app/src/components/MainMenu/MainMenu.tsx) |
| **Fragment 自动重跑** | `@st.fragment(rerun="auto")` → 前端定时发送 fragment rerun 请求 | [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/frontend/app/src/App.tsx) |

> **注意**: `st.cache_data` / `st.cache_resource` 的 TTL 过期不会主动触发脚本重跑。缓存过期是惰性检查的（访问时判断），仅影响缓存命中与否，不参与 rerun 调度。

#### 快重跑 (Fast Reruns) 优化

**文件**: [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/app_session.py#L466-L483)

```python
if self._scriptrunner is not None:
    if (
        bool(config.get_option("runner.fastReruns"))
        and not rerun_data.fragment_id
    ):
        # 终止当前 ScriptRunner，创建新的
        self._scriptrunner.request_stop()
        self._scriptrunner = None
        # 下方创建新 ScriptRunner 立即启动
```

**工作原理**:
- 当 `runner.fastReruns=true` 时，新 rerun 请求会**终止当前正在运行**的脚本线程
- 创建全新的 ScriptRunner 立即执行，减少等待时间
- 对 Fragment 运行无效（Fragment 需要维持上下文）

---

### 4.4 多线程模型与线程安全

Streamlit 采用明确的多线程分工：

| 线程 | 职责 | 执行的代码 |
|------|------|-----------|
| **事件循环线程** (EventLoop Thread) | WebSocket 消息收发、会话管理、消息分发 | Runtime._loop_coroutine(), AppSession 事件处理 |
| **ScriptRunner 线程** (每个会话) | 执行用户脚本代码 | 用户编写的 `app.py`、所有 `st.*` 组件调用 |
| **StarletteSender 线程** (每个连接) | WebSocket 异步发送 | 后台 Task 发送 ForwardMsg |

**线程安全机制**:

1. **ScriptRunContext 线程本地存储**:
   ```python
   # script_run_context.py
   _SCRIPT_RUN_CONTEXT = threading.local()
   ```

2. **跨线程通信使用 call_soon_threadsafe**:
   ```python
   # ScriptRunner 线程 → 事件循环线程
   self._event_loop.call_soon_threadsafe(
       lambda: self._handle_scriptrunner_event_on_event_loop(...)
   )
   ```
   所有 ForwardMsg 的入队和 flush 操作都通过 `call_soon_threadsafe` 调度到事件循环线程执行，确保单线程访问。

3. **ForwardMsgQueue 是非线程安全的**:
   官方文档明确说明 [forward_msg_queue.py#L32-L33](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/forward_msg_queue.py#L32-L33)
   "ForwardMsgQueue is not thread-safe - a queue should only be used from a single thread."
   但在实际使用中，它只在事件循环线程上被操作（enqueue 通过 call_soon_threadsafe 投递，flush 在 Runtime 主循环中），因此是安全的。

4. **SessionState 包装为 SafeSessionState**:
   每次访问时调用 `_yield_callback` 检查是否有挂起的 STOP 或 RERUN 请求，实现协作式中断。

---

### 4.5 会话关闭与资源清理

**路径**: 浏览器关闭标签 → WebSocket disconnect → 会话清理

```
浏览器关闭 → WebSocketDisconnect
        │
        ▼
runtime.disconnect_session(session_id)
        │
        ▼
session.request_script_stop()       # 终止脚本线程
session.disconnect_file_watchers()  # 停止文件监听
session.clear_session_caches()      # 清理 session 级缓存
        │
        ▼
保存到 SessionStorage (用于重连)
        │
        ▼
config.ttl 时间后被彻底清理
        │
        ▼
session.shutdown()                  # 彻底清理所有资源
```

**文件**: [websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/websocket_session_manager.py#L170-L193)

---

## 五、关键交互点总结

### 5.1 参数解析 → 脚本装载

```
configurator_options (装饰器阶段)
    ↓ 收集所有 CLI flags 到 kwargs
main_run(target, args, **kwargs)
    ↓
_main_run(file, args, flag_options=kwargs)
    ├─ _config._main_script_path = file   # 供后续 config/secret 发现使用
    └─ bootstrap.load_config_options(flag_options)  # CLI flags → 全局 config
```

### 5.2 脚本装载 → 服务启动

```
discover_asgi_app(path)
    ↓ AppDiscoveryResult
    ├─ is_asgi_app=True
    │   └─ run_asgi_app() → UvicornRunner.run()
    │       └─ uvicorn 导入脚本 → st.App.__call__ → Runtime.start()
    │
    └─ is_asgi_app=False
        └─ run() → Server.__init__(Runtime) → Server.start()
            └─ UvicornServer.start() → Starlette lifespan → Runtime.start()
```

### 5.3 两条路径的汇合点

无论走哪条路径，最终都会到达：

1. **Starlette 应用构建**: `create_starlette_app(runtime)` 或 `App._build_starlette_app()`
2. **Runtime 启动**: `Runtime.start()` 创建事件循环同步原语并启动主协程
3. **Socket 监听**: uvicorn 在预绑定的 socket 上接受 HTTP 和 WebSocket 连接

---

## 六、文件索引

| 文件 | 角色 |
|------|------|
| [lib/streamlit/__main__.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/__main__.py) | CLI 入口 |
| [lib/streamlit/web/cli.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/cli.py) | Click 命令定义、参数解析、分支调度 |
| [lib/streamlit/web/bootstrap.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/bootstrap.py) | 配置加载、环境准备、服务启动入口 |
| [lib/streamlit/web/server/app_discovery.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/app_discovery.py) | AST 静态分析检测 ASGI App |
| [lib/streamlit/web/server/server.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/server.py) | Server 包装类，组合 Runtime + UvicornServer |
| [lib/streamlit/web/server/starlette/starlette_server.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/starlette/starlette_server.py) | UvicornServer/UvicornRunner 端口绑定与启动 |
| [lib/streamlit/web/server/starlette/starlette_app.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/starlette/starlette_app.py) | Starlette 应用构建、st.App ASGI 兼容类 |
| [lib/streamlit/runtime/runtime.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/runtime.py) | Runtime 核心：会话管理、消息循环 |
| [lib/streamlit/runtime/websocket_session_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/websocket_session_manager.py) | 基于 WebSocket 的会话生命周期管理 |
| [lib/streamlit/runtime/app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/app_session.py) | 单个会话的状态管理、Rerun 调度 |
| [lib/streamlit/runtime/scriptrunner/script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py) | 脚本线程创建与执行、Rerun 队列处理、Yield Point |
| [lib/streamlit/runtime/forward_msg_queue.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/forward_msg_queue.py) | ForwardMsg 消息队列、Delta 消息合并优化 |
| [lib/streamlit/runtime/state/safe_session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/runtime/state/safe_session_state.py) | SessionState 线程安全包装、yield_callback 机制 |
| [lib/streamlit/commands/execution_control.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/commands/execution_control.py) | st.rerun()、st.switch_page() 等执行控制命令 |
| [lib/streamlit/web/server/starlette/starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py) | WebSocket 连接处理、BackMsg 路由、Origin 验证 |
| [frontend/app/src/App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/frontend/app/src/App.tsx) | 前端主组件：连接状态管理、Rerun 请求触发 |
| [frontend/lib/src/WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/frontend/lib/src/WidgetStateManager.ts) | Widget 状态管理、更新消息批量发送 |
| [frontend/connection/src/ConnectionManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/233-streamlit/frontend/connection/src/ConnectionManager.ts) | WebSocket 连接管理、重连逻辑 |
