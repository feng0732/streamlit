# 主动重跑（Rerun）与停止（Stop）语义梳理

本文档按代码执行顺序，梳理 Streamlit 中脚本控制信号（ScriptRequests）、异常中断（ScriptControlException 体系）和会话状态（AppSessionState / ScriptRunnerEvent）三者之间的关系。

---

## 一、核心文件索引

| 层级 | 文件 | 职责 |
|------|------|------|
| 用户 API 层 | [execution_control.py](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/commands/execution_control.py) | `st.stop()` / `st.rerun()` / `st.switch_page()` 入口 |
| 异常定义层 | [exceptions.py](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/exceptions.py) | `StopException` / `RerunException` 类定义 |
| 控制信号层 | [script_requests.py](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py) | `ScriptRequests` 状态机（CONTINUE/STOP/RERUN） |
| 执行引擎层 | [script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py) | `ScriptRunner` 线程主循环、yield 点检查 |
| 异常捕获层 | [exec_code.py](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner/exec_code.py) | `exec_func_with_error_handling` 异常分类处理 |
| 会话管理层 | [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py) | `AppSession` 会话状态、ScriptRunner 生命周期管理 |
| 上下文层 | [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py) | `ScriptRunContext` 运行时上下文 |

---

## 二、异常中断体系

### 2.1 异常类层次（[exceptions.py#L19-L44](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/exceptions.py#L19-L44)）

```
BaseException
  └── ScriptControlException  （继承 BaseException 而非 Exception，
        │                       避免被用户 try/except Exception 吞掉）
        ├── StopException     ：静默停止脚本执行
        └── RerunException    ：携带 RerunData，停止后立即重跑
```

### 2.2 关键设计决策
- **继承 `BaseException`**：保证 `st.rerun()` / `st.stop()` 在 `try/except Exception:` 块中仍能生效。
- **RerunException 携带 `rerun_data`**：保存重跑所需的 widget 状态、query_string、fragment 队列等。

---

## 三、用户 API 入口层（第一层）

### 3.1 `st.stop()` 调用链（[execution_control.py#L44-L68](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/commands/execution_control.py#L44-L68)）

```
用户代码调用 st.stop()
    │
    ▼
获取 ScriptRunContext ctx
    │
    ▼
ctx.script_requests.request_stop()     ──→ 设置 ScriptRequests._state = STOP
    │
    ▼
st.empty()  ──→ 强制触发 yield point（进入第五节的检查流程）
```

### 3.2 `st.rerun()` 调用链（[execution_control.py#L139-L192](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/commands/execution_control.py#L139-L192)）

```
用户代码调用 st.rerun(scope="app"|"fragment")
    │
    ▼
校验 scope 合法性
    │
    ▼
获取 ctx，收集当前运行时快照：
  - query_string
  - page_script_hash
  - cached_message_hashes
  - fragment_id_queue（scope="fragment" 时构造）
    │
    ▼
ctx.script_requests.request_rerun(RerunData)  ──→ 设置 state = RERUN，保存数据
    │
    ▼
st.empty()  ──→ 强制触发 yield point
```

### 3.3 `st.switch_page()` 调用链（[execution_control.py#L194-L347](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/commands/execution_control.py#L194-L347)）

与 `st.rerun()` 基本相同，区别在于：
1. 额外校验目标页面路径 / StreamlitPage 合法性
2. 先设置 `query_params`（切换页面时重置 query）
3. **sleep 2 倍 flush 间隔**：确保 query params 消息先被前端接收
4. RerunData 中 `page_script_hash` 设置为目标页面

---

## 四、控制信号状态机（第二层）

### 4.1 ScriptRequestType 三态（[script_requests.py#L30-L42](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L30-L42)）

| 状态 | 含义 | 是否终态 |
|------|------|----------|
| `CONTINUE` | 脚本正常运行 | 否 |
| `STOP` | 脚本需停止（不可逆转） | **是** |
| `RERUN` | 脚本需重跑（可被合并/转换） | 否 |

状态转移图：
```
          request_stop()
  ┌───────────────────────────────────┐
  │                                   ▼
CONTINUE ──request_rerun()──► RERUN ──request_stop()──► STOP
  ▲                              │
  │                              │ on_scriptrunner_yield()
  │                              │ 或 on_scriptrunner_ready()
  └──────────────────────────────┘
            返回 CONTINUE
```

### 4.2 `request_stop()`（[script_requests.py#L169-L174](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L169-L174)）

```python
def request_stop(self) -> None:
    with self._lock:
        self._state = ScriptRequestType.STOP  # 无条件，原子操作
```

- **无返回值**，无条件成功
- **一经设置不可撤回**：`request_rerun()` 在 STOP 下返回 False

### 4.3 `request_rerun(new_data)`（[script_requests.py#L176-L248](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L176-L248)）

| 当前状态 | 行为 | 返回值 |
|----------|------|--------|
| STOP | 拒绝请求 | `False` |
| CONTINUE | 保存 `new_data`，设置 state=RERUN | `True` |
| RERUN | **合并**新旧 RerunData：<br>• WidgetStates：新值优先，trigger 值保留<br>• fragment_id_queue：追加或替换<br>• full-app rerun 清空 fragment 队列 | `True` |

> **语义混淆点 1**：STOP 为终态，所以 `st.stop()` 之后同一 ScriptRunner 内的 `st.rerun()` 会被静默忽略（返回 False）。

### 4.4 `on_scriptrunner_yield()`（[script_requests.py#L250-L291](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L250-L291)）

脚本运行中每个 yield 点调用，双次检查（fast-path + 锁内确认）：

| 当前状态 | 额外条件 | 返回 | 副作用 |
|----------|----------|------|--------|
| CONTINUE | - | `None` | 无 |
| RERUN | fragment 非抢占式（auto fragment） | `None` | 无（等完整跑完再处理） |
| RERUN | 其他情况 | `ScriptRequest(RERUN, data)` | state → CONTINUE |
| STOP | - | `ScriptRequest(STOP)` | state 保持 STOP |

> **语义混淆点 2**：fragment 的 auto-rerun（如 widget 交互触发）不会抢占当前执行，只会排队等下一轮；而 `st.rerun(scope="fragment")` 是抢占式的。

### 4.5 `on_scriptrunner_ready()`（[script_requests.py#L293-L311](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L293-L311)）

