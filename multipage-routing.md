# Streamlit 多页应用页面发现与路由实现分析

## 概述

Streamlit 多页应用（Multipage App, MPA）经历了两个版本的演进：

- **v1（pages/ 目录模式）**：自动扫描 `pages/` 目录下的 Python 文件作为页面
- **v2（st.navigation 模式）**：通过 `st.navigation()` + `st.Page()` 显式声明页面

v2 是当前的推荐方式，提供了更灵活的页面配置和动态导航能力。

---

## 一、三层标识体系：url_path、page_script_hash 与 main_script_hash

### 1.1 概念总览

每个页面有两个核心标识，再加整个应用入口的一个标识，共三层：

| 标识 | 含义 | 存在范围 | 示例 |
|------|------|----------|------|
| **url_path (url_pathname)** | 对外暴露的 URL 路径名，用户在浏览器地址栏看到的相对路径 | 页面级 | `""`、`"dashboard"`、`"settings"` |
| **page_script_hash** | 由 url_path 哈希而来，前后端通信时的内部页面标识 | 页面级 | `"0e5751c2..."` (32 位 BLAKE2b) |
| **main_script_hash** | 整个应用主入口脚本的哈希（v2 固定不变，v1 为主脚本内容哈希） | 应用级 | `"d4a7d3a1..."` |

### 1.2 url_path → page_script_hash 的计算

**核心代码**：
- `StreamlitPage._script_hash` 属性：`page.py` 中 `_script_hash = calc_hash(url_path)`
- `calc_hash()`：`util.py` 中使用 **BLAKE2b**，输出 32 位十六进制（16 字节）

**哈希函数**：
```python
def calc_hash(s: bytes | str) -> str:
    b = s.encode("utf-8") if isinstance(s, str) else s
    h = hashlib.blake2b(digest_size=16, usedforsecurity=False)
    h.update(b)
    return h.hexdigest()
```

### 1.3 默认页（根路径）的特殊对应关系

默认页有三条硬性规则，来自 `StreamlitPage.url_path` property：

```python
@property
def url_path(self) -> str:
    return "" if self._default else self._url_path
```

| 规则 | 说明 |
|------|------|
| **url_path 强制为空字符串** | 只要 `_default=True`，无论 `url_path=` 参数传什么，对外暴露的 `url_path` 都是 `""` |
| **page_script_hash = calc_hash("")** | 因 url_path 固定为 `""`，默认页的内部标识始终是空串的哈希 |
| **用户自定义 url_path 被忽略** | 若设置 `default=True`，`url_path` 参数即使传了也不生效，文档明确提示 |

> ⚠️ **对应关系本质**：用户说的「根路径」就是 `url_path=""`，对应到后端内部就是 `calc_hash("")`，这两个值在整个 v2 系统中一一对应。页面列表中只有一个页面可以拥有 `url_path=""`。

### 1.4 非默认页的推导链

以一个文件页为例：
```python
st.Page("reports/sales_dashboard.py", url_path="sales")
```

推导链：
```
文件路径: reports/sales_dashboard.py
            ↓ 从文件名推断或用户指定
url_path:  "sales"            （去掉 .py，_sanitize_url_path 小写化、去特殊字符）
            ↓ calc_hash
page_script_hash:  "a1b2c3..."  （BLAKE2b 32 位 hex）
```

**重复检查**：`st.navigation()` 在构建 `pagehash_to_pageinfo` 字典时，若发现两个 page 计算出相同的 `script_hash`（即 url_path 相同），会抛出异常：
```
"Multiple Pages specified with URL pathname ... URL pathnames must be unique."
```

---

## 二、页面扫描与发现机制

### 2.1 v1: pages/ 目录自动扫描

在 v1 模式下，Streamlit 会自动扫描主脚本同级目录下的 `pages/` 文件夹，将其中的 `.py` 文件识别为页面。

**页面命名规则**（`PAGE_FILENAME_REGEX = ([0-9]*)[_ -]*(.*)\.py`）：
- `01_🚀_Home.py` → 排序号 `01`，图标 `🚀`，名称 `Home`
- `About_Us.py` → 无排序号，名称 `About_Us`

**关键函数**：

| 函数 | 作用 |
|------|------|
| `page_icon_and_name()` | 从文件名提取图标和名称 |
| `page_sort_key()` | 生成排序键（数字前缀 + 名称） |

