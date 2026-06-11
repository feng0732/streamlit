# Streamlit ScriptRunner 重跑与调度机制深度分析

## 1. 整体架构概览

Streamlit 的脚本重跑机制是一个**多线程协作**的复杂系统，涉及前后端四层交互：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        前端 (Browser)                                │
│  ┌─────────────────┐     ┌──────────────────────┐                  │
│  │ WidgetStateMgr  │────▶│  ConnectionManager    │                  │
│  │ (状态收集/批处理)│     │  (WebSocket通信)       │                  │
│  └─────────────────┘     └──────────────────────┘                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ WebSocket (BackMsg/ForwardMsg)
┌──────────────────────────────▼──────────────────────────────────────┐
│                     AppSession (主线程/事件循环)                      │
│  ┌─────────────────┐     ┌──────────────────────┐                  │
│  │ handle_backmsg  │────▶│  request_rerun()      │                  │
│  │ (消息分发)       │     │  _create_scriptrunner │                  │
│  └─────────────────┘     └──────────────────────┘                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ on_event (Blinker Signal)
┌──────────────────────────────▼──────────────────────────────────────┐
│                  ScriptRunner (独立脚本线程)                          │
│  ┌─────────────────┐     ┌──────────────────────┐                  │
│  │ ScriptRequests  │────▶│  _run_script_thread   │                  │
│  │ (状态机/队列)    │     │  _maybe_handle_...    │ (yield点检查)   │
│  └─────────────────┘     └──────────────────────┘                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ RerunException/StopException
┌──────────────────────────────▼──────────────────────────────────────┐
│                  执行层 (exec + 用户代码)                              │
│  ┌─────────────────┐     ┌──────────────────────┐                  │
│  │ ScriptRunContext│────▶│  用户脚本 / Fragments  │                  │
│  │ ThreadState     │     │  st.foo() yield点     │                  │
│  └─────────────────┘     └──────────────────────┘                  │
└─────────────────────────────────────────────────────────────────────┘
```

**核心类文件**：
- [script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py) - ScriptRunner 核心实现
- [script_requests.py](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py) - 任务排队状态机
- [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py) - 执行上下文
- [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/app_session.py) - 会话管理层
- [execution_control.py](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/commands/execution_control.py) - st.rerun/st.stop
- [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/lib/src/WidgetStateManager.ts) - 前端状态管理
- [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/app/src/App.tsx) - 前端主控

---

## 2. 重跑触发机制

重跑触发有**多条路径**，最终都汇聚到 `ScriptRequests.request_rerun()`。

### 2.1 前端触发：用户交互（最常见）

#### 流程说明

当用户与 widget 交互时，触发链路如下：

```
用户点击按钮/修改输入
        │
        ▼
Widget 组件调用 WidgetStateManager.setXxxValue()
        │
        ├── 非Form widget: 直接进入批处理
        │       │
        │       ▼
        │   onWidgetValueChanged() 
        │       │
        │       ▼
        │   scheduleFlush(fragmentId)  ──┐
        │                                │ 合并同一macrotask内的多次修改
        └── Form widget: 暂存到 FormState │
                │                        │
                ▼                        │
        syncFormsWithPendingChanges()    │
                │                        │
                ▼                        ▼
        用户点击 Submit ──────────► submitForm()
                                         │
                                         ▼
                                 setTimeout(..., 0) 宏任务级别批处理
                                         │
                                         ▼
                                 sendUpdateWidgetsMessage()
                                         │
                                         ▼
                                 App.sendRerunBackMsg()
                                         │
                                         ▼
                                 ConnectionManager.sendMessage()
                                         │
                                         ▼
                                 BackMsg.rerun_script (WebSocket)
```

#### 关键实现：前端批处理与 fragmentId 冲突

WidgetStateManager 中的 [scheduleFlush](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/lib/src/WidgetStateManager.ts#L1071-L1123) 是前端批处理的核心。它不仅合并 widget 值，还必须处理**同一批次更新来自不同 fragment** 的边缘情况：

```typescript
// 同一 JavaScript macrotask 内的多次 widget 修改被合并为一次 rerun 请求
private scheduleFlush(fragmentId: string | undefined): void {
  // ── 规则 1：scheduledFragmentId 的写入策略 ──
  // 第一次调用：如果当前 stored 为 undefined，则写入本次 fragmentId
  // 后续调用：stored 已存在 → 不再覆盖（第一个到达的 fragmentId 获胜）
  if (this.scheduledFragmentId === undefined) {
    this.scheduledFragmentId = fragmentId
  }

  // ── 规则 2：冲突检测与告警 ──
  // 当已存在的 scheduledFragmentId 与本次传入的 fragmentId
  // 都不为 undefined 且互不相同时，判定为"一批更新来自多个 fragment"
  if (
    this.scheduledFragmentId !== undefined &&
    fragmentId !== undefined &&
    fragmentId !== this.scheduledFragmentId &&
    !this.hasFragmentIdConflict  // 防止同一批内重复告警
  ) {
    // 只发出一次警告，不会抛出异常，不会改变处理策略
    LOG.warn(
      "Unexpected state: Multiple different fragmentIds detected in a single batch of widget updates. Proceeding with flushing updates using fragmentId '%s' for this batch.",
      this.scheduledFragmentId
    )
    this.hasFragmentIdConflict = true  // 置位，本批后续调用不再告警
  }

  // ── 规则 3：宏任务排期（只排一次） ──
  if (this.flushScheduled) {
    return
  }

  this.flushScheduled = true
  setTimeout(() => {
    // 1. 使用"第一个到达的 scheduledFragmentId"发送整个批次
    //    → 即使批次中有来自其他 fragment 的 widget 状态也用同一个 fragmentId
    this.sendUpdateWidgetsMessage(this.scheduledFragmentId)
    
    // 2. 清理 trigger widget 状态（按钮等一次性触发）
    this.pendingTriggerIds.forEach(id => this.deleteWidgetState(id))
    this.pendingTriggerIds.clear()
    this.triggerFlushResolvers.forEach(r => r())
    this.triggerFlushResolvers = []

    // 3. 重置所有调度标志（为本下一次宏任务准备）
    this.flushScheduled = false
    this.scheduledFragmentId = undefined
    this.hasFragmentIdConflict = false
  }, 0)
}
```

**批处理设计的三大动机**：
- 避免一次点击触发多个独立的 rerun 请求（HTTP / WS 报文放大）
- 后端只处理最新的请求，中间请求会丢失（见 `_coalesce_widget_states`）
- 批处理保证所有 widget 修改被一次性送达，不出现"部分先重跑"的不一致

---

#### 跨 fragment 批处理冲突：后端四阶段处理与显示不同步

冲突规则总结：**"以第一个到达的 fragmentId 为准，发出 LOG.warn 告警后强行合并"**。下面以典型场景逐步说明后端的处理链路，以及为什么会出现"值已改、屏未变"的不一致。

##### 场景示例

假设页面有两个独立的局部 fragment：
- **Fragment A** (`fragmentId="a-counter"`)：包含 `counter_button`（带 `on_click=incr` 回调）
- **Fragment B** (`fragmentId="b-form"`)：包含 `name_input`（带 `on_change=validate` 回调）

同一 **JavaScript macrotask**（同一个 React 合成事件回调，或同一段同步 JS）内两者几乎同时触发，调用顺序为：
```
scheduleFlush("a-counter")   ← 第一个到达，scheduledFragmentId = "a-counter"
scheduleFlush("b-form")      ← 第二个到达，与已存值不同 → 触发 LOG.warn
```

##### 阶段 0：实际产生的 BackMsg（前端发送）

最终 `sendUpdateWidgetsMessage()` 发出的**单个** `rerun_script` 消息中：
```
fragment_id = "a-counter"     ← 用第一个到达者（A 的 fragmentId）
widget_states = {
  counter_button: {triggerValue: true},  ← A 的按钮
  name_input:    {stringValue: "Bob"}    ← B 的输入（"混入"同一个包）
}
```
注意：两个 widget 的状态都在包里，但 `fragment_id` 只标了一个。

---

##### 阶段 1：整体接收 —— widget 状态不分归属，全部先写入

消息到达后经 [AppSession.request_rerun()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/app_session.py#L406-L488) 构造 `RerunData`，进入 `ScriptRunner._run_script()`，最终在 `code_to_exec()` 闭包内首先执行：

```python
# ScriptRunner._run_script 中的 code_to_exec() 函数体 [L707-L711]
if rerun_data.widget_states is not None:
    self._session_state.on_script_will_rerun(rerun_data.widget_states)
