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
│  uvicorn.Server                                                        │
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
│  │   └─ 等待 started 信号                                            │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
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
