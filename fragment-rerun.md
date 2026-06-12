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
