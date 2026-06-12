# st.form 批量提交模型代码协作分析

## 阅读说明

- 本文档分析 Streamlit `st.form` 批量提交模型的前后端协作机制,涵盖表单收集、批量提交、校验反馈、局部更新四大阶段。
- 正文中的源码引用统一指向**文末第九章的代码索引**,使用「索引名称」标注,点击文末对应条目可直接跳转到源码行。
- 代码块中的 `# 文件名 - 函数名 第X行` 为上下文注释,非可点击链接。

---

## 一、整体架构概览

st.form 的批量提交模型是一个跨前后端的完整协作体系,核心围绕 **"表单收集 → 批量提交 → 校验反馈 → 局部更新"** 四个阶段展开。

```
┌─────────────────────────────────────────────────────────────────┐
│                        Python 后端                              │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │ form.py  │    │ form_utils.py│    │ session_state.py     │  │
│  │ FormMixin│    │ FormData     │    │ register_widget()    │  │
│  └─────┬────┘    └──────┬───────┘    └──────────┬───────────┘  │
│        │                │                       │              │
│        └────────────────┴───────────┬───────────┘              │
│                                      │                          │
│                             Protobuf 消息传递                   │
└──────────────────────────────────────┼──────────────────────────┘
                                       │
┌──────────────────────────────────────┼──────────────────────────┐
│                        前端 (React + TypeScript)                  │
│  ┌───────────────────────────────────▼───────────────────────┐   │
│  │                    WidgetStateManager                     │   │
│  │  - widgetStates (全局状态)                                │   │
│  │  - forms Map<fromId, FormState>                          │   │
│  │  - FormsData (不可变, 用于 React 订阅)                    │   │
│  └──────┬───────────────────┬───────────────────────────────┘   │
│         │                   │                                   │
│  ┌──────▼──────┐    ┌──────▼───────┐    ┌───────────────────┐  │
│  │   Form.tsx  │    │FormSubmitBtn │    │ 各 Widget 组件     │  │
│  │ 容器+校验   │    │ 提交触发器    │    │ TextInput/Slider.. │  │
│  └─────────────┘    └──────────────┘    └───────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、表单收集:Widget 如何归集到 Form

### 2.1 后端:Form 容器与 Widget 归属

> 对应代码索引:「Form 容器创建」「Form 归属判断」「Submit Button 创建+注册」,详见文末第九章

#### Form 容器的创建

`st.form(key)` 创建一个表单容器,核心机制:

```python
# form.py - FormMixin.form()
form_id = key
block_dg = self.dg._block(block_proto)
block_dg._form_data = FormData(form_id)  # 附加到 DeltaGenerator
return block_dg
```

`FormData` 是一个简单的命名元组,存储在 `DeltaGenerator._form_data` 上:

```python
# form_utils.py
class FormData(NamedTuple):
    form_id: str
```

#### Widget 归属判断

每个 widget 创建时通过 `current_form_id(dg)` 判断自己是否在表单内:

```python
# form_utils.py - _current_form()
def _current_form(this_dg: DeltaGenerator) -> FormData | None:
    # 方式1: 直接检查 dg._form_data
    if this_dg._form_data is not None:
        return this_dg._form_data

    # 方式2: 通过 st.xxx 调用时,遍历 context_dg_stack 向上查找
    if this_dg == this_dg._main_dg:
        for dg in reversed(context_dg_stack.get()):
            if dg._form_data is not None:
                return dg._form_data
    # 方式3: 通过 dg.xxx 调用时,检查 parent
    else:
        parent = this_dg._parent
        if parent is not None and parent._form_data is not None:
            return parent._form_data
```

> **关键设计:** 表单归属通过 DeltaGenerator 链向上查找,支持嵌套容器(columns、expander 等)内的 widget 自动归属于外层 form。

#### Widget 的 form_id 注入

每个 widget 的 protobuf 消息中都会携带 `form_id` 字段,以 button 为例:

```python
# button.py - _button()
form_id = current_form_id(self.dg) if is_form_submitter else ""
button_proto.form_id = form_id
```

非 submitter 的 widget 也会在 proto 中携带 form_id,用于前端识别归属。

### 2.2 前端:WidgetStateManager 的双状态存储

> 对应代码索引:「表单值写入分流」「值读取(表单优先)」,详见文末第九章

前端通过 `WidgetStateManager` 管理所有 widget 状态,采用 **双层存储** 设计:

```
WidgetStateManager
├── widgetStates: WidgetStateDict      # 全局已提交状态
└── forms: Map<formId, FormState>      # 各表单的待提交状态
    └── FormState
        ├── widgetStates: WidgetStateDict  # 表单内待提交值
        ├── clearOnSubmit: boolean
        ├── enterToSubmit: boolean
        └── formCleared: Signal           # 表单清除信号
