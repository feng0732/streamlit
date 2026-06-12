# Streamlit 多页应用核心标识与路由机制

本文围绕三个核心问题展开：

1. **默认页的内部 hash 到底按什么值算？按对外空路径，还是按页面自己保存的路径？**
2. **`page_script_hash` 和 `main_script_hash` 分别基于什么计算、承担什么职责？**
3. **直接访问不存在的路径、或者匹配到的是外部 URL 页，会走哪条链路最终回到默认页？PageNotFound 提示触发的精确条件是什么？**

---

## 一、默认页的"两条路径"：一个容易忽略的关键裂隙

### 1.1 两个不同的属性

`StreamlitPage` 内部有两个看起来像、但实际取值不同的属性：

| 属性 | 类型 | 对默认页的取值 | 对非默认页的取值 |
|------|------|---------------|-----------------|
| `_url_path` | 普通字段（`str`） | 保存**原始推导值**，如 `"home"` | 保存原始推导值或用户指定值 |
| `url_path` | `@property` | `""`（被 `_default=True` 强制覆盖） | 等于 `_url_path` |

**代码证据**：

```python
# 字段赋值——不区分默认页
self._url_path = inferred_name        # 从文件名/函数名推导
if url_path is not None:
    self._url_path = stripped_url_path  # 或用户指定的 url_path 参数

# property——默认页强制返回空串
@property
def url_path(self) -> str:
    return "" if self._default else self._url_path
```

> **关键**：`_url_path` 字段在 `__init__` 里赋完值就不再改了。`st.navigation()` 把某个页面的 `_default` 改为 `True` 时，**不会**同步修改 `_url_path`。因此默认页始终同时持有一个非空内部路径和一个空的对外路径。

### 1.2 _script_hash 用的是哪条路径？

```python
@property
def _script_hash(self) -> str:
    return calc_hash(self._url_path)    # ← 用的是内部字段 _url_path
```

**结论：默认页的内部 hash 是按 `_url_path`（原始推导值）计算的，不是按对外空路径 `""` 计算的。**

这和 `url_path` property 返回的空串形成了**裂隙**：

| 页面 | `_url_path`（内部字段） | `url_path`（property） | `_script_hash` |
|------|------------------------|----------------------|----------------|
| 默认页 `home.py` | `"home"` | `""` | `calc_hash("home")` |
| Dashboard 页 `dashboard.py` | `"dashboard"` | `"dashboard"` | `calc_hash("dashboard")` |
| Settings 页（用户指定） | `"settings"` | `"settings"` | `calc_hash("settings")` |

> ⚠️ 只有非默认页的 `url_path` 和 `_url_path` 是一致的，所以只有非默认页满足 `_script_hash = calc_hash(url_path)`。默认页**不满足**这个等式。

### 1.3 为什么这个裂隙不导致 bug？

虽然默认页的 `_script_hash = calc_hash("home")` 不等于 `calc_hash("")`，但系统各处的使用方式是**自洽**的：

| 使用位置 | 用了哪个值 | 为什么不出错 |
|----------|-----------|-------------|
| `pagehash_to_pageinfo` 字典的 key | `page._script_hash` = `calc_hash(_url_path)` | 注册和查找都用同一个 key |
| `pagehash_to_pageinfo["url_pathname"]` | `page.url_path` = `""` | 用于字符串匹配，和前端提取的 pageName 对齐 |
| Navigation proto `app_pages[i].pageScriptHash` | `page._script_hash` = `calc_hash(_url_path)` | 前端拿到后用于精确查找 |
| Navigation proto `app_pages[i].urlPathname` | `page.url_path` = `""` | 前端用于地址栏展示 |
| `fallback_page_hash` | `default_page._script_hash` = `calc_hash(_url_path)` | 在 `_pages` 字典中能找到默认页 |
| `_resolve_page_script` 字符串匹配 | `p["url_pathname"] == intended_page_name` | 用 `url_pathname=""` 匹配根路径 |

**自洽原则**：注册表 key 和查找 key 都来自 `_script_hash`（基于 `_url_path`），展示和字符串匹配都来自 `url_path` property（基于对外空串）。两条线各走各的，不会交叉。

---

## 二、三层标识：对外路径、页面标识、应用标识

### 2.1 总览