```

[SessionState.on_script_will_rerun()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/state/session_state.py#L641-L651) 内部三步**不区分 fragment 归属**：

```
① _reset_triggers()        → 清除所有旧 trigger（按钮等一次性值），为新值腾位置
② _compact_state()         → 把 _new_widget_state 合并到 _old_state，产生对比基线
③ set_widgets_from_proto() → 遍历 widget_states 里的每一条，写入 _new_widget_state
                             ↑ 不看 widget 属于哪个 fragment，全部照单全收
④ _call_callbacks()        → 对所有"值发生变化"的 widget 调回调
```

**阶段 1 的关键副作用（对 B 来说已经发生了）**：
- ✅ `name_input` 的新值 `"Bob"` 已经被永久写入 `_new_widget_state`（即使 B 不是目标 fragment）
- ✅ 如果 `validate()` 是 `name_input` 的 `on_change` 回调，它**也会被完整调用**
  - 执行位置：[_call_callbacks()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/state/session_state.py#L653-L692) 遍历所有变化的 widget
  - fragment 感知：回调在 [_execute_widget_callback()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/state/session_state.py#L693-L735) 中被 `ThreadState.scoped(in_fragment_callback=True)` 临时包裹
  - ⚠️ 注意：**只设置了 `in_fragment_callback=True`，没有设置 `fragment_id`**！
  - 所以 `ThreadState.get().fragment_id is None` → 回调内产生的 delta **没有** fragment 归属标签
  - B 的回调如果做了 `st.session_state.x = value` 之类的副作用，也已全局生效

---

###### 阶段 1.5：回调输出边界 —— st.toast / st.error / st.write 会不会直接显示？

这是最容易误解的点。答案是：**会产生消息，但行为和位置都不对**。以下结合代码逐层说明：

**回调执行时的 ThreadState 快照**：

| 字段 | 值 | 原因 |
|------|---|------|
| `active_script_hash` | `main.py` 的哈希 | `ctx.reset()` 时 `ThreadState.initialize()` 设置 |
| `fragment_id` | **`None`** | `_execute_widget_callback` 只设置了 `in_fragment_callback`，没有设置 `fragment_id` |
| `in_fragment_callback` | `True` | `ThreadState.scoped(in_fragment_callback=True)` 临时注入 |
| `delta_path` | `None` | reset 后还没进入任何 fragment 执行 |
| `is_parallel_worker` | `False` | 主线程执行 |

**`st.toast()` / `st.error()` / `st.write()` 的调用链**：

```
用户回调中调用 st.error("Invalid name")
        │
        ▼
DeltaGenerator._enqueue("alert", alert_proto)
        │
        ├─ ① 检查：_maybe_print_fragment_callback_warning()
        │   → 检测到 in_fragment_callback=True
        │   → 发出 CLI 警告："displaying elements is not officially supported
        │      because those elements will replace the existing elements
        │      at the top of your app."
        │
        ├─ ② 构造 ForwardMsg
        │   msg.delta.new_element.alert = alert_proto
        │   msg.metadata.delta_path = dg._cursor.delta_path
        │   ← 此时 cursor 是 reset 后的初始值，路径指向页面顶部
        │
        └─ ③ enqueue_message(msg)
            │
            ├─ 读取 ts.fragment_id → 是 None
            ├─ 因此 msg.delta.fragment_id 保持为空（不打标签）
            │  [enqueue_message() 中代码：](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L474-L479)
            │  if ts.fragment_id and msg.WhichOneof("type") == "delta":
            │      msg.delta.fragment_id = ts.fragment_id
            │
            └─ ctx.enqueue(msg) → 进入 ForwardMsgQueue
```

**关键结论：回调产生的 delta 有三个特征**：
1. ✅ **会被正常发送到前端** —— 它们进入 ForwardMsgQueue，随这次 script run 的所有消息一起 flush
2. ❌ **没有 fragment_id 标签** —— 前端会把它们当作 full-app 的全局 delta
3. ❌ **delta_path 指向页面顶部** —— 因为回调在 `wrapped_fragment()` 之前执行，cursor 还没恢复到 fragment 的位置

---

**各类回调输出的实际表现（按 API 分）**：

| API | 类型 | 是否产生消息 | 显示位置 | 效果 |
|-----|------|-------------|---------|------|
| `st.toast()` | 全局通知 | ✅ 产生 toast delta | 页面右上角（全局） | **正常显示** ✅ —— toast 本来就是全局的，不依赖 fragment 容器 |
| `st.error()` / `st.warning()` / `st.info()` / `st.success()` | Alert 元素 | ✅ 产生 alert delta | **页面顶部**（替换顶部元素） | **位置错乱** ❌ —— 本该在 Fragment B 内显示，结果跑到了页面最顶端 |
| `st.write()` / `st.markdown()` / `st.dataframe()` 等 | 内容元素 | ✅ 产生对应 delta | **页面顶部**（替换/追加） | **位置错乱** ❌ —— 写出的内容出现在错误的位置 |
| `st.rerun()` | 控制流 | — | — | 被捕获为 `RerunException`，转为 `st.warning("Calling st.rerun() within a callback is a no-op.")` 警告，写在顶部 |
| `st.sidebar.write()` | 侧边栏 | ✅ 产生 delta | 侧边栏 | 正常？但 fragment 内写 sidebar 本来就报错，见 `_enqueue` 中的检查 |

---

**哪些显示内容必须等目标 fragment 函数体重跑？**

答案：**所有需要出现在正确 fragment 容器内的显示内容**，都必须等 `wrapped_fragment()` 执行。原因是：

只有在 `wrapped_fragment()` 内部，才会同时满足两个条件：
1. `ThreadState.scoped(fragment_id=fragment_id)` → delta 打上正确的 `fragment_id` 标签
2. `ctx.cursors = deepcopy(cursors_snapshot)` + `context_dg_stack.set(deepcopy(dg_stack_snapshot))` → cursor/路径恢复到该 fragment 声明时的位置，delta_path 正确指向 fragment 容器内

这两个条件缺一不可：
- 只有 fragment_id 但路径不对 → 前端收到带 fragment_id 的 delta 但找不到对应位置
- 只有路径对但没 fragment_id → 前端当作全局 delta，不会局部刷新

**只有目标 fragment 函数体内的输出才能正确地**：
- 被打上 `fragment_id="a-counter"` 标签
- 具有正确的 delta_path（指向 Fragment A 的容器内部）
- 前端收到后精准更新 Fragment A 的 DOM 容器

非目标 fragment 的函数体根本不执行（阶段 2），所以它们内部的任何显示内容都不会更新。

---

##### 阶段 2：函数体选择 —— 只按 `fragment_ids_this_run` 执行

回调全部跑完后才真正执行用户代码，此时 [ScriptRunner._run_script()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L613-L800) 做关键分支：

```python
# [L613-L617]：只从 RerunData.fragment_id 生成队列，B 不在其中
fragment_ids_this_run = self._fragment_storage.order_fragment_ids(
    rerun_data.fragment_id_queue    # queue = ["a-counter"]
)

