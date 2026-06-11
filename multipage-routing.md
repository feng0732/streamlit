# Streamlit 多页应用页面发现与路由实现分析

## 概述

Streamlit 多页应用（Multipage App, MPA）经历了两个版本的演进：

- **v1（pages/ 目录模式）**：自动扫描 `pages/` 目录下的 Python 文件作为页面
- **v2（st.navigation 模式）**：通过 `st.navigation()` + `st.Page()` 显式声明页面

v2 是当前的推荐方式，提供了更灵活的页面配置和动态导航能力。

---

## 一、页面扫描与发现机制

### 1.1 v1: pages/ 目录自动扫描

在 v1 模式下，Streamlit 会自动扫描主脚本同级目录下的 `pages/` 文件夹，将其中的 `.py` 文件识别为页面。

**核心实现文件**：
- [source_util.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/source_util.py) — 页面名称/图标提取工具
- [pages_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/pages_manager.py) — 页面管理器

**页面命名规则**（[PAGE_FILENAME_REGEX](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/source_util.py#L58-L58)）：

```
正则: ([0-9]*)[_ -]*(.*)\.py
```

文件名格式示例：
- `01_🚀_Home.py` → 排序号 `01`，图标 `🚀`，名称 `Home`
- `About_Us.py` → 无排序号，名称 `About_Us`

**关键函数**：

| 函数 | 作用 | 位置 |
|------|------|------|
| `page_icon_and_name()` | 从文件名提取图标和名称 | [source_util.py#L80-L97](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/source_util.py#L80-L97) |
| `page_sort_key()` | 生成排序键（数字前缀 + 名称） | [source_util.py#L61-L77](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/source_util.py#L61-L77) |

**v1 检测逻辑**（[PagesManager.__init__](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/pages_manager.py#L60-L63)）：
```python
if PagesManager.uses_pages_directory is None:
    PagesManager.uses_pages_directory = Path(
        self.main_script_parent / "pages"
    ).exists()
```

### 1.2 v2: st.Page 显式声明

v2 模式下，页面通过 `st.Page()` 显式创建，不再依赖目录扫描。页面源可以是三种类型：

1. **Python 文件路径**（`str` 或 `Path`）
2. **可调用对象**（`Callable`，如函数）
3. **外部 URL**（`"https://..."`）

**核心实现文件**：
- [page.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/navigation/page.py) — StreamlitPage 类定义

**StreamlitPage 初始化流程**（[page.py#L264-L389](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/navigation/page.py#L264-L389)）：

```
输入 page 参数
    │
    ├─ 是外部 URL? → 验证 title 必填，设置 _external_url
    │
    ├─ 是 Path/str 文件路径? → 解析为绝对路径，验证文件存在
    │
    └─ 是 Callable? → 直接使用
         │
         ▼
    推断标题 (inferred_name)
    推断图标 (inferred_icon)
    计算 url_path（默认从名称推断）
    计算 _script_hash（基于 url_path 的哈希）
```

**URL 路径清理函数** `_sanitize_url_path()`（[page.py#L31-L45](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/navigation/page.py#L31-L45)）：
- 转小写
- 空白字符替换为下划线
- 移除 `& # ? / \ : * " < > | '` 等特殊字符
- 合并连续下划线

---

## 二、导航注册逻辑

### 2.1 st.navigation 入口

**核心文件**：[navigation.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/commands/navigation.py)

`st.navigation()` 是 v2 多页应用的核心入口，做了以下关键事情：

1. **禁用 v1 模式**：设置 `PagesManager.uses_pages_directory = False`
2. **页面类型转换**：将各种 page-like 对象转为 `StreamlitPage`
3. **默认页确定**：查找 `default=True` 的页面，没有则取第一个非外部页面
4. **构建页面注册表**：`pagehash_to_pageinfo` 字典
5. **发送导航 proto 消息**：通知前端导航菜单配置
6. **解析当前页面**：调用 `set_pages_and_resolve()` 确定要运行的页面
7. **返回当前页面对象**：用户调用 `.run()` 执行页面

### 2.2 导航注册完整流程

```
st.navigation(pages)
    │
    ├─ 1. 转换页面类型 convert_to_streamlit_page()
    │     ├─ StreamlitPage → 直接使用
    │     ├─ str/Path →  new StreamlitPage(page)
    │     └─ Callable → new StreamlitPage(page)
    │
    ├─ 2. 确定默认页
    │     ├─ 遍历查找 _default=True 的页面
    │     └─ 没有则取第一个非外部页面，并设为 default
    │
    ├─ 3. 构建 pagehash_to_pageinfo 映射
    │     └─ key: page._script_hash (url_path 的哈希)
    │        value: {page_script_hash, page_name, icon, script_path, url_pathname}
    │
    ├─ 4. 构建 Navigation proto 消息
    │     ├─ position: sidebar/top/hidden
    │     ├─ sections: 分组标题列表
    │     └─ app_pages: 页面列表（含脚本哈希、名称、图标、默认标记等）
    │
    ├─ 5. ctx.pages_manager.set_pages_and_resolve()
    │     └─ 设置页面注册表并解析当前页面
    │
    ├─ 6. 标记可执行页面 _can_be_called = True
    │
    └─ 7. ctx.enqueue(msg) 发送导航消息给前端
```

### 2.3 PagesManager 页面管理器

**核心文件**：[pages_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/pages_manager.py)

`PagesManager` 负责管理应用的所有页面，是后端路由的核心。

**关键属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `_main_script_path` | `str` | 主脚本（入口文件）路径 |
| `_main_script_hash` | `str` | 主脚本的哈希值 |
| `_pages` | `dict` | 页面注册表：`{page_hash: PageInfo}` |
| `_intended_page_script_hash` | `str` | 意图运行的页面哈希（来自前端） |
| `_intended_page_name` | `str` | 意图运行的页面名称（来自 URL） |
| `_current_page_script_hash` | `str` | 当前正在执行的页面哈希 |

**关键方法**：

| 方法 | 作用 |
|------|------|
| `set_script_intent()` | 设置意图运行的页面（哈希 + 名称） |
| `set_pages_and_resolve()` | 原子性设置页面注册表并解析当前页面 |
| `_resolve_page_script()` | 根据意图解析实际要运行的页面 |
| `get_initial_active_script()` | 获取初始活动脚本（v2 始终运行主脚本） |
| `get_page_script_byte_code()` | 从脚本缓存获取页面字节码 |

---

## 三、路径匹配与路由切换

### 3.1 页面标识体系

Streamlit 使用双层标识体系：

| 标识 | 说明 | 用途 |
|------|------|------|
| `url_pathname` | URL 友好的路径名 | 浏览器地址栏显示、URL 路由 |
| `page_script_hash` | 路径名的哈希值 | 内部唯一标识、前后端通信 |

**哈希计算**（[util.py: calc_hash](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/util.py)）：
```python
page._script_hash = calc_hash(page._url_path)
```

默认页的 `url_path` 始终为 `""`（空字符串）。

### 3.2 后端页面解析逻辑

`_resolve_page_script()` 是核心路由解析函数（[pages_manager.py#L170-L207](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/pages_manager.py#L170-L207)）：

```
解析优先级:
    1. intended_page_script_hash (优先，精确匹配)
    │    └─ 在 _pages 字典中查找 → 找到返回，否则 fallback
    │
    2. intended_page_name (URL 路径名匹配)
    │    └─ 遍历 _pages.values() 匹配 url_pathname
    │
    3. fallback_page_hash (默认页哈希)
         └─ 返回默认页
```

### 3.3 前端路径匹配

**核心文件**：[AppNavigation.ts](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/frontend/app/src/util/AppNavigation.ts)

**前端页面查找** `findPageByUrlPath()`（[AppNavigation.ts#L200-L217](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/frontend/app/src/util/AppNavigation.ts#L200-L217)）：

```typescript
findPageByUrlPath(pathname: string): IAppPage | null {
  // 先解码 URL（处理浏览器的 Unicode 编码）
  decodedPathname = decodeURIComponent(pathname)
  // 在 appPages 中查找 urlPathname 匹配的页面
  // 匹配方式: decodedPathname.endsWith("/" + appPage.urlPathname)
  // 没找到则返回 mainPage（默认页）
}
```

### 3.4 切页触发方式

页面切换有三种触发方式：

#### 方式 1: 用户点击导航菜单 / page_link
- 前端发送 `SET_CURRENT_PAGE_NAME` 消息给后端
- 后端触发 rerun，携带新的 `page_script_hash`

#### 方式 2: st.switch_page() 编程式切换

**核心实现**：[execution_control.py#L194-L347](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/commands/execution_control.py#L194-L347)

```python
def switch_page(page, *, query_params=None):
    # 1. 解析目标页面，获取 page_script_hash
    #    - StreamlitPage 对象: 直接取 _script_hash
    #    - 文件路径: 在 pages_manager 中查找匹配的 script_path
    #
    # 2. 设置 query_params（默认为空，清除原有参数）
    #
    # 3. 请求 rerun，携带 page_script_hash
    ctx.script_requests.request_rerun(
        RerunData(page_script_hash=page_script_hash, ...)
    )
```

#### 方式 3: 浏览器直接访问 URL
- 新会话初始化时从 URL 提取页面名称
- `set_script_intent()` 设置意图页面
- 首次脚本运行时解析

### 3.5 脚本运行流程中的路由

**核心文件**：[script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py)

切页时的脚本运行流程（[script_runner.py#L559-L636](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L559-L636)）：

```
收到 RERUN 请求 (携带 page_script_hash)
    │
    ├─ 1. pages_manager.set_script_intent()
    │     └─ 设置意图页面哈希和名称
    │
    ├─ 2. pages_manager.get_initial_active_script()
    │     └─ v2 模式下始终返回主脚本（入口文件）
    │
    ├─ 3. 检测页面是否变化
    │     └─ previous_page_script_hash != page_script_hash
    │
    ├─ 4. 页面变化时的处理
    │     ├─ 过滤 query_params（只保留主脚本和新页面的）
    │     └─ 清理 widget 状态
    │
    ├─ 5. ctx.reset()
    │     ├─ 设置当前 page_script_hash
    │     ├─ 重置 widget/form/fragment 追踪
    │     └─ 初始化 ThreadState
    │
    └─ 6. 执行主脚本
          └─ 主脚本中调用 st.navigation() → 返回当前页面 → pg.run()
```

**关键点**：v2 模式下，**每次 rerun 都执行主脚本（入口文件）**，页面逻辑通过 `pg.run()` 在主脚本内部调用。这与 v1 模式直接执行页面脚本不同。

---

## 四、切页状态保持机制

### 4.1 Session State 的跨页保持

`st.session_state` 在同一会话内是**全局共享**的，跨页面切换会保持。

**证据**：
- 会话状态存储在 `AppSession` 中，而不是单个页面
- 页面切换只是脚本重新执行，不会创建新会话
- 入口文件中定义的带 key widget 在跨页时保持状态

**推荐实践**（来自文档示例）：
```python
# 入口文件 streamlit_app.py
st.sidebar.selectbox("Foo", ["A", "B", "C"], key="foo")  # 跨页保持
pg = st.navigation([page1, page2])
pg.run()
```

### 4.2 Widget 状态的清理

页面切换时，不在新页面上的 widget 状态会被清理。

**清理机制**（[session_state.py: remove_stale_widgets()](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/state/session_state.py#L248-L257)）：
- 每次脚本运行后，比较 `active_widget_ids` 和现有状态
- 不在活跃列表中的 widget 状态被移除
- 入口文件中的 widget 每次都运行，因此状态保持

**切页时的显式清理**（[script_runner.py#L577-L594](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L577-L594)）：
```python
previous_page_script_hash = ctx.page_script_hash
if previous_page_script_hash != page_script_hash:
    # 页面变化时，使用 rerun_data 中的 widget 状态
    # 来维护部分 widget 状态（如页面链接按钮触发的切换）
```

### 4.3 Query Parameters 的处理

页面切换时 query params 会被**过滤**，避免不同页面的参数相互干扰。

**核心逻辑**（[script_runner.py#L595-L606](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L595-L606)）：

```python
# 仅保留主脚本和目标页面的 query params
valid_script_hashes = {main_script_hash, page_script_hash}
with self._session_state.query_params() as qp:
    qp.populate_from_query_string(
        rerun_data.query_string, valid_script_hashes
    )
    qp.set_initial_query_params_from_current()
```

**同页 rerun vs 跨页切换**（[script_run_context.py#L305-L307](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L305-L307)）：
- **同页 rerun**（widget 交互）：从 URL 填充 query params
- **跨页切换**：在 script_runner 中提前过滤，防止旧页面参数污染新页面

### 4.4 active_script_hash 与元素归属

Streamlit 使用 `active_script_hash` 来标记每个消息（元素）属于哪个页面。

**核心机制**（[script_run_context.py#L312-L330](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L312-L330)）：

```python
def enqueue(self, msg: ForwardMsg) -> None:
    # 每条消息都携带 active_script_hash
    msg.metadata.active_script_hash = ThreadState.get().active_script_hash
    ...
```

**页面执行时的 hash 切换**（[page.py#L484-L494](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/navigation/page.py#L484-L494)）：

```python
def run(self) -> None:
    with ctx.run_with_active_hash(self._script_hash):
        # 执行页面代码时，active_script_hash = 页面哈希
        # 这样页面内产生的元素都标记为该页面所有
        if isinstance(self._page, Path):
            exec(code, module.__dict__)
        else:
            self._page()
```

**前端元素清理**（[AppNavigation.ts#L230-L232](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/frontend/app/src/util/AppNavigation.ts#L230-L232)）：
```typescript
clearPageElements(elements: AppRoot, mainScriptHash: string): AppRoot {
  return elements.filterMainScriptElements(mainScriptHash)
}
```

这确保了切页时，旧页面的 UI 元素被正确清理，而主脚本（入口文件）的元素保留。

### 4.5 状态保持总结表

| 状态类型 | 跨页是否保持 | 说明 |
|----------|-------------|------|
| `st.session_state` | ✅ 是 | 会话级全局状态 |
| 入口文件的 widget | ✅ 是 | 每次 rerun 都重新执行，状态通过 key 保留 |
| 页面内的 widget | ❌ 否 | 切页后不再活跃，被清理 |
| `st.query_params` | ⚠️ 过滤 | 仅保留主脚本和当前页面的参数 |
| Fragment 状态 | ❌ 否 | 切页后 fragment 不再活跃 |
| 上传文件 | ✅ 是 | `UploadedFileManager` 是会话级的 |

---

## 五、核心数据结构

### 5.1 PageInfo（页面信息）

**定义**：[source_util.py#L33-L38](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/source_util.py#L33-L38)

```python
class PageInfo(TypedDict):
    script_path: ScriptPath       # Python 文件路径
    page_script_hash: PageHash    # 页面脚本哈希
    icon: NotRequired[Icon]       # 图标
    page_name: NotRequired[PageName]  # 页面名称
    url_pathname: NotRequired[str]   # URL 路径名
```

### 5.2 StreamlitPage（页面对象）

**核心属性**（[page.py#L211-L498](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/navigation/page.py#L211-L498)）：

| 属性 | 类型 | 说明 |
|------|------|------|
| `_page` | `Path \| Callable \| None` | 页面源（文件或函数） |
| `_title` | `str` | 页面标题 |
| `_icon` | `str` | 页面图标 |
| `_url_path` | `str` | URL 路径 |
| `_default` | `bool` | 是否默认页 |
| `_visibility` | `"visible" \| "hidden"` | 导航可见性 |
| `_external_url` | `str \| None` | 外部 URL（非空则为外链页） |
| `_can_be_called` | `bool` | 是否可执行（由 st.navigation 授权） |
| `_script_hash` | `str` | 基于 url_path 的哈希 |

### 5.3 Navigation Proto（导航消息）

前端通过 protobuf 接收导航配置，包含：
- `position`: 导航位置（SIDEBAR / TOP / HIDDEN）
- `sections`: 分组标题列表
- `app_pages`: AppPage 列表
- `page_script_hash`: 当前页面哈希
- `expanded`: 是否展开
- `visible_items`: 可见项数量

---

## 六、关键文件索引

| 文件 | 职责 |
|------|------|
| [navigation.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/commands/navigation.py) | st.navigation 命令实现 |
| [page.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/navigation/page.py) | StreamlitPage 类定义 |
| [pages_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/pages_manager.py) | 页面管理器（后端路由核心） |
| [source_util.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/source_util.py) | 页面名称/图标提取工具 |
| [script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py) | 脚本运行器（切页执行流程） |
| [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py) | 脚本运行上下文（active_hash 管理） |
| [execution_control.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/commands/execution_control.py) | st.switch_page / st.rerun 实现 |
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/app_session.py) | 应用会话 |
| [AppNavigation.ts](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/frontend/app/src/util/AppNavigation.ts) | 前端导航逻辑 |
| [context_util.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/runtime/context_util.py) | URL 路径处理工具 |
| [local_sources_watcher.py](file:///d:/fz/0601/solo-dogfeeding/code/218-streamlit/lib/streamlit/watcher/local_sources_watcher.py) | 页面文件监听 |