**v1 检测逻辑**（`PagesManager.__init__`）：
```python
if PagesManager.uses_pages_directory is None:
    PagesManager.uses_pages_directory = Path(
        self.main_script_parent / "pages"
    ).exists()
```

### 2.2 v2: st.Page 显式声明

v2 模式下，页面通过 `st.Page()` 显式创建，不再依赖目录扫描。页面源可以是三种类型：

1. **Python 文件路径**（`str` 或 `Path`）
2. **可调用对象**（`Callable`，如函数）
3. **外部 URL**（`"https://..."`）

**StreamlitPage 初始化流程**：

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
    计算内部 _url_path（从文件名推断，或使用 url_path= 参数）
         │
         ▼
    运行时访问 url_path property:
      ├─ default=True  → 返回 ""
      └─ default=False → 返回内部 _url_path
         │
         ▼
    计算 page_script_hash = calc_hash(url_path)
```

**URL 路径清理函数** `_sanitize_url_path()`：
- 转小写
- 空白字符替换为下划线
- 移除 `& # ? / \ : * " < > | '` 等特殊字符
- 合并连续下划线

---

## 三、导航注册逻辑

### 3.1 st.navigation 入口

`st.navigation()` 是 v2 多页应用的核心入口，做了以下关键事情：

1. **禁用 v1 模式**：设置 `PagesManager.uses_pages_directory = False`
2. **页面类型转换**：将各种 page-like 对象转为 `StreamlitPage`
3. **默认页确定**：查找 `default=True` 的页面，没有则取第一个非外部页面
4. **构建页面注册表**：`pagehash_to_pageinfo` 字典
5. **发送导航 proto 消息**：通知前端导航菜单配置
6. **解析当前页面**：调用 `set_pages_and_resolve()` 确定要运行的页面
7. **返回当前页面对象**：用户调用 `.run()` 执行页面

### 3.2 默认页确定流程（关键）

```python
# 第 1 轮: 遍历找 _default=True 的页面
for page in all_pages:
    if page._default:
        # 冲突检测：只能有一个
        if default_page is not None:
            raise "Multiple Pages specified with default=True"
        default_page = page

# 第 2 轮: 没找到 → 取第一个非外部页面，并强制设为 default
if default_page is None:
    non_external_pages = [p for p in page_list if not p.is_external]
    default_page = non_external_pages[0]
    default_page._default = True   # ← 这里修改了 _default 属性
```

> **重要**：一旦某页被选为默认页，它的 `_default` 就变为 `True`，从而 `url_path` property 开始返回 `""`，它的 `_script_hash` 就变为 `calc_hash("")`。也就是说，**默认页在运行时的内部标识取决于「谁被选中」，而不是「用户传了什么参数」。**

### 3.3 导航注册完整流程

```
st.navigation(pages)
    │
    ├─ 1. 转换页面类型 convert_to_streamlit_page()
    │     ├─ StreamlitPage → 直接使用
    │     ├─ str/Path → new StreamlitPage(page)
    │     └─ Callable → new StreamlitPage(page)
    │
    ├─ 2. 确定默认页（见上文）
    │
    ├─ 3. 构建 pagehash_to_pageinfo 映射
    │     └─ key:   page._script_hash  (= calc_hash(page.url_path))
    │        value: {page_script_hash, page_name, icon, script_path, url_pathname}
    │        （同名 url_path 哈希冲突 → 抛异常）
    │
    ├─ 4. 构建 Navigation proto 消息
    │     ├─ position: sidebar/top/hidden
    │     ├─ sections: 分组标题列表
    │     └─ app_pages: 每个页面写入 {page_script_hash, page_name, icon,
    │                  is_default, section_header, url_pathname, is_hidden}
    │
    ├─ 5. ctx.pages_manager.set_pages_and_resolve()
    │     └─ 设置页面注册表并解析当前页面（见第四章）
    │
    ├─ 6. 找到实际返回的页面对象 → 标记 _can_be_called = True
    │     └─ 外部页面不能直接通过 URL 访问 → 回退到默认页 + PageNotFound
    │
    ├─ 7. msg.navigation.page_script_hash = 即将执行的页面哈希
    │     ctx.set_mpa_v2_page(...)  记录到 ScriptRunContext
    │
    └─ 8. ctx.enqueue(msg) 发送导航消息给前端
```

