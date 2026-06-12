# Query Params 同步路径梳理

## 一、整体架构

Streamlit 的查询参数同步采用 **前后端分离 + WebSocket 消息驱动** 的架构：

- **后端**：Python 侧维护 `QueryParams` 状态，通过 `ForwardMsg.page_info_changed` 向前端推送 URL 变化
- **前端**：TypeScript 侧通过 `WidgetStateManager` 管理 URL 参数绑定，通过 `history.pushState/replaceState` 更新浏览器地址栏
- **通信协议**：WebSocket 双向通道，后端 → 前端用 `ForwardMsg`，前端 → 后端用 `BackMsg`

核心文件：
- 后端：[query_params.py](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py)、[session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py)、[script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py)
- 前端：[WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts)、[App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx)、[AppNavigation.ts](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/util/AppNavigation.ts)

---

## 二、参数读取路径（URL → 后端）

### 2.1 初始页面加载

**触发时机**：用户首次访问应用或刷新页面

**流程**：

```
浏览器 URL
  ↓
前端 App 构造函数
  → state.queryParams = window.location.search (初始值)
  ↓
WebSocket 连接建立 → 发送 rerunScript BackMsg
  → queryString: 从 URL 读取的完整 query string
  ↓
后端 script_runner.py 处理 rerun 请求
  → ctx.query_string = rerun_data.query_string
  ↓
QueryParams.set_initial_query_params(query_string)
  → 存储到 _initial_query_params (用于后续 widget 播种)
  ↓
脚本执行期间：widget 注册时调用 _handle_query_param_binding
  → query_params.get_initial_value(user_key) 读取初始值
  → parse_url_param() 解析为对应类型
  → 播种到 widget state 和 session state
```

