# Streamlit 多页应用核心标识与路由机制

本文聚焦解答三个核心疑问：

1. **默认页为什么对外显示为空 URL 路径？这个空路径从哪来？**
2. **页面内部标识 `page_script_hash` 和应用级 `main_script_hash` 分别基于什么计算？二者为什么不能混淆？**
3. **直接访问不存在的路径为什么必然返回默认页？回退判定在代码里的具体位置？**

---

## 一、三层标识全景：先搞清楚每一层的输入源

| 层级 | 标识名 | 面向谁 | 计算输入 | 代码位置 | 示例值 |
|------|--------|--------|----------|---------|--------|
| ① 对外层 | **`url_path`（url_pathname）** | 浏览器用户 / 地址栏 | 页面标题 / 文件名 / 用户传参（默认页强制 `""`） | `StreamlitPage.url_path` property | `""`、`"dashboard"`、`"user_settings"` |
| ② 页面内部层 | **`page_script_hash`** | 前后端通信 / 页面注册表 key | **`calc_hash(url_path)`** | `StreamlitPage._script_hash` property，实际调用 `calc_hash()` | 32 位十六进制（BLAKE2b，16 字节） |
| ③ 应用层 | **`main_script_hash`** | 主脚本 / 公共元素标记 | **`calc_hash(入口文件绝对路径)`** | `PagesManager.__init__`，一次性计算 | 32 位十六进制（BLAKE2b，16 字节） |

> **核心原则**：
> - 对外层 ① 和 内部层 ② 是**强绑定关系**，一对一由哈希函数映射；
> - 应用层 ③ 与前两者**完全无关**，它是对「入口文件路径」哈希，不是对 URL 路径哈希。
> - 三者都使用同一个 `calc_hash()` 函数，但输入源完全不同。

### 1.1 统一哈希函数 `calc_hash()`

```python
def calc_hash(s: bytes | str) -> str:
    """BLAKE2b 快速哈希，输出 32 位十六进制（16 字节）。"""
    b = s.encode("utf-8") if isinstance(s, str) else s
    h = hashlib.blake2b(digest_size=16, usedforsecurity=False)
    h.update(b)
    return h.hexdigest()
```

**关键特性**：
- 输入相同 → 输出必然相同；输入不同 → 输出概率上不同
- 字符串走 UTF-8 编码再哈希
- 非安全用途，所以 `usedforsecurity=False`，性能优先

---

## 二、对外层：为什么默认页的 `url_path` 必须是空字符串

### 2.1 代码证据：`StreamlitPage.url_path` property

```python
# StreamlitPage 类内部
@property
def url_path(self) -> str:
    """对外暴露的 URL 路径名。默认页恒返回 ""。"""
    return "" if self._default else self._url_path
```

这里是三层判断：

| 判断条件 | 结果 | 说明 |
|----------|------|------|
| `self._default == True` | 返回 `""` | **强制覆盖**，不管内部 `_url_path` 原本存的是什么 |
| `self._default == False` | 返回 `self._url_path` | 使用初始化时推导或用户指定的路径 |

### 2.2 `_default` 标记是怎么来的

`_default` 的来源有两条路径：

**路径 A：用户显式声明**
```python
home_page = st.Page("views/home.py", title="Home", default=True)
# ↑ home_page._default = True
```

**路径 B：`st.navigation()` 内部自动回退（最常见）**

```python
# 第 1 轮：遍历找用户已显式声明 default=True 的页面
for page in all_pages:
    if page._default:
        if default_page is not None:
            raise StreamlitAPIException("Multiple Pages specified with default=True")
        default_page = page

# 第 2 轮：没有用户声明 → 取第一个非外部页面，强制 _default=True
if default_page is None:
    non_external_pages = [p for p in page_list if not p.is_external]
    default_page = non_external_pages[0]
    default_page._default = True   # ← 就地修改对象属性
```

> **为什么大多数开发者「没写 default=True 也有默认页」？**
> 因为路径 B 是兜底逻辑，90% 的应用不显式声明 `default=True`，在 `st.navigation()` 里直接被「第一个非外部页面」拿走了默认位，它的 `url_path` property 随即开始返回 `""`。

### 2.3 `default=True` 时 `url_path=` 参数被忽略的证据