脚本**开始前**或**完成后**调用：

| 当前状态 | 返回 | 副作用 |
|----------|------|--------|
| RERUN | `ScriptRequest(RERUN, data)` | state → CONTINUE（继续循环） |
| CONTINUE / STOP | `ScriptRequest(STOP)` | state → STOP（退出循环） |

> **语义混淆点 3**：CONTINUE 也会被转成 STOP！意味着脚本跑完一轮后，如果没有新的 RERUN 请求排队，ScriptRunner 就直接进入终态，等待下一次 `AppSession.request_rerun()` 创建新的 runner。

---

## 五、执行引擎层（第三层）：ScriptRunner 主循环

### 5.1 线程入口 `_run_script_thread()`（[script_runner.py#L378-L435](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L378-L435)）

```
创建 ScriptRunContext，绑定到线程
    │
    ▼
request = on_scriptrunner_ready()   ◄──┐
    │                                   │
    ▼                                   │
while request.type == RERUN:            │ 外层循环
    │                                   │ （驱动多轮脚本执行）
    ├─ _run_script(request.rerun_data)  │
    │                                   │
    └─ request = on_scriptrunner_ready() ┘
    │
    ▼
收到 STOP → 发送 SHUTDOWN 事件，线程退出
```

### 5.2 Yield 点检查 `_maybe_handle_execution_control_request()`（[script_runner.py#L457-L511](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L457-L511)）

**被调用时机**：
1. 每次 `_enqueue_forward_msg()`（即几乎每个 `st.xxx` 命令）
2. `SafeSessionState` 的 `yield_callback`（会话状态访问时）
3. `code_to_exec` 执行完毕后的显式检查

检查顺序（优先级从高到低）：

```
① 线程检查：非脚本线程？
   ├─ 是 → 检查 parallel_coordinator.should_stop() → 抛 StopException
   └─ 否 → 继续

② _execing 标志：脚本不在 exec() 中？
   └─ 是 → 直接返回（不处理）

③ Parallel worker 异常：parallel_coordinator.worker_exception ?
   └─ 有 → 重新抛出该异常（用户代码异常优先）

④ on_scriptrunner_yield() → request
   ├─ None → 返回（无事发生）
   ├─ RERUN → raise RerunException(rerun_data)
   └─ STOP  → raise StopException()
```

> **语义混淆点 4**：parallel worker 的异常优先级比 rerun/stop 信号更高。这意味着并行 fragment 中抛出的用户异常会先被传递。

### 5.3 `_run_script(rerun_data)` 内层循环（[script_runner.py#L528-L856](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L528-L856)）

```
while True:  ◄─────────────────────────┐
  │                                     │ 内层循环
  ├─ ① 准备阶段                         │ （驱动 st.rerun() 触发的
  │   • LocalSourcesWatcher 回调         │   同线程内重跑）
  │   • 清理 media session refs
  │   • 设置页面意图，获取 active_script
  │   • 页面切换时：清理 widget state、处理 query params
  │   • 构造 fragment_ids_this_run
  │   • ctx.reset(...)
  │   • 发送 SCRIPT_STARTED 事件
  │
  ├─ ② 编译阶段
  │   • 通过 ScriptCache 获取字节码
  │   • 编译失败 → SCRIPT_STOPPED_WITH_COMPILE_ERROR，return
  │
  ├─ ③ 执行阶段
  │   • 构造 __main__ 模块
  │   • code_to_exec() 中包含：
  │   │   ├─ modified_sys_path()
  │   │   ├─ _set_execing_flag()  ← 设置 _execing=True
  │   │   ├─ on_script_will_rerun() ← widget 回调执行
  │   │   ├─ ctx.on_script_start()
  │   │   ├─ fragment 模式：逐个运行 fragment 函数
  │   │   └─ full 模式：exec(code) + coordinator.join()
  │   │       └─ 所有 st.xxx 间接触发第五节检查
  │   └─ 末尾显式调用 _maybe_handle_execution_control_request()
  │
  ├─ ④ exec_func_with_error_handling(...) 捕获异常 ──→ 进入第六节
  │
  ├─ ⑤ 结果判定
  │   ├─ rerun_exception_data ≠ None
  │   │   ├─ finished_event = SCRIPT_STOPPED_FOR_RERUN
  │   │   ├─ _on_script_finished(...)
  │   │   ├─ rerun_data = rerun_exception_data
  │   │   └─ 继续内层 while True 循环 ◄─┘
  │   │
  │   └─ 否则（正常完成 / StopException / 用户异常）
  │       ├─ 根据 fragment_id_queue 设置 finished_event：
  │       │   • FRAGMENT_STOPPED_WITH_SUCCESS
  │       │   • SCRIPT_STOPPED_WITH_SUCCESS
  │       ├─ _on_script_finished(...)
  │       └─ break 跳出内层循环
  │
  └──────────────────────────────────────┘
```

> **语义混淆点 5**：存在**双层循环**！外层循环在 `_run_script_thread()` 中处理跨轮次的 RERUN 信号；内层循环在 `_run_script()` 中处理 `st.rerun()` 触发的同线程立即重跑（通过 RerunException 捕获后的 `continue`）。

---

## 六、异常捕获分类层（第四层）

### 6.1 `exec_func_with_error_handling()`（[exec_code.py#L76-L167](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner/exec_code.py#L76-L167)）

返回五元组：`(result, run_without_errors, rerun_exception_data, premature_stop, uncaught_exception)`

| 捕获的异常 | `run_without_errors` | `rerun_exception_data` | `premature_stop` | 其他副作用 |
|------------|----------------------|------------------------|------------------|------------|
| **无异常** | `True` | `None` | `False` | - |
| **RerunException** | `True` | `e.rerun_data` | **`False`** | 清理 cursors 和 dg_stack |
| **StopException** | `True` | `None` | **`True`** | - |
| **FragmentHandledException** | `False` | `None` | `True` | - |
| **普通 Exception** | `False` | `None` | `True` | `handle_user_script_exception()` 渲染错误 UI |

### 6.2 `premature_stop` 的含义（关键！）

| 值 | 含义 | 对 SessionState 的影响 |
|----|------|------------------------|
| `False` | **不是**提前停止，或因 rerun 停止 | `on_script_finished()` 正常执行，**清理未出现的 widget** |
| `True` | 脚本提前结束（stop / 用户异常） | **跳过** widget 清理，保留所有 widget 状态 |