**关键代码**：
- 前端初始值：[App.tsx#L355](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L355-L355)
- 后端初始播种：[session_state.py#L1111-L1146](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1111-L1146)
- URL 解析函数：[query_params.py#L149-L241](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L149-L241)

### 2.2 浏览器历史导航（前进/后退）

**触发时机**：用户点击浏览器前进/后退按钮，触发 `popstate` 事件

**流程**：

```
window.popstate 事件
  ↓
App.onHistoryChange()
  → 解析 document.location.pathname 找到目标页面
  → 同一页面且只有锚点变化时直接返回
  ↓
App.onPageChange(pageScriptHash, undefined, preserveQueryParams=true)
  ↓
sendRerunBackMsg()
  → preserveQueryParams=true 表示保留 URL 中的所有参数
  → queryString 直接从当前 URL 读取
  ↓
后端 script_runner 处理页面切换
  → populate_from_query_string(query_string, valid_script_hashes)
  → 过滤掉不属于当前页面的绑定参数
  → set_initial_query_params_from_current() 更新初始值
```

**关键代码**：
- popstate 监听：[App.tsx#L691](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L691-L691)
- onHistoryChange：[App.tsx#L1470-L1488](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L1470-L1488)
- MPA 参数过滤：[query_params.py#L723-L777](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L723-L777)

### 2.3 MPA 页面切换（站内导航）

**触发时机**：用户点击侧边栏导航、`st.page_link()` 或 `st.switch_page()`

**流程**：

```
前端触发 onPageChange(pageScriptHash)
  ↓
sendRerunBackMsg()
  → preserveQueryParams=false (默认)
  → widgetMgr.filterParamsForPageChange() 过滤参数
    - 保留 embed 相关参数
    - 保留绑定到 widget 的参数（通过 paramKeyToWidgetId 查找）
    - 清除其他所有参数
  ↓
后端 script_runner
  → populate_from_query_string(query_string, valid_script_hashes)
    - 基于现有 bindings 过滤掉其他页面的参数
    - 清除已解绑 widget 的 binding
  → set_initial_query_params_from_current()
  → 新页面脚本执行，widget 重新绑定和播种
```

**关键代码**：
- 页面切换参数过滤：[WidgetStateManager.ts#L1205-L1238](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1205-L1238)
- 后端 MPA 过滤：[script_runner.py#L594-L608](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L594-L608)

---

## 三、参数写入路径（后端 → URL）

### 3.1 用户代码直接设置 (st.query_params)

**触发时机**：用户代码调用 `st.query_params["key"] = value` 或 `.update()` / `.from_dict()`

**流程**：

```
st.query_params["foo"] = "bar"
  ↓
QueryParamsProxy.__setitem__()
  ↓
QueryParams.__setitem__()
  → 检查是否为绑定参数（绑定参数禁止直接设置）
  → 检查是否为受保护参数（embed, embed_options）
  → _set_item_internal() 更新内部字典
  → _send_query_param_msg() 发送 ForwardMsg
    ↓
    ForwardMsg.page_info_changed.query_string
    → 通过 ctx.enqueue() 加入消息队列
    ↓
前端 handlePageInfoChanged(pageInfo)
  → window.history.pushState({}, "", targetUrl)
  → setState({ queryParams: queryString })
  → 通知 host (SET_QUERY_PARAM)
```

**关键代码**：
- 写入入口：[query_params_proxy.py#L58-L60](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params_proxy.py#L58-L60)
- 发送消息：[query_params.py#L409-L419](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L409-L419)
- 前端接收：[App.tsx#L1145-L1157](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L1145-L1157)

### 3.2 Widget 绑定值变化 (bind="query-params")

**两条路径**：前端 UI 触发 和 后端编程触发

#### 路径 A：用户交互（前端 → URL → 后端）

```
用户操作 widget (slider, checkbox, etc.)
  ↓
WidgetStateManager.setXxxValue(widget, value, { fromUi: true })
  → maybeSyncValueToUrl(widgetId, source, value)
    ↓
    convertToUrlValue() 转换为 URL 格式
    shouldClearUrlParam() 判断是否需要清除（默认值折叠）
    updateUrlParam() 更新 URL
      → window.history.replaceState()
      → notifyQueryParamsChange() 通知 App 更新 state
    ↓
onWidgetValueChanged() → scheduleFlush()
  → sendRerunBackMsg() 发送 widget states 到后端
  ↓
后端处理 rerun，同步 widget 值
  → 同时同步到 query_params（通过 widget binding 机制）
```

**关键代码**：
- URL 同步入口：[WidgetStateManager.ts#L1408-L1430](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1408-L1430)
- URL 更新：[WidgetStateManager.ts#L1351-L1402](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1351-L1402)

#### 路径 B：后端编程设置（后端 → URL）

```
st.session_state["widget_key"] = new_value
  ↓
脚本执行 → widget 注册
  → register_widget() 中检测绑定参数
    → user_key 在 _new_session_state 中
    → 调用 set_corrected_value() 或 remove_param()
    → _send_query_param_msg() 发送 ForwardMsg
  ↓
前端 handlePageInfoChanged()
  → window.history.pushState() 更新 URL
```

**关键代码**：
- 后端同步逻辑：[session_state.py#L1045-L1097](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1045-L1097)

### 3.3 URL 自动校正

**触发时机**：URL 参数经过解析和校验后，实际值与 URL 中表示不一致时

**场景**：
- 值被 clamp 到 min/max 范围
- 数组值去重或过滤无效选项
- 日期/时间格式标准化

**流程**：

```
_seed_widget_from_url() 解析 URL 值
  → deserialized_value 与 parsed_value 比较
  → 不相等则调用 _auto_correct_url_if_needed()
    → query_params.set_corrected_value()
    → 发送 page_info_changed ForwardMsg
  ↓
前端接收 → pushState 更新 URL
```

**关键代码**：
- 自动校正：[session_state.py#L1257-L1271](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1257-L1271)

---

## 四、深度分析：页面状态恢复

### 4.1 核心状态存储结构

后端 SessionState 中有三个关键存储区域，它们的优先级和用途各不相同：

| 存储区域 | 类型 | 说明 |
|---------|------|------|
| `_new_widget_state` | WStates | 前端传来的 widget 值（用户交互），**优先级最高** |
| `_new_session_state` | dict | 本次脚本执行中用户代码通过 `st.session_state["k"] = v` 设置的值 |
| `_old_state` | dict | 历史值的压缩存储（上一次脚本执行结束后，将 `_new_widget_state` 和 `_new_session_state` 合并压缩到这里） |

**关键代码**：[session_state.py#L422-L429](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L422-L429)

### 4.2 初始加载（首次访问）

**场景**：用户第一次打开页面，或刷新页面后建立了新的 session。

**判断标志**：`widget_id not in self._old_state`（即该 widget 从未在此 session 中注册过）。

**恢复优先级**（从高到低）：
1. `_new_widget_state`（前端传了 widget 值）→ 不用 URL，用用户交互值
2. `_initial_query_params`（URL 参数）→ 有值则播种
3. 默认值 → 使用 widget 默认值

**代码逻辑** [session_state.py#L1135-L1146](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1135-L1146)：
```python
# _handle_query_param_binding 中
if widget_id in self._new_widget_state:  # 用户交互值存在
    return False  # 不播种 URL，用户交互优先

is_initial_load = widget_id not in self._old_state
if not is_initial_load and user_key in self._new_session_state:
    return False  # 非首次加载且代码设置了值，代码值优先

url_value = self.query_params.get_initial_value(user_key)
if url_value is None:
    return False  # URL 无值，用默认值

# 走到这里：首次加载 + URL 有值 + 无用户交互 → 播种 URL
return self._seed_widget_from_url(...)
```

**播种成功后**：值同时存入 `_new_widget_state` 和 `_old_state`（通过 `self[widget_id] = value` 触发 `__setitem__`）。

### 4.3 后续 Rerun（同一页面内的重复运行）

**场景**：用户操作了某个 widget 导致脚本重新运行，或点击了 Rerun 按钮。

**判断标志**：`widget_id in self._old_state`（该 widget 之前已注册过）。

**恢复优先级**（从高到低）：
1. `_new_widget_state`（前端传了 widget 值）→ 用户交互值
2. `_new_session_state`（本次脚本中代码设置了值）→ 编程设置值
3. `_old_state`（历史压缩值）→ 保持上一次的值
4. URL 参数 → **不会** 从 URL 重新播种（因为 `not is_initial_load`）

**关键防御逻辑**：非首次加载时，`_new_session_state` 中的值会覆盖 URL 值，防止 URL 参数在后续 rerun 中反覆盖用户的交互或代码设置。

### 4.4 页面刷新（F5 / Cmd+R）

**场景**：用户手动刷新页面，前端状态全部丢失，WebSocket 重新连接，后端创建新 session。

**恢复过程**：
1. 前端 `WidgetStateManager` 中的 `boundWidgets`、`paramKeyToWidgetId` 全部清空
2. 前端所有 widget 值丢失
3. 后端创建新的 `SessionState`，`_old_state`、`_new_widget_state`、`_new_session_state` 全部为空
4. 从 URL 重新读取 query string，存入 `_initial_query_params`
5. 脚本重新执行，widget 重新注册，**按初始加载规则从 URL 播种**
6. 用户之前通过 `st.session_state["k"]` 设置的非绑定值 **全部丢失**
7. 只有绑定到 `query-params` 的 widget 值可以从 URL 恢复

### 4.5 URL 播种后的自动校正

如果从 URL 解析出的值经过 deserializer 后与原始 URL 表示不一致（例如被 clamp、去重、格式转换），后端会自动校正 URL：

**代码逻辑** [session_state.py#L1257-L1271](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1257-L1271)：
```python
# _auto_correct_url_if_needed 中
parsed_value = parse_url_param(url_value, value_type)
deserialized_value = deserializer(parsed_value)
reserialized = serializer(deserialized_value)
coerced = _coerce_value_for_query_url(reserialized, value_type)

if coerced != url_value:
    # 值经过转换后变化了（如 clamp、去重、格式标准化）
    # 回写校正后的值到 URL
    self.set_corrected_value(param_key, reserialized, value_type)
```

**校正场景示例**：
- `?page=10` 但 slider 的 max 是 5 → 校正为 `?page=5`
- `?tags=foo,foo,bar` → 去重后校正为 `?tags=foo,bar`
- `?date=2024-1-1` → 格式标准化为 `?date=2024-01-01`

---

## 五、深度分析：前端交互后的地址回写是否会循环更新

### 5.1 完整调用链分析

用户操作 widget 时，完整的同步链路如下：

```
① 用户拖动 slider (value=75)
  ↓
② 前端 WidgetStateManager.setIntValue(widget, 75, { fromUi: true })
  → maybeSyncValueToUrl(widgetId, { fromUi: true }, 75)
    → source.fromUi == true → 继续
    → updateUrlParam("page", "75", ...)
      → newSearch === currentSearch? 75 != 50 → 不等
      → window.history.replaceState(...)  ✅ 第一次更新 URL
    → onWidgetValueChanged() → scheduleFlush()
  ↓
③ 前端 sendRerunBackMsg()
  → BackMsg 携带 widget_states: [{id: "$$slider-1", intValue: 75}]
  ↓
④ 后端 script_runner 处理
  → widget_states 存入 SessionState._new_widget_state
  → 脚本执行 → slider 注册
  ↓
⑤ 后端 register_widget("$$slider-1", user_key="page", bind="query-params")
  → _handle_query_param_binding(...)
    → widget_id in self._new_widget_state → YES (有用户交互值)
    → return False ❌ 不播种 URL
    → url_value_seeded = False
  ↓
⑥ 后端 register_widget 后半段 URL 同步 [session_state.py#L1064-L1097]
  → widget_value != default_value? YES (75 != 50)
  → user_key in _old_state? 视情况
  → user_key in _new_session_state? NO (用户交互值在 _new_widget_state)
  → stored_param_matches_corrected_value? 不进入此分支
  → 最终走到 discard_param_no_forward_msg(user_key)
  ❌ 不发送 ForwardMsg
  ↓
⑦ 脚本执行完成 → 无 page_info_changed 消息发送
  ↓
⑧ 前端 handlePageInfoChanged 不会被调用
  ❌ 不会第二次更新 URL
```

**结论：不会循环更新！**

### 5.2 防止循环的三道防线

| 防线 | 位置 | 逻辑 |
|------|------|------|
| 防线 1 | 前端 `updateUrlParam` | `newSearch === currentSearch` 时直接 return，不做无意义更新 |
| 防线 2 | 后端 `_handle_query_param_binding` | `widget_id in _new_widget_state` 时跳过 URL 播种，不触发回写 |
| 防线 3 | 后端 `register_widget` URL 同步逻辑 | `user_key in _new_session_state` 为 false（用户交互值在 `_new_widget_state`），不进入发送 ForwardMsg 的分支 |

**关键代码**：
- 防线 1：[WidgetStateManager.ts#L1394-L1396](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1394-L1396)
- 防线 2：[session_state.py#L1136-L1137](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1136-L1137)
- 防线 3：[session_state.py#L1067-L1088](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1067-L1088)

### 5.3 例外：后端会主动回写的场景

只有以下场景后端会主动发送 `page_info_changed` ForwardMsg 给前端，触发第二次 URL 更新：

| 场景 | 触发条件 |
|------|---------|
| **值丢失恢复** | `user_key in _old_state` 且 `not has_param(user_key)` 且 `not in _new_session_state`（如页面导航后 remount） |
| **编程设置** | `user_key in _new_session_state` 且 `not url_value_seeded` 且值不匹配（`st.session_state["k"] = v`） |
| **编程重置为默认** | `user_key in _new_session_state` 且值等于默认值且 `has_param(user_key)` |
| **URL 自动校正** | `_seed_widget_from_url` 中 `coerced != url_value`（值被 clamp/去重/格式转换） |

这些场景都有一个共同点：**值的变化来自后端逻辑，而非前端用户交互**。

### 5.4 前端 handlePageInfoChanged 的潜在问题

值得注意的是，前端 `handlePageInfoChanged` **没有去重检查** [App.tsx#L1145-L1157](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L1145-L1157)：

```typescript
handlePageInfoChanged = (pageInfo: PageInfo): void => {
  const { queryString } = pageInfo
  const targetUrl = document.location.pathname + (queryString ? `?${queryString}` : "")
  window.history.pushState({}, "", targetUrl)  // 无条件 pushState！
  this.setState({ queryParams: queryString })
  // ...
}
```

如果后端没有上述三道防线，这里会产生冗余的历史记录。但由于后端的精准控制，实际不会出现问题。

---

## 六、深度分析：跨页面切换时参数保留与清理

### 6.1 两层过滤机制

跨页面切换时，参数会经过**前端 + 后端**两层过滤：

```
用户点击侧边栏 → 切换到 Page B
  ↓
① 前端过滤（第一层）
  → filterParamsForPageChange(embedParams)
  ↓
② 发送 BackMsg.rerunScript(queryString=过滤后)
  ↓
③ 后端过滤（第二层）
  → populate_from_query_string(queryString, valid_script_hashes)
  ↓
④ set_initial_query_params_from_current()
  ↓
⑤ Page B 脚本执行
```

### 6.2 第一层：前端过滤

**代码位置**：[WidgetStateManager.ts#L1205-L1238](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1205-L1238)

```typescript
filterParamsForPageChange(embedParams: string): string {
  // 1. 没有绑定的 widget → 只保留 embed 参数
  if (this.paramKeyToWidgetId.size === 0) {
    return embedParams
  }

  // 2. 从当前 URL 提取所有绑定 widget 的参数
  this.paramKeyToWidgetId.forEach((_, paramKey) => {
    const values = currentUrl.searchParams.getAll(paramKey)
    if (values.length === 1) {
      boundParamsObj[paramKey] = values[0]
    } else if (values.length > 1) {
      boundParamsObj[paramKey] = values
    }
  })

  // 3. 拼接 embed 参数 + 绑定 widget 参数
  return embedParams ? `${embedParams}&${boundParamsStr}` : boundParamsStr
}
```

**前端过滤规则**：

| 参数类型 | 是否保留 | 说明 |
|---------|---------|------|
| `embed` / `embed_options` | ✅ 保留 | 来自 `preserveEmbedQueryParams()`，始终保留 |
| 绑定到 widget 的参数 | ✅ 保留 | 通过 `paramKeyToWidgetId` Map 查找，保留当前页面所有绑定 widget 的参数 |
| 自由参数（`st.query_params["foo"]`） | ❌ 清除 | 没有绑定关系，直接丢弃 |

### 6.3 第二层：后端过滤

**代码位置**：[query_params.py#L723-L777](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L723-L777)

```python
def populate_from_query_string(
    self, query_string: str, valid_script_hashes: set[str]
) -> None:
    # 1. 解析 URL 参数
    parsed = parse_query_string(query_string)

    # 2. 构建新的 _query_params 和 _initial_query_params
    new_query_params: dict[str, str | list[str]] = {}
    new_bindings_by_param: dict[str, QueryParamBinding] = {}

    # 3. 遍历当前所有绑定
    for param_key, binding in self._bindings_by_param.items():
        # 只保留属于当前页面的绑定
        if binding.script_hash in valid_script_hashes:
            # 如果 URL 中有这个参数，从 URL 取值
            if param_key in parsed:
                new_query_params[param_key] = parsed[param_key]
                del parsed[param_key]  # 从 parsed 中移除，避免重复处理
            # 否则从现有 _query_params 取值（保留后端状态）
            elif param_key in self._query_params:
                new_query_params[param_key] = self._query_params[param_key]
            new_bindings_by_param[param_key] = binding

    # 4. 处理 URL 中未绑定的参数（自由参数）
    for param_key, value in parsed.items():
        if param_key.lower() not in EMBED_QUERY_PARAMS_KEYS:
            new_query_params[param_key] = value

    # 5. 替换现有状态
    self._query_params = new_query_params
    self._bindings_by_param = new_bindings_by_param
    # 重建 _bindings_by_widget
    self._bindings_by_widget.clear()
    for binding in new_bindings_by_param.values():
        self._bindings_by_widget[binding.widget_id] = binding
```

**后端过滤规则**：

| 参数类型 | 是否保留 | 说明 |
|---------|---------|------|
| 当前页面的绑定参数 | ✅ 保留 | `binding.script_hash in valid_script_hashes` |
| 其他页面的绑定参数 | ❌ 清除 | 其他页面的 widget 已不存在，清除 binding 和 param |
| URL 中的自由参数 | ✅ 保留（如果前端没清） | 但前端第一层已经清掉了，实际不会到这里 |
| embed 参数 | ✅ 保留 | 特殊保护，不进入 `new_query_params`，但也不会被清除 |

**注意**：`valid_script_hashes = {main_script_hash, page_script_hash}`，即主页面（通常是 `Home.py`）和当前页面的绑定都会保留。这意味着跨页面导航时，主页面 widget 的绑定参数也会被保留。

### 6.4 同页面刷新 vs 跨页面导航的区别

| 行为 | 同页面刷新（F5） | 跨页面导航（Page A → Page B） |
|------|-----------------|----------------------------|
| URL 参数来源 | 完整 URL | 前端过滤后（embed + 当前页绑定） |
| 前端过滤 | 无（`preserveQueryParams=true`） | 有（`filterParamsForPageChange`） |
| 后端过滤 | 无（同页面，script_hash 不变） | 有（按 script_hash 过滤） |
| 绑定参数保留 | 全部保留 | 只保留当前页 + 主页面的 |
| 自由参数保留 | 全部保留（前端未过滤） | 全部清除（前端已过滤） |
| session_state | 全部丢失（新 session） | 绑定参数保留，其他丢失 |

### 6.5 Stale Widget 清理

除了跨页面切换，每次脚本运行结束后还会清理 **stale widget**（本次脚本中未注册的 widget）：

**代码位置**：[query_params.py#L779-L830](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L779-L830)

```python
def remove_stale_bindings(
    self,
    active_widget_ids: frozenset[str],
    fragment_ids_this_run: list[str] | None = None,
    widget_metadata: dict[str, Any] | None = None,
) -> None:
    for widget_id in self._bindings_by_widget:
        if widget_id in active_widget_ids:
            continue  # 活跃 widget，保留

        # Fragment 运行时，其他 fragment 的 widget 不清理
        if fragment_ids_this_run and widget_metadata:
            metadata = widget_metadata.get(widget_id)
            if metadata and metadata.fragment_id not in fragment_ids_this_run:
                continue  # 其他 fragment 的 widget，保留

        stale_widget_ids.append(widget_id)

    # 清理 stale widget 的 binding 和 URL 参数
    for widget_id in stale_widget_ids:
        binding = self._bindings_by_widget.get(widget_id)
        if binding:
            param_key = binding.param_key
            if param_key in self._query_params:
                del self._query_params[param_key]
                params_removed = True
        self.unbind_widget(widget_id)

    # 有参数被清除时，发送 ForwardMsg 更新前端 URL
    if params_removed:
        self._send_query_param_msg()
```

**常见的 stale 场景**：
- 条件渲染的 widget（`if st.checkbox("显示滑块"): st.slider(bind="query-params")`）
- 循环生成的 widget（`for i in range(n): st.slider(key=f"foo_{i}", bind="query-params")`）当 n 减小时
- 页面结构变化，某些 widget 不再渲染

---

## 七、URL 更新边界

### 7.1 pushState vs replaceState

| 场景 | API | 原因 |
|------|-----|------|
| 后端 page_info_changed 消息 | `pushState` | 每次后端更新都产生一条历史记录 |
| 前端 widget 值变化 (UI) | `replaceState` | 同一页面内的连续交互不产生多条历史 |
| MPA 页面导航 | `pushState` | 页面切换是显式导航，产生历史记录 |
| 初始加载 | 无 | URL 本来就在那儿 |

**关键代码**：
- pushState（后端驱动）：[App.tsx#L1149](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L1149-L1149)
- replaceState（前端 UI 驱动）：[WidgetStateManager.ts#L1400](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1400-L1400)
- 页面导航 pushState：[App.tsx#L1332](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L1332-L1332)

### 7.2 默认值折叠 (Hide-at-Default)

当 widget 值等于其默认值时，URL 参数会被**移除**（而不是保留默认值）。

**判断逻辑**：
```
shouldClearUrlParam(urlValue, binding):
  1. 值为空 且 不允许空 → 清除
  2. 值等于默认值 → 清除（默认值折叠）
  3. 其他情况 → 保留
```

**例外**：
- 对于 `clearable=true` 且默认值非空的 widget，空值（`?foo=`）会保留在 URL 中
- `string_value` 类型的空字符串 `""` 是有效值（如 text_input）

**关键代码**：
- 默认值判断：[WidgetStateManager.ts#L1322-L1345](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1322-L1345)
- 后端默认值折叠：[session_state.py#L1089-L1097](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1089-L1097)

### 7.3 受保护参数

以下参数不能通过 `st.query_params` 或 widget 绑定操作：

| 参数 | 说明 |
|------|------|
| `embed` | 嵌入模式标记 |
| `embed_options` | 嵌入选项（如 show_loading_screen_v2 等） |

这些参数：
- 迭代/访问时被过滤掉（`__iter__`, `__len__`, `__getitem__`）
- 不能被设置或删除（会抛 `StreamlitAPIException`）
- 跨页面导航时始终保留
- 通过 `isEmbed()` 等专用函数读取

**关键代码**：
- 受保护参数定义：[query_params.py#L35-L45](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L35-L45)
- 保护检查：[query_params.py#L308-L309](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L308-L309)

### 7.4 条件性更新边界

**不更新 URL 的情况**：

1. **非 UI 来源的值变化**：`source.fromUi = false` 时不触发 URL 同步（如后端推送的默认值）
2. **值未实际变化**：`updateUrlParam` 会比较新旧 search，相同则跳过
3. **fragment 运行**：fragment 之外的 widget 绑定不清除（`remove_stale_bindings` 会跳过其他 fragment 的 widget）
4. **绑定参数禁止直接操作**：已绑定到 widget 的参数不能通过 `st.query_params` 直接 set/del/clear

**关键代码**：
- UI 来源判断：[WidgetStateManager.ts#L1413](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1413-L1413)
- 绑定参数保护：[query_params.py#L324-L328](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L324-L328)
- Fragment 保留逻辑：[query_params.py#L808-L813](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L808-L813)

---

## 八、双向同步总结

```
┌─────────────────────────────────────────────────────────────┐
│                     浏览器 URL 地址栏                        │
└─────────────┬───────────────────────────┬───────────────────┘
              │                           │
   popstate  │                           │ pushState/replaceState
              │                           │
┌─────────────▼───────────────────────────▼───────────────────┐
│                    前端 App / WidgetStateManager             │
│  - boundWidgets Map (widgetId → QueryParamBinding)          │
│  - paramKeyToWidgetId Map (paramKey → widgetId)            │
│  - state.queryParams (App 级状态，用于导航保留)             │
└─────────────┬───────────────────────────┬───────────────────┘
              │                           │
  BackMsg.rerunScript                    │ ForwardMsg.page_info_changed
  (queryString + widgetStates)            │ (query_string)
              │                           │
┌─────────────▼───────────────────────────▼───────────────────┐
│                      后端 SessionState                        │
│  - QueryParams 实例                                          │
│    - _query_params: dict (当前参数值)                        │
│    - _initial_query_params: dict (初始 URL 值，用于播种)    │
│    - _bindings_by_param / _bindings_by_widget (绑定关系)    │
│  - _new_widget_state: WStates (前端用户交互值)               │
│  - _new_session_state: dict (用户代码设置值)                 │
│  - _old_state: dict (历史压缩值)                             │
└─────────────────────────────────────────────────────────────┘
```

**核心同步原则**：
1. **前端 URL 是用户交互的第一响应者**：UI 操作先改 URL（replaceState），再发请求
2. **后端是状态权威**：后端通过三道防线防止循环回写，只在必要时推送 URL 校正
3. **URL 参数是可选的**：等于默认值时折叠，只保留有意义的状态
4. **绑定参数和自由参数分离**：绑定参数随 widget 生命周期管理，自由参数由用户代码管理
5. **跨页面两层过滤**：前端清除自由参数，后端按 script_hash 清除其他页面的绑定
