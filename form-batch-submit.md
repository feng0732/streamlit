# st.form 批量提交模型代码协作分析

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

**核心文件:**
- [form.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/form.py)
- [form_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/lib/form_utils.py)
- [button.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/widgets/button.py)

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
# form_utils.py - current_form_id()
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
# button.py - _button() 第1650行
form_id = current_form_id(self.dg) if is_form_submitter else ""
button_proto.form_id = form_id
```

非 submitter 的 widget 也会在 proto 中携带 form_id,用于前端识别归属。

### 2.2 前端:WidgetStateManager 的双状态存储

**核心文件:**
- [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts)

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

**核心文件:**
- [FormSubmitButton.tsx](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/widgets/Form/FormSubmitButton.tsx)
- [useSubmitFormViaEnterKey.ts](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/hooks/useSubmitFormViaEnterKey.ts)

#### 方式1:点击提交按钮

```typescript
// FormSubmitButton.tsx - handleSubmit() 第65行
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

`submitForm` 是批量提交的核心方法,执行以下关键步骤:

```typescript
// WidgetStateManager.ts - submitForm() 第346行
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

**核心文件:**
- [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/app_session.py)
- [script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py)
- [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py)

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
> 在 `_dispatch_trigger_callbacks` 中,`widget_proto_state.trigger_value == true` 会匹配并执行 `metadata.callbacks["click"]`(即用户的 `on_click`)。

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
  widgetProps: { disabled?: boolean },
  commit: () => void,
  fragmentId?: string
): void {
  const widgetMgr = useContext(WidgetManagerContext)
  
  const onEnterPressed = useCallback((): void => {
    if (widgetProps.disabled) return       // 1. widget 自身禁用
    if (!widgetMgr.allowFormEnterToSubmit(formId)) return  // 2. 表单级校验
    commit()                               // 3. 先提交当前 widget 的值
    widgetMgr.submitForm(formId, fragmentId)  // 4. 再触发表单提交
  }, [widgetProps.disabled, widgetMgr, formId, commit, fragmentId])
  
  // 绑定 keydown 事件...
}
```

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

> **判断优先级(从高到低):**
> 1. 当前输入 widget 自身 `disabled=true` → 拦截
> 2. 表单显式 `enter_to_submit=False` → 拦截  
> 3. 表单无任何 submit button → 拦截
> 4. 第一个 submit button `disabled=true` → 拦截
> 5. 全部通过 → 允许回车提交

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

**核心文件:**
- [Form.tsx](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/widgets/Form/Form.tsx)

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

**核心文件:**
- [FormsContext.tsx](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/core/FormsContext.tsx)

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

`allowFormEnterToSubmit` 检查是否允许回车提交:

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

## 八、关键代码索引

| 功能模块 | 文件 | 关键位置 |
|---------|------|---------|
| Form 容器创建 | [form.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/form.py) | `FormMixin.form()` 第76行 |
| Form 归属判断 | [form_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/lib/form_utils.py) | `current_form_id()` 第62行 |
| Submit Button 创建 | [button.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/elements/widgets/button.py) | `_button()` 第1616行 |
| Widget 状态管理器 | [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts) | `submitForm()` 第346行 |
| 表单值写入分流 | [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts) | `createWidgetState()` 第873行 |
| 批处理调度 | [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts) | `scheduleFlush()` 第1071行 |
| Enter 键提交校验 | [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/WidgetStateManager.ts) | `allowFormEnterToSubmit()` 第932行 |
| 后端 rerun 分发 | [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/app_session.py) | `request_rerun()` 第420行 |
| 后端脚本主循环 | [script_runner.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/scriptrunner/script_runner.py) | `_run_script()` 第528行 |
| 状态同步+回调触发 | [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py) | `on_script_will_rerun()` 第641行 |
| 回调调度(两条路径) | [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py) | `_call_callbacks()` 第653行 |
| Widget 注册读取值 | [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py) | `register_widget()` 第998行 |
| Trigger 重置 | [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py) | `_reset_triggers()` 第880行 |
| 脚本结束清理 | [session_state.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/state/session_state.py) | `on_script_finished()` 第866行 |
| Fragment 定义/存储 | [fragment.py](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/lib/streamlit/runtime/fragment.py) | `_fragment()` 第332行、`MemoryFragmentStorage` 第180行 |
| Form 组件 | [Form.tsx](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/widgets/Form/Form.tsx) | 主组件第57行 |
| 提交按钮组件 | [FormSubmitButton.tsx](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/widgets/Form/FormSubmitButton.tsx) | 主组件第41行 |
| 表单清除助手 | [FormClearHelper.ts](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/widgets/Form/FormClearHelper.ts) | `useFormClearHelper()` 第92行 |
| Widget 基础 Hook | [useBasicWidgetState.ts](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/hooks/useBasicWidgetState.ts) | `useBasicWidgetState()` 第255行 |
| Enter 键提交 Hook | [useSubmitFormViaEnterKey.ts](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/hooks/useSubmitFormViaEnterKey.ts) | 默认导出第39行 |
| Forms 上下文 | [FormsContext.tsx](file:///d:/fz/0601/solo-dogfeeding/code/219-streamlit/frontend/lib/src/components/core/FormsContext.tsx) | 第41行 |