### 3.4 PagesManager 页面管理器

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
| `set_script_intent(hash, name)` | 设置意图运行的页面（哈希 + 名称），每次 rerun 前被调用 |
| `set_pages_and_resolve(registry, fallback)` | 原子性设置页面注册表并解析当前页面 |
| `_resolve_page_script()` | 根据意图解析实际要运行的页面 |
| `get_initial_active_script()` | 获取初始活动脚本（v2 始终运行主脚本） |
| `get_page_script_byte_code()` | 从脚本缓存获取页面字节码 |

---

## 四、路径匹配与路由切换

### 4.1 路由解析核心：_resolve_page_script()

`PagesManager._resolve_page_script()` 是后端路由解析的唯一入口。

**完整优先级链**：
```
解析优先级（从高到低）：

1. _intended_page_script_hash（精确匹配，来自 RerunData.page_script_hash）
   │
   ├─ 在 _pages 字典中查找 hash → 找到 → 返回该 PageInfo
   │                           → 没找到 → 进入 fallback
   │
2. _intended_page_name（模糊匹配，来自 RerunData.page_name）
   │
   └─ 遍历 _pages.values()，找到 url_pathname == page_name 的 → 返回
        遍历完毕没找到 → 进入 fallback

3. fallback_page_hash（默认页哈希，由 st.navigation 传入的 default_page._script_hash）
   │
   └─ 返回默认页 PageInfo
```

> **重点**：`page_script_hash` 优先于 `page_name`。前者是精确哈希，允许重名页面被区分；后者是从 URL 路径名推断，只在首次访问还没建立哈希表时使用。

### 4.2 直接访问 URL：前后端定位流程（核心）

#### 场景：用户在浏览器输入 `http://app.example.com/dashboard` 首次打开应用

**Step 1 — 前端启动，还未建立 WebSocket**
- `ConnectionManager` 读取 `document.location.pathname` = `/dashboard`
- 还未从后端收到 `appPages` 列表，**不知道 `/dashboard` 对应的 `pageScriptHash`**

**Step 2 — 首次 `sendRerunBackMsg()` 触发时的分支判断**
（在 `App.tsx` `sendRerunBackMsg` 方法中）

```typescript
if (pageScriptHash) {
    // 分支 A：用户明确指定了 pageScriptHash（点击导航菜单时走这里）
} else if (currentPageScriptHash) {
    // 分支 B：已有 currentPageScriptHash（点击 "Rerun" 按钮时走这里）
} else {
    // 分支 C：currentPageScriptHash 为空 → 首次加载 / 直访 URL
    // 此时还没收到后端的页面注册表，只能用 URL 字符串推断
    pageName = extractPageNameFromPathName(
        document.location.pathname,  // "/dashboard"
        baseUriParts.pathname         // "/" 或应用部署前缀
    )
    // 结果 pageName = "dashboard"
    pageScriptHash = ""
}
```

**Step 3 — `extractPageNameFromPathName()` 算法**
```
输入: pathname = "/myapp/dashboard", basePath = "/myapp"

  1. pathname.replace(basePath, "")    →  "/dashboard"
  2. replace(new RegExp("^/?"), "")    →   "dashboard"
  3. replace(new RegExp("/$"), "")     →   "dashboard"  （去掉末尾斜杠）
  4. decodeURIComponent(...)           →   "dashboard"  （解码中文等特殊字符）
```
- 根路径访问 `"/"` 时，三步替换后得到 `""`
- 返回值就是「对外 url_path」

**Step 4 — 前端组装 BackMsg 发给后端**
```protobuf
BackMsg.rerunScript: {
    pageScriptHash: "",          // 空！（首次加载无法计算）
    pageName:       "dashboard", // 从 URL 提取的字符串
    queryString:    "...",
    widgetStates:   {...}
}
```

**Step 5 — 后端 ScriptRunner 处理 RerunData**
1. `PagesManager.set_script_intent(page_script_hash="", page_name="dashboard")`
   → `_intended_page_script_hash = ""`, `_intended_page_name = "dashboard"`

2. 开始执行主脚本，主脚本调用 `st.navigation(...)`

3. `st.navigation` 内部调用 `set_pages_and_resolve(pagehash_to_pageinfo, fallback=default_hash)`

