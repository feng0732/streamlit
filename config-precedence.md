# Streamlit 配置加载优先级机制详解

## 总览

Streamlit 配置系统采用 **7 层覆盖**策略：高优先级来源的值会覆盖低优先级来源的值。整个加载流程由 `get_config_options()` 函数统一调度，定义在 [lib/streamlit/config.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2751-L2834)。

> **核心提醒**：环境变量有**两套独立机制**——敏感选项和非敏感选项走完全不同的代码路径，这也是最容易混淆的地方。

---

## 一、优先级总表（从低到高）

| 优先级 | 层级 | 来源 | 适用范围 | `where_defined` 标识 |
|--------|------|------|----------|---------------------|
| 1（最低） | 默认值层 | 代码内 `_create_option()` 定义的 `default_val` 或装饰器函数 | 所有配置项 | `"<default>"` |
| 2 | 全局配置文件 | `~/.streamlit/config.toml` | 所有非 `sensitive` 选项 | 配置文件绝对路径 |
| 3 | 项目配置文件 | `$CWD/.streamlit/config.toml` | 同上 | 配置文件绝对路径 |
| 4 | 脚本配置文件 | 主脚本所在目录下的 `.streamlit/config.toml` | 同上 | 配置文件绝对路径 |
| 5 | 敏感环境变量层 | `STREAMLIT_*` 系列环境变量 | 仅 `sensitive=True` 的选项 | `"environment variable"` |
| 6 | CLI / 非敏感环境变量层 | CLI flag 与 Click 库的 `envvar` 机制 | 仅 `sensitive=False` 的选项 | `"command-line argument or environment variable"` |
| 7（最高） | 运行时层 | 脚本内调用 `st.set_option()` | 仅 `scriptable=True` 的选项 | `"<user defined>"` |

---

## 二、各层级边界与代码路径详解

### 2.1 第 1 层：默认值

默认值注册在 `_config_options_template` 字典中，有两种定义方式。

**形式 A：静态默认值**

```python
# lib/streamlit/config.py
_create_option(
    "global.disableWidgetStateDuplicationWarning",
    default_val=False,
    type_=bool,
)
```

**形式 B：动态计算默认值（装饰器语法）**

```python
# lib/streamlit/config.py
@_create_option("global.developmentMode", visibility="hidden", type_=bool)
@util.memoize
def _global_development_mode() -> bool:
    return "site-packages" not in __file__
```

装饰器返回的函数保存在 `ConfigOption._get_val_func` 中，每次访问 `.value` 属性时调用（可用 `@util.memoize` 缓存结果）。

> 边界：默认值是所有其他层的基础。每次触发重新解析时，都会从 `_config_options_template` 深拷贝出一份全新的 `_config_options`，在此之上逐步叠加各层覆盖。

---

### 2.2 第 2-4 层：三层 config.toml 文件

#### 加载顺序

文件查找列表由 `get_config_files()` 返回，定义在 [lib/streamlit/config.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2726-L2748)。列表顺序决定加载顺序，后加载的覆盖先加载的：

```python
def get_config_files(file_name):
    config_files = [
        file_util.get_streamlit_file_path(file_name),        # 全局层：~/.streamlit/
        file_util.get_project_streamlit_file_path(file_name), # 项目层：$CWD/.streamlit/
    ]
    if _main_script_path is not None:
        # 脚本层：主脚本目录下的 .streamlit/（若与上不同则追加）
        config_files.append(script_level_config)
    return config_files
```

每层文件都会调用 `_update_config_with_toml()` 解析，内部递归遍历 TOML 层级结构，最终通过 `_set_option()` 直接覆盖已有值。

#### config.toml 内的 `env:` 引用

在 config.toml 中可使用 `env:VAR_NAME` 语法引用外部环境变量，由 `_maybe_read_env_variable()` 解析，定义在 [lib/streamlit/config.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2672-L2703)：

```toml
[server]
port = "env:MY_APP_PORT"
```

```python
def _maybe_read_env_variable(value):
    if isinstance(value, str) and value.startswith("env:"):
        var_name = value[len("env:"):]
        env_var = os.environ.get(var_name)
        if env_var is not None:
            return _maybe_convert_to_number(env_var)
    return value
```

