# Streamlit 配置加载优先级机制详解

## 总览

Streamlit 配置系统采用 **7 层来源 + 2 条环境变量分支**的覆盖策略，整个加载流程由 `get_config_options()` 函数统一调度，定义位于 `lib/streamlit/config.py`。

> **三个核心校正（需特别注意）**：
> 1. 三层 config.toml 文件**不过滤敏感选项**，`server.cookieSecret` 等可直接写入。
> 2. 第 5 层（敏感环境变量）与第 6 层（CLI + 非敏感环境变量）**处理的字段集合完全互斥**，同一字段不会同时出现在两层里，因此不存在同字段"高低优先级"的竞争关系。
> 3. `theme.base` 指向外部主题文件/URL 后，**最终一定会被改写为 `"light"` 或 `"dark"`**，用户写入的路径/URL 不会保留在最终配置中。

---

## 一、优先级总表（从低到高）

| 优先级 | 层级 | 来源 | 覆盖字段集合 | `where_defined` 标识 |
|--------|------|------|--------------|---------------------|
| 1（最低） | 默认值层 | 代码内 `_create_option()` 定义的 `default_val` 或装饰器函数 | 全部配置项 | `"<default>"` |
| 2 | 全局配置文件 | `~/.streamlit/config.toml` | 全部配置项（含 `sensitive`） | 配置文件的实际路径 |
| 3 | 项目配置文件 | `$CWD/.streamlit/config.toml` | 全部配置项（含 `sensitive`） | 配置文件的实际路径 |
| 4 | 脚本配置文件 | 主脚本所在目录下的 `.streamlit/config.toml` | 全部配置项（含 `sensitive`） | 配置文件的实际路径 |
| 5 | 敏感环境变量分支 | `STREAMLIT_*` 系列环境变量 | **仅** `sensitive=True` 的字段（当前共 2 个：`server.cookieSecret`、`mapbox.token`） | `"environment variable"` |
| 6 | CLI + 非敏感环境变量分支 | CLI flag 与 Click 库的 `envvar` 机制 | **仅** `sensitive=False` 的字段（其余所有配置项） | `"command-line argument or environment variable"` |
| 7（最高） | 运行时层 | 脚本内调用 `st.set_option()` | 仅 `scriptable=True` 的字段子集 | `"<user defined>"` |

> **层级 5 与 6 的关系说明**：两者是**并行分支**而非高低层级。每个字段的 `sensitive` 属性在注册时就已确定，决定了该字段走哪条分支读取环境变量；两个分支之间没有共享字段，因此不会出现第 6 层"覆盖"第 5 层的情况。

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

### 2.2 第 2-4 层：三层 config.toml 文件（含 sensitive 字段）

#### 加载顺序

文件查找列表由 `get_config_files()` 返回，定义位于 `lib/streamlit/config.py`。列表顺序决定加载顺序，后加载的覆盖先加载的：

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

每层文件都会调用 `_update_config_with_toml()` 解析，内部递归遍历 TOML 层级结构，最终调用 `_set_option()` 直接覆盖已有值。

#### 关键代码证据：不过滤 sensitive

`_update_config_with_toml()` 的核心赋值语句（`lib/streamlit/config.py`）：

```python
def process_section(section_path: str, section_data: dict[str, Any]) -> None:
    for name, value in section_data.items():
        option_name = f"{section_path}.{name}"
        # ...（仅对 theme 的嵌套子 section 做特殊校验，无 sensitive 判断）
        else:
            # It's a regular config option, set it
            # 注意：这里 _set_option 前没有任何 if opt.sensitive 判断
            _set_option(option_name, _maybe_read_env_variable(value), where_defined)
```

> **校正结论**：`_set_option` 前只有 theme 嵌套结构的合法性检查，**没有任何对 `sensitive` 属性的过滤**。因此 `server.cookieSecret`、`mapbox.token` 完全可以写入 config.toml 并正常生效。

#### config.toml 内的 `env:` 引用

在 config.toml 中可使用 `env:VAR_NAME` 语法引用外部环境变量，由 `_maybe_read_env_variable()` 解析，定义位于 `lib/streamlit/config.py`：