文档字符串（`Page()` 函数 docstring）明确写着：

> `url_path` can't include forward slashes; paths can't include subdirectories.
> The default page will have a pathname of `""`, indicating the root URL of the app.
> **If you set `default=True`, `url_path` is ignored.**
> If `default` is `True`, then the page will have an empty pathname and **`url_path` will be ignored**.

初始化逻辑也有对应校验：
```python
if stripped_url_path.strip() == "" and not default:
    raise StreamlitAPIException(
        "The URL path cannot be an empty string unless the page is the default page."
    )
```

即：**空串 `""` 只有默认页能持有**，非默认页如果想设空路径会报错。这从 API 层面就保证了 `url_path=""` 唯一标识默认页。

---

## 三、内部层：`page_script_hash` 如何由对外路径映射而来

### 3.1 计算链：`url_path → calc_hash() → page_script_hash`

```python
# StreamlitPage 类内部（注意这是 property，不是普通属性）
@property
def _script_hash(self) -> str:
    return calc_hash(self.url_path)    # ← 注意调用的是 url_path property！
```

**关键点**：这里调用的是 **`self.url_path`（property）**，不是 `self._url_path`（内部字段）。

这意味着什么？

| 页面 | 内部字段 `_url_path` | `_default` | `self.url_path`（property） | `_script_hash` |
|------|---------------------|------------|------------------------------|----------------|
| 默认页 | `"home"`（从文件名推导） | `True` | `""`  ← 被覆盖 | `calc_hash("")` |
| Dashboard 页 | `"sales_dashboard"` | `False` | `"sales_dashboard"` | `calc_hash("sales_dashboard")` |
| Settings 页 | `"settings"` | `False` | `"settings"` | `calc_hash("settings")` |

> ⚠️ **最容易混淆的点**：默认页在内部字段 `_url_path` 里可能存着 `"home"`（从 `home.py` 推导来的），但真正参与哈希计算的是 `self.url_path` property 返回的 `""`。**对外展示给用户看的是什么字符串，内部就对什么字符串哈希**。

### 3.2 注册表构建：`pagehash_to_pageinfo`

在 `st.navigation()` 内部，用 `_script_hash` 作为 key 构建字典：

```python
pagehash_to_pageinfo: dict[PageHash, PageInfo] = {}

for page in nav_sections.flat_values():
    script_hash = page._script_hash     # = calc_hash(page.url_path)
    if script_hash in pagehash_to_pageinfo:
        raise StreamlitAPIException(
            f"Multiple Pages specified with URL pathname {page.url_path}. "
            "URL pathnames must be unique."
        )
    pagehash_to_pageinfo[script_hash] = {
        "page_script_hash": script_hash,
        "page_name": page.title,
        "icon": page.icon,
        "script_path": str(page._page) if isinstance(page._page, Path) else "",
        "url_pathname": page.url_path,   # ← 存的就是对外路径 "" 或 "xxx"
    }
```

**冲突检测原理**：如果两个页面算出来 `script_hash` 相同，意味着它们的 `url_path` 相同 → 报「URL pathnames must be unique」错误。哈希在这里其实是「路径唯一」的替身检查。

---

## 四、应用层：`main_script_hash` 为什么和页面没关系

### 4.1 计算来源

```python
# PagesManager.__init__
def __init__(self, main_script_path: ScriptPath, ...):
    self._main_script_path = main_script_path
    self._main_script_hash: PageHash = calc_hash(main_script_path)  # ← 入口文件路径！
    ...
```

**输入是「入口 Python 文件的路径」**，比如 `D:\myapp\streamlit_app.py`。和任何页面的 `url_path` 都没关系，和 `page_script_hash` 更不会冲突（输入集合完全不同，概率上不碰撞）。

### 4.2 它的用途：标记公共元素归属

```python
# ScriptRunContext.enqueue() — 每一条发往前端的 ForwardMsg 都打标签
def enqueue(self, msg: ForwardMsg) -> None:
    msg.metadata.active_script_hash = ThreadState.get().active_script_hash
    ...
```

- **入口文件执行期间**：`active_script_hash = main_script_hash`
  → 入口文件里创建的 `st.title()`、`st.sidebar.selectbox(...)` 等元素，都被标记为 `main_script_hash`