> **边界要点**：
> - `env:` 引用是**所在 config.toml 层级的一部分**，不是独立的环境变量层。
> - 它在 TOML 解析阶段就地展开，与所在文件的优先级完全一致。例如全局 config.toml 里的 `env:` 引用，优先级仍然低于项目级 config.toml 中的普通字面量值。
> - 若引用的环境变量不存在，会保留 `env:xxx` 字符串原样作为值（不报错，值就是该字符串）。
> - 仅在 config.toml 中可用，CLI 层不支持此语法。

---

### 2.3 第 5 层：敏感选项的环境变量

**仅适用于 `sensitive=True` 的配置项**，由 `_update_config_with_sensitive_env_var()` 处理，定义在 [lib/streamlit/config.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2537-L2550)：

```python
def _update_config_with_sensitive_env_var(config_options):
    for opt_name, opt_val in config_options.items():
        if not opt_val.sensitive:
            continue
        env_var_value = os.environ.get(opt_val.env_var)
        if env_var_value is None:
            continue
        _set_option(opt_name, env_var_value, _DEFINED_BY_ENV_VAR)
```

环境变量名规则由 `ConfigOption.env_var` 属性给出，定义在 [lib/streamlit/config_option.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_option.py#L309-L312)：

```python
@property
def env_var(self) -> str:
    name = self.key.replace(".", "_")
    return f"STREAMLIT_{to_snake_case(name).upper()}"
```

示例：`server.cookieSecret` 对应环境变量 `STREAMLIT_SERVER_COOKIE_SECRET`。

> **边界要点**：
> - 仅 `sensitive=True` 的选项能通过此层的环境变量设置，典型如 `server.cookieSecret`、`mapbox.token`。
> - 这层在三层 config.toml **之后**、CLI flag **之前**生效。
> - 敏感选项**不能通过 CLI flag 设置**（尝试时会直接报错退出）。
> - 此层的 `where_defined` 标记为 `"environment variable"`。

---

### 2.4 第 6 层：CLI flag 与非敏感环境变量

这一层最容易误解：**非敏感选项的环境变量不是在 config.py 里读取，而是在 CLI 层通过 Click 库的 `envvar` 机制统一处理**。

核心代码位于 [lib/streamlit/web/cli.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/web/cli.py#L86-L110) 的 `configurator_options()` 装饰器：

```python
def configurator_options(func):
    for _, value in reversed(_config._config_options_template.items()):
        parsed_parameter = _convert_config_option_to_click_option(value)
        if value.sensitive:
            # 敏感选项：伪装成隐藏 CLI 参数，用户一旦传了就报错
            click_option_kwargs = {
                "expose_value": False,
                "hidden": True,
                "is_eager": True,
                "callback": _make_sensitive_option_callback(value),
            }
        else:
            # 非敏感选项：同时绑定 CLI flag 和环境变量
            click_option_kwargs = {
                "show_envvar": True,
                "envvar": parsed_parameter["envvar"],  # 关键：Click 自动读环境变量
            }
        config_option = click.option(...)
```

Click 的 `envvar` 参数语义如下：
- 若用户传了 CLI flag，使用 CLI flag 的值；
- 若用户没传 CLI flag 但设置了对应环境变量，使用环境变量的值；
- 两者都没提供，则值为 `None`（不参与覆盖）。

Click 解析后的结果通过 `flag_options` → `options_from_flags` 传入配置系统，中转代码在 [lib/streamlit/web/bootstrap.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/web/bootstrap.py#L286-L294)：

```python
options_from_flags = {
    name.replace("_", "."): val
    for name, val in flag_options.items()
    if val is not None and val != ()
}
config.get_config_options(force_reparse=True, options_from_flags=options_from_flags)
```

最终在 `get_config_options()` 内部逐个设置：

```python
for opt_name, opt_val in options_from_flags.items():
    _set_option(opt_name, opt_val, _DEFINED_BY_FLAG)
```

> **边界要点**：
> - 非敏感选项的环境变量**与 CLI flag 同属一层**，由 Click 在命令解析阶段统一裁决：CLI flag 优先级高于环境变量（Click 默认行为）。
> - 非敏感选项的环境变量**优先级高于三层 config.toml，也高于敏感环境变量层**。
> - 无论最终是 CLI flag 还是环境变量胜出，`where_defined` 一律标记为 `"command-line argument or environment variable"`，无法从配置对象区分具体来源。
> - 敏感选项走第 5 层生效，非敏感选项走第 6 层生效——两套机制完全独立。

---

### 2.5 第 7 层：运行时修改（st.set_option()）

用户脚本中调用 `st.set_option()`，最终进入 `set_user_option()`，定义在 [lib/streamlit/config.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L146-L191)：

```python
def set_user_option(key, value):
    opt = _config_options_template[key]
    if opt.scriptable:        # 白名单校验
        set_option(key, value)
        return
    raise StreamlitAPIException(...)
```

此层的 `where_defined` 标记为 `"<user defined>"`。

> **边界要点**：
> - 只有 `scriptable=True` 的选项允许运行时修改。当前可运行时修改的选项包括：`client.showErrorDetails`、`client.toolbarMode`、`client.showSidebarNavigation`、`logger.enableRich`、`server.enableArrowTruncation` 等。
> - 运行时修改只影响当前进程中的配置对象，不会回写到任何 config.toml 文件。
> - server 相关选项即使标记为 scriptable，变更也可能需要重启服务才能生效（修改时会打印警告）。

---

## 三、配置来源追踪：where_defined

每个 `ConfigOption` 对象都带有 `where_defined` 字段，记录当前值的最终来源。该字段在 `set_value()` 中每次赋值时更新，定义在 [lib/streamlit/config_option.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_option.py#L242-L260)：

```python
def set_value(self, value, where_defined=None):
    self._get_val_func = lambda: value
    self.where_defined = where_defined or ConfigOption.DEFAULT_DEFINITION
    self.is_default = value == self.default_val
```

所有特殊常量汇总：

| 常量 | 值 | 含义 |
|------|----|------|
| `ConfigOption.DEFAULT_DEFINITION` | `"<default>"` | 使用默认值，未被任何来源覆盖 |
| `ConfigOption.STREAMLIT_DEFINITION` | `"<streamlit>"` | 由 Streamlit 内部代码设置 |
| `config._USER_DEFINED` | `"<user defined>"` | 由脚本内 `st.set_option()` 设置 |
| `config._DEFINED_BY_FLAG` | `"command-line argument or environment variable"` | 由 CLI flag 或非敏感环境变量设置 |
| `config._DEFINED_BY_ENV_VAR` | `"environment variable"` | 由敏感环境变量设置 |

可通过 `config.get_where_defined("server.port")` 查询具体选项的来源。

---

## 四、特殊机制：主题继承（theme.base）

当 `theme.base` 指向一个本地 TOML 文件或 URL（而不是简单的 `"light"` 或 `"dark"`）时，会触发额外的主题继承流程。该流程由 `process_theme_inheritance()` 处理，定义在 [lib/streamlit/config_util.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_util.py#L744-L887)，发生在所有常规配置源加载完成**之后**。

### 4.1 主题继承的子优先级链

在主题继承内部，同样遵循从低到高的覆盖顺序：

| 子层级 | 来源 | 对应常规层级 |
|--------|------|---------------|
| T1 | `theme.base` 引用的主题文件 | （最低） |
| T2 | 全局 config.toml 中的主题选项 | 第 2 层 |
| T3 | 项目级 config.toml 中的主题选项 | 第 3 层 |
| T4 | 脚本级 config.toml 中的主题选项 | 第 4 层 |
| T5 | 敏感环境变量中的主题选项 | 第 5 层（一般主题选项不 sensitive） |
| T6 | CLI / 非敏感环境变量中的主题选项 | 第 6 层 |

### 4.2 主题继承执行流程

```python
def process_theme_inheritance(config_options, ...):
    # 1. 加载 theme.base 指向的主题文件
    theme_file_content = _load_theme_file(base_value, ...)

    # 2. 提取当前已设置的主题覆盖值，并按来源分类
    high_precedence_theme_options = {...}   # env var / CLI 的主题选项
    config_theme_overrides = {...}           # 各层 config.toml 的主题选项

    # 3. 清空所有主题选项（保留 theme.base 本身）
    for opt_name in theme_options_to_remove:
        set_option_func(opt_name, None, "reset for theme inheritance")

    # 4. 写入主题文件中的值（最低优先级 T1）
    _set_theme_options_recursive(theme_section, "theme", set_option_func,
                                 f"base theme file: {base_value}")

    # 5. 恢复 config.toml 各层的覆盖值（T2-T4）
    for opt_name, opt_data in config_theme_overrides.items():
        set_option_func(opt_name, opt_data["value"], opt_data["where_defined"])

    # 6. 恢复环境变量与 CLI 的覆盖值（T5-T6，最高优先级）
    for opt_name, opt_data in high_precedence_theme_options.items():
        set_option_func(opt_name, opt_data["value"], opt_data["where_defined"])
```

主题配置的深层字典合并由 `_deep_merge_theme_dicts()` 完成，定义在 [lib/streamlit/config_util.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_util.py#L684-L701)。

---

## 五、冲突检测与自动修正

常规配置加载完成后，通过 `on_config_parsed` 信号触发 `_check_conflicts()` 进行冲突检查，定义在 [lib/streamlit/config.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2837-L2879)。目前包含两条规则：

1. **开发模式端口冲突**：当 `global.developmentMode=true` 时，禁止设置 `server.port` 或 `browser.serverPort`，直接抛异常。

2. **XSRF/CORS 冲突**：若 `server.enableXsrfProtection=true` 但 `server.enableCORS=false`，系统会**自动将 `server.enableCORS` 强制改为 true** 并打印警告。

这是配置系统中唯一的"隐性覆盖"机制——在所有显式来源加载完毕后，再由代码主动改写某个选项的值。

---

## 六、综合示例

### 场景 1：环境变量两套机制对比

同时观察 `server.port`（非 sensitive）与 `server.cookieSecret`（sensitive）：

| 配置项 | 来源 | 值 | 生效层级 | 说明 |
|--------|------|----|----------|------|
| `server.port` | 默认值 | 8501 | 1 | 静态默认 |
| `server.port` | 全局 config.toml | 8502 | 2 | 覆盖默认值 |
| `server.port` | 环境变量 `STREAMLIT_SERVER_PORT=8503` | 8503 | 6 | 非敏感选项走 CLI 层 Click 的 `envvar` |
| `server.cookieSecret` | 默认值 | 随机生成 | 1 | 动态默认（装饰器函数） |
| `server.cookieSecret` | 全局 config.toml | `abc123` | 2 | 覆盖默认值 |
| `server.cookieSecret` | 环境变量 `STREAMLIT_SERVER_COOKIE_SECRET=xyz789` | `xyz789` | 5 | 敏感选项走独立环境变量层 |

说明：虽然两者都通过环境变量生效，但 `server.port` 属于第 6 层、`server.cookieSecret` 属于第 5 层。若项目级 config.toml 同时对 `server.port` 赋值，会被第 6 层覆盖；对 `server.cookieSecret` 赋值则被第 5 层覆盖。

### 场景 2：config.toml 中的 `env:` 引用

全局 config.toml：

```toml
[server]
port = "env:MY_PORT"   # env: 引用，与全局文件同优先级
```

项目级 config.toml：

```toml
[server]
port = 9000
```

假设环境变量 `MY_PORT=8888`。

**最终结果**：`server.port = 9000`——项目级 config.toml 的字面量值覆盖了全局 config.toml 中展开的 `env:` 引用值，因为项目级（第 3 层）整体优先级高于全局级（第 2 层）。

### 场景 3：CLI flag vs 环境变量（非敏感选项）

同时设置环境变量 `STREAMLIT_SERVER_PORT=8888`，并在 CLI 传 `--server.port 9999`。

**最终结果**：`server.port = 9999`——Click 裁决 CLI flag 优先于环境变量，两者的 `where_defined` 都会标记为 `"command-line argument or environment variable"`，无法从配置对象区分具体来源。

---

## 七、关键文件索引

| 文件（仓库相对路径） | 作用 |
|----------------------|------|
| [lib/streamlit/config.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py) | 配置系统主模块：定义全部选项、加载合并主逻辑、敏感环境变量处理 |
| [lib/streamlit/config_option.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_option.py) | `ConfigOption` 类：存储单个配置项的元数据与值，维护 `where_defined` |
| [lib/streamlit/config_util.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_util.py) | 配置工具：主题继承处理、`streamlit config show` 输出 |
| [lib/streamlit/web/cli.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/web/cli.py) | CLI 入口：非敏感选项的环境变量通过 Click 的 `envvar` 机制解析 |
| [lib/streamlit/web/bootstrap.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/web/bootstrap.py) | 启动引导：将 CLI 解析结果传入配置系统 |
| [lib/tests/streamlit/config_test.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/tests/streamlit/config_test.py) | 单元测试：包含各层覆盖优先级的验证用例 |
