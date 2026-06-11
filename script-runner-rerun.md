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

#### 跨 fragment 批处理冲突的后果（对局部重跑的影响）

冲突规则总结：**"以第一个到达的 fragmentId 为准，告警后强行合并"**。这在真实场景中可能造成以下影响：

##### 场景示例

假设页面有两个独立的局部 fragment：
- **Fragment A** (`fragmentId="a-counter"`)：包含 `counter_button`
- **Fragment B** (`fragmentId="b-form"`)：包含 `name_input`

同一 macrotask 内两者的 widget 几乎同时触发（例如：一个 React 合成事件冒泡链中两个组件都触发了 `onChange`，或某段代码同步 `forEach` 多个组件的 `setState`），调用顺序为：
```
scheduleFlush("a-counter")   ← 第一个到达，scheduledFragmentId = "a-counter"
scheduleFlush("b-form")      ← 第二个到达，与已存值不同 → 触发 LOG.warn
```

##### 实际产生的 BackMsg

最终发出的 `rerun_script` 消息中：
```
fragment_id = "a-counter"     ← 第一个到达的值
widget_states = {
  counter_button: {triggerValue: true},  ← Fragment A 的按钮
  name_input:    {stringValue: "Bob"}    ← Fragment B 的输入（被"错放"）
}
```

##### 后端实际执行路径

1. **接收阶段**：[AppSession.request_rerun()](file:///d:/fz/0601/solo-dogfeeding/code/213-streamlit/lib/streamlit/runtime/app_session.py#L406-L488) 收到消息
   - `fragment_id="a-counter"` 被解析后进入 `request_rerun(rerun_data)`
   - `widget_states` 被整体写入 session_state，**不分 fragment 归属**

2. **排队阶段**：`ScriptRequests.request_rerun(RerunData(fragment_id="a-counter"))`
   - Fragment 队列：`["a-counter"]`（只有 A，没有 B）
   - `widget_states` 内两个 widget 的值都被存入 session state

3. **执行阶段**：
   - 只运行 **Fragment A** 的函数体（因为 fragment_id_queue 只有 `"a-counter"`）
   - Fragment A 的 delta 被发送，其 widget 刷新
   - **Fragment B 的函数体不执行**

4. **最终前端状态**：
   - ✅ `counter_button` 的触发值被正确应用，Fragment A 重新渲染
   - ⚠️ `name_input` 的**值已被存入 session_state**（下次任何 rerun 都会带上）
   - ⚠️ 但 Fragment B 所在的 DOM 容器**没有刷新**（因为 Fragment B 的函数没有重跑）
   - ⚠️ 可能出现"URL bar 显示了新值/调试工具看到新 state，但 Fragment B 的屏幕内容没刷新"的不一致

##### 什么情况会触发此冲突？

典型触发模式（开发者应当避免）：
- 使用自定义 JS 同步地 dispatch 多个组件的输入事件
- 在顶层 `useEffect` 中对多个 fragment 的 widget 批量 `setValue`
- 复杂 React 事件链中，跨 fragment 的组件都在同一个合成事件回调里修改状态
- 测试脚本使用 Playwright/Cypress 同步地 `fill()` 多个位于不同 fragment 的输入框

##### 告警信号

冲突发生时在浏览器 console 会看到类似日志：
```
WidgetStateManager: Unexpected state: Multiple different fragmentIds detected in
a single batch of widget updates. Proceeding with flushing updates using
fragmentId 'a-counter' for this batch.
```

**这是排查"局部重跑没触发"类 bug 的首选信号** —— 如果看到这条 warning，基本可以判定用户交互在同一宏任务内跨多个 fragment 触发了状态变更，需要在组件层面加微任务拆分（`queueMicrotask`/`await 0`）把变更分散到不同 macrotask。

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