> **语义混淆点 6（最大混淆点）**：
> - Rerun 虽然也中断了脚本，但 `premature_stop = False` → **会清理** widget 状态（因为下一轮会重建）
> - Stop 的 `premature_stop = True` → **不清理** widget 状态（保持页面显示）
>
> 这也是为什么 RerunException 在 `_run_script` 中决定"继续内层循环"，而 StopException 决定"break 退出"。

### 6.3 `_on_script_finished()`（[script_runner.py#L857-L883](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L857-L883)）

```
① _join_wake_coordinator = None

② premature_stop == False ?
   └─ 是 → session_state.on_script_finished(widget_ids)
           └─ 删除未出现的 widget，触发 computed state 重算

③ 发送 ScriptRunnerEvent（三种之一）：
   • SCRIPT_STOPPED_FOR_RERUN       ──→ 将进入内层循环下一轮
   • SCRIPT_STOPPED_WITH_SUCCESS    ──→ 外层 on_scriptrunner_ready() 决定去向
   • FRAGMENT_STOPPED_WITH_SUCCESS  ──→ 同上

④ 清理孤儿媒体文件
⑤ 可选：强制 GC
```

---

## 七、会话管理层（第五层）

### 7.1 AppSessionState 三态（[app_session.py#L79-L82](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L79-L82)）

```
APP_NOT_RUNNING  ──request_rerun()──►  APP_IS_RUNNING
       ▲                                    │
       │                                    │
       │                     ScriptRunnerEvent（成功/错误/完成）
       │                                    │
       └────────────────────────────────────┘

无论哪种状态，SHUTDOWN_REQUESTED 都可随时进入（不可逆转）
```

### 7.2 `request_rerun(client_state)`（[app_session.py#L406-L488](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L406-L488)）

```
① SHUTDOWN_REQUESTED ? → 丢弃请求，return

② 从 client_state 构造 RerunData（widget_states / query_string / fragment_id 等）

③ 已有 _scriptrunner ?
   │
   ├─ 是：
   │   ├─ fastReruns 开启 且 非 fragment rerun
   │   │   ├─ _scriptrunner.request_stop()  ◄── 先停止旧的
   │   │   └─ _scriptrunner = None           ◄── 丢弃引用
   │   │
   │   └─ 否则：
   │       └─ success = _scriptrunner.request_rerun(rerun_data)
   │          └─ success == True → return（复用现有 runner）
   │
   └─ 否 / request_rerun 返回 False：
       └─ _create_scriptrunner(rerun_data)   ◄── 新建 runner 并启动
```

> **语义混淆点 7**：
> - `fastReruns=True` 时，full-app rerun 不等当前 ScriptRunner 跑完，**先 STOP 再新建**，避免在旧脚本上抛 RerunException。
> - fragment rerun 始终复用现有 ScriptRunner，不创建新 runner。

### 7.3 `request_script_stop()`（[app_session.py#L490-L496](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L490-L496)）

只是透传：`_scriptrunner.request_stop()`

### 7.4 事件处理 `_handle_scriptrunner_event_on_event_loop()`（[app_session.py#L599-L699](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L599-L699)）

| 事件 | AppSessionState 变化 | 前端消息 | 其他副作用 |
|------|---------------------|----------|------------|
| `SCRIPT_STARTED` | → `APP_IS_RUNNING` | 1. `NEW_SESSION`<br>2. `session_status_changed`（如果状态变化） | 更新 `page_script_hash` 到 `_client_state` |
| `SCRIPT_STOPPED_WITH_SUCCESS` | → `APP_NOT_RUNNING` | 1. `SCRIPT_FINISHED`(SUCCESSFULLY)<br>2. `session_status_changed`（如果状态变化） | `update_watched_modules()` + `update_watched_pages()` |
| `SCRIPT_STOPPED_WITH_COMPILE_ERROR` | → `APP_NOT_RUNNING` | 1. `SCRIPT_FINISHED`(COMPILE_ERROR)<br>2. `session_status_changed`（如果状态变化） | 发送异常详情给前端 |
| `FRAGMENT_STOPPED_WITH_SUCCESS` | → `APP_NOT_RUNNING` | 1. `SCRIPT_FINISHED`(FRAGMENT_SUCCESSFULLY)<br>2. `session_status_changed`（如果状态变化） | `update_watched_modules()` + `update_watched_pages()` |
| `SCRIPT_STOPPED_FOR_RERUN` | → `APP_NOT_RUNNING` | 1. `SCRIPT_FINISHED`(**FINISHED_EARLY_FOR_RERUN**)<br>2. `session_status_changed`（RUNNING→NOT_RUNNING） | `update_watched_modules()`（不更新 pages） |
| `SHUTDOWN` | 不改变状态<br>（保持 NOT_RUNNING） | **无**前端消息 | 1. 保存 `_client_state`（query/page_hash/context_info）<br>2. `_scriptrunner = None`<br>3. SHUTDOWN_REQUESTED 时清理 media + session caches |
| `ENQUEUE_FORWARD_MSG` | 不改变 | 透传 forward_msg | - |

