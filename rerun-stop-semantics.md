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

| 事件 | AppSessionState 变化 | 前端消息 |
|------|---------------------|----------|
| `SCRIPT_STARTED` | → `APP_IS_RUNNING` | `NEW_SESSION`（清空重渲染） |
| `SCRIPT_STOPPED_WITH_SUCCESS` | → `APP_NOT_RUNNING` | `SCRIPT_FINISHED` + status=SUCCESSFULLY |
| `SCRIPT_STOPPED_WITH_COMPILE_ERROR` | → `APP_NOT_RUNNING` | `SCRIPT_FINISHED` + status=COMPILE_ERROR |
| `FRAGMENT_STOPPED_WITH_SUCCESS` | → `APP_NOT_RUNNING` | `SCRIPT_FINISHED` + status=FRAGMENT_SUCCESSFULLY |
| `SCRIPT_STOPPED_FOR_RERUN` | **保持当前**（不切换） | 仅发送脚本中的消息 |
| `SHUTDOWN` | → `_scriptrunner = None` | 保存 `_client_state` 供下次连接使用 |

> **语义混淆点 8**：`SCRIPT_STOPPED_FOR_RERUN` 不改变 AppSessionState，也不发送 FINISHED 消息给前端，因为紧接着就是下一轮 SCRIPT_STARTED。用户感知不到中间有"停止"。

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
| 8 | AppSession 状态事件 | SCRIPT_STOPPED_FOR_RERUN → 保持 RUNNING | SCRIPT_STOPPED_WITH_SUCCESS → NOT_RUNNING |
| 9 | 前端感知 | 不发送 FINISHED，直接下一轮 START | 发送 FINISHED_SUCCESSFULLY |
| 10 | 继承关系 | 同继承 BaseException，不被 except Exception 捕获 | 同左 |

---

## 十、"容易混在一起"的核心原因

1. **两者都是同一异常体系**：`StopException` 和 `RerunException` 都继承自 `ScriptControlException`，在 `except` 分支中经常被并列处理，但语义天差地别。

2. **调用入口代码几乎相同**：`st.stop()` 和 `st.rerun()` 都是先 request_xxx 再 `st.empty()`，区别仅在于 ScriptRequests 内部状态和异常携带的数据。

3. **都触发"脚本停止"**：用户视角看两者都让当前脚本不再继续执行后续代码；区别在于"停止之后做什么"——Rerun 立即清理重来，Stop 保留现场结束。

4. **`premature_stop` 的反直觉**：Rerun 明明"中断"了脚本，却 `premature_stop=False`，会执行 cleanup。这是因为从 widget 生命周期看，rerun 是新一轮的开始，旧元素可以清理；而 stop 是真的停了，要保留当前页面元素。

5. **双层循环 + 双状态机**：外层 ScriptRequests 三态 × 内层 AppSession 三态 × 双层循环的组合，让"停止"概念有多种层次（ScriptRequests.STOP、内层 break、外层退出、ScriptRunner.SHUTDOWN、AppSession.NOT_RUNNING），与"重跑"的多类触发（用户 st.rerun、前端交互、fastReruns 新建 runner、fragment auto）交织在一起。