# [L715-L775]：只遍历队列中的 id，逐个取 wrapped_fragment 执行
if fragment_ids_this_run:
    for fragment_id in fragment_ids_this_run:     # 只会遍历 "a-counter"
        wrapped_fragment = self._fragment_storage.lookup(fragment_id)
        try:
            wrapped_fragment()    ← 只执行 A 的函数体，B 完全跳过
        except Exception:
            pass   ← 异常已在 fragment 内部渲染，不影响其他
else:
    exec(code, module.__dict__)   ← full-app 执行路径（这次不走）
```

**阶段 2 的关键差异（为什么"只执行 A"）**：
- `widget_states` 是"整个页面共享"的 session state，所有 widget 一视同仁先写
- `fragment_ids_this_run` 是"这次要重跑哪些局部"的**调度指令**，由前端传的 `fragment_id` 决定
- 两者是**独立**的两套机制：写入状态 ≠ 刷新对应区域的渲染

---

##### 阶段 3：stale 判断 —— 非目标 fragment 的 widget state 被故意保留

脚本执行完进入 [SessionState.on_script_finished(widget_ids_this_run)](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/state/session_state.py#L866-L946)，这一步**不是"清掉所有没出现过的 widget"**，而是要结合 fragment 语境判断：

```python
# 实际判断函数 [_is_stale_widget()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/state/session_state.py#L1325-L1339)
def _is_stale_widget(metadata, active_widget_ids, fragment_ids_this_run):
    # 一个 widget 只有在以下条件都不满足时才被视为 stale（会被删除）：
    #   ① id 在本次运行期间真正被访问过（widget_ids_this_run）← A 中出现的 widget
    #   ② 或者：本次是 fragment 局部重跑，且该 widget 不属于本次的任何目标 fragment
    #           → B 的 widget 命中此条！保留！
    return not (
        metadata.id in active_widget_ids
        or (fragment_ids_this_run and metadata.fragment_id not in fragment_ids_this_run)
    )
```

**对场景的影响**：
- `counter_button`（A 内）：本次 `wrapped_fragment()` 执行时访问过 → 非 stale → 正常保留
- `name_input`（B 内）：B 没执行，所以没在 `active_widget_ids` 里出现；但因为它是 fragment 局部重跑，且 B 不在目标 fragment 列表中 → 条件② 命中 → **也被保留**
- 结果：**两个 widget 的 state 都完好地留在 session 里**，不会因"本轮没跑 B"而被清掉

---

##### 阶段 4：显示不同步的根本原因 —— 区分"本地 UI 状态"与"后端派生内容"

首先明确 Streamlit 前端控件有两套完全独立的渲染驱动：

| 渲染来源 | 存储位置 | 更新时机 | 依赖后端 delta？ |
|----------|---------|---------|------------------|
| **本地 UI 状态** | React 组件的 `useState()`（`uiValue`、`focused`、`showPassword` 等） | 用户交互时（onChange/onFocus 等）立即更新 | ❌ 不需要 delta |
| **后端派生内容** | Element 树（由 ForwardMsg delta 增量构建） | 收到后端 ForwardMsg 时才更新 | ✅ 完全依赖 |

因此，"非目标区域一定仍显示旧值"的说法**不成立**——实际表现取决于控件本身的渲染逻辑，分三类：

---

###### 4.1 文本输入类（`st.text_input` / `st.number_input`）：有独立本地 UI 状态

以 [TextInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/lib/src/components/widgets/TextInput/TextInput.tsx) 为例，它维护两个分离的状态：

```typescript
// 状态 1：本地 UI 状态（用户正在输入但未 commit 的值）
const [uiValue, setUiValue] = useState<string | null>(...)
const [dirty, setDirty] = useState(false)

// 状态 2：后端同步值（来自 WidgetStateManager 的确认值）
const [value, setValueWithSource] = useBasicWidgetState<...>(...)