```

#### Widget 值的写入分流

widget 值更新时,根据 `formId` 和 `source.fromUi` 决定写入位置:

```typescript
// WidgetStateManager.ts - createWidgetState() 第873行
private createWidgetState(widget: WidgetInfo, source: Source): WidgetState {
  const addToForm = isValidFormId(widget.formId) && source.fromUi
  const widgetStateDict = addToForm
    ? this.getOrCreateFormState(widget.formId as string).widgetStates
    : this.widgetStates

  return widgetStateDict.createState(widget.id)
}
```

> **核心规则:**
> - **用户交互产生的值** (`fromUi=true`) 且属于表单 → 写入 `form.widgetStates`(待提交区)
> - **服务器推送的值** (`fromUi=false`) → 直接写入全局 `widgetStates`
> - **不属于表单的 widget** → 直接写入全局并触发 rerun

这种设计确保了表单内的用户输入不会立即触发脚本重运行,实现了"批量收集"。

---

## 三、批量提交:从点击到后端执行

### 3.1 提交触发点

> 对应代码索引:「提交按钮组件」「Enter 键提交 Hook」,详见文末第九章

#### 方式1:点击提交按钮

```typescript
// FormSubmitButton.tsx - handleSubmit()
const handleSubmit = useCallback((): void => {
  if (isDisabled) return
  widgetMgr.submitForm(element.formId, fragmentId, element)
}, [isDisabled, widgetMgr, element, fragmentId])
```

#### 方式2:回车键提交

表单内的输入控件(TextInput、NumberInput 等)按 Enter 键可触发提交:

```typescript
// useSubmitFormViaEnterKey.ts
if (widgetMgr.allowFormEnterToSubmit(formId)) {
  widgetMgr.submitForm(formId, fragmentId)
}
```

### 3.2 submitForm 核心流程

> 对应代码索引:「submitForm 核心流程」,详见文末第九章

`submitForm` 是批量提交的核心方法,执行以下关键步骤:

```typescript
// WidgetStateManager.ts - submitForm()
public submitForm(
  formId: string,
  fragmentId: string | undefined,
  actualSubmitButton?: WidgetInfo
): void {
  const form = this.getOrCreateFormState(formId)
  
  // 步骤1: 确定被点击的 submit button
  let selectedSubmitButton
  if (actualSubmitButton !== undefined) {
    selectedSubmitButton = actualSubmitButton
  } else if (submitButtons?.length > 0) {
    selectedSubmitButton = submitButtons[0]  // Enter 键时用第一个
  }
  
  // 步骤2: 设置 submit button 的 trigger 值
  if (selectedSubmitButton) {
    this.createWidgetState(selectedSubmitButton, { fromUi: true }).triggerValue = true
  }
  
  // 步骤3: 将表单待提交值拷贝到全局状态
  this.widgetStates.copyFrom(form.widgetStates)
  form.widgetStates.clear()
  
  // 步骤4: 发送 rerun 消息到后端
  this.sendUpdateWidgetsMessage(fragmentId)
  this.syncFormsWithPendingChanges()
  
  // 步骤5: 清理 submit button 的 trigger 状态
  if (selectedSubmitButton) {
    this.deleteWidgetState(selectedSubmitButton.id)
  }
  
  // 步骤6: clear_on_submit 时触发清除信号
  if (form.clearOnSubmit) {
    form.formCleared.emit()
  }
}
```

### 3.3 消息批处理: scheduleFlush 机制

除了表单提交外,非表单 widget 的值变更也会被批处理:

```typescript
// WidgetStateManager.ts - onWidgetValueChanged() 第801行
private onWidgetValueChanged(
  formId: string | undefined,
  source: Source,
  fragmentId: string | undefined
): void {
  if (isValidFormId(formId)) {
    this.syncFormsWithPendingChanges()  // 仅更新 pending 标记
  } else if (source.fromUi) {
    this.scheduleFlush(fragmentId)      // 调度下一个 macrotask 发送
  }
}
```

`scheduleFlush` 使用 `setTimeout(..., 0)` 将同一 JavaScript 事件循环周期内的多次 widget 更新合并为一次消息:

```typescript
// WidgetStateManager.ts - scheduleFlush() 第1071行
private scheduleFlush(fragmentId: string | undefined): void {
  if (this.flushScheduled) return  // 已调度则跳过
  
  this.flushScheduled = true
  setTimeout(() => {
    this.sendUpdateWidgetsMessage(this.scheduledFragmentId)
    // 清理 trigger 状态、resolver 等
    this.flushScheduled = false
    this.scheduledFragmentId = undefined
  }, 0)
}
```

> **批处理策略:**
> - **表单内 widget**: 不触发 rerun,累积到 form.widgetStates
> - **非表单 widget**: 通过 scheduleFlush 合并同一 macrotask 内的多次变更
> - **表单提交**: 立即合并到全局状态并发送消息

### 3.4 后端接收与处理

后端通过 `session_state` 接收批量提交的 widget 状态:

```python
# session_state.py - register_widget()
# 每个 widget 在脚本执行时通过 register_widget 注册并获取当前值
```

`FormSubmitButton` 的值是 `trigger_value` 类型(一次性触发器),后端收到后执行 `on_click` 回调,然后返回新的页面状态。

---

## 3.5 后端状态恢复与脚本重跑完整链路

> 对应代码索引:「AppSession 接收 rerun」「后端脚本主循环」「状态同步+回调触发入口」「回调调度(两条路径)」,详见文末第九章

当后端收到前端发来的 rerun 消息(带 widget_states)后,完整执行链路如下:

#### Step 1: AppSession 接收消息并分发

```python
# app_session.py - request_rerun() 第420行
def request_rerun(self, client_state: ClientState | None = None) -> None:
    # 从 client_state 中提取 widget_states、fragment_id 等
    fragment_id = client_state.fragment_id
    
    # fragment 存在性预检:防止被全量 rerun 清理后产生孤儿 ScriptRunner
    if fragment_id and not self._fragment_storage.contains(fragment_id):
        return  # 静默丢弃

    rerun_data = RerunData(
        widget_states=client_state.widget_states,
        fragment_id=fragment_id or None,
        # ... query_string, page_script_hash 等
    )
    
    # 决定是复用现有 ScriptRunner 还是新建
    if self._scriptrunner is not None:
        if fastReruns and not rerun_data.fragment_id:
            self._scriptrunner.request_stop()  # 打断当前,新建 ScriptRunner
            self._scriptrunner = None
        else:
            success = self._scriptrunner.request_rerun(rerun_data)
            if success: return
    
    self._create_scriptrunner(rerun_data)
```

#### Step 2: ScriptRunner._run_script() 主循环

```python
# script_runner.py - _run_script() 第528行
def _run_script(self, rerun_data: RerunData) -> None:
    while True:
        # 页面切换时的 widget 清理(略)
        # ...
        
        # 重置 ScriptRunContext,注入 fragment_ids_this_run
        if rerun_data.fragment_id_queue:
            fragment_ids_this_run = self._fragment_storage.order_fragment_ids(
                rerun_data.fragment_id_queue
            )
        ctx.reset(fragment_ids_this_run=fragment_ids_this_run, ...)
        
        # 编译脚本,准备执行环境
        code = self._script_cache.get_bytecode(script_path)
        module = self._new_module("__main__")
        sys.modules["__main__"] = module
        
        # ============ 核心:code_to_exec 闭包 ============
        def code_to_exec(...) -> None:
            with modified_sys_path(self._main_script_path), self._set_execing_flag():
                # ★★★ 关键1: widget 状态同步 + 回调触发(在脚本执行之前)
                if rerun_data.widget_states is not None:
                    self._session_state.on_script_will_rerun(
                        rerun_data.widget_states
                    )
                
                ctx.on_script_start()
                
                # ★★★ 关键2: 选择执行范围 —— 全量脚本 or fragment(s)
                if fragment_ids_this_run:
                    # --- Fragment 局部重跑分支 ---
                    for fragment_id in fragment_ids_this_run:
                        wrapped_fragment = self._fragment_storage.lookup(fragment_id)
                        try:
                            wrapped_fragment()  # 只执行 fragment 函数体
                        finally:
                            # 清理该 fragment 下的过时嵌套 fragment
                            registered_ids = self._fragment_storage.ids_registered_after(...)
                            self._fragment_storage.clear_stale_descendants(
                                fragment_id, registered_ids
                            )
                else:
                    # --- 全量脚本重跑分支 ---
                    if PagesManager.uses_pages_directory:
                        _mpa_v1(self._main_script_path)
                    else:
                        exec(code, module.__dict__)  # 执行整个用户脚本
                    coordinator.join()  # 等待并行 fragment 完成
                    self._fragment_storage.clear(
                        new_fragment_ids=ctx.new_fragment_ids.snapshot()
                    )
                
                self._session_state.maybe_check_serializable()
                self._maybe_handle_execution_control_request()
        
        # 执行并捕获异常
        (_, run_without_errors, rerun_exception_data, ...) = exec_func_with_error_handling(
            code_to_exec, ctx
        )
        
        # 处理 RerunException(循环继续) 或正常结束(break)
        if rerun_exception_data is not None:
            rerun_data = rerun_exception_data
        else:
            break
    
    # 脚本结束后的清理
    self._on_script_finished(ctx, finished_event, premature_stop)