```toml
[server]
cookieSecret = "env:MY_COOKIE_SECRET"   # sensitive 字段同样可用 env: 引用
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
> - `sensitive` 字段（如 `server.cookieSecret`）使用 `env:` 引用时同样有效，读取到的敏感值不会被单独处理。
> - 若引用的环境变量不存在，会保留 `env:xxx` 字符串原样作为值（不报错，值就是该字符串）。
> - 仅在 config.toml 中可用，CLI 层不支持此语法。

---

### 2.3 分支 A：敏感环境变量（第 5 层，仅覆盖 `sensitive=True` 字段）

#### 适用字段

当前代码中 `sensitive=True` 的配置项仅有两项：
- `server.cookieSecret`（`lib/streamlit/config.py` L831）
- `mapbox.token`（`lib/streamlit/config.py` L1213，已标记 deprecated）

#### 处理逻辑

由 `_update_config_with_sensitive_env_var()` 处理，定义位于 `lib/streamlit/config.py`：

```python
def _update_config_with_sensitive_env_var(config_options):
    for opt_name, opt_val in config_options.items():
        if not opt_val.sensitive:   # 显式跳过非 sensitive 字段
            continue
        env_var_value = os.environ.get(opt_val.env_var)
        if env_var_value is None:
            continue
        _set_option(opt_name, env_var_value, _DEFINED_BY_ENV_VAR)
```

环境变量名规则由 `ConfigOption.env_var` 属性给出，定义位于 `lib/streamlit/config_option.py`：

```python
@property
def env_var(self) -> str:
    name = self.key.replace(".", "_")
    return f"STREAMLIT_{to_snake_case(name).upper()}"
```

示例：`server.cookieSecret` → `STREAMLIT_SERVER_COOKIE_SECRET`

> **边界要点**：
> - 仅 `sensitive=True` 的字段进入此分支；非 sensitive 字段在此处被 `if not opt_val.sensitive: continue` 显式跳过。
> - 这一分支在三层 config.toml **之后**、CLI flag **之前**执行；对同字段（如 `server.cookieSecret`）会覆盖前面 4 层的值。
> - 敏感字段**不能通过 CLI flag 设置**（在 CLI 层会触发报错回调并直接退出）。
> - 此分支的 `where_defined` 标记为 `"environment variable"`。

---

### 2.4 分支 B：CLI flag + 非敏感环境变量（第 6 层，仅覆盖 `sensitive=False` 字段）

这一分支最容易误解：**非 sensitive 字段的环境变量不是在 config.py 里读取，而是在 CLI 层通过 Click 库的 `envvar` 机制统一处理**。

#### 核心证据：sensitive 字段不绑定 Click 的 envvar

代码位于 `lib/streamlit/web/cli.py` 的 `configurator_options()` 装饰器：

```python
def configurator_options(func):
    for _, value in reversed(_config._config_options_template.items()):
        parsed_parameter = _convert_config_option_to_click_option(value)
        if value.sensitive:
            # 敏感字段：伪装成隐藏 CLI 参数，用户一旦传了就报错
            # 注意：此处故意不传入 envvar，Click 不会读环境变量
            click_option_kwargs = {
                "expose_value": False,
                "hidden": True,
                "is_eager": True,
                "callback": _make_sensitive_option_callback(value),
            }
        else:
            # 非敏感字段：同时绑定 CLI flag 和环境变量
            click_option_kwargs = {
                "show_envvar": True,
                "envvar": parsed_parameter["envvar"],  # 关键：Click 自动读环境变量
            }
        config_option = click.option(
            parsed_parameter["option"],
            parsed_parameter["param"],
            help=parsed_parameter["description"],
            type=parsed_parameter["type"],
            multiple=parsed_parameter["multiple"],
            **click_option_kwargs,
        )
        func = config_option(func)
    return func
