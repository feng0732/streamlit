# 状态提示组件机制深度解析

本文档深入解析 Streamlit 中状态提示组件（`st.status`）的工作机制，包括异步反馈、长任务反馈、消息队列和页面刷新之间的关系。

---

## 1. 组件概述

`st.status` 是一个用于显示长任务运行状态的容器组件。它基于 Expandable 块实现，提供三种状态：

| 状态 | 图标 | 说明 |
|------|------|------|
| `running` | spinner 旋转动画 | 任务正在运行中 |
| `complete` | :material/check: | 任务完成 |
| `error` | :material/error: | 任务出错 |

**核心文件：**
- 后端：[mutable_status_container.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/elements/lib/mutable_status_container.py)
- 前端 Expander：[Expander.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/components/elements/Expander/Expander.tsx)
- 前端 StatusWidget：[StatusWidget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/components/StatusWidget/StatusWidget.tsx)

---

## 2. 后端实现（Python 端）

### 2.1 StatusContainer 类

`StatusContainer` 继承自 `DeltaGenerator`，是状态提示组件的核心类。

**关键属性：**
- `_current_proto`: 当前的 BlockProto 配置
- `_current_state`: 当前状态（running/complete/error）
- `_delta_path`: Delta 路径，用于定位组件在页面中的位置

### 2.2 初始化流程

```python
# 在 _create() 静态方法中
1. 创建 BlockProto.Expandable 配置
2. 根据状态设置图标（spinner / :material/check: / :material/error:）
3. 调用 parent._block() 创建块并获取 StatusContainer 实例
4. 保存 delta_path 和 current_proto 用于后续更新
5. time.sleep(0.05) - 防止更新过快导致前端只显示最终状态
```

**重要设计细节：** 初始化后的 50ms 休眠是为了确保前端有时间渲染初始状态。如果初始化后立即调用 `.update()`，可能会因为消息合并或刷新时序问题导致初始状态不显示。

### 2.3 update() 方法

`update()` 方法是实现异步反馈的核心：

```python
def update(self, *, label=None, expanded=None, state=None):
    # 1. 构造 ForwardMsg
    msg = ForwardMsg()
    msg.metadata.delta_path[:] = self._delta_path
    msg.delta.add_block.CopyFrom(self._current_proto)
    
    # 2. 根据参数更新 proto
    if expanded is not None: ...
    if label is not None: ...
    if state is not None:
        # 更新图标
        # running -> "spinner"
        # complete -> ":material/check:"
        # error -> ":material/error:"
    
    # 3. 入队消息
    enqueue_message(msg)
```

关键点：
- 通过 `delta_path` 准确定位要更新的块
- 只更新指定的字段，未指定的字段保持不变
- 调用 `enqueue_message()` 将消息发送到前端

### 2.4 上下文管理器（with 语句）

`StatusContainer` 实现了上下文管理器协议，可以自动管理状态：

```python
with st.status("Processing...") as status:
    # 执行长任务
    do_some_work()
    status.update(label="Halfway there!")
    # ... 继续执行
# 退出 with 块时自动更新为 complete 状态
```

`__exit__` 方法逻辑：
```python
def __exit__(self, exc_type, exc_val, exc_tb):
    if self._current_state == "running":
        time.sleep(0.05)  # 同样的防过快更新机制
        if exc_type is not None:
            self.update(state="error")  # 有异常 -> error
        else:
            self.update(state="complete")  # 无异常 -> complete
    return super().__exit__(...)
```

---

## 3. 消息队列机制

### 3.1 ForwardMsgQueue

[forward_msg_queue.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/forward_msg_queue.py) 是消息队列的核心实现。

**队列结构：**
```
_queue: list[ForwardMsg]        # 消息列表
_delta_index_map: dict          # delta_path -> 队列索引的映射
```

### 3.2 消息入队与合并优化

`enqueue()` 方法会尝试合并相同 delta_path 的 Delta 消息，减少前端渲染次数：

```python
def enqueue(self, msg: ForwardMsg):
    if not _is_composable_message(msg):
        self._queue.append(msg)
        return
    
    delta_key = tuple(msg.metadata.delta_path)
    if delta_key in self._delta_index_map:
        # 尝试合并到已有的同路径消息
        index = self._delta_index_map[delta_key]
        old_msg = self._queue[index]
        composed_msg = _maybe_compose_delta_msgs(old_msg, msg)
        if composed_msg is not None:
            self._queue[index] = composed_msg
            return
    
    # 无法合并，追加新消息
    self._delta_index_map[delta_key] = len(self._queue)
    self._queue.append(msg)
```

**不可合并的消息类型：**
- 非 Delta 消息
- `new_transient` 类型的 Delta（瞬态元素如 spinner）
- `add_block` 类型的 Delta（因为块可能有依赖的子节点）

### 3.3 消息入队的触发链

```
st.status(label, state) 
    → StatusContainer._create()
    → DeltaGenerator._block()
    → ... 最终生成 Delta 消息 ...
    → enqueue_message(msg)  [script_run_context.py]
    → ctx.enqueue(msg)
    → self._enqueue(msg_to_send)  [回调到 AppSession]
    → AppSession._enqueue_forward_msg(msg)
    → self._browser_queue.enqueue(msg)
    → self._message_enqueued_callback()  # 通知 Runtime 有新消息
```

### 3.4 队列清空策略

`clear()` 方法支持保留生命周期消息，用于脚本中断重跑的场景：

```python
def clear(self, retain_lifecycle_msgs=False, fragment_ids_this_run=None):
    if not retain_lifecycle_msgs:
        self._queue = []
    else:
        # 只保留生命周期相关消息
        self._queue = [msg for msg in self._queue 
                       if msg.type in {'new_session', 'script_finished', ...}]
    self._delta_index_map = {}
```

---

## 4. 页面刷新机制

### 4.1 Runtime 主循环

[runtime.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/runtime.py) 中的主循环负责定期刷新消息：

```python
# 简化的主循环逻辑
while True:
    if state == ONE_OR_MORE_SESSIONS_CONNECTED:
        async_objs.need_send_data.clear()
        
        for active_session_info in session_mgr.list_active_sessions():
            msg_list = active_session_info.session.flush_browser_queue()
            for msg in msg_list:
                self._send_message(session_info, msg)
                await asyncio.sleep(0)  # 每个消息后让出控制权
        
        await asyncio.sleep(MESSAGE_FLUSH_INTERVAL_SECS)  # 刷新间隔
    
    # 等待新消息或停止信号
    await asyncio.wait([must_stop, need_send_data], return_when=FIRST_COMPLETED)
```

**两种刷新触发方式：**
1. **定时刷新**：每隔 `MESSAGE_FLUSH_INTERVAL_SECS` 秒自动刷新一次
2. **事件触发**：调用 `need_send_data.set()` 立即触发刷新（消息入队时通过回调触发）

### 4.2 AppSession 中的消息处理

[app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/app_session.py) 是连接 ScriptRunner 和 Runtime 的桥梁。

**关键方法：**

| 方法 | 作用 |
|------|------|
| `flush_browser_queue()` | 清空浏览器队列并返回所有消息 |
| `_enqueue_forward_msg()` | 将消息加入队列并触发回调 |
| `_on_scriptrunner_event()` | 处理 ScriptRunner 事件（线程安全转发） |

### 4.3 ScriptRunner 事件机制

[script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py) 定义了脚本运行的生命周期事件：

```python
class ScriptRunnerEvent(Enum):
    # 控制事件
    SCRIPT_STARTED = "SCRIPT_STARTED"
    SCRIPT_STOPPED_WITH_SUCCESS = "SCRIPT_STOPPED_WITH_SUCCESS"
    SCRIPT_STOPPED_WITH_COMPILE_ERROR = "SCRIPT_STOPPED_WITH_COMPILE_ERROR"
    SCRIPT_STOPPED_FOR_RERUN = "SCRIPT_STOPPED_FOR_RERUN"
    FRAGMENT_STOPPED_WITH_SUCCESS = "FRAGMENT_STOPPED_WITH_SUCCESS"
    SHUTDOWN = "SHUTDOWN"
    
    # 数据事件
    ENQUEUE_FORWARD_MSG = "ENQUEUE_FORWARD_MSG"
```

**事件处理流程：**
```
ScriptRunner 线程（脚本执行）
    → 触发 ScriptRunnerEvent.ENQUEUE_FORWARD_MSG
    → AppSession._on_scriptrunner_event()  [脚本线程中调用]
    → _event_loop.call_soon_threadsafe(...)  [转发到主线程事件循环]
    → AppSession._handle_scriptrunner_event_on_event_loop()  [主线程处理]
    → _enqueue_forward_msg(forward_msg)  [入队]
    → _message_enqueued_callback()  [通知 Runtime]
    → Runtime 主循环刷新消息
```

