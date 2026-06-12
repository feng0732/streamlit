# Streamlit 自定义组件 iframe 桥接机制深度解析

## 概述

Streamlit 提供了两代自定义组件系统，它们采用了完全不同的桥接架构：

| 版本 | 名称 | 桥接方式 | 隔离机制 | 典型应用 |
|------|------|----------|----------|----------|
| v1 | CustomComponent | iframe + postMessage | iframe 沙箱 | 第三方复杂组件 |
| v2 | BidiComponent | 直接 DOM 渲染 | Shadow DOM (可选) | 内置/轻量组件 |

---

## 一、v1 自定义组件 (iframe 桥接)

### 1.1 整体架构

v1 自定义组件基于 **iframe + postMessage** 实现双向通信。整体架构分为三层：

```
┌──────────────────────────────────────────────────────┐
│  Python 后端                                          │
│  - CustomComponent 类                                  │
│  - component_arrow (数据序列化)                        │
│  - 组件注册表 (LocalComponentRegistry)                │
└──────────────────────┬───────────────────────────────┘
                       │ Protobuf (Delta/ForwardMsg)
                       ▼
┌──────────────────────────────────────────────────────┐
│  前端宿主 (Streamlit App)                            │
│  - ComponentRegistry (消息分发中心)                   │
│  - ComponentInstance (每个组件实例一个)                │
│  - WidgetStateManager (状态管理)                      │
└──────────────────────┬───────────────────────────────┘
                       │ postMessage (iframe 桥接)
                       ▼
┌──────────────────────────────────────────────────────┐
│  iframe 内的组件前端                                   │
│  - streamlit.js (组件 SDK)                            │
│  - Streamlit 类 (setComponentValue, setFrameHeight)  │
│  - EventTarget (接收 render 事件)                     │
└──────────────────────────────────────────────────────┘
```

### 1.2 iframe 通信机制

#### 消息通道建立

1. **宿主 → iframe**: 通过 `iframe.contentWindow.postMessage()` 发送消息
2. **iframe → 宿主**: 通过 `window.parent.postMessage()` 发送消息
3. **消息识别**: 所有从 iframe 发出的消息都带有 `isStreamlitMessage: true` 标志