```

Click 的 `envvar` 参数语义（对 `sensitive=False` 字段）：
- 若用户传了 CLI flag，使用 CLI flag 的值；
- 若用户没传 CLI flag 但设置了对应环境变量，使用环境变量的值；
- 两者都没提供，则值为 `None`（不参与覆盖）。

Click 解析后的结果通过 `flag_options` → `options_from_flags` 传入配置系统，中转代码位于 `lib/streamlit/web/bootstrap.py`：

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

#### 与分支 A（敏感环境变量）的字段互斥关系

| 字段属性 | 读取环境变量的位置 |
|----------|-------------------|
| `sensitive=True` | 走 config.py 的 `_update_config_with_sensitive_env_var()`（分支 A / 第 5 层），CLI 层故意不传 `envvar` 不读 |
| `sensitive=False` | 走 CLI 层 Click 的 `envvar` 机制（分支 B / 第 6 层），config.py 层被 `if not opt_val.sensitive: continue` 跳过 |

> **边界要点**：
> - 第 5 层和第 6 层**不存在对同一字段的高低优先级竞争**——每个字段因 `sensitive` 属性不同，只会命中其中一层的环境变量处理。
> - 对非敏感字段而言，CLI flag 和环境变量同属第 6 层，由 Click 统一裁决：CLI flag 优先级高于环境变量（Click 默认行为）。
> - 对非敏感字段而言，第 6 层的覆盖对象是前 4 层（默认值 + 三层 config.toml）；对敏感字段而言，第 5 层的覆盖对象是前 4 层。
> - 无论最终是 CLI flag 还是环境变量胜出非敏感字段，`where_defined` 一律标记为 `"command-line argument or environment variable"`，无法从配置对象区分具体来源。

---

### 2.5 第 7 层：运行时修改（st.set_option()）

用户脚本中调用 `st.set_option()`，最终进入 `set_user_option()`，定义位于 `lib/streamlit/config.py`：

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

每个 `ConfigOption` 对象都带有 `where_defined` 字段，记录当前值的最终来源。该字段在 `set_value()` 中每次赋值时更新，定义位于 `lib/streamlit/config_option.py`：

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
| `config._DEFINED_BY_FLAG` | `"command-line argument or environment variable"` | 由 CLI flag 或**非敏感**环境变量设置（分支 B） |
| `config._DEFINED_BY_ENV_VAR` | `"environment variable"` | 由**敏感**环境变量设置（分支 A） |

可通过 `config.get_where_defined("server.port")` 查询具体选项的来源。

---

## 四、特殊机制：主题继承（theme.base）

当 `theme.base` 指向一个本地 TOML 文件或 URL（而不是简单的 `"light"` 或 `"dark"`）时，会触发额外的主题继承流程。该流程由 `process_theme_inheritance()` 处理，定义位于 `lib/streamlit/config_util.py`，发生在所有常规配置源加载完成**之后**。

### 4.1 关键校正：theme.base 最终一定落成 "light" 或 "dark"

主题继承的专门处理逻辑（`lib/streamlit/config_util.py` L850-L858）：

```python
# Handle theme.base - always set it to a valid value ("light" or "dark", not a path/URL)
theme_file_base = theme_file_content.get("theme", {}).get("base")
if theme_file_base:
    # 优先用引用主题文件中声明的 base 值（必须是 "light" 或 "dark"）
    set_option_func(
        "theme.base", theme_file_base, f"base theme file: {base_value}"
    )
else:
    # 主题文件没写 base，默认回退到 "light"
    set_option_func("theme.base", "light", "default light theme")