这种线程安全的事件转发机制确保了：
- 脚本可以在独立线程中运行
- 消息队列的操作始终在主线程事件循环中执行
- 避免了多线程并发修改队列的问题

---

## 5. 前端消息处理

### 5.1 WebSocket 消息接收

前端通过 WebSocket 连接接收后端发送的 ForwardMsg 消息。

[App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx) 中的 `handleMessage()` 方法是消息分发中心：

```typescript
handleMessage = (msgProto: ForwardMsg): void => {
    dispatchProto(msgProto, "type", {
        newSession: (msg) => this.handleNewSession(msg),
        delta: (deltaMsg) => this.handleDeltaMsg(deltaMsg, metadata, hash),
        scriptFinished: (status) => this.handleScriptFinished(status),
        sessionEvent: (evtMsg) => this.handleSessionEvent(evtMsg),
        // ... 其他消息类型
    })
}
```

### 5.2 Delta 消息处理

`handleDeltaMsg()` 方法将 Delta 应用到渲染树：

```typescript
handleDeltaMsg = (deltaMsg, metadataMsg, elementHash?) => {
    this.setState(prevState => ({
        elements: prevState.elements.applyDelta(
            prevState.scriptRunId,
            deltaMsg,
            metadataMsg,
            elementHash
        ),
    }))
}
```

### 5.3 AppRoot 渲染树

[AppRoot.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/render-tree/AppRoot.ts) 管理整个应用的渲染树结构。

**Delta 类型处理：**

| Delta 类型 | 处理方法 | 说明 |
|-----------|----------|------|
| `newElement` | `addElement()` | 添加/替换元素节点 |
| `addBlock` | `addBlock()` | 添加/替换块节点（如 status 容器） |
| `newTransient` | `addTransient()` | 添加瞬态元素 |

**状态更新的具体过程（以 status.update(state="complete") 为例）：**

```
后端发送 Delta(type=addBlock, delta_path=[...])
    → 前端 handleDeltaMsg()
    → AppRoot.applyDelta(scriptRunId, delta, metadata)
    → AppRoot.addBlock(deltaPath, block, scriptRunId, ...)
    → SetNodeByDeltaPathVisitor.setNodeAtPath()  [定位并替换节点]
    → BlockNode 被新的 BlockProto 替换
    → React setState 触发重新渲染
    → Expander 组件读取新的 icon 属性
    → 图标从 spinner 变为 check
```

### 5.4 脚本运行状态与 StatusWidget

[ScriptRunState.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/ScriptRunState.ts) 定义了脚本运行状态：

```typescript
enum ScriptRunState {
  NOT_RUNNING = "notRunning",
  RUNNING = "running",
  RERUN_REQUESTED = "rerunRequested",
  STOP_REQUESTED = "stopRequested",
  COMPILATION_ERROR = "compilationError",
}
```

[StatusWidget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/components/StatusWidget/StatusWidget.tsx) 是顶部状态栏组件，显示整体脚本运行状态：

- 脚本运行时显示 "Running" 和停止按钮
- 脚本文件变更时显示 "Rerun" 提示
- 连接状态异常时显示连接状态

注意：`StatusWidget` 和 `st.status` 是两个不同的概念：
- `StatusWidget`：全局状态栏，显示整个脚本的运行状态
- `st.status`：页面中的状态容器，显示单个任务的状态

### 5.5 脚本结束处理

`handleScriptFinished()` 处理脚本结束事件：

```typescript
handleScriptFinished(status): void {
    if (status === FINISHED_SUCCESSFULLY || 
        status === FINISHED_FRAGMENT_RUN_SUCCESSFULLY) {
        // 清除过期节点
        this.setState(({ scriptRunId, fragmentIdsThisRun, elements }) => ({
            elements: elements.clearStaleNodes(scriptRunId, fragmentIdsThisRun),
        }), () => {
            this.removeInactiveWidgetState()  // 清理不活跃的小部件状态
        })
    }
    // 通知订阅者
    this.state.scriptFinishedHandlers.forEach(handler => handler())
}
```

---

## 6. 长任务反馈完整流程

### 6.1 典型使用场景

```python
import streamlit as st
import time

with st.status("Downloading data...", expanded=True) as status:
    st.write("Searching for data...")
    time.sleep(2)
    status.update(label="Downloading data...", state="running")
    st.write("Found URL.")
    time.sleep(1)
    status.update(label="Download complete!", state="complete")
```

### 6.2 时序详解

下面是一次完整的长任务反馈的时序流程：

```
  用户脚本线程                  Runtime 主循环                前端 React
      |                            |                            |
      |  st.status("Task...")      |                            |
      |  → 创建 BlockProto         |                            |
      |  → 生成 Delta 消息         |                            |
      |  → enqueue_message()       |                            |
      |  → 回调通知有新消息         |                            |
      |                            | need_send_data.set()       |
      |                            | 触发刷新                   |
      |                            | flush_browser_queue()     |
      |                            | 发送 WebSocket 消息        |
      |                            | -------------------------> |
      |                            |                            | handleMessage()
      |                            |                            | → handleDeltaMsg()
      |                            |                            | → AppRoot.applyDelta()
      |                            |                            | → setState() 触发渲染
      |                            |                            | → Expander 显示 spinner
      |  time.sleep(0.05)          |                            |
      |  (防过快更新)               |                            |
      |                            |                            |
      |  status.write("Step 1")    |                            |
      |  → 入队 Delta 消息         |                            |
      |  → 触发刷新通知             |                            |
      |                            |  (定时/事件触发刷新)       |
      |                            | 发送消息                   |
      |                            | -------------------------> |
      |                            |                            | 渲染新内容
      |                            |                            |
      |  status.update(            |                            |
      |    label="Step 2",         |                            |
      |    state="running"         |                            |
      |  )                         |                            |
      |  → 构造 add_block Delta    |                            |
      |  → 更新 icon/label         |                            |
      |  → 入队消息                 |                            |
      |                            | 发送消息                   |
      |                            | -------------------------> |
      |                            |                            | 更新 Expander 状态
      |                            |                            |
      |  ... 任务继续执行 ...       |                            |
      |                            |                            |
      |  status.update(            |                            |
      |    state="complete"        |                            |
      |  )                         |                            |
      |  → 图标变为 check          |                            |
      |                            | 发送消息                   |
      |                            | -------------------------> |
      |                            |                            | 状态更新为完成
      |                            |                            |
      |  退出 with 块               |                            |
      |  __exit__() 检测无异常      |                            |
      |  → (已是 complete，跳过)    |                            |
      |                            |                            |
```

### 6.3 关键技术点

**1. 增量更新而非全量重渲染**

每次 `status.update()` 只发送一个 `add_block` 类型的 Delta 消息，前端只更新对应路径的 BlockNode，不会重渲染整个页面。

**2. 消息合并优化**

如果在两次刷新之间对同一个 status 进行了多次更新，且消息类型可合并，则队列中只保留最后一次的状态。

**但要注意：** `add_block` 类型的 Delta 消息**不会被合并**（见 `_maybe_compose_delta_msgs`），因为块可能有依赖的子节点。多次 update 会产生多条消息。

**3. 防闪烁设计**

- 初始化和退出上下文管理器时的 `time.sleep(0.05)` 确保状态变化可见
- 脚本重跑时保留生命周期消息，避免状态闪烁
- 过期节点清理只在脚本成功结束时执行

---

## 7. 顶部状态栏与页面内状态容器的消息依赖关系

### 7.1 两套独立但关联的状态系统

Streamlit 存在两套状态反馈系统，它们通过不同的消息类型驱动，但共享相同的脚本生命周期事件：

| 系统 | 组件 | 消息来源 | 显示内容 |
|------|------|----------|----------|
| **全局状态栏** | `StatusWidget` | `sessionStatusChanged` / `scriptFinished` / 用户操作 | 整体脚本运行状态、连接状态、重跑提示 |
| **页面内状态容器** | `st.status` (Expander) | `delta` (add_block / newElement) | 单个任务的运行进度、详细信息 |

### 7.2 消息类型与触发关系

一次完整的脚本运行会产生以下消息序列，同时驱动两套系统：

