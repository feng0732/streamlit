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

## 六、消息信任边界

### 6.1 问题本质

postMessage 是浏览器原生的跨文档通信 API，任何窗口都可以向另一个窗口发送消息，因此宿主页面接收到的 `message` 事件**天然不可信**。核心问题：

1. 任何嵌入在页面中的第三方脚本都可能调用 `window.postMessage` 伪造消息
2. 同源策略下，同源的其他窗口/iframe 也可以发送消息
3. 消息内容（包括 `type`、`isStreamlitMessage` 等字段）可以被任意构造

### 6.2 Streamlit 的三层信任校验

[ComponentRegistry.ts#L109-L142](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentRegistry.ts#L109-L142) 中的 `onMessageEvent` 实现了三层递进校验：

```
收到 window.message 事件
       │
       │  ┌─────────────────────────────────────────────┐
       │  │ 第一层: 协议标志过滤                          │
       │  │ 检查 event.data.isStreamlitMessage === true  │
       │  │ 过滤掉绝大多数无关消息                        │
       │  └─────────────────────────────────────────────┘
       ▼
  通过第一层
       │
       │  ┌─────────────────────────────────────────────┐
       │  │ 第二层: 消息来源精确匹配                      │
       │  │ 用 event.source (MessageEventSource)         │
       │  │ 查找 msgListeners Map                        │
       │  │ 只有已注册的 iframe contentWindow 才能匹配    │
       │  └─────────────────────────────────────────────┘
       ▼
  通过第二层
       │
       │  ┌─────────────────────────────────────────────┐
       │  │ 第三层: 消息类型校验                          │
       │  │ 检查 event.data.type 是否为已知类型           │
       │  │ 未知类型被忽略并记录警告                      │
       │  └─────────────────────────────────────────────┘
       ▼
  消息被转发到对应 ComponentInstance
```

#### 第一层：`isStreamlitMessage` 标志

```typescript
if (
  isNullOrUndefined(event.data) ||
  !Object.hasOwn(event.data, "isStreamlitMessage")
) {
  return  // 过滤掉所有不带标志的消息
}
```

[ComponentRegistry.ts#L110-L116](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentRegistry.ts#L110-L116)

这是一个**弱校验**：任何知道 Streamlit 消息格式的代码都可以设置 `isStreamlitMessage: true`。它的作用是**过滤噪声**——页面上可能有大量其他 postMessage 通信（如 iframe resizer、分析脚本等），这层过滤能快速排除无关消息。

#### 第二层：`event.source` 精确匹配 —— 核心安全机制

```typescript
const listener = this.msgListeners.get(event.source)
if (isNullOrUndefined(listener)) {
  LOG.warn("Received component message for unregistered ComponentInstance!")
  return
}
```

[ComponentRegistry.ts#L125-L132](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentRegistry.ts#L125-L132)

这是**最关键的信任边界**。`event.source` 是浏览器在 `MessageEvent` 上自动设置的 `MessageEventSource` 引用，**不可被伪造**。即使恶意代码构造了一个包含 `isStreamlitMessage: true` 的消息并发送，浏览器的 `event.source` 仍然会指向发送方的真实 window 对象。

`msgListeners` Map 的 key 是 `MessageEventSource`（即 iframe 的 `contentWindow` 引用），注册时机在 [ComponentInstance.tsx#L382-L385](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentInstance.tsx#L382-L385)：

```typescript
registry.registerListener(
  contentWindow,      // <-- 这就是 iframe.contentWindow
  createIframeMessageHandler(onBackMsgRef)
)
```

**只有 ComponentInstance 自己创建的 iframe 发出的消息才能匹配到对应的 listener**。这意味着：

- 外部第三方脚本发送的伪造消息 → `event.source` 指向发送方 window → 在 Map 中找不到 → 被丢弃
- 其他 iframe 实例发送的消息 → `event.source` 指向另一个 iframe → 在 Map 中找不到对应的 listener → 被丢弃
- 同一个 iframe 的消息 → `event.source` 精确匹配 → 被路由到正确的 ComponentInstance

#### 第三层：消息类型校验

```typescript
const { type } = event.data
if (isNullOrUndefined(type)) {
  LOG.warn("Received Streamlit message with no type!")
  return
}
```

[ComponentRegistry.ts#L134-L138](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentRegistry.ts#L134-L138)

在 `createIframeMessageHandler` 中还有进一步的类型校验，未知消息类型的 `switch` 分支会落入 `default` 并记录警告：

[componentUtils.tsx#L169-L171](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/componentUtils.tsx#L169-L171)

### 6.3 注册/注销的生命周期管理

消息信任边界的另一面是**注册的生命周期管理**，确保不出现悬挂的监听器：

```
ComponentInstance 挂载
       │
       ├─ useEffect 注册监听器
       │    registry.registerListener(contentWindow, handler)
       │
       │  ... 组件存活期间 ...
       │
       ├─ useEffect cleanup 注销监听器
       │    registry.deregisterListener(contentWindow)
       │
       └─ 组件卸载后，该 iframe 发出的消息不再被处理
```

[ComponentInstance.tsx#L373-L395](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/ComponentInstance.tsx#L373-L395)

当 iframe 被销毁（例如组件从页面上移除），其 `contentWindow` 引用变为无效，Map 中对应的条目通过 cleanup 函数被显式移除。即使 cleanup 未执行（极端情况），浏览器也会在 iframe 销毁后使 `event.source` 不再匹配已注册的 source。

### 6.4 postMessage 的 targetOrigin 问题

**宿主 → iframe 方向**：`sendRenderMessage` 使用 `iframe.contentWindow.postMessage(data, "*")`

[componentUtils.tsx#L264-L278](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/CustomComponent/componentUtils.tsx#L264-L278)

`"*"` 作为 targetOrigin 意味着不限制接收方的 origin。这在安全上不是最佳实践，但由于组件可能从任意源加载（包括开发服务器），严格限定 origin 会破坏组件的灵活性。

**iframe → 宿主方向**：`window.parent.postMessage(data, "*")`

[streamlit.ts#L237-L244](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/component-lib/src/streamlit.ts#L237-L244)

同样使用 `"*"`。由于 iframe 内的组件 SDK 无法预知宿主页面的 origin，所以无法指定更严格的 targetOrigin。

**风险评估**：虽然 `targetOrigin: "*"` 允许消息被任意源接收，但由于：
1. 宿主端通过 `event.source` 精确匹配来校验来源
2. 回传的值通过 widget ID 关联到特定组件实例
3. 后端有 session_id 校验

因此即使消息在传输层被截获，攻击者也无法将值注入到错误的组件或会话中。

### 6.5 v1 vs v2 的信任边界对比

| 维度 | v1 (iframe) | v2 (BidiComponent) |
|------|-------------|---------------------|
| 消息通道 | postMessage (跨文档) | 直接函数调用 (同文档) |
| 来源校验 | `event.source` 精确匹配 | 无需校验（闭包绑定） |
| 伪造难度 | 高（需控制已注册的 iframe） | 不适用（无外部消息通道） |
| 潜在攻击面 | 恶意 iframe 注入 | 恶意脚本注入（需同源） |

v2 组件不使用 postMessage，状态更新通过闭包中捕获的 `widgetMgr.setJsonValue()` 直接调用。这意味着**不存在跨文档消息伪造的攻击面**，但也意味着 v2 组件的 JS 代码运行在与宿主相同的 JavaScript 上下文中，拥有完全的 DOM 访问权限。

---

## 七、消息回传与会话关联

### 7.1 问题本质

当用户在 iframe 内的组件中交互（如点击按钮），组件通过 `Streamlit.setComponentValue(value)` 将新值回传给宿主。关键问题是：

1. **组件实例隔离**：同一页面上可能有多个同类组件实例，如何确保值更新到正确的实例？
2. **会话隔离**：多个用户同时访问同一应用，如何确保值更新到正确的用户会话？

### 7.2 前端侧的组件实例关联

#### 关联链路

```
iframe 发出 SET_COMPONENT_VALUE
       │
       ▼
ComponentRegistry.onMessageEvent()
       │
       ├─ event.source → 找到 listener
       │  (精确匹配到唯一的 ComponentInstance)
       ▼
createIframeMessageHandler() 中的回调
       │
       ├─ 从 callbacks.current 解构出 element (ComponentInstanceProto)
       │    element.id = "组件实例的唯一 widget ID"
       │
       ▼
handleSetComponentValue(value, dataType, source, element, widgetMgr, fragmentId)
       │
       ├─ widgetMgr.setJsonValue(element, value, source, fragmentId)
       │    element 满足 WidgetInfo 接口: { id: string, formId?: string }
       │    ↓
       │    createWidgetState(widget, source)
       │      → widgetStateDict.createState(widget.id)
       │      → WidgetState.id = element.id (唯一标识)
       │
       ▼
onWidgetValueChanged(formId, source, fragmentId)
       │
       ▼
scheduleFlush() → sendUpdateWidgetsMessage()
       │
       ▼
WidgetStateManager 发送包含所有 WidgetState 的 rerunScript BackMsg
```

#### 关键数据流：element.id 如何成为 widget 的唯一标识

1. **Python 端生成 ID**：
   [custom_component.py#L174-L187](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/components/v1/custom_component.py#L174-L187)

   ```python
   computed_id = compute_and_register_element_id(
       "component_instance",
       user_key=key,
       key_as_main_identity={"name", "url"},
       dg=dg,
       name=self.name,
       url=self.url,
       json_args=serialized_json_args,
       special_args=special_args,
   )
   element.component_instance.id = computed_id
   ```

   这个 ID 由组件类型、key、名称、URL 和参数内容联合计算，**在同一脚本运行的相同位置产生相同 ID**，在不同位置或不同参数下产生不同 ID。

2. **前端携带 ID**：`ComponentInstanceProto` 通过 protobuf 传输到前端，`element.id` 被保持不变。

3. **回传时使用 ID**：`handleSetComponentValue` 直接将整个 `element` 对象（满足 `WidgetInfo` 接口）传给 `widgetMgr.setJsonValue(element, value, ...)`，其中 `element.id` 就是 widget 的唯一标识。

#### 同页面多实例不干扰的原因

假设同一页面有两个相同类型的组件实例 A 和 B：

```
组件 A 的 iframe (contentWindow_A)
  │
  ├─ 注册: msgListeners.set(contentWindow_A, handler_A)
  │         handler_A 闭包中捕获 element_A (id = "组件A的widgetId")
  │
组件 B 的 iframe (contentWindow_B)
  │
  ├─ 注册: msgListeners.set(contentWindow_B, handler_B)
  │         handler_B 闭包中捕获 element_B (id = "组件B的widgetId")
```

当组件 A 发送 `SET_COMPONENT_VALUE`：
1. `event.source` = `contentWindow_A` → 查 Map → 匹配到 `handler_A`
2. `handler_A` 中 `element` = `element_A` → `widgetMgr.setJsonValue(element_A, value, ...)`
3. WidgetStateManager 使用 `element_A.id` 创建/更新对应的 WidgetState

**组件 B 的 WidgetState 完全不受影响**，因为：
- 消息来源 (`event.source`) 不同，不会路由到 `handler_B`
- 即使路由错误，`element_A.id ≠ element_B.id`，WidgetState 也是按 ID 隔离的

### 7.3 后端侧的会话关联

#### 完整的数据流：从前端值更新到后端脚本重运行

```
前端 WidgetStateManager
       │
       ├─ sendUpdateWidgetsMessage()
       │    → 创建 WidgetStates protobuf (包含所有活跃 widget 的值)
       │    → 每个 WidgetState.id = 组件实例的唯一 ID
       │
       ▼
App.sendRerunBackMsg()
       │  [App.tsx#L1917-L2007]
       │
       ├─ 创建 BackMsg { rerunScript: { widgetStates, ... } }
       │
       ▼
ConnectionManager.sendMessage()
       │
       ▼
WebsocketConnection.sendMessage()
       │  [WebsocketConnection.tsx#L665-L679]
       │
       ├─ 编码为 Protobuf → 通过 WebSocket 发送
       │
       ▼
服务端 Starlette WebSocket Handler
       │  [starlette_websocket.py#L510]
       │
       ├─ runtime.handle_backmsg(session_id, back_msg)
       │    session_id 来自 WebSocket 连接建立时分配的 ID
       │
       ▼
Runtime.handle_backmsg()
       │  [runtime.py#L509-L533]
       │
       ├─ 根据 session_id 查找对应的 AppSession
       │    session_info = self._session_mgr.get_active_session_info(session_id)
       │
       ├─ 如果 session 不存在或已断开 → 丢弃消息
       │
       ▼
AppSession.handle_backmsg()
       │  [app_session.py#L342-L346]
       │
       ├─ 解析 rerunScript → 提取 widgetStates
       │
       ▼
AppSession.request_rerun(client_state)
       │  [app_session.py#L406-L464]
       │
       ├─ 构建 RerunData:
       │    widget_states = client_state.widget_states
       │    (包含所有 widget ID → 值的映射)
       │
       ▼
ScriptRunner 使用 widget_states 执行脚本
       │
       ├─ register_widget() 根据组件 ID 查找已注册的 widget
       │    → 反序列化用户提交的值
       │    → 返回当前值 (新值或默认值)
       │
       └─ 组件函数返回对应的 widget_value
```

#### 多用户会话不干扰的原因

1. **WebSocket 连接隔离**：每个浏览器标签页与服务器建立一个独立的 WebSocket 连接。

   [WebsocketConnection.tsx#L517-L544](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/connection/src/WebsocketConnection.tsx#L517-L544)

   ```typescript
   const sessionTokens = await this.getSessionTokens()
   this.websocket = new WebSocket(uri, ["streamlit", ...sessionTokens])
   ```

   `sessionTokens` 中包含可选的 `lastSessionId`，用于重连时恢复到之前的会话。

2. **服务端 session_id 路由**：WebSocket 连接建立时，服务端为每个连接分配唯一的 `session_id`。

   [starlette_websocket.py#L449-L510](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py#L449-L510)

   ```python
   session_id = runtime.connect_session(...)
   # ... 后续所有来自该 WebSocket 的 BackMsg 都携带此 session_id
   runtime.handle_backmsg(session_id, back_msg)
   ```

3. **AppSession 隔离**：每个 `session_id` 对应一个独立的 `AppSession` 实例，拥有独立的 `SessionState`、`ScriptRunner` 和 widget 状态。

   [runtime.py#L526-L533](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/runtime/runtime.py#L526-L533)

   ```python
   session_info = self._session_mgr.get_active_session_info(session_id)
   if session_info is None:
       _LOGGER.debug("Discarding BackMsg for disconnected session")
       return
   session_info.session.handle_backmsg(msg)
   ```

4. **后端操作请求的 session 校验**：对于需要指定 session 的操作（如 `backend_operation_request`），后端会显式比对请求中的 `session_id` 与当前 `AppSession.id`：

   [app_session.py#L999-L1013](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/runtime/app_session.py#L999-L1013)

   ```python
   if request.session_id != self.id:
       _LOGGER.warning("Rejecting backend operation request: session ID mismatch")
       # ... 拒绝请求
       return
   ```

#### 完整的隔离层次图

```
┌──────────────────────────────────────────────────────────┐
│ 浏览器标签页 1                                            │
│  ┌────────────────────────────────────────────────────┐  │
│  │ WidgetStateManager (实例 1)                         │  │
│  │  ├─ widgetId_A → WidgetState (组件A的值)            │  │
│  │  └─ widgetId_B → WidgetState (组件B的值)            │  │
│  │                                                     │  │
│  │ WebSocket 连接 1 (session_id = "s1")               │  │
│  │  └─ 所有 BackMsg 都通过此连接发送到服务端的 s1 会话  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ 浏览器标签页 2                                            │
│  ┌────────────────────────────────────────────────────┐  │
│  │ WidgetStateManager (实例 2)                         │  │
│  │  ├─ widgetId_A → WidgetState (组件A的值, 可能不同)  │  │
│  │  └─ widgetId_B → WidgetState (组件B的值, 可能不同)  │  │
│  │                                                     │  │
│  │ WebSocket 连接 2 (session_id = "s2")               │  │
│  │  └─ 所有 BackMsg 都通过此连接发送到服务端的 s2 会话  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘

服务端:
  AppSession(s1) → SessionState → 独立的 widget 状态和脚本运行
  AppSession(s2) → SessionState → 独立的 widget 状态和脚本运行
```

### 7.4 v2 组件的会话关联

v2 组件的值回传不经过 postMessage，而是通过闭包中捕获的 `widgetMgr` 直接调用：

[useHandleJsContent.ts#L83-L104](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/lib/src/components/widgets/BidiComponent/hooks/useHandleJsContent.ts#L83-L104)

```typescript
const setStateValue = (name: string, value: T[keyof T]): void => {
  const existingValue = getWidgetValue()
  const newValue = { ...existingValue, [name]: value }
  void widgetMgr.setJsonValue(
    { id: componentIdForWidgetMgr, formId },  // <-- 同样使用唯一 widget ID
    newValue,
    { fromUi: true },
    fragmentId
  )
}
```

`componentIdForWidgetMgr` 就是 `element.id`（由 Python 端 `compute_and_register_element_id` 生成），与 v1 使用完全相同的 WidgetStateManager 和 widget ID 机制。因此 v2 组件的会话关联和隔离逻辑与 v1 在 `WidgetStateManager` 层之后完全一致。

### 7.5 不可变关联的保证

从 iframe 到 widget ID 的关联是**创建时绑定、不可篡改**的：

1. `ComponentInstance` 的 `onBackMsgRef` 在 `useEffect` 中被持续更新为最新的 `element`
2. `createIframeMessageHandler` 使用 `ref` 模式，始终读取最新的 `callbacks.current`
3. `element.id` 来自 Python 端的 protobuf，iframe 内的组件**无法修改**它
4. iframe 内的组件只负责发送 `{ value, dataType }`，**不参与** widget ID 的确定

这意味着：即使恶意 iframe 试图发送一个带有伪造 widget ID 的消息，这个 ID 也不会被使用——因为消息路由和 ID 匹配发生在宿主端的 `ComponentRegistry` 和 `createIframeMessageHandler` 中，iframe 只能触发预定义的消息类型，而消息处理逻辑使用的是宿主端闭包中绑定的 `element`。

---

## 八、关键文件索引

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
| [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/app/src/App.tsx) | 前端应用入口，sendRerunBackMsg 实现 |
| [ConnectionManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/connection/src/ConnectionManager.ts) | WebSocket 连接管理器 |
| [WebsocketConnection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/frontend/connection/src/WebsocketConnection.tsx) | WebSocket 连接实现，含 session token 传递 |
| [starlette_websocket.py](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/web/server/starlette/starlette_websocket.py) | 服务端 WebSocket 端点，session_id 路由 |
| [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/222-streamlit/lib/streamlit/runtime/app_session.py) | AppSession，会话级 BackMsg 处理和 rerun |