// 双向同步规则：[useUpdateUiValue](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/lib/src/hooks/useUpdateUiValue.ts#L28-L44)
useUpdateUiValue(value, uiValue, setUiValue, dirty)
// → 仅当 dirty=false（用户没有正在编辑）时，才把后端 value 同步到 uiValue
```

**commit 触发路径**（非 form 内输入框）：
- 用户每个 keystroke → `onChange` → `setDirty(true)` + `setUiValue(newValue)` → **屏幕立即显示新值**
- 此时**不在 form 内**（`useOnInputChange` [L79-L85](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/lib/src/hooks/useOnInputChange.ts#L79-L85)）→ **不会**立即同步到 WidgetStateManager，不会触发 rerun
- 用户 blur 或按 Enter → `commitWidgetValue()` → `setValueWithSource({value: uiValue, fromUi: true})` → 同步到 WidgetStateManager → `scheduleFlush`

**冲突场景下 TextInput 的实际表现**（非目标 Fragment B 的 name_input）：
| 时点 | 本地 uiValue | 后端 WidgetStateManager 值 | 屏幕显示 | 备注 |
|------|-------------|---------------------------|---------|------|
| 用户刚输入完，未 blur | `"Bob"` | `""`（旧值） | ✅ **显示 "Bob"** | 纯本地 uiValue 驱动，不需要 delta |
| blur 触发 commit → 与 A 同 macrotask 合并 → 后端执行完 | `"Bob"` | `"Bob"`（阶段 1 已写入） | ✅ **仍显示 "Bob"** | uiValue 没变，dirty=false 了，也不用等后端 |
| 后端 on_change 回调内 `st.error("Invalid name")` | `"Bob"` | `"Bob"` | ❌ **看不到错误提示** | 错误提示是后端派生内容，依赖 delta，函数体没执行就没 delta |

**结论**：非目标区域的 TextInput **输入框本身的文字会更新为新值**（因为 onChange 直接写了本地 uiValue），但**所有需要后端代码执行才能产生的派生内容（错误提示、联动显示、toast 等）不会出现**。

---

###### 4.2 表单输入类（`st.form` 内的所有控件 + Checkbox / Button）：状态即显示

**Checkbox / Toggle**（[Checkbox.tsx](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/lib/src/components/widgets/Checkbox/Checkbox.tsx)）：
- **没有**独立的 uiValue，直接用 `useBasicWidgetState` 返回的 `value` 渲染
- `isSelected={value}` → 勾选状态完全由 value 驱动
- `onChange` → `setValueWithSource({value: isSelected, fromUi: true})` → 立即 commit 到 WidgetStateManager
- 但 value 的更新**同时发生在本地**（`useEffect` 在 `nextValueWithSource` 变化时 `setCurrentValue(nextValueWithSource.value)`）

**Button**（[Button.tsx](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/lib/src/components/widgets/Button/Button.tsx)）：
- 完全没有本地持久状态，只触发 `setTriggerValue`
- 按下的视觉反馈是 CSS `:active` 伪类，松开就恢复

**冲突场景下表单类控件的实际表现**（非目标 Fragment B 的控件）：
| 控件类型 | 本地状态 | 后端 state | 屏幕显示 | 派生内容 |
|---------|---------|-----------|---------|---------|
| 非 form 内 Checkbox | ✅ `setCurrentValue(true)` 已设，React 重渲染 | ✅ 已写入 session_state | ✅ **显示为勾选** | 如 `if st.checkbox("Show"): st.dataframe(df)` 的 dataframe → ❌ 不显示（函数体没跑） |
| Form 内的 Checkbox（未点 Submit） | ❌ onChange 只更新本地 value，不触发 scheduleFlush | ❌ 未 commit，还在本地 | ✅ **显示为勾选** | 无（等 Submit 才会发） |
| Form 内的 TextInput（正在输入） | ✅ uiValue="Bob" | ❌ 未 commit，只在本地 widgetMgr 的 FormState | ✅ **显示 "Bob"** | 无 |
| 点击 Form 内 Submit → 与 A 的 button 同 macrotask 合并 | ✅ 所有 form 控件本地状态已更新 | ✅ 全部已写入 session_state | ✅ **所有控件显示新状态** | `st.success("Saved!")`、`st.write(form_data)` 等 → ❌ 不显示 |

**结论**：表单类控件的**交互视觉状态（勾选、输入文字等）都会立即更新**（本地 React 状态驱动），但**表单提交后后端代码产生的任何派生内容（成功提示、计算结果、跳转等）都不会出现**。

---

###### 4.3 纯 delta 依赖区域（`st.write`、`st.dataframe`、图表、`st.metric` 等）：完全依赖后端

这些不是"控件"，没有本地状态，渲染内容 100% 由后端 delta 决定：
- `st.write("Hello, " + st.session_state.name)`
- `st.line_chart(data)`
- `st.metric("Score", score)`
- `st.container()` 内所有由 Python 代码生成的内容

**冲突场景下的表现**：
- ❌ **一定还是旧内容**——没有 delta 就没有任何更新
- 即使 `st.session_state.name` 已经变成 `"Bob"`（阶段 1 已写入），`st.write` 的字符串也不会变，因为那行 Python 代码根本没执行
- 这才是最容易造成"逻辑 vs 显示不一致"的区域：
  - 后端 `validate()` 回调已经把 `st.session_state.valid = True` 改了
  - 但 Fragment B 内的 `if valid: st.success("Valid!") else: st.error("Invalid")` 没执行
  - 屏幕上可能还显示着上一次的"Invalid"红色错误，而实际上 backend 已经认为合法了

---

###### 4.4 综合对比表（非目标 Fragment B）

| 层面 | Fragment A（目标） | Fragment B 内文本输入 | Fragment B 内 checkbox | Fragment B 内 st.write/chart |
|------|--------------------|----------------------|-----------------------|-----------------------------|
| widget state 是否写入 | ✅ 已写入 | ✅ **也已写入** | ✅ **也已写入** | （不涉及 widget） |
| 回调是否执行 | ✅ `incr` 已执行 | ✅ **`validate` 也执行了** | ✅ 回调也执行 | （不涉及回调） |
| 函数体是否执行 | ✅ `wrapped_fragment()` | ❌ 完全跳过 | ❌ 完全跳过 | ❌ 完全跳过 |
| 新 delta 是否产生 | ✅ 所有 `st.foo()` 带 fragment_id 标签 | ⚠️ **只有回调里的**（无 fragment_id、路径错） | ⚠️ **只有回调里的**（无 fragment_id、路径错） | ❌ 0 条 |
| 控件自身显示 | ✅ 正常 | ✅ **显示新输入值**（本地 uiValue 驱动） | ✅ **显示勾选状态**（本地 value 驱动） | ❌ **仍显示旧内容** |
| 回调中的 `st.toast()` | ✅ 正常显示（全局） | ✅ **也会显示**（toast 是全局的，不需要 fragment 容器） | ✅ **也会显示** | （不涉及） |
| 回调中的 `st.error()` / `st.success()` | ✅ 正常显示在 A 内 | ❌ **显示在页面顶部**（位置错乱，不在 B 容器内） | ❌ **显示在页面顶部** | （不涉及） |
| 函数体内的派生内容（提示/图表/业务展示） | ✅ 全部正确显示 | ❌ **不更新**（函数体没跑） | ❌ **不更新**（函数体没跑） | ❌ **不更新** |
| `st.session_state` 调试面板 | ✅ 一致 | ✅ 值已更新 | ✅ 值已更新 | ✅ 值已更新 |

> **关键澄清**：
> - 回调内的 `st.toast()` **会显示**，因为 toast 是全局浮动通知，不依赖 fragment 容器
> - 回调内的 `st.error()` / `st.write()` **也会产生 delta**，但位置不对（跑到页面顶部）
> - Fragment B 函数体内部的所有展示内容（比如 `if valid: st.success(...)`）**不会更新**，因为 B 的函数体根本没执行 |

---

###### 4.5 最终用户体验的完整图景

回到最初的场景（A 是计数器 button，B 是带 `on_change=validate` 的 name_input），用户实际会观察到：

1. ✅ **计数器数字立即 +1**（Fragment A 完全正常，函数体重跑，所有内容正确更新）
2. ✅ **名字输入框中显示 "Bob"**（TextInput 的 uiValue 已更新，不是旧值——本地 React 状态驱动）
3. ✅ **右上角弹出了 "Hello Bob" 的 toast**（`validate()` 回调里的 `st.toast()` 被执行，toast 是全局的，正常显示）
4. ⚠️ 但页面**顶部**多出一个绿色的 "✅ Valid name" 提示框（`validate()` 回调里的 `st.success()` 被执行，但 cursor 指向页面顶部，所以跑到了页面最上面）
5. ❌ 而 **Fragment B 内**的 `if name_valid: st.success("Valid") else: st.error("Invalid")` 仍是红色的 `❌ Invalid name` 提示（上一次的残留，因为 B 的函数体没执行）
6. ✅ URL bar 或调试面板中 `st.session_state.name` 已经是 `"Bob"`，`name_valid` 也是 `True`
7. ✅ 更隐蔽：如果 `validate()` 调了 `requests.post("/api/save", json={"name": name})`，后端数据库已经存了 `"Bob"`
8. 直到下一次 full-app rerun 或用户在 Fragment B 内点击任何东西 → B 的函数体重新执行 → B 容器内的错误提示突然变成成功提示，页面顶部那个"错位的"成功提示仍然在（直到下次 full-app rerun 被清理掉）

**这是最坑的 bug 模式**：
- 控件本身看起来是对的（输入值变了）
- 全局反馈也有了（toast 弹了）
- 但 fragment 内部的业务 UI 没更新（该显示成功的还显示错误）
- 页面顶部还莫名其妙多出一个元素
- 调试看 session_state 又是对的
- 极易让开发者误以为是"前端渲染 bug"或"缓存问题"，实际上是回调输出 + 局部重跑的边界问题

---

##### 什么情况会触发此冲突？

典型触发模式（开发者应当避免）：
- 使用自定义 JS 同步地 dispatch 多个组件的输入事件
- 在顶层 `useEffect` 中对多个 fragment 的 widget 批量 `setValue`
- 复杂 React 事件链中，跨 fragment 的组件都在同一个合成事件回调里修改状态
- 测试脚本使用 Playwright/Cypress 同步地 `fill()` 多个位于不同 fragment 的输入框
- 一个高阶组件的 `onClick` 回调同时通过回调 props 修改了多个子 fragment 的内部状态

##### 告警信号与修复建议

冲突发生时在浏览器 console 会看到类似日志：
```
WidgetStateManager: Unexpected state: Multiple different fragmentIds detected in
a single batch of widget updates. Proceeding with flushing updates using
fragmentId 'a-counter' for this batch.
```

**这是排查"局部重跑没触发 + 状态与显示不一致"类 bug 的首选信号**。

**修复方式（重要：不要用微任务拆分！）**：

因为 `scheduleFlush` 的批处理是基于 **`setTimeout(fn, 0)` 的 macrotask 排期**的，**同一个 macrotask 内的所有 `scheduleFlush` 调用无论插入多少个微任务都会被合并到同一个 flush**里：
- ❌ **错误做法**：在组件间加 `queueMicrotask(() => setValue(...))` 或 `await Promise.resolve(0)` —— 这些仍在同一个 macrotask 内，`flushScheduled=true` 已经被设置，后续调用直接 early return，不会产生第二个 flush。

- ✅ **正确做法**：确保跨 fragment 的状态变更分散到**不同的 macrotask**：
  ```typescript
  // 正确：Fragment A 的变更在当前 macrotask
  widgetStateManager.setTriggerValue(idA, true);

  // Fragment B 的变更放到下一个 macrotask
  setTimeout(() => {
    widgetStateManager.setStringValue(idB, "Bob");
  }, 0);
  ```
  这样会触发**两次独立的 flush → 两个独立的 BackMsg.rerun_script → 两个独立的 fragment rerun**，每个都带自己正确的 `fragment_id`，A 和 B 的函数体都会被执行，UI 和状态保持一致。

如果是 Playwright/Cypress 测试脚本：
  ```typescript
  // ❌ 同步 fill，可能合并
  await page.locator('#inputA').fill('x')  // 同步
  await page.locator('#inputB').fill('y')  // 可能在同一 macrotask 链

  // ✅ 插入事件循环边界（中间加一次 force waitForResponse/networkidle 或显式 wait）
  await page.locator('#inputA').fill('x');
  await page.waitForResponse('**/stream**');  // 等 A 的这次 stream 响应结束
  await page.locator('#inputB').fill('y');
  ```

#### 前端发送 BackMsg

[App.sendRerunBackMsg()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/app/src/App.tsx#L1917-L2011) 组装完整的 rerun 数据：

```typescript
this.sendBackMsg(new BackMsg({
  rerunScript: {
    queryString,           // URL 查询参数
    widgetStates,          // 所有 widget 的当前状态
    pageScriptHash,        // 目标页面哈希
    pageName,              // 页面名称（首次加载时用）
    fragmentId,            // Fragment ID（局部重跑）
    isAutoRerun,           // 是否自动定时重跑
    cachedMessageHashes,   // 前端已缓存的消息哈希
    contextInfo: {         // 浏览器上下文（时区、语言等）
      timezone, locale, url, isEmbedded, colorScheme...
    }
  }
}))
```

### 2.2 后端触发：用户代码内主动调用

#### st.rerun()

在 [execution_control.py](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/commands/execution_control.py#L140-L191) 中实现：

```python
def rerun(*, scope: Literal["app", "fragment"] = "app") -> NoReturn:
    ctx = get_script_run_ctx()
    if ctx and ctx.script_requests:
        ctx.script_requests.request_rerun(RerunData(
            query_string=ctx.query_string,
            page_script_hash=ctx.page_script_hash,
            fragment_id_queue=_new_fragment_id_queue(ctx, scope),
            is_fragment_scoped_rerun=scope == "fragment",
            cached_message_hashes=ctx.cached_message_hashes,
            context_info=ctx.context_info,
        ))
        st.empty()  # 强制触发一个 yield 点，立即执行重跑