4. 进入 `_resolve_page_script()`：
   - 第 1 级：`_intended_page_script_hash = ""` → 空字符串不在字典里，跳过
   - 第 2 级：`_intended_page_name = "dashboard"` → 遍历值
     ```python
     for page_info in _pages.values():
         if page_info["url_pathname"] == "dashboard":
             return page_info  # ← 匹配成功！
     ```
   - 返回该页面对应的 `PageInfo`

5. 如果用户访问的是根路径 `/`：
   - `pageName = ""` → 遍历 `url_pathname == ""` → 命中默认页 → 返回默认页
   - 或在第 2 级也没匹配（比如 url_pathname 根本不是空串）→ fallback 到默认页哈希 → 仍是默认页

**Step 6 — 后端返回 NewSession + Navigation 消息**

NewSession 消息里携带：
```protobuf
msg.new_session.page_script_hash = resolved_page_script_hash  // 实际解析到的页面哈希
msg.new_session.main_script_hash = main_script_hash           // 应用级，固定
```

Navigation 消息里携带所有页面：
```protobuf
msg.navigation.page_script_hash = resolved_page_script_hash   // 当前执行的页面
msg.navigation.app_pages = [
    { pageScriptHash: "0e57...", urlPathname: "",        isDefault: true,  ... },
    { pageScriptHash: "a1b2...", urlPathname: "dashboard", isDefault: false, ... },
    ...
]
```

**Step 7 — 前端收到，建立哈希表与当前页记忆**
- `appPages = navigationMsg.appPages` — 现在知道每个 `urlPathname` 对应的 `pageScriptHash`
- `currentPageScriptHash = navigationMsg.pageScriptHash`
- 下次用户点击导航菜单、点击 `st.page_link`、浏览器前进后退时，前端**不再需要 page_name**，可以直接用 `pageScriptHash` 发起 rerun

#### 场景汇总表

| 触发方式 | 前端取值 | 发送给后端的 RerunData | 后端 _resolve_page_script 命中的级别 |
|----------|----------|------------------------|--------------------------------------|
| 首次直访根路径 | pathname → `""` | `pageScriptHash=""`, `pageName=""` | 通常 fallback 或级别 2 匹配空串 |
| 首次直访 `/dashboard` | pathname → `"dashboard"` | `pageScriptHash=""`, `pageName="dashboard"` | 级别 2 url_pathname 匹配 |
| 点击侧边导航菜单 | 知道 `pageScriptHash` | `pageScriptHash="a1b2..."`, `pageName=""` | 级别 1 精确哈希匹配 |
| 点击 `st.page_link` | 知道 `pageScriptHash` | `pageScriptHash="a1b2..."`, `pageName=""` | 级别 1 精确哈希匹配 |
| 浏览器前进/后退（popstate）| `findPageByUrlPath(pathname)` → 拿到 `pageScriptHash` | `pageScriptHash="a1b2..."`, `pageName=""` | 级别 1 精确哈希匹配 |
| 「重新运行」按钮 | 取 `currentPageScriptHash` | 发送当前页哈希 | 级别 1 精确哈希匹配 |
| `st.switch_page(...)` | 后端直接设置 RerunData | `page_script_hash=目标哈希` | 级别 1 精确哈希匹配 |

### 4.3 前端 URL 反向查找：findPageByUrlPath

用于浏览器前进/后退按钮，收到 `popstate` 事件时：

```typescript
// onHistoryChange 中
targetAppPage = this.appNavigation.findPageByUrlPath(document.location.pathname)
```

算法（`AppNavigation.findPageByUrlPath`）：
```
1. decodedPathname = decodeURIComponent(document.location.pathname)

2. 遍历 appPages：
   for (p of appPages) {
       if (decodedPathname.endsWith("/" + p.urlPathname)) {
           return p
       }
       // 特殊情况：默认页 urlPathname="" → "/..." + "" → endsWith("/") 不匹配
       // 这时候用 mainPage 兜底
   }

3. 遍历没找到 → 返回 mainPage（即默认页）
```

> **默认页匹配原理**：默认页 `urlPathname=""`，`endsWith("/" + "")` = `endsWith("/")`。
> - 访问 `/dashboard` → 匹配不到 → 返回 `mainPage`? 不——先匹配到了非默认页的 `/dashboard`
> - 访问 `/` → 遍历所有非默认页都不匹配 → 返回 `mainPage`（默认页）
> - 访问 `/unknown` → 遍历所有非默认页都不匹配 → 返回 `mainPage`（默认页）