| 层级 | 标识名 | 面向谁 | 计算输入 | 代码位置 |
|------|--------|--------|----------|---------|
| ① 对外层 | **`url_path` / `url_pathname`** | 浏览器地址栏 / 前端 URL 展示 | 页面名称推导或用户指定（默认页 property 返回 `""`） | `StreamlitPage.url_path` property |
| ② 页面层 | **`page_script_hash`** | 前后端通信的页面唯一 key | **`calc_hash(_url_path)`** — 内部字段，不是 property | `StreamlitPage._script_hash` property |
| ③ 应用层 | **`main_script_hash`** | 入口脚本的元素归属标记 | **`calc_hash(入口脚本路径)`** — 与页面完全无关 | `PagesManager.__init__` |

### 2.2 统一哈希函数

```python
def calc_hash(s: bytes | str) -> str:
    b = s.encode("utf-8") if isinstance(s, str) else s
    h = hashlib.blake2b(digest_size=16, usedforsecurity=False)
    h.update(b)
    return h.hexdigest()    # 32 位十六进制
```

三层标识共用同一个哈希函数，但**输入源完全不同**：

| 标识 | 输入到 calc_hash 的字符串 | 输入来源 |
|------|--------------------------|---------|
| `url_path`（默认页） | 不经过 calc_hash | property 直接返回 `""` |
| `page_script_hash` | 页面内部 `_url_path` 字段值 | 如 `"home"`、`"dashboard"` |
| `main_script_hash` | 入口脚本文件路径字符串 | 如 `"/myapp/streamlit_app.py"` |

### 2.3 层级 ① 对外路径 `url_path`

**职责**：决定浏览器地址栏显示什么，决定前端 `extractPageNameFromPathName` 提取的字符串能匹配到谁。

**对默认页的特殊行为**：
- `url_path` property 检测 `_default=True` → 返回 `""`
- API 层禁止非默认页持有空串（初始化时会抛异常）
- 因此外部观察者永远看到默认页的 URL 路径是 `""`

**对非默认页**：
- `url_path` 等于 `_url_path`，从文件名/函数名/用户参数推导
- 经过 `_sanitize_url_path` 清理（小写、去特殊字符、合并下划线）

### 2.4 层级 ② 页面标识 `page_script_hash`

**职责**：作为 `pagehash_to_pageinfo` 注册表的 key，作为前后端通信中唯一标识一个页面的凭证。

**计算来源**：
```python
@property
def _script_hash(self) -> str:
    return calc_hash(self._url_path)    # 内部字段，非 property
```

**在注册表中的角色**：
```python
# st.navigation() 构建注册表
pagehash_to_pageinfo[page._script_hash] = {
    "page_script_hash": page._script_hash,     # = calc_hash(_url_path)
    "url_pathname":      page.url_path,         # = "" if default else _url_path
    ...
}
```

**在前端 Navigation proto 中的角色**：
```python
p.page_script_hash = page._script_hash   # 前端拿到后用于精确 rerun
p.url_pathname    = page.url_path        # 前端拿到后用于地址栏和 URL 匹配
```

**唯一性保证**：如果两个页面的 `_script_hash` 相同（即 `_url_path` 相同），`st.navigation()` 会抛异常。对于默认页，虽然 `url_path=""`，但 `_url_path` 通常不重复，所以冲突检测仍然有效。

### 2.5 层级 ③ 应用标识 `main_script_hash`

**职责**：标记入口文件产生的 UI 元素归属，使这些元素在切页时被保留。

**计算来源**：
```python
# PagesManager.__init__
self._main_script_hash = calc_hash(main_script_path)  # 入口脚本路径
```

**使用位置 1 — 元素归属标记**：

脚本运行期间，`active_script_hash` 决定每条 ForwardMsg 属于谁：

```python
# ScriptRunContext.reset() 中初始化
ThreadState.initialize(active_script_hash=self.pages_manager.main_script_hash)

# 入口文件执行期间
#   → active_script_hash = main_script_hash
#   → 入口文件产生的所有元素都带 main_script_hash 标签

# pg.run() 执行页面代码期间
#   → with ctx.run_with_active_hash(page._script_hash):
#   → 页面内产生的元素带 page_script_hash 标签
```

**使用位置 2 — 前端切页清理**：