- **`pg.run()` 页面执行期间**：`active_script_hash = page._script_hash`
  → 页面文件里创建的元素，被标记为对应的 `page_script_hash`

**切页时前端清理逻辑**：
```typescript
clearPageElements(elements, mainScriptHash) {
  return elements.filterMainScriptElements(mainScriptHash)
  // → 保留所有 active_script_hash === mainScriptHash 的元素
  // → 清除其他（即属于具体页面的）元素
}
```

> 所以 `main_script_hash` 本质是「公共元素保护伞」：入口文件里写的 UI 跨页保留，页面文件里写的 UI 切页就扔。这就是 Streamlit 推荐把公共导航栏、公共筛选器放在入口文件的底层原因。

---

## 五、三层标识的对应关系总结表

以一个典型三页应用为例，入口文件 `streamlit_app.py`，页面分别是 `home.py`、`dashboard.py`、用户指定了 `url_path="settings"` 的第三页。

| 项目 | 默认页（home.py） | 第二页（dashboard.py） | 第三页（url_path="settings"） | 入口文件公共元素 |
|------|-------------------|------------------------|-------------------------------|------------------|
| **① 对外层**<br>`url_path` / `url_pathname` | `""`（被 property 强制覆盖） | `"dashboard"`（从文件名 dashboard.py 推导） | `"settings"`（用户指定） | —— 无 URL，不参与路由 |
| **② 内部层**<br>`page_script_hash` | `calc_hash("")` | `calc_hash("dashboard")` | `calc_hash("settings")` | —— 不参与路由 |
| **③ 应用层**<br>`main_script_hash` | —— 不使用 | —— 不使用 | —— 不使用 | `calc_hash("streamlit_app.py"的完整路径)` |
| 浏览器地址栏显示 | `http://app/` 或 `http://app` | `http://app/dashboard` | `http://app/settings` | —— |
| `pg.run()` 期间 active_hash | `calc_hash("")` | `calc_hash("dashboard")` | `calc_hash("settings")` | 入口执行期间 = main_script_hash |
| 切页时元素是否清理 | ❌ 清理（是具体页面元素） | ❌ 清理 | ❌ 清理 | ✅ 保留（是 main_script_hash） |

---

## 六、不存在路径为什么必然返回默认页：完整判定链

### 6.1 代码核心位置：`PagesManager._resolve_page_script()`

这是所有路由解析的唯一出口：

```python
def _resolve_page_script(self, fallback_page_hash: PageHash = "") -> PageInfo | None:
    # 条件 1：意图哈希不为空 → 精确哈希匹配，匹配不到就拿 fallback 兜底
    if self.intended_page_script_hash:
        return self._pages.get(
            self.intended_page_script_hash,
            self._pages.get(fallback_page_hash, None),  # ← 哈希匹配失败，回退
        )

    # 条件 2：意图名称不为空 → url_pathname 字符串精确匹配，匹配不到返回 None
    if self.intended_page_name:
        return next(
            filter(
                lambda p: p and (p["url_pathname"] == self.intended_page_name),
                self._pages.values(),
            ),
            None,  # ← 注意：字符串匹配失败时返回 None，没有 inline fallback
        )

    # 条件 3：意图和名称都为空（直接访问根路径）→ 直接取 fallback
    return self._pages.get(fallback_page_hash, None)
```

注意**条件 2 和条件 1 的不对称性**：
- 哈希匹配失败 → 代码内**立即** `dict.get(..., fallback)` 回退到默认页
- 字符串匹配失败 → 返回 `None`，要在上层判断后再走 fallback

### 6.2 上层调用：`set_pages_and_resolve()` 与最终回退位置

```python
# 在 _navigation() 里调用
found_page = ctx.pages_manager.set_pages_and_resolve(
    pagehash_to_pageinfo,
    fallback_page_hash=default_page._script_hash,  # ← = calc_hash("")
)

# --- 接下来是真正的回退逻辑 ---
page_to_return = None
if found_page:                            # 情况 A：_resolve_page_script 找到了东西
    found_page_script_hash = found_page["page_script_hash"]
    matching_pages = [p for p in page_list if p._script_hash == found_page_script_hash]
    if len(matching_pages) > 0:
        page_to_return = matching_pages[0]

# 情况 B：命中了外部 URL 页 → 外部页不能直接 URL 访问，置空
if page_to_return and page_to_return.is_external:
    page_to_return = None

# 情况 C：found_page 为 None，或匹配页面不存在，或外部页置空
if not page_to_return:                        # ← 回退最终触发点
    send_page_not_found(ctx)                  # ← 发送 PageNotFound 给前端
    page_to_return = default_page             # ← 兜底：使用默认页
```