```

#### Step 3: on_script_will_rerun —— 状态同步与回调触发

这是后端处理表单提交的**核心入口**,发生在用户脚本代码执行之前:

```python
# session_state.py - on_script_will_rerun() 第641行
def on_script_will_rerun(self, latest_widget_states: WidgetStatesProto) -> None:
    # 子步骤 A: 清理上次残留的 trigger 值(防止误触发)
    self._reset_triggers()
    
    # 子步骤 B: 状态压缩 —— 把 _new_widget_state 合并到 _old_state
    #         这一步让 _widget_changed() 能正确判断"值是否改变"
    self._compact_state()
    
    # 子步骤 C: 用前端发来的批量状态更新 _new_widget_state
    self.set_widgets_from_proto(latest_widget_states)
    
    # 子步骤 D: ★★★ 触发所有变更 widget 的回调(包括 submit button)
    self._call_callbacks()
```

#### Step 4: _call_callbacks —— 回调调度的两条路径

```python
# session_state.py - _call_callbacks() 第653行
def _call_callbacks(self) -> None:
    # ===== 路径 1: 单回调 (on_change / on_click 传统模式) =====
    changed_widget_ids_for_single_callback = [
        wid for wid in self._new_widget_state
        if self._widget_changed(wid)
        and metadata.callback is not None
    ]
    for wid in changed_widget_ids_for_single_callback:
        self._new_widget_state.call_callback(wid)  # 直接执行 metadata.callback
    
    # ===== 路径 2: 多回调(trigger 聚合模式,Component v2 等)=====
    for wid in list(self._new_widget_state.states.keys()):
        metadata = self._new_widget_state.widget_metadata.get(wid)
        if not metadata or metadata.callbacks is None:
            continue
        
        # 子路径 2a: trigger 调度 (bool + JSON trigger 聚合器)
        self._dispatch_trigger_callbacks(wid, metadata, args, kwargs)
        
        # 子路径 2b: JSON 值变化调度(浅 diff,按 key 分别回调)
        if metadata.value_type == "json_value":
            self._dispatch_json_change_callbacks(wid, metadata, args, kwargs)
```

> **表单提交回调在哪里触发?**
> FormSubmitButton 使用 `trigger_value` 类型,前端在 `submitForm()` 中把它设为 `true`。
> 由于 `on_click` 注册在 `metadata.callback`(单回调字段),它通过**路径 1**触发:
> `_widget_changed()` 检测到值从 `False → True`,然后 `call_callback()` 直接执行 `metadata.callback`。
> `_dispatch_trigger_callbacks` 只处理 `json_trigger_value`(Component v2 多事件),不处理 bool 类型的 `trigger_value`。
> 详见第八章 8.1 节深度分析。

#### Step 5: on_script_finished —— 运行后清理

```python
# session_state.py - on_script_finished() 第866行
def on_script_finished(self, widget_ids_this_run: frozenset[str]) -> None:
    # A: 重置所有 trigger 值为 false / null
    #    确保 trigger 是一次性的,下次脚本运行不会重复触发
    self._reset_triggers()
    
    # B: 删除未访问的 widget 状态(页面切换 / 条件渲染)
    self._remove_stale_widgets(widget_ids_this_run)
