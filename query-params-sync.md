# Query Params 同步路径梳理

## 一、整体架构

Streamlit 的查询参数同步采用 **前后端分离 + WebSocket 消息驱动** 的架构：

- **后端**：Python 侧维护 `QueryParams` 状态，通过 `ForwardMsg.page_info_changed` 向前端推送 URL 变化
- **前端**：TypeScript 侧通过 `WidgetStateManager` 管理 URL 参数绑定，通过 `history.pushState/replaceState` 更新浏览器地址栏
- **通信协议**：WebSocket 双向通道，后端 → 前端用 `ForwardMsg`，前端 → 后端用 `BackMsg`

核心文件：
- 后端：[query_params.py](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py)、[session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py)、[script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py)
- 前端：[WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts)、[App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx)

---

## 二、核心概念：三层状态存储

后端 `SessionState` 中有三个独立的存储区域，理解它们是理解整个同步机制的基础：

| 存储区域 | 类型 | 存储内容 | 生命周期 |
|---------|------|---------|---------|
| `_new_widget_state` | `WStates` | 前端传来的 widget 值（用户交互触发的 rerun 才有值） | 单次脚本执行期间有效，`on_script_finished` 后清理 |
| `_new_session_state` | `dict` | 本次脚本执行中用户代码通过 `st.session_state["k"] = v` 设置的值 | 单次脚本执行期间有效，`on_script_finished` 后合并到 `_old_state` |
| `_old_state` | `dict` | 历史值的压缩存储（上一次脚本执行结束后的快照） | 跨多次脚本执行持久，直到 widget 变为 stale 被清理 |