### 4.4 切页触发方式

#### 方式 1: 用户点击导航菜单 / page_link
- 前端发送 `SET_CURRENT_PAGE_NAME` 消息（`currentPageName + currentPageScriptHash`）
- 最终转化为 `sendRerunBackMsg`，携带 `pageScriptHash`
- 后端触发 rerun，RerunData 带 page_script_hash → 级别 1 匹配

#### 方式 2: st.switch_page() 编程式切换

```python
def switch_page(page, *, query_params=None):
    # 1. 解析目标页面，获取 page_script_hash
    #    - StreamlitPage 对象: 直接取 _script_hash
    #    - 文件路径: 在 pages_manager 中查找匹配的 script_path
    #
    # 2. 设置 query_params（默认为空 dict，清除原有参数）
    #
    # 3. 请求 rerun，携带 page_script_hash
    ctx.script_requests.request_rerun(
        RerunData(page_script_hash=page_script_hash, ...)
    )
```

#### 方式 3: 浏览器直接访问 URL
见本章 4.2 节的完整流程。

### 4.5 脚本运行流程中的路由

切页时的脚本运行流程（`script_runner.py` 核心片段）：

```
收到 RERUN 请求 (携带 page_script_hash, page_name)
    │
    ├─ 1. pages_manager.set_script_intent(hash, name)
    │     └─ _intended_page_script_hash = hash
    │        _intended_page_name       = name
    │
    ├─ 2. pages_manager.get_initial_active_script()
    │     └─ v2 模式下始终返回主脚本（入口文件）
    │
    ├─ 3. previous_page_script_hash != page_script_hash → 页面变化
    │     ├─ 过滤 query_params（只保留主脚本和新页面的）
    │     └─ 触发 widget 状态清理条件
    │
    ├─ 4. ctx.reset()
    │     ├─ 设置当前 page_script_hash（从 RerunData 来）
    │     ├─ 重置 widget/form/fragment 追踪
    │     └─ 初始化 ThreadState
    │
    └─ 5. 执行主脚本
          └─ 主脚本中调用 st.navigation()
                → 内部 set_pages_and_resolve() → 真正解析页面
                → 返回当前 StreamlitPage → pg.run()
                → pg.run() 内部用 run_with_active_hash 执行页面代码
```

**v2 关键设计**：每次 rerun 都执行**主脚本**（入口文件），页面逻辑通过 `pg.run()` 在主脚本内部调用。这与 v1 模式直接执行页面脚本不同。v1 模式在这一步会切换 `get_initial_active_script()` 返回不同的页面脚本。

---

## 五、切页状态保持机制

### 5.1 Session State 的跨页保持

`st.session_state` 在同一会话内是**全局共享**的，跨页面切换会保持。

**证据**：
- 会话状态存储在 `AppSession` 中，而不是单个页面
- 页面切换只是脚本重新执行，不会创建新会话
- 入口文件中定义的带 key widget 在跨页时保持状态

**推荐实践**：
```python
# 入口文件 streamlit_app.py
st.sidebar.selectbox("Foo", ["A", "B", "C"], key="foo")  # 跨页保持
pg = st.navigation([page1, page2])
pg.run()
```

### 5.2 Widget 状态的清理

页面切换时，不在新页面上的 widget 状态会被清理。

**清理机制**（`session_state.py: remove_stale_widgets()`）：
- 每次脚本运行后，比较 `active_widget_ids` 和现有状态
- 不在活跃列表中的 widget 状态被移除
- 入口文件中的 widget 每次都运行，因此状态保持

**切页时的显式清理**：
- `previous_page_script_hash != page_script_hash` 时判定页面已变化
- 使用 rerun_data 中的 widget 状态来维护部分活跃 widget 状态
- 前端 `onPageChange` 中提前计算 `activeWidgetIds = activeWidgetIdsFromMainScriptOnly`

### 5.3 Query Parameters 的处理

页面切换时 query params 会被**过滤**，避免不同页面的参数相互干扰。