关键代码参考:
- 宿主发送消息: [componentUtils.tsx#L242-L279](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/componentUtils.tsx#L242-L279)
- iframe 发送消息: [streamlit.ts#L232-L245](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/component-lib/src/streamlit.ts#L232-L245)

#### 消息协议

**从宿主 (Streamlit) 到组件 (iframe)**

| 消息类型 | 枚举值 | 数据结构 | 触发时机 |
|----------|--------|----------|----------|
| RENDER | `streamlit:render` | `{ args, dfs, disabled, theme }` | 组件就绪后 & 属性变化时 |

**从组件 (iframe) 到宿主 (Streamlit)**

| 消息类型 | 枚举值 | 数据结构 | 触发时机 |
|----------|--------|----------|----------|
| COMPONENT_READY | `streamlit:componentReady` | `{ apiVersion: number }` | 组件加载完成 |
| SET_COMPONENT_VALUE | `streamlit:setComponentValue` | `{ value, dataType }` | 组件值变化 |
| SET_FRAME_HEIGHT | `streamlit:setFrameHeight` | `{ height: number }` | 组件高度变化 |

消息枚举定义: [enums.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/enums.ts)

### 1.3 事件转发流程

#### ComponentRegistry - 消息分发中心

[ComponentRegistry.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentRegistry.ts) 是整个 iframe 通信的核心调度器：

```
window.message 事件
       │
       ▼
onMessageEvent()  ←── 全局唯一监听器
       │
       ├─ 检查 isStreamlitMessage 标志
       │
       ├─ 检查 event.source (MessageEventSource)
       │
       ▼
msgListeners Map (source → listener)
       │
       ▼
对应 ComponentInstance 的消息处理器
```

**关键设计**:
- 全局只注册一个 `window.message` 监听器（在 ComponentRegistry 构造函数中）
- 使用 `Map<MessageEventSource, ComponentMessageListener>` 按来源分发
- 每个 ComponentInstance 注册/注销自己的监听器（基于 iframe 的 contentWindow）

注册与注销代码:
- 注册: [ComponentRegistry.ts#L50-L59](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentRegistry.ts#L50-L59)
- 注销: [ComponentRegistry.ts#L61-L66](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentRegistry.ts#L61-L66)

#### 消息处理流程

`createIframeMessageHandler()` 函数创建每个组件实例的消息处理器：

[componentUtils.tsx#L94-L173](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/componentUtils.tsx#L94-L173)

```
收到消息 (type, data)
       │
       ├─ COMPONENT_READY
       │    ├─ 校验 apiVersion
       │    └─ 调用 componentReadyCallback()
       │         └─ 发送首次 RENDER 消息
       │
       ├─ SET_COMPONENT_VALUE
       │    ├─ 检查 isReady 状态
       │    └─ handleSetComponentValue()
       │         ├─ json → widgetMgr.setJsonValue
       │         ├─ dataframe → widgetMgr.setArrowValue
       │         └─ bytes → widgetMgr.setBytesValue
       │
       └─ SET_FRAME_HEIGHT
            ├─ 检查 isReady 状态
            └─ frameHeightCallback()
                 └─ 直接设置 iframe.height (避免重渲染)
```

**重要设计细节**:
- 使用 `ref` 模式传递回调，避免频繁注册/注销监听器
- `isReady` 状态守卫：组件未就绪前忽略值和高度消息
- 高度更新直接操作 DOM，不走 React 重渲染路径，避免性能问题

### 1.4 组件挂载流程

#### Python 端挂载

[custom_component.py](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/components/v1/custom_component.py)

1. **组件声明**: `declare_component(name, path, url)` 创建 `CustomComponent` 对象
2. **组件注册**: 运行时注册到 `Runtime.component_registry`
3. **实例创建**: 调用组件函数时执行 `create_instance()`
   - 参数分类: JSON 可序列化的进 `json_args`，bytes/dataframe 进 `special_args`
   - 生成组件 ID: `compute_and_register_element_id()`
   - 注册 widget: `register_widget()`
   - 入队 delta: `dg._enqueue("component_instance", ...)`

#### 前端挂载

[ComponentInstance.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentInstance.tsx)

```
ElementNodeRenderer 渲染
       │
       ▼
ComponentInstance 组件挂载
       │
       ├─ 1. 计算 iframe src (getSrc)
       │    ├─ 从 ComponentRegistry 获取 URL
       │    └─ 添加 streamlitUrl 查询参数
       │
       ├─ 2. 渲染 iframe (StyledComponentIframe)
       │    ├─ sandbox 策略
       │    ├─ allow (feature policy)
       │    └─ 初始高度
       │
       ├─ 3. useEffect: 注册消息监听器
       │    └─ registry.registerListener(contentWindow, handler)
       │
       ├─ 4. 等待 COMPONENT_READY 消息
       │    ├─ 显示 Skeleton 加载态
       │    └─ 60秒超时警告
       │
       └─ 5. 就绪后发送 RENDER 消息
            └─ args, dfs, disabled, theme
```

#### iframe src 构造

[ComponentInstance.tsx#L88-L113](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentInstance.tsx#L88-L113)

每个 iframe 的 src 会附加查询参数：
- `streamlitUrl`: 父页面的 origin + pathname，用于组件识别宿主
- `__streamlit_parent_client_id`: 父客户端 ID（用于嵌套组件场景）

### 1.5 数据传输

#### 参数序列化 (Python → Frontend)

1. **JSON 参数**: 直接序列化为 JSON 字符串 (`json_args`)
2. **特殊参数** (`special_args`):
   - `bytes`: 二进制数据
   - `arrowdataframe`: DataFrame 数据 (Arrow 格式)

#### 前端参数解析

[componentUtils.tsx#L190-L236](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/componentUtils.tsx#L190-L236)

- `json_args` → 解析为 JS 对象
- `bytes` → 合并到 args 对象
- `arrowdataframe` → 单独放到 `dfs` 数组中（因为无法通过 postMessage 直接传递 ArrowTable 实例）

#### 组件侧数据重建

[streamlit.ts#L176-L229](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/component-lib/src/streamlit.ts#L176-L229)

在 iframe 内，组件 SDK 负责：
1. 接收 `args` 和 `dfs`
2. 将 `dfs` 中的 Arrow 数据重建为 `ArrowTable` 实例
3. 合并到 `args` 对象中
4. 通过 `CustomEvent` 派发 `streamlit:render` 事件

---

## 二、v2 双向组件 (BidiComponent)

### 2.1 架构差异

v2 组件**不使用 iframe**，而是直接渲染到宿主 DOM 中，通过 Shadow DOM 提供可选的样式隔离。

| 特性 | v1 (iframe) | v2 (BidiComponent) |
|------|-------------|---------------------|
| 渲染容器 | iframe | 普通 div / ShadowRoot |
| 通信方式 | postMessage | 直接函数调用 |
| 样式隔离 | 天然隔离 | Shadow DOM (可选) |
| JS 隔离 |  iframe 沙箱 | 同上下文执行 |
| 数据传输 | postMessage 序列化 | 直接内存访问 |
| 加载方式 | 加载独立 HTML | 注入 JS/CSS/HTML |

### 2.2 两种渲染模式

[BidiComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/BidiComponent/BidiComponent.tsx)

根据 `isolateStyles` 属性选择模式：

- **隔离模式** (`IsolatedComponent`): 使用 Shadow DOM，样式完全隔离
- **非隔离模式** (`NonIsolatedComponent`): 直接渲染到普通 DOM

#### 隔离模式 (Shadow DOM)

[IsolatedComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/BidiComponent/IsolatedComponent.tsx)

```
挂载阶段
    │
    ├─ 创建容器 div
    ├─ attachShadow({ mode: "open" })
    ├─ 标记 shadowRoot 就绪
    │
    └─ 内容注入
         ├─ useHandleHtmlAndCssContent
         └─ useHandleJsContent
```

### 2.3 JS 执行机制

[useHandleJsContent.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/BidiComponent/hooks/useHandleJsContent.ts)

#### 加载方式

1. **内联 JS**: 通过 `Blob URL` 动态创建模块脚本
2. **外部 JS**: 通过 `<script type="module">` 加载

#### 执行模型

组件 JS 模块的默认导出函数会被调用，传入 `FrontendRendererArgs`：

```typescript
{
  name: string,           // 组件名称
  data: unknown,          // 数据
  key: string,            // 组件实例 ID
  parentElement: HTMLElement | ShadowRoot,  // 父容器
  setStateValue: (name, value) => void,     // 设置状态
  setTriggerValue: (name, value) => void,   // 触发事件
}
```

#### 状态更新机制

- `setStateValue`: 更新组件状态值，写入 WidgetStateManager，类型为 `json_value`
- `setTriggerValue`: 触发事件，类型为 `json_trigger_value`，会立即触发重运行
  - 在表单内时，setTriggerValue 会被忽略（触发器不允许在表单中）

---

## 三、会话隔离机制

### 3.1 组件注册与会话

#### Python 端

[component_registry.py](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/components/v1/component_registry.py)

- 组件注册在 `Runtime` 级别（全局单例）
- `LocalComponentRegistry` 维护所有已注册组件
- 每个会话通过 `ScriptRunContext` 访问同一注册表

关键代码: `Runtime.instance().component_registry`

参考: [runtime.py](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/runtime/runtime.py)

### 3.2 Widget 状态隔离

#### 状态存储

[WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/WidgetStateManager.ts)

- 每个前端实例（浏览器标签页）有一个 `WidgetStateManager`
- 状态以 `widgetId → WidgetState` 的 Map 形式存储
- 每个组件实例有唯一的 widget ID

#### 组件 ID 生成

Python 端通过 `compute_and_register_element_id()` 生成唯一 ID：

参考: [custom_component.py#L174-L187](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/components/v1/custom_component.py#L174-L187)

ID 计算考虑因素:
- 组件类型 (`component_instance`)
- 用户提供的 `key`（如果有）
- 组件名称 `name`
- 组件 URL `url`
- 参数内容（用于无 key 组件的身份识别）

### 3.3 多会话场景

Streamlit 的会话隔离主要体现在：

1. **后端会话**: 每个用户连接对应一个 `ScriptRunner` 和 `SessionState`
2. **前端会话**: 每个标签页对应一个 WebSocket 连接和 WidgetStateManager
3. **组件实例**: 每个组件调用生成独立的 ElementNode 和 widget ID

**注意**: v1 组件的 iframe 本身不感知会话，它只与父窗口通信。会话隔离由父窗口的 WidgetStateManager 和 component ID 保证。

### 3.4 v2 组件的状态隔离

v2 组件通过以下机制保证隔离：

1. **Context 隔离**: 每个 BidiComponent 实例有自己的 `BidiComponentContextProvider`
2. **Widget ID**: 每个实例有唯一的 widget ID
3. **Shadow DOM**: 隔离模式下样式和 DOM 完全隔离
4. **Cleanup 函数**: 组件卸载时执行清理，防止内存泄漏

---

## 四、安全策略

### 4.1 iframe Sandbox 策略

[IFrameUtil.ts#L21-L77](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/util/IFrameUtil.ts#L21-L77)

默认启用的 sandbox 权限：

| 权限 | 说明 |
|------|------|
| `allow-forms` | 允许提交表单 |
| `allow-modals` | 允许打开模态窗口 |
| `allow-popups` | 允许弹出窗口 |
| `allow-popups-to-escape-sandbox` | 弹出窗口不继承沙箱 |
| `allow-same-origin` | 允许同源访问（重要：与 allow-scripts 组合会降低安全性） |
| `allow-scripts` | 允许运行脚本 |
| `allow-downloads` | 允许触发下载 |

**重要说明**: 由于同时启用了 `allow-same-origin` 和 `allow-scripts`，iframe 实际上可以移除自身的 sandbox 属性。这是产品决策，为了解锁更多用例（组件的 Python 代码本来就不受沙箱限制）。

### 4.2 iframe Feature Policy

[IFrameUtil.ts#L83-L170](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/util/IFrameUtil.ts#L83-L170)

默认允许的浏览器特性包括：
- `accelerometer`, `ambient-light-sensor`, `gyroscope`, `magnetometer` (传感器)
- `autoplay`, `picture-in-picture` (媒体)
- `camera`, `microphone` (音视频输入)
- `clipboard-write` (剪贴板写入)
- `fullscreen` (全屏)
- `geolocation` (地理位置)
- `payment` (支付)
- `usb`, `xr-spatial-tracking` (高级特性)
- 等等...

### 4.3 v2 组件的安全考量

v2 组件直接在宿主上下文中执行 JS，**没有 iframe 沙箱保护**：

- **内联 JS**: 通过 Blob URL 加载，与主文档同上下文
- **外部 JS**: 通过 `<script type="module">` 加载
- **安全假设**: 组件代码是受信任的（开发者自己或已知第三方）

---

## 五、完整消息时序图

### v1 组件初始化时序

```
Python 后端           前端宿主             iframe 组件
    │                    │                     │
    ├─ 发送 Delta        │                     │
    │   (component_instance)                   │
    │                    │                     │
    │                    ├─ 渲染 ComponentInstance
    │                    │    (创建 iframe)    │
    │                    │                     │
    │                    ├─────────────────────► 加载 HTML
    │                    │                     │
    │                    │                     ├─ 调用 Streamlit.setComponentReady()
    │                    │                     │
    │                    │  ◄──────────────────┤ COMPONENT_READY
    │                    │                     │
    │                    ├─ componentReadyCallback()
    │                    │                     │
    │                    ├─────────────────────► RENDER (args, dfs, theme)
    │                    │                     │
    │                    │                     ├─ 组件渲染
    │                    │                     │
    │                    │  ◄──────────────────┤ SET_FRAME_HEIGHT
    │                    │                     │
    │                    ├─ 更新 iframe 高度   │
    │                    │                     │
```

### v1 组件值更新时序

```
用户交互
  │
  ▼
iframe 组件           前端宿主             Python 后端
    │                    │                     │
    ├─ Streamlit.setComponentValue(value)     │
    │                    │                     │
    ├────────────────────►                     │
    │  SET_COMPONENT_VALUE                    │
    │                    │                     │
    │                    ├─ widgetMgr.setJsonValue()
    │                    │   (或 Arrow/Bytes)  │
    │                    │                     │
    │                    ├─ 触发 RERUN         │
    │                    │   (通过 WebSocket)  │
    │                    ├─────────────────────►
    │                    │                     │
    │                    │                     ├─ 重新执行脚本
    │                    │                     │
    │                    │  ◄──────────────────┤ 新的 Delta
    │                    │                     │
    │                    ├─────────────────────► RENDER (新 args)
    │                    │                     │
```

---

## 六、关键文件索引

| 文件路径 | 说明 |
|----------|------|
| [ComponentInstance.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentInstance.tsx) | v1 组件实例 (iframe 宿主端) |
| [ComponentRegistry.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentRegistry.ts) | v1 消息分发中心 |
| [componentUtils.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/componentUtils.tsx) | v1 消息处理与数据序列化 |
| [enums.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/enums.ts) | 消息类型枚举 |
| [streamlit.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/component-lib/src/streamlit.ts) | 组件 SDK (iframe 端) |
| [IFrameUtil.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/util/IFrameUtil.ts) | iframe 安全策略 |
| [BidiComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/BidiComponent/BidiComponent.tsx) | v2 组件入口 |
| [IsolatedComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/BidiComponent/IsolatedComponent.tsx) | v2 Shadow DOM 隔离模式 |
| [useHandleJsContent.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/BidiComponent/hooks/useHandleJsContent.ts) | v2 JS 执行与状态管理 |
| [custom_component.py](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/components/v1/custom_component.py) | v1 Python 端组件实现 |
| [main.py](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/components/v2/bidi_component/main.py) | v2 Python 端组件实现 |
| [base_component_registry.py](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/components/types/base_component_registry.py) | 组件注册表接口 |
| [WidgetStateManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/WidgetStateManager.ts) | Widget 状态管理器 |
| [AppRoot.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/render-tree/AppRoot.ts) | 渲染树根节点 |
| [ElementNodeRenderer.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/core/Block/ElementNodeRenderer.tsx) | 元素节点渲染器 |