```typescript
clearPageElements(elements, mainScriptHash) {
  return elements.filterMainScriptElements(mainScriptHash)
  // → 保留 active_script_hash === mainScriptHash 的元素（入口文件的）
  // → 清除其他的（具体页面的）
}
```

**使用位置 3 — query_params 过滤**：

```python
# script_runner.py 中，跨页切换时
valid_script_hashes = {main_script_hash, page_script_hash}
# 只保留属于入口脚本或当前页面的 query params
```

> **`main_script_hash` 和 `page_script_hash` 的本质区别**：
> - `main_script_hash` 是「谁的元素不该被清除」的判据
> - `page_script_hash` 是「当前在哪个页面」的凭证
> - 两者输入源不同、用途不同、永远不可能相等（概率意义上）

---

## 三、三层标识的完整对应关系表

以三页应用为例：入口 `streamlit_app.py`，页面 `home.py`（默认）、`dashboard.py`、第三页指定了 `url_path="settings"`。

| 项目 | 默认页（home.py） | 第二页（dashboard.py） | 第三页（url_path="settings"） | 入口文件公共 UI |
|------|-------------------|------------------------|-------------------------------|-----------------|
| **`_url_path`（内部字段）** | `"home"` | `"dashboard"` | `"settings"` | — |
| **`url_path`（property）** | **`""`** ← 被覆盖 | `"dashboard"` | `"settings"` | — |
| **`_script_hash`** | `calc_hash("home")` | `calc_hash("dashboard")` | `calc_hash("settings")` | — |
| **浏览器地址栏** | `http://app/` | `http://app/dashboard` | `http://app/settings` | — |
| **注册表 key** | `calc_hash("home")` | `calc_hash("dashboard")` | `calc_hash("settings")` | — |
| **注册表 url_pathname** | `""` | `"dashboard"` | `"settings"` | — |
| **Navigation proto pageScriptHash** | `calc_hash("home")` | `calc_hash("dashboard")` | `calc_hash("settings")` | — |
| **Navigation proto urlPathname** | `""` | `"dashboard"` | `"settings"` | — |
| **run_with_active_hash 用值** | `calc_hash("home")` | `calc_hash("dashboard")` | `calc_hash("settings")` | `calc_hash(入口脚本路径)` |
| **切页时元素清理** | ❌ 清理 | ❌ 清理 | ❌ 清理 | ✅ 保留 |

> **默认页的核心特征**：对用户来说是根路径 `""`，对系统内部来说 hash 来自 `"home"`。两条线在注册表的同一条 `PageInfo` 记录里汇合——key 是内部 hash，`url_pathname` 是对外空串。

---

## 四、路由解析：两个匹配入口如何到达同一个默认页

### 4.1 `_resolve_page_script()` 的三个分支

```python
def _resolve_page_script(self, fallback_page_hash=""):
    if self._pages is None:
        return None

    # 分支 1：有 page_script_hash → 精确匹配注册表 key
    if self.intended_page_script_hash:
        return self._pages.get(
            self.intended_page_script_hash,
            self._pages.get(fallback_page_hash, None),  # 匹配失败 → 用 fallback
        )

    # 分支 2：有 page_name → 用 url_pathname 字符串匹配
    if self.intended_page_name:
        return next(
            filter(
                lambda p: p["url_pathname"] == self.intended_page_name,
                self._pages.values(),
            ),
            None,  # 匹配失败 → 返回 None（不直接 fallback）
        )

    # 分支 3：两个都没有 → 直接取 fallback
    return self._pages.get(fallback_page_hash, None)
```

**fallback_page_hash 是什么？**

```python
# st.navigation() 传入
fallback_page_hash = default_page._script_hash   # = calc_hash(default_page._url_path)
```

### 4.2 访问根路径时的两种到达路径

用户访问 `http://app/` → 前端提取 `pageName = ""` → 后端 `set_script_intent(hash="", name="")`

- 分支 1：`intended_page_script_hash = ""` → 空字符串为 falsy → **跳过**
- 分支 2：`intended_page_name = ""` → 空字符串为 falsy → **跳过**
- 分支 3：`self._pages.get(fallback_page_hash)` → 用 `calc_hash("home")` 查表 → **命中默认页**

> 所以根路径访问走的是**分支 3**，通过 fallback hash 直接查注册表。

### 4.3 前端已知 hash 时的到达路径