**核心逻辑**（`script_runner.py` 中）：
```python
# 仅保留主脚本和目标页面的 query params
valid_script_hashes = {main_script_hash, page_script_hash}
with self._session_state.query_params() as qp:
    qp.populate_from_query_string(
        rerun_data.query_string, valid_script_hashes
    )
    qp.set_initial_query_params_from_current()
```

**同页 rerun vs 跨页切换**：
- **同页 rerun**（widget 交互）：从 URL 填充 query params
- **跨页切换**：在 script_runner 中提前过滤，防止旧页面参数污染新页面

### 5.4 active_script_hash 与元素归属

Streamlit 使用 `active_script_hash` 来标记每个消息（元素）属于哪个页面。

**核心机制**（`script_run_context.py` 中）：
```python
def enqueue(self, msg: ForwardMsg) -> None:
    msg.metadata.active_script_hash = ThreadState.get().active_script_hash
    ...
```

**页面执行时的 hash 切换**（`StreamlitPage.run()`）：
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

> **重要**：主脚本（入口文件）中在 `st.navigation()` 之前创建的元素，其 `active_script_hash = main_script_hash`。这意味着这些元素「不属于任何具体页面」，切页时不会被清理。

**前端元素清理**：
```typescript
clearPageElements(elements: AppRoot, mainScriptHash: string): AppRoot {
  return elements.filterMainScriptElements(mainScriptHash)
  // 过滤掉 active_script_hash != main_script_hash 的所有元素
  // 即保留主脚本产生的元素，清除掉旧页面代码中产生的元素
}
```

这确保了切页时，旧页面的 UI 元素被正确清理，而主脚本（入口文件）的元素保留。

### 5.5 状态保持总结表

| 状态类型 | 跨页是否保持 | 说明 |
|----------|-------------|------|
| `st.session_state` | ✅ 是 | 会话级全局状态 |
| 入口文件中（`pg.run()` 之前）的 widget | ✅ 是 | 每次 rerun 都重新执行，且元素 active_script_hash = main_script_hash，不被清理 |
| 入口文件中（`pg.run()` 之后）的 widget | ✅ 是 | 同上 |
| 页面文件 / Callable 内的 widget | ❌ 否 | 切页后 active_script_hash 变化，对应元素全部清理 |
| `st.query_params` | ⚠️ 过滤 | 仅保留主脚本和当前页面绑定的参数 |
| Fragment 状态 | ❌ 否 | 切页后 fragment 不再活跃 |
| 上传文件 | ✅ 是 | `UploadedFileManager` 是会话级的 |

---

## 六、默认页回退逻辑全景

### 6.1 何时触发回退

| 回退场景 | 触发位置 | 表现 |
|----------|---------|------|
| 访问不存在的 URL 路径 | `_resolve_page_script` 级别 1、2 都不匹配 | 返回默认页，前端可选显示 PageNotFound |
| 默认页选择时 pages 列表为空 | `_navigation` → `non_external_pages[0]` | 无，前面已抛异常 "At least one non-external page is required" |
| 外部页面被直接 URL 访问 | `_navigation` → `page_to_return.is_external` → `send_page_not_found` | 显示 PageNotFound，返回默认页执行 |
| Navigation 消息 pageScriptHash 在前端找不到 | `handleNavigation` 中 `find(...)` 失败，TypeScript 断言 | 实际会 `as IAppPage`，应配合后端保证一致 |
| popstate 时 pathname 匹配不到任何页面 | `findPageByUrlPath` 遍历完没匹配 | 返回 `mainPage`（默认页） |
| 意图 page_script_hash 不在注册表 | `_resolve_page_script` 级别 1 匹配失败 | fallback 到级别 2 → 仍失败 → 返回默认页哈希 |

### 6.2 回退链路图

```
用户访问 /foo/bar（不存在的页面）
    │
    ▼
前端 extractPageNameFromPathName("/foo/bar", "/") → "foo/bar"
sendBackMsg: pageScriptHash="", pageName="foo/bar"
    │
    ▼
后端 set_script_intent("", "foo/bar")
主脚本执行 → st.navigation(...)
    │
    ▼
_pages 注册表中没有 url_pathname="foo/bar"
级别 1: _intended_page_script_hash = "" → 跳过
级别 2: 遍历 _pages.values() 全部不匹配 "foo/bar"
    │
    ▼
fallback: 返回 default_page._script_hash  (= calc_hash(""))
    │
    ▼
found_page = 默认页的 PageInfo
匹配到 StreamlitPage → 标记 _can_be_called = True
msg.navigation.page_script_hash = 默认页哈希
    │
    ├─ 分支 A: 原本意图为空 / 根路径
    │    → 直接执行默认页，不报错
    │
    └─ 分支 B: 原本意图有值但匹配失败
         → 发送 PageNotFound 消息给前端
         → 前端显示 "You have requested page /foo/bar, but no corresponding file was found..."
         → 仍然执行默认页（保证应用不会白屏）
```