**关键代码**：[session_state.py#L422-L429](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L422-L429)

读取值的优先级（`_getitem` 方法）[session_state.py#L548-L589](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L548-L589)：
1. `_new_session_state[user_key]`（代码设置值）
2. `_new_widget_state[widget_id]`（用户交互值）
3. `_old_state[widget_id]`（历史 widget 值）
4. `_old_state[user_key]`（历史 session 值）

---

## 三、URL 播种：值落到了哪些状态区

### 3.1 播种触发条件

URL 播种只在以下条件同时满足时发生 [session_state.py#L1135-L1146](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1135-L1146)：

```python
# _handle_query_param_binding 中
if widget_id in self._new_widget_state:
    return False          # ❶ 有用户交互值 → 不播种

is_initial_load = widget_id not in self._old_state
if not is_initial_load and user_key in self._new_session_state:
    return False          # ❷ 非首次加载且代码设置了值 → 不播种

url_value = self.query_params.get_initial_value(user_key)
if url_value is None:
    return False          # ❸ URL 中没有这个参数 → 不播种

return self._seed_widget_from_url(...)  # ❹ 全部通过 → 播种
```

**结论**：URL 播种只在 **首次加载 + URL 有值 + 无用户交互 + 无代码设置** 时发生。

### 3.2 播种后的值去向

`_seed_widget_from_url` 方法中，值被写入 **两个地方** [session_state.py#L1234-L1236](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1234-L1236)：

```python
# Store the value in widget and session state
self._new_widget_state.set_from_value(widget_id, deserialized_value)  # ①
self._new_session_state[user_key] = deserialized_value                # ②
```

| 存储区域 | 写入方式 | key |
|---------|---------|-----|
| `_new_widget_state` | ✅ 写入 | `widget_id`（如 `$$slider-1`） |
| `_new_session_state` | ✅ 写入 | `user_key`（如 `"page"`） |
| `_old_state` | ❌ **不直接写入** | — |

> **重要澄清**：URL 播种不会直接写入 `_old_state`。`_old_state` 只有在脚本运行结束后，通过 `on_script_finished` → `_compact_state()` 将 `_new_widget_state` 和 `_new_session_state` 合并压缩时才会更新。

### 3.3 播种后的自动校正

如果 URL 值经过解析/反序列化后与原始 URL 表示不一致（clamp、去重、格式转换等），后端会自动校正 URL [session_state.py#L1257-L1286](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1257-L1286)：

```python
serialized_value = metadata.serializer(deserialized_value)
if serialized_value != parsed_value:
    # 值经过转换后变化了
    self.query_params.set_corrected_value(user_key, serialized_value, value_type)
    # → 发送 page_info_changed ForwardMsg → 前端 pushState 更新 URL
```

---

## 四、深度分析：前端交互后的回写是否会循环

### 4.1 完整调用链

用户拖动 slider 时，完整同步链路如下（以 slider 从 50 拖到 75 为例）：

```
① 用户拖动 slider (value=75)
  ↓
② 前端 WidgetStateManager.setIntValue(widget, 75, { fromUi: true })
  → maybeSyncValueToUrl(widgetId, { fromUi: true }, 75)
    → source.fromUi == true → 继续
    → updateUrlParam("page", "75", ...)
      → newSearch === currentSearch?  "75" != "50" → 不等
      → window.history.replaceState(...)  ✅ 第一次更新 URL
    → onWidgetValueChanged() → scheduleFlush()
  ↓
③ 前端 sendRerunBackMsg()
  → BackMsg 携带 widget_states: [{id: "$$slider-1", intValue: 75}]
  ↓
④ 后端 script_runner 处理
  → widget_states 存入 SessionState._new_widget_state
  → 脚本执行 → slider 调用 register_widget
  ↓
⑤ 后端 _handle_query_param_binding(...)
  → widget_id in self._new_widget_state → YES
  → return False ❌ 不播种 URL（用户交互优先）
  → url_value_seeded = False
  ↓
⑥ 后端 register_widget 后半段 URL 同步 [session_state.py#L1064-L1097]
  → widget_value != default_value? YES (75 != 50)
  → 分支 1: user_key in _old_state and not has_param?
    → 初始加载后 _old_state 有值，但 _query_params 也有值 → 不进入
  → 分支 2: user_key in _new_session_state and not url_value_seeded?
    → user_key 在 _new_session_state 中吗？
    → 关键：用户交互值只在 _new_widget_state，不在 _new_session_state
    → 所以 → 不进入此分支
  → 分支 3: 值等于默认值且 user_key in _new_session_state?
    → NO (值不等于默认值)
  → else:
    → discard_param_no_forward_msg(user_key)
    → ❌ 不发送 ForwardMsg
  ↓
⑦ 脚本执行完成 → on_script_finished
  → 没有 page_info_changed 消息
  ↓
⑧ 前端 handlePageInfoChanged 不会被调用
  ❌ 不会第二次更新 URL
```

**结论：不会循环更新。**

### 4.2 三道防线

| 防线 | 位置 | 逻辑 |
|------|------|------|
| 防线 1 | 前端 `updateUrlParam` | `newSearch === currentSearch` 时 return，避免无意义更新 |
| 防线 2 | 后端 `_handle_query_param_binding` | `widget_id in _new_widget_state` 时跳过 URL 播种 |
| 防线 3 | 后端 `register_widget` URL 同步 | `user_key in _new_session_state` 为 false（用户交互值在 `_new_widget_state`），不发送 ForwardMsg |

**关键代码**：
- 防线 1：[WidgetStateManager.ts#L1394-L1396](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1394-L1396)
- 防线 2：[session_state.py#L1136-L1137](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1136-L1137)
- 防线 3：[session_state.py#L1067-L1088](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1067-L1088)

### 4.3 后端主动回写的场景

只有 **值变化来源于后端逻辑** 时，后端才会发送 `page_info_changed` 触发第二次 URL 更新：

| 场景 | 触发条件 | 代码位置 |
|------|---------|---------|
| **值丢失恢复** | `user_key in _old_state` 且 `not has_param(user_key)` 且 `not in _new_session_state`（页面导航后 remount、条件渲染重新出现） | [session_state.py#L1067-L1076](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1067-L1076) |
| **编程设置** | `user_key in _new_session_state` 且 `not url_value_seeded` 且值不匹配（`st.session_state["k"] = v`） | [session_state.py#L1077-L1088](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1077-L1088) |
| **编程重置为默认** | `user_key in _new_session_state` 且值等于默认值且 `has_param(user_key)` | [session_state.py#L1089-L1095](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1089-L1095) |
| **URL 自动校正** | 播种时 `serialized_value != parsed_value`（clamp/去重/格式转换） | [session_state.py#L1257-L1286](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1257-L1286) |

---

## 五、深度分析：跨页面切换的真实过滤流程

跨页面切换经历 **前端 + 后端两层过滤**，顺序是：前端先过滤 → 发请求 → 后端再过滤。

### 5.1 第一层：前端过滤

**触发条件**：`pageScriptHash !== currentPageScriptHash && !preserveQueryParams` [App.tsx#L1951](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L1951-L1951)

（浏览器前进/后退时 `preserveQueryParams=true`，跳过前端过滤）

**过滤逻辑** [WidgetStateManager.ts#L1205-L1238](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1205-L1238)：

```typescript
filterParamsForPageChange(embedParams: string): string {
  // 从当前 URL 提取所有绑定 widget 的参数
  this.paramKeyToWidgetId.forEach((_, paramKey) => {
    const values = currentUrl.searchParams.getAll(paramKey)
    // 存入 boundParamsObj
  })
  // 拼接 embed 参数 + 绑定 widget 参数
  return embedParams + boundParamsStr
}
```

**前端过滤结果**：

| 参数类型 | 是否保留 | 判断依据 |
|---------|---------|---------|
| `embed` / `embed_options` | ✅ 保留 | `preserveEmbedQueryParams()` 专门提取 |
| 绑定到 widget 的参数 | ✅ 保留 | `paramKeyToWidgetId` Map 中存在 |
| 自由参数（`st.query_params["foo"]`） | ❌ 清除 | 不在 `paramKeyToWidgetId` 中 |

> 注意：前端的 `paramKeyToWidgetId` 只包含 **当前页面** 已渲染的绑定 widget。

### 5.2 第二层：后端过滤

**触发时机**：`previous_page_script_hash != page_script_hash` 时，在脚本执行 **之前** 调用 [script_runner.py#L578-L611](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L578-L611)

**执行顺序**：
```
① populate_from_query_string(query_string, valid_script_hashes)
  → 按 script_hash 过滤 binding 和参数
② set_initial_query_params_from_current()
  → 用过滤后的 _query_params 设置 _initial_query_params
③ on_script_finished(widget_ids)
  → 常规 stale widget 清理
④ ctx.reset(...)
  → 重置上下文
⑤ 脚本执行
```

**过滤逻辑** [query_params.py#L723-L777](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L723-L777)：

```python
def populate_from_query_string(self, query_string, valid_script_hashes):
    parsed = parse_qs(query_string)
    self.clear_with_no_forward_msg()  # 先清空所有！
    
    for key, val in parsed.items():
        binding = self._bindings_by_param.get(key)
        if valid_script_hashes and binding and binding.script_hash not in valid_script_hashes:
            # 其他页面的 binding → 清除
            stale_widget_ids.append(binding.widget_id)
            continue  # 不保留这个参数
        
        # 保留这个参数
        self.set_with_no_forward_msg(key, val=...)
    
    # 清除其他页面的 binding
    for widget_id in stale_widget_ids:
        self.unbind_widget(widget_id)
```

`valid_script_hashes = {main_script_hash, page_script_hash}`，即：
- **当前页面**的绑定 → 保留
- **主页面（Home.py）**的绑定 → 保留
- **其他页面**的绑定 → 清除

**后端过滤结果**：

| 参数类型 | 是否保留 | 判断依据 |
|---------|---------|---------|
| 当前页面的绑定参数 | ✅ 保留 | `binding.script_hash == page_script_hash` |
| 主页面的绑定参数 | ✅ 保留 | `binding.script_hash == main_script_hash` |
| 其他页面的绑定参数 | ❌ 清除 | binding 存在但 script_hash 不在白名单 |
| URL 中的自由参数 | ✅ 保留（如果没被前端清掉） | 没有 binding 的参数直接保留 |
| embed 参数 | ✅ 保留 | 受保护，不进入 `_query_params` 字典但始终在 URL 中 |

### 5.3 两层过滤的净效果

由于前端已经清除了自由参数，后端的自由参数保留逻辑实际上起不到作用。最终净效果：

| 参数类型 | 跨页面导航后 |
|---------|------------|
| 当前页面 + 主页面的绑定参数 | ✅ 保留 |
| 其他页面的绑定参数 | ❌ 清除（后端按 script_hash 过滤） |
| 自由参数 | ❌ 清除（前端已过滤） |
| embed 参数 | ✅ 始终保留 |

### 5.4 同页面刷新 vs 跨页面导航对比

| 维度 | 同页面刷新（F5） | 跨页面导航（Page A → B） |
|------|----------------|------------------------|
| 前端过滤 | 无（`preserveQueryParams=true`） | 有（`filterParamsForPageChange`） |
| 后端过滤 | 无（同页面，script_hash 不变） | 有（`populate_from_query_string` 按 script_hash） |
| 绑定参数保留 | 全部（URL 原样传给后端） | 只保留当前页 + 主页面的 |
| 自由参数保留 | 全部（URL 原样传） | 全部清除（前端过滤） |
| SessionState | 新 session，全部丢失 | 旧 session，绑定值通过 URL 传递 |
| Widget 值来源 | 重新从 URL 播种 | 从 URL 播种（新页面的 widget） |

---

## 六、深度分析：Stale 绑定值为什么还能保留

### 6.1 什么是 Stale Widget

Stale widget = 上一次脚本运行中存在、但本次脚本运行中没有注册的 widget。常见场景：
- 条件渲染：`if st.checkbox("显示"): st.slider(bind="query-params")` 取消勾选后
- 循环数量减少：`for i in range(n): st.slider(key=f"x_{i}")` 当 n 从 5 变 3 时，`x_3`、`x_4` 变 stale
- 页面结构变化

### 6.2 bound_preserved 机制

`_remove_stale_widgets` 方法中，有一个特殊的 **值保留机制** [session_state.py#L912-L956](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L912-L956)：

```python
def _remove_stale_widgets(self, active_widget_ids):
    # ❶ 清理前：先捕获 bound stale widget 的当前值
    bound_preserved: dict[str, Any] = {}
    for key in self._old_state:
        if (
            is_element_id(key)                    # 是 widget id
            and key in self._query_param_bound_widget_ids  # 是绑定 widget
            and key in wid_key_map                # 有 user_key 映射
            and _is_stale_widget(...)             # 确实是 stale 的
        ):
            user_key = wid_key_map[key]
            bound_preserved[user_key] = self._getitem(key, user_key)  # 读最新值

    # ❷ 清理 _new_widget_state 中的 stale widget
    self._new_widget_state.remove_stale_widgets(...)

    # ❸ 清理 _old_state 中的 stale widget（按 widget_id 存的条目）
    self._old_state = {
        k: v for k, v in self._old_state.items()
        if not is_element_id(k) or not _is_stale_widget(...)
    }

    # ❹ 重新加回：用 user_key 作为键，存回 _old_state
    self._old_state.update(bound_preserved)

    # ❺ 清理 query param bindings 和 URL 参数
    self.query_params.remove_stale_bindings(...)
```

**关键洞察**：值保留了，但 **binding 和 URL 参数被清掉了**。

| 存储 | 变化 |
|------|------|
| `_old_state[widget_id]` | ❌ 被删除（stale widget 清理） |
| `_old_state[user_key]` | ✅ 被保留（通过 `bound_preserved` 重新写入） |
| `_bindings_by_param / _bindings_by_widget` | ❌ 被删除（`remove_stale_bindings`） |
| `_query_params[param_key]` | ❌ 被删除（URL 参数清掉） |
| 浏览器 URL | ❌ 被更新（收到 `page_info_changed` 后清除参数） |

### 6.3 保留的值有什么用

当 widget **重新出现** 时（条件渲染重新勾选、循环数量增加回来），保留的值会被用来恢复状态和 URL：

```
widget 重新注册 → register_widget
  ↓
widget_id not in _old_state  → 是首次注册？
  → 但 user_key in _old_state  → 有值！（bound_preserved 保留的）
  ↓
_handle_query_param_binding:
  → widget_id not in _new_widget_state  → 无用户交互
  → is_initial_load = widget_id not in _old_state = True  → 是初始加载
  → user_key in _new_session_state?  → 视情况
  → 尝试从 URL 播种？  → URL 已经被清掉了 → 失败
  ↓
register_widget 后半段 URL 同步 [session_state.py#L1067-L1076]:
  → widget_value != default_value? YES
  → user_key in _old_state? YES  ✅（bound_preserved 保留的值）
  → not has_param(user_key)? YES  ✅（URL 参数已被清掉）
  → user_key not in _new_session_state? YES
  → 条件全部满足！
  → set_corrected_value(user_key, serialized_value, value_type)
  → _send_query_param_msg()
  → ✅ URL 参数被恢复！
```

**恢复后的效果**：
- widget 值从 `_old_state[user_key]` 恢复（而不是默认值）
- URL 参数被重新写入（通过 `set_corrected_value` → `page_info_changed`）
- 就像 widget 从来没有消失过一样

### 6.4 与跨页面切换的区别

| 场景 | 值是否保留 | URL 参数是否保留 | Binding 是否保留 |
|------|----------|----------------|----------------|
| **条件渲染 widget 消失**（同页面内 stale） | ✅ 值保留在 `_old_state[user_key]` | ❌ URL 参数清除 | ❌ Binding 清除 |
| **跨页面导航**（Page A → B） | ❌ 值丢失（新页面 widget 重新注册） | ❌ URL 参数清除 | ❌ Binding 清除 |
| **浏览器前进后退**（Page B → A） | 视情况（session 是否还在） | ✅ URL 保留所有参数 | ❌ 重建（重新注册时） |

---

## 七、URL 更新边界

### 7.1 pushState vs replaceState

| 场景 | API | 代码位置 |
|------|-----|---------|
| 后端 `page_info_changed` 消息 | `pushState` | [App.tsx#L1149](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L1149-L1149) |
| 前端 widget UI 交互 | `replaceState` | [WidgetStateManager.ts#L1400](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1400-L1400) |
| MPA 页面导航（侧边栏点击） | `pushState` | [App.tsx#L1332](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/app/src/App.tsx#L1332-L1332) |

### 7.2 默认值折叠 (Hide-at-Default)

widget 值等于默认值时，URL 参数会被移除。

**前端判断** [WidgetStateManager.ts#L1322-L1345](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/frontend/lib/src/WidgetStateManager.ts#L1322-L1345)：
```typescript
shouldClearUrlParam(urlValue, binding):
  1. 值为空 且 不允许空 → 清除
  2. 值等于默认值 → 清除
  3. 其他 → 保留
```

**后端判断** [session_state.py#L1178-L1186](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/session_state.py#L1178-L1186)：
```python
if deserialized_value == default_value:
    self._clear_url_param(user_key)
    return False
```

**例外**：`clearable=true` 且默认值非空的 widget，空值 `?foo=` 会保留。

### 7.3 受保护参数

`embed` 和 `embed_options`（大小写不敏感）受到特殊保护：
- 迭代/访问 `st.query_params` 时不可见
- 不能通过 `st.query_params` 设置或删除
- 跨页面导航时始终保留
- 通过 `isEmbed()` / `getEmbedOptions()` 等专用函数读取

**关键代码**：[query_params.py#L35-L45](file:///d:/fz/0601/solo-dogfeeding/code/232-streamlit/lib/streamlit/runtime/state/query_params.py#L35-L45)

### 7.4 条件性更新边界

**不更新 URL 的情况**：

1. **非 UI 来源**：`source.fromUi = false` 不触发前端 URL 同步
2. **值未变化**：`newSearch === currentSearch` 时跳过
3. **Fragment 运行**：其他 fragment 的 widget 不清理 binding
4. **绑定参数禁直接操作**：已绑定的参数不能通过 `st.query_params` 直接 set/del/clear

---

## 八、双向同步总图

```
┌─────────────────────────────────────────────────────────────┐
│                     浏览器 URL 地址栏                        │
└─────────────┬───────────────────────────┬───────────────────┘
              │                           │
   popstate  │                           │ replaceState (UI交互)
              │                           │ pushState (后端推送 / 页面导航)
┌─────────────▼───────────────────────────▼───────────────────┐
│              前端 WidgetStateManager / App                   │
│  boundWidgets Map       paramKeyToWidgetId Map              │
│  state.queryParams (App 级，用于导航保留)                   │
└─────────────┬───────────────────────────┬───────────────────┘
              │                           │
 BackMsg.rerunScript                     │ ForwardMsg.page_info_changed
 (queryString + widget_states)           │ (query_string)
              │                           │
┌─────────────▼───────────────────────────▼───────────────────┐
│                  后端 SessionState                           │
│                                                              │
│  三层状态存储：                                               │
│    _new_widget_state    ← 前端用户交互值                     │
│    _new_session_state   ← 代码 st.session_state[k]=v        │
│    _old_state           ← 历史压缩值                         │
│                                                              │
│  QueryParams：                                                │
│    _query_params          当前参数值                         │
│    _initial_query_params  初始 URL 值（用于播种）           │
│    _bindings_by_param     参数 → binding                    │
│    _bindings_by_widget    widget_id → binding               │
└─────────────────────────────────────────────────────────────┘
```

**核心同步原则**：
1. **前端先响应**：UI 交互先 `replaceState` 改 URL，再发请求
2. **后端防循环**：三道防线确保用户交互不会触发二次回写
3. **URL 是可选项**：等于默认值时折叠，只保留有意义的状态
4. **值比 URL 更持久**：Stale widget 的值通过 `bound_preserved` 保留在 `_old_state`，URL 参数和 binding 被清除，重新出现时恢复
5. **跨页面两层过滤**：前端清自由参数，后端按 script_hash 清其他页面的绑定