用户点击导航菜单 → 前端发送 `pageScriptHash = calc_hash("home")`

- 分支 1：`intended_page_script_hash = calc_hash("home")` → 在注册表中查找 → **命中默认页**

> 此时走的是**分支 1**，用内部 hash 精确匹配。

### 4.4 访问不存在的路径

用户访问 `http://app/nonexistent` → 前端提取 `pageName = "nonexistent"`

- 分支 1：`intended_page_script_hash = ""` → 跳过
- 分支 2：`intended_page_name = "nonexistent"` → 遍历所有 `url_pathname`，找不到 → 返回 `None`
- 分支 2 返回 None → 上层 `if not page_to_return` → 触发 `send_page_not_found()` + 赋值 `default_page`

### 4.5 访问存在的子路径

用户访问 `http://app/dashboard` → 前端提取 `pageName = "dashboard"`

- 分支 1：`intended_page_script_hash = ""` → 跳过
- 分支 2：`intended_page_name = "dashboard"` → 遍历 `url_pathname`，找到 `"dashboard"` → **命中**

---

## 五、回退判定的完整链路

### 5.1 回退触发的统一条件：`if not page_to_return`

所有回退路径最终都汇聚到同一个判定：

```python
if not page_to_return:
    send_page_not_found(ctx)
    page_to_return = default_page
```

> 也就是说：**回退的唯一触发条件是 `page_to_return` 为 falsy（通常是 `None`），与 `found_page` 本身是否为 None 没有直接绑定关系。** `found_page` 只是上游的一个中间结果，而不是最终判据。

### 5.2 三条让 `page_to_return` 变为 None 的路径

#### 路径 A：`_resolve_page_script` 直接返回 None

上游 `_resolve_page_script()` 三个分支中任何一个返回 None：
- 分支 1 走到 inline fallback，但 fallback_hash 在注册表中也查不到（极端异常情况）
- 分支 2：字符串匹配 `url_pathname == intended_page_name`，filter 没有找到匹配项，`next(..., None)` 返回 None
- 分支 3：`dict.get(fallback_page_hash, None)` 返回 None（fallback_hash 在注册表也查不到，极端异常）

```python
found_page = ctx.pages_manager.set_pages_and_resolve(...)
# found_page = None
page_to_return = None   # 初始值就是 None，if found_page: 块不执行
```

**典型场景**：用户访问不存在的 URL 路径（如 `/nonexistent`），分支 2 filter 无匹配 → None。

#### 路径 B：`found_page` 非 None，但在 `page_list` 中找不到对应的 StreamlitPage

`found_page` 来自 `_pages` 注册表（`PageInfo` 字典），里面有 `page_script_hash`。接下来需要用这个 hash 在 `page_list`（Python 中的 StreamlitPage 对象列表）里做二次匹配：

```python
if found_page:
    found_page_script_hash = found_page["page_script_hash"]
    matching_pages = [
        p for p in page_list if p._script_hash == found_page_script_hash
    ]
    if len(matching_pages) > 0:   # ← 如果没有匹配上
        page_to_return = matching_pages[0]
    # 否则 page_to_return 保持为初始值 None
```

**典型场景**：理论层面的"防护性分支"。注册表和 `page_list` 都是在 `st.navigation()` 里由同一份数据构建的，正常情况下 hash 应该一一对应。代码留了这个兜底分支以防不一致。

#### 路径 C：匹配到了页面，但它是外部 URL 页，被显式置空

```python
# 路径 A 或 B 之后，page_to_return 可能已经是某个 StreamlitPage
# 现在做外部页过滤：
if page_to_return and page_to_return.is_external:
    page_to_return = None    # ← 显式置空
```

**典型场景**：外部 URL 页（如 `st.Page("https://docs.streamlit.io", title="Docs")`）也有自己的 `_url_path`（`"docs"`）和 `_script_hash`（`calc_hash("docs")`），也会被加入注册表。如果用户猜到它的 URL 路径并直接访问：
1. `_resolve_page_script` 分支 2 匹配到 `url_pathname="docs"` → `found_page` 非 None
2. 在 `page_list` 里找到对应的外部页 StreamlitPage → `page_to_return = 外部页`
3. 外部页过滤：`is_external == True` → `page_to_return = None`
4. 进入 `if not page_to_return` → 触发 `send_page_not_found` + 赋值默认页

