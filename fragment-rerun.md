# Streamlit 局部重跑（Fragment Rerun）机制深度解析

局部重跑是 Streamlit 最精妙也最易误解的特性之一。核心问题：**当用户点击 fragment 内的按钮时，如何做到只重跑 fragment 代码，而页面其余部分的状态、UI 元素、Widget 值均保持不变？**

答案取决于三套机制的精密配合：

1. **片段边界（Fragment Boundary）** — 用 `delta_path` 划分 UI 树的作用域，明确 fragment 内部与外部的分界线
2. **状态继承（State Inheritance）** — 在定义时快照上下文状态，在重跑时恢复，使 fragment 能"回到原点"重新渲染
3. **执行调度（Execution Scheduling）** — 通过 `fragment_id_queue` 和 `RerunData` 协调脚本线程，让"局部重跑"替代"全局重跑"

三者的关系可以用一句话概括：**调度决定"跑哪个函数"，状态继承决定"在哪个上下文里跑"，边界决定"跑出来的结果送到前端哪里"。**

---

## 一、片段边界：delta_path 定义的作用域围栏

### 1.1 什么是片段边界

片段边界不是语法上的函数大括号，而是 **UI 元素树中的子树范围**。每个 Streamlit 元素在前端的 DOM 树中有一个唯一的位置坐标，称为 `delta_path` —— 一个整数元组，例如 `(0, 3, 1, 2)`，表示"第 0 个根容器 → 第 3 个区块 → 第 1 个子区块 → 第 2 个元素"。

当 `@st.fragment` 装饰的函数被调用时，Streamlit 会自动在其外部包裹一个 `st.container()`，这个容器的 `delta_path` 前缀就是该 fragment 的**边界围栏**。所有 fragment 内部产生的元素，其 `delta_path` 都必须以这个前缀开头。

> 代码定位：[wrapped_fragment 中的 delta_path 设置](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L417-L441)

```python
with ThreadState.scoped(fragment_id=fragment_id):
    result = None
    with active_hash_context:
        container_ctx = (
            contextlib.nullcontext() if skip_container else st.container()
        )
        with container_ctx:
            active_dg = context_dg_stack.get()[-1]
            ThreadState.update(
                delta_path=tuple(
                    (active_dg._cursor.delta_path if active_dg._cursor else [])[:-1]
                )
            )
            result = non_optional_func(*args, **kwargs)
```

关键点：
- `st.container()` 会占用 UI 树中的一个位置（产生一个新的 `delta_path` 索引）
- `delta_path` 被设置为容器路径的 **`[:-1]`**（去掉最后一位），因为容器自身占了一个位置，内部元素从子索引开始
- 例如容器路径是 `[0, 3, 0]`，则 fragment 的 `delta_path` 为 `(0, 3)`，fragment 内部所有元素的路径都将以 `(0, 3, ...)` 开头
- 这个 `delta_path` 被存入线程级别的 `ThreadState` 中，后续所有 `st.*` 调用都可以通过它判断"我在不在 fragment 内部"

### 1.2 边界检查的两种场景

#### 场景 A：平行 fragment 的外部写入禁止

当 fragment 以 `parallel=True` 运行时（多个线程并发执行），严禁写入 fragment 边界外的容器。因为多个线程同时操作同一个 `RunningCursor` 会产生数据竞争。