```
后端 ScriptRunner 线程
    │
    ├─→ SCRIPT_STARTED 事件
    │      ├─→ 发送 new_session 消息 (script_run_id, scriptIsRunning=true)
    │      │      └─→ 前端 handleNewSession()
    │      │              ├─→ 更新 scriptRunId (关键：新的运行ID)
    │      │              ├─→ clearTransientNodes()
    │      │              └─→ handleSessionStatusChanged(scriptIsRunning=true)
    │      │                      └─→ scriptRunState = RUNNING
    │      │                              └─→ StatusWidget 显示 "Running + Stop"
    │      │
    │      └─→ 进入用户脚本执行
    │
    ├─→ 用户执行 st.status("任务A")
    │      └─→ 发送 delta(add_block) 消息
    │              └─→ 前端 handleDeltaMsg()
    │                      └─→ AppRoot.applyDelta()
    │                              └─→ 创建 Expander 节点，图标=spinner
    │
    ├─→ 用户执行 status.update(state="complete")
    │      └─→ 发送 delta(add_block) 消息
    │              └─→ 前端 handleDeltaMsg()
    │                      └─→ AppRoot.addBlock() 替换同路径节点
    │                              └─→ Expander 图标变为 check
    │
    └─→ SCRIPT_STOPPED_WITH_SUCCESS 事件
           ├─→ 发送 script_finished (FINISHED_SUCCESSFULLY) 消息
           │      └─→ 前端 handleScriptFinished()
           │              ├─→ clearStaleNodes() 清理旧节点
           │              ├─→ removeInactiveWidgetState() 清理 widget
           │              └─→ incrementMessageCacheRunCount()
           │
           └─→ (如果新脚本开始，则循环回到顶部)
```

### 7.3 关键消息的相互依赖

**1. `new_session` 消息：渲染树的新运行锚点**

