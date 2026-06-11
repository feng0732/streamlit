# Widget ID 生成与冲突解决机制分析

## 一、整体架构概览

Streamlit 的 Widget ID 系统是一个**确定性哈希 + 多层命名空间 + 原子性检测**的组合系统，确保在复杂的多页面、多表单、并行执行场景下，每个 widget 都能获得唯一且稳定的标识。

核心流程：
```
Widget 创建
    ↓
compute_and_register_element_id()  [utils.py#L181-L262]
    ├─ 命名空间组装 (active_script_hash, form_id, container)
    ├─ _compute_element_id()        [utils.py#L149-L178]
    │   └─ 生成格式: $$ID-<hash>-<user_key>
    └─ _register_element_id()       [utils.py#L113-L146]
        ├─ user_key 去重检测
        └─ element_id 去重检测
            └─ ThreadSafeSet.check_and_add()  [thread_safe_set.py#L39-L44]
```

---

## 二、标识生成 (ID Generation)

### 2.1 核心算法：_compute_element_id

**代码位置**：[utils.py#L149-L178](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py#L149-L178)

```python
def _compute_element_id(
    element_type: str,
    user_key: str | None = None,
    **kwargs: SAFE_VALUES | Iterable[SAFE_VALUES],
) -> str:
    h = util.create_fast_hasher()
    h.update(element_type.encode("utf-8"))
    if user_key:
        h.update(user_key.encode("utf-8"))
    for k, v in kwargs.items():
        h.update(str(k).encode("utf-8"))
        h.update(str(v).encode("utf-8"))
    return f"{GENERATED_ELEMENT_ID_PREFIX}-{h.hexdigest()}-{user_key}"
```

**设计要点**：

1. **确定性哈希**：使用快速哈希算法，相同输入永远产生相同输出
2. **输入组成**：
   - `element_type`：widget 类型（如 "button", "slider"）
   - `user_key`：用户提供的唯一键（可选）
   - `**kwargs`：所有影响 ID 的稳定参数

3. **ID 格式**：`$$ID-<hex_digest>-<user_key>`
   - 前缀 `$$ID`：用于快速识别是自动生成的 ID
   - 中间哈希：保证唯一性
   - 后缀 `user_key`：便于调试和反向解析（通过 `user_key_from_element_id` 提取）

**示例 ID**：
- 无 user_key: `$$ID-a1b2c3d4e5f6-None`
- 有 user_key: `$$ID-x9y8z7w6v5u4-my_widget_key`

### 2.2 ID 计算的前置处理：compute_and_register_element_id

**代码位置**：[utils.py#L181-L262](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py#L181-L262)

这是对外暴露的主入口，负责在调用 `_compute_element_id` 之前进行命名空间组装和参数过滤。

---

## 三、命名空间划分 (Namespace Partitioning)

命名空间是避免冲突的第一道防线。通过多层级的命名空间隔离，确保即使 widget 参数完全相同，只要处于不同上下文，ID 也会不同。

### 3.1 四层命名空间

| 层级 | 参数 | 作用 | 代码位置 |
|------|------|------|----------|
| 1 | `active_script_hash` | 页面级隔离，不同页面的相同 widget 获得不同 ID | [utils.py#L249](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py#L249-L249) |
| 2 | `form_id` | 表单级隔离，不同表单内的相同 widget 获得不同 ID | [utils.py#L252](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py#L252-L252) |
| 3 | `active_dg_root_container` | 区域级隔离（主区域 vs 侧边栏） | [utils.py#L256](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py#L256-L256) |
| 4 | `user_key` | 用户自定义的唯一标识 | [utils.py#L167-L172](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py#L167-L172) |

### 3.2 key_as_main_identity 模式

**代码位置**：[utils.py#L232-L243](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py#L232-L243)

当 `user_key` 存在且 `key_as_main_identity` 为 `True` 或 `set` 时，会**忽略大部分命令参数**，只使用：
- 白名单内的 kwargs（当 `key_as_main_identity` 是 set 时）
- `active_script_hash`（始终保留）

**设计意图**：对于某些 widget，用户提供的 key 应该是主要身份标识，参数变化不应导致 ID 变化。

```python
ignore_command_kwargs = user_key is not None and (
    (key_as_main_identity is True) or isinstance(key_as_main_identity, set)
)

if isinstance(key_as_main_identity, set) and user_key:
    kwargs_to_use = {k: v for k, v in kwargs.items() if k in key_as_main_identity}
else:
    kwargs_to_use = {} if ignore_command_kwargs else {**kwargs}
```

### 3.3 命名空间组装逻辑

```python
# 始终添加页面级命名空间
kwargs_to_use["active_script_hash"] = ThreadState.get().active_script_hash

# 仅当不忽略命令参数时，添加表单和容器信息
if dg and not ignore_command_kwargs:
    kwargs_to_use["form_id"] = current_form_id(dg)
    kwargs_to_use["active_dg_root_container"] = dg._active_dg._root_container
```

---

## 四、重复检测 (Duplicate Detection)

### 4.1 检测机制：_register_element_id

**代码位置**：[utils.py#L113-L146](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py#L113-L146)

```python
def _register_element_id(
    ctx: ScriptRunContext, element_type: str, element_id: str
) -> None:
    if not element_id:
        return

    user_key = user_key_from_element_id(element_id)
    
    # 第一步：检测 user_key 重复
    if user_key and not ctx.widget_user_keys_this_run.check_and_add(user_key):
        raise StreamlitDuplicateElementKey(user_key)

    # 第二步：检测 element_id 重复
    if not ctx.widget_ids_this_run.check_and_add(element_id):
        raise StreamlitDuplicateElementId(element_type)
```

**检测逻辑**是**两步式**的：
1. **先查 user_key**：确保用户提供的 key 唯一
2. **再查 element_id**：确保自动生成的 ID 唯一

### 4.2 原子性检测：ThreadSafeSet.check_and_add

**代码位置**：[thread_safe_set.py#L39-L44](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/scriptrunner_utils/thread_safe_set.py#L39-L44)

```python
def check_and_add(self, value: T) -> bool:
    """Atomically check membership and add. Returns True if the value was new."""
    with self._lock:
        is_new = value not in self._data
        self._data.add(value)
        return is_new
```

**关键特性**：
- **原子操作**：在同一个锁内完成「检查是否存在」和「添加」，避免竞态条件
- **线程安全**：通过 `threading.Lock` 保护，支持并行 fragment 执行
- **返回值语义**：`True` 表示是新值（成功），`False` 表示已存在（冲突）

### 4.3 检测集合的生命周期

**代码位置**：[script_run_context.py#L268-L270](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L268-L270)

```python
self.widget_ids_this_run.clear()
self.widget_user_keys_this_run.clear()
self.form_ids_this_run.clear()
```

三个 `ThreadSafeSet` 在每次脚本运行开始时（`reset()` 中）被清空，确保：
- 只检测**同一次运行**内的重复
- 不同运行之间互不干扰

---

## 五、冲突类型与 ID 冲突恢复策略

### 5.1 ID 冲突类型

#### 类型一：StreamlitDuplicateElementKey

**代码位置**：[errors.py#L141-L151](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/errors.py#L141-L151)

**触发条件**：两个 widget 使用了相同的 `key` 参数

**错误消息**：
```
There are multiple elements with the same `key='my_key'`.
To fix this, please make sure that the `key` argument is unique for
each element you create.
```

#### 类型二：StreamlitDuplicateElementId

**代码位置**：[errors.py#L125-L138](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/errors.py#L125-L138)

**触发条件**：两个 widget 虽然没有 user_key，但自动生成的 ID 相同（即所有影响 ID 的参数都相同）

**错误消息**：
```
There are multiple `button` elements with the same auto-generated ID.
When this element is created, it is assigned an internal ID based on
the element type and provided parameters. Multiple elements with the
same type and parameters will cause this error.

To fix this error, please pass a unique `key` argument to the `button` element.
```

### 5.2 ID 冲突的恢复策略：Fail-Fast

对于 ID 冲突（重复 key / 重复自动生成 ID），Streamlit 采用 **Fail-Fast** 策略，直接抛异常，不做自动恢复：

1. **正确性优先**：自动分配不同 ID 可能导致 widget 状态混乱
2. **明确性**：让用户明确知道需要提供唯一标识
3. **可预测性**：ID 是确定性的，用户可以推理为什么会冲突

**用户侧恢复方式**：
- 对于 `StreamlitDuplicateElementKey`：修改重复的 key
- 对于 `StreamlitDuplicateElementId`：为冲突的 widget 添加唯一的 `key` 参数

### 5.3 并发场景的 ID 冲突检测

在并行 fragment 执行场景下，`ThreadSafeSet` 的原子性 `check_and_add` 保证：
- **恰好一个成功**：N 个线程尝试注册相同 ID，只有一个返回 `True`
- **其余全部失败**：其他线程返回 `False` 并抛出异常
- **一致状态**：锁保证了集合状态的一致性

**测试验证**：[thread_safe_set_test.py#L141-L165](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/tests/streamlit/runtime/scriptrunner_utils/thread_safe_set_test.py#L141-L165)

---

## 五-B、绑定 Widget 的值保留与 URL 回写策略

> **注意**：上面第五节讲的是 ID 冲突的恢复（Fail-Fast），但 Streamlit 还有一套完全不同的"恢复"机制——针对 `bind="query-params"` 的绑定 widget，在页面切换或组件卸载/重挂载时**主动保留值并回写 URL 参数**。这是"恢复"的真正核心，不是 Fail-Fast。

### 5B.1 整体流程概览

当一个绑定 widget（`bind="query-params"`）因为页面切换或条件渲染而消失时，系统需要：
1. **在清理阶段保留其值**（而不是直接丢弃）
2. **在 widget 重新注册时恢复该值**
3. **在必要时回写 URL 参数**（保证浏览器地址栏与后端状态一致）

```
脚本运行结束
    ↓
on_script_finished()
    ↓
_remove_stale_widgets()             [session_state.py#L906-L972]
    ├─ 扫描 _old_state 中已绑定且已过期的 widget
    ├─ 通过 self._getitem() 读取最新值（穿透 _new_widget_state → _old_state）
    ├─ 以 user_key 为键存入 bound_preserved
    ├─ 删除过期 widget 的 _old_state 和 _new_widget_state
    └─ 将 bound_preserved 写回 _old_state（user_key 键）

下次脚本运行
    ↓
register_widget()                   [session_state.py#L998-L1109]
    ├─ _handle_query_param_binding() [session_state.py#L1111-L1146]
    │   ├─ 注册绑定关系 (bind_widget)
    │   ├─ 优先级判断：用户交互 > 代码设置 > URL 初始值
    │   └─ 如果 URL 有值且优先级允许 → _seed_widget_from_url()
    ├─ 值解析：widget_id → _old_state[user_key] → 已保留的值
    └─ URL 回写同步（非默认值时的三种情况）
```

### 5B.2 值保留机制：_remove_stale_widgets

**代码位置**：[session_state.py#L906-L972](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/session_state.py#L906-L972)

当脚本运行结束，`on_script_finished` 被调用时，需要清理本轮未出现的 widget。但对于 `bind="query-params"` 的 widget，值不能简单丢弃——否则用户切回页面后已设的值就丢了。

**保留流程**（三步走）：

**第一步：在清理前快照已绑定过期 widget 的值**

```python
bound_preserved: dict[str, Any] = {}
for key in self._old_state:
    if (
        is_element_id(key)                           # 是 widget ID 格式的键
        and key in self._query_param_bound_widget_ids # 且该 widget 是绑定 widget
        and key in wid_key_map                        # 且有 user_key 映射
        and _is_stale_widget(...)                     # 且本轮确实已过期
    ):
        user_key = wid_key_map[key]
        try:
            bound_preserved[user_key] = self._getitem(key, user_key)
        except KeyError:
            bound_preserved[user_key] = self._old_state[key]
```

关键点：
- `self._getitem(key, user_key)` 会穿透 `_new_widget_state` → `_old_state` 的完整查找链，确保读到**最新值**（可能是前端刚推过来的用户交互值，而不只是上一轮的旧值）
- 值以 **user_key** 为键保存（而非 widget_id），这样下次页面重新注册 widget 时即使 widget_id 变了，也能通过 user_key 找到

**第二步：正常清理过期 widget**

```python
self._new_widget_state.remove_stale_widgets(active_widget_ids, ...)
self._old_state = {k: v for k, v in self._old_state.items()
                   if not (is_element_id(k) and _is_stale_widget(...))}
```

**第三步：将保留值回写 _old_state**

```python
self._old_state.update(bound_preserved)
```

这样，已绑定 widget 的值以 user_key 为键存在于 `_old_state` 中，等待下次注册时被恢复。

### 5B.3 值恢复与 URL 播种：_handle_query_param_binding

**代码位置**：[session_state.py#L1111-L1146](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/session_state.py#L1111-L1146)

当绑定 widget 重新注册时，`register_widget` 首先调用 `_handle_query_param_binding`，按优先级决定是否从 URL 播种值：

```python
def _handle_query_param_binding(self, metadata, user_key, widget_id) -> bool:
    # 1. 注册绑定关系
    self.query_params.bind_widget(
        param_key=user_key, widget_id=widget_id,
        value_type=metadata.value_type, script_hash=script_hash,
    )

    # 2. 优先级判断：前端用户交互值 > 代码设置值 > URL 初始值
    if widget_id in self._new_widget_state:   # 用户已交互 → 不播种
        return False
    is_initial_load = widget_id not in self._old_state
    if not is_initial_load and user_key in self._new_session_state:
        return False                          # 代码已设值 → 不播种

    # 3. URL 有值且优先级允许 → 播种
    url_value = self.query_params.get_initial_value(user_key)
    if url_value is None:
        return False
    return self._seed_widget_from_url(metadata, user_key, widget_id, url_value)
```

**优先级规则**（从高到低）：

| 优先级 | 来源 | 条件 | 说明 |
|--------|------|------|------|
| 1 | 前端用户交互 | `widget_id in _new_widget_state` | 用户刚操作过，值最权威 |
| 2 | 代码设置 | `user_key in _new_session_state` | `st.session_state["k"] = v` 设过值 |
| 3 | URL 初始值 | `get_initial_value()` 有返回 | 首次加载或 URL 有值时播种 |

**重要区分**：页面切换回来时，`_remove_stale_widgets` 已经将保留值写入 `_old_state[user_key]`，而 `register_widget` 的首次注册判断条件是：

```python
if (widget_id not in self
    and (user_key is None or user_key not in self)
    and not url_value_seeded):
```

由于 `user_key in self` 会查找 `_old_state`，保留值存在时 **不会** 走首次注册分支，而是走已有值查找分支。这意味着被保留的值自然成为了 widget 的当前值。

### 5B.4 URL 参数回写：register_widget 中的同步逻辑

**代码位置**：[session_state.py#L1063-L1097](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/session_state.py#L1063-L1097)

值解析完成后，`register_widget` 需要同步 widget 值和 URL 参数。这是最精细的部分，分为**非默认值**和**默认值**两大路径：

#### 路径 A：widget 值 ≠ 默认值

```python
if widget_value != default_value:
```

**情况 A1：保留值恢复（页面切换回来）**

```python
if (user_key in self._old_state                    # 值来自 _remove_stale_widgets 保留
    and not self.query_params.has_param(user_key)  # URL 参数丢失（页面切换导致）
    and user_key not in self._new_session_state):  # 非代码设置
    self.query_params.set_corrected_value(user_key, serialized, metadata.value_type)
    restored_bound_value = True
```

触发场景：用户在页面 A 设置了绑定 widget 值，切到页面 B（widget 被清理，值被保留到 `_old_state[user_key]`），再切回页面 A。此时 URL 参数可能已被清理，需要回写。

**关键**：`user_key in self._old_state` 这个守卫条件确保只有通过 `_remove_stale_widgets` 显式保留的值才会触发回写。直接通过 `st.session_state["k"] = v` 设置的值存放在 widget_id 键下，不满足此条件。

**情况 A2：代码设置值同步 URL**

```python
elif (user_key in self._new_session_state          # 代码设置过值
      and not url_value_seeded                     # 不是从 URL 播种的
      and (widget_id in self._old_state
           or user_key in self._old_state)):       # 之前有旧值
    serialized = metadata.serializer(widget_value)
    if not self.query_params.stored_param_matches_corrected_value(
            user_key, serialized, metadata.value_type):
        self.query_params.set_corrected_value(
            user_key, serialized, metadata.value_type)
```

触发场景：用户代码在 widget 注册之前通过 `st.session_state["k"] = new_val` 设置了值，且后端 URL 快照与实际值不一致。此时需要同步 URL，保证刷新后值一致。

**注意**：这里有去重优化——`stored_param_matches_corrected_value` 检查当前存储的 URL 参数是否已经等于新值，避免发送冗余的 `page_info_changed` 消息。

#### 路径 B：widget 值 = 默认值

```python
elif (user_key in self._new_session_state          # 代码设了默认值
      and not url_value_seeded
      and self.query_params.has_param(user_key)    # URL 还有参数
      and (widget_id in self._old_state
           or user_key in self._old_state)):
    self.query_params.remove_param(user_key)        # → 通知前端删除 URL 参数
else:
    self.query_params.discard_param_no_forward_msg(user_key)  # → 仅后端清理，不通知前端
```

**情况 B1**：代码主动重置为默认值 → 调用 `remove_param`，发送 ForwardMsg 让浏览器 URL 也更新。

**情况 B2**：前端已经清掉了 URL 参数（同页 rerun 时默认值折叠），但后端 `_query_params` 缓存未刷新 → 调用 `discard_param_no_forward_msg`，只清理后端缓存，不重复发消息。

### 5B.5 restored_bound_value 的作用

**代码位置**：[session_state.py#L1105-L1107](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/session_state.py#L1105-L1107)

```python
widget_value_changed = (
    user_key is not None and self.is_new_state_value(user_key)
) or restored_bound_value
```

当 `restored_bound_value = True` 时，`widget_value_changed` 也为 True，这意味着：
- 返回给 widget 调用方的 `RegisterWidgetResult.value_changed = True`
- 前端会收到指令更新该 widget 的显示值，而不是使用默认值

这确保了：**页面切换回来后，前端 widget 显示的是恢复的值而非默认值**。

### 5B.6 完整场景还原：页面切换的值保留与回写

```
时间线：
  T1: 用户在页面 A 的 st.selectbox("色", ["红","绿","蓝"], key="color",
      bind="query-params") 选择了 "绿"
      → URL: ?color=Green
      → _old_state["$$ID-xxx-color"] = "绿"

  T2: 用户切换到页面 B
      → on_script_finished() 调用
      → _remove_stale_widgets():
          1. 发现 $$ID-xxx-color 是绑定 widget 且已过期
          2. bound_preserved["color"] = "绿"  (通过 _getitem 读取最新值)
          3. 删除 $$ID-xxx-color 的 _old_state 条目
          4. _old_state["color"] = "绿"  (以 user_key 为键回写)
      → query_params.remove_stale_bindings() 清理绑定关系和 URL 参数

  T3: 用户切回页面 A
      → register_widget() 被调用:
          1. _handle_query_param_binding():
             - widget_id 不在 _new_widget_state → 用户没交互
             - widget_id 不在 _old_state → 首次加载
             - URL 无参数 (被清理了) → url_value_seeded = False
          2. widget_id not in self, 但 user_key="color" in self
             → 不走首次注册分支，走已有值查找
             → self["color"] 找到 _old_state["color"] = "绿"
             → widget_value = "绿"
          3. URL 回写同步:
             - widget_value != default_value ("绿" != "红")
             - user_key "color" in _old_state ✓
             - URL 无参数 ✓
             - user_key 不在 _new_session_state ✓
             → set_corrected_value("color", "Green", "string_value")
             → restored_bound_value = True
          4. widget_value_changed = True → 前端更新 widget 显示 "绿"
      → 最终: widget 显示 "绿", URL 恢复为 ?color=Green
```

### 5B.7 bind=None 时的清理

**代码位置**：[session_state.py#L1023-L1026](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/session_state.py#L1023-L1026)

```python
elif metadata.bind is None and user_key is not None:
    self._query_param_bound_widget_ids.discard(widget_id)
    self.query_params.unbind_and_clear_param(widget_id)
```

如果 widget 之前有 `bind="query-params"` 但现在不再绑定（`bind=None`），系统会：
1. 从 `_query_param_bound_widget_ids` 中移除
2. 解除绑定关系并从 URL 中删除该参数（通过 `unbind_and_clear_param`，会发送 ForwardMsg 更新前端 URL）

---

## 六、ID 解析与辅助函数

### 6.1 user_key_from_element_id

**代码位置**：[common.py#L225-L234](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/common.py#L225-L234)

```python
def user_key_from_element_id(element_id: str) -> str | None:
    user_key: str | None = element_id.split("-", maxsplit=2)[-1]
    return None if user_key == "None" else user_key
```

**作用**：从 element_id 中提取 user_key，用于：
- 冲突检测（`_register_element_id`）
- session_state 的 key ↔ id 映射

### 6.2 ID 类型判断

```python
# 是否是自动生成的 element_id
def is_element_id(key: str) -> bool:
    return key.startswith(GENERATED_ELEMENT_ID_PREFIX)

# 是否是带有 user_key 的 element_id
def is_keyed_element_id(key: str) -> bool:
    return is_element_id(key) and not key.endswith("-None")
```

---

## 七、典型冲突场景分析

### 场景一：循环中创建相同 widget

```python
# ❌ 冲突：三个 button 参数完全相同
for i in range(3):
    st.button("Click me")  # StreamlitDuplicateElementId

# ✅ 解决：添加唯一 key
for i in range(3):
    st.button("Click me", key=f"btn_{i}")
```

### 场景二：侧边栏和主区域有相同 widget

```python
# ✅ 不会冲突：active_dg_root_container 命名空间不同
st.button("Click me")          # 主区域
st.sidebar.button("Click me")  # 侧边栏
```

### 场景三：不同页面有相同 widget

```python
# ✅ 不会冲突：active_script_hash 命名空间不同
# page1.py
st.button("Click me")

# page2.py
st.button("Click me")
```

### 场景四：表单内外有相同 widget

```python
# ✅ 不会冲突：form_id 命名空间不同
st.button("Click me")  # 表单外

with st.form("my_form"):
    st.button("Click me")  # 表单内
```

---

## 八、关键代码索引表

| 功能模块 | 文件 | 行号 |
|---------|------|------|
| ID 计算入口 | [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py) | L181-L262 |
| 核心哈希算法 | [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py) | L149-L178 |
| 重复检测逻辑 | [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/elements/lib/utils.py) | L113-L146 |
| 原子检测集合 | [thread_safe_set.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/scriptrunner_utils/thread_safe_set.py) | L39-L44 |
| 上下文重置（清空集合） | [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py) | L268-L270 |
| user_key 重复异常 | [errors.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/errors.py) | L141-L151 |
| element_id 重复异常 | [errors.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/errors.py) | L125-L138 |
| ID 解析辅助函数 | [common.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/common.py) | L225-L246 |
| 绑定值保留（_remove_stale_widgets） | [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/session_state.py) | L906-L972 |
| Widget 注册与 URL 回写 | [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/session_state.py) | L998-L1109 |
| 查询参数绑定优先级 | [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/session_state.py) | L1111-L1146 |
| URL 播种与自动校正 | [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/session_state.py) | L1148-L1251 |
| 绑定关系注册与清理 | [query_params.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/query_params.py) | L465-L548 |
| 过期绑定清理 | [query_params.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/query_params.py) | L779-L830 |
| MPA 页面切换参数过滤 | [query_params.py](file:///d:/fz/0601/solo-dogfeeding/code/217-streamlit/lib/streamlit/runtime/state/query_params.py) | L723-L777 |

---

## 九、设计权衡总结

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| 确定性哈希 ID | 稳定、可预测、可复现 | 参数变化会导致 ID 变化 |
| 四层命名空间 | 天然隔离大部分冲突 | 理解成本较高 |
| ID 冲突 Fail-Fast | 避免隐性 bug，行为明确 | 要求用户理解 ID 生成规则 |
| ThreadSafeSet 原子检测 | 支持并行执行，无竞态 | 异常可能在任意线程抛出 |
| key_as_main_identity 模式 | 灵活适应不同 widget 需求 | 增加了逻辑复杂度 |
| 绑定值以 user_key 保留 | 页面切换后可恢复，不受 widget_id 变化影响 | _old_state 中 user_key 与 widget_id 混存 |
| URL 回写三路径分支 | 精确覆盖恢复/同步/清理场景 | 代码路径复杂，理解成本高 |
| discard_param_no_forward_msg | 避免冗余 ForwardMsg | 需要区分前端已删 vs 后端缓存过期 |