```

> **校正结论**：用户在 config.toml 或环境变量中写入的外部路径/URL（如 `theme.base = "./themes/my_theme.toml"` 或远程 URL），**仅被用作加载主题文件的指针**，不会作为最终值保留。主题继承流程执行完毕后，`theme.base` 一定是 `"light"` 或 `"dark"` 之一。

配套证据还有两处：
1. `_set_theme_options_recursive()` 在递归写入主题文件的选项时，**显式跳过** `theme.base`（L725-L727），避免用引用文件里的值覆盖这个专门逻辑。
2. 收集覆盖值时（L817、L833），同样排除 `opt_name == "theme.base"`，防止恢复原始的外部路径/URL。

### 4.2 主题继承的子优先级链

在主题继承内部，除 `theme.base` 外的其他主题选项（如 `theme.primaryColor`）遵循从低到高的覆盖顺序：

| 子层级 | 来源 | 对应常规层级 |
|--------|------|---------------|
| T1 | `theme.base` 引用的主题文件 | （最低） |
| T2 | 全局 config.toml 中的主题选项 | 第 2 层 |
| T3 | 项目级 config.toml 中的主题选项 | 第 3 层 |
| T4 | 脚本级 config.toml 中的主题选项 | 第 4 层 |
| T5 | 环境变量中的主题选项（sensitive 走分支 A，非 sensitive 走分支 B） | 第 5 / 6 层（主题字段通常为非 sensitive，实际走分支 B） |
| T6 | CLI flag 中的主题选项 | 第 6 层 |

### 4.3 主题继承执行流程

```python
def process_theme_inheritance(config_options, ...):
    # 1. 加载 theme.base 指向的主题文件（仅用作模板，base 值不落盘）
    theme_file_content = _load_theme_file(base_value, ...)

    # 2. 提取当前已设置的主题覆盖值，并按来源分类（排除 theme.base）
    high_precedence_theme_options = {...}   # env var / CLI 的主题选项
    config_theme_overrides = {...}           # 各层 config.toml 的主题选项

    # 3. 清空所有主题选项（保留 theme.base，待下一步专门改写）
    for opt_name in theme_options_to_remove:
        set_option_func(opt_name, None, "reset for theme inheritance")

    # 4. 专门改写 theme.base 为 "light" 或 "dark"，绝不保留外部路径/URL
    theme_file_base = theme_file_content.get("theme", {}).get("base")
    if theme_file_base:
        set_option_func("theme.base", theme_file_base, f"base theme file: {base_value}")
    else:
        set_option_func("theme.base", "light", "default light theme")

    # 5. 写入主题文件中的其他选项（最低优先级 T1，跳过 theme.base）
    _set_theme_options_recursive(theme_section, "theme", set_option_func,
                                 f"base theme file: {base_value}")

    # 6. 恢复 config.toml 各层的覆盖值（T2-T4）
    for opt_name, opt_data in config_theme_overrides.items():
        set_option_func(opt_name, opt_data["value"], opt_data["where_defined"])

    # 7. 恢复环境变量与 CLI 的覆盖值（T5-T6，最高优先级）
    for opt_name, opt_data in high_precedence_theme_options.items():
        set_option_func(opt_name, opt_data["value"], opt_data["where_defined"])