[app_session.py#L783-L858](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/app_session.py#L783-L858)

`new_session` 消息的作用是标记新一次脚本运行的开始，为渲染树建立新的锚点：
- 携带 **`script_run_id`**：这是后续所有 Delta 消息的"归属标签"，用于判断节点是否过期
- 携带 **`fragment_ids_this_run`**：标记本次是全量运行还是 fragment 运行
- 携带 **`initialize.session_status`**：仅在首次初始化/重连时通过 `handleInitialization()` 间接设置 `scriptRunState`；普通运行时不改变 `scriptRunState`

**`new_session` 不负责切换 StatusWidget 状态**。在普通运行中，`handleNewSession()` 只更新 `scriptRunId` 和清除瞬态节点，完全不触碰 `scriptRunState`。顶部状态栏的 RUNNING / NOT_RUNNING 切换由 `session_status_changed` 消息驱动。

前端收到 `new_session` 后 [App.tsx#L1349-L1441](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L1349-L1441)：
```typescript
// 同页面、同应用的正常重跑（普通运行）
this.setState(prevState => ({
    // 先清除瞬态节点（如 spinners）
    elements: prevState.elements.clearTransientNodes(fragmentIdsThisRun),
    scriptRunId,  // 更新 scriptRunId —— 这是 new_session 的核心职责
    // 注意：没有 scriptRunState 更新！
}))
```

**2. `delta` 消息：页面内状态更新的载体**

每个 Delta 消息都会被打上当前 `scriptRunId` 的标签：
```typescript
// AppRoot.applyDelta() 中
this.addBlock(deltaPath, block, scriptRunId, ...)
// → 新创建的 BlockNode.scriptRunId = 当前 scriptRunId
```

**3. `script_finished` 消息：节点清理的触发点**

`script_finished` **不改变 `scriptRunState`**。它的唯一职责是：
- 对 `FINISHED_SUCCESSFULLY` / `FINISHED_FRAGMENT_RUN_SUCCESSFULLY`：执行 `clearStaleNodes()` + `removeInactiveWidgetState()` + `incrementMessageCacheRunCount()`
- 对 `FINISHED_EARLY_FOR_RERUN`：仅通知 `scriptFinishedHandlers`，不清理
- 对 `FINISHED_WITH_COMPILE_ERROR`：不执行任何操作

顶部状态栏切回 NOT_RUNNING 由紧随 `script_finished` 之后到达的 `session_status_changed(scriptIsRunning=false)` 完成，而非 `script_finished` 本身。

### 7.4 StatusWidget 状态机

[StatusWidget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/components/StatusWidget/StatusWidget.tsx) 中的状态转换逻辑。核心原则：**`scriptRunState` 切换唯一由 `session_status_changed` 消息驱动**，`new_session` 和 `script_finished` 从不直接修改 `scriptRunState`。

#### 各消息职责边界

| 消息 | 对 `scriptRunState` 的影响 | 核心职责 |
|------|--------------------------|---------|
| **`session_status_changed(scriptIsRunning=true)`** | ✅ 切到 `RUNNING` | 唯一驱动进入运行状态的消息 |
| **`session_status_changed(scriptIsRunning=false)`** | ✅ 切到 `NOT_RUNNING` | 唯一驱动回到未运行状态的消息 |
| **`new_session`** | ❌ 不直接修改（普通运行）<br>✅ 首次/重连：通过 `handleInitialization` 间接调用 `handleSessionStatusChanged` | 分发 `script_run_id`、配置、页面信息，清除瞬态节点 |
| **`script_finished`** | ❌ 从不修改 | 触发节点清理、Widget 清理、缓存清理 |

---

#### 普通运行场景的完整状态机

```
初始状态: NOT_RUNNING

[1] 用户点击按钮 → 前端发送 BackMsg 到后端
       ↓
[2] 后端 SCRIPT_STARTED 事件
       ├─→ 发送 new_session (script_run_id=A)
       │     └─→ 前端 handleNewSession(): 更新 scriptRunId，清除瞬态节点
       │         ❗ scriptRunState 仍为 NOT_RUNNING（普通运行不走 handleInitialization）
       │
       └─→ 函数末尾检测状态变化
             └─→ 发送 session_status_changed(scriptIsRunning=true)
                   └─→ 前端 handleSessionStatusChanged()
                         ├─→ 条件：scriptIsRunning=true && state≠STOP_REQUESTED
                         └─→ scriptRunState = RUNNING  ← 切到运行中
       ↓
[3] 运行 500ms 后 (RUNNING_MAN_DISPLAY_DELAY_TIME_MS)
       └─→ showRunningMan = true
       └─→ 显示 "Running Man" 动画 + Stop 按钮
       ↓
[4] 脚本执行完成 → 后端 SCRIPT_STOPPED_WITH_SUCCESS 事件
       ├─→ 发送 script_finished(FINISHED_SUCCESSFULLY)
       │     └─→ 前端 handleScriptFinished()
       │           ├─→ clearStaleNodes() 清理节点
       │           ├─→ removeInactiveWidgetState() 清理 Widget
       │           └─→ ❗ scriptRunState 仍为 RUNNING（从不修改 scriptRunState）
       │
       └─→ 函数末尾检测状态变化
             └─→ 发送 session_status_changed(scriptIsRunning=false)
                   └─→ 前端 handleSessionStatusChanged()
                         ├─→ 条件：!scriptIsRunning && state≠RERUN_REQUESTED && state≠COMPILATION_ERROR
                         └─→ scriptRunState = NOT_RUNNING  ← 切回未运行
```

---

#### 重跑过渡场景的完整状态机

```
当前状态: NOT_RUNNING（脚本已完成）

[1] 用户点击 Rerun 按钮
       └─→ 前端 rerunScript() 被调用
             ├─→ this.setState({ scriptRunState: RERUN_REQUESTED })  ← 前端乐观更新
             └─→ 发送 BackMsg(rerun_script) 到后端
       ↓
[2] 后端中断旧脚本 → SCRIPT_STOPPED_FOR_RERUN 事件
       └─→ 发送 script_finished(FINISHED_EARLY_FOR_RERUN)
             └─→ 前端 handleScriptFinished()
                   ├─→ ❌ 跳过 clearStaleNodes（防闪烁）
                   └─→ ❗ scriptRunState 仍为 RERUN_REQUESTED
       ↓
[3] 后端启动新脚本 → SCRIPT_STARTED 事件
       ├─→ 发送 new_session (script_run_id=B)
       │     └─→ 前端 handleNewSession(): 更新 scriptRunId，清除瞬态节点
       │         ❗ scriptRunState 仍为 RERUN_REQUESTED
       │
       └─→ 函数末尾检测状态变化
             └─→ 发送 session_status_changed(scriptIsRunning=true)
                   └─→ 前端 handleSessionStatusChanged()
                         ├─→ 条件：scriptIsRunning=true && state≠STOP_REQUESTED
                         ├─→ RERUN_REQUESTED ≠ STOP_REQUESTED → 条件满足
                         └─→ scriptRunState = RUNNING  ← 切到运行中
       ↓
[4] 运行 500ms 后 → showRunningMan = true
       ↓
[5] 新脚本完成 → SCRIPT_STOPPED_WITH_SUCCESS 事件
       ├─→ 发送 script_finished(FINISHED_SUCCESSFULLY)
       │     └─→ 前端 handleScriptFinished(): 清理节点
       │           ❗ scriptRunState 仍为 RUNNING
       │
       └─→ 发送 session_status_changed(scriptIsRunning=false)
             └─→ 前端 handleSessionStatusChanged()
                   ├─→ 条件：!scriptIsRunning && state≠RERUN_REQUESTED && state≠COMPILATION_ERROR
                   ├─→ RUNNING ≠ RERUN_REQUESTED → 条件满足
                   └─→ scriptRunState = NOT_RUNNING  ← 切回未运行
```

---

#### 状态切换条件详解 [App.tsx#L1225-L1250](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L1225-L1250)

**切到 RUNNING 的条件：**
```typescript
if (scriptIsRunning && prevState !== STOP_REQUESTED) {
    scriptRunState = RUNNING
}
```
- 如果用户已经点击了 Stop（状态为 `STOP_REQUESTED`），则忽略运行中的通知

**切到 NOT_RUNNING 的条件：**
```typescript
if (!scriptIsRunning && 
    prevState !== RERUN_REQUESTED && 
    prevState !== COMPILATION_ERROR) {
    scriptRunState = NOT_RUNNING
}
```
- 如果用户已经点击了 Rerun（状态为 `RERUN_REQUESTED`），则忽略"脚本已停止"的通知（因为新脚本马上要开始）
- 如果脚本有编译错误（状态为 `COMPILATION_ERROR`），则保留错误状态不覆盖

关键的防闪烁设计：脚本开始运行后不会立即显示 Running Man，而是等待 500ms。如果脚本在 500ms 内完成，用户就不会看到短暂的"闪烁"效果。

---

## 8. 三种生命周期消息的功能与发送时机

Streamlit 使用三种核心生命周期消息驱动全局状态流转。它们的功能、发送时机和接收后的行为各不相同，共同构成状态反馈的骨架。

### 8.1 消息类型功能对照表

| 消息类型 | 后端生成方法 | 核心功能 | 携带关键字段 |
|----------|------------|---------|------------|
| **`new_session`** | `_create_new_session_message()` | 标记新一次脚本运行的开始，分发运行配置 | `script_run_id`、`scriptIsRunning`、`fragment_ids_this_run`、`config`、`theme`、`initialize.session_status`、`pages` |
| **`session_status_changed`** | `_create_session_status_changed_message()` | 通知运行状态或 Run-On-Save 设置变更 | `script_is_running`、`run_on_save` |
| **`script_finished`** | `_create_script_finished_message()` | 标记脚本运行结束，触发清理 | `status`（5 种枚举值） |

### 8.2 `new_session` 消息详解

**发送时机** [app_session.py#L680-L686](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/app_session.py#L680-L686)：

```python
# 触发 ScriptRunnerEvent.SCRIPT_STARTED 时
self._clear_queue(fragment_ids_this_run)  # 先清空旧队列（保留生命周期消息）
msg = self._create_new_session_message(
    page_script_hash, fragment_ids_this_run, pages
)
self._enqueue_forward_msg(msg)
```

在以下事件后必定发送：
- 新用户首次连接（页面打开）
- 用户点击 Rerun / Always Rerun
- 用户修改 Widget（如按下按钮）
- Fragment 触发重跑
- 连接断开后重连时（由前端 `sendUpdateWidgetsMessage` 触发）

**前端接收处理** [App.tsx#L1349-L1441](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L1349-L1441)：

```
handleNewSession(newSessionProto)
    ├─→ 首次连接或断线重连 → handleInitialization()
    │      └─→ sessionInfo.setCurrent() + handleSessionStatusChanged()
    ├─→ 非 Fragment 运行
    │      ├─→ cleanupAutoReruns()
    │      ├─→ processThemeInput()
    │      ├─→ setState 更新 config、toolbarMode 等
    │      └─→ appNavigation.handleNewSession()
    ├─→ Fragment 运行
    │      └─→ setState 更新 fragmentIdsThisRun + latestRunTime
    └─→ 关键分支判断
           ├─ 同 appHash 且同 pageScriptHash（正常重跑）
           │     └─→ setState: clearTransientNodes() + 更新 scriptRunId
           └─ appHash 或 pageScriptHash 变化（跨页面/跨应用）
                 └─→ clearAppState(): 完全清空渲染树 + 清理 Widget 状态
```

**关键细节**：
- `script_run_id` 是每次运行唯一的 UUID，后续所有 Delta 消息都继承这个 ID
- 每次收到 `new_session`，前端会置 `hasReceivedNewSession = true`（防止旧运行的 finished 消息误触发缓存清理）

### 8.3 `session_status_changed` 消息详解

**发送时机** [app_session.py#L762-L775](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/app_session.py#L762-L775)：

```python
# 在 _on_scriptrunner_event 末尾，状态切换时发送
app_was_running = prev_state == AppSessionState.APP_IS_RUNNING
app_is_running = self._state == AppSessionState.APP_IS_RUNNING
if app_is_running != app_was_running:
    self._enqueue_forward_msg(self._create_session_status_changed_message())

# 另外，用户修改 Run-On-Save 设置时也发送
def _handle_set_run_on_save_request(self, new_value):
    self._run_on_save = new_value
    self._enqueue_forward_msg(self._create_session_status_changed_message())
```

发送场景：
1. `APP_NOT_RUNNING → APP_IS_RUNNING`（脚本开始运行）
2. `APP_IS_RUNNING → APP_NOT_RUNNING`（脚本结束运行）
3. 用户通过界面修改 `run_on_save` 设置

**前端接收处理** [App.tsx#L1219-L1265](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L1219-L1265)：

```
handleSessionStatusChanged(statusChangeProto)
    ├─→ scriptIsRunning=true
    │      ├─ 当前非 STOP_REQUESTED → scriptRunState = RUNNING
    │      └─ 若当前显示编译错误对话框 → 关闭它
    ├─→ scriptIsRunning=false
    │      └─ 当前非 RERUN_REQUESTED 且非 COMPILATION_ERROR
    │            → scriptRunState = NOT_RUNNING
    ├─→ 更新 userSettings.runOnSave
    └─→ scriptIsRunning=true 时重置 scriptChangedOnDisk=false
```

**与 `new_session` 的关系**：`new_session.initialize.session_status` 中也会携带同样的运行状态，并在 `handleInitialization()` 中调用 `handleSessionStatusChanged()`。同时，独立的 `session_status_changed` 消息也会在 `SCRIPT_STARTED` 事件处理末尾发送（排在 `new_session` 之后）。所以**首次连接时，`new_session` 通过 `handleInitialization` 先触发状态更新，独立的 `session_status_changed` 消息随后到达并执行幂等操作**。

### 8.4 `script_finished` 消息详解

**发送时机与状态枚举** [app_session.py#L688-L736](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/app_session.py#L688-L736)：

| ScriptRunnerEvent | ScriptFinishedStatus | 前端是否清理节点 |
|-------------------|---------------------|----------------|
| `SCRIPT_STOPPED_WITH_SUCCESS` | `FINISHED_SUCCESSFULLY` | ✅ 是 |
| `FRAGMENT_STOPPED_WITH_SUCCESS` | `FINISHED_FRAGMENT_RUN_SUCCESSFULLY` | ✅ 是（精细清理） |
| `SCRIPT_STOPPED_WITH_COMPILE_ERROR` | `FINISHED_WITH_COMPILE_ERROR` | ❌ 否 |
| `SCRIPT_STOPPED_FOR_RERUN` | `FINISHED_EARLY_FOR_RERUN` | ❌ 否（防闪烁） |

**前端接收处理** [App.tsx#L1593-L1657](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L1593-L1657)：

```
handleScriptFinished(status)
    ├─→ 对 SUCCESSFULLY / EARLY_FOR_RERUN / FRAGMENT_SUCCESS
    │     └─→ 微任务队列（下一轮事件循环）执行所有 scriptFinishedHandlers
    │
    ├─→ 仅对 SUCCESSFULLY / FRAGMENT_SUCCESS
    │     └─→ setState: clearStaleNodes()
    │           callback: removeInactiveWidgetState()
    │
    └─→ 仅对非 EARLY_FOR_RERUN 且 hasReceivedNewSession=true
          └─→ incrementMessageCacheRunCount()
                （按 maxCachedMessageAge 过期旧消息缓存）
```

**关键边界条件**：
- `FINISHED_WITH_COMPILE_ERROR` 完全不执行任何 handler，也不清理
- `FINISHED_EARLY_FOR_RERUN` 只触发 handler 通知，但**不清理节点、不清理消息缓存**
- `hasReceivedNewSession` 守卫防止旧运行的 finished 消息（因网络延迟先于新运行的 new_session 到达）错误地清理当前新运行的缓存

---

## 9. 四种典型场景的边界条件分析

### 9.1 场景一：首次初始化

**后端流程**：
1. 前端建立 WebSocket，发送 `Hello` BackMsg
2. Runtime 创建 AppSession，创建 ScriptRunner
3. ScriptRunner 发射 `SCRIPT_STARTED` 事件
4. 发送 `new_session` 消息（`scriptIsRunning=true`，携带完整配置）
5. 进入用户脚本，执行 ST.status() 等 → 发送 Delta 消息
6. 脚本完成 → `FINISHED_SUCCESSFULLY` → 发送 `script_finished` + `session_status_changed`（`scriptIsRunning=false`）

**前端边界条件**：
- `sessionInfo.isSet == false` 时进入 `handleInitialization()`，设置 sessionInfo（包含 streamlitVersion、environmentInfo、sessionId、userInfo 等）
- `appHash` 初始为空，`pageScriptHash` 初始为默认值 → 走 `clearAppState()` 分支（而非轻量的 clearTransientNodes）
- 首次连接不检测 Streamlit 版本变化（未设置 previous version 时跳过）

**关键代码**：`INITIAL_SCRIPT_RUN_ID = "<null>"` 是初始占位值，直到首次收到 `new_session` 才被替换。

### 9.2 场景二：断连后重连

**连接状态机** [ConnectionState.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/connection/src/ConnectionState.ts)：

```
INITIAL → CONNECTING → PINGING_SERVER → CONNECTED
                        ↓
                    DISCONNECTED_FOREVER（超过最大重试次数）
```

**前端重连处理** [App.tsx#L862-L941](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L862-L941)：

```
handleConnectionStateChanged(CONNECTED)
    ├─→ 重置 connectionErrorDismissed=false
    ├─→ 判断是否需要请求重跑（满足任一条件就重跑）：
    │     1. sessionInfo.last 不存在（首次建立连接）
    │     2. lastRunWasInterrupted：scriptRunState===RUNNING（断连时脚本在跑）
    │     3. wasRerunRequested：scriptRunState===RERUN_REQUESTED
    │     4. fragmentIdsThisRun.length > 0（上次跑了 fragment）
    │     5. autoReruns.length > 0（有定时自动重跑）
    │
    ├─→ 需要重跑：
    │     └─→ widgetMgr.sendUpdateWidgetsMessage(undefined)
    │           （携带所有 Widget 当前值，触发脚本带状态重跑）
    │
    ├─→ 不需要重跑但显示了连接错误对话框：
    │     └─→ setState({ dialog: null }) 关闭对话框
    │
    ├─→ 向 Host 发送 WEBSOCKET_CONNECTED 事件
    └─→ flushSync 同步更新 connectionState（确保每个连接状态变更都被 render 观察到）
```

**状态容器（st.status）的重连边界**：
- 断连期间，已渲染的 DOM 保持不变（页面内状态容器视觉上仍存在）
- StatusWidget 切换为显示连接状态（Pinging / Disconnected），而非脚本运行状态
- 重连成功后，若脚本被重新执行，会发送新的 `new_session` → Delta 流 → `script_finished`，按正常流程覆盖旧节点
- 若断连时脚本正在运行 `st.status` 的长任务，重连后会重新从头执行，旧状态容器会被新 Delta 覆盖

### 9.3 场景三：正常运行（无中断）

**消息顺序的严格保证** [app_session.py#L688-L766](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/app_session.py#L688-L766)：
```
[1] new_session (script_run_id=A, scriptIsRunning=true)
[2] delta (type=add_block, 创建 st.status 容器)
[3] delta (type=new_element, status.write 内容 1)
[4] delta (type=add_block, status.update(label="..."))
[5] delta (type=new_element, status.write 内容 2)
[6] script_finished (FINISHED_SUCCESSFULLY)                ← 先发 script_finished
[7] session_status_changed (scriptIsRunning=false)          ← 后发 session_status_changed
```

**后端发送顺序详解**：

在 `_on_scriptrunner_event` 函数中处理 `SCRIPT_STOPPED_WITH_SUCCESS` 事件时：

```python
prev_state = self._state  # = APP_IS_RUNNING

# 处理停止事件
self._state = AppSessionState.APP_NOT_RUNNING               # 1. 状态变更
self._enqueue_forward_msg(self._create_script_finished_message(status))  # 2. 先发 script_finished

# ... 其他处理（更新 watched modules 等）

# 函数末尾检测状态变化
app_was_running = prev_state == AppSessionState.APP_IS_RUNNING  # = True
app_is_running = self._state == AppSessionState.APP_IS_RUNNING  # = False
if app_is_running != app_was_running:
    self._enqueue_forward_msg(self._create_session_status_changed_message())  # 3. 后发 session_status_changed
```

关键：`session_status_changed` 的发送逻辑在**函数末尾**，在所有事件类型处理完成之后统一执行。所以 `script_finished` 先入队，`session_status_changed` 后入队。

**边界条件**：
- 前端按接收顺序处理。收到消息 [6] 时先执行 `clearStaleNodes()` 清理旧节点，但**此时 scriptRunState 还是 RUNNING**（要到消息 [7] 才更新为 NOT_RUNNING）
- 在 [6] 和 [7] 之间，用户看到：页面已清理旧节点，顶部状态栏仍显示"Running"
- 这是一个短暂的中间窗口，通常在毫秒级别，用户几乎感知不到

**状态容器的正常完成边界**：
- 若 `with st.status()` 块中出现未捕获异常，`__exit__` 将状态置为 `error`，此时 `script_finished` 可能仍是 `FINISHED_SUCCESSFULLY`（如果异常在 Streamlit 捕获范围内，脚本整体仍算成功）
- 顶部状态栏变为 NOT_RUNNING，页面内 status 容器显示 error 图标，两者反映不同层面的状态

### 9.4 场景四：脚本重跑清理

**后端重跑触发路径**：
```
用户点击 Rerun
    → 前端发送 BackMsg(rerun_script) + scriptRunState 乐观设为 RERUN_REQUESTED
    → AppSession._handle_rerun_script_request()
    → ScriptRunner.request_stop(ScriptStopException)
    → ScriptRunner 发送 SCRIPT_STOPPED_FOR_RERUN
    → 发送 script_finished(FINISHED_EARLY_FOR_RERUN)
    → 立即创建新 ScriptRunner
    → 新 ScriptRunner 发送 SCRIPT_STARTED
    → 发送 new_session(script_run_id=NEW)
```

**前端重跑清理边界** [App.tsx#L2000-L2010](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L2000-L2010)：

请求重跑时前端先重置 `hasReceivedNewSession = false`：

```
rerunScript()
    → this.hasReceivedNewSession = false  ← 关键：清空守卫
    → sendBackMsg(BackMsg(rerun_script, ...))
```

这是为了防止**旧运行的 script_finished（FINISHED_EARLY_FOR_RERUN 或 SUCCESSFULLY）** 在 `new_session` 到达之前误触发缓存清理。

**状态容器在重跑中的生命周期边界**：

| 时间点 | 顶部 StatusWidget | 页面内 status 容器 | 触发事件 |
|--------|-----------------|-------------------|---------|
| T0（重跑前） | NOT_RUNNING | BlockNode(run-1, state=complete) | — |
| T1（用户点击 Rerun） | RERUN_REQUESTED（乐观） | 不变（仍是 run-1） | `this.setState({scriptRunState: RERUN_REQUESTED})` |
| T2（旧脚本中断） | RERUN_REQUESTED | 不变（EARLY_FOR_RERUN 跳过清理） | `handleScriptFinished(EARLY_FOR_RERUN)` → 无清理 |
| T3（收到 new_session run-2） | RUNNING | 不变（仍显示旧 run-1 内容） | `handleNewSession()`：设 scriptRunId=run-2，清除瞬态 |
| T4（新脚本执行 st.status） | RUNNING | 被新 Delta 替换为 run-2 | `handleDeltaMsg()`：SetNodeByDeltaPathVisitor |
| T5（新脚本完成） | NOT_RUNNING | run-2 节点保留 | `handleScriptFinished(SUCCESSFULLY)`：清理非 run-2 节点 |

**核心边界：T2 → T3 之间页面不闪烁**
- 旧脚本以 `FINISHED_EARLY_FOR_RERUN` 结束 → 前端**跳过 clearStaleNodes**
- `new_session(run-2)` 到达时只执行 `clearTransientNodes()`（清除 spinner 等瞬态元素），不清除 BlockNode
- 旧的 status 容器（run-1）在 T2 到 T4 之间仍然显示，直到新脚本在同 delta_path 上创建新的 status 容器（run-2）把它覆盖
- 如果新脚本不再创建该 status 容器（代码改了），则该旧节点会一直保留到 T5，被 `clearStaleNodes(run-2)` 作为过期节点清除

### 9.5 三种场景的顶部状态栏更新来源对比

顶部状态栏（StatusWidget）的核心状态是 `scriptRunState`（RUNNING / NOT_RUNNING / RERUN_REQUESTED 等）和 `connectionState`（CONNECTED / PINGING_SERVER 等）。不同场景下，驱动 `scriptRunState` 更新的消息来源**完全不同**。

**核心判断条件** [App.tsx#L1365-L1368](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L1365-L1368)：
```typescript
if (!this.sessionInfo.isSet || !this.sessionInfo.current.isConnected) {
  this.handleInitialization(newSessionProto)
}
```

只有当 `sessionInfo.isSet == false`（首次初始化）或 `isConnected == false`（重连后）时，才会调用 `handleInitialization()`，进而通过 `new_session` 中的 `initialize.session_status` 间接更新 `scriptRunState`。普通运行场景不满足此条件。

---

#### 场景 1：首次初始化

**阶段一：WebSocket 连接阶段**
```
connectionState 变化:
INITIAL → CONNECTING → PINGING_SERVER → CONNECTED
       ↓            ↓                 ↓
handleConnectionStateChanged() 每次变更都被 flushSync 强制同步
       ↓
StatusWidget 显示连接状态信息（"Connecting..."、"Pinging Server..."）
```
此时 `scriptRunState` 仍为初始值，StatusWidget 由 `connectionState` 驱动。

**阶段二：收到 new_session 消息**
```
handleNewSession(newSession)
    ├─→ !sessionInfo.isSet → true（首次）
    │
    └─→ handleInitialization(newSession)
          ├─→ sessionInfo.setCurrent(...)  ← 标记为已初始化
          └─→ handleSessionStatusChanged(initialize.sessionStatus)
                └─→ scriptRunState = RUNNING 或 NOT_RUNNING
```

**来源汇总（首次初始化）：**
| 阶段 | 驱动消息/事件 | 效果 |
|------|-------------|------|
| 连接中 | `connectionState` 变更 | 显示连接状态 |
| 脚本开始 | `new_session.initialize.session_status` | 间接触发 `handleSessionStatusChanged()` → `RUNNING` |
| 脚本结束 | 独立的 `session_status_changed(scriptIsRunning=false)` | `NOT_RUNNING` |

注意：首次初始化时，**也会**收到独立的 `session_status_changed(scriptIsRunning=true)` 消息，但它排在 `new_session` 之后到达。由于前端在处理 `new_session` 时已通过 `handleInitialization()` 将 `scriptRunState` 设为 `RUNNING`，后续收到同样状态的 `session_status_changed` 是幂等操作，没有实际效果。因此脚本开始时的 RUNNING 状态**第一次**是由 `new_session.initialize.session_status` 设置的。

---

#### 场景 2：断连后重连

**阶段一：断连检测**
```
connectionState → PINGING_SERVER
    ↓
handleConnectionStateChanged(PINGING_SERVER)
    ├─→ sessionInfo.disconnect()  ← isConnected 设为 false
    └─→ flushSync 同步更新 connectionState
          ↓
StatusWidget 显示 "Pinging Server..."（由 connectionState 驱动）
```

**阶段二：重连成功**
```
connectionState → CONNECTED
    ↓
handleConnectionStateChanged(CONNECTED)
    ├─→ 判断是否需要重跑脚本（5 种条件）
    │     └─→ 需要重跑：sendUpdateWidgetsMessage() 触发后端脚本运行
    │
    └─→ flushSync 同步更新 connectionState
          ↓
StatusWidget 不再显示连接状态
```

**阶段三：后端脚本开始，发送 new_session**
```
handleNewSession(newSession)
    ├─→ sessionInfo.current.isConnected → false（断连时被置为 false）
    │
    └─→ handleInitialization(newSession)
          ├─→ sessionInfo.setCurrent(...)  ← isConnected 设为 true
          └─→ handleSessionStatusChanged(initialize.sessionStatus)
                └─→ scriptRunState = RUNNING
```

**来源汇总（断连后重连）：**
| 阶段 | 驱动消息/事件 | 效果 |
|------|-------------|------|
| 断连中 | `connectionState` 变更 | 显示连接状态 |
| 重连成功 | `connectionState → CONNECTED` | 清除连接状态显示 |
| 脚本开始 | `new_session.initialize.session_status` | 间接触发 `handleSessionStatusChanged()` → `RUNNING` |
| 脚本结束 | 独立的 `session_status_changed(scriptIsRunning=false)` | `NOT_RUNNING` |

重连场景与首次初始化非常类似，区别在于：
- 断连时 `sessionInfo.isSet` 仍为 true，但 `isConnected` 被置为 false
- 重连后走 `handleInitialization()` 是因为 `isConnected` 为 false，而非 `isSet` 为 false
- 重连后可能触发脚本重跑（带 Widget 当前值），也可能不触发

---

#### 场景 3：普通运行（非首次、非重连）

普通运行是最常见的场景：用户点击按钮、勾选复选框、点击 Rerun 按钮等。

**脚本开始时**：
```
后端 SCRIPT_STARTED 事件:
1. self._state = APP_IS_RUNNING
2. 发送 new_session 消息
3. 函数末尾检测到状态变化 → 发送独立的 session_status_changed(scriptIsRunning=true)

前端处理:
1. handleNewSession()
   ├─→ sessionInfo.isSet = true 且 isConnected = true
   └─→ 不走 handleInitialization()，scriptRunState 保持不变（或之前的 RERUN_REQUESTED）

2. handleSessionStatusChanged(scriptIsRunning=true)
   └─→ scriptRunState = RUNNING
```

**脚本结束时**：
```
后端 SCRIPT_STOPPED_WITH_SUCCESS 事件:
1. self._state = APP_NOT_RUNNING
2. 发送 script_finished(FINISHED_SUCCESSFULLY)
3. 函数末尾检测到状态变化 → 发送独立的 session_status_changed(scriptIsRunning=false)

前端处理:
1. handleScriptFinished(SUCCESSFULLY)
   └─→ clearStaleNodes() + removeInactiveWidgetState()

2. handleSessionStatusChanged(scriptIsRunning=false)
   └─→ scriptRunState = NOT_RUNNING
```

**来源汇总（普通运行）：**
| 阶段 | 驱动消息/事件 | 效果 |
|------|-------------|------|
| 脚本开始 | 独立的 `session_status_changed(scriptIsRunning=true)` | 直接调用 `handleSessionStatusChanged()` → `RUNNING` |
| 脚本结束 | `script_finished(FINISHED_SUCCESSFULLY)` | 清理节点，但不直接改变 `scriptRunState` |
| 脚本结束后 | 独立的 `session_status_changed(scriptIsRunning=false)` | 直接调用 `handleSessionStatusChanged()` → `NOT_RUNNING` |

关键区别：**普通运行时，`new_session` 不会改变 `scriptRunState`**，因为不满足 `!sessionInfo.isSet || !isConnected` 的条件。StatusWidget 完全由独立的 `session_status_changed` 消息驱动。

---

**三种场景对比总表：**

| 场景 | 脚本开始时 scriptRunState 更新来源 | 脚本结束时 scriptRunState 更新来源 | 是否走 handleInitialization() |
|------|---------------------------------|---------------------------------|-----------------------------|
| 首次初始化 | `new_session.initialize.session_status` | 独立的 `session_status_changed(=false)` | ✅ 是（`!isSet`） |
| 断连后重连 | `new_session.initialize.session_status` | 独立的 `session_status_changed(=false)` | ✅ 是（`!isConnected`） |
| 普通运行 | 独立的 `session_status_changed(=true)` | 独立的 `session_status_changed(=false)` | ❌ 否 |

### 9.6 跨场景的共享边界条件

**条件 1：hasReceivedNewSession 守卫**
- 初始值：`false`
- 收到 `new_session`：`true`
- 调用 `rerunScript()` 或 `fragmentRunRequest()`：`false`
- 作用：确保缓存清理只在"当前运行的结束消息"到达时执行，而不是"上一次运行的迟到消息"

**条件 2：scriptRunId 的标签作用**
- 初始值：`INITIAL_SCRIPT_RUN_ID = "<null>"`
- `new_session` 到达后被替换为新的 UUID
- 所有 Delta 消息生成的节点都打上当前 scriptRunId 标签
- `clearStaleNodes()` 仅删除 scriptRunId 不匹配的节点

**条件 3：connectionState 与 scriptRunState 的联合判定**
- StatusWidget 显示 Running Man 的条件：`connectionState === CONNECTED && scriptRunState in [RUNNING, RERUN_REQUESTED]`
- 若连接断开（PINGING_SERVER / DISCONNECTED_FOREVER），无论脚本是否在运行，StatusWidget 都显示连接状态，而不显示脚本运行状态
- Widget 输入被禁用条件：`connectionState !== CONNECTED`

**条件 4：FINISHED_* 状态清理策略表**

| status 值 | 通知 handler | clearStaleNodes | removeInactiveWidget | incrementMessageCache |
|-----------|------------|-----------------|---------------------|----------------------|
| `FINISHED_SUCCESSFULLY` | ✅ | ✅ | ✅ | ✅（需 hasReceivedNewSession） |
| `FINISHED_FRAGMENT_RUN_SUCCESSFULLY` | ✅ | ✅（精细） | ✅ | ✅（需 hasReceivedNewSession） |
| `FINISHED_EARLY_FOR_RERUN` | ✅ | ❌ | ❌ | ❌ |
| `FINISHED_WITH_COMPILE_ERROR` | ❌ | ❌ | ❌ | ❌ |

---

## 10. 脚本重新运行时旧节点不立即清除的原因

### 10.1 核心设计原则：防止闪烁

Streamlit 的页面渲染遵循一个重要原则：**旧元素一直显示，直到新元素覆盖它**。这就是为什么脚本重跑时旧的 `st.status` 容器不会立即消失。

### 10.2 具体实现机制

**机制一：FINISHED_EARLY_FOR_RERUN 跳过清理**

后端发送中断型结束消息时 [app_session.py#L730-L736](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/app_session.py#L730-L736)：

```python
elif event == ScriptRunnerEvent.SCRIPT_STOPPED_FOR_RERUN:
    self._state = AppSessionState.APP_NOT_RUNNING
    self._enqueue_forward_msg(
        self._create_script_finished_message(
            ForwardMsg.FINISHED_EARLY_FOR_RERUN  # ← 关键：不是 SUCCESSFULLY
        )
    )
```

前端收到此状态时 [App.tsx#L1593-L1633](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L1593-L1633)：

```typescript
handleScriptFinished(status): void {
    if (status === FINISHED_SUCCESSFULLY || 
        status === FINISHED_FRAGMENT_RUN_SUCCESSFULLY) {
        // 只有成功完成才清理！
        this.setState({
            elements: elements.clearStaleNodes(scriptRunId, fragmentIdsThisRun),
        })
    }
    // FINISHED_EARLY_FOR_RERUN 跳过 clearStaleNodes
    // 旧元素保留在页面上
}
```

**机制二：后端队列保留生命周期消息**

新脚本启动时 `_clear_queue()` 会保留旧队列中的生命周期消息 [app_session.py#L565-L567](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/app_session.py#L565-L567)：

```python
def _clear_queue(self, fragment_ids_this_run=None):
    self._browser_queue.clear(
        retain_lifecycle_msgs=True,  # ← 保留 new_session, script_finished 等
        fragment_ids_this_run=fragment_ids_this_run
    )
```

并且 `_update_script_finished_message()` 会将旧的 `FINISHED_SUCCESSFULLY` 改为 `FINISHED_EARLY_FOR_RERUN` [forward_msg_queue.py#L236-L264](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/forward_msg_queue.py#L236-L264)，防止旧的成功消息触发前端清理。

**机制三：节点清理基于 scriptRunId**

每个渲染树节点都有 `scriptRunId` 属性，`ClearStaleNodeVisitor` 据此判断节点是否过期 [ClearStaleNodeVisitor.ts#L61-L69](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/render-tree/visitors/ClearStaleNodeVisitor.ts#L61-L69)：

```typescript
visitBlockNode(node: BlockNode): AppNode | undefined {
    if (!this.isFragmentRun) {
        // 非 fragment 模式：删除所有 scriptRunId 不匹配的节点
        if (node.scriptRunId !== this.currentScriptRunId) {
            return undefined  // ← 标记为过期，删除
        }
    }
    // ...
}
```

在 `clearStaleNodes()` 被调用之前，旧 scriptRunId 的节点虽然不匹配，但仍然保留在渲染树中。

### 10.3 时序图解：重跑时的元素生命周期

```
T0: 初始状态
    AppRoot.main.children = [
        BlockNode(scriptRunId="run-1", <-- st.status("旧任务完成")
                  children=[ElementNode("完成内容", scriptRunId="run-1")])
    ]
    StatusWidget: NOT_RUNNING

T1: 用户点击按钮，触发重跑
    → scriptRunState = RERUN_REQUESTED (前端乐观更新)
    → StatusWidget 显示 "Rerun requested"
    → 页面元素暂时无变化！旧的 status 容器仍然可见

T2: 后端中断旧脚本，发送 FINISHED_EARLY_FOR_RERUN
    → handleScriptFinished(EARLY_FOR_RERUN)
    → 跳过 clearStaleNodes()
    → 旧节点(scriptRunId="run-1")仍然保留！

T3: 后端启动新脚本，发送 new_session(scriptRunId="run-2", scriptIsRunning=true)
    → handleNewSession()
    → 更新 scriptRunId = "run-2"
    → clearTransientNodes() 清除瞬态元素
    → handleSessionStatusChanged(scriptIsRunning=true)
    → scriptRunState = RUNNING
    → StatusWidget 显示 Running Man
    → 旧 status 容器仍然可见（尚未被覆盖）

T4: 新脚本从头开始执行，发送 Delta 消息
    → Delta 1: add_block(scriptRunId="run-2") 创建新的 status 容器
      → SetNodeByDeltaPathVisitor 在 deltaPath [0] 处替换节点
      → 旧 BlockNode("run-1") 被新 BlockNode("run-2") 替换
      → 用户看到新的 "Running" 状态容器

T5: 新脚本发送更多 Delta 消息
    → 按顺序覆盖旧路径上的节点

T6: 新脚本完成，发送 FINISHED_SUCCESSFULLY
    → clearStaleNodes("run-2")
    → 删除所有 scriptRunId != "run-2" 的节点
    → 如果旧脚本有节点没被覆盖（如新脚本更短），此时被清理
    → removeInactiveWidgetState() 清理 widget
```

### 10.4 为什么这是好的设计？

| 场景 | 如果立即清除 | 实际（延迟清除） |
|------|------------|----------------|
| 快速重跑 | 页面空白 → 闪烁 | 旧内容保持，新内容渐进替换 |
| 新旧内容相同 | 删除再重建 → 闪烁 | 只有变化的部分被更新 |
| 重跑失败 | 内容丢失 | 旧内容仍然可用 |
| Fragment 运行 | 无关内容被清掉 | 只更新相关 fragment 区域 |

---

## 11. 脚本成功结束时的完整清理流程

### 11.1 清理时机

清理发生在 `handleScriptFinished()` 收到 `FINISHED_SUCCESSFULLY` 状态时：

```typescript
// App.tsx handleScriptFinished
if (status === FINISHED_SUCCESSFULLY || 
    status === FINISHED_FRAGMENT_RUN_SUCCESSFULLY) {
    this.setState(
        ({ scriptRunId, fragmentIdsThisRun, elements }) => ({
            elements: elements.clearStaleNodes(
                scriptRunId,
                fragmentIdsThisRun
            ),
        }),
        () => {
            this.removeInactiveWidgetState()  // setState 回调中执行
        }
    )
}
```

### 11.2 清理步骤详解

#### 步骤 1：clearStaleNodes() - 清除过期渲染节点

[AppRoot.ts#L340-L369](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/render-tree/AppRoot.ts#L340-L369)

```typescript
public clearStaleNodes(currentScriptRunId, fragmentIdsThisRun?): AppRoot {
    const visitor = new ClearStaleNodeVisitor(
        currentScriptRunId,
        fragmentIdsThisRun
    )
    const newChildren = this.root.children.map(node =>
        this.ensureBlockNode(node.accept(visitor))
    )
    return new AppRoot(..., new BlockNode(..., newChildren, ...))
}
```

**ClearStaleNodeVisitor 的清理规则：**

| 运行模式 | 节点类型 | 清理条件 |
|----------|---------|----------|
| **全量运行** | BlockNode | `node.scriptRunId !== currentScriptRunId` → 删除 |
| **全量运行** | ElementNode | `node.scriptRunId !== currentScriptRunId` → 删除 |
| **Fragment 运行** | BlockNode (属于 fragment) | 父 fragment 被修改但自己不是当前 run → 删除 |
| **Fragment 运行** | BlockNode (不属于任何 fragment) | 保留，不会被清除 |
| **Fragment 运行** | ElementNode (不属于 fragment) | 保留，不会被清除 |

Fragment 模式的精细清理示例：
```
AppRoot
├─ BlockNode(fragmentId="A", scriptRunId=NEW) ← 当前 fragment, 保留
│  └─ ElementNode(scriptRunId=OLD)             ← 父 fragment 被修改，删除
├─ BlockNode(fragmentId="B", scriptRunId=OLD) ← 不在本次 fragment，保留（可能以后用）
└─ ElementNode(no fragmentId, scriptRunId=OLD) ← 不属于 fragment，保留
```

#### 步骤 2：removeInactiveWidgetState() - 清理 Widget 状态

[App.tsx#L1696-L1705](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L1696-L1705)

```typescript
private removeInactiveWidgetState(): void {
    // 1. 遍历渲染树，收集所有活跃的元素 ID 和块 ID
    const { elements, blockIds } = this.state.elements.getActiveIds()
    const activeIds = new Set([
        ...elements.map(e => getElementId(e)).filter(notUndefined),
        ...blockIds,
    ])
    // 2. 删除 WidgetStateManager 中不在 activeIds 中的条目
    this.widgetMgr.removeInactive(activeIds)
}
```

这确保了：
- 不再渲染的 Widget（如用户删除了某个 `st.slider`）的用户输入值被清除
- 保留仍然存在的 Widget 的用户输入

#### 步骤 3：incrementMessageCacheRunCount() - 清理消息缓存

[App.tsx#L1635-L1656](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx#L1635-L1656)

```typescript
if (status !== FINISHED_EARLY_FOR_RERUN && this.hasReceivedNewSession) {
    this.connectionManager.incrementMessageCacheRunCount(
        maxCachedMessageAge,
        fragmentIdsThisRun
    )
}
```

作用：移除超过缓存寿命的 ForwardMsg 引用，释放浏览器内存。

### 11.3 完整清理时序

```
FINISHED_SUCCESSFULLY 消息到达
    │
    ├─→ 微任务队列（Promise.resolve().then）
    │      └─→ 执行所有 scriptFinishedHandlers 回调
    │           (组件订阅的结束事件)
    │
    └─→ setState() 触发 React 更新
           │
           ├─→ clearStaleNodes() 同步执行
           │      ├─→ ClearStaleNodeVisitor 遍历渲染树
           │      │   ├─→ visitBlockNode() 检查每个块
           │      │   ├─→ visitElementNode() 检查每个元素
           │      │   └─→ visitTransientNode() 检查瞬态节点
           │      └─→ 构建新的 AppRoot（不包含过期节点）
           │
           ├─→ React 提交新的 AppRoot 到 DOM
           │      └─→ 用户看到页面更新（旧节点消失）
           │
           └─→ setState callback 中
                  └─→ removeInactiveWidgetState()
                         ├─→ ElementsSetVisitor 收集活跃 IDs
                         └─→ WidgetStateManager.removeInactive()
    │
    └─→ incrementMessageCacheRunCount() （独立调用，非 React 内）
```

---

## 12. 异步反馈原理总结

### 12.1 核心机制

Streamlit 的状态提示组件实现异步反馈依赖以下几个关键机制的协同工作：

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户脚本线程                              │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐              │
│  │ st.status │ →  │  .update()│ →  │  .write() │  同步执行    │
│  └─────┬─────┘    └─────┬─────┘    └─────┬─────┘              │
│        │                │                │                    │
│        └────────────────┴────────────────┘                    │
│                         ↓                                       │
│              enqueue_message() 入队                             │
└─────────────────────────┬───────────────────────────────────────┘
                          │  线程安全事件转发
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                     主线程事件循环 (Runtime)                     │
│                                                                 │
│  ForwardMsgQueue  ── 定期刷新 ──→  WebSocket 发送               │
│  (消息缓冲)          (消息合并)                                   │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼  WebSocket 消息
┌─────────────────────────────────────────────────────────────────┐
│                       前端 React 应用                            │
│                                                                 │
│  handleMessage() → applyDelta() → setState() → 重渲染           │
│  (消息分发)       (增量更新)      (状态更新)                     │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 各层职责

| 层级 | 组件 | 职责 |
|------|------|------|
| API 层 | `StatusContainer` | 提供用户友好的 API（with 语句、update 方法） |
| 消息生成层 | `DeltaGenerator` / `enqueue_message` | 将 UI 变更转化为 Delta 消息 |
| 消息队列层 | `ForwardMsgQueue` | 缓冲消息、合并优化 |
| 会话层 | `AppSession` | 管理会话状态、转发脚本事件 |
| 运行时层 | `Runtime` | 主循环调度、WebSocket 通信 |
| 前端连接层 | `ConnectionManager` | WebSocket 连接管理 |
| 前端渲染层 | `AppRoot` / `Block` / `Expander` | 渲染树管理、增量更新 |

### 12.3 为什么不需要刷新整个页面

状态提示组件的更新是**增量式**的，这是 Streamlit 高效性的重要体现：

1. **后端层面**：只发送变化了的 Delta 消息，而非整个页面状态
2. **前端层面**：只更新对应 delta_path 的节点，React 只重渲染受影响的组件
3. **状态保留**：BlockNode 替换时会继承同类型块的子节点（widget state、React state）

这种设计使得长任务中的频繁状态更新也能保持流畅的用户体验。

---

## 13. 相关文件索引

### 后端核心文件
- [mutable_status_container.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/elements/lib/mutable_status_container.py) - StatusContainer 实现
- [forward_msg_queue.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/forward_msg_queue.py) - 消息队列
- [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/app_session.py) - 会话管理
- [runtime.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/runtime.py) - 运行时主循环
- [script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py) - 脚本运行器
- [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py) - 脚本运行上下文

### 前端核心文件
- [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/App.tsx) - 主应用组件、消息处理、清理调度
- [AppRoot.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/render-tree/AppRoot.ts) - 渲染树根节点
- [BlockNode.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/render-tree/BlockNode.ts) - 块节点（含 scriptRunId）
- [Block.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/components/core/Block/Block.tsx) - 块渲染组件
- [Expander.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/components/elements/Expander/Expander.tsx) - 可展开容器
- [StatusWidget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/app/src/components/StatusWidget/StatusWidget.tsx) - 顶部状态栏
- [ScriptRunState.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/ScriptRunState.ts) - 脚本运行状态枚举
- [ScriptRunContext.tsx](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/components/core/ScriptRunContext.tsx) - 脚本运行状态上下文
- [ConnectionState.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/connection/src/ConnectionState.ts) - 连接状态枚举（INITIAL/CONNECTED/DISCONNECTED_FOREVER 等）
- [ClearStaleNodeVisitor.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/render-tree/visitors/ClearStaleNodeVisitor.ts) - 过期节点清理访问者
- [ClearTransientNodesVisitor.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/render-tree/visitors/ClearTransientNodesVisitor.ts) - 瞬态节点清理
- [SetNodeByDeltaPathVisitor.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/render-tree/visitors/SetNodeByDeltaPathVisitor.ts) - 按路径设置节点
- [FilterMainScriptElementsVisitor.ts](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/frontend/lib/src/render-tree/visitors/FilterMainScriptElementsVisitor.ts) - 主脚本元素过滤

### 测试/示例文件
- [st_status.py](file:///d:/fz/0601/solo-dogfeeding/code/237-streamlit/e2e_playwright/st_status.py) - 状态组件测试页面