```

**关键点**：
- `st.rerun()` 之后调用 `st.empty()` 是为了立即触发 yield 检查
- 支持 `scope="fragment"` 仅重跑当前 fragment
- **在 full-app rerun 过程中调用 fragment scope 会抛异常**

#### st.switch_page()

[switch_page()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/commands/execution_control.py#L195-L347) 会：
1. 解析目标页面路径 → `page_script_hash`
2. 设置 query params
3. `sleep(2 * MESSAGE_FLUSH_INTERVAL_SECS)` 等待前一条消息刷新完成
4. 调用 `request_rerun()` + `st.empty()`

#### st.stop()

[stop()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/commands/execution_control.py#L45-L67) 调用 `request_stop()`，不是重跑而是终止当前执行。

### 2.3 其他触发来源

| 触发源 | 入口函数 | 场景 |
|--------|---------|------|
| 文件变化 | [AppSession._on_source_file_changed()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/app_session.py#L538-L562) | 热重载，代码文件修改后自动重跑 |
| secrets文件变化 | `_on_secrets_file_changed()` | secrets.toml 修改 |
| 工具栏"Always rerun" | `rerunScript(alwaysRunOnSave=True)` | 启用 runOnSave |
| 前端自动重跑 | Fragment 的 `@st.fragment(run_every=...)` | 定时自动重跑 fragment |
| 页面导航切换 | `onPageChange()` | 点击侧边栏页面链接 |

---

## 3. 任务排队与调度逻辑

### 3.1 ScriptRequests 状态机

[ScriptRequests](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L157-L311) 是一个线程安全的有限状态机，有三种状态：

```
           ┌──────────────────────────────────────────┐
           │                                          │
           ▼                                          │
     ┌──────────┐  request_rerun()           ┌─────────────┐
     │ CONTINUE │ ─────────────────────────▶ │    RERUN    │
     └──────────┘                            └──────┬──────┘
           │                                       │
           │ request_stop()                        │ on_yield()
           ▼                                       │ 返回请求并重置
     ┌──────────┐                                  │
     │   STOP   │ ◀────────────────────────────────┘
     └──────────┘
       终端状态，不可逆转
```

#### 三种 ScriptRequestType

```python
class ScriptRequestType(Enum):
    CONTINUE = "CONTINUE"  # 脚本正常运行中
    RERUN    = "RERUN"     # 有待处理的重跑请求
    STOP     = "STOP"      # 终止（不可恢复）
```

### 3.2 重跑请求合并策略

当已经处于 `RERUN` 状态时又收到新的 rerun 请求，会执行**合并（coalescing）**：

#### Widget 状态合并：[_coalesce_widget_states()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L100-L154)

```python
def _coalesce_widget_states(old_states, new_states):
    # 基本原则：新值覆盖旧值
    # 特殊处理：trigger_values（按钮/chat_input 等一次性触发）
    #   - 如果旧状态中 trigger=True，而新状态中对应值未设置
    #   - 则保留旧的 trigger=True，避免按钮点击丢失
```

**场景举例**：
1. 用户点击按钮A → 发送 `{A: trigger=True}` → 设置 `_state=RERUN`
2. 还没处理完，用户又在输入框输入文字 → 发送 `{input: "text"}`
3. 合并结果：`{A: trigger=True, input: "text"}`，两个操作都被保留

#### Fragment 队列合并

```python
if new_data.fragment_id:
    # 新增单个 fragment：追加到队列尾部（去重）
    fragment_id_queue = [*old_queue]
    if new_data.fragment_id not in fragment_id_queue:
        fragment_id_queue.append(new_data.fragment_id)
elif new_data.fragment_id_queue:
    # 提供了完整队列：直接替换
    fragment_id_queue = new_data.fragment_id_queue
else:
    # Full-app rerun：清空 fragment 队列
    fragment_id_queue = []
```

### 3.3 抢占式 vs 非抢占式调度

ScriptRunner 使用**协作式抢占**，不是真正的线程抢占：

#### Yield 点检查：[_maybe_handle_execution_control_request()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L457-L511)

这个函数在**每次 enqueue ForwardMsg 时被调用**（即大多数 `st.foo()` 调用时）：

```python
def _maybe_handle_execution_control_request(self) -> None:
    # 1. 快速路径（不加锁）：
    if self._state == CONTINUE:
        return  # 无事可做，立即返回
    if self._state == RERUN and fragment_run_should_not_preempt():
        return  # Fragment 自动重跑不抢占当前执行
    
    # 2. 慢速路径（加锁）：
    with self._lock:
        if self._state == RERUN:
            self._state = CONTINUE
            raise RerunException(rerun_data)  # 抛出异常中断执行
        if self._state == STOP:
            raise StopException()
```

**关键点**：
- 使用 `BaseException` 子类（`RerunException`、`StopException`）避免被用户的 `try/except Exception` 捕获
- **普通 Fragment rerun 不抢占**：只有 `is_fragment_scoped_rerun=True`（即 `st.rerun(scope="fragment")`）才会中断当前执行
- 非抢占的 fragment rerun 会等待当前脚本跑完后再执行

### 3.4 Fast Reruns 机制

在 [AppSession.request_rerun()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/app_session.py#L406-L488) 中有一个特殊分支：

```python
if runner.fastReruns and not rerun_data.fragment_id:
    # 终止旧 ScriptRunner，创建新 ScriptRunner
    self._scriptrunner.request_stop()
    self._scriptrunner = None
    # ... 创建新的 ScriptRunner 立即执行