> 代码定位：[delta_generator.py 中的平行写入检查](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/delta_generator.py#L507-L526)

```python
ts = ThreadState.get()
if ts.is_parallel_worker:
    fragment_path = ts.delta_path
    cursor_path = tuple(dg._cursor.delta_path) if dg._cursor else ()
    if fragment_path and not _is_inside_fragment_path(cursor_path, fragment_path):
        raise StreamlitAPIException(
            "Writing to containers outside a parallel fragment is not "
            "allowed during the initial page load..."
        )
```

判定函数 `_is_inside_fragment_path` 的逻辑非常简洁：

> 代码定位：[_is_inside_fragment_path 前缀匹配](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/delta_generator.py#L726-L733)

```python
def _is_inside_fragment_path(
    cursor_path: tuple[int, ...],
    fragment_path: tuple[int, ...],
) -> bool:
    if len(cursor_path) < len(fragment_path):
        return False
    return cursor_path[: len(fragment_path)] == fragment_path
```

**前缀匹配原则**：只要 cursor 的路径前缀等于 fragment 路径，就算在边界内。这与文件系统路径匹配的逻辑一致——`/a/b/c` 在 `/a/b` 的边界内。

#### 场景 B：顺序 fragment 的 Widget 位置约束

即使是顺序执行的 fragment，Widget 也不能写到边界外。这由 `check_fragment_path_policy` 保证（与上面的 `_is_inside_fragment_path` 逻辑类似）。

非 Widget 元素（如 `st.write`）**可以**写到外部容器，但会在每次 fragment 重跑时累积，直到下一次全局重跑才会被清理——这正是文档中提到的"elements will accumulate in those containers"的根因。

### 1.3 边界的核心作用：局部清理

fragment 重跑时，前端只清理该 fragment 容器内的旧元素，然后重新渲染。这是通过消息中的 `fragment_id` 元数据实现的：

> 代码定位：[enqueue_message 中的 fragment_id 标记](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L475-L478)

```python
ts = ThreadState.get()
if ts.fragment_id and msg.WhichOneof("type") == "delta":
    msg.delta.fragment_id = ts.fragment_id
```

每个 fragment 产生的 `ForwardMsg` 都会被打上 `fragment_id` 标签，前端据此知道哪些 delta 属于同一个 fragment，从而在重跑时只替换对应子树的内容。**没有 `fragment_id` 标签的消息，前端会当作全局消息处理。**

**边界机制小结**：边界由两件事共同完成——后端用 `delta_path` 前缀做写入范围校验，前端用 `fragment_id` 做局部替换。两者缺一不可。

---

## 二、状态继承：定义时快照 + 重跑时恢复

这是理解 fragment 的最关键机制。问题在于：**当 fragment 单独重跑时，它怎么知道"应该渲染到页面的哪个位置"？** 毕竟全局脚本并没有执行，`cursors` 和 `context_dg_stack` 都被 `ctx.reset()` 清空了。

答案是：**在 fragment 定义时（即全局脚本第一次执行到装饰器调用时），把当时的上下文状态做一个深拷贝快照，等 fragment 单独重跑时，用这个快照恢复现场。**

### 2.1 快照的四件套

> 代码定位：[wrap() 函数中的快照捕获](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L366-L376)

在 `wrap()` 函数中（装饰器被调用时执行），会捕获四个关键状态：

```python
parent_fragment_id_at_def = ThreadState.get().fragment_id

cursors_snapshot = deepcopy(ctx.cursors)
dg_stack_snapshot = deepcopy(context_dg_stack.get())
fragment_id = calc_hash(
    f"{non_optional_func.__module__}.{get_object_name(non_optional_func)}"
    f"{dg_stack_snapshot[-1]._get_delta_path_str()}{additional_hash_info}"
)

initialized_active_script_hash = ThreadState.get().active_script_hash
```

四个值各自的角色：

| 快照值 | 含义 | 为什么需要 |
|--------|------|-----------|
| `parent_fragment_id_at_def` | 定义时的父 fragment ID | 支持嵌套 fragment 的树形结构，存储时记录父子关系 |
| `cursors_snapshot` | 每个根容器的游标位置 | 重跑时恢复游标，使新产生的 `delta_path` 与首次一致 |
| `dg_stack_snapshot` | DeltaGenerator 栈 | 重跑时恢复容器嵌套层级，使 `st.container()` 等上下文管理器能正确工作 |
| `initialized_active_script_hash` | 当前脚本 hash | 确保 Widget ID 计算与定义时一致（跨页面导航场景） |

这四个值通过闭包被 `wrapped_fragment` 函数永久捕获。

### 2.2 恢复逻辑：只有 fragment 重跑时才恢复

在 `wrapped_fragment()` 执行时（这是装饰器返回的实际运行函数），会判断当前是否处于 fragment 重跑中：

> 代码定位：[wrapped_fragment 中的快照恢复](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L389-L410)

```python
if ctx.fragment_ids_this_run:
    ctx.cursors = deepcopy(cursors_snapshot)
    context_dg_stack.set(deepcopy(dg_stack_snapshot))

ctx.new_fragment_ids.check_and_add(fragment_id)

active_hash_context = (
    ctx.run_with_active_hash(initialized_active_script_hash)
    if initialized_active_script_hash != ThreadState.get().active_script_hash
    else contextlib.nullcontext()
)
```

**核心洞察**：
- `ctx.fragment_ids_this_run` 非空 = 这是一次 fragment 局部重跑（不是全局脚本执行）
- 恢复 `cursors` 和 `dg_stack` 后，从 fragment 的视角看，世界和"全局脚本第一次执行到这里时"完全一致
- 所以 fragment 内部调用 `st.button`、`st.write` 等命令时，**游标的推进路径与第一次完全相同**，产生的 `delta_path` 也完全相同
- 这就是为什么 fragment 重跑时元素能精确渲染到原来的位置，而不是堆到页面顶部

**为什么不做快照恢复就不行**？看 [ctx.reset()](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L267) 就明白了：

```python
self.cursors = {}
```

`reset()` 会把 `cursors` 清空成空字典。如果不恢复快照，fragment 内部的元素会从 `(0, 0)` 开始渲染，全部堆到页面的左上角。

### 2.3 Widget ID 的一致性保证

Streamlit 的 Widget ID 是由 `(active_script_hash + key/label + delta_path)` 哈希计算的。`active_script_hash` 决定了"这个 Widget 属于哪个页面脚本"。

如果 fragment 定义在页面 A，然后用户导航到页面 B 后又回来，`active_script_hash` 就会不同。如果不做处理，Widget ID 会变，导致：
- 用户在 fragment 里输入的文本丢失
- `st.session_state` 中的 key 对不上

因此 `initialized_active_script_hash` 被冻结在**定义时刻**的值。每次 fragment 执行（无论全局还是局部），如果当前 `active_script_hash` 与定义时不同，就会通过 `ctx.run_with_active_hash()` 临时切换回去，确保 Widget ID 的跨运行一致性。

### 2.4 状态继承本质上是一次"时间旅行"

把 fragment 的执行环境精确回滚到它被定义那一刻的上下文，只在这个隔离沙盒内执行代码，然后把产生的 delta 通过 `fragment_id` 标记送回前端做局部替换。

**注意边界**：快照恢复只影响 `cursors` 和 `dg_stack`，不影响 `st.session_state`——session_state 是全局共享的，fragment 重跑时的修改对其他 fragment 和全局脚本可见。

---

## 三、执行调度：fragment_id_queue 驱动的分支执行

前面解决了"怎么定义边界"和"怎么恢复状态"，现在的问题是：**当用户点击 fragment 内的按钮时，ScriptRunner 怎么知道应该只跑 fragment 而不是全局脚本？**

### 3.1 三层调度对象

调度链路由三个数据结构串联：

| 层次 | 结构 | 作用 | 代码位置 |
|------|------|------|----------|
| 请求层 | `RerunData` | 携带"这次重跑是全局还是局部、要跑哪些 fragment"的信息 | [RerunData 定义](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L44-L67) |
| 上下文层 | `fragment_ids_this_run` | 存放在 `ScriptRunContext` 中，告知当前执行线程"你在 fragment 重跑模式" | [fragment_ids_this_run 字段](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L221) |
| 存储层 | `FragmentStorage` | 根据 fragment_id 取出被注册的 `wrapped_fragment` 闭包函数 | [FragmentStorage 协议](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L79-L173) |

三层之间的关系：**请求层**把"要跑哪些 fragment"装进 `RerunData.fragment_id_queue` → **上下文层**在 `ctx.reset()` 时把这个队列设为 `ctx.fragment_ids_this_run` → **存储层**在执行时根据 `fragment_id` 从 `FragmentStorage` 取出闭包函数执行。

### 3.2 RerunData 的关键字段

> 代码定位：[RerunData dataclass](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L44-L67)

```python
@dataclass(frozen=True)
class RerunData:
    query_string: str = ""
    widget_states: WidgetStates | None = None
    page_script_hash: str = ""
    page_name: str = ""

    fragment_id: str | None = None
    fragment_id_queue: list[str] = field(default_factory=list)
    is_fragment_scoped_rerun: bool = False
    is_auto_rerun: bool = False
    cached_message_hashes: set[str] = field(default_factory=set)
    context_info: ContextInfo | None = None
```

需要区分两个 fragment 相关字段：
- **`fragment_id`**：由前端传来的**单个**触发源 fragment，会在 `ScriptRequests.request_rerun()` 中被转换为 `fragment_id_queue` 的一个元素
- **`fragment_id_queue`**：本次要执行的 fragment 队列，可能包含多个 fragment（如多个定时器同时触发）

转换逻辑在 [request_rerun](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L197-L203)：

```python
if new_data.fragment_id:
    new_data = replace(
        new_data,
        fragment_id=None,
        fragment_id_queue=[new_data.fragment_id],
    )
```

### 3.3 从前端到执行：完整调度链路

```
前端用户点击 fragment 内的按钮
    ↓
BackMsg 携带 fragment_id 发送到服务端
    ↓
AppSession.request_rerun() 创建 RerunData(fragment_id=..., ...)   ← [app_session.py#L453-L462]
    ↓
ScriptRequests.request_rerun() 把 fragment_id 转为 fragment_id_queue  ← [script_requests.py#L197-L203]
    ↓
ScriptRunner._run_script 从 RERUN 请求中取出 rerun_data
    ↓
fragment_id_queue 非空 → 调用 order_fragment_ids() 排序     ← [script_runner.py#L614-L617]
    ↓
ctx.reset(fragment_ids_this_run=排序后的队列)                ← [script_runner.py#L619-L626]
    ↓
进入 if fragment_ids_this_run: 分支                          ← [script_runner.py#L715]
    ↓
逐个从 FragmentStorage.lookup(fragment_id) 取出闭包执行       ← [script_runner.py#L720-L748]
```

### 3.4 ScriptRunner 的执行分支

在 [ScriptRunner._run_script](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L715-L800) 的核心循环中，有一个关键的 `if/else` 分叉：

```python
ctx.on_script_start()

if fragment_ids_this_run:
    for fragment_id in fragment_ids_this_run:
        registration_sequence_before = (
            self._fragment_storage.registration_sequence()
        )
        try:
            wrapped_fragment = self._fragment_storage.lookup(fragment_id)
        except FragmentStorageKeyError:
            continue

        try:
            wrapped_fragment()
        except (RerunException, StopException):
            raise
        except Exception:
            pass
        finally:
            registered_ids = self._fragment_storage.ids_registered_after(
                registration_sequence_before
            )
            self._fragment_storage.clear_stale_descendants(
                fragment_id, registered_ids
            )
else:
    coordinator = cast("ParallelFragmentCoordinator", ctx.parallel_coordinator)
    try:
        if PagesManager.uses_pages_directory:
            _mpa_v1(self._main_script_path)
        else:
            exec(code, module.__dict__)
        coordinator.join()
    except BaseException:
        coordinator.drain()
        raise
    self._fragment_storage.clear(
        new_fragment_ids=ctx.new_fragment_ids.snapshot()
    )
```

两个分支的差异对照：

| 维度 | 分支 A：fragment 重跑 | 分支 B：全局重跑 |
|------|----------------------|-----------------|
| 执行单元 | 逐个执行 `fragment_id_queue` 中的闭包 | `exec(code)` 执行整个脚本文件 |
| 游标初始化 | `wrapped_fragment` 内部恢复各自的快照 | `ctx.reset()` 把所有游标置空 |
| fragment 存储 | `clear_stale_descendants()` 清理子树孤儿 | `clear()` 清理所有未重注册的 fragment |
| 平行 fragment | 无（全部顺序执行） | 通过 `ParallelFragmentCoordinator` 并发调度 |
| 异常处理 | 单个 fragment 异常被吞，不中断队列 | 异常中止整个脚本 |

**孤儿清理的差异**尤其值得注意：
- 分支 A 只清理 `clear_stale_descendants(root_fragment_id, newly_registered_ids)`——如果嵌套子 fragment 在本次重跑中没有重新注册（比如条件分支变了），它就被视为"孤儿"移除
- 分支 B 执行 `clear(new_fragment_ids=...)`——所有不在本次全局脚本执行中重新注册的 fragment 都会被清理

### 3.5 st.rerun(scope="fragment") 的调度逻辑

用户可以在 fragment 内显式调用 `st.rerun(scope="fragment")` 来触发自身重跑。这个调用的处理非常特殊：

> 代码定位：[_new_fragment_id_queue](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/commands/execution_control.py#L70-L111)

```python
def _new_fragment_id_queue(
    ctx: ScriptRunContext, scope: Literal["app", "fragment"]
) -> list[str]:
    if scope == "app":
        return []

    curr_queue = ctx.fragment_ids_this_run

    if not curr_queue:
        raise StreamlitAPIException(
            'scope="fragment" can only be specified from `@st.fragment`-decorated '
            "functions during fragment reruns."
        )

    new_queue = list(
        dropwhile(lambda x: x != ThreadState.get().fragment_id, curr_queue)
    )
    return new_queue
```

设计意图：
- **禁止全局运行时调用**：如果当前是全局脚本执行（`fragment_ids_this_run` 为空），即使你在 fragment 函数体内，也不能用 `scope="fragment"`。因为这时脚本可能还执行到了 fragment 后面的代码，如果突然跳到 fragment 重新执行，后面的全局代码就被跳过了，状态会不一致
- **截断队列**：如果队列是 `[A, B, C]`，正在执行 B 时调用了 `scope="fragment"`，新队列变成 `[B, C]`。B 重跑后 C 仍然会被顺序执行，保持原有调度顺序

### 3.6 排队与祖先优先排序

当多个 fragment 同时触发重跑（例如两个 `run_every` 定时器同时到期），它们会排队执行。排序算法保证**祖先 fragment 先于后代执行**：

> 代码定位：[order_fragment_ids](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L278-L306)

```python
def order_fragment_ids(self, fragment_ids: list[str]) -> list[str]:
    def has_queued_ancestor(fragment_id, queued):
        return any(
            ancestor_id in queued
            for ancestor_id in self._iter_ancestor_ids(fragment_id)
        )

    remaining = list(fragment_ids)
    ordered = []
    while remaining:
        queued = set(remaining)
        for i, fid in enumerate(remaining):
            if not has_queued_ancestor(fid, queued):
                ordered.append(fid)
                del remaining[i]
                break
        else:
            ordered.extend(remaining)
            break
    return ordered
```

算法是贪心选择：每一轮从剩余队列中挑出"没有任何祖先也在排队"的 fragment，保证父级先渲染、子级后渲染。这与 [MemoryFragmentStorage](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L180-L330) 中通过 `_parent_by_id` 维护的树形父子关系配合工作。

**为什么祖先必须先执行**？因为子 fragment 的 `cursors_snapshot` 和 `dg_stack_snapshot` 是在父 fragment 的上下文中捕获的。如果子 fragment 先于父 fragment 执行，恢复的快照可能对应的是父 fragment 重跑前的旧状态，导致渲染位置错误。

### 3.7 fragment 重跑不抢占全局脚本

一个精妙的设计：当全局脚本正在运行时，fragment 的重跑请求**不会抢占**正在运行的全局脚本。这由 `_fragment_run_should_not_preempt_script` 保证：

> 代码定位：[_fragment_run_should_not_preempt_script](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L86-L97)

```python
def _fragment_run_should_not_preempt_script(
    fragment_id_queue: list[str],
    is_fragment_scoped_rerun: bool,
) -> bool:
    return bool(fragment_id_queue) and not is_fragment_scoped_rerun
```

在 [on_scriptrunner_yield](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L261-L267) 中，如果这个函数返回 `True`，ScriptRunner 不会中断当前脚本：

```python
if self._state == ScriptRequestType.CONTINUE or (
    self._state == ScriptRequestType.RERUN
    and _fragment_run_should_not_preempt_script(
        self._rerun_data.fragment_id_queue,
        self._rerun_data.is_fragment_scoped_rerun,
    )
):
    return None  # 不中断，继续执行
```

逻辑是：只要 `fragment_id_queue` 非空且不是 `scope="fragment"` 触发的重跑，就不抢占。因为普通的 fragment 重跑（由 Widget 交互或定时器触发）只影响 fragment 自身的容器，不影响全局脚本的正确性，等全局脚本跑完再执行 fragment 即可。

而 `st.rerun(scope="fragment")` 的 `is_fragment_scoped_rerun=True`，此时不进入"不抢占"分支——因为这是用户显式请求的 fragment 重跑，应该立即生效。

### 3.8 多 fragment 请求的合并（coalescing）

当已有 RERUN 请求排队时，新的 fragment 请求会被合并到现有队列中：

> 代码定位：[request_rerun 合并逻辑](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L208-L243)

```python
if self._state == ScriptRequestType.RERUN:
    coalesced_states = _coalesce_widget_states(...)

    if new_data.fragment_id:
        fragment_id_queue = [*self._rerun_data.fragment_id_queue]
        if new_data.fragment_id not in fragment_id_queue:
            fragment_id_queue.append(new_data.fragment_id)
    elif new_data.fragment_id_queue:
        fragment_id_queue = new_data.fragment_id_queue
    else:
        fragment_id_queue = []
```

三种情况：
1. **新请求带单个 `fragment_id`**：追加到现有队列末尾（去重）
2. **新请求自带 `fragment_id_queue`**（来自 `st.rerun(scope="fragment")`）：直接用新队列替换
3. **新请求不带任何 fragment 信息**：清空队列——这是全局重跑请求，fragment 都会在全局脚本中重新执行

---

## 四、三套机制的配合：一个端到端的故事

把三套机制串起来，看一次完整的 fragment 重跑生命周期：

### 4.1 全局首次运行（注册阶段）

```
exec(code, module.__dict__)
    ↓ 执行到 @st.fragment 装饰的函数调用
wrap(*args, **kwargs) 被调用
    ↓ 捕获快照
    cursors_snapshot = deepcopy(ctx.cursors)      # 记录当前游标位置
    dg_stack_snapshot = deepcopy(dg_stack)          # 记录当前容器栈
    initialized_active_script_hash = ...            # 记录脚本 hash
    ↓ 生成 fragment_id
    fragment_id = calc_hash(module + name + path)
    ↓ 注册闭包
    ctx.fragment_storage.register(fragment_id, wrapped_fragment, parent=...)
    ↓ 首次执行
    wrapped_fragment()
        ctx.fragment_ids_this_run 为 None → 不恢复快照
        with st.container():                        # 创建边界容器
            ThreadState.update(delta_path=...)      # 设置边界路径
            result = user_func(*args, **kwargs)     # 执行用户代码
        ↓ 每个 st.* 调用
        enqueue_message() → msg.delta.fragment_id = fragment_id  # 打标签
```

### 4.2 用户交互触发 fragment 重跑

```
前端：用户点击 fragment 内的按钮
    ↓
BackMsg(fragment_id="abc123") → AppSession
    ↓
AppSession.request_rerun(RerunData(fragment_id="abc123"))
    ↓
ScriptRequests.request_rerun()
    fragment_id → fragment_id_queue=["abc123"]
    ↓
ScriptRunner._run_script 取出 rerun_data
    ↓
order_fragment_ids(["abc123"]) → ["abc123"]
    ↓
ctx.reset(fragment_ids_this_run=["abc123"])
    cursors = {}    ← 游标被清空！
    ↓
进入 if fragment_ids_this_run: 分支
    ↓
wrapped_fragment = storage.lookup("abc123")
    ↓
wrapped_fragment()
    ctx.fragment_ids_this_run 非空 → 恢复快照！
    ctx.cursors = deepcopy(cursors_snapshot)        # 游标回到定义时位置
    context_dg_stack.set(deepcopy(dg_stack_snapshot))  # 栈回到定义时状态
    ↓
    with st.container():                            # 容器路径与首次完全一致
        ThreadState.update(delta_path=...)          # 边界路径与首次完全一致
        result = user_func(*args, **kwargs)
    ↓ 每个 st.* 调用
    delta_path 从恢复的游标位置推进 → 与首次路径完全一致
    enqueue_message() → msg.delta.fragment_id = "abc123"
    ↓
前端收到带 fragment_id 的 delta → 只替换该容器的子树
```

### 4.3 如果没有状态继承会怎样

假设去掉快照恢复步骤：

```
ctx.reset() → cursors = {} → 游标全部归零
    ↓
wrapped_fragment() 不恢复快照
    ↓
st.container() 从 (0, 0) 开始 → 跑到页面左上角
    ↓
fragment 内的 st.button("click me") 的 delta_path = (0, 0, 0, 0)
    与首次的 (0, 3, 0, 0) 完全不同
    ↓
前端在 (0, 0, 0, 0) 位置渲染按钮 → 页面布局完全错乱
```

### 4.4 如果没有边界标记会怎样

假设去掉 `fragment_id` 标签：

```
wrapped_fragment() 执行正常，delta_path 也正确
    ↓
enqueue_message() 不打 fragment_id 标签
    ↓
前端不知道这些 delta 属于哪个 fragment → 当作全局消息处理
    ↓
全局替换整个页面的 UI 树 → 页面闪烁，非 fragment 部分丢失
```

### 4.5 如果没有调度队列会怎样

假设 ScriptRunner 不区分 fragment 重跑和全局重跑：

```
用户点击 fragment 内按钮
    ↓
ScriptRunner 执行 exec(code) → 跑整个脚本
    ↓
所有 fragment 外的代码也重跑 → 性能退化为全局重跑
    ↓
fragment 的存在失去意义
```

---

## 五、平行 fragment 的特殊边界

`parallel=True` 的 fragment 在全局首次运行时会被派发到独立线程：

> 代码定位：[_dispatch_parallel_fragment](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L679-L713)

```python
def _dispatch_parallel_fragment(ctx, fragment_id, wrapped_fragment):
    coordinator = ctx.parallel_coordinator
    with st.container():
        dg_stack_with_container = deepcopy(context_dg_stack.get())
    coordinator.submit(
        _run_parallel_fragment,
        fragment_id,
        wrapped_fragment,
        dg_stack_with_container,
    )
```

关键设计：
- **主线程预分配容器**：`st.container()` 在主线程执行，确保前端立即看到容器占位
- **工作线程设置跳过标记**：`pre_allocated_container_fragment_id=fragment_id`，让 `wrapped_fragment` 知道容器已创建，跳过 `st.container()` 调用
- **工作线程的写入范围受边界约束**：`is_parallel_worker=True`，任何写到边界外的操作都会被 `_is_inside_fragment_path` 拦截

> 代码定位：[_run_parallel_fragment](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L716-L754)

```python
def _run_parallel_fragment(fragment_id, wrapped_fragment, dg_stack_snapshot):
    context_dg_stack.set(dg_stack_snapshot)
    ThreadState.update(
        pre_allocated_container_fragment_id=fragment_id,
        is_parallel_worker=True,
    )
    try:
        wrapped_fragment()
    except RerunException as e:
        coordinator.request_rerun(e)
    except StopException:
        coordinator.request_stop()
    except FragmentHandledException:
        return
```

平行 fragment 重跑时仍然是**顺序执行**的——只有在全局首次运行时才并行。这是因为 fragment 重跑时需要恢复 `ctx.cursors` 等共享状态，并发操作会引发竞争。

---

## 六、总结：三套机制的协作契约

| 机制 | 负责什么 | 失效后果 | 关键代码位置 |
|------|---------|---------|-------------|
| 片段边界 | 划定"fragment 的地盘在哪里"，前端据此做局部替换 | 元素渲染到错误位置或全局替换导致闪烁 | [fragment.py#L420-L441](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L420-L441)、[script_run_context.py#L475-L478](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L475-L478) |
| 状态继承 | 让 fragment 重跑时"回到原点"，游标和栈与首次一致 | delta_path 错乱，元素堆叠到页面顶部 | [fragment.py#L366-L394](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L366-L394) |
| 执行调度 | 决定跑哪个函数、跑的顺序、是否抢占全局脚本 | fragment 重跑退化为全局重跑或执行顺序错误 | [script_requests.py#L86-L97](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L86-L97)、[script_runner.py#L614-L626](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L614-L626) |

三者的协作契约是：
1. **调度**把 `fragment_id_queue` 交给上下文，决定"跑谁"
2. **状态继承**在闭包执行前恢复快照，决定"在哪个世界里跑"
3. **边界**在 delta 产出时打标签，决定"跑出来的结果送到前端哪里"

任何一环缺失，fragment 重跑都无法正确工作。

---

## 七、外部 Widget 状态：stale 判定与保留条件

前面讲了 fragment 内部的机制，但最容易困惑的问题是：**fragment 重跑时，fragment 外面的 widget 状态会不会丢？**

答案是：**不会丢，因为它们被标记为"非 stale"而被保留。** 这依赖于三层机制的精密配合。

### 7.1 前提：每个 Widget 都携带 fragment_id 元数据

每个 widget 在注册时，其 [WidgetMetadata](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/state/common.py#L146) 中都会记录 `fragment_id` 字段：

```python
class WidgetMetadata(Generic[T]):
    id: str
    deserializer: WidgetDeserializer[T] = ...
    serializer: WidgetSerializer[T] = ...
    value_type: ValueFieldName
    callback: WidgetCallback | None = None
    ...
    fragment_id: str | None = None   # ← 归属标记
```

这个 `fragment_id` 在 widget 注册时从 `ThreadState` 中捕获：

> 代码定位：[widgets.py 中的 fragment_id 捕获](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/state/widgets.py#L169)

```python
fragment_id=ThreadState.get().fragment_id if ctx else None,
```

规则很简单：
- 在 fragment 内部注册的 widget，`fragment_id` 等于该 fragment 的 ID
- 在全局脚本中注册的 widget（fragment 外面），`fragment_id` 为 `None`

### 7.2 stale 判定的核心逻辑：`_is_stale_widget`

每次脚本（或 fragment）执行完毕后，`SessionState.on_script_finished()` 会被调用，它内部会调用 `_remove_stale_widgets()` 清理不再需要的 widget 状态。

stale 判定的核心函数是 [_is_stale_widget](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/state/session_state.py#L1325-L1339)：

```python
def _is_stale_widget(
    metadata: WidgetMetadata[Any] | None,
    active_widget_ids: frozenset[str],
    fragment_ids_this_run: list[str] | None,
) -> bool:
    if not metadata:
        return True

    # 如果正在运行一个或多个 fragment，但这个 widget 与正在运行的任何
    # fragment 都无关，则不应标记为 stale，因为它的值可能在未来的
    # fragment 运行或全局脚本运行中仍需使用。
    return not (
        metadata.id in active_widget_ids
        or (fragment_ids_this_run and metadata.fragment_id not in fragment_ids_this_run)
    )
```

这个函数的返回值取 `not (A or B)` 的反逻辑，拆解成**保留条件**更易读：

| 保留条件（使 widget **不**被清理） | 含义 |
|-----------------------------------|------|
| `metadata.id in active_widget_ids` | widget 在本次运行中被渲染了 |
| `fragment_ids_this_run and metadata.fragment_id not in fragment_ids_this_run` | 正在做 fragment 重跑，且 widget 不属于本次运行的任何 fragment |

**只有当两个条件都不满足时，widget 才会被标记为 stale 并清理。**

### 7.3 四种场景的判定结果

假设我们有三个 widget：
- `W_global`：在全局脚本中注册，`fragment_id = None`
- `W_fragA`：在 fragment A 中注册，`fragment_id = "fragA"`
- `W_fragB`：在 fragment B 中注册，`fragment_id = "fragB"`

现在看四种运行场景：

#### 场景 1：全局脚本重跑（`fragment_ids_this_run = None`）
此时第二个条件为 `False`，只有"本次运行中被渲染"的 widget 会被保留：

```
_is_stale_widget(W_global) = not (W_global in active_ids or False)
  → 如果 W_global 被渲染 → 保留
  → 如果 W_global 因条件分支没被渲染 → stale，清理

_is_stale_widget(W_fragA) = not (W_fragA in active_ids or False)
  → 如果 fragment A 被调用 → W_fragA 被渲染 → 保留
  → 如果 fragment A 没被调用 → stale，清理
```

这符合直觉：全局重跑时，所有没被渲染的 widget 都是 stale 的。

#### 场景 2：只重跑 fragment A（`fragment_ids_this_run = ["fragA"]`）
第二个条件为 `True` 当且仅当 widget 不属于 fragA：

```
_is_stale_widget(W_global) = not (W_global in {} or True)
                          = not (False or True) = not True = False
  → 不 stale，保留！

_is_stale_widget(W_fragA) = not (W_fragA in active_ids or False)
  → 如果 W_fragA 在 fragA 中被渲染 → 保留
  → 如果 W_fragA 因条件分支没被渲染 → stale，清理

_is_stale_widget(W_fragB) = not (W_fragB in {} or True)
                          = not True = False
  → 不 stale，保留！
```

**关键结论**：fragment 重跑时，所有外部 widget（全局的和属于其他 fragment 的）都会被无条件保留，无论它们在本次运行中是否被"渲染"（实际上 fragment 重跑根本不会执行外部代码）。

#### 场景 3：同时重跑 fragment A 和 B（`fragment_ids_this_run = ["fragA", "fragB"]`）

```
_is_stale_widget(W_global) = not (False or True) = False → 保留

_is_stale_widget(W_fragA) = not (W_fragA in active_ids or False)
  → 取决于是否在 fragA 中渲染

_is_stale_widget(W_fragB) = not (W_fragB in active_ids or False)
  → 取决于是否在 fragB 中渲染
```

属于本次运行 fragment 的 widget 才需要检查是否被渲染；外部的全部保留。

#### 场景 4：嵌套 fragment 重跑（`fragment_ids_this_run = ["fragParent", "fragChild"]`）

逻辑与场景 3 一致：属于 parent 或 child 的 widget 需要检查渲染状态；全局的和属于其他 fragment 的全部保留。

### 7.4 清理流程的完整链路

```
ScriptRunner._run_script 执行完毕（无论 fragment 还是全局）
    ↓
ScriptRunner._on_script_finished()
    ↓
_session_state.on_script_finished(widget_ids_this_run.snapshot())   ← [script_runner.py#L868]
    ↓
self._remove_stale_widgets(active_widget_ids)                        ← [session_state.py#L878]
    ↓
    1. 捕获 query-param-bound 的 stale widget 的当前值（防止丢失）
    2. self._new_widget_state.remove_stale_widgets(active, frag_ids)  ← 清理新状态
    3. self._old_state = {过滤掉 stale 的}                             ← 清理旧状态
    4. 重新添加被保留的 query param bound 值
    5. self.query_params.remove_stale_bindings(active, frag_ids, meta) ← 清理 query param 绑定
```

> 代码定位：[_remove_stale_widgets 完整实现](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/state/session_state.py#L906-L972)

### 7.5 Query-param-bound Widget 的特殊处理

对于绑定了 query params 的 widget（通过 `bind="query-params"` 或 `st.params`），`_remove_stale_widgets` 有一个特殊保护步骤：

```python
# 在清理前，先捕获 query-param-bound 的 stale widget 的值
bound_preserved: dict[str, Any] = {}
for key in self._old_state:
    if (
        is_element_id(key)
        and key in self._query_param_bound_widget_ids
        and key in wid_key_map
        and _is_stale_widget(meta, active_widget_ids, ctx.fragment_ids_this_run)
    ):
        user_key = wid_key_map[key]
        bound_preserved[user_key] = self._getitem(key, user_key)

# ... 执行清理 ...

# 重新添加到 _old_state，用 user_key 而不是 element_id
self._old_state.update(bound_preserved)
```

这确保了即使 widget 被标记为 stale（例如 fragment 被移除了），它绑定的 query param 值也不会丢失，而是被"转移"到 `st.session_state` 的 user key 下。

---

## 八、Query Params 的保留逻辑

与 widget 状态类似，query params 在 fragment 重跑时也遵循"**只清理本次运行 fragment 内的 stale 绑定**"原则。

### 8.1 保留条件：`remove_stale_bindings`

> 代码定位：[remove_stale_bindings 实现](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/state/query_params.py#L779-L831)

```python
def remove_stale_bindings(
    self,
    active_widget_ids: frozenset[str],
    fragment_ids_this_run: list[str] | None = None,
    widget_metadata: dict[str, Any] | None = None,
) -> None:
    stale_widget_ids = []
    for widget_id in self._bindings_by_widget:
        if widget_id in active_widget_ids:
            continue  # 活跃的 widget，保留

        # fragment 重跑场景：保留不属于本次运行 fragment 的 widget
        if fragment_ids_this_run and widget_metadata:
            metadata = widget_metadata.get(widget_id)
            if metadata and metadata.fragment_id not in fragment_ids_this_run:
                continue  # 外部 widget，保留

        stale_widget_ids.append(widget_id)  # 标记为 stale

    # 清理 stale 的绑定和 URL 参数
    for widget_id in stale_widget_ids:
        binding = self._bindings_by_widget.get(widget_id)
        if binding:
            param_key = binding.param_key
            if param_key in self._query_params:
                del self._query_params[param_key]
        self.unbind_widget(widget_id)
```

与 `_is_stale_widget` 的判定逻辑**完全一致**：
- 活跃的 widget → 保留
- fragment 重跑时，不属于本次运行 fragment 的 widget → 保留
- 其他情况 → stale，清理绑定和 URL 参数

### 8.2 两种运行模式的对照

| 场景 | 清理范围 | 保留范围 |
|------|---------|---------|
| 全局重跑（`fragment_ids_this_run=None`） | 所有未被渲染的 widget 的绑定 | 本次渲染的 widget 的绑定 |
| Fragment 重跑（`fragment_ids_this_run=["fragA"]`） | 仅 fragA 内未被渲染的 widget 的绑定 | 全局 widget 的绑定 + 其他 fragment 的绑定 + fragA 内被渲染的绑定 |

### 8.3 MPA 页面导航的特殊处理

注意文档中的注释：

> Note: Page-based cleanup for MPA navigation is handled separately via
> `populate_from_query_string()` which is called before the script runs.

多页应用的页面切换时，query params 的过滤在 `populate_from_query_string()` 中完成（根据 `page_script_hash` 过滤），这发生在脚本运行之前，与 fragment 逻辑是两条独立的路径。

---

## 九、消息队列（ForwardMsgQueue）的局部清理规则

最后一块拼图是：**fragment 重跑时，哪些消息会被清理？哪些会被保留？**

### 9.1 清理时机：`SCRIPT_STARTED` 事件

每次脚本（或 fragment）开始运行前，`AppSession` 在处理 `SCRIPT_STARTED` 事件时会调用 `_clear_queue()`：

> 代码定位：[SCRIPT_STARTED 事件处理](file:///d:/fz/0601\solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/app_session.py#L665-L686)

```python
if event == ScriptRunnerEvent.SCRIPT_STARTED:
    ...
    self._clear_queue(fragment_ids_this_run)
    msg = self._create_new_session_message(page_script_hash, fragment_ids_this_run, pages)
    self._enqueue_forward_msg(msg)
```

### 9.2 保留条件：`ForwardMsgQueue.clear()`

> 代码定位：[ForwardMsgQueue.clear() 实现](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/forward_msg_queue.py#L110-L162)

```python
def clear(
    self,
    retain_lifecycle_msgs: bool = False,
    fragment_ids_this_run: list[str] | None = None,
) -> None:
    if not retain_lifecycle_msgs:
        self._queue = []
    else:
        self._queue = [
            _update_script_finished_message(msg, fragment_ids_this_run is not None)
            for msg in self._queue
            if msg.WhichOneof("type") in {
                "new_session", "script_finished",
                "session_status_changed", "parent_message",
                "page_info_changed",
            }
            or (
                # fragment 重跑场景，保留以下消息：
                fragment_ids_this_run is not None
                and (
                    # 1. 非 delta 消息（与 fragment 无关）
                    msg.delta is None
                    # 2. 是 delta 但不属于本次运行的任何 fragment
                    or (
                        msg.delta is not None
                        and (
                            msg.delta.fragment_id is None
                            or msg.delta.fragment_id not in fragment_ids_this_run
                        )
                    )
                )
            )
        ]
```

`retain_lifecycle_msgs` 在 `_clear_queue()` 中总是传 `True`：

```python
def _clear_queue(self, fragment_ids_this_run: list[str] | None = None) -> None:
    self._browser_queue.clear(
        retain_lifecycle_msgs=True, fragment_ids_this_run=fragment_ids_this_run
    )
```

### 9.3 保留规则的三层过滤

队列中的每条消息会经过三层判定，只要满足任意一层就被保留：

```
消息 msg 会被保留吗？
    ↓
第 1 层：是生命周期消息吗？（new_session、script_finished 等 5 种）
    → 是 → 保留
    → 否 → 继续
    ↓
第 2 层：是 fragment 重跑吗？（fragment_ids_this_run is not None）
    → 否 → 不保留（清空）
    → 是 → 继续
    ↓
第 3 层：是 fragment 重跑，检查 delta 归属：
    a. msg.delta is None → 不是 delta 消息 → 保留
    b. msg.delta.fragment_id is None → 属于全局脚本 → 保留
    c. msg.delta.fragment_id not in fragment_ids_this_run → 属于其他 fragment → 保留
    d. 否则 → 属于本次运行的 fragment → 不保留（清理）
```

**一句话总结**：fragment 重跑时，**只清理即将被重新渲染的那些 fragment 的旧 delta 消息**，其余全部保留。

### 9.4 `script_finished` 消息的特殊处理

在保留消息时，`_update_script_finished_message()` 会修改 `script_finished` 消息的状态：

> 代码定位：[_update_script_finished_message](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/forward_msg_queue.py#L236-L263)

```python
def _update_script_finished_message(msg: ForwardMsg, is_fragment_run: bool) -> ForwardMsg:
    if msg.WhichOneof("type") == "script_finished" and (
        is_fragment_run is False
        or msg.script_finished != ForwardMsg.ScriptFinishedStatus.FINISHED_SUCCESSFULLY
    ):
        msg.script_finished = ForwardMsg.ScriptFinishedStatus.FINISHED_EARLY_FOR_RERUN
    return msg
```

关键规则：
- **全局重跑被打断**：把之前的 `script_finished` 改成 `FINISHED_EARLY_FOR_RERUN`，告诉前端"之前的运行被打断了，不要重置 widget 状态"
- **fragment 重跑**：**不修改** `FINISHED_SUCCESSFULLY` 状态——因为这是全局脚本成功完成的标记，fragment 无权修改全局状态。前端需要这个标记来知道"上一次全局重跑成功了，可以正常清理"

### 9.5 两种清理模式的对照

| 维度 | 全局重跑的清理 | Fragment 重跑的清理 |
|------|---------------|-------------------|
| 生命周期消息 | 保留 | 保留 |
| 全局 delta（`fragment_id=None`） | 清理（被新全局 delta 替代） | **保留**（不会被重新渲染） |
| 其他 fragment 的 delta | 清理（被新全局脚本重新生成） | **保留**（不会被重新渲染） |
| 本次运行 fragment 的 delta | 清理 | 清理（被新 fragment delta 替代） |
| script_finished 状态 | 改为 FINISHED_EARLY_FOR_RERUN | 保留 FINISHED_SUCCESSFULLY（如果有的话） |

---

## 十、四套外部状态的协同总览

现在把 fragment 内部的三套机制（边界、状态继承、调度）与外部的三套保留机制（widget 状态、query params、消息队列）放在一起，形成完整图景：

```
┌─────────────────────────────────────────────────────────────┐
│                 fragment 重跑触发                            │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                执行调度（Execution Scheduling）               │
│  • fragment_id_queue 决定跑哪些 fragment                       │
│  • 祖先优先排序保证父 fragment 先于子 fragment 执行            │
│  • fragment 重跑不抢占全局脚本（scope="fragment" 除外）        │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                状态继承（State Inheritance）                 │
│  • 恢复 cursors_snapshot → 游标位置与定义时一致               │
│  • 恢复 dg_stack_snapshot → 容器嵌套与定义时一致              │
│  • 冻结 active_script_hash → Widget ID 跨运行一致             │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                片段边界（Fragment Boundary）                 │
│  • delta_path 前缀划定 fragment 的作用域                     │
│  • 平行 fragment 用 _is_inside_fragment_path 禁止外写        │
│  • 每个 delta 打上 fragment_id 标签 → 前端局部替换           │
└─────────────────────────────┬───────────────────────────────┘
                              │
         ┌────────────────────┴────────────────────┐
         │                                         │
┌────────▼─────────┐                    ┌──────────▼──────────┐
│  外部状态保留逻辑 │                    │  内部状态清理逻辑   │
└──────────────────┘                    └─────────────────────┘
         │                                         │
┌────────▼─────────────────────────────────────────▼──────────┐
│                    清理总时机：on_script_finished            │
│  在 SCRIPT_STARTED 时清理消息队列，在脚本结束时清理 widget     │
└─────────────────────────────┬───────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
┌────────▼───────┐  ┌──────────▼─────────┐  ┌───────▼──────────┐
│ Widget 状态    │  │ Query Params 绑定  │  │ 消息队列 Delta   │
│ _is_stale_widget│  │ remove_stale_     │  │ ForwardMsgQueue. │
│ 判定逻辑       │  │ bindings           │  │ clear 保留逻辑   │
│                │  │                    │  │                  │
│ 保留：         │  │ 保留：            │  │ 保留：           │
│ • 活跃的       │  │ • 活跃的          │  │ • 生命周期消息    │
│ • 外部的       │  │ • 外部的          │  │ • 全局 delta      │
│                │  │                    │  │ • 其他 fragment  │
│ 清理：         │  │ 清理：            │  │   的 delta       │
│ • 本 fragment  │  │ • 本 fragment     │  │                  │
│   内未渲染的   │  │   内未渲染的       │  │ 清理：           │
│                │  │                    │  │ • 本 fragment    │
│                │  │                    │  │   的旧 delta     │
└────────────────┘  └────────────────────┘  └──────────────────┘
```

### 10.1 一个贯穿全局的判定原则

所有外部状态（widget、query params、消息队列）的保留/清理逻辑，都遵循同一个判定模式：

```python
if 是全局重跑:
    只保留本次活跃的
elif 是 fragment 重跑:
    保留所有外部的（不属于本次运行 fragment 的）
    对于本次运行 fragment 内部的，只保留活跃的
```

这个模式由 `fragment_ids_this_run` 这个"全局开关"统一控制，贯穿 widget state、query params、forward msg queue 三套子系统。

### 10.2 三类 stale 的对照表

| 状态类型 | 判定函数 | 核心保留条件 |
|---------|---------|-------------|
| Widget 状态 | `_is_stale_widget()` | `not (活跃 or (fragment_run and 外部))` → 取反 |
| Query Params 绑定 | `remove_stale_bindings()` 内联逻辑 | 活跃 or (fragment_run and 外部) → 直接判定 |
| 消息队列 Delta | `ForwardMsgQueue.clear()` 内联逻辑 | 生命周期 or (fragment_run and (非delta or 外部)) |

### 10.3 常见疑问解答

**Q: fragment 重跑时，全局的 `st.checkbox` 状态会丢吗？**
A: 不会。它的 `metadata.fragment_id = None`，不属于任何 fragment，在 `_is_stale_widget` 中命中第二个保留条件，被保留。

**Q: fragment A 重跑时，fragment B 内的 widget 状态会丢吗？**
A: 不会。B 的 `fragment_id` 不在 `fragment_ids_this_run` 中，命中第二个保留条件，被保留。

**Q: fragment 重跑时，全局的 `st.write("hello")` 会被清理吗？**
A: 不会。它的 delta 消息 `fragment_id = None`，在 `ForwardMsgQueue.clear()` 中被保留。前端不会替换它。

**Q: fragment 内条件渲染的 widget 没显示，它的状态会丢吗？**
A: 会。如果它在本次 fragment 重跑中未被渲染（不在 `active_widget_ids` 中），且属于本次运行的 fragment，就会被 `_is_stale_widget` 判定为 stale 并清理。这与全局脚本的行为一致。

**Q: fragment 被从代码中移除了，它的 widget 状态什么时候清理？**
A: 在下一次**全局重跑**时清理。fragment 重跑不会清理其他 fragment 的状态。

---

## 十一、总结：七套机制的完整协作契约

现在将所有机制汇总：

| 机制 | 核心职责 | 关键代码 |
|------|---------|---------|
| **片段边界** | 划定作用域，前端局部替换 | [fragment.py#L420-L441](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L420-L441)、[script_run_context.py#L475-L478](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L475-L478) |
| **状态继承** | 重跑时回到定义时的上下文 | [fragment.py#L366-L394](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L366-L394) |
| **执行调度** | 决定跑谁、跑的顺序、是否抢占 | [script_requests.py#L86-L97](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_requests.py#L86-L97)、[script_runner.py#L614-L626](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L614-L626) |
| **Widget 状态保留** | stale 判定，保留外部状态 | [session_state.py#L1325-L1339](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/state/session_state.py#L1325-L1339) |
| **Query Params 保留** | 保留外部绑定，不随意清 URL | [query_params.py#L779-L831](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/state/query_params.py#L779-L831) |
| **消息队列清理** | 只清理本次 fragment 的旧 delta | [forward_msg_queue.py#L110-L162](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/forward_msg_queue.py#L110-L162) |

贯穿所有机制的核心开关是 `fragment_ids_this_run`：
- 它为 `None` → 全局重跑模式，所有外部状态都可能被清理
- 它非 `None` → fragment 重跑模式，所有外部状态被保护性保留

这个开关从 `RerunData.fragment_id_queue` 一路传递到 `ScriptRunContext`，再到 `SessionState`、`QueryParams`、`ForwardMsgQueue`，像一条红线串起了 Streamlit 整个运行时系统。

---

## 十二、前端渲染树：节点模型与生命周期标记

前文聚焦后端，但 fragment 的局部替换最终要在前端落地。Streamlit 前端维护一棵**不可变渲染树**（Immutable Render Tree），每次 delta 到达都会产生一棵新树，React 通过浅比较只更新变化的节点。

### 12.1 三种节点类型

> 代码定位：[AppNode 接口](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/AppNode.interface.ts#L65-L109)

每个节点都携带两个关键元数据：

| 字段 | 含义 | 来源 |
|------|------|------|
| `scriptRunId` | 产生该节点的脚本运行 ID | 后端 `SCRIPT_STARTED` 时生成，随 `NewSession` 消息下发 |
| `fragmentId` | 产生该节点的 fragment ID | 后端 `delta.fragment_id` 字段 |

三种节点类型：

**BlockNode**（容器节点，对应 `addBlock` delta）

> 代码定位：[BlockNode](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/BlockNode.ts#L29-L57)

```typescript
export class BlockNode implements AppNode {
  readonly children: AppNode[]
  readonly deltaBlock: BlockProto
  readonly scriptRunId: string
  readonly fragmentId?: string
  readonly activeScriptHash: string
}
```

BlockNode 是分支节点，持有 `children` 数组。fragment 的 `st.container()` 对应的就是一个 BlockNode，其 `fragmentId` 等于该 fragment 的 ID。

**ElementNode**（叶子节点，对应 `newElement` delta）

> 代码定位：[ElementNode](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/ElementNode.ts#L28-L58)

```typescript
export class ElementNode implements AppNode {
  readonly element: Element
  readonly metadata: ForwardMsgMetadata
  readonly scriptRunId: string
  readonly fragmentId?: string
  readonly activeScriptHash: string
  readonly elementHash?: string  // 内容哈希，用于 payload 复用
}
```

ElementNode 是叶子节点，承载实际的 UI 元素（按钮、文本、图表等）。

**TransientNode**（临时节点，对应 `newTransient` delta）

> 代码定位：[TransientNode](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/TransientNode.ts#L28-L52)

```typescript
export class TransientNode implements AppNode {
  readonly anchor?: AppNode          // 持久化的"锚点"节点
  readonly transientNodes: ElementNode[]  // 临时元素列表
  readonly scriptRunId: string
  readonly fragmentId: undefined     // 显式为 undefined
}
```

TransientNode 是**传输层包装**，代表"在某个位置短暂出现的元素"。它维护一个 `anchor`（持久节点）和一组 `transientNodes`（临时元素）。典型场景是 `st.toast`、`st.chat_message` 等临时通知。

**关键设计**：TransientNode 的 `fragmentId` 显式设为 `undefined`——临时节点不属于任何 fragment，它们是全局性的。

### 12.2 delta 如何变成节点：`applyDelta`

> 代码定位：[AppRoot.applyDelta](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/AppRoot.ts#L238-L301)

```typescript
public applyDelta(
  scriptRunId: string,
  delta: Delta,
  metadata: ForwardMsgMetadata,
  elementHash?: string
): AppRoot {
  const { deltaPath, activeScriptHash } = metadata
  switch (delta.type) {
    case "newElement": {
      // 可能复用 elementHash 匹配的已有节点 payload
      return this.addElement(
        deltaPath, scriptRunId, element, metadata,
        activeScriptHash, delta.fragmentId, elementHash
      )
    }
    case "addBlock": {
      // 同类型 Block 继承已有 children（保留 Widget/React 状态）
      return this.addBlock(
        deltaPath, block, scriptRunId,
        activeScriptHash, delta.fragmentId, deltaMsgReceivedAt
      )
    }
    case "newTransient": {
      // 创建 TransientNode 包装临时元素
      return this.addTransient(
        deltaPath, scriptRunId, transient,
        metadata, activeScriptHash, delta.fragmentId
      )
    }
  }
}
```

每个 delta 都携带 `delta.fragmentId`，该值被写入创建的节点中。因此**每个节点的 `fragmentId` 在创建时就被永久确定**，后续的 stale 判定完全依赖这个标记。

### 12.3 Block 继承 children：Widget/React 状态保留的关键

> 代码定位：[addBlock 中的 children 继承](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/AppRoot.ts#L443-L503)

```typescript
private addBlock(...): AppRoot {
  // 如果替换同类型 Block，继承已有 children
  let children: AppNode[] = []
  if (
    existingNode instanceof BlockNode &&
    existingNode.deltaBlock.type === block.type
  ) {
    const isDialogWithDifferentIdentity = ...
    if (!isDialogWithDifferentIdentity) {
      children = existingNode.children  // 继承！
    }
  }
  // ...
}
```

这确保了当 fragment 重跑发送新的 `addBlock` delta 时（比如 fragment 容器本身），**已有子节点不会被丢弃**。React 的 key-based 协调机制可以识别这些节点是"同一个"，从而保留 DOM 状态和 Widget 状态。新的 delta 会在后续步骤中替换或追加到这些 children 中。

---

## 十三、前端清理链路：两步走的时序控制

前端的清理发生在两个关键时间点，形成一个"**先清临时、后清 stale**"的两步走流程。

### 13.1 第一步：NewSession 到达 → 清理 Transient 节点

当 `NewSession` 消息到达前端时（对应后端的 `SCRIPT_STARTED` 事件），`App.tsx` 首先清空临时节点：

> 代码定位：[handleNewSession 中的 clearTransientNodes](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/app/src/App.tsx#L1425-L1441)

```typescript
if (appHash === newSessionHash && prevPageScriptHash === newPageScriptHash) {
  this.setState(prevState => ({
    // Clear the transient nodes before executing everything else.
    elements: prevState.elements.clearTransientNodes(fragmentIdsThisRun),
    scriptRunId,
  }))
} else {
  this.clearAppState(...)  // 全局清空（页面切换等）
}
```

**为什么先清 Transient**？因为 TransientNode 里的元素（如 toast）是"上一次运行产生的临时 UI"，在新运行开始前应该先消失，避免与新的临时元素混淆。

**注意时序**：清理 Transient 发生在 delta 开始流入**之前**——此时 `NewSession` 消息已到，但新的 `newElement`/`addBlock` delta 还没到。所以清理是"先打扫再装修"。

### 13.2 第二步：ScriptFinished → 清理 Stale 节点

当脚本运行完成，前端收到 `script_finished` 消息后，执行 stale 清理：

> 代码定位：[handleScriptFinished 中的 clearStaleNodes](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/app/src/App.tsx#L1593-L1632)

```typescript
handleScriptFinished(status: ForwardMsg.ScriptFinishedStatus): void {
  if (
    status === FINISHED_SUCCESSFULLY ||
    status === FINISHED_EARLY_FOR_RERUN ||
    status === FINISHED_FRAGMENT_RUN_SUCCESSFULLY
  ) {
    if (
      status === FINISHED_SUCCESSFULLY ||
      status === FINISHED_FRAGMENT_RUN_SUCCESSFULLY
    ) {
      // 只对成功完成的运行做 stale 清理，不做于 FINISHED_EARLY_FOR_RERUN
      this.setState(({ scriptRunId, fragmentIdsThisRun, elements }) => {
        return {
          elements: elements.clearStaleNodes(scriptRunId, fragmentIdsThisRun),
        }
      }, () => {
        this.removeInactiveWidgetState()
      })
    }
  }
}
```

**关键约束**：
- `FINISHED_EARLY_FOR_RERUN` **不做 stale 清理**——因为这次运行被打断了，新的运行可能还在路上，如果此时清理会导致 UI 闪烁（元素短暂消失再出现）
- `FINISHED_FRAGMENT_RUN_SUCCESSFULLY` **做 stale 清理**——fragment 运行成功了，可以安全清理该 fragment 内的 stale 节点
- 清理完成后调用 `removeInactiveWidgetState()`，通知 `WidgetStateManager` 清理不再存在的 Widget 的前端状态

### 13.3 完整的时序图

```
后端：SCRIPT_STARTED → NewSession 消息
    ↓
前端：handleNewSession()
    ↓
前端：setState({ elements: clearTransientNodes(fragmentIdsThisRun) })
    ↓ ← 此时旧 toast/通知消失，但所有 ElementNode 和 BlockNode 仍在
    ↓
后端：delta 消息流（newElement / addBlock / newTransient）
    ↓
前端：setState({ elements: applyDelta(scriptRunId, delta, ...) })
    ↓ ← 每次 delta 到达都会产生新的 elements 树
    ↓
后端：SCRIPT_FINISHED → script_finished 消息
    ↓
前端：handleScriptFinished()
    ↓
前端：setState({ elements: clearStaleNodes(scriptRunId, fragmentIdsThisRun) })
    ↓ ← 此时 stale 节点被裁剪，只保留本次运行的节点 + 外部节点
    ↓
前端：removeInactiveWidgetState()
    ↓ ← WidgetStateManager 清理不存在的 Widget 前端状态
```

---

## 十四、TransientNode 的局部清理：`ClearTransientNodesVisitor`

### 14.1 清理逻辑

> 代码定位：[ClearTransientNodesVisitor](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/visitors/ClearTransientNodesVisitor.ts#L29-L84)

```typescript
export class ClearTransientNodesVisitor implements AppNodeVisitor<AppNode | undefined> {
  private readonly fragmentIdsThisRun: string[]

  visitBlockNode(node: BlockNode): AppNode | undefined {
    // 关键：不属于本次运行的 fragment 的 Block 不遍历
    if (
      this.fragmentIdsThisRun.length > 0 &&
      node.fragmentId &&
      !this.fragmentIdsThisRun.includes(node.fragmentId)
    ) {
      return node  // 原样返回，不清理
    }
    // 递归清理子节点中的 TransientNode
    // ...
  }

  visitElementNode(node: ElementNode): AppNode | undefined {
    return node  // ElementNode 不是临时节点，不做处理
  }

  visitTransientNode(node: TransientNode): AppNode | undefined {
    return node.anchor  // 丢弃临时元素，只保留锚点
  }
}
```

### 14.2 裁剪范围

| 节点类型 | 全局重跑 | Fragment 重跑 |
|---------|---------|---------------|
| BlockNode（全局，`fragmentId=undefined`） | 递归清理内部 Transient | 递归清理内部 Transient |
| BlockNode（本 fragment，`fragmentId` 在列表中） | N/A | 递归清理内部 Transient |
| BlockNode（其他 fragment，`fragmentId` 不在列表中） | N/A | **直接返回，不遍历** |
| ElementNode | 原样保留 | 原样保留 |
| TransientNode | 丢弃临时元素，保留 anchor | 丢弃临时元素，保留 anchor |

**最关键的设计**：fragment 重跑时，不属于本次运行的 BlockNode **不会被遍历**，这意味着其他 fragment 内的 TransientNode 也**不会被清理**。这保证了 fragment A 的重跑不会影响 fragment B 中的临时通知。

### 14.3 anchor 的保留意义

`visitTransientNode` 返回 `node.anchor`，意味着临时元素被丢弃，但底层的持久元素被保留。例如一个 `st.chat_message` 中有临时 toast：
- 临时 toast 被移除
- `chat_message` 本身（作为 anchor）继续存在

如果 anchor 也为 `undefined`，则整个 TransientNode 消失——该位置变为空。

---

## 十五、Stale 节点的局部清理：`ClearStaleNodeVisitor`

这是前端最核心的清理逻辑，也是最容易困惑的部分。

### 15.1 两个判定维度的配合

每个节点有两个判定维度：
1. **`scriptRunId`**：该节点是在哪次脚本运行中创建的
2. **`fragmentId`**：该节点属于哪个 fragment（全局节点为 `undefined`）

`ClearStaleNodeVisitor` 维护三个状态：

```typescript
class ClearStaleNodeVisitor {
  readonly currentScriptRunId: string     // 当前运行 ID
  readonly fragmentIdsThisRun: string[]   // 本次运行的 fragment ID 列表
  readonly fragmentIdOfBlock?: string     // 当前正在遍历的 Block 的 fragmentId
}
```

`fragmentIdOfBlock` 是一个**向下传递的上下文标记**，表示"当前遍历路径上，最近的一个被本次运行修改的 Block 的 fragmentId"。它的作用是标记"裁剪范围"——只有在这个范围**内部**的节点才可能被判定为 stale。

### 15.2 BlockNode 的三种判定路径

> 代码定位：[visitBlockNode](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/visitors/ClearStaleNodeVisitor.ts#L61-L123)

```typescript
visitBlockNode(node: BlockNode): AppNode | undefined {
  if (!this.isFragmentRun) {
    // 路径 A：全局重跑 → 简单规则
    if (node.scriptRunId !== this.currentScriptRunId) {
      return undefined  // 不是本次运行的 → stale，裁掉
    }
  } else {
    // 路径 B：fragment 重跑 → 更精细的规则

    // B1：父 Block 被修改了，但这个 Block 没被修改 → stale
    if (this.fragmentIdOfBlock && node.scriptRunId !== this.currentScriptRunId) {
      return undefined
    }

    // B2：这个 Block 属于本次运行的 fragment，且被本次运行修改了
    // → 设置 fragmentIdOfBlock，告知子节点"你在一个被修改的 Block 内"
    if (
      node.fragmentId &&
      this.fragmentIdsThisRun.includes(node.fragmentId) &&
      node.scriptRunId === this.currentScriptRunId
    ) {
      clearStaleNodeVisitor = new ClearStaleNodeVisitor(
        this.currentScriptRunId,
        this.fragmentIdsThisRun,
        node.fragmentId  // ← 向下传递！
      )
    }
  }

  // 递归清理 children
  // ...
}
```

三种路径的含义：

**路径 A（全局重跑）**：极其简单——所有 `scriptRunId !== currentScriptRunId` 的 Block 都裁掉。因为全局重跑会重新产生所有节点，旧的都不需要了。

**路径 B1（fragment 重跑 + 父 Block 被修改但子 Block 没有）**：如果父 Block 是本次运行修改的（`fragmentIdOfBlock` 有值），但子 Block 没有被修改（`scriptRunId` 不匹配），说明这个子 Block 是"上次运行留下的残留"，应该被裁掉。

**路径 B2（fragment 重跑 + 本 Block 被本次运行修改）**：如果这个 Block 属于本次运行的 fragment 且被本次运行更新了（`scriptRunId === currentScriptRunId`），就创建一个新的 visitor 实例，把 `fragmentIdOfBlock` 设为该 Block 的 `fragmentId`。这个标记会传递给所有子节点。

### 15.3 ElementNode 的三重保留条件

> 代码定位：[visitElementNode](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/visitors/ClearStaleNodeVisitor.ts#L125-L141)

```typescript
visitElementNode(node: ElementNode): AppNode | undefined {
  if (this.isFragmentRun) {
    // fragment 重跑时，满足以下任一条件就保留：
    if (
      !node.fragmentId ||                    // 条件 1：不属于任何 fragment（全局元素）
      !this.fragmentIdOfBlock ||             // 条件 2：不在被修改的 Block 内
      node.scriptRunId === this.currentScriptRunId  // 条件 3：已被本次运行更新
    ) {
      return node
    }
  }
  // 全局重跑：只保留本次运行产生的元素
  return node.scriptRunId === this.currentScriptRunId ? node : undefined
}
```

拆解 fragment 重跑时的三重保留条件：

| 条件 | 含义 | 典型场景 |
|------|------|---------|
| `!node.fragmentId` | 元素不属于任何 fragment | 全局脚本中的 `st.write("hello")` |
| `!this.fragmentIdOfBlock` | 遍历路径上没有被本次运行修改的 Block | 同一页面但不在 fragment 树内的元素 |
| `node.scriptRunId === currentScriptRunId` | 元素被本次运行更新了 | fragment 重跑后新产生的 `st.button("click me")` |

**只有当三个条件都不满足时，ElementNode 才被判定为 stale**：
- `node.fragmentId` 有值（属于某个 fragment）
- `this.fragmentIdOfBlock` 有值（在本次运行修改的 Block 内）
- `node.scriptRunId !== currentScriptRunId`（没被本次运行更新）

这意味着：**只有"在本次运行修改的 fragment Block 内部、且没有被重新渲染"的元素才是 stale 的**。

### 15.4 TransientNode 的分级降级逻辑

> 代码定位：[visitTransientNode](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/visitors/ClearStaleNodeVisitor.ts#L143-L179)

```typescript
visitTransientNode(node: TransientNode): AppNode | undefined {
  // 特殊情况：transient 在本次运行中被显式清空了
  // 直接恢复 anchor，防止 anchor 在后续步骤中被误判为 stale
  if (
    node.scriptRunId === this.currentScriptRunId &&
    node.transientNodes.length === 0 &&
    node.anchor
  ) {
    return node.anchor.accept(this)
  }

  // 分别检查 anchor 和 transient elements 的 stale 状态
  const anchorNode = node.anchor?.accept(this)
  const transientNodes = node.updateTransientNodes(element => {
    return element.accept(this) as ElementNode | undefined
  })

  // 全部 stale → 整个 TransientNode 消失
  if (!anchorNode && transientNodes.length === 0) {
    return undefined
  }

  // 只有临时元素 stale → 降级为 anchor 节点
  if (transientNodes.length === 0) {
    return anchorNode
  }

  // anchor 和临时元素都保留 → 保留 TransientNode
  return new TransientNode(...)
}
```

三级降级：

| anchor 状态 | 临时元素状态 | 结果 |
|------------|-------------|------|
| stale | stale | `undefined`（整个 TransientNode 消失） |
| 保留 | stale | anchor 节点（TransientNode 降级为普通节点） |
| stale | 保留 | 新 TransientNode（anchor 为空但临时元素仍存在） |
| 保留 | 保留 | 新 TransientNode（两者都保留） |

**特殊优化**：如果 TransientNode 在本次运行中被显式清空（`transientNodes.length === 0`），直接用 anchor 替换自身，然后递归检查 anchor 是否 stale。这防止了"anchor 被误判为 stale"的问题——因为如果 anchor 也是本次运行产生的，它的 `scriptRunId` 匹配，就不会被误裁。

### 15.5 完整的判定流程图

```
ClearStaleNodeVisitor 遍历渲染树
    ↓
遇到 BlockNode
    ↓
是全局重跑吗？
    ├─ 是 → scriptRunId 匹配？
    │        ├─ 匹配 → 递归子节点
    │        └─ 不匹配 → 裁掉（返回 undefined）
    │
    └─ 否（fragment 重跑）→
         ├─ fragmentIdOfBlock 有值 且 scriptRunId 不匹配？
         │    └─ 是 → 裁掉（父 Block 被修改，此 Block 是残留）
         │
         ├─ 本 Block 属于本次运行的 fragment 且被本次运行更新？
         │    └─ 是 → 设 fragmentIdOfBlock = node.fragmentId
         │            递归子节点（使用新 visitor 实例）
         │
         └─ 否 → 递归子节点（使用当前 visitor 实例）
    ↓
遇到 ElementNode
    ↓
是全局重跑吗？
    ├─ 是 → scriptRunId 匹配？匹配→保留，不匹配→裁掉
    └─ 否（fragment 重跑）→
         满足任一保留条件？
         ├─ !node.fragmentId → 保留（全局元素）
         ├─ !fragmentIdOfBlock → 保留（不在修改范围内）
         └─ scriptRunId 匹配 → 保留（被本次运行更新）
         否则 → 裁掉
    ↓
遇到 TransientNode
    ↓
分别对 anchor 和 transientElements 递归判定
    ├─ 全部 stale → 消失
    ├─ anchor 保留、临时元素 stale → 降级为 anchor
    └─ 至少有临时元素保留 → 保留 TransientNode
```

---

## 十六、渲染期实时 stale 感知：`isElementStale`

上面的 `ClearStaleNodeVisitor` 在脚本结束时**一次性清理**所有 stale 节点。但还有另一个问题：**在脚本运行期间（delta 还在流入时），已经旧的元素如何处理？**

答案在 [isElementStale](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/components/core/Block/utils.ts#L43-L70)：

```typescript
export function isElementStale(
  node: AppNode,
  scriptRunState: ScriptRunState,
  scriptRunId: string,
  fragmentIdsThisRun?: Array<string>
): boolean {
  if (scriptRunState === ScriptRunState.RERUN_REQUESTED) {
    // 重跑刚被请求 → 所有元素即将变 stale
    return true
  }

  if (scriptRunState === ScriptRunState.RUNNING) {
    if (fragmentIdsThisRun?.length) {
      // fragment 重跑 → 只有属于本 fragment 且未更新的元素才是 stale
      return Boolean(
        node.fragmentId &&
        fragmentIdsThisRun.includes(node.fragmentId) &&
        node.scriptRunId !== scriptRunId
      )
    }
    // 全局重跑 → 所有未更新的元素都是 stale
    return node.scriptRunId !== scriptRunId
  }

  return false
}
```

这个函数在**每个 React 组件渲染时**被调用，用于决定元素是否应该以"禁用/灰色"样式显示。与 `ClearStaleNodeVisitor` 的判定逻辑一脉相承，但作用不同：

| 机制 | 时机 | 作用 | 结果 |
|------|------|------|------|
| `isElementStale` | 脚本运行**期间** | 实时感知 stale，UI 上禁用旧元素 | 元素变灰但仍在 DOM 中 |
| `ClearStaleNodeVisitor` | 脚本运行**结束** | 从渲染树中移除 stale 节点 | 元素从 DOM 中移除 |

**`RERUN_REQUESTED` 的特殊处理**：当用户点击"Rerun"按钮后、新脚本还没开始运行之前（`scriptRunState === RERUN_REQUESTED`），**所有元素都标记为 stale**。这是为了防止用户在等待期间与旧元素交互，导致意外行为。但 fragment 重跑时不会这样——只有该 fragment 内的旧元素被标记为 stale。

---

## 十七、Logo 的 stale 保留：`clearStaleNodes` 的额外逻辑

> 代码定位：[clearStaleNodes 中的 logo 保留](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/AppRoot.ts#L340-L369)

```typescript
public clearStaleNodes(
  currentScriptRunId: string,
  fragmentIdsThisRun?: Array<string>
): AppRoot {
  // ... 应用 ClearStaleNodeVisitor ...

  // fragment 重跑时保留 logo，防止闪烁（Issue #10350/#10382）
  const isFragmentRun = fragmentIdsThisRun && fragmentIdsThisRun.length > 0
  const appLogo =
    isFragmentRun || this.appLogo?.scriptRunId === currentScriptRunId
      ? this.appLogo
      : null

  return new AppRoot(this.mainScriptHash, ..., appLogo)
}
```

Logo 不在渲染树内部，而是 `AppRoot` 的独立属性。fragment 重跑时，即使 logo 的 `scriptRunId` 不匹配当前运行，也会被保留。这是为了避免 fragment 重跑导致 logo 短暂消失再出现的闪烁问题。

---

## 十八、端到端示例：Fragment 重跑的前端树演变

假设页面结构如下：

```
AppRoot
├── main
│   ├── BlockNode (global, scriptRunId="run1")
│   │   └── ElementNode: st.write("Header") [fragmentId=undefined, scriptRunId="run1"]
│   ├── BlockNode (fragA, scriptRunId="run1", fragmentId="fragA")
│   │   └── ElementNode: st.button("Click") [fragmentId="fragA", scriptRunId="run1"]
│   └── TransientNode (scriptRunId="run1")
│       ├── anchor: ElementNode: st.text("Status") [fragmentId=undefined, scriptRunId="run1"]
│       └── transient: ElementNode: st.toast("Saved!") [fragmentId=undefined, scriptRunId="run1"]
└── sidebar
    └── ElementNode: st.checkbox("Debug") [fragmentId=undefined, scriptRunId="run1"]
```

### 阶段 1：NewSession 到达（fragment 重跑 fragA，scriptRunId="run2"）

```
clearTransientNodes(fragmentIdsThisRun=["fragA"])

├── main BlockNode (global) → 递归
│   ├── global BlockNode → 递归
│   │   └── ElementNode → 保留
│   ├── fragA BlockNode → fragmentId 在列表中 → 递归
│   │   └── ElementNode → 保留
│   └── TransientNode → 返回 anchor
│       └── 结果：ElementNode: st.text("Status")
└── sidebar → 递归
    └── ElementNode → 保留

结果：toast 消失，其他不变
```

### 阶段 2：Delta 流入

```
applyDelta("run2", delta_addBlock_fragA, ...)  → fragA BlockNode 更新，scriptRunId="run2"
applyDelta("run2", delta_newElement_button, ...) → fragA 内的 button 被替换

此时的树（新旧混杂）：
├── main
│   ├── BlockNode (global, scriptRunId="run1")           ← 旧的
│   │   └── ElementNode: st.write("Header") [run1]       ← 旧的
│   ├── BlockNode (fragA, scriptRunId="run2", fragmentId="fragA")  ← 新的
│   │   └── ElementNode: st.button("Clicked!") [fragA, run2]       ← 新的
│   └── ElementNode: st.text("Status") [run1]             ← 旧的（从 Transient 降级来的）
└── sidebar
    └── ElementNode: st.checkbox("Debug") [run1]          ← 旧的
```

注意：此时**旧的节点仍然存在**，React 会根据 `isElementStale` 把旧节点显示为灰色/禁用状态。

### 阶段 3：ScriptFinished → clearStaleNodes

```
clearStaleNodes("run2", ["fragA"])

├── main BlockNode → 全局重跑？否 → fragmentIdOfBlock=undefined
│   ├── global BlockNode → scriptRunId="run1" ≠ "run2"
│   │   → fragmentIdOfBlock 有值？否（undefined）
│   │   → 不裁（路径 B1 不触发）→ 递归
│   │   └── ElementNode: Header → !node.fragmentId → 保留（条件 1）
│   │
│   ├── fragA BlockNode → scriptRunId="run2" = "run2"，fragmentId="fragA" 在列表中
│   │   → 创建新 visitor(fragmentIdOfBlock="fragA") → 递归
│   │   └── ElementNode: button → scriptRunId="run2" = "run2" → 保留（条件 3）
│   │
│   └── ElementNode: Status → !node.fragmentId → 保留（条件 1）
│
└── sidebar → 递归
    └── ElementNode: checkbox → !node.fragmentId → 保留（条件 1）

结果：所有节点都保留了！因为 fragA 内的 button 已被更新，fragA 外的元素不在修改范围内。
```

**如果 fragA 重跑后 button 消失了**（比如条件分支变了）：

```
clearStaleNodes("run2", ["fragA"])

├── fragA BlockNode → 创建新 visitor(fragmentIdOfBlock="fragA") → 递归
│   └── （没有 button 的 ElementNode 了，因为 fragA 重跑没有渲染它）
│       → fragA BlockNode 的 children 为空
│       → 不需要裁任何 stale 元素，因为 stale 的 button 本来就不在新的 children 里了
```

**如果嵌套子 fragment（fragChild）在重跑时没有重新注册**：

```
假设 fragA 重跑时，fragChild 的装饰器函数没被调用
→ 后端 FragmentStorage.clear_stale_descendants(fragA, registered_ids) 清理 fragChild
→ 前端不再收到 fragChild 的 delta

clearStaleNodes("run2", ["fragA"])
├── fragA BlockNode → fragmentIdOfBlock="fragA"
│   └── fragChild BlockNode → scriptRunId="run1" ≠ "run2"
│       → fragmentIdOfBlock="fragA" 有值
│       → 路径 B1：父 Block 被修改，此 Block 没被修改 → 裁掉！
│       → 返回 undefined

结果：fragChild 整棵子树被裁掉，fragA 中只剩本次运行产生的节点。
```

---

## 十九、前后端 stale 判定的统一范式

后端的 `_is_stale_widget`、`remove_stale_bindings`、`ForwardMsgQueue.clear` 和前端的 `ClearStaleNodeVisitor`、`isElementStale` 虽然实现语言不同，但遵循**完全统一的判定范式**：

```
if 全局重跑:
    不属于本次运行 → stale
elif fragment 重跑:
    if 不属于本次运行的 fragment → 保留
    elif 属于本次运行的 fragment 且被更新 → 保留
    else（属于本次运行的 fragment 但没被更新）→ stale
```

| 层 | 判定机制 | "不属于本次 fragment" | "被本次运行更新" |
|----|---------|---------------------|----------------|
| 后端 Widget | `_is_stale_widget` | `metadata.fragment_id not in fragment_ids_this_run` | `metadata.id in active_widget_ids` |
| 后端 QueryParam | `remove_stale_bindings` | `metadata.fragment_id not in fragment_ids_this_run` | `widget_id in active_widget_ids` |
| 后端消息队列 | `ForwardMsgQueue.clear` | `msg.delta.fragment_id not in fragment_ids_this_run` | 消息已被清理（不保留即代表"被替代"） |
| 前端 ElementNode | `ClearStaleNodeVisitor` | `!node.fragmentId \|\| !fragmentIdOfBlock` | `node.scriptRunId === currentScriptRunId` |
| 前端实时感知 | `isElementStale` | `!node.fragmentId \|\| !fragmentIdsThisRun.includes(node.fragmentId)` | `node.scriptRunId === scriptRunId` |

**统一的核心开关**：`fragment_ids_this_run`（后端）/ `fragmentIdsThisRun`（前端）。同一个值，从 `RerunData` 一路传递到前端 `App.tsx` 的 `this.state.fragmentIdsThisRun`，再传入 `clearStaleNodes`、`clearTransientNodes`、`isElementStale` 等所有判定点。

---

## 二十、前端清理机制总结表

| 机制 | 触发时机 | 清理范围 | 保留条件 | 关键代码 |
|------|---------|---------|---------|---------|
| `clearTransientNodes` | NewSession 到达 | 所有 TransientNode 的临时元素 | 不遍历非本次运行的 fragment Block | [ClearTransientNodesVisitor](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/visitors/ClearTransientNodesVisitor.ts#L29-L84) |
| `clearStaleNodes` | ScriptFinished | stale 的 BlockNode / ElementNode | 全局元素、外部 fragment 元素、本次更新的元素 | [ClearStaleNodeVisitor](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/visitors/ClearStaleNodeVisitor.ts#L40-L179) |
| `isElementStale` | React 渲染期间 | 无（只读判定） | 同 `clearStaleNodes` 的 ElementNode 规则 | [isElementStale](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/components/core/Block/utils.ts#L43-L70) |
| `removeInactiveWidgetState` | `clearStaleNodes` 之后 | 前端 WidgetStateManager 中的无效状态 | 仍存在于渲染树中的 Widget | [handleScriptFinished](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/app/src/App.tsx#L1619-L1631) |

四个机制形成一条清理流水线：

```
NewSession → clearTransientNodes → [delta 流入] → ScriptFinished → clearStaleNodes → removeInactiveWidgetState
     ↑ 清理临时元素                      ↑ 实时禁用旧元素         ↑ 裁剪 stale 节点       ↑ 清理 Widget 前端状态
     ↑ 先打扫                           ↑ isElementStale        ↑ 最终收敛              ↑ 收尾
```

每一步都比上一步更"决绝"：从"隐藏临时元素"到"禁用旧元素"到"移除旧节点"到"清理 Widget 状态"。而 `fragmentIdsThisRun` 贯穿始终，确保每一步都只影响本次运行的 fragment 范围，**不影响外部世界的任何状态**。

---

## 二十一、裁剪边界的精确定义：什么算"在 fragment 内部"

前面多次提到"裁剪范围"和"外部节点"，但边界究竟在哪里？这是理解局部清理的关键。

### 21.1 边界判定的两层模型

前端的 fragment 边界判定采用**两层模型**，与后端的 `delta_path` 前缀匹配异曲同工：

```
层 1：节点自身的 fragmentId 标记（身份属性）
层 2：遍历路径上的 fragmentIdOfBlock 上下文（位置属性）
```

每个节点有一个**身份属性** `fragmentId`：它属于哪个 fragment。这是节点创建时永久打上的烙印。

每次遍历时有一个**位置属性** `fragmentIdOfBlock`：当前遍历到的位置，是否在"本次运行修改的 Block"内部。这是动态变化的上下文。

### 21.2 为什么需要两层？一层不行吗？

只靠 `node.fragmentId` 是不够的。反例：

```
fragment A 的容器 Block（fragmentId="fragA"，scriptRunId="run1"）
    └── 内部的普通 Block（没有 fragmentId，scriptRunId="run1"）
            └── ElementNode（没有 fragmentId，scriptRunId="run1"）
```

这里内部的 Block 和 Element 的 `fragmentId` 都是 `undefined`，但它们**确实在 fragment A 的边界内**。如果只看 `node.fragmentId`，会错误地认为它们是"全局元素"而保留。

所以必须有第二层——**位置属性** `fragmentIdOfBlock`，它沿着树向下传递，标记"你现在在哪个 fragment 的势力范围内"。

### 21.3 `fragmentIdOfBlock` 的传播规则

> 代码定位：[visitBlockNode 中的新 visitor 创建](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/visitors/ClearStaleNodeVisitor.ts#L84-L94)

`fragmentIdOfBlock` 的传播有严格的触发条件：

```typescript
if (
  node.fragmentId &&                           // 条件 1：这个 Block 有 fragmentId
  this.fragmentIdsThisRun.includes(node.fragmentId) &&  // 条件 2：属于本次运行
  node.scriptRunId === this.currentScriptRunId  // 条件 3：被本次运行更新过
) {
  clearStaleNodeVisitor = new ClearStaleNodeVisitor(
    this.currentScriptRunId,
    this.fragmentIdsThisRun,
    node.fragmentId  // ← 设置为这个 Block 的 fragmentId
  )
}
```

**三个条件缺一不可**：
1. Block 本身有 `fragmentId`（它是 fragment 的容器）
2. 该 `fragmentId` 在本次运行列表中
3. 该 Block 的 `scriptRunId` 等于当前运行 ID（被本次运行更新过）

只有同时满足，才会把 `fragmentIdOfBlock` 设置为该 fragment 的 ID，向下传播。

**关键洞察**：`fragmentIdOfBlock` 一旦设置，就不会被清除（除非遇到另一个满足条件的 fragment Block 重新设置）。它像一个"进入了 fragment 领域"的印章，一旦进入，内部所有节点都在这个领域内。

### 21.4 裁剪范围的精确边界

结合两层模型，裁剪范围的精确边界是：

> **从根节点向下遍历，遇到第一个"属于本次运行且被本次运行更新的 fragment Block"时，其所有子孙节点都进入了裁剪范围。**

换句话说：
- **裁剪范围 = 所有属于 `fragmentIdsThisRun` 且 `scriptRunId === currentScriptRunId` 的 Block 的子树**
- **保留范围 = 裁剪范围之外的所有节点**

在裁剪范围内，`scriptRunId` 不匹配的节点就是 stale 的，会被裁掉。在保留范围内，所有节点都保留。

### 21.5 与后端 `delta_path` 的对应关系

前后端的边界判定是同构的：

| 维度 | 后端（Python） | 前端（TypeScript） |
|------|---------------|-------------------|
| 边界标记 | `delta_path` 前缀 | `fragmentId` + `fragmentIdOfBlock` 上下文 |
| 判定函数 | `_is_inside_fragment_path()` | `fragmentIdOfBlock` 隐式判定 |
| 匹配原则 | 路径前缀匹配 | 子树包含关系 |

后端的 `delta_path` 是精确的坐标前缀匹配，前端的 `fragmentIdOfBlock` 是更粗粒度的"子树包含"判定。两者等价——因为一个 fragment 的所有元素都在其 container Block 的子树内。

---

## 二十二、保留条件的完备性分析：为什么不会误删

`ClearStaleNodeVisitor` 的保留条件看起来复杂，但背后有清晰的逻辑保证——**该保留的一定保留，该清理的一定清理**。

### 22.1 ElementNode 的保留条件完备性

回顾 [visitElementNode](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/visitors/ClearStaleNodeVisitor.ts#L125-L141) 的三条件：

```typescript
if (this.isFragmentRun) {
  if (
    !node.fragmentId ||           // 条件 A：元素没有 fragmentId
    !this.fragmentIdOfBlock ||    // 条件 B：不在裁剪范围内
    node.scriptRunId === this.currentScriptRunId  // 条件 C：已被本次更新
  ) {
    return node  // 保留
  }
}
return node.scriptRunId === this.currentScriptRunId ? node : undefined
```

**三个条件的逻辑覆盖**：

| 元素在哪 | 有无 fragmentId | 是否在裁剪范围 | 是否被更新 | 结果 | 对应条件 |
|---------|---------------|---------------|-----------|------|---------|
| 全局（fragment 外） | 无 | 否 | - | 保留 | A + B |
| 其他 fragment 内 | 有 | 否 | - | 保留 | B |
| 本 fragment 内，已更新 | 有 | 是 | 是 | 保留 | C |
| 本 fragment 内，未更新 | 有 | 是 | 否 | 清理 | 都不满足 |

四种情况全覆盖，没有遗漏，也没有误判。

### 22.2 BlockNode 的保留条件完备性

回顾 [visitBlockNode](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/lib/src/render-tree/visitors/ClearStaleNodeVisitor.ts#L61-L123) 的判定：

| Block 类型 | 是否在裁剪范围 | scriptRunId 匹配 | 结果 |
|-----------|---------------|-----------------|------|
| 全局 Block | 否 | - | 保留（递归子节点） |
| 其他 fragment 的 Block | 否 | - | 保留（递归子节点） |
| 本 fragment 容器 Block（本次更新） | - | 是 | 保留（设置 context 后递归） |
| 本 fragment 内的子 Block（未更新） | 是 | 否 | 清理（返回 undefined） |
| 本 fragment 内的子 Block（已更新） | 是 | 是 | 保留（递归子节点） |

**关键设计**：Block 节点本身很少被直接清理（除了路径 B1 的情况），大多数情况下 Block 是保留的，只是其内部的子节点可能被清理。这确保了页面的容器结构不会乱。

### 22.3 为什么不会误删全局元素

全局元素（`fragmentId = undefined`）的节点，在 `visitElementNode` 中命中条件 A（`!node.fragmentId`）被保留。这是最外层的保护罩——**任何没有被打上 fragment 烙印的元素，永远不会因为 fragment 重跑而被清理**。

这对应后端的逻辑：`metadata.fragment_id not in fragment_ids_this_run` 的 widget 被保留。前后端在这一点上是对称的。

---

## 二十三、界面收敛的物理意义：为什么是两步而不是一步

"先清 transient、后清 stale"的两步清理不是随意设计的，而是对应着界面收敛的**两个物理阶段**。

### 23.1 第一阶段：临时元素退场（NewSession 时）

**目的**：让"上一轮的临时反馈"先消失，为新运行腾出视觉空间。

- 时间点：`NewSession` 消息到达，delta 还没开始流入
- 清理对象：TransientNode 中的临时元素（toast、chat message 通知等）
- 保留对象：anchor 持久节点、所有 ElementNode、所有 BlockNode
- 视觉效果：toast 消失、临时通知消失，但页面主体结构不动

**为什么必须在 NewSession 时清理而不是等到 ScriptFinished**？
因为临时元素是"上一次运行的产物"，新运行开始后它们就不再有意义。如果等到运行结束才清理，运行期间用户会看到新旧临时元素混杂的混乱状态。

### 23.2 第二阶段：陈旧元素下架（ScriptFinished 时）

**目的**：移除本次运行没有重新渲染的旧元素，完成最终收敛。

- 时间点：`script_finished` 消息到达，所有 delta 已处理完毕
- 清理对象：裁剪范围内 `scriptRunId` 不匹配的节点
- 保留对象：裁剪范围外的所有节点 + 裁剪范围内已更新的节点
- 视觉效果：fragment 内不再显示的元素平滑消失，页面达到最终稳定状态

**为什么必须等到 ScriptFinished 才清理**？
因为在运行期间，delta 是逐条流入的。如果边流入边清理，会导致：
1. 闪烁：旧元素先消失，新元素后出现，中间有空白
2. 顺序错误：先清理了容器，后收到子元素的 delta，子元素无处安放
3. 状态丢失：Widget 的 React 状态在节点删除时会丢失，即使新 delta 很快又创建了同名节点

所以必须**等所有 delta 都到齐了，一次性裁剪**，让 React 在一次 re-render 中完成新旧交替，保证流畅和正确。

### 23.3 两步收敛的时序意义

```
时间轴 →
  │
  ├─ NewSession 到达
  │    ↓ clearTransientNodes
  │    临时元素消失
  │    （页面主体不动，用户几乎无感知）
  │
  ├─ delta 1 到达 → applyDelta
  ├─ delta 2 到达 → applyDelta
  ├─ ... （逐个更新，新旧节点并存）
  │    期间 isElementStale 让旧元素变灰
  │    （用户感知：旧元素逐渐被替换，新元素逐个出现）
  │
  └─ ScriptFinished 到达
       ↓ clearStaleNodes
       一次性移除所有 stale 节点
       （最终收敛，页面达到稳定状态）
```

两步收敛的好处：
- **平滑过渡**：不会出现"全白一下再出来"的闪烁
- **渐进呈现**：用户可以边看边等，而不是盯着空白屏幕
- **状态保留**：同类型的新旧元素替换时，React 可以复用 DOM 节点，保留滚动位置、输入焦点等状态

### 23.4 与后端 `ForwardMsgQueue` 的协同

前端的两步清理与后端 `ForwardMsgQueue.clear()` 的保留策略是一一对应的：

| 阶段 | 后端动作 | 前端动作 | 协同点 |
|------|---------|---------|-------|
| 运行开始 | 清理旧 fragment delta（SCRIPT_STARTED 时的 `_clear_queue`） | 清理 Transient 节点 | 两端都先做"轻量清理" |
| 运行中 | 持续发送新 delta | 逐个 applyDelta，isElementStale 实时禁用 | 新旧并存，渐进更新 |
| 运行结束 | 保留非 fragment delta | clearStaleNodes 裁剪 stale 节点 | 两端都保留外部状态 |

这种前后端对称的设计，确保了消息流和渲染树的状态始终一致，不会出现后端说"有这个元素"但前端说"我删了"的矛盾。

---

## 二十四、边界的传递性：嵌套 fragment 的清理范围

当 fragment 嵌套时，清理范围会沿着父子关系传递。这是一个容易被忽略但至关重要的细节。

### 24.1 嵌套场景的树结构

```
Block (fragParent, scriptRunId="run2")  ← 父 fragment 容器
    ├── Element: "Parent header" [fragParent, run2]
    ├── Block (fragChild, scriptRunId="run2")  ← 子 fragment 容器
    │   └── Element: "Child content" [fragChild, run2]
    └── Block (没有 fragmentId, scriptRunId="run1")  ← 普通内部 Block
        └── Element: "Old content" [undefined, run1]
```

### 24.2 只重跑父 fragment 的情况

`fragmentIdsThisRun = ["fragParent"]`

遍历过程：
1. 遇到 fragParent Block → 满足三条件（有 fragmentId、在列表中、scriptRunId 匹配）→ 设置 `fragmentIdOfBlock = "fragParent"`
2. 递归子节点：
   - Parent header → `fragmentId = "fragParent"`，`fragmentIdOfBlock = "fragParent"`，scriptRunId 匹配 → 保留（条件 C）
   - fragChild Block → `fragmentId = "fragChild"`，`fragmentIdOfBlock = "fragParent"`，scriptRunId 匹配 → 保留
     - 注意：fragChild 虽然在列表外，但它的 scriptRunId 匹配，所以不会被路径 B1 裁掉
     - 但 fragChild 的 `fragmentId` 不在 `fragmentIdsThisRun` 中，所以不会创建新的 visitor（不重新设置 fragmentIdOfBlock）
   - 普通内部 Block → `scriptRunId = "run1"` ≠ "run2"，`fragmentIdOfBlock = "fragParent"` 有值 → **路径 B1：裁掉！**

**结论**：只重跑父 fragment 时，父 fragment 内的所有"非子 fragment 的"旧节点都会被清理，但子 fragment 的容器（如果被更新了）会保留。

### 24.3 只重跑子 fragment 的情况

`fragmentIdsThisRun = ["fragChild"]`

遍历过程：
1. 遇到 fragParent Block → `scriptRunId = "run2"`，`fragmentIdOfBlock = undefined` → 不触发路径 B1 → 递归
   - 注意：fragParent 的 `fragmentId` 不在列表中，所以不会设置 `fragmentIdOfBlock`
2. 递归到 fragChild Block → `fragmentId = "fragChild"` 在列表中，`scriptRunId = "run2"` 匹配 → **设置 `fragmentIdOfBlock = "fragChild"`**
3. 继续递归 fragChild 的子节点 → 在裁剪范围内

**关键问题**：父 fragment 的 Block 没有被标记为"在裁剪范围内"，它内部的其他元素（非 fragChild 的）会不会被误清理？

答案是**不会**。因为 `fragmentIdOfBlock` 只有在进入 fragChild 时才被设置，fragParent 的其他子节点遍历的时候 `fragmentIdOfBlock` 还是 `undefined`，所以它们都命中条件 B（不在裁剪范围内）被保留。

**这证明了边界的精确性**：清理范围严格限制在被重跑的 fragment 的子树内，不会"向上泄漏"到父 fragment，也不会"侧向泄漏"到兄弟节点。

### 24.4 嵌套清理与后端 `clear_stale_descendants` 的对应

前端的嵌套清理与后端的 [clear_stale_descendants](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/lib/streamlit/runtime/fragment.py#L246-L263) 是协同工作的：

- 后端：决定哪些嵌套子 fragment 是"stale 后裔"，从存储中移除，不再产生 delta
- 前端：决定哪些节点是 stale 的，从渲染树中移除

后端先决定"哪些 fragment 不跑了"，前端再根据收到的 delta 决定"哪些节点该删了"。两者形成完整的清理闭环。

---

## 二十五、收敛一致性：前后端状态的最终对齐

fragment 重跑结束后，前后端的状态必须精确对齐，否则用户会看到不一致的界面。

### 25.1 收敛后的一致性保证

一次成功的 fragment 重跑结束后，以下状态应该完全一致：

| 状态 | 后端 | 前端 | 对齐方式 |
|------|------|------|---------|
| Fragment 存储 | `FragmentStorage` 中的注册 fragment | 渲染树中带 `fragmentId` 的 Block | 通过 delta 流同步 |
| Widget 状态 | `SessionState` 中的活跃 widget | `WidgetStateManager` + 渲染树中的 widget 元素 | `removeInactiveWidgetState` 对齐 |
| Query Params 绑定 | `QueryParams` 中的绑定关系 | URL 查询字符串 | 通过 `pageInfoChanged` 同步 |
| 渲染内容 | 各 fragment 的 delta 输出 | 渲染树中的元素节点 | 逐个 delta 应用 + stale 清理 |

### 25.2 `removeInactiveWidgetState` 的收尾作用

> 代码定位：[removeInactiveWidgetState](file:///d:/fz/0601/solo-dogfeeding/code/227-streamlit/frontend/app/src/App.tsx#L1696-L1705)

```typescript
private removeInactiveWidgetState(): void {
  const { elements, blockIds } = this.state.elements.getActiveIds()
  const activeIds = new Set([
    ...Array.from(elements).map(element => getElementId(element)).filter(notUndefined),
    ...blockIds,
  ])
  this.widgetMgr.removeInactive(activeIds)
}
```

这是清理流水线的**最后一步**。在 `clearStaleNodes` 把 stale 节点从渲染树移除后，这一步把对应的 Widget 前端状态也清理掉。

**为什么需要单独这一步**？因为：
1. Widget 的状态分为两部分：渲染树中的节点（UI）+ `WidgetStateManager` 中的值（数据）
2. `clearStaleNodes` 只清理了渲染树，`WidgetStateManager` 里的数据还在
3. 必须在渲染树稳定后，以树中的活跃元素为准，清理掉不再存在的 Widget 的数据状态

这与后端的 `_remove_stale_widgets` 是对称的——后端在脚本结束时清理 stale widget 状态，前端在渲染树稳定后清理 stale widget 状态。

### 25.3 收敛失败的安全边界

如果因为网络抖动或消息乱序导致前后端不一致，有三道安全边界：

1. **`FINISHED_EARLY_FOR_RERUN` 保护**：如果运行被打断，不做 stale 清理，避免误删
2. **全局重跑兜底**：下一次全局重跑会重建整个渲染树，任何不一致都会被修正
3. **session 重建**：连接断开重连后，整个 session 重建，状态完全重置

fragment 重跑是"局部优化"，全局重跑是"真相之源"。所有 fragment 级别的状态最终都可以通过全局重跑校准。

---

## 二十六、总结：局部清理的三层边界契约

从后端到前端，fragment 的局部清理形成了三层边界，层层保护，确保"只清理该清理的，保留该保留的"：

```
┌──────────────────────────────────────────────────────────────┐
│                      第一层：调度边界                          │
│  fragment_id_queue 决定"只跑这些 fragment"                    │
│  → 只有被调度的 fragment 才会产生新的 delta                    │
│  → 只有产生新 delta 的 fragment 才可能有 stale 节点           │
└─────────────────────────────┬────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────┐
│                      第二层：消息边界                          │
│  ForwardMsgQueue.clear() 只清理本次 fragment 的旧 delta       │
│  → 前端只会收到新的 delta，不会收到外部的重渲染                │
│  → 外部元素的 delta 消息始终保留在队列中                       │
└─────────────────────────────┬────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────┐
│                      第三层：渲染树边界                        │
│  ClearStaleNodeVisitor 只裁剪"被修改的 fragment 子树"内的       │
│  stale 节点                                                   │
│  → 全局元素、其他 fragment 的元素都被保护                      │
│  → 同 fragment 内被更新的元素也被保留                          │
└─────────────────────────────┬────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────┐
│                    最终收敛：界面稳定态                        │
│  - 外部元素纹丝不动                                           │
│  - 本 fragment 内：旧的去，新的留                             │
│  - Widget 状态、Query Params、Session State 前后端对齐         │
└──────────────────────────────────────────────────────────────┘
```

三层边界的共同核心是同一个值：`fragment_ids_this_run`（后端）/ `fragmentIdsThisRun`（前端）。

- **向上**：它决定了 ScriptRunner 执行哪些 fragment（调度边界）
- **中间**：它决定了哪些消息被清理、哪些状态被保留（消息边界 / 状态边界）
- **向下**：它决定了前端渲染树中哪些节点可能被裁剪（渲染树边界）

同一个开关，贯穿调度、状态、渲染三层，这就是 Streamlit fragment 局部重跑机制的底层设计哲学——**用一个核心变量，控制所有相关子系统的行为，确保全局一致性**。