> **外部页不能直接通过 URL 访问**的底层原因：外部页 `_page = None`（没有 Python 文件也没有 Callable），`pg.run()` 会什么都不执行，所以必须强制回退。

### 5.3 三条路径的汇总判定图

```
set_pages_and_resolve(registry, fallback_hash) → found_page
    │
    │ found_page is None？
    ├─ 是 → page_to_return = None （路径 A）
    │
    └─ 否 → 取 found_page_script_hash
         │
         │ page_list 中有匹配 hash 的 StreamlitPage？
         ├─ 否 → page_to_return = None （路径 B）
         │
         └─ 是 → page_to_return = matching_pages[0]
              │
              │ page_to_return.is_external？
              ├─ 是 → page_to_return = None （路径 C）
              │
              └─ 否 → page_to_return = 正常页面，不回退

最终统一判定：
    if not page_to_return:
        send_page_not_found(ctx)          # ← 三路径 A/B/C 都会触发
        page_to_return = default_page     # ← 三路径 A/B/C 都兜底
```

### 5.4 场景速查表：每条路径各自对应什么用户行为

| 场景 | 让 `page_to_return` 为空的路径 | `found_page` 值 | send_page_not_found？ | 用户体验 |
|------|-------------------------------|-----------------|----------------------|---------|
| 访问根路径 `/` | —（正常命中，无需回退） | 非 None（分支 3 fallback 成功） | **否** | 正常首页 |
| 点击导航菜单到默认页 | —（正常命中） | 非 None（分支 1 哈希精确命中） | **否** | 正常首页 |
| 访问存在的子路径 `/dashboard` | —（正常命中） | 非 None（分支 2 字符串匹配命中） | **否** | 正常页面 |
| 访问不存在的路径 `/nope` | **路径 A**（分支 2 filter 无匹配 → None） | None | **是** | 提示 + 首页 |
| 哈希匹配失败（构造不存在的 hash） | —（分支 1 inline fallback 命中默认页） | 非 None（默认页 PageInfo） | **否** | **静默**跳首页 |
| 猜到并直访外部 URL 页的路径 `/docs` | **路径 C**（先匹配成功，再被 `is_external` 置 None） | 非 None，但被置空 | **是** | 提示 + 首页 |
| 注册表与 page_list hash 不一致（极端） | **路径 B**（found_page 非 None，但 page_list 无匹配） | 非 None | **是** | 提示 + 首页 |

> ⚠️ **关于「哈希匹配失败静默不提示」的理解**：它不是因为走了 `if not page_to_return` 分支而被跳过——相反，它根本就**没进入**这个分支。因为分支 1 的 `dict.get(key, dict.get(fallback))` 在 key 查不到时 inline 取了 fallback（默认页），`found_page` 是正常的默认页 PageInfo，路径 B 和路径 C 也都不会把它置空，所以 `page_to_return` 是非 None 的，`if not page_to_return` 条件为假，自然就跳过了提示。

### 5.5 为什么根路径不触发 PageNotFound

根路径访问时：
- `set_script_intent(hash="", name="")` → 两个意图值都是空串
- 空串都是 falsy → 跳过分支 1 和分支 2
- 分支 3：`dict.get(fallback_page_hash)` → fallback_hash 是 `default_page._script_hash`，必然在注册表 → 返回默认页 PageInfo → 非 None
- 路径 B：在 page_list 找到默认页 → `page_to_return` = 默认页
- 路径 C：默认页不是外部页 → 不过滤
- 最终 `page_to_return` 非 None → `if not page_to_return` 为假 → 不触发 PageNotFound

### 5.6 PageNotFound 消息的前端处理细节

后端 `send_page_not_found()` 硬编码了 `page_name = ""`：

```python
def send_page_not_found(ctx):
    msg = ForwardMsg()
    msg.page_not_found.page_name = ""
    ctx.enqueue(msg)
```

