# Streamlit 模态弹层（Dialog）机制分析

## 一、整体架构概览

Streamlit 的模态弹层实现横跨 **Python 后端** 和 **TypeScript 前端**，涉及装饰器、DeltaGenerator、Protobuf 传输、Fragment 执行模型和 React 组件树等多个层次。核心路径如下：

```
@st.dialog 装饰器 → Dialog._create（注册 Block） → Dialog.open（发送 is_open=true）
       → _fragment 包裹内容 → FragmentStorage.register
       → 前端 Block.tsx 识别 dialog block → Dialog.tsx 渲染 Modal
       → 用户关闭 → handleClose → WidgetStateManager.setTriggerValue
       → 后端回调 on_dismiss / rerun
```

---

## 二、弹层注册

### 2.1 装饰器入口

[dialog_decorator](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/elements/dialog_decorator.py#L142) 是用户侧入口 `@st.dialog("title")` 的实现。核心逻辑在 [_dialog_decorator](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/elements/dialog_decorator.py#L62)：

1. **校验**：检查不嵌套（`_assert_no_nested_dialogs`）、不在 parallel fragment 中（`_check_not_parallel_worker`）
2. **创建 Dialog Block**：通过 `event_dg._dialog(...)` 创建 `Dialog` 实例
3. **调用 `dialog.open()`**：发送 `is_open=True` 的 Protobuf 消息
4. **将内容包裹为 Fragment**：用 `_fragment` 装饰 `dialog_content`，注册到 `FragmentStorage`
5. **在 Dialog 上下文中执行 Fragment**：`with dialog: fragmented_dialog_content()`

### 2.2 event_dg — 独立的渲染容器

弹层不在 MAIN/SIDEBAR 容器中渲染，而是使用 `event_dg`（RootContainer.EVENT）。这保证了弹层不受父容器主题影响（例如 sidebar 中的按钮触发弹层，弹层不会继承 sidebar 样式）。

[DeltaGeneratorSingleton](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/delta_generator_singletons.py#L95) 初始化了四个根容器：

```python
self._main_dg    = DeltaGenerator(root_container=_RootContainer.MAIN)
self._sidebar_dg = DeltaGenerator(root_container=_RootContainer.SIDEBAR)
self._event_dg   = DeltaGenerator(root_container=_RootContainer.EVENT)
self._bottom_dg  = DeltaGenerator(root_container=_RootContainer.BOTTOM)
```

对应 [RootContainer.proto](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/proto/streamlit/proto/RootContainer.proto#L24) 中的枚举 `MAIN=0, SIDEBAR=1, EVENT=2, BOTTOM=3`。

前端的 [AppRoot](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/render-tree/AppRoot.ts#L139) 按顺序维护四个子 BlockNode：

```typescript
const main    = new BlockNode(mainScriptHash, mainNodes, ...)
const sidebar = new BlockNode(mainScriptHash, [], ...)
const event   = new BlockNode(mainScriptHash, [], ...)   // ← Dialog 渲染在这里
const bottom  = new BlockNode(mainScriptHash, [], ...)
```

### 2.3 Dialog._create — Block Proto 构建

[Dialog._create](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/elements/lib/dialog.py#L83) 执行以下步骤：

1. **构建 BlockProto**：设置 `block_proto.dialog.title`、`dismissible`、`width`（映射为枚举）、`icon`
2. **计算 element_id**：通过 `compute_and_register_element_id` 生成稳定 ID，基于 `title + dismissible + width + icon + on_dismiss` 的组合。此 ID 用于：
   - 前端区分不同弹层，防止显示旧弹层的内容（issue #10907）
   - 作为 widget 注册标识（当 `on_dismiss` 被激活时）
3. **设置 block_proto.id**：始终设置，用于前端弹层身份识别
4. **条件性 Widget 注册**：仅当 `on_dismiss != "ignore"` 时：
   - 设置 `block_proto.dialog.id = element_id`（前端据此判断是否触发 rerun）
   - 调用 `register_widget(element_id, ..., value_type="trigger_value")` 注册为 trigger 类型 widget
5. **记录 delta_path**：保存当前游标的 delta_path，供后续 `_update` 方法使用
6. **创建 Dialog 实例**：通过 `parent._block(block_proto=block_proto, dg_type=Dialog)` 创建

### 2.4 单例保护 — 一次只允许一个弹层

[_assert_first_dialog_to_be_opened](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/elements/lib/dialog.py#L57) 检查 `ScriptRunContext.has_dialog_opened` 标志：

- 当 `should_open=True` 且已有弹层打开时，抛出 `StreamlitAPIException`
- 首次打开时将 `script_run_ctx.has_dialog_opened = True`
- 该标志在 `ScriptRunContext.reset()` 中被重置为 `False`（每次脚本运行开始时）

[ScriptRunContext.has_dialog_opened](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py#L224) 的重置时机：

```python
def reset(self, ...):
    ...
    self.has_dialog_opened = False
```

### 2.5 嵌套保护

[_assert_no_nested_dialogs](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/elements/dialog_decorator.py#L35) 检查当前 DeltaGenerator 栈中是否已有 `dialog` 类型的祖先 block：

```python
last_dg_in_current_context = get_last_dg_added_to_context_stack()
if last_dg_in_current_context and "dialog" in set(
    last_dg_in_current_context._ancestor_block_types
):
    raise StreamlitAPIException("Dialogs may not be nested inside other dialogs.")
```

---

## 三、事件触发

### 3.1 打开弹层 — Dialog.open

[Dialog.open](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/elements/lib/dialog.py#L195) 调用 `Dialog._update(True)`，该方法：

1. 调用 `_assert_first_dialog_to_be_opened(True)` 检查单例约束
2. 构建 `ForwardMsg`，设置 `delta_path` 指向弹层 Block 位置
3. 复制 `_current_proto` 到消息中，并设置 `dialog.is_open = True`
4. 通过 `enqueue_message(msg)` 发送到前端

### 3.2 关闭弹层 — 前端触发

前端 [Dialog.tsx](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/components/elements/Dialog/Dialog.tsx#L93) 的 `handleClose` 回调：

```typescript
const handleClose = useCallback(() => {
    setIsOpen(false)
    if (id && widgetMgr) {
        void widgetMgr.setTriggerValue(
            { id, formId: "" },
            { fromUi: true },
            fragmentId
        )
    }
}, [id, widgetMgr, fragmentId])
```

**关键判断**：当 `dialog.id` 存在时（即后端设置了 `on_dismiss != "ignore"`），关闭弹层会通过 `WidgetStateManager.setTriggerValue` 发送一个 trigger widget 事件到后端。

`setTriggerValue` 将 `triggerValue = true` 写入 widget state proto，并通过 `updateWidgets` 消息发送到后端。后端 `SessionState` 在脚本运行前读取此值，调用注册的 `on_change_handler`（即 `on_dismiss` 回调），并决定是否 rerun。

### 3.3 on_dismiss 的三种模式

| on_dismiss 值 | dialog.id 设置 | Widget 注册 | 关闭行为 |
|---|---|---|---|
| `"ignore"` | 不设置 | 不注册 | 纯 UI 关闭，不触发 rerun |
| `"rerun"` | 设置 | 注册（无 callback） | 关闭时触发完整 rerun |
| `callable` | 设置 | 注册（有 callback） | 关闭时触发 rerun，先执行 callback |

### 3.4 非 dismissible 弹层的键盘拦截

当 `dismissible=False` 时，Dialog.tsx 会拦截 `R` 键的快捷键行为（防止用户按 R 触发 rerun 绕过弹层）：

```typescript
if (isOpen && e.key.toLowerCase() === "r" && !element.dismissible) {
    // 不在输入框中时阻止 R 键冒泡
    e.preventDefault()
    e.stopPropagation()
}
```

使用 `capture: true` 在 document 级别拦截，优先于 App 层的快捷键处理。

---

## 四、页面状态隔离

### 4.1 Fragment 执行模型

Dialog 的内容函数被 `_fragment` 包裹，继承 Fragment 的所有隔离特性：

[dialog_decorator.py#L100-105](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/elements/dialog_decorator.py#L100)：

```python
fragmented_dialog_content = cast(
    "Callable[[], None]",
    _fragment(
        dialog_content, additional_hash_info=get_object_name(non_optional_func)
    ),
)
```

这意味着：
- **Widget 交互只 rerun Fragment**：弹层内的 widget 交互（如 text_input、button）只重新执行弹层函数，不触发全脚本 rerun
- **FragmentStorage 注册**：弹层内容函数作为 Fragment 注册到 `ctx.fragment_storage`，key 为基于函数模块名 + delta_path 的哈希
- **独立 cursor 快照**：Fragment 创建时保存 `cursors` 和 `dg_stack` 的深拷贝，rerun 时恢复，确保元素写入正确位置

### 4.2 EVENT 容器隔离

弹层写入 EVENT 容器而非 MAIN 容器，实现了：

1. **布局隔离**：弹层不参与主区域的流式布局，始终以 Modal 形式覆盖在页面上方
2. **主题隔离**：不继承 sidebar/bottom 等容器的主题样式
3. **渲染树隔离**：在 AppRoot 中，event 是独立的 BlockNode 子树，clearStaleNodes 等操作按容器分别处理

### 4.3 IsDialogContext — React Context 隔离

[IsDialogContext](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/components/core/IsDialogContext.ts) 是一个 React Context，默认值 `false`。当组件在 Dialog 内部渲染时被设置为 `true`：

```tsx
function DialogWithProvider(props) {
    return (
        <IsDialogContext.Provider value={true}>
            <Dialog {...props} />
        </IsDialogContext.Provider>
    )
}
```

消费此 Context 的组件包括：
- **StreamlitMarkdown**：在弹层内禁用锚点标题的 heading anchor 链接
- **Heading**：同上，弹层内的标题不生成锚点

### 4.4 disableFullscreenMode 传播

[Block.tsx](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/components/core/Block/Block.tsx#L286) 中，Dialog block 自动向下传播 `disableFullscreenMode=true`：

```typescript
const disableFullscreenMode =
    props.disableFullscreenMode ||
    notNullOrUndefined(node.deltaBlock.dialog) ||
    notNullOrUndefined(node.deltaBlock.popover)
```

弹层内的元素（如图表、图片）不会显示全屏按钮。

### 4.5 弹层身份防重 — 防止旧内容残留

[AppRoot.addBlock](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/render-tree/AppRoot.ts#L471) 中，当替换 dialog block 时检查身份：

```typescript
const isDialogWithDifferentIdentity =
    block.dialog &&
    existingNode.deltaBlock.dialog &&
    block.id !== existingNode.deltaBlock.id

if (!isDialogWithDifferentIdentity) {
    children = existingNode.children  // 继承子节点
}
```

如果新旧 dialog 的 `block.id` 不同（即属性不同的弹层），不继承旧弹层的子节点，防止显示上一个弹层残留的元素内容。

---

## 五、端到端协作流程

### 5.1 首次打开弹层

```
用户代码: vote("A")  →  _dialog_decorator.wrap()
  ├─ _check_not_parallel_worker()         ← 并行 fragment 安全检查
  ├─ _assert_no_nested_dialogs()          ← 嵌套检查
  ├─ event_dg._dialog(title, ...)         ← 在 EVENT 容器创建 Dialog Block
  │    └─ Dialog._create()
  │         ├─ compute_and_register_element_id()  ← 生成稳定 ID
  │         ├─ register_widget() (条件)           ← on_dismiss 时注册
  │         └─ parent._block(block_proto)          ← 创建 Block 节点
  ├─ dialog.open()                         ← 发送 is_open=True 的 ForwardMsg
  │    └─ _update(True)
  │         ├─ _assert_first_dialog_to_be_opened(True)  ← 单例检查
  │         └─ enqueue_message(msg)          ← 通过 WebSocket 发送
  ├─ _fragment(dialog_content)             ← 包裹为 Fragment
  │    └─ fragment_storage.register(fragment_id, wrapped_fragment)
  └─ with dialog: fragmented_dialog_content()  ← 在 Dialog 上下文中执行
```

### 5.2 弹层内 Widget 交互（Fragment rerun）

```
用户在弹层内操作 widget
  → 前端 WidgetStateManager 发送 widget update
  → ScriptRunner 检测到 fragment_id 对应的 widget
  → 仅执行该 Fragment 的 wrapped_fragment()
  → 弹层内容更新，主页面不 rerun
```

### 5.3 关闭弹层

**程序关闭**：弹层内调用 `st.rerun()`，触发完整脚本 rerun。由于条件不再满足，弹层函数不会被调用，新脚本运行中弹层自然消失。

**用户关闭**（点击遮罩/X 按钮/ESC）：
- `on_dismiss="ignore"`：纯前端 `setIsOpen(false)`，无后端通知
- `on_dismiss="rerun"`：`setTriggerValue` → 后端 rerun 整个脚本
- `on_dismiss=callback`：`setTriggerValue` → 后端先执行 callback，再 rerun

---

## 六、关键文件索引

| 层级 | 文件 | 职责 |
|---|---|---|
| 装饰器 | [dialog_decorator.py](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/elements/dialog_decorator.py) | `@st.dialog` 入口，校验与 Fragment 包裹 |
| Dialog 类 | [dialog.py](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/elements/lib/dialog.py) | Block 创建、open/close、Widget 注册 |
| DG 单例 | [delta_generator_singletons.py](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/delta_generator_singletons.py) | event_dg 等根容器初始化 |
| Fragment | [fragment.py](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/runtime/fragment.py) | Fragment 执行模型、存储 |
| 运行上下文 | [script_run_context.py](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/lib/streamlit/runtime/scriptrunner_utils/script_run_context.py) | `has_dialog_opened` 状态、消息队列 |
| Proto 定义 | [Block.proto](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/proto/streamlit/proto/Block.proto#L118) | Dialog 消息结构 |
| 根容器 Proto | [RootContainer.proto](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/proto/streamlit/proto/RootContainer.proto) | EVENT 容器枚举 |
| 前端 Dialog | [Dialog.tsx](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/components/elements/Dialog/Dialog.tsx) | React 弹层组件、关闭处理 |
| 前端 Modal | [Modal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/components/shared/Modal/Modal.tsx) | 底层 Modal 渲染（react-aria） |
| Block 渲染 | [Block.tsx](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/components/core/Block/Block.tsx) | 识别 dialog block 并渲染 Dialog |
| 渲染树 | [AppRoot.ts](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/render-tree/AppRoot.ts) | 四容器树结构、弹层身份防重 |
| 上下文隔离 | [IsDialogContext.ts](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/components/core/IsDialogContext.ts) | 弹层内 React Context 标记 |
| Widget 管理 | [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/231-streamlit/frontend/lib/src/WidgetStateManager.ts) | setTriggerValue 触发 dismiss 回调 |

---

## 七、边界问题：三个关键概念

### 7.1 关闭弹层后重跑的范围

弹层的「关闭触发重跑」与「弹层内 Widget 交互触发重跑」属于两种完全不同的重跑范围，核心区别在于重跑的**调度单位**：

#### 重跑范围的层级模型

可以把 Streamlit 脚本的重跑范围想象为三层嵌套结构：

```
┌─────────────────────────────────────────────────┐
│  L3: 完整脚本重跑（Full Script Rerun）            │
│  ┌─────────────────────────────────────────────┐ │
│  │  L2: 弹层 Fragment 重跑（仅执行弹层内容函数） │ │
│  │  ┌───────────────────────────────────────┐  │ │
│  │  │  L1: 弹层内 Widget 级局部状态          │  │ │
│  │  └───────────────────────────────────────┘  │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

- **L2 弹层 Fragment 重跑**：弹层内 Widget（如输入框、按钮、滑块）的交互。此时仅执行弹层装饰器包裹的那一个函数，主脚本其他部分（主页面所有逻辑）**完全不执行**。这是 Fragment 执行模型的天然特性——弹层本质上就是一个「自动 Fragment」。

- **L3 完整脚本重跑**：以下事件会触发最外层的完整重跑：
  1. 用户在弹层内显式调用 `st.rerun()`（无 scope 参数时默认全脚本）
  2. `on_dismiss="rerun"` 配置下，用户手动关闭弹层（点击遮罩 / X 按钮 / ESC）
  3. `on_dismiss=callable` 配置下，用户手动关闭弹层（先执行 callback，再做完整重跑）
  4. 主页面上的 Widget 交互（此时弹层函数是否被调用取决于外层条件判断）

#### on_dismiss 对重跑范围的决定作用

`on_dismiss` 实际上是一个「关闭动作是否穿透到主脚本」的开关：

| on_dismiss 配置 | 关闭时重跑范围 | 说明 |
|---|---|---|
| `"ignore"` | **无重跑** | 纯前端行为，弹层 DOM 被卸载，后端不感知。这是默认值——因为大多数弹层关闭后不应无条件重跑。 |
| `"rerun"` | **完整脚本重跑** | 关闭动作等价于一个「触发型 Widget」被激活，后端按 widget 交互处理，进入标准的完整脚本执行流程。 |
| `callable 回调` | **完整脚本重跑 + 回调优先** | 回调函数在主脚本执行之前被调用，属于 Widget 的 on_change 机制。若回调内部再次触发重跑，则在同一轮调度中合并处理。 |

> **关键理解**：`on_dismiss` 的 `"rerun"` 和回调模式，本质是将弹层本身注册为一个**一次性触发 Widget**（trigger_value 类型）。当用户关闭弹层时，前端向该 Widget 写入 `true`，后端在重跑前读取此值并触发相应的 on_change 逻辑。重跑范围之所以是完整脚本，是因为 Widget 重跑范围由 Widget 所在的 Fragment 层级决定——而「弹层作为触发型 Widget」的 ID 被注册在 EVENT 容器、而非弹层内部的 Fragment 中，因此属于最顶层 Fragment，触发完整重跑。

#### 典型场景的重跑路径

**场景 A：弹层内提交按钮 → 关闭弹层**

```
用户点击弹层内「提交」按钮
  → 按钮属于弹层内 Fragment 的 Widget
  → 先执行弹层 Fragment 重跑
  → 按钮回调中执行 st.rerun()
  → st.rerun() 抛出 RerunException，向上冒泡到脚本运行器
  → 脚本运行器中断当前 Fragment 执行，调度一次完整脚本重跑
  → 完整脚本重跑时，外层 if 条件（如 st.button 状态）通常已不满足
  → 弹层函数不被调用，弹层自然消失
```

**场景 B：点击弹层遮罩 → on_dismiss="rerun"**

```
用户点击弹层外遮罩
  → 前端 Dialog 组件执行 handleClose
  → 检查到 dialog.id 存在（因为 on_dismiss≠ignore）
  → 通过 WidgetStateManager.setTriggerValue 写入 trigger_value=true
  → 前端发送 updateWidgets 消息
  → 后端识别该 Widget 属于顶层（fragment_id 为空或不匹配任何 Fragment）
  → 执行完整脚本重跑
  → 若脚本中弹层打开条件未变，则弹层可能再次打开（这是用户需要避免的常见 bug）
```

> 场景 B 提示一个重要的工程惯例：如果 `on_dismiss` 设为 `"rerun"` 或回调，通常需要配合 Session State 中的「弹层已关闭」标记来防止重跑后条件判断仍为真，导致弹层立即重新弹出。

---

### 7.2 弹层内容多次重跑时保留哪些状态

弹层的重跑分为「同一次打开期间内的多次 Fragment 重跑」和「跨越完整脚本重跑的再次打开」两种情况，状态保留规则截然不同。

#### 情况一：同一次打开期间的 Fragment 重跑

这是最常见的场景：用户在弹层内操作 Widget（如输入文字、切换开关），弹层持续打开，内容函数被反复执行。此时保留的状态分为五类：

| 状态类别 | 是否保留 | 保留机制 |
|---|---|---|
| **弹层内 Widget 的值** | ✅ 保留 | 标准的 Widget State 机制。Widget ID 由「弹层稳定 ID + 写入位置」共同确定，Fragment 级重跑不会清空 Widget 值。输入框中已输入的文字、滑块位置、开关状态全部保留。 |
| **st.session_state（全局会话状态）** | ✅ 保留 | Session State 是会话级全局存储，任何层级的重跑都不影响。弹层内对 Session State 的写入对主页面立即可见（重跑后主页面能读到）。 |
| **Fragment 定义时的游标快照** | ✅ 保留 | Fragment 在首次注册时对 cursors（元素写入游标）和 dg_stack（容器栈）做深拷贝。每次 Fragment 重跑时用快照恢复，确保第 N 次重跑的元素写入位置与第 1 次完全一致，不会因主页面元素增减而错位。 |
| **弹层内的局部变量** | ❌ 不保留 | 每次 Fragment 重跑都是从头执行内容函数，函数内部声明的普通变量（`x = 1` 之类）每次都重新初始化。需要跨重跑持久化请使用 `st.session_state` 或 `st.cache_data`。 |
| **弹层的打开/关闭状态（前端 isOpen）** | ✅ 保留 | 前端用 React `useState` 持有 `isOpen`。Fragment 级重跑只更新 `children`（弹层内容），不卸载 Dialog 组件本体，因此 `isOpen` 保持为 `true`，弹层不会意外关闭。 |

#### 情况二：完整脚本重跑后再次打开弹层

用户完成交互后关闭弹层，触发完整脚本重跑，之后再次点击按钮打开同一个弹层。此时：

| 状态类别 | 是否保留 | 说明 |
|---|---|---|
| **弹层内 Widget 的值** | ❌ 不保留 | 完整脚本重跑会执行 `_reset_triggers` 重置所有 trigger_value，但更关键的是：若弹层函数未被调用，则弹层内的 Widget 根本不会被注册到当前脚本运行的活跃 Widget 集合中。下次打开时 Widget 的值由 Session State 中的初始值或 `value` 参数决定。 |
| **st.session_state** | ✅ 保留 | 会话级全局存储，与弹层是否打开无关。如果在弹层关闭前将重要值写入 Session State，下次打开时可恢复。 |
| **Fragment Storage 中的函数定义** | ✅ 保留 | 完整脚本重跑会清理部分 Fragment，但如果弹层函数在完整脚本中再次被执行（同一段装饰器代码），Fragment 会重新注册。若装饰器代码位置相同、调用栈相同，则 fragment_id 相同，注册覆盖旧条目。 |
| **前端已有的弹层 DOM 节点** | ✅ 选择性保留 | 如果新弹层的稳定 ID（block.id）与旧弹层相同 → 继承已有子节点（React state 保留）。如果稳定 ID 不同 → 卸载旧弹层 DOM，创建全新弹层。这是防止「串弹层」的关键机制（详见 7.3 节）。 |

#### 状态保留中的陷阱：非 dismissible + Widget 缓存

如果一个弹层设置 `dismissible=False`（不可手动关闭），且用户在弹层内操作 Widget 触发 Fragment 重跑，此时如果 Widget 触发了完整脚本重跑（例如按钮回调里调了 `st.rerun()`），需要注意：

1. 完整脚本重跑会将 `has_dialog_opened` 标志重置为 `False`，因此新脚本运行中可以再次打开弹层。
2. 但如果弹层打开条件仍为真，**新弹层和旧弹层可能拥有相同的稳定 ID**，这意味着前端会复用旧弹层的 DOM 节点。如果弹层内部依赖「首次打开时初始化某些局部状态」，这些状态不会被重新初始化。
3. 解决方案是在每次弹层打开时显式重置依赖的状态，或利用不同参数生成不同的弹层稳定 ID（通过 title / icon / on_dismiss 等参数的差异化）。

---

### 7.3 如何避免旧弹层内容串到主页面

「串弹层」指以下两种异常表现：

- **类型 A**：打开弹层 B 时，看到弹层 A 的残留内容（widget、文本、图表等）
- **类型 B**：弹层关闭后，主页面出现本该在弹层内的元素（主页面「串」进了弹层内容）

这两类异常的根本原因和防护机制完全不同。

#### 类型 A：弹层之间互相串内容 —— 机制与防护

**根本原因**：前端渲染树在替换 Block 节点时，默认会**继承同类型旧节点的子节点**，以保留 React 组件内部状态（如未提交的输入、折叠状态等）。如果没有弹层身份区分，打开新弹层时前端不会清空旧弹层的 children，直接复用旧 DOM，导致内容混杂。

**防护机制：弹层稳定 ID （block.id）的身份比对**

每个弹层在创建时都会根据自身属性（title、dismissible、width、icon、on_dismiss 配置）计算一个稳定的哈希 ID，并写入 BlockProto 的顶层 `id` 字段。前端在替换 dialog block 时执行如下判断逻辑：

```
收到新的 dialog Block：
  1. 在渲染树中定位 delta_path 相同位置的旧节点
  2. 判断「旧节点也是 dialog 类型」且「新旧 block.id 不同」？
     ├─ 是（身份不同的弹层）→ 不继承子节点，children = [] 新建
     └─ 否（同身份弹层更新）→ 继承旧节点的 children，保留 React state
```

**设计意图**：稳定 ID 相同说明是「同一个弹层的内容刷新」（如 Fragment 重跑更新内容），此时保留子节点可以避免 Widget 值丢失、减少 DOM 重建闪烁。稳定 ID 不同说明是「不同弹层的切换」，必须彻底清空才能避免串内容。

**典型风险场景**：如果开发者用完全相同的标题、配置依次打开两个不同的弹层（例如 A 和 B 都是 `@st.dialog("确认")`，默认 small、dismissible=True），则两者的稳定 ID 完全相同，前端会误认为是同弹层更新而串内容。这种情况需要通过 icon 或 on_dismiss 参数人为区分，或在逻辑上确保不会交替出现配置完全相同的不同弹层。

#### 类型 B：弹层内容串到主页面 —— 机制与防护

**根本原因**：弹层内容在后端是通过 DeltaGenerator 的「写入游标（cursor）」定位到 EVENT 容器中的某个 delta_path。如果在完整脚本重跑或 Fragment 重跑时，cursor 的起始位置没有正确恢复，弹层内容的 delta_path 可能落到 MAIN 容器（主页面）范围内，前端就会将其渲染在主页面中。

**防护机制：四层防线**

**防线 1 — 独立容器（EVENT 根容器）**

弹层永远写入 EVENT 容器（delta_path 第一个索引为 2，而非 MAIN 的 0 或 SIDEBAR 的 1），在渲染树层级上与主页面是兄弟节点，天然不可能交叉进入主页面的 DOM 结构中。即使前端渲染出错，也是 EVENT 容器下的内容错乱，不会影响 MAIN 容器。

**防线 2 — Fragment 注册时的游标快照**

弹层内容函数在首次注册为 Fragment 时，对 ScriptRunContext 的 `cursors` 字典（按 root_container 索引的游标集合）和 `context_dg_stack` 做深拷贝。每次 Fragment 重跑时，在执行内容函数之前先执行：

```
ctx.cursors = deepcopy(快照中的 cursors)
context_dg_stack.set(deepcopy(快照中的 dg_stack))
```

这确保了无论主页面在上一次完整脚本运行后增加或减少了多少元素，弹层 Fragment 内部写入元素时的起始位置始终与首次注册时一致，不会因主页面游标偏移而错位写入其他容器。

**防线 3 — 弹层注册位置绑定 event_dg**

`@st.dialog` 装饰器内部显式使用 `event_dg._dialog(...)` 创建弹层 Block，而非用户当前上下文的 dg。即使用户在 `with st.sidebar:` 内部调用弹层函数（当前 dg_stack 指向 SIDEBAR 容器），弹层 Block 仍会写入 EVENT 容器。这从源头上杜绝了「在 Sidebar 里调用的弹层写入 Sidebar 区域」的可能。

**防线 4 — 完整脚本重跑时的容器清理**

每次完整脚本重跑时，前端渲染树对 EVENT 容器的处理与 MAIN、SIDEBAR、BOTTOM 容器完全对称：
- `clearStaleNodes`：按 scriptRunId 标记并移除不在当前脚本运行中的节点
- `clearTransientNodes`：清理瞬时元素（Toast、Balloon 等）
- `filterMainScriptElements`：页面切换时保留/过滤对应脚本哈希的元素

即使某次 Fragment 重跑出现异常写入（例如游标计算 bug），下一次完整脚本重跑也会把不属于当前运行的节点全部清掉，不会永久累积脏数据。

#### 串弹层异常的排查思路

如果实际观察到了串内容现象，可以按以下路径定位：

1. **类型 A（弹层互串）**：检查两个弹层是否配置完全相同 → 尝试在 title 末尾加空格或设置不同 icon，确认稳定 ID 是否区分开
2. **类型 B（串到主页面）**：在弹层函数内打印 `st.container()` 的类型信息，确认 dg_stack 顶层容器是否为 EVENT → 检查是否有代码在弹层内修改了全局 cursors（不允许的操作）
3. **Fragment 重跑异常**：对比完整脚本重跑与 Fragment 重跑时，主页面元素的 delta_path 是否变化 → 如果主页面每次重跑后 delta_path 不同，说明有条件性渲染在主页面产生了不确定元素数，需要固定渲染顺序

---

> 三个边界问题的统一心智模型：把弹层想象为「附着在 EVENT 容器上的独立小窗口」，窗口内部有自己的状态空间（Fragment），窗口之间靠身份牌（稳定 ID）区分，窗口与主页面之间靠物理边界（EVENT vs MAIN 容器）隔离。任何「串」类异常，本质上都是身份牌被混淆或物理边界被突破。
