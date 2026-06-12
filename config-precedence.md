# Streamlit 配置加载优先级机制详解

## 总览

Streamlit 配置系统采用 **7 层覆盖**策略，高层级的值会覆盖低层级的值。加载入口是 `get_config_options()` 函数，位于 [config.py#L2751-L2834](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2751-L2834)。

**特别注意：环境变量有**两套独立机制**（敏感选项和非敏感选项走完全不同的路径，这也是最容易混淆的地方。

---

## 一、优先级总表（从低到高）

| 优先级 | 层级 | 来源 | 适用范围 | where_defined 标识 |
|--------|------|------|----------|--------------------|
| 1（最低） | 默认值层 | 代码内 `_create_option()` 定义的 `default_val` 或装饰器函数 | 所有配置项 | `"<default>"` |
| 2 | 全局配置文件 | `~/.streamlit/config.toml | 所有非 sensitive 的选项 | 文件绝对路径 |
| 3 | 项目配置文件 | `$CWD/.streamlit/config.toml | 同上 | 文件绝对路径 |
| 4 | 脚本配置文件 | 主脚本目录下 `.streamlit/config.toml` | 同上 | 文件绝对路径 |
| 5 | 敏感环境变量层 | `STREAMLIT_* 环境变量 | 仅 `sensitive=True` 的选项 | `"environment variable"` |
| 6 | CLI / 非敏感环境变量层 | CLI flag + Click 解析的 `envvar` 机制 | 仅 `sensitive=False` 的选项 | `"command-line argument or environment variable"` |
| 7（最高） | 运行时层 | `st.set_option()` | 仅 `scriptable=True` 的选项 | `"<user defined>"` |

---

## 二、各层级边界与代码路径详解

### 2.1 第 1 层：默认值

默认值注册在 `_config_options_template` 字典中，有两种定义方式：

**形式 A：静态默认值**

```python
# config.py#L386-L397
_create_option(
    "global.disableWidgetStateDuplicationWarning",
    default_val=False,
    type_=bool,
)
```

**形式 B：动态计算默认值（装饰器语法）**

```python
# config.py#L411-L424
@_create_option("global.developmentMode", visibility="hidden", type_=bool)
@util.memoize
def _global_development_mode() -> bool:
    return "site-packages" not in __file__
```

装饰器函数保存在 `ConfigOption._get_val_func` 中，每次访问 `.value` 时调用（可通过 `@util.memoize` 可缓存结果）。

> 边界：默认值是所有其他层的基础，每次重新解析都会从 `_config_options_template` 深拷贝出一份全新的 `_config_options`。

---

### 2.2 第 2-4 层：三层 config.toml 文件

#### 加载顺序

由 `get_config_files()` 在 [config.py#L2726-L2748](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2726-L2748) 返回文件列表，按顺序加载，后加载的覆盖先加载的：

```python
def get_config_files(file_name):
    config_files = [
        file_util.get_streamlit_file_path(file_name),       # 全局: ~/.streamlit/
        file_util.get_project_streamlit_file_path(file_name), # 项目: $CWD/.streamlit/
    ]
    if _main_script_path is not None:
        # 脚本: 主脚本目录/.streamlit/ （如果与上不同）
        config_files.append(script_level_config)
    return config_files
```

每层都调用 `_update_config_with_toml()` 解析，内部递归遍历 TOML 结构并调用 `_set_option()` 直接覆盖。

#### config.toml 中的 `env:` 引用

在 config.toml 内可以使用 `env:VAR_NAME` 语法引用外部环境变量，由 `_maybe_read_env_variable()` 在 [config.py#L2672-L2703](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2672-L2703) 解析：

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
> - `env:` 引用是 **所在 config.toml 层级的一部分**，不是独立的环境变量层。
> - 它发生在 TOML 解析阶段，和所在文件同优先级。例如全局 config.toml 里的 `env:` 引用，优先级仍然低于项目级 config.toml 的普通值。
> - 解析失败（环境变量不存在）时，保留 `env:xxx` 字符串原样作为值（不会报错，只是值就是字符串）。
> - 只在 config.toml 中可用，CLI 层没有这个语法。

---

### 2.3 第 5 层：敏感选项的环境变量

**只适用于 `sensitive=True` 的配置项，由 `_update_config_with_sensitive_env_var()` 在 [config.py#L2537-L2550](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2537-L2550) 处理。

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

环境变量名规则（`ConfigOption.env_var` 属性，定义在 [config_option.py#L309-L312](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_option.py#L309-L312)：

```python
@property
def env_var(self) -> str:
    name = self.key.replace(".", "_")
    return f"STREAMLIT_{to_snake_case(name).upper()}"
```

例如 `server.cookieSecret` → `STREAMLIT_SERVER_COOKIE_SECRET`

> **边界要点**：
> - 仅 `sensitive=True` 的选项才能通过环境变量设置（如 `server.cookieSecret`、`mapbox.token`）。
> - 这层在三层 config.toml **之后**，CLI flag **之前**。
> - 敏感选项**不能通过 CLI flag 设置**（会直接报错退出）。
> - `where_defined` 标记为 `"environment variable"`。

---

### 2.4 第 6 层：CLI flag 与非敏感环境变量

这一层最容易误解：**非敏感选项的环境变量不是在 config.py 里处理，而是在 CLI 层通过 Click 库的 `envvar` 机制处理**。

#### 机制代码在 [cli.py#L86-L110](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/web/cli.py#L86-L110) 的 `configurator_options()` 装饰器：

```python
def configurator_options(func):
    for _, value in reversed(_config._config_options_template.items()):
        parsed_parameter = _convert_config_option_to_click_option(value)
        if value.sensitive:
            # 敏感选项：不允许通过 CLI 设置
            click_option_kwargs = {
                "expose_value": False,
                "hidden": True,
                "is_eager": True,
                "callback": _make_sensitive_option_callback(value),
            }
        else:
            # 非敏感选项：同时支持 CLI flag 和环境变量
            click_option_kwargs = {
                "show_envvar": True,
                "envvar": parsed_parameter["envvar"],  # 关键：Click 自动从环境变量读
            }
        config_option = click.option(...)
```

Click 的 `envvar` 参数意味着：
- 如果用户传了 CLI flag，用 CLI flag 的值
- 如果用户没传 CLI flag，但设置了对应环境变量，用环境变量的值
- 两者都没传，用 None（不覆盖）

解析后的值通过 `flag_options` → options_from_flags` 传入 `get_config_options()`：

```python
# bootstrap.py#L286-L294
options_from_flags = {
    name.replace("_", "."): val
    for name, val in flag_options.items()
    if val is not None and val != ()
}
config.get_config_options(force_reparse=True, options_from_flags=options_from_flags)
```

然后在 `get_config_options()` 内循环设置：

```python
for opt_name, opt_val in options_from_flags.items():
    _set_option(opt_name, opt_val, _DEFINED_BY_FLAG)
```

> **边界要点**：
> - 非敏感选项的环境变量**和 CLI flag 是同一层**，由 Click 统一解析，CLI flag 优先级高于环境变量（Click 默认行为）。
> - 非敏感选项的环境变量 **优先级高于 config.toml 和敏感环境变量层**。
> - 不管是 CLI flag 还是环境变量触发，`where_defined` 都标记为 `"command-line argument or environment variable"`，无法区分来源。
> - 敏感选项走第 层生效，非敏感选项走第 6 层。这是两套完全独立的环境变量机制。

---

### 2.5 第 7 层：运行时修改（st.set_option()）

用户脚本调用 `st.set_option()`，最终调用 `set_user_option()` 在 [config.py#L146-L191](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L146-L191)：

```python
def set_user_option(key, value):
    opt = _config_options_template[key]
    if opt.scriptable:       # 白名单校验
        set_option(key, value)
        return
    raise StreamlitAPIException(...)
```

`where_defined` 标记为 `"<user defined>"`。

> **边界要点**：
> - 只有 `scriptable=True` 的选项可以运行时修改。
> - 当前可运行时修改的选项：`client.showErrorDetails`、`client.toolbarMode`、`client.showSidebarNavigation`、`logger.enableRich`、`server.enableArrowTruncation` 等。
> - 运行时修改只影响当前运行的脚本实例，不会写入配置文件。
> - 某些 server 相关选项运行时修改无效（会导致警告：如果 server 选项变更需要重启）。

---

## 三、配置来源追踪：where_defined

每个 `ConfigOption` 的 `where_defined` 字段记录当前值的最终来源。在 `set_value()` 中每次更新，见 [config_option.py#L242-L260](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_option.py#L242-L260)：

```python
def set_value(self, value, where_defined=None):
    self._get_val_func = lambda: value
    self.where_defined = where_defined or ConfigOption.DEFAULT_DEFINITION
    self.is_default = value == self.default_val
```

特殊常量汇总：

| 常量 | 值 | 含义 |
|------|----|------|
| `ConfigOption.DEFAULT_DEFINITION` | `"<default>"` | 使用默认值，未被覆盖 |
| `ConfigOption.STREAMLIT_DEFINITION` | `"<streamlit>"` | Streamlit 内部代码设置 |
| `config._USER_DEFINED` | `"<user defined>"` | `st.set_option()` 设置 |
| `config._DEFINED_BY_FLAG` | `"command-line argument or environment variable"` | CLI flag 或非敏感环境变量 |
| `config._DEFINED_BY_ENV_VAR` | `"environment variable"` | 敏感环境变量 |

可通过 `config.get_where_defined("server.port")` 查询。

---

## 四、特殊机制：主题继承（theme.base）

当 `theme.base` 指向本地 TOML 文件或 URL（非简单的 `"light"` / `"dark"`）时，会触发主题继承流程。由 `process_theme_inheritance()` 在 [config_util.py#L744-L887](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_util.py#L744-L887) 处理，发生在所有常规配置源加载完成**之后**。

### 4.1 主题继承的子优先级链

主题继承内部也遵循整体优先级（从低到高）：

| 子层级 | 来源 | 对应常规层级 |
|--------|------|---------------|
| T1 | theme.base 引用的主题文件 | （最低） |
| T2 | 全局 config.toml 的主题选项 | 第 2 层 |
| T3 | 项目级 config.toml 的主题选项 | 第 3 层 |
| T4 | 脚本级 config.toml 的主题选项 | 第 4 层 |
| T5 | 敏感环境变量的主题选项 | 第 5 层（一般主题选项不 sensitive） |
| T6 | CLI / 非敏感环境变量的主题选项 | 第 6 层 |

### 4.2 主题继承执行流程

```python
def process_theme_inheritance(config_options, ...):
    # 1. 加载 theme.base 指向的主题文件
    theme_file_content = _load_theme_file(base_value, ...)

    # 2. 记录当前已设置的主题覆盖值（按来源分类）
    high_precedence_theme_options = {...}   # env var / CLI 的主题选项
    config_theme_overrides = {...}           # 各层 config.toml 的主题选项

    # 3. 清空所有主题选项（除 theme.base 外）
    for opt_name in theme_options_to_remove:
        set_option_func(opt_name, None, "reset for theme inheritance")

    # 4. 设置主题文件中的值（最低 T1）
    _set_theme_options_recursive(theme_section, "theme", set_option_func,
                                 f"base theme file: {base_value}")

    # 5. 恢复 config.toml 中的覆盖值（T2-T4）
    for opt_name, opt_data in config_theme_overrides.items():
        set_option_func(opt_name, opt_data["value"], opt_data["where_defined"])

    # 6. 恢复环境变量和 CLI 的覆盖值（T5-T6，最高）
    for opt_name, opt_data in high_precedence_theme_options.items():
        set_option_func(opt_name, opt_data["value"], opt_data["where_defined"])
```

深层合并使用 `_deep_merge_theme_dicts()` 递归字典合并，见 [config_util.py#L684-L701](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_util.py#L684-L701)。

---

## 五、冲突检测与自动修正

配置加载完成后，通过 `on_config_parsed` 信号触发 `_check_conflicts()` 在 [config.py#L2837-L2879](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py#L2837-L2879) 检查冲突：

1. **开发模式端口冲突**：`global.developmentMode=true` 时，不能设置 `server.port` 或 `browser.serverPort`（直接抛异常）

2. **XSRF/CORS 冲突**：若 `server.enableXsrfProtection=true` 但 `server.enableCORS=false`，会**自动将 `server.enableCORS` 覆盖为 true** 并打印警告

这是唯一的"隐性覆盖"机制 —— 配置系统会在加载后主动修改选项值。

---

## 六、综合示例

### 场景 1：环境变量两套机制对比

假设 `server.port`（非 sensitive）和 `server.cookieSecret`（sensitive）：

| 配置项 | 来源 | 值 | 生效层级 | 说明 |
|--------|------|----|----------|------|
| `server.port` | 默认值 | 8501 | 1 | 静态默认 |
| `server.port` | 全局 config.toml | 8502 | 2 | 覆盖默认 |
| `server.port` | 环境变量 `STREAMLIT_SERVER_PORT=8503 | 8503 | 6 | **非敏感选项走 CLI 层 envvar |
| `server.cookieSecret` | 默认值 | 随机生成 | 1 | 动态默认 |
| `server.cookieSecret` | 全局 config.toml | abc123 | 2 | 覆盖默认 |
| `server.cookieSecret` | 环境变量 `STREAMLIT_SERVER_COOKIE_SECRET=xyz789 | xyz789 | 5 | **敏感选项走独立 env var 层** |

注意：虽然都是环境变量，但 `server.port` 在第 6 层、`server.cookieSecret` 在第 5 层。如果项目级 config.toml 中的 `server.port` 会被环境变量覆盖（第 3 层 < 第 6 层），但 `server.cookieSecret` 也会被环境变量覆盖（第 2 层 < 第 5 层）。

### 场景 2：config.toml 中的 env: 引用

全局 config.toml：

```toml
[server]
port = "env:MY_PORT"  # env: 引用，和全局文件同优先级
```

项目级 config.toml：

```toml
[server]
port = 9000
```

如果环境变量 `MY_PORT=8888`。

**最终结果**：`server.port = 9000` —— 项目级文件的值覆盖了全局文件的 env: 引用值，因为项目级优先级更高。

### 场景 3：CLI flag vs 环境变量（非敏感选项）

环境变量 `STREAMLIT_SERVER_PORT=8888`，同时 CLI 传 `--server.port 9999`。

**最终结果**：`server.port = 9999` —— CLI flag 优先级高于环境变量（Click 的默认行为），`where_defined` 都是 `"command-line argument or environment variable"`。

---

## 七、关键文件索引

| 文件 | 作用 |
|------|------|
| [config.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config.py) | 配置系统主模块，定义所有选项、加载合并逻辑、敏感环境变量处理 |
| [config_option.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_option.py) | `ConfigOption` 类，存储单个配置项的元数据和值 |
| [config_util.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/config_util.py) | 配置工具函数，含主题继承处理、`config show` 输出 |
| [cli.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/web/cli.py) | CLI 入口，非敏感选项的环境变量由 Click 的 envvar 机制处理 |
| [bootstrap.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/streamlit/web/bootstrap.py) | 启动引导，将 CLI flag 传入 config 系统 |
| [config_test.py](file:///d:/fz/0601/solo-dogfeeding/code/234-streamlit/lib/tests/streamlit/config_test.py) | 覆盖优先级的单元测试用例 |