```

---

## 3.6 回车提交禁用判断的完整链路

回车提交的判断完全在**前端**完成,后端不参与。涉及两个关键调用点:

#### 判断一:useSubmitFormViaEnterKey Hook(提交前拦截)

```typescript
// useSubmitFormViaEnterKey.ts - 默认导出 第39行
export default function useSubmitFormViaEnterKey(
  formId: string,
  commitWidgetValue: () => void,       // widget 内部的 commit 函数
  callCommitWidgetValue: boolean,      // 是否需要先 commit(如 dirty 检查)
  widgetMgr: WidgetStateManager,
  fragmentId?: string,
  requireCommandKey = false            // 某些 widget 如 Textarea 需要 Cmd+Enter
): (e: SubmitFormKeyboardEvent) => void {
  return useCallback(
    (e: SubmitFormKeyboardEvent): void => {
      // 检查 1: 键位校验(Enter 键 + 可选 Cmd/Ctrl)
      const isCommandKeyPressed = requireCommandKey
        ? e.metaKey || e.ctrlKey
        : true
      if (!isEnterKeyPressed(e) || !isCommandKeyPressed) {
        return
      }

      e.preventDefault()

      // 检查 2: 是否需要先 commit 当前输入值(如 TextInput 的 dirty 检查)
      if (callCommitWidgetValue) {
        commitWidgetValue()
      }

      // 检查 3: 表单级规则校验(allowFormEnterToSubmit)
      if (widgetMgr.allowFormEnterToSubmit(formId)) {
        widgetMgr.submitForm(formId, fragmentId)
      }
    },
    [
      formId,
      fragmentId,
      callCommitWidgetValue,
      commitWidgetValue,
      widgetMgr,
      requireCommandKey,
    ]
  )
}
```

> **注意:** `useSubmitFormViaEnterKey` **没有** `widgetProps.disabled` 参数,也不做 widget 自身禁用检查。
> 如果 widget 被禁用,浏览器会直接阻止 `keydown` 事件触发,事件根本不会到达该 Hook。

#### 判断二:allowFormEnterToSubmit(表单级规则)

```typescript
// WidgetStateManager.ts - allowFormEnterToSubmit() 第932行
public allowFormEnterToSubmit(formId: string): boolean {
  // 层级 1: 必须属于有效表单
  if (!isValidFormId(formId)) return false
  
  // 层级 2: 用户显式关闭 enterToSubmit (st.form(..., enter_to_submit=False))
  const form = this.forms.get(formId)
  if (form && !form.enterToSubmit) return false
  
  // 层级 3: 默认规则 —— 第一个 submit button 不能是 disabled
  const firstSubmitButton = this.formsData.submitButtons.get(formId)?.[0]
  if (!firstSubmitButton) return false  // 没有 submit button → 不允许
  return !firstSubmitButton.disabled
}
```

> **判断优先级(从高到低,均为 JS 代码层实际判断):**
> 1. Enter 键未按下 或 Cmd/Ctrl 不满足 `requireCommandKey` → 拦截(Hook 入口)
> 2. 表单显式 `enter_to_submit=False` → 拦截(allowFormEnterToSubmit 层级 2)
> 3. 表单无任何 submit button → 拦截(allowFormEnterToSubmit 层级 3)
> 4. 第一个 submit button `disabled=true` → 拦截(allowFormEnterToSubmit 层级 4)
> 5. 全部通过 → 允许回车提交
>
> **补充说明:** 输入 widget 自身 `disabled` 不在任何 JS 代码中判断。浏览器原生禁用的元素不会派发 `keydown` 事件,Hook 根本不会被调用。

---

## 3.7 提交回调、clear_on_submit、fragment 局部重跑的执行顺序

这是整个表单机制最关键的时序,横跨前后端和同步/异步边界。

### 完整时序(以"fragment 内的 clear_on_submit 表单"为例)

```
 时间轴   前端(JS 主线程)                    后端(Python 脚本线程)
  │
  │  T0  用户点击 submit button
  │─────┐
  │     ▼
  │  T1  FormSubmitButton.handleSubmit()
  │      └─ widgetMgr.submitForm(formId, fragmentId, element)
  │
  │  T2  submitForm() 内部步骤(全部同步)
  │      ├─ ① 确定 selectedSubmitButton = element
  │      ├─ ② createWidgetState(submitBtn, {fromUi:true}).triggerValue = true
  │      ├─ ③ widgetStates.copyFrom(form.widgetStates)  ← 表单值合并到全局
  │      ├─ ④ form.widgetStates.clear()
  │      ├─ ⑤ sendUpdateWidgetsMessage(fragmentId)  ──────────► 发送到后端
  │      ├─ ⑥ syncFormsWithPendingChanges()
  │      ├─ ⑦ deleteWidgetState(submitBtn.id)       ← 清理前端 trigger
  │      └─ ⑧ form.clearOnSubmit ? form.formCleared.emit() : 跳过
  │
  │  T3  formCleared 信号广播(同步,但通过 useEffect 调度)
  │      └─ 各订阅 widget 的 handleFormCleared 被调用
  │          └─ setNextValueWithSource({value: default, fromUi: true})
  │              └─ useEffect → updateWidgetMgrState
  │                  └─ createWidgetState(widget, {fromUi: true})
  │                      ↳ 因为 widget.formId 有效,写入 form.widgetStates
  │                        (★ 关键:不会触发 rerun,只是暂存下次提交的默认值)
  │
  │         . . . . 网络传输: WidgetStates proto 消息 . . . . .
  │                                                     │
  │                                                     ▼
  │  T4                                    AppSession.request_rerun()
  │                                        ├─ 预检 fragment 是否存在
  │                                        └─ ScriptRunner.request_rerun()
  │
  │  T5                                    ScriptRunner._run_script()
  │                                        │
  │                                        ▼
  │  T6                                    code_to_exec 闭包执行
  │                                        ├─ A. on_script_will_rerun(widget_states)
  │                                        │   ├─ _reset_triggers()
  │                                        │   ├─ _compact_state()
  │                                        │   ├─ set_widgets_from_proto()
  │                                        │   └─ _call_callbacks()  ← ★ on_click 在此执行
  │                                        │
  │                                        ├─ B. ctx.on_script_start()
  │                                        │
  │                                        ├─ C. 选择执行范围
  │                                        │   └─ fragment_ids_this_run 非空
  │                                        │       └─ 只执行 wrapped_fragment()
  │                                        │           └─ Fragment 函数体
  │                                        │               (widget register_widget
  │                                        │                读取最新 widget 状态)
  │                                        │
  │                                        └─ D. maybe_check_serializable()
  │
  │  T7                                    on_script_finished()
  │                                        ├─ _reset_triggers()  ← 重置后端 trigger
  │                                        └─ _remove_stale_widgets()
  │
  │         . . . . 网络传输: ForwardMsg (新页面 Delta) . . . . .
  │                                                     │
  │                                                     ▼
  │  T8  前端收到新 Delta,React 重渲染
  │      ├─ widget 组件从 proto 获取新 setValue(如有)
  │      └─ 各 widget 的 defaultValue(来自 form.widgetStates)生效
  │         因为后端 fragment 重跑会重新下发所有 widget 的默认值
```

### 关键执行顺序总结

| 阶段 | 序号 | 动作 | 发生位置 | 说明 |
|------|------|------|---------|------|
| **前端提交** | ① | 设置 submit btn trigger=true | 前端同步 | `trigger_value` 是一次性信号 |
| | ② | 合并 form.widgetStates → 全局 widgetStates | 前端同步 | 这一步才让表单值参与后端计算 |
| | ③ | sendUpdateWidgetsMessage(fragmentId) | 前端同步 → 后端异步 | fragmentId 决定后端执行范围 |
| | ④ | deleteWidgetState(submitBtn) | 前端同步 | 立即清理前端 trigger |
| | ⑤ | form.formCleared.emit() | 前端同步 | clear_on_submit 才触发 |
| **Widget 重置** | ⑥ | 各 widget reset 为默认值 | 前端 useEffect(微任务) | `fromUi:true` → 写入 form.widgetStates,不触发 rerun |
| **后端状态同步** | ⑦ | _reset_triggers() | 后端脚本线程 | 清除旧 trigger,防止误触发 |
| | ⑧ | _compact_state() | 后端脚本线程 | new→old,为变更检测做准备 |
| | ⑨ | set_widgets_from_proto() | 后端脚本线程 | 写入前端提交的批量值 |
| **回调触发** | ⑩ | _call_callbacks() | 后端脚本线程 | ★ on_click / on_change 在此执行,**早于用户脚本代码** |
| **脚本执行** | ⑪ | fragment 函数 / 全脚本 | 后端脚本线程 | register_widget() 读取新状态 |
| **运行后清理** | ⑫ | _reset_triggers() | 后端脚本线程 | 确保 trigger 是一次性的 |
| | ⑬ | _remove_stale_widgets() | 后端脚本线程 | 清理条件渲染消失的 widget |
| **前端更新** | ⑭ | React 重渲染 | 前端 | 接收新 Delta,应用默认值 |

### 关键约束与设计意图

1. **回调先于脚本代码执行**: `_call_callbacks()` 在 `exec(code)` 或 `wrapped_fragment()` 之前调用。这意味着回调中对 `st.session_state` 的修改,用户脚本代码能看到。

2. **trigger 的双重清理**: 前端 `submitForm()` 立即删 + 后端 `_reset_triggers()` 前后各一次,三重保险确保 trigger 值的"一次性"语义。

3. **clear_on_submit 的写入位置**: 重置值写入 `form.widgetStates` 而非全局状态,因此不会触发额外 rerun。这些值在下一次用户输入时作为新的初始值参与收集。

4. **fragment 的执行范围**: `fragment_ids_this_run` 非空时只执行对应的 `wrapped_fragment()`,全脚本中的其余代码被完全跳过。但 `on_script_will_rerun`、`on_script_finished` 等生命周期钩子仍完整执行。

5. **fragment 存在性预检**: `AppSession.request_rerun()` 中提前检查 `fragment_storage.contains(fragment_id)`,避免全量 rerun 已清理 fragment 后产生孤儿 ScriptRunner 导致事件不完整。

---

## 四、校验反馈:表单状态的可视化反馈

### 4.1 缺失提交按钮警告

> 对应代码索引:「Form 容器组件」,详见文末第九章

Form 组件会检测表单内是否有 submit button,没有则显示错误警告:

```typescript
// Form.tsx 第68-97行
const { formsData } = useRequiredContext(FormsContext)
const submitButtons = formsData.submitButtons.get(formId)
const hasSubmitButton = submitButtons !== undefined && submitButtons.length > 0