三个条件形成「回退漏斗」：

```
_resolve_page_script() 返回值
    │
    ├─ None → 直接穿过，进入 not page_to_return 分支
    │
    └─ 非 None → 拿到 page_script_hash
         │
         ├─ 在 page_list（StreamlitPage 列表）里匹配不到 → page_to_return 仍为 None
         │
         ├─ 匹配到了但它是 external 页面 → page_to_return = None
         │
         └─ 匹配到且是内部页 → page_to_return = 该页（不回退）
```

因此**最终回退率 100%**：只要走了 `not page_to_return` 分支，就必然赋值 `default_page`。不可能有别的结果。

### 6.3 `send_page_not_found()` 什么时候触发，什么时候不触发

| 场景 | `page_to_return` 是否为 None | 触发 send_page_not_found？ | 用户体验 |
|------|------------------------------|----------------------------|---------|
| 访问根路径 `/` | ❌ 非 None（命中默认页 fallback） | **否** | 正常显示首页，无任何提示 |
| 访问存在的页面 `/dashboard` | ❌ 非 None | **否** | 正常显示对应页 |
| 访问不存在的页面 `/foo/bar` | ✅ None | **是** | 顶部提示「Page not found」，下方显示首页内容 |
| 直接访问外部页的 URL（如果有人猜到） | ✅ None（被 is_external 过滤） | **是** | 同上 |
| 意图哈希（page_script_hash）在注册表不存在 | ❌ 非 None（dict.get 的 fallback 返回默认页） | **否** | 静默跳首页 |
| `page_script_hash=""` 且 `page_name=""`（首次进入或刷新根页） | ❌ 非 None | **否** | 正常首页 |

> ⚠️ **最微妙的场景**：用户输入一个不存在的 `page_script_hash`（只有精通内部机制的人才能构造），条件 1 的 `dict.get()` 直接拿到 fallback 页 → 不触发 PageNotFound 提示，**静默跳到首页**。

### 6.4 完整链路图：用户访问 `/nonexistent`

```
Step 1: 浏览器发送 HTTP 请求，加载 index.html
Step 2: 前端 JS 启动 WebSocket 连接，读取 document.location.pathname
        → pathname = "/nonexistent"
        → 此时 currentPageScriptHash 为空（还没收到 Navigation 消息）

Step 3: 前端 sendRerunBackMsg() 走分支 C（currentPageScriptHash 为空）
        extractPageNameFromPathName("/nonexistent", basePath) → "nonexistent"
        BackMsg.rerunScript: { pageScriptHash: "", pageName: "nonexistent" }

Step 4: 后端 ScriptRunner 处理
        set_script_intent(page_script_hash="", page_name="nonexistent")
        → _intended_page_script_hash = ""
        → _intended_page_name = "nonexistent"

Step 5: 执行入口文件 → st.navigation()
        构建 pagehash_to_pageinfo 注册表，fallback = default_page._script_hash = calc_hash("")
        调用 set_pages_and_resolve(registry, fallback)

Step 6: _resolve_page_script() 内部判定
        if intended_page_script_hash: "" → 条件 1 跳过
        if intended_page_name:     "nonexistent" → 进入条件 2
            filter(lambda p: p["url_pathname"] == "nonexistent", registry.values())
            所有条目的 url_pathname 分别是 ""、"dashboard"、"settings"
            → 没有匹配项，filter 返回空迭代器
            → next(..., None) → 返回 None

Step 7: 上层拿到 found_page = None
        page_to_return 初始化 = None
        跳过 found_page 的匹配分支

        进入：if not page_to_return:
            → send_page_not_found(ctx)  # 发送 PageNotFound ForwardMsg
            → page_to_return = default_page  # 兜底赋值

Step 8: msg.navigation.page_script_hash = default_page._script_hash
        ctx.enqueue(msg)  # 发送导航 + NewSession 消息

Step 9: 前端收到
        → appPages 更新为注册表
        → currentPageScriptHash = 默认页哈希
        → maybeUpdatePageUrl(newPageName="")  # URL 改回 "/"
        → 渲染默认页内容
        → PageNotFound 元素显示在顶部（可选消失）
```