---

## 七、核心数据结构

### 7.1 PageInfo（页面信息）

```python
class PageInfo(TypedDict):
    script_path: str                # Python 文件路径（v2 的 Callable 页为空）
    page_script_hash: str           # = calc_hash(url_pathname)
    icon: NotRequired[str]          # 图标字符串
    page_name: NotRequired[str]     # 显示标题
    url_pathname: NotRequired[str]  # 对外 URL 路径名（默认页为 ""）
```

### 7.2 StreamlitPage（页面对象）

**核心属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `_page` | `Path \| Callable \| None` | 页面源（文件或函数；外部 URL 为 None） |
| `_title` | `str` | 页面标题 |
| `_icon` | `str` | 页面图标 |
| `_url_path` | `str` | 内部存储的 URL 路径（非默认页原始值） |
| `_default` | `bool` | 是否默认页 |
| `_visibility` | `"visible" \| "hidden"` | 导航可见性 |
| `_external_url` | `str \| None` | 外部 URL（非空则为外链页） |
| `_can_be_called` | `bool` | 是否可执行（由 st.navigation 授权） |
| `url_path` (property) | `str` | **对外**：`_default ? "" : _url_path` |
| `_script_hash` (property) | `str` | 内部标识：`calc_hash(url_path)` |

### 7.3 RerunData（重运行数据）

```python
@dataclass(frozen=True)
class RerunData:
    query_string: str = ""                        # ? 后面的部分
    widget_states: WidgetStates | None = None     # 前端 widget 状态
    page_script_hash: str = ""                    # 内部页面标识（精确匹配）
    page_name: str = ""                           # 对外 URL 路径名（模糊匹配）
    fragment_id: str | None = None                # fragment 局部刷新
    cached_message_hashes: set[str] = field(...)  # 前端缓存的消息哈希
    ...
```

---

## 八、关键文件索引

| 文件（相对项目根） | 职责 |
|------|------|
| `lib/streamlit/commands/navigation.py` | `st.navigation` 命令实现，默认页确定，注册表构建 |
| `lib/streamlit/navigation/page.py` | `StreamlitPage` 类，url_path / _script_hash property 定义 |
| `lib/streamlit/runtime/pages_manager.py` | 页面管理器，`_resolve_page_script` 三级匹配，`set_script_intent` |
| `lib/streamlit/source_util.py` | 页面名称/图标提取工具，v1 目录扫描辅助 |
| `lib/streamlit/util.py` | `calc_hash()` BLAKE2b 哈希函数 |
| `lib/streamlit/runtime/scriptrunner/script_runner.py` | 脚本运行器，切页时 active_script_hash 切换、query_param 过滤 |
| `lib/streamlit/runtime/scriptrunner_utils/script_run_context.py` | `enqueue()` 给每条消息打 `active_script_hash` 标签 |
| `lib/streamlit/commands/execution_control.py` | `st.switch_page` / `st.rerun` 实现 |
| `lib/streamlit/runtime/app_session.py` | `_create_new_session_message` 组装 `NewSession` 消息，带 `page_script_hash`、`main_script_hash` |
| `frontend/app/src/util/AppNavigation.ts` | `handleNavigation`、`findPageByUrlPath`、`clearPageElements`，前端侧页面查找与清理 |
| `frontend/app/src/App.tsx` | `sendRerunBackMsg` 三大分支、`onHistoryChange` 监听 popstate、`extractPageNameFromPathName` |
| `frontend/lib/src/util/utils.ts` | `extractPageNameFromPathName()` URL 路径名提取算法 |
| `lib/streamlit/runtime/context_util.py` | URL 路径处理工具 |
| `lib/streamlit/watcher/local_sources_watcher.py` | 页面文件监听（v1 模式下的 pages/ 目录变更） |