> **修正：原理解不一致点 1**
> 
> ❌ 之前理解：`SCRIPT_STOPPED_FOR_RERUN` 不改变 AppSessionState，也不发送 FINISHED 消息给前端
> 
> ✅ 实际代码（[app_session.py#L730-L738](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L730-L738)）：
> - `self._state = APP_NOT_RUNNING` → **会**改变状态
> - `_enqueue_forward_msg(FINISHED_EARLY_FOR_RERUN)` → **会**发送完成消息
> - 还会触发 `session_status_changed`（因为状态从 RUNNING 变为 NOT_RUNNING）
> - 紧接着下一轮 SCRIPT_STARTED 会再次把状态切回 RUNNING，所以用户看到的是"快速闪烁"而非完全无感

---

## 八、完整时序对比：st.stop() vs st.rerun()

### 8.1 `st.stop()` 时序

```
用户线程（脚本线程）                    AppSession/主线程
     │                                     │
     │  st.stop()                          │
     │    ├─ request_stop() → state=STOP   │
     │    └─ st.empty() → _enqueue...      │
     │         └─ _maybe_handle...         │
     │            ├─ on_scriptrunner_yield()
     │            │   → ScriptRequest(STOP)
     │            └─ raise StopException ─┐│
     │                                     ││
     │  exec_func_with_error_handling:    ││
     │    catch StopException             ││
     │    → premature_stop=True           ││
     │                                     ││
     │  _on_script_finished():            ││
     │    → premature_stop=True,          ││
     │      跳过 widget 清理              ││
     │    → emit SCRIPT_STOPPED_WITH_SUCCESS
     │                                     ││
     │  内层循环 break                     ││
     │                                     ││
     │  on_scriptrunner_ready()           ││
     │    → state=STOP(终态)              ││
     │    → return ScriptRequest(STOP)    ││
     │                                     ││
     │  外层循环退出                       ││
     │  emit SHUTDOWN ───────────────────►│
     │                                     │
     │                                     ▼
     │                          _handle_scriptrunner_event:
     │                            APP_IS_RUNNING → APP_NOT_RUNNING
     │                            发送 FINISHED_SUCCESSFULLY
     ▼
  线程终止
```

### 8.2 `st.rerun()` 时序

```
用户线程（脚本线程）                    AppSession/主线程
     │                                     │
     │  st.rerun()                         │
     │    ├─ request_rerun(data)           │
     │    │   → state=RERUN, 存 data       │
     │    └─ st.empty() → _enqueue...      │
     │         └─ _maybe_handle...         │
     │            ├─ on_scriptrunner_yield()
     │            │   → ScriptRequest(RERUN)
     │            │   → state=CONTINUE
     │            └─ raise RerunException ─┐│
     │                                     ││
     │  exec_func_with_error_handling:    ││
     │    catch RerunException            ││
     │    → rerun_exception_data = data   ││
     │    → premature_stop=False          ││
     │    → 清理 cursors/dg_stack         ││
     │                                     ││
     │  _on_script_finished():            ││
     │    → premature_stop=False          ││
     │      执行 widget 清理              ││
     │    → emit SCRIPT_STOPPED_FOR_RERUN ││
     │                                     ││
     │  内层循环：rerun_data = data        ││
     │  （不 break，继续下一轮）            ││
     │                                     ││
     │  ┌──── 重新开始 _run_script ─────┐  │
     │  │  准备、编译、执行（新数据）      │  │
     │  └───────────────────────────────┘  │
     │                                     │
     │  ...直到没有 RerunException...      │
     │                                     │
     │  on_scriptrunner_ready()           ││
     │    ├─ 有排队 RERUN → 继续外层循环   ││
     │    └─ 无 → state→STOP, 退出         │
     ▼
```

---

## 九、语义混淆点汇总（对照表）

| # | 场景 | Rerun（主动重跑） | Stop（停止） |
|---|------|-------------------|--------------|
| 1 | ScriptRequests 终态后 | STOP 后 request_rerun 返回 False，被忽略 | STOP 为终态，不可逆 |
| 2 | Fragment 抢占性 | `scope="fragment"` 抢占；auto 不抢占 | 始终抢占 |
| 3 | 脚本完成后的默认行为 | 有 RERUN 排队则 CONTINUE，否则 STOP | CONTINUE/STOP 都转成 STOP |
| 4 | Parallel worker 异常优先级 | 比 rerun 信号低（worker 异常先抛） | 同左 |
| 5 | 循环层级 | 内层 `_run_script` 内同线程循环 | 内层 break，外层循环退出，线程结束 |
| 6 | `premature_stop` 标志 | **False** → 清理 widget 状态 | **True** → 保留 widget 状态 |
| 7 | fastReruns 模式 | full-app 时 STOP 旧 runner，新建新 runner | 无 fastReruns 分支 |
| 8 | AppSession 状态事件 | SCRIPT_STOPPED_FOR_RERUN → **NOT_RUNNING**（暂态，很快被下一轮 START 切回 RUNNING） | SCRIPT_STOPPED_WITH_SUCCESS → NOT_RUNNING |
| 9 | 前端感知 | 发送 **FINISHED_EARLY_FOR_RERUN** + session_status_changed，紧接着 NEW_SESSION | 发送 FINISHED_SUCCESSFULLY + session_status_changed |
| 10 | 继承关系 | 同继承 BaseException，不被 except Exception 捕获 | 同左 |

---

## 十、"容易混在一起"的核心原因

1. **两者都是同一异常体系**：`StopException` 和 `RerunException` 都继承自 `ScriptControlException`，在 `except` 分支中经常被并列处理，但语义天差地别。

2. **调用入口代码几乎相同**：`st.stop()` 和 `st.rerun()` 都是先 request_xxx 再 `st.empty()`，区别仅在于 ScriptRequests 内部状态和异常携带的数据。

3. **都触发"脚本停止"**：用户视角看两者都让当前脚本不再继续执行后续代码；区别在于"停止之后做什么"——Rerun 立即清理重来，Stop 保留现场结束。

4. **`premature_stop` 的反直觉**：Rerun 明明"中断"了脚本，却 `premature_stop=False`，会执行 cleanup。这是因为从 widget 生命周期看，rerun 是新一轮的开始，旧元素可以清理；而 stop 是真的停了，要保留当前页面元素。

5. **双层循环 + 双状态机**：外层 ScriptRequests 三态 × 内层 AppSession 三态 × 双层循环的组合，让"停止"概念有多种层次（ScriptRequests.STOP、内层 break、外层退出、ScriptRunner.SHUTDOWN、AppSession.NOT_RUNNING），与"重跑"的多类触发（用户 st.rerun、前端交互、fastReruns 新建 runner、fragment auto）交织在一起。

---

## 十一、会话层边界——已停止 runner 与 fragment 重跑的交互

### 11.1 当遇到已停止的 runner 时，fragment 重跑的处理方式

`AppSession.request_rerun()` 的完整分支逻辑（[app_session.py#L466-L488](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L466-L488)）：

```
已有 _scriptrunner ?
    │
    ├─ 是：
    │   ├─ 条件 A：fastReruns 开启 AND 非 fragment rerun
    │   │   ├─ _scriptrunner.request_stop()      ◄── STOP 旧 runner
    │   │   └─ _scriptrunner = None              ◄── 丢弃引用
    │   │
    │   └─ 条件 B：其他（fastReruns 关闭 或 fragment rerun）
    │       ├─ success = _scriptrunner.request_rerun(rerun_data)
    │       │
    │       ├─ success == True  → return  ◄── 复用现有 runner，万事大吉
    │       │
    │       └─ success == False → 继续向下  ◄── runner 已 STOP，无法接受请求
    │
    └─ 否 / 上一步返回 False：
        └─ _create_scriptrunner(rerun_data)    ◄── 新建 runner 并启动
```

**`request_rerun()` 返回 False 的唯一条件**（[script_requests.py#L186-L189](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L186-L189)）：
```python
if self._state == ScriptRequestType.STOP:
    return False
```

> **关键边界点**：当 ScriptRunner 跑完一轮脚本后，`on_scriptrunner_ready()` 会将 state 从 CONTINUE 转为 STOP（见第 4.5 节混淆点 3）。此时 ScriptRunner 还未退出线程，正在准备发送 SHUTDOWN 事件，但 `_state` 已经是 STOP 终态。

**在这个时间窗口内，如果收到 fragment 重跑请求**：
1. `_scriptrunner.request_rerun()` → 返回 False（因为 `_state == STOP`）
2. 条件判断继续向下，走到 `_create_scriptrunner(rerun_data)`
3. **新建 ScriptRunner**，并将 `self._scriptrunner` 指向新实例

---

### 11.2 为何旧 runner 的事件会被忽略（sender 检查机制）

#### 11.2.1 跨线程事件传递机制

ScriptRunner 事件的传递是**异步跨线程**的（[app_session.py#L569-L597](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L569-L597)）：

```
脚本线程（ScriptRunner）                     主线程（EventLoop）
     │                                            │
     │  on_event.send(event)                      │
     │    └─ _on_scriptrunner_event(sender=self)  │
     │         └─ call_soon_threadsafe(...) ─────►│
     │                                            │  排队中...
     │                                            │
     │                                            ▼
     │                                    _handle_scriptrunner_event_on_event_loop(
     │                                        sender=old_runner,
     │                                        event=SHUTDOWN,
     │                                        ...
     │                                    )
```

事件从脚本线程发出后，通过 `call_soon_threadsafe()` 排队到主线程的 event loop。这意味着**事件的发出和处理之间存在时间差**。

#### 11.2.2 sender 身份检查

在事件处理函数的最开始，有一道关键的闸门（[app_session.py#L654-L661](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L654-L661)）：

```python
if sender is not self._scriptrunner:
    _LOGGER.debug("Ignoring event from non-current ScriptRunner: %s", event)
    return
```

**这是一个对象身份比较（`is` 而非 `==`）**，检查发送事件的 ScriptRunner 对象是否就是 AppSession 当前持有的 `_scriptrunner` 引用。

#### 11.2.3 典型时序：事件被忽略的完整过程

```
  时间轴 →
    │
    ├─ T1: ScriptRunner A 跑完脚本，on_scriptrunner_ready()
    │       → state = STOP
    │       → 外层 while 循环退出
    │
    ├─ T2: ScriptRunner A 准备发送 SHUTDOWN 事件
    │       （构造 ClientState，包含 query_string、page_script_hash 等）
    │
    ├─ T3: 前端发来 fragment 重跑请求（在 T2 之后，但在 T6 之前）
    │       ↓
    │       AppSession.request_rerun(fragment_id="xxx")
    │         ├─ _scriptrunner 仍是 A
    │         ├─ A.request_rerun() → 返回 False（A._state 已是 STOP）
    │         ├─ 新建 ScriptRunner B
    │         └─ self._scriptrunner = B  ◄── 引用已更新！
    │
    ├─ T4: ScriptRunner B 开始启动，连接事件处理器
    │
    ├─ T5: 主线程 event loop 开始处理排队的事件
    │
    ├─ T6: 处理 ScriptRunner A 的 SHUTDOWN 事件
    │       ├─ sender = A
    │       ├─ self._scriptrunner = B
    │       ├─ sender is not self._scriptrunner → True！
    │       └─ return  ◄── 事件被完全忽略！
    │
    └─ T7: ScriptRunner A 的线程静默退出，无人知晓
```

**被忽略的 SHUTDOWN 事件会导致两个严重问题**（[app_session.py#L427-L441](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L427-L441)）：

1. **`_client_state` 未保存**：SHUTDOWN 事件本应更新 `self._client_state` 为 A 最终的 ClientState（[app_session.py#L746-L753](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L746-L753)）。这意味着下次重跑时，可能丢失 context_info（如浏览器时区）等信息。

2. **`_scriptrunner` 引用不一致**：SHUTDOWN 事件本应将 `self._scriptrunner = None`（[app_session.py#L753](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L753)），但由于事件被忽略，虽然 `_scriptrunner` 已被设为 B，但 A 的清理逻辑未执行。更严重的是，如果 B 之后也完成了，它的 SHUTDOWN 会把引用置 None，系统状态变得混乱。

3. **full-app-run 清理不执行**：SHUTDOWN 事件关联的 media file 清理、session cache 清理等（[app_session.py#L749-L750](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L749-L750)）也不会执行。

---

### 11.3 这些分支与 fast rerun 的关联

#### 11.3.1 两条路径都会导致"旧 runner 事件被忽略"

| 触发场景 | 路径 | `_scriptrunner` 何时更新 | 旧 runner 状态 |
|----------|------|-------------------------|----------------|
| **fastReruns 模式**（full-app rerun） | [app_session.py#L467-L476](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L467-L476)：<br>`request_stop()` → `_scriptrunner = None` → 创建新 runner | 在创建新 runner **之前**就已置 None | 被显式 request_stop |
| **fragment 遇到 STOP 态 runner** | [app_session.py#L477-L488](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L477-L488)：<br>`request_rerun()` 返回 False → 创建新 runner | 在创建新 runner **之后**更新引用 | 已自然进入 STOP 终态 |

**共同点**：两者都会在旧 runner 的 SHUTDOWN 事件被处理之前，更新 `_scriptrunner` 引用，导致 sender 检查失败。

#### 11.3.2 关键差异：`fragment_storage.clear()` 的时机

**full-app 模式**（[script_runner.py#L798-L800](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L798-L800)）：
```python
# 只有 full-app 正常跑完才会走到这里
self._fragment_storage.clear(
    new_fragment_ids=ctx.new_fragment_ids.snapshot()
)
```

- `fragment_storage.clear()` 会**删除所有未在本次运行中重新注册的 fragment**
- 这意味着：full-app 跑完后，所有 `@st.fragment` 装饰的函数都被重新注册到新的存储中，旧的 fragment_id 全部失效

**fragment 模式**：
- 不会调用 `fragment_storage.clear()`，fragment_id 始终有效
- 但如果 fragment 是在 full-app 运行中被删除（如条件渲染不再走到 fragment），它也会被 `clear_stale_descendants` 清理

#### 11.3.3 fastReruns 与 fragment 的互斥设计

注意 `fastReruns` 条件中的 `not rerun_data.fragment_id`（[app_session.py#L468-L470](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L468-L470)）：

```python
if (
    bool(config.get_option("runner.fastReruns"))
    and not rerun_data.fragment_id  # 关键：只有非 fragment rerun 才走 fastReruns
):
```

**设计意图**：
- full-app rerun 可以安全地"杀旧建新"，因为新 runner 会重新执行整个脚本，重建所有状态
- fragment rerun 不能杀旧 runner，因为 fragment 依赖于当前 runner 的 `fragment_storage`、`session_state` 等上下文
- 但如果旧 runner 已经 STOP（跑完了一轮），就不得不新建 runner

---

### 11.4 防御性修复：fragment 存在性预检查

为了解决 issue #9921（对话框有时无法关闭），代码中增加了一道**提前检查**（[app_session.py#L427-L448](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L427-L448)）：

```python
# Early check whether this fragment still exists in the fragment storage or
# might have been removed by a full app run.
if fragment_id and not self._fragment_storage.contains(fragment_id):
    _LOGGER.info(
        "The fragment with id %s does not exist anymore - "
        "it might have been removed during a preceding full-app rerun.",
        fragment_id,
    )
    return  # 直接丢弃请求，不创建新 runner
```

**这道检查解决的问题**：
- 如果前面的 full-app runner 已经执行了 `fragment_storage.clear()`，那么该 fragment_id 已失效
- 此时即使创建了新 runner 去跑这个 fragment，也会在 `_run_script` 中抛出 `FragmentStorageKeyError`
- 提前检查可以避免创建不必要的 runner，也避免了新旧 runner 交替导致的事件丢失

**但这道检查不解决的问题**：
- 如果 full-app runner 跑完了脚本，但 `fragment_storage.clear()` 已经执行（fragment 已失效），此时检查能拦住
- 但如果 fragment 仍然有效（比如刚跑完还没触发 clear，或者根本没有 full-app 运行），检查通过，还是会创建新 runner，旧 runner 的 SHUTDOWN 事件还是会被忽略

---

### 11.5 完整的 Race Condition 全景图（issue #9921 场景）

```
 T0: 用户打开对话框（fragment "dialog_frag"）
     → fragment_id = "dialog_frag" 被注册到 fragment_storage

 T1: 用户点击对话框中的按钮，触发 fragment 交互
     → 前端发送 rerun_script BackMsg，fragment_id="dialog_frag"

 T2: 同时，某个 full-app 重跑正在进行（比如 fastReruns 触发）
     │
     ├─ T2a: ScriptRunner A（full-app）正常完成
     │      ├─ 执行 fragment_storage.clear()  ◄── "dialog_frag" 被清除！
     │      ├─ on_scriptrunner_ready()
     │      │   → state = STOP
     │      ├─ 准备发送 SHUTDOWN 事件
     │      └─ call_soon_threadsafe(SHUTDOWN, sender=A)  ◄── 排队到主线程
     │
     └─ T2b: 前端的 fragment 重跑请求到达 AppSession
          ├─ request_rerun(client_state.fragment_id="dialog_frag")
          │
          ├─ 检查 fragment_storage.contains("dialog_frag")
          │   → False！（因为 T2a 已经 clear 了）
          │
          └─ 直接 return，不做任何处理
              └─ 对话框不会关闭，因为没有新的 runner 去执行关闭逻辑
```

如果**没有**这道预检查，会发生更糟的情况：
```
 T2b 没有预检查的路径：
   ├─ _scriptrunner 仍是 A
   ├─ A.request_rerun() → 返回 False（A.state 已是 STOP）
   ├─ 创建 ScriptRunner B
   ├─ B 开始运行 fragment "dialog_frag"
   ├─ B 在 _run_script 中调用 fragment_storage.lookup("dialog_frag")
   │   → 抛出 FragmentStorageKeyError！
   └─ 整个脚本出错，对话框可能显示错误信息
```

---

### 11.6 语义混淆点补充（第 11-13 点）

| # | 场景 | 说明 |
|---|------|------|
| 11 | **STOP 态的两种含义** | ScriptRequests.STOP 既可以是"被 request_stop() 强制停止"，也可以是"脚本自然跑完进入终态"。后者虽然叫 STOP，但 ScriptRunner 还活着，还在发 SHUTDOWN 事件。 |
| 12 | **sender 检查的双刃剑** | `sender is not self._scriptrunner` 能有效防止过期事件干扰新 runner，但也会导致正常的 SHUTDOWN 清理逻辑被跳过。这是正确性 vs 简单性的权衡。 |
| 13 | **fastReruns 与 fragment 的不对称** | full-app rerun 可以主动 STOP 旧 runner 再新建；fragment rerun 只能被动接受旧 runner 的状态（复用或被拒）。这种不对称是理解所有边界情况的钥匙。 |

---

### 11.7 边界情况汇总决策树

```
收到 rerun 请求（带 fragment_id）
    │
    ├─ ▶ fragment_id 存在于 fragment_storage 吗？
    │   ├─ 否 → 丢弃请求（issue #9921 修复）
    │   └─ 是 → 继续
    │
    ├─ ▶ 已有 _scriptrunner 吗？
    │   ├─ 否 → 新建 runner，结束
    │   └─ 是 → 继续
    │
    ├─ ▶ fastReruns 开启 且 非 fragment？
    │   ├─ 是 → STOP 旧 runner，丢弃引用，新建 runner
    │   │      （旧 runner 的 SHUTDOWN 事件会被 sender 检查忽略）
    │   └─ 否 → 继续
    │
    ├─ ▶ _scriptrunner.request_rerun() 返回？
    │   ├─ True → 复用成功，结束
    │   └─ False → runner 已 STOP，新建 runner
    │            （旧 runner 的 SHUTDOWN 事件会被 sender 检查忽略）
    │
    └─ 新建 runner 完成
        └─ 新 runner 开始运行，旧 runner 的后续事件全部被忽略
```

---

## 十二、修正：旧 runner 事件被忽略的真实影响范围

### 12.1 原理解不一致点 2

> ❌ 之前理解：只有 SHUTDOWN 事件会被忽略，丢失的主要是 `_client_state`
>
> ✅ 实际代码：**可能有多个事件连续被忽略**，丢失范围涵盖前端状态、后端清理、会话状态三个维度。具体哪些事件被忽略，取决于时间窗口的大小。

### 12.2 被忽略事件的时间窗口

**时间窗口**：从旧 runner 发出第一个未处理事件，到 `self._scriptrunner` 被更新为新 runner 引用之后。

```
  时间轴 →
    │
    ├─ T1: 旧 runner A 发送 SCRIPT_STOPPED_* 事件
    │       ↓ call_soon_threadsafe
    │       事件进入主线程队列（未处理）
    │
    ├─ T2: 旧 runner A 发送 SHUTDOWN 事件
    │       ↓ call_soon_threadsafe
    │       事件进入主线程队列（未处理）
    │
    ├─ T3: 新请求到达 request_rerun()
    │       → 创建新 runner B
    │       → self._scriptrunner = B
    │
    └─ T4: 主线程开始处理队列中的事件
            ├─ 处理 A 的 SCRIPT_STOPPED_* → sender 检查失败，忽略
            └─ 处理 A 的 SHUTDOWN → sender 检查失败，忽略
```

**可能被忽略的事件列表**（按发送顺序）：

| 事件 | 发送时机 | 被忽略概率 |
|------|----------|-----------|
| `ENQUEUE_FORWARD_MSG` | 脚本运行过程中的每条 `st.xxx` 消息 | 低（如果脚本已跑完则没有） |
| `SCRIPT_STOPPED_WITH_SUCCESS` | 一轮脚本正常完成时 | 高 |
| `SCRIPT_STOPPED_FOR_RERUN` | 一轮脚本因 rerun 中断时 | 中 |
| `FRAGMENT_STOPPED_WITH_SUCCESS` | 一轮 fragment 完成时 | 高 |
| `SHUTDOWN` | ScriptRunner 线程退出前 | 最高 |

---

### 12.3 各类事件被忽略的具体损失

#### 12.3.1 `SCRIPT_STOPPED_WITH_SUCCESS` 被忽略

**后端损失**（[app_session.py#L688-L715](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L688-L715)）：
- ❌ `self._state = APP_NOT_RUNNING` 不执行 → 但新 runner 的 SCRIPT_STARTED 会设为 RUNNING，**最终状态可能是对的**（一直是 RUNNING）
- ❌ `_local_sources_watcher.update_watched_modules()` 不执行 → 文件监听模块列表可能过时
- ❌ `_local_sources_watcher.update_watched_pages()` 不执行 → 页面监听列表可能过时

**前端损失**（[App.tsx#L1593-L1657](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/frontend/app/src/App.tsx#L1593-L1657)）：
- ❌ `scriptFinishedHandlers` 不触发 → 依赖脚本完成回调的逻辑不执行
- ❌ `elements.clearStaleNodes()` 不执行 → **旧元素残留，不会被清理**
- ❌ `removeInactiveWidgetState()` 不执行 → widget 状态内存泄漏
- ❌ `connectionManager.incrementMessageCacheRunCount()` 不执行 → 消息缓存不过期
- ❌ `session_status_changed` 不发送 → 前端运行状态指示器可能不准

> **关键影响**：前端页面上会残留上一轮的元素，因为 `clearStaleNodes` 只在收到 `FINISHED_SUCCESSFULLY` 时才执行。如果接下来是 fragment 运行，它不会清空全量元素，页面就会有重复/残留元素。

#### 12.3.2 `FRAGMENT_STOPPED_WITH_SUCCESS` 被忽略

与 SCRIPT_STOPPED_WITH_SUCCESS 类似，区别在于：
- 前端收到的是 `FINISHED_FRAGMENT_RUN_SUCCESSFULLY`
- 同样会触发 `clearStaleNodes`，但只清理 fragment 范围内的 stale nodes
- `update_watched_pages` 仍然会调用

#### 12.3.3 `SCRIPT_STOPPED_FOR_RERUN` 被忽略

**后端损失**：
- ❌ `self._state = APP_NOT_RUNNING` 不执行
- ❌ `_local_sources_watcher.update_watched_modules()` 不执行
- ❌ `session_status_changed` 不发送

**前端损失**（[App.tsx#L1635-L1656](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/frontend/app/src/App.tsx#L1635-L1656)）：
- ✅ `scriptFinishedHandlers` **仍然会触发**（FINISHED_EARLY_FOR_RERUN 在第一分支）
- ✅ **不清理** stale elements（设计如此，避免闪烁）
- ✅ **不递增** message cache run count（设计如此，等新 session）
- ❌ `hasReceivedNewSession` 校验下的缓存递增逻辑跳过

> 注意：`FINISHED_EARLY_FOR_RERUN` 的设计初衷就是"快速过渡，不做清理"，所以即使被忽略，影响也相对较小。真正危险的是 `SCRIPT_STOPPED_WITH_SUCCESS` 被忽略。

#### 12.3.4 `SHUTDOWN` 被忽略

**后端损失**（[app_session.py#L740-L753](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L740-L753)）：
- ❌ `self._client_state = client_state` 不执行 → **会话级状态丢失**
  - `query_string`：脚本最终的查询参数
  - `page_script_hash`：脚本最终的页面哈希
  - `context_info`：浏览器时区、语言、用户代理等上下文信息
- ❌ `self._scriptrunner = None` 不执行 → 但新 runner 已覆盖引用，通常无害
- ❌ SHUTDOWN_REQUESTED 模式下：
  - `media_file_mgr.clear_session_refs()` 不执行 → 媒体文件内存泄漏
  - `clear_session_caches()` 不执行 → 缓存不清理

**前端损失**：无（SHUTDOWN 不向前端发消息）

> **关键影响**：`_client_state` 是会话级持久状态。如果下一次重跑是**内部触发**的（如文件变化、定时器），会直接使用 `self._client_state` 作为初始状态。如果 SHUTDOWN 事件被忽略，这些内部触发的重跑可能使用过时的 query_string 或 context_info。

---

### 12.4 与 fastReruns 的关联：有意忽略 vs 无意忽略

| 场景 | 忽略性质 | 原因 | 风险等级 |
|------|---------|------|---------|
| **fastReruns（full-app）** | ✅ **有意忽略** | 主动 STOP 旧 runner，丢弃引用，立即新建。新 runner 会重跑整个脚本，覆盖所有状态。 | 低 |
| **fragment 遇 STOP 态 runner** | ❌ **无意忽略** | 旧 runner 已自然跑完，被动发现无法复用，被迫新建。旧 runner 的完成事件还在队列中未处理。 | 中高 |

#### 12.4.1 为什么 fastReruns 下忽略是安全的

fastReruns 路径（[app_session.py#L467-L476](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L467-L476)）：
1. **先 STOP 再新建**：主动控制时机，不是被动发现
2. **full-app 覆盖式重跑**：新 runner 发 `NEW_SESSION` 消息，前端清空所有元素，重建 widget 状态
3. **新 runner 完成时会重做所有清理**：`update_watched_modules`、`clearStaleNodes` 等都会在新 runner 完成时执行
4. **client_state 由前端请求提供**：每次 rerun 都带新的 client_state，不依赖 SHUTDOWN 保存的旧值

所以 fastReruns 下忽略旧事件是**设计使然**，相当于"推倒重来"。

#### 12.4.2 为什么 fragment 下忽略有风险

fragment 遇 STOP 态路径（[app_session.py#L477-L488](file:///d:/fz/0601/solo-dogfeeding/code/230-streamlit/lib/streamlit/runtime/app_session.py#L477-L488)）：
1. **被动发现**：不知道旧 runner 处于什么阶段，有多少事件在队列中
2. **fragment 增量更新**：新 runner 只更新 fragment 范围内的元素，**不会清空全量元素**
3. **旧的完成事件如果被忽略**：`clearStaleNodes` 不执行 → 页面残留旧元素
4. **context_info 等会话状态可能丢失**：如果后续有内部触发的重跑，会用旧的 client_state

这也是 issue #9921 的根本原因之一——full-app run 的完成事件被忽略，前端不清理 stale 元素，导致对话框有时候"关不上"（实际上是旧的对话框元素没被清掉，和新的重叠在一起）。

---

### 12.5 事件忽略风险的分级评估

| 影响维度 | fastReruns（full-app） | fragment 遇 STOP 态 |
|---------|-----------------------|-------------------|
| 前端元素正确性 | 低风险（NEW_SESSION 清空） | **中高风险**（增量更新，旧元素残留） |
| 前端 widget 状态 | 低风险（重建） | **中风险**（不清理 inactive widget） |
| 前端消息缓存 | 低风险（新 run id） | **中风险**（不递增 cache run count） |
| 后端文件监听 | 低风险（新 runner 完成时更新） | 低风险（通常不致命） |
| 后端 client_state | 低风险（每次请求都带新的） | **中风险**（内部触发的重跑可能用旧值） |
| 后端媒体/缓存清理 | 低风险（新 runner 完成时或 SHUTDOWN_REQUESTED 时清理） | 低风险（通常有其他清理机制） |

---

### 12.6 语义混淆点补充（第 14-16 点）

| # | 场景 | 说明 |
|---|------|------|
| 14 | **忽略事件的范围** | 不是只忽略 SHUTDOWN，而是**所有来自旧 runner 的排队事件**都会被忽略，包括 SCRIPT_STOPPED_* 系列。 |
| 15 | **有意 vs 无意忽略** | fastReruns 路径下的忽略是有意设计（推倒重来），fragment 遇 STOP 态路径下的忽略是副作用（可能有 bug）。 |
| 16 | **SHUTDOWN 的真实作用** | 不是"通知前端关闭"，而是**保存会话快照**（client_state）和清理引用，供后续内部触发的重跑使用。 |

---

### 12.7 完整事件链对比：fastReruns vs fragment STOP 态

#### fastReruns 路径（有意忽略，安全）

```
T1: 前端发来 full-app rerun 请求
    → request_rerun()
    → fastReruns 开启且非 fragment
    → _scriptrunner.request_stop()  ◄── 主动 STOP
    → _scriptrunner = None          ◄── 主动丢弃引用
    → 创建新 runner B
    → B 开始运行，发 SCRIPT_STARTED + NEW_SESSION

T2: 旧 runner A 处理 STOP 请求，停止脚本
    → 发 SCRIPT_STOPPED_WITH_SUCCESS  ◄── 在队列中，sender=A
    → 发 SHUTDOWN                    ◄── 在队列中，sender=A

T3: 主线程处理事件
    → A 的 SCRIPT_STOPPED_* → sender 不是 B → 忽略
    → A 的 SHUTDOWN → sender 不是 B → 忽略
    → B 的 SCRIPT_STARTED → sender=B → 正常处理
       → NEW_SESSION 清空前端所有元素 ◄── 覆盖了忽略的影响

结果：前端页面被新的 full-app run 完全重建，忽略无感知。
```

#### fragment 遇 STOP 态路径（无意忽略，有风险）

```
T1: 旧 runner A 自然完成一轮 full-app 脚本
    → 发 SCRIPT_STOPPED_WITH_SUCCESS  ◄── 在队列中，sender=A
    → on_scriptrunner_ready() → state=STOP
    → 准备发 SHUTDOWN

T2: 前端发来 fragment rerun 请求（在 T1 事件被处理之前）
    → request_rerun(fragment_id="xxx")
    → _scriptrunner 仍是 A
    → A.request_rerun() → False（A.state 已是 STOP）
    → 创建新 runner B
    → self._scriptrunner = B  ◄── 引用更新
    → B 开始运行 fragment

T3: 主线程处理事件
    → A 的 SCRIPT_STOPPED_WITH_SUCCESS → sender 不是 B → 忽略！
       → 前端不执行 clearStaleNodes ◄── 旧元素残留
       → 前端不执行 removeInactiveWidgetState
       → 前端不递增 message cache
    → A 的 SHUTDOWN → sender 不是 B → 忽略！
       → _client_state 不更新 ◄── 会话状态丢失
    → B 的 SCRIPT_STARTED → sender=B → 正常处理
       → fragment 范围内更新元素
       → 但旧的 full-app 元素还在！

结果：页面上残留旧元素，可能出现重叠或"关不上"的现象（issue #9921）。
```