// 脚本运行结束后才显示警告(避免运行中误报)
const { scriptRunState } = useContext(ScriptRunContext)
const scriptNotRunning = scriptRunState === ScriptRunState.NOT_RUNNING

if (!hasSubmitButton && !showWarning && scriptNotRunning) {
  setShowWarning(true)
}
```

### 4.2 提交按钮注册与 FormsContext

> 对应代码索引:「Forms 上下文」「提交按钮组件」,详见文末第九章

`FormsContext` 提供表单状态的 React 订阅机制,数据来源于 `WidgetStateManager.formsData`:

```typescript
// FormsContext.tsx
export interface FormsContextProps {
  formsData: FormsData  // 不可变数据结构
}
```

`FormsData` 是不可变对象,每次变更产生新实例:

```typescript
// WidgetStateManager.ts 第85行
export interface FormsData {
  formsWithPendingChanges: Set<string>       // 有待提交更改的表单
  formsWithUploads: Set<string>              // 有上传中的表单
  submitButtons: Map<string, SubmitButtonProto[]>  // 各表单的提交按钮
}
```

FormSubmitButton 挂载时注册自己:

```typescript
// FormSubmitButton.tsx 第60-63行
useEffect(() => {
  widgetMgr.addSubmitButton(formId, element)
  return () => widgetMgr.removeSubmitButton(formId, element)
}, [widgetMgr, formId, element])
```

### 4.3 文件上传期间禁用提交

表单内有文件上传正在进行时,submit button 自动禁用:

```typescript
// FormSubmitButton.tsx 第48-49行
const { formsData } = useRequiredContext(FormsContext)
const hasInProgressUpload = formsData.formsWithUploads.has(formId)

const isDisabled = disabled || hasInProgressUpload
```

### 4.4 Enter 键提交校验

`allowFormEnterToSubmit` 检查是否允许回车提交(仅判断表单级规则,不判断输入 widget 自身是否 disabled——后者由浏览器 DOM 层保证):

```typescript
// WidgetStateManager.ts 第932行
public allowFormEnterToSubmit(formId: string): boolean {
  if (!isValidFormId(formId)) return false
  
  // 用户显式设置了 enterToSubmit=false
  const form = this.forms.get(formId)
  if (form && !form.enterToSubmit) return false
  
  // 第一个 submit button 不能是 disabled
  const firstSubmitButton = this.formsData.submitButtons.get(formId)?.[0]
  if (!firstSubmitButton) return false
  return !firstSubmitButton.disabled
}
```

---

## 五、局部更新:清除、Fragment 与状态同步

### 5.1 clear_on_submit 机制

当表单设置了 `clear_on_submit=True` 时,提交后会自动重置所有 widget 到默认值。

#### 信号机制

```typescript
// WidgetStateManager.ts - FormState 类
class FormState {
  public readonly formCleared = new Signal()  //  typed-signals 信号
  
  // submitForm 中触发
  if (form.clearOnSubmit) {
    form.formCleared.emit()
  }
}
```

#### Widget 端的响应

每个 widget 通过 `useFormClearHelper` hook 订阅清除信号:

```typescript
// FormClearHelper.ts - useFormClearHelper()
export function useFormClearHelper({
  element,
  widgetMgr,
  onFormCleared,
}: FormClearHelperArgs): void {
  useEffect(() => {
    if (!widgetMgr || !isValidFormId(element.formId)) return
    
    const formClearListener = widgetMgr.addFormClearedListener(
      element.formId,
      onFormCleared
    )
    return () => formClearListener.disconnect()
  }, [element, widgetMgr, onFormCleared])
}
```

在 `useBasicWidgetState` 中,清除时重置为默认值:

```typescript
// useBasicWidgetState.ts - handleFormCleared() 第142行
const handleFormCleared = useCallback((): void => {
  setNextValueWithSource({
    value: getDefaultState(widgetMgr, element),
    fromUi: true,
  })
  onFormCleared?.()  // 额外的本地清理回调
}, [...])
```

> **注意:** 因为 widget 在表单内,`fromUi=true` 的值更新会写入 `form.widgetStates` 而不会立即触发 rerun。但由于提交时已经触发了 rerun,所以清除后的值会在下一次渲染中体现。

### 5.2 Fragment 局部重运行

表单提交支持 `fragmentId`,实现只重运行部分代码:

```typescript
// WidgetStateManager.ts - submitForm()
this.sendUpdateWidgetsMessage(fragmentId)
```

`sendUpdateWidgetsMessage` 将 `fragmentId` 传递给后端:

```typescript
// WidgetStateManager.ts - sendUpdateWidgetsMessage() 第831行
public sendUpdateWidgetsMessage(
  fragmentId: string | undefined,
  isAutoRerun: boolean | undefined = undefined
): void {
  this.props.sendRerunBackMsg(
    this.widgetStates.createWidgetStatesMsg(),
    fragmentId,  // 传递 fragmentId
    undefined,
    isAutoRerun
  )
}
```

### 5.3 待提交状态追踪与 UI 反馈

`FormsData.formsWithPendingChanges` 追踪哪些表单有未提交的更改:

```typescript
// WidgetStateManager.ts - syncFormsWithPendingChanges() 第818行
private syncFormsWithPendingChanges(): void {
  const pendingFormIds = new Set<string>()
  this.forms.forEach((form, formId) => {
    if (form.hasPendingChanges) {
      pendingFormIds.add(formId)
    }
  })
  
  this.updateFormsData(draft => {
    draft.formsWithPendingChanges = pendingFormIds
  })
}
```

> 目前这个状态主要用于内部追踪,可扩展为"未保存更改"提示等 UI 功能。

---

## 六、完整联动时序图

以用户在表单内输入文字并点击提交为例:

```
用户操作                    前端 WidgetStateManager          后端 SessionState
  │                              │                              │
  │  输入文字                    │                              │
  ├─────────────────────────────►│                              │
  │  setStringValue(fromUi=true) │                              │
  │                              │                              │
  │  写入 form.widgetStates      │                              │
  │  (不触发 rerun)              │                              │
  │  syncFormsWithPendingChanges │                              │
  │                              │                              │
  │  点击提交按钮                │                              │
  ├─────────────────────────────►│                              │
  │  submitForm(formId)          │                              │
  │                              │                              │
  │  1. 设置 submit btn trigger  │                              │
  │  2. copy form → widgetStates │                              │
  │  3. 清空 form.widgetStates   │                              │
  │  4. sendUpdateWidgetsMsg()   │                              │
  ├────────────────────────────────────────────────────────────►│
  │                              │  WidgetStates 批量消息        │
  │                              │                              │
  │                              │  解析 widget 状态            │
  │                              │  执行 submit 回调            │
  │                              │  重运行脚本                  │
  │                              │                              │
  │◄────────────────────────────────────────────────────────────┤
  │      新页面 Delta (含结果)    │                              │
  │                              │                              │
  │  [可选] clear_on_submit      │                              │
  │  form.formCleared.emit()     │                              │
  │  各 widget 重置为默认值       │                              │