else:
    # 正常路径：请求当前 ScriptRunner rerun
    success = self._scriptrunner.request_rerun(rerun_data)
```

**Fast Reruns 的目的**：
- 避免旧线程 cleanup 开销阻塞新执行
- 对于重量级运行中的脚本，快速响应新请求
- **不适用于 fragment rerun**（fragment 需要在同一线程中保留状态）

---

## 4. 执行上下文管理

Streamlit 的上下文系统分**两个层次**，挂载方式因线程类型不同而不同：

| 层次 | 存储方式 | 挂载对象 | 传播机制 |
|------|---------|---------|---------|
| ScriptRunContext | 线程对象属性 (`setattr`) | 每个 `threading.Thread` 实例 | 直接复制引用到子线程对象 |
| FragmentThreadState | `contextvars.ContextVar` | Python 上下文栈（每个 Context 独立） | 父线程快照 → 子线程初始化 / `copy_context()` 完整克隆 |

### 4.1 脚本线程：上下文的创建与自挂载

脚本线程是 Streamlit 中最重要的线程，上下文在 [ScriptRunner._run_script_thread()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L378-L435) 入口处被创建和挂载：

```
新脚本线程启动
      │
      ▼
① 构建 ScriptRunContext 实例
   session_id / _enqueue / script_requests
   session_state / pages_manager 等全部字段
      │
      ▼
② add_script_run_ctx(current_thread, ctx)
   → "自挂载"分支（thread is current_thread）
   a. setattr(thread, SCRIPT_RUN_CONTEXT_ATTR_NAME, ctx)
      └─ 将 ctx 引用直接绑定到当前线程对象
   b. ThreadState.initialize(active_script_hash=main_script_hash)
      └─ 在当前 ContextVar 中注入初始 FragmentThreadState
      （注意：此时 fragment_id=None，还在 full-app 顶层）
      │
      ▼
③ while 循环开始每次 rerun：_run_script(rerun_data)
   └─ 在内部调用 ctx.reset(...)，其中再次调用：
      ThreadState.initialize(active_script_hash=...)
      → 覆盖步骤②的初始化，注入本轮运行正确的 active_script_hash
```

**ScriptRunContext 核心字段**（每个脚本线程一份，引用存储）：

```python
@dataclass
class ScriptRunContext:
    # 会话标识
    session_id: str
    main_script_path: str
    
    # 消息发送通道（回调到 ScriptRunner._enqueue_forward_msg）
    _enqueue: Callable[[ForwardMsg], None]
    
    # 脚本请求引用（用于 st.rerun / st.stop 内部触发）
    script_requests: ScriptRequests | None
    
    # 当前运行数据
    query_string: str
    page_script_hash: str           # 通过 property 代理到 pages_manager
    fragment_ids_this_run: list[str] | None  # None=full app run
    
    # 状态管理
    session_state: SafeSessionState
    widget_ids_this_run: ThreadSafeSet[str]   # 本轮出现过的 widget
    cached_message_hashes: set[str]           # 前端缓存优化
    
    # 并行 fragment 协调（每次 reset 重建新实例）
    parallel_coordinator: ParallelFragmentCoordinator | None
```

#### 重置流程：[reset()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L249-L307)

**每次 rerun 前必须调用 reset**，确保同一脚本线程跨多次运行不残留旧状态：

```python
def reset(self, query_string, page_script_hash, fragment_ids_this_run, ...):
    # 1. 清空所有增量收集器（线程安全集合）
    self.widget_ids_this_run.clear()
    self.widget_user_keys_this_run.clear()
    self.form_ids_this_run.clear()
    self.cursors = {}                 # 清空 delta 路径光标
    self.tracked_commands = []        # 清空性能统计
    
    # 2. 页面与查询参数
    self.pages_manager.set_current_page_script_hash(page_script_hash)
    
    # 3. 重建并行协调器（旧的被 GC，worker 线程池被关闭）
    self.parallel_coordinator = ParallelFragmentCoordinator(
        yield_check=yield_check,  # 即 _maybe_handle_execution_control_request
        max_workers=config["runner.parallelMaxWorkers"],
    )
    
    # 4. 重置 FragmentThreadState（通过 ContextVar.set 覆盖当前上下文）
    ThreadState.initialize(active_script_hash=main_script_hash)
    # → 此时 fragment_id=None；进入具体 fragment 执行时再 update 写入
    
    # 5. 同页 rerun：从 URL 恢复 query_params
    if is_same_page:
        qp.set_initial_query_params(query_string)
        qp.populate_from_query_string(query_string)
```

### 4.2 FragmentThreadState：细粒度上下文状态

使用 Python 3.7+ 的 `contextvars.ContextVar`（**不是** `threading.local`），这是理解跨上下文传播的关键。

[FragmentThreadState](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L72-L93) 字段：

```python
@dataclass(frozen=True)  # 不可变，所有修改通过 dataclasses.replace()
class FragmentThreadState:
    fragment_id: str | None                    # 当前执行的 fragment
    delta_path: tuple[int, ...] | None         # Delta 生成器的当前路径
    in_fragment_callback: bool                 # 是否在 fragment 回调中
    active_script_hash: str                    # 当前页面脚本哈希
    pre_allocated_container_fragment_id: str | None  # 并行 fragment 优化
    is_parallel_worker: bool                   # 是否在并行 worker 线程中
```

**关键设计：ContextVar + frozen dataclass 的组合**

| 特性 | 说明 |
|------|------|
| `ContextVar` 独立于 `threading.local` | 每个 `Context` 对象有自己的 ContextVar 副本；同一个线程内切换 Context 可得到不同的值 |
| `copy_context()` 完整克隆 | 父线程调用 `contextvars.copy_context()` 得到所有 ContextVar 的当前快照，可在其他线程/上下文中运行 |
| frozen dataclass 禁止就地修改 | `ThreadState.update()` 通过 `dataclasses.replace()` 创建新实例再 `ContextVar.set()`，天然线程/上下文安全 |
| 修改不可见性 | 子 Context 中的 `ContextVar.set()` 不会写回父 Context，实现写入隔离 |

[ThreadState API](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L116-L174) 提供四个操作：
- `initialize(**fields)` → 创建全新 FragmentThreadState 并 `ContextVar.set()`
- `get()` → 读取当前 Context 的冻结快照
- `update(**fields)` → `replace()` 后 `set()`，生成新状态
- `scoped(**overrides)` → 上下文管理器，退出时自动恢复旧值（使用 `ContextVar.reset(token)`）

### 4.3 三种线程类型的上下文挂载与传播

Streamlit 实际有**三种线程**挂载上下文的方式，机制完全不同：

---

#### 类型 A：用户自定义子线程（通过 `add_script_run_ctx`）

这是用户代码中最常见的情况，例如：

```python
thread = threading.Thread(target=my_func)
add_script_run_ctx(thread)  # 在父线程（脚本线程）中，子线程启动前调用
thread.start()
```

[add_script_run_ctx()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L344-L436) 执行如下操作：

```
父线程（脚本线程）调用 add_script_run_ctx(child_thread)
      │
      ├─ ① ScriptRunContext：直接挂到子线程对象上
      │    setattr(child_thread, SCRIPT_RUN_CONTEXT_ATTR_NAME, ctx)
      │    注意：是复制引用到子线程对象，不是 ContextVar 方式
      │
      ├─ ② FragmentThreadState：无法跨线程读 ContextVar
      │    ContextVars 不会自动穿越线程边界 → 所以提前在父线程读
      │    parent_ts = ThreadState.get()  ← 得到父线程冻结快照
      │    setattr(child_thread, _FRAGMENT_THREAD_STATE_FIELDS_ATTR, asdict(parent_ts))
      │
      └─ ③ 包装 thread.run（通过方法替换猴子补丁）
           if not installed:
               original_run = child_thread.run
               def _run_with_thread_state(*a, **kw):
                   # 从对象属性读回快照，在子线程的根 Context 中注入
                   fields = getattr(child_thread, _FRAGMENT_THREAD_STATE_FIELDS_ATTR)
                   ThreadState.initialize(**fields)
                   original_run(*a, **kw)
               child_thread.run = _run_with_thread_state
               sentinel = True  ← 防止重复包装
