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

## 四、页面状态恢复

### 4.1 初始加载状态恢复

**优先级规则**：
1. **用户交互值**（前端 widget state）> URL 值
2. **初始加载时**：URL 值 > 默认值
3. **后续 rerun 时**：session_state 值 > URL 值

```
首次加载：
  widget 首次注册 → widget_id 不在 _old_state
  → url_value = query_params.get_initial_value(key)
  → 有值 → 播种到 widget state 和 session state
  → 无值 → 使用默认值

后续 rerun：
  widget 已存在 → widget_id 在 _old_state
  → 用户交互值存在（_new_widget_state）→ 使用用户值
  → 代码设置值存在（_new_session_state）→ 使用代码值
  → 否则保持旧值
```

**关键代码**：
- 优先级判断：[session_state.py#L1135-L1146](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1135-L1146)

### 4.2 页面刷新 / 重连恢复

```
页面刷新：
  → 所有前端状态丢失
  → WebSocket 重新连接
  → 后端创建新 session
  → 从 URL 重新播种所有绑定 widget 的值
  → 用户代码设置的 session_state 值丢失

重连（会话保持）：
  → WebSocket 断开后自动重连
  → sessionInfo.last 存在时复用
  → 后端会话仍然存活，状态保留
  → widget 值从后端恢复
```

### 4.3 MPA 页面间状态

- **绑定到 widget 的参数**：跨页面导航时保留（前提是新页面有相同 key 的 widget）
- **非绑定参数**（`st.query_params` 直接设置的）：跨页面导航时被清除
- **embed 参数**：始终保留

**关键代码**：
- 跨页面参数过滤：[WidgetStateManager.ts#L1205-L1238](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1205-L1238)
- 后端页面过滤：[query_params.py#L723-L777](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L723-L777)

---

## 五、URL 更新边界

### 5.1 pushState vs replaceState

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

### 5.2 默认值折叠 (Hide-at-Default)

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

### 5.3 受保护参数

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

### 5.4 条件性更新边界

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

## 六、双向同步总结

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
│  - widget state / session state                              │
└─────────────────────────────────────────────────────────────┘
```

**核心同步原则**：
1. **前端 URL 是用户交互的第一响应者**：UI 操作先改 URL（replaceState），再发请求
2. **后端是状态权威**：后端确认后通过 page_info_changed 再 pushState 一次，确保一致
3. **URL 参数是可选的**：等于默认值时折叠，只保留有意义的状态
4. **绑定参数和自由参数分离**：绑定参数随 widget 生命周期管理，自由参数由用户代码管理