```

---

## 七、关键设计模式与技术要点

### 7.1 不可变数据 + Context 订阅

`FormsData` 采用不可变设计(immer produce),每次变更产生新对象。配合 React Context,只有真正依赖表单状态的组件(Form、FormSubmitButton)才会重渲染,避免了整个组件树的无效重渲染。

### 7.2 双层状态存储

- **全局层** `widgetStates`: 已确认、会参与后端计算的状态
- **表单层** `form.widgetStates`: 待提交、用户临时输入的状态

这种分离实现了"批量收集"的核心语义,同时保证了后端推送的值能直接反映到 UI。

### 7.3 信号模式 (Signal Pattern)

表单清除使用 `typed-signals` 实现发布订阅,而非 React 状态传递。原因:
- 表单内 widget 数量可能很多
- 清除事件是低频但需要广播的事件
- 信号模式比 Context/Props 穿透更高效

### 7.4 Macrotask 批处理

`scheduleFlush` 利用事件循环的 macrotask 特性,将同步代码块中的多次 widget 更新合并为一次后端消息,减少网络通信和后端重运行次数。

---

## 八、核心机制深度解析

### 8.1 提交按钮回调在哪个回调链中触发

表单提交按钮(`st.form_submit_button`)的回调在后端通过**两条路径**触发,取决于回调是通过 `on_click` 还是多回调 API 注册的。

#### 8.1.1 注册阶段:回调的元数据注入

用户代码:
```python
clicked = st.form_submit_button("提交", on_click=my_handler, args=(1, 2))
```

在后端 `_button()` 方法中,`on_click` 通过 `register_widget` 辅助函数写入 WidgetMetadata:

```python
# button.py - _button() 第1702行
button_state = register_widget(
    button_proto.id,
    on_change_handler=on_click,     # ← on_click 被当作 on_change_handler 传入
    args=args,
    kwargs=kwargs,
    deserializer=ButtonSerde().deserialize,
    serializer=ButtonSerde().serialize,
    value_type="trigger_value",     # ← 关键:使用 trigger_value 类型
)
```

`register_widget` 辅助函数将 `on_change_handler` 存入 `WidgetMetadata.callback`(单回调字段):

```python
# widgets.py - register_widget() 第160行
metadata = WidgetMetadata(
    element_id,
    deserializer, serializer,
    value_type=value_type,
    callback=on_change_handler,     # 单回调
    callbacks=callbacks,            # 多回调(dict)
    callback_args=args,
    callback_kwargs=kwargs,
    # ...
)
```

同时 `ButtonSerde` 的反序列化器决定了 trigger 值如何被读取:
```python
# button.py - ButtonSerde 第124行
class ButtonSerde:
    def deserialize(self, ui_value: bool | None) -> bool:
        return ui_value or False   # None 或 False → False, True → True
```

#### 8.1.2 触发阶段:两条回调路径

后端 `_call_callbacks()` 中存在两条互不重叠的路径:

```
                    _call_callbacks()
                           │
           ┌───────────────┴───────────────┐
           │                               │
    路径 1: 单回调                     路径 2: 多回调
    metadata.callback ≠ None          metadata.callbacks ≠ None
    (on_change / on_click)             (dict 形式,如 {"click": fn})
           │                               │
  ┌────────┴─────────┐           ┌─────────┴───────────┐
  │                  │           │                     │
_widget_changed     否          trigger_value?     json_value 变化?
  == True?                       == True/有值?
  │                  │           │                     │
  是                  ──          是                     是
  │                               │                     │
call_callback(wid)     _dispatch_trigger_callbacks  _dispatch_json_change_callbacks
  │                               │
  执行 metadata.callback()        │
                                  ▼
                        对 bool trigger_value:
                        遍历 metadata.callbacks?
                        ┌────────────────────────────────┐
                        │ 注意:bool trigger_value 不     │
                        │ 通过 _dispatch_trigger_callbacks │
                        │ 执行!因为它只检查               │
                        │ json_trigger_value 字段。        │
                        └────────────────────────────────┘
```

**FormSubmitButton 的实际触发路径**是 **路径 1**(单回调路径):

```python
# session_state.py - _call_callbacks() 第657行
changed_widget_ids_for_single_callback = [
    wid
    for wid in self._new_widget_state
    if self._widget_changed(wid)              # ← 值从 False→True
    and (metadata := ...).callback is not None # ← on_click 注册在此
]
for wid in changed_widget_ids_for_single_callback:
    self._new_widget_state.call_callback(wid)
```

`_widget_changed()` 比较 `_old_state` 和 `_new_widget_state`:
```python
# session_state.py - _widget_changed() 第857行
def _widget_changed(self, widget_id: str) -> bool:
    new_value = self._new_widget_state.get(widget_id)  # True(前端设的)
    old_value = self._old_state.get(widget_id)          # False(上一次结束时 reset 的)
    return new_value != old_value                       # True → 触发回调!
```

> **关键结论:**
> - FormSubmitButton 的 `on_click` 通过**路径 1(单回调路径)**触发,而非 `_dispatch_trigger_callbacks`。
> - 触发的前提是 `trigger_value` 从 `False` → `True`——这由前端 `submitForm()` 设为 `True`,后端 `on_script_finished()` 重置为 `False` 共同保证。
> - `_dispatch_trigger_callbacks` 只处理 `json_trigger_value`(Component v2 的多事件聚合),不处理 bool 类型的 `trigger_value`。

---

### 8.2 回车提交的实际依赖条件

回车提交的判断**完全在前端完成**,后端不参与任何校验。判断链路分为两部分:**浏览器 DOM 层前置拦截**(不在 JS 代码中)和 **JS 代码层检查**(在 Hook 及 WidgetStateManager 中)。

#### 8.2.1 完整判断链路(代码层)

```
用户按下 Enter 键
      │
      ▼