```

**传播结果**：
- ✅ `get_script_run_ctx()` → 从子线程对象属性读，正常工作
- ✅ `ThreadState.get().fragment_id` → 与父线程调用 `add_script_run_ctx` 那一刻的值相同
- ⚠️ 之后父线程再 `ThreadState.update()` → 子线程**看不到**（快照是一次性的，不是 live 的）
- ⚠️ 子线程内的 `ThreadState.update()` → 只改子线程自己的 Context，父线程也看不到

---

#### 类型 B：并行 Fragment Worker（通过 ParallelFragmentCoordinator）

用户不显式创建这些线程，由 `@st.fragment(parallel=True)` 触发。传播在 [ParallelFragmentCoordinator.submit()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/parallel_coordinator.py#L106-L143) 中完成：

```
脚本线程（父）调用 coordinator.submit(wrapped_fragment)
      │
      ├─ ① 双重快照
      │   ctx = get_script_run_ctx()          ← 读线程对象属性
      │   captured = contextvars.copy_context()  ← 克隆所有 ContextVar
      │        （这比 add_script_run_ctx 的 asdict 快照更强：
      │         保留了 in_cached_function 等其他 ContextVar 的值）
      │
      ├─ ② 提交到 ThreadPoolExecutor，包装执行函数
      │   def tracked():
      │       try:
      │           with _scoped_ctx_attach(ctx):
      │               # 临时挂 ScriptRunContext：setattr(current_thread, ctx)
      │               # 退出时恢复线程上之前的 ctx（或删除）
      │               captured.run(fn, *args)
      │                   ↑ 重点：在复制的 Context 内运行 fn
      │                   → FragmentThreadState 等 ContextVar 全部在 captured 内
      │       finally:
      │           _outstanding -= 1; notify
      │
      └─ ③ executor.submit(tracked) → 进入线程池
```

**传播效果（与类型 A 的关键差异）**：

| 维度 | add_script_run_ctx（类型 A） | parallel Coordinator（类型 B） |
|------|------------------------------|--------------------------------|
| ScriptRunContext 挂载 | 永久挂到子线程对象 | 每次调用 `with _scoped_ctx_attach` 临时绑定/恢复（线程池线程复用，不能留脏 ctx） |
| FragmentThreadState 传播 | `asdict()` 快照 → `initialize()` 重建 | `copy_context()` 完整克隆 → `Context.run()`，零信息丢失 |
| 写入可见性 | 双向不可见（各自 Context） | 双向不可见（captured 是独立副本，update 只改 worker 的 Context） |
| `is_parallel_worker` 标志 | 用户线程通常为 False | submit 前已在父线程中 update 为 True，被完整带入 |

**特别注意写入隔离**：
- Worker 中执行 `ThreadState.update(delta_path=(1,2,3))` 不会污染脚本线程
- Worker 中 `in_cached_function.set(True)` 也不会外传
- 因此 worker 之间不会互相干扰，也不会干扰主线程

---

#### 类型 C：脚本线程（自挂载，见 4.1 节回顾）

- 入口调用 `add_script_run_ctx(current_thread, ctx)`
- 进入 `elif ctx is not None and thread is current_thread` 分支
- 直接在当前 ContextVar 中 `ThreadState.initialize(active_script_hash)`
- 后续每次 `ctx.reset()` 覆盖初始化

---

**三种方式的统一检查点**：[enqueue_message()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L468-L479) 是所有 `st.foo()` 调用都会走到的路径，它从两个层次取上下文：

```python
def enqueue_message(msg: ForwardMsg) -> None:
    ctx = get_script_run_ctx()  # → 从线程对象属性读 ScriptRunContext
    if ctx is None: raise NoSessionContext()

    ts = ThreadState.get()      # → 从当前 ContextVar 读 FragmentThreadState
    if ts.fragment_id and msg.type == "delta":
        msg.delta.fragment_id = ts.fragment_id  ← 把 delta 归属到对应 fragment

    ctx.enqueue(msg)
```

这保证了：
- 即使在类型 A/B 子线程中写 `st.write("hello")`，也能正确把消息发送到当前会话的 AppSession
- 如果在 fragment 内（`ts.fragment_id != None`），delta 会被打上 fragment 标签，前端局部重跑时能正确定位刷新范围

---

## 5. 前端刷新配合机制

### 5.1 后端事件 → 前端消息

ScriptRunner 通过 [Blinker Signal](https://pypi.org/project/blinker/) 发出事件，AppSession 在主线程中处理：

```
ScriptRunner (脚本线程)
    │ on_event.send(SCRIPT_STARTED, ...)
    ▼
AppSession._on_scriptrunner_event()
    │ call_soon_threadsafe()  ── 跨线程到事件循环
    ▼
AppSession._handle_scriptrunner_event_on_event_loop()
    │ 根据事件类型 enqueue ForwardMsg
    ▼
ForwardMsgQueue (按顺序存储)
    │ flush_browser_queue() 被 server 定期调用
    ▼
WebSocket → 前端 ConnectionManager → App.onMessage()
```

#### ScriptRunnerEvent 到 ForwardMsg 的映射

| ScriptRunnerEvent | 对应 ForwardMsg | 前端行为 |
|-------------------|----------------|----------|
| `SCRIPT_STARTED` | `new_session` + `session_status_changed` | 重置元素树，开始接收新 delta |
| `ENQUEUE_FORWARD_MSG` | 具体 delta/element 消息 | 追加到元素树渲染 |
| `SCRIPT_STOPPED_WITH_SUCCESS` | `script_finished` + `session_status_changed` | 标记运行结束，清理 inactive widget |
| `SCRIPT_STOPPED_FOR_RERUN` | `script_finished(FINISHED_EARLY_FOR_RERUN)` | 标记提前结束，不清理 |
| `FRAGMENT_STOPPED_WITH_SUCCESS` | `script_finished(FINISHED_FRAGMENT_RUN_SUCCESSFULLY)` | fragment 局部结束 |
| `SCRIPT_STOPPED_WITH_COMPILE_ERROR` | 异常信息 + `script_finished` | 显示编译错误对话框 |
| `SHUTDOWN` | （保存 client_state，内部使用） | - |

### 5.2 NewSession：前端的"重跑信号"

[App.handleNewSession()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/frontend/app/src/App.tsx#L1349-L1442) 是每次 rerun 的前端入口：

```typescript
handleNewSession(newSessionProto: NewSession): void {
  // 1. 标记已收到 new session（用于判断后续 finished 是否匹配本次 rerun）
  this.hasReceivedNewSession = true
  
  // 2. 首次连接或重连：初始化配置、主题等
  if (!this.sessionInfo.isSet) {
    this.handleInitialization(newSessionProto)
  }
  
  // 3. Full-app rerun vs fragment rerun：分支处理
  if (!fragmentIdsThisRun.length) {
    // Full rerun：清理 auto reruns，更新配置，重置导航
    this.cleanupAutoReruns()
    this.processThemeInput(...)
    this.appNavigation.handleNewSession(...)
  }
  
  // 4. 关键：清理 transient nodes（对话框、toast等临时元素）
  //    或清空整个 app state（页面切换/首次加载）
  if (sameApp && samePage) {
    elements = elements.clearTransientNodes(fragmentIdsThisRun)
  } else {
    this.clearAppState(...)  // 全量清空
  }
}
```

### 5.3 消息缓存与增量刷新

为减少网络传输，Streamlit 实现了**ForwardMsg 缓存机制**：

#### 后端侧：发送时检查缓存

[ScriptRunContext.enqueue()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L312-L330)：

```python
def enqueue(self, msg: ForwardMsg):
    # 1. 计算消息哈希（基于内容的 SHA256）
    populate_hash_if_needed(msg)
    
    # 2. 如果消息可缓存且前端已有此哈希 → 发送引用而非完整内容
    if msg.metadata.cacheable and msg.hash in self.cached_message_hashes:
        msg_to_send = create_reference_msg(msg)  # 只有 hash，没有实际内容
    
    self._enqueue(msg_to_send)