---

## 七、七层触发方式对应的三层标识取值（汇总速查表）

| 触发方式 | 前端 currentPageScriptHash 状态 | 前端取值手段 | BackMsg.rerunScript | _resolve_page_script 命中分支 |
|----------|---------------------------------|-------------|---------------------|--------------------------------|
| 首次访问 `/`（根路径） | 空 | `extractPageNameFromPathName` 得到 `""` | `{hash: "", name: ""}` | 条件 3：直接 `dict.get(fallback)` → 默认页 |
| 首次访问 `/dashboard`（存在）| 空 | `extractPageNameFromPathName` 得到 `"dashboard"` | `{hash: "", name: "dashboard"}` | 条件 2：filter 匹配到 `url_pathname="dashboard"` 的条目 |
| 首次访问 `/nope`（不存在）| 空 | `extractPageNameFromPathName` 得到 `"nope"` | `{hash: "", name: "nope"}` | 条件 2：filter 无匹配 → 返回 None → 上层兜底到默认页 + PageNotFound |
| 点击导航菜单 / page_link | 非空 | Navigation 消息里直接有 `pageScriptHash` | `{hash: "a1b2...", name: ""}` | 条件 1：`dict.get("a1b2...", fallback)` → 精确命中 |
| 浏览器前进/后退（popstate）| 非空 | `findPageByUrlPath(pathname)` 查表拿 `pageScriptHash` | `{hash: "a1b2...", name: ""}` | 条件 1：精确命中 |
| 重新运行按钮 | 非空 | 直接读 `state.currentPageScriptHash` | `{hash: 当前哈希, name: ""}` | 条件 1：精确命中当前页 |
| `st.switch_page(...)` | 已存在（后端产生） | 后端直接构造 RerunData，不经过前端 | `{page_script_hash: 目标哈希}` | 条件 1：`dict.get(目标哈希, fallback)` → 命中或回退 |

---

## 八、关键代码位置索引（相对项目根路径）

| 文件 | 关注点 | 说明 |
|------|--------|------|
| `lib/streamlit/navigation/page.py` | `StreamlitPage.url_path` property、`StreamlitPage._script_hash` property、`StreamlitPage.__init__` 中 `default` 存储 | 默认页强制 `url_path=""`、内部哈希基于对外 property 计算 |
| `lib/streamlit/commands/navigation.py` | `_navigation()` 中默认页确定、`pagehash_to_pageinfo` 构建、`send_page_not_found` 触发条件、回退赋值 `page_to_return = default_page` | 回退判定的最上层位置 |
| `lib/streamlit/runtime/pages_manager.py` | `PagesManager.__init__`（main_script_hash 计算）、`_resolve_page_script()`（三层分支判定）、`set_pages_and_resolve()` | 路由解析核心、main_script_hash 来源 |
| `lib/streamlit/util.py` | `calc_hash()` 定义 | 统一哈希函数 |
| `lib/streamlit/runtime/scriptrunner_utils/script_run_context.py` | `ScriptRunContext.enqueue()` 给每条 ForwardMsg 打 `active_script_hash` 标签 | main_script_hash 与页面元素归属的连接点 |
| `frontend/app/src/App.tsx` | `sendRerunBackMsg()` 三大分支判定、`onHistoryChange` popstate 监听、`extractPageNameFromPathName` 调用 | 首次直访时的前端解析入口 |
| `frontend/app/src/util/AppNavigation.ts` | `findPageByUrlPath()`、`clearPageElements()`、`handleNavigation()` | 前端反向 URL 查表、元素清理策略 |
| `frontend/lib/src/util/utils.ts` | `extractPageNameFromPathName()` 实现 | 浏览器 pathname 到对外 url_path 的转换算法 |
| `lib/streamlit/runtime/app_session.py` | `_create_new_session_message()` 中 `msg.new_session.page_script_hash`、`main_script_hash` 字段填充 | 后端到前端 NewSession 消息的 hash 装配点 |