useSubmitFormViaEnterKey Hook (JS 层)
      │
      ├─ 检查 1: isEnterKeyPressed(e) ?      ← 是否真的是 Enter 键
      ├─ 检查 2: requireCommandKey ?
      │          └─ 若需要,检查 metaKey || ctrlKey
      │     └─ 否 → return,终止
      │
      ├─ e.preventDefault()
      │
      ├─ 检查 3: callCommitWidgetValue ?    ← 是否需要先 commit 输入值
      │     └─ 是 → commitWidgetValue()      ← 把当前输入写入 WidgetStateManager
      │
      └─ 检查 4: allowFormEnterToSubmit(formId) ? ← 表单级规则
            ├─ 层级 a: formId 有效 ?
            ├─ 层级 b: form.enterToSubmit != false ? ← 用户显式关闭?
            ├─ 层级 c: 表单有 >=1 个 submit button ?
            └─ 层级 d: 第一个 submit button.disabled != true ?
                  │
                  └─ 全部通过 → widgetMgr.submitForm(formId, fragmentId)
```

#### 8.2.2 代码级依赖分析

`useSubmitFormViaEnterKey` Hook 的完整实现:

```typescript
// useSubmitFormViaEnterKey.ts 第39行
export default function useSubmitFormViaEnterKey(
  formId: string,
  commitWidgetValue: () => void,       // 输入控件内部的 commit 函数
  callCommitWidgetValue: boolean,      // 是否需要先 commit(如 Textarea 不需要)
  widgetMgr: WidgetStateManager,
  fragmentId?: string,
  requireCommandKey = false            // 某些 widget 如 Textarea 需要 Cmd+Enter
): (e: SubmitFormKeyboardEvent) => void {
  return useCallback((e) => {
    // 前置:键位检查
    const isCommandKeyPressed = requireCommandKey ? e.metaKey || e.ctrlKey : true
    if (!isEnterKeyPressed(e) || !isCommandKeyPressed) return

    e.preventDefault()

    // 先 commit 当前输入值(如 TextInput 的 onBlur 语义)
    if (callCommitWidgetValue) {
      commitWidgetValue()
    }

    // 表单级校验(下面展开)
    if (widgetMgr.allowFormEnterToSubmit(formId)) {
      widgetMgr.submitForm(formId, fragmentId)
    }
  }, [...])
}
```

`allowFormEnterToSubmit` 的四层检查:

```typescript
// WidgetStateManager.ts 第932行
public allowFormEnterToSubmit(formId: string): boolean {
  // 层级 1:必须是合法表单
  if (!isValidFormId(formId)) return false

  // 层级 2:用户显式设置 enter_to_submit=False
  const form = this.forms.get(formId)
  if (form && !form.enterToSubmit) return false

  // 层级 3:表单必须有 submit button
  const firstSubmitButton = this.formsData.submitButtons.get(formId)?.[0]
  if (!firstSubmitButton) return false

  // 层级 4:第一个 submit button 不能被禁用
  return !firstSubmitButton.disabled
}
```

#### 8.2.3 实际依赖条件汇总

回车提交必须**同时满足**以下所有代码层条件:

| # | 依赖条件 | 检查位置 | 失效场景 |
|---|---------|---------|---------|
| 1 | 按下 Enter 键(`keyCode=13` 或 `key="Enter")` | Hook 入口 `isEnterKeyPressed(e)` | 按了其他键 |
| 2 | 若 `requireCommandKey=true`,还需 Cmd 或 Ctrl 按下 | Hook 入口判断 | Textarea 等多行控件只响应 Cmd+Enter |
| 3 | 表单 `enter_to_submit != false`(默认 `true`) | `allowFormEnterToSubmit` 层级 2 | 用户显式 `st.form(..., enter_to_submit=False)` |
| 4 | 表单内至少有一个 `st.form_submit_button` | `allowFormEnterToSubmit` 层级 3 | 表单缺少提交按钮 |
| 5 | **DOM 顺序第一个** submit button 的 `disabled=false` | `allowFormEnterToSubmit` 层级 4 | 第一个提交按钮被禁用 |
| 6 | (可选)`callCommitWidgetValue=true` 时 commit 成功执行 | Hook 内部调用 | 当前输入值无法写入 WidgetStateManager(极少发生) |

> **特别注意:**
> - 输入 widget 自身 `disabled` **不在任何 JS 代码中判断**。浏览器原生禁用的元素不派发 `keydown` 事件,Hook 不会被调用。
> - 回车提交选择的是 **DOM 顺序第一个** submit button(由 `FormSubmitButton` 组件 `useEffect` 注册顺序决定),不一定是视觉上第一个。
> - 以上条件中,真正由 `allowFormEnterToSubmit()` 函数内部判断的只有 #3、#4、#5 三条。

---

### 8.3 clear_on_submit 的默认值如何恢复到控件返回值

`clear_on_submit=True` 时,表单提交后所有 widget 重置为默认值。这个过程涉及**前端信号重置**和**后端状态回流**两个阶段,最终默认值通过 `register_widget()` 返回到用户代码。

#### 8.3.1 阶段一:前端信号重置(提交后立即同步执行)

```
submitForm() 触发
       │
       ▼
form.clearOnSubmit == true ?
       │
       └─ 是 → form.formCleared.emit()   ← typed-signals 广播
                │
                ▼
        各订阅 widget 的 handleFormCleared
        (通过 useFormClearHelper Hook 注册)
                │
                ▼
        setNextValueWithSource({
          value: getDefaultState(widgetMgr, element),
          fromUi: true
        })
                │
                ▼
        useEffect → updateWidgetMgrState()
                │
                ▼
        widgetMgr.createWidgetState(widget, { fromUi: true })
                │
                └─ 因为 widget.formId 有效 → 写入 form.widgetStates
                   (不会触发 rerun,只是暂存)
```

关键代码:
```typescript
// useBasicWidgetState.ts - handleFormCleared 第142行
const handleFormCleared = useCallback((): void => {
  setNextValueWithSource({
    value: getDefaultState(widgetMgr, element),  // 获取默认值
    fromUi: true,                                 // 标记为用户产生
  })
  onFormCleared?.()
}, [...])
```

`getDefaultState` 的优先级(后端 setValue > proto default):
```typescript
// useBasicWidgetState.ts - getDefaultState 第279行
const getDefaultState = useCallback((_wm, el) => {
  if (el.setValue) {
    return getCurrStateFromProto(el)   // 后端通过 setValue 强制设置的值
  }
  return getDefaultStateFromProto(el)  // proto 中的 default 字段
}, [getDefaultStateFromProto, getCurrStateFromProto])
```

#### 8.3.2 阶段二:WidgetStateManager 的值读取优先级

前端 WidgetStateManager 读取 widget 值时,表单内的值优先从 `form.widgetStates` 读取:

```typescript
// WidgetStateManager.ts - getWidgetState() 第885行
private getWidgetState(widget: WidgetInfo): WidgetState | undefined {
  // ★ 表单内 widget 先读 form.widgetStates,再回退到全局
  if (isValidFormId(widget.formId)) {
    const formState = this.forms
      .get(widget.formId)
      ?.widgetStates.getState(widget.id)
    if (notNullOrUndefined(formState)) {
      return formState   // 刚被 clear_on_submit 重置的默认值在这里
    }
  }
  return this.widgetStates.getState(widget.id)  // 全局已提交值
}
```

这样,clear_on_submit 重置后的值立即对前端 UI 可见。

#### 8.3.3 阶段三:后端回流 —— register_widget 返回默认值

后端脚本重跑(无论是 fragment 还是全量)时,每个 widget 都通过 `register_widget()` 注册并获取返回值:

```python
# session_state.py - register_widget() 第998行
def register_widget(self, metadata, user_key):
    widget_id = metadata.id
    self._set_widget_metadata(metadata)
    
    # 关键分支:widget 是否已经在 state 中?
    if (
        widget_id not in self
        and (user_key is None or user_key not in self)
        and not url_value_seeded
    ):
        # 第一次注册 → 保存默认值
        initial_widget_value = deepcopy(metadata.deserializer(None))  # ← 默认值
        self._new_widget_state.set_from_value(widget_id, initial_widget_value)
    
    # 从当前 state 中读取值(可能是用户提交的值,也可能是默认值)
    widget_value = cast("T", self[widget_id])
    widget_value = deepcopy(widget_value)
    
    return RegisterWidgetResult(widget_value, widget_value_changed)