```

#### 前端侧：缓存管理

- `ConnectionManager.getCachedMessageHashes()` 返回当前缓存的哈希列表
- 每次 rerun 时，这些哈希通过 `RerunData.cached_message_hashes` 传回后端
- `incrementMessageCacheRunCount()` 管理缓存生命周期（超过年龄的被淘汰）

**效果**：对于重复渲染的图表、文本等元素，第二次 rerun 只发送几字节的引用。

### 5.4 防抖动与状态一致性

#### `hasReceivedNewSession` 标志

```typescript
// 发送 rerun 请求时重置为 false
this.hasReceivedNewSession = false

// 收到 NewSession 后设为 true
// 只有 hasReceivedNewSession=true 时收到的 script_finished
// 才被认为是本次 rerun 的结束
```

**防止场景**：
1. 用户点击按钮 → 发送 rerun 请求1
2. 但请求1还没处理完，用户又触发了请求2
3. 请求1的 NewSession 到了 → `hasReceivedNewSession=true`
4. 请求1 的 script_finished 到了 → 正常处理
5. 请求2 的 NewSession 到了 → 重置为 false 再设为 true
6. 请求2 的 script_finished 到了 → 正常处理

（配合 `AppSession` 忽略"非当前 ScriptRunner"的事件，这是双保险）

---

## 6. 完整时序：一次按钮点击的全流程

以下是用户点击 `st.button("Click me")` 后发生的**完整事件序列**：

```
 T0 前端: 用户点击按钮
    │
    ▼
 T1 WidgetStateManager.setTriggerValue(buttonId, {fromUi: true})
    │  - 设置 widgetState.triggerValue = true
    │  - 加入 pendingTriggerIds
    │  - 调用 scheduleFlush(undefined)
    │
    ▼
 T2 setTimeout(..., 0) 回调执行
    │  - sendUpdateWidgetsMessage(undefined)
    │    → 调用 sendRerunBackMsg(widgetStates, undefined)
    │  - 组装 BackMsg.rerun_script 包含:
    │    {query_string, widget_states={button: trigger=true}, ...}
    │  - WebSocket 发送到后端
    │  - hasReceivedNewSession = false
    │  - 清空 pendingTriggerIds
    │
    ▼
 T3 AppSession.handle_backmsg(msg)  [主线程]
    │  msg_type == "rerun_script"
    │  → _handle_rerun_script_request()
    │
    ▼
 T4 AppSession.request_rerun(client_state)  [主线程]
    │
    ├─► 如果 fastReruns=True 且非 fragment:
    │      scriptrunner.request_stop()
    │      创建新 ScriptRunner → start()
    │
    └─► 否则:
           scriptrunner.request_rerun(rerun_data)
           │  ScriptRequests._state = RERUN
           │  存储 rerun_data
           │
    ▼
 T5 ScriptRunner 线程: 在下次 st.foo() 调用时
    │  _enqueue_forward_msg()
    │    → _maybe_handle_execution_control_request()
    │      → _requests.on_scriptrunner_yield()
    │        返回 ScriptRequest(RERUN, rerun_data)
    │
    ▼
 T6 抛出 RerunException(rerun_data) [脚本线程]
    │  向上冒泡到 exec_func_with_error_handling()
    │  → 捕获 → rerun_exception_data = e.rerun_data
    │  → 清空 cursors 和 dg_stack
    │
    ▼
 T7 _run_script() 中的 while 循环继续
    │  rerun_data = rerun_exception_data  ← 使用新数据
    │  进入下一次迭代
    │
    ▼
 T8 发送事件: SCRIPT_STOPPED_FOR_RERUN  [跨线程]
    │  → AppSession enqueues: script_finished(FINISHED_EARLY_FOR_RERUN)
    │
    ▼
 T9 新一轮 run 开始：
    │  a) ctx.reset(query_string, page_hash, ...) ← 重置上下文
    │  b) 发送事件: SCRIPT_STARTED
    │     → AppSession enqueues: new_session + session_status_changed
    │  c) 编译/执行用户脚本
    │     → 每个 st.foo() enqueue ForwardMsg(delta)
    │
    ▼
 T10 执行完成：
    │  发送事件: SCRIPT_STOPPED_WITH_SUCCESS
    │  → AppSession enqueues: script_finished + session_status_changed
    │  → 更新 file watcher 监听列表
    │
    ▼
 T11 前端: 定期 flush 收到所有 ForwardMsg
    │  a) new_session → handleNewSession()
    │     - 清空 transient nodes
    │     - 准备接收新内容
    │  b) 多个 delta 消息 → 渲染按钮、文本等
    │  c) script_finished → 标记运行状态为 NOT_RUNNING
    │  d) session_status_changed → 隐藏右上角 running 指示器
    │
    ▼
 T12 用户看到更新后的页面 ✓
```

---

## 7. 关键设计模式总结

| 模式 | 应用位置 | 作用 |
|------|---------|------|
| **有限状态机** | `ScriptRequests` (CONTINUE/RERUN/STOP) | 统一请求状态管理，线程安全 |
| **协作式抢占** | `_maybe_handle_execution_control_request` | 避免真正的线程中断，简化资源清理 |
| **异常流控制** | `RerunException`/`StopException` 继承 `BaseException` | 不被用户 try/except 拦截 |
| **宏任务批处理** | `scheduleFlush(setTimeout 0)` | 合并同一事件循环内的多次 widget 修改 |
| **请求合并** | `_coalesce_widget_states` | 避免重复 rerun 丢失按钮触发 |
| **上下文变量** | `contextvars.ContextVar` + `FragmentThreadState` | 并行/异步场景下的安全状态传播 |
| **跨线程桥接** | `loop.call_soon_threadsafe` + Blinker Signal | 脚本线程与事件循环线程的安全通信 |
| **内容哈希缓存** | `ForwardMsg.hash` + `cached_message_hashes` | 显著减少重复渲染的网络流量 |
| **非抢占 fragment** | `_fragment_run_should_not_preempt_script` | 保证 full app 逻辑完整执行 |
| **双保险去重** | 忽略非当前 ScriptRunner 事件 + `hasReceivedNewSession` | 前后端独立过滤过期消息 |

---

## 8. 调试时的关注点

当遇到重跑相关 Bug 时，优先检查以下节点：

1. **Yield 点缺失**：用户代码有长时间运行的循环且不调用任何 `st.foo()`，导致无法及时响应 rerun
2. **Fragment scoped 调用时机**：在 full-app rerun 中调用 `st.rerun(scope="fragment")` 会抛错
3. **Trigger 值合并**：同一 macrotask 内多次点击按钮是否被正确合并
4. **新旧 ScriptRunner 事件竞态**：fastReruns 开启时，旧 runner 的事件应被忽略
5. **线程上下文绑定**：自定义线程池中的 `st.foo()` 调用需先 `add_script_run_ctx()`
6. **前后端状态不一致**：检查 `hasReceivedNewSession` 标志和 `scriptRunId` 是否匹配