前端拿到后：
```typescript
onPageNotFound = (pageName?: string): void => {
    const errMsg = pageName
        ? `You have requested page /${pageName}, but no corresponding file was found...`
        : "The page that you have requested does not seem to exist"
    this.showError("Page not found", {
        message: `${errMsg}. Running the app's main page.`,
    })
}
```

由于后端永远传 `""`（空字符串，falsy），所以前端**永远只显示通用提示**「The page that you have requested does not seem to exist」，不会在错误文案里携带用户实际访问的不存在路径。

同时前端 `handlePageNotFound` 会：
- `currentPageScriptHash` 临时设为 `mainScriptHash`
- 向主机发送一条 `SET_CURRENT_PAGE_NAME`，`currentPageName=""`，`currentPageScriptHash=mainScriptHash`
- 最终 URL 改回根路径 `""`

---

## 六、前后端各自用什么值定位页面（7 种场景速查）

| 触发方式 | 前端取值 | 发给后端的 RerunData | 后端命中分支 |
|----------|---------|---------------------|-------------|
| 首次访问根路径 `/` | `extractPageNameFromPathName` → `""` | `hash=""`, `name=""` | 分支 3：fallback |
| 首次访问存在的子路径 `/dashboard` | `extractPageNameFromPathName` → `"dashboard"` | `hash=""`, `name="dashboard"` | 分支 2：url_pathname 匹配 |
| 首次访问不存在的路径 `/nope` | `extractPageNameFromPathName` → `"nope"` | `hash=""`, `name="nope"` | 分支 2：无匹配 → None → 回退 |
| 点击导航菜单 | 从 Navigation proto 拿到 `pageScriptHash` | `hash="calc_hash(...)"`, `name=""` | 分支 1：哈希精确匹配 |
| 浏览器前进/后退 | `findPageByUrlPath` 查表拿 `pageScriptHash` | `hash="calc_hash(...)"`, `name=""` | 分支 1：哈希精确匹配 |
| 重新运行按钮 | 读当前 `currentPageScriptHash` | `hash=当前哈希`, `name=""` | 分支 1：精确命中 |
| `st.switch_page(...)` | 后端直接构造 RerunData | `page_script_hash=目标哈希` | 分支 1：精确匹配或 inline fallback |

---

## 七、关键代码位置索引

| 文件（相对项目根） | 关注点 |
|------|--------|
| `lib/streamlit/navigation/page.py` — `_script_hash` property | **`calc_hash(self._url_path)`** — 用内部字段，非 property |
| `lib/streamlit/navigation/page.py` — `url_path` property | **`"" if self._default else self._url_path`** — 默认页强制空串 |
| `lib/streamlit/navigation/page.py` — `__init__` 中 `_url_path` 赋值 | 初始化后不再改变，`st.navigation` 改 `_default` 不改 `_url_path` |
| `lib/streamlit/commands/navigation.py` — 默认页确定 | 遍历找 `_default=True`，没有则取第一个非外部页并就地设 `_default=True` |
| `lib/streamlit/commands/navigation.py` — 注册表构建 | key = `page._script_hash`，`url_pathname` = `page.url_path`（property） |
| `lib/streamlit/commands/navigation.py` — 回退判定 | `if not page_to_return` → `send_page_not_found` + 赋值 `default_page` |
| `lib/streamlit/runtime/pages_manager.py` — `__init__` | **`main_script_hash = calc_hash(main_script_path)`** — 入口脚本路径 |
| `lib/streamlit/runtime/pages_manager.py` — `_resolve_page_script` | 三分支判定：哈希精确 → 字符串匹配 → fallback |
| `lib/streamlit/util.py` — `calc_hash()` | BLAKE2b 统一哈希函数 |
| `lib/streamlit/runtime/scriptrunner_utils/script_run_context.py` — `reset()` | 初始化 `active_script_hash = main_script_hash` |
| `lib/streamlit/runtime/scriptrunner_utils/script_run_context.py` — `run_with_active_hash()` | 切换到 `page._script_hash` 执行页面代码 |
| `frontend/app/src/App.tsx` — `sendRerunBackMsg` | 三分支判定：有 hash → 有当前 hash → 空（用 extractPageNameFromPathName） |
| `frontend/lib/src/util/utils.ts` — `extractPageNameFromPathName()` | pathname 去 basePath 去首尾斜杠 + decodeURIComponent |
| `frontend/app/src/util/AppNavigation.ts` — `findPageByUrlPath()` | `url_pathname.endsWith("/" + p.urlPathname)` 反向查表 |
| `frontend/app/src/util/AppNavigation.ts` — `clearPageElements()` | 保留 `main_script_hash` 标记的元素，清除其他 |