```

主题配置的深层字典合并由 `_deep_merge_theme_dicts()` 完成，定义位于 `lib/streamlit/config_util.py`。

---

## 五、冲突检测与自动修正

常规配置加载完成后，通过 `on_config_parsed` 信号触发 `_check_conflicts()` 进行冲突检查，定义位于 `lib/streamlit/config.py`。目前包含两条规则：

1. **开发模式端口冲突**：当 `global.developmentMode=true` 时，禁止设置 `server.port` 或 `browser.serverPort`，直接抛异常。

2. **XSRF/CORS 冲突**：若 `server.enableXsrfProtection=true` 但 `server.enableCORS=false`，系统会**自动将 `server.enableCORS` 强制改为 true** 并打印警告。

这是配置系统中唯一的"隐性覆盖"机制——在所有显式来源加载完毕后，再由代码主动改写某个选项的值。

---

## 六、综合示例

### 场景 1：环境变量两条分支的对比（非竞争关系）

| 配置项 | `sensitive` | 来源 | 值 | 实际命中层 | 说明 |
|--------|-------------|------|----|-----------|------|
| `server.port` | False | 默认值 | 8501 | 1 | 静态默认 |
| `server.port` | False | 全局 config.toml | 8502 | 2 | 覆盖默认值 |
| `server.port` | False | 环境变量 `STREAMLIT_SERVER_PORT=8503` | 8503 | **分支 B（第 6 层）** | 走 CLI 层 Click 的 `envvar` |
| `server.cookieSecret` | True | 默认值 | 随机生成 | 1 | 动态默认（装饰器函数） |
| `server.cookieSecret` | True | 全局 config.toml | `abc123` | 2 | **config.toml 允许写 sensitive**，覆盖默认值 |
| `server.cookieSecret` | True | 环境变量 `STREAMLIT_SERVER_COOKIE_SECRET=xyz789` | `xyz789` | **分支 A（第 5 层）** | 走 config.py 的敏感环境变量 |

说明：
- `server.port` 和 `server.cookieSecret` **不会在同一条分支里竞争**——前者只走分支 B，后者只走分支 A。
- 对 `server.cookieSecret` 来说：项目级 config.toml（第 3 层）的优先级低于其分支 A（第 5 层），环境变量胜出。
- 对 `server.port` 来说：项目级 config.toml（第 3 层）的优先级低于其分支 B（第 6 层），环境变量胜出。

### 场景 2：config.toml 中的 `env:` 引用（含 sensitive 字段）

全局 config.toml：

```toml
[server]
port = "env:MY_PORT"           # env: 引用，与全局文件同优先级
cookieSecret = "env:MY_SECRET"  # sensitive 字段同样可用 env: 引用
```

项目级 config.toml：

```toml
[server]
port = 9000
```

假设环境变量 `MY_PORT=8888`、`MY_SECRET=x1y2z3`。

**最终结果**：
- `server.port = 9000`——项目级 config.toml 的字面量值覆盖了全局文件展开的 `env:` 引用值，因为项目级（第 3 层）整体优先级高于全局级（第 2 层）。
- `server.cookieSecret = "x1y2z3"`——全局 config.toml 中的 `env:` 引用成功展开并写入（若项目级没写此字段则生效）。

### 场景 3：CLI flag vs 环境变量（同一分支 B 内的内部竞争）

同时设置环境变量 `STREAMLIT_SERVER_PORT=8888`，并在 CLI 传 `--server.port 9999`。

**最终结果**：`server.port = 9999`——同属分支 B（第 6 层），由 Click 内部裁决：CLI flag 优先于环境变量。两者的 `where_defined` 都会标记为 `"command-line argument or environment variable"`，无法从最终配置对象区分具体来源。

### 场景 4：theme.base 指向外部主题文件后最终值的变化

全局 config.toml：

```toml
[theme]
base = "./themes/corporate_dark.toml"   # 指向外部主题文件
primaryColor = "#ff0000"                # 用户自定义覆盖
```

假设 `./themes/corporate_dark.toml` 内容：

```toml
[theme]
base = "dark"          # 内联的 base 值
primaryColor = "#0000ff"
backgroundColor = "#111111"
```

**最终结果**：
- `theme.base = "dark"`——**不是**用户写入的 `"./themes/corporate_dark.toml"`，而是引用文件里声明的 `base` 值。
- `theme.primaryColor = "#ff0000"`——用户 config.toml 中的覆盖优先于主题文件。
- `theme.backgroundColor = "#111111"`——来自引用主题文件的默认值。

如果引用主题文件没写 `base`（无 `base = "dark"` 行），则最终 `theme.base = "light"`（默认回退）。

---

## 七、关键文件索引

| 文件（仓库相对路径） | 作用 |
|----------------------|------|
| `lib/streamlit/config.py` | 配置系统主模块：定义全部选项、加载合并主逻辑、敏感环境变量分支（A）处理 |
| `lib/streamlit/config_option.py` | `ConfigOption` 类：存储单个配置项的元数据与值，维护 `where_defined`，定义 `env_var` 命名规则 |
| `lib/streamlit/config_util.py` | 配置工具：主题继承处理（含 `theme.base` 的 `"light"`/`"dark"` 回退逻辑）、`streamlit config show` 输出 |
| `lib/streamlit/web/cli.py` | CLI 入口：非敏感选项的环境变量通过 Click 的 `envvar` 机制解析（分支 B），敏感字段用隐藏 flag + 报错回调拦截 |
| `lib/streamlit/web/bootstrap.py` | 启动引导：将 CLI 解析结果传入配置系统 |
| `lib/tests/streamlit/config_test.py` | 单元测试：包含各层覆盖优先级的验证用例 |