```

**clear_on_submit 后默认值恢复的两种情形:**

| 情形 | 触发条件 | 默认值来源 |
|------|---------|-----------|
| A | 非 fragment 表单,全脚本重跑 | `metadata.deserializer(None)`,即 widget 自身定义的默认值 |
| B | fragment 内表单,fragment 局部重跑 | 同上,但只执行 fragment 内的 register_widget |

> **关键洞察:**
> `clear_on_submit` 的前端重置写入 `form.widgetStates`,保证用户**立即可见** UI 已清空;
> 后端 `register_widget()` 在下一次脚本运行时重新从默认值初始化,保证**代码返回值**与 UI 一致。
> 这两条路径缺一不可:前者负责 UX,后者保证 Python 代码拿到正确值。

---

## 九、关键代码索引

> 说明:表格中"代码位置"列使用 `文件:行号` 格式,点击即可跳转到对应源码。

| 功能模块 | 代码位置 | 说明 |
|---------|---------|------|
| **表单容器与归属** | | |
| Form 容器创建 | [form.py:76](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/form.py#L76-L76) | `FormMixin.form()` 创建 `FormData` 并附加到 DeltaGenerator |
| Form 归属判断 | [form_utils.py:62](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/lib/form_utils.py#L62-L62) | `current_form_id()` 沿 DG 链向上查找 form_id |
| **提交按钮** | | |
| Submit Button 创建+注册 | [button.py:1616](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/widgets/button.py#L1616-L1616) | `_button()` 设置 `trigger_value` 类型,on_click 写入 metadata |
| Button 序列化/反序列化 | [button.py:124](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/widgets/button.py#L124-L124) | `ButtonSerde`,None/False → False,True → True |
| register_widget 辅助函数 | [widgets.py:42](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/widgets.py#L42-L42) | 构造 WidgetMetadata,写入 callback/callbacks |
| **前端状态管理** | | |
| submitForm 核心流程 | [WidgetStateManager.ts:346](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts#L346-L346) | trigger 设置→值合并→发送消息→清理→信号广播 |
| 表单值写入分流 | [WidgetStateManager.ts:873](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts#L873-L873) | `createWidgetState()`:fromUi+form → form.widgetStates |
| 值读取(表单优先) | [WidgetStateManager.ts:885](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts#L885-L885) | `getWidgetState()` 先 form 后全局 |
| Enter 键提交校验 | [WidgetStateManager.ts:932](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts#L932-L932) | `allowFormEnterToSubmit()` 四层判断 |
| Macrotask 批处理调度 | [WidgetStateManager.ts:1071](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts#L1071-L1071) | `scheduleFlush()` 合并同事件循环的多次更新 |
| **后端状态与回调** | | |
| AppSession 接收 rerun | [app_session.py:420](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/app_session.py#L420-L420) | `request_rerun()`:fragment 预检→分发 ScriptRunner |
| 后端脚本主循环 | [script_runner.py:528](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py#L528-L528) | `_run_script()`:on_script_will_rerun→执行脚本→on_script_finished |
| 状态同步+回调触发入口 | [session_state.py:641](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py#L641-L641) | `on_script_will_rerun()`:reset→compact→set proto→call callbacks |
| 回调调度(两条路径) | [session_state.py:653](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py#L653-L653) | `_call_callbacks()`:路径 1 单回调、路径 2 多回调 |
| 单回调执行 | [widgets.py:320](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/widgets.py#L320-L320) | `call_callback()` 直接执行 metadata.callback |
| Trigger 回调聚合 | [session_state.py:737](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py#L737-L737) | `_dispatch_trigger_callbacks()`:仅处理 json_trigger_value |
| 值变化检测 | [session_state.py:857](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py#L857-L857) | `_widget_changed()`:比较 _new_widget_state vs _old_state |
| Widget 注册+返回值 | [session_state.py:998](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py#L998-L998) | `register_widget()`:第一次则存默认值,返回 deepcopy 的当前值 |
| Trigger 重置 | [session_state.py:880](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py#L880-L880) | `_reset_triggers()`:trigger_value→False,其他 trigger→None |
| 脚本结束清理 | [session_state.py:866](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py#L866-L866) | `on_script_finished()`:reset triggers + 删除 stale widgets |
| **Fragment** | | |
| Fragment 装饰器+存储 | [fragment.py:332](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/fragment.py#L332-L332) | `_fragment()` 定义+`MemoryFragmentStorage` 内存存储 |
| **前端组件与 Hook** | | |
| Form 容器组件 | [Form.tsx:57](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/widgets/Form/Form.tsx#L57-L57) | 缺失 submit button 警告、Enter 禁用提示 |
| 提交按钮组件 | [FormSubmitButton.tsx:41](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/widgets/Form/FormSubmitButton.tsx#L41-L41) | handleSubmit、上传中自动禁用 |
| 表单清除助手 Hook | [FormClearHelper.ts:92](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/widgets/Form/FormClearHelper.ts#L92-L92) | `useFormClearHelper()` 订阅 formCleared 信号 |
| Widget 状态 Hook | [useBasicWidgetState.ts:255](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/hooks/useBasicWidgetState.ts#L255-L255) | `useBasicWidgetState()`:默认值获取、setValue 响应、form clear |
| Widget 客户端状态 Hook | [useBasicWidgetState.ts:85](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/hooks/useBasicWidgetState.ts#L85-L85) | `useBasicWidgetClientState()`:初始化、form clear handler |
| Enter 键提交 Hook | [useSubmitFormViaEnterKey.ts:39](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/hooks/useSubmitFormViaEnterKey.ts#L39-L39) | 键位检查→commit→allowFormEnterToSubmit→submitForm |
| Forms 上下文 | [FormsContext.tsx:41](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/core/FormsContext.tsx#L41-L41) | 不可变 FormsData 的 React Context 订阅 |
