# Metric 指标渲染协同流程

本文档系统梳理 Streamlit 中 `st.metric()` 指标组件从 Python 后端到前端渲染的完整链路，重点阐述 **数值格式（format）**、**差值颜色（delta_color）** 与 **前端组件状态** 三者如何协同工作。

---

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Python 后端层                                │
│  st.metric() → 参数校验 → 格式/颜色/方向判定 → Protobuf 序列化     │
│                         ↓ (WebSocket ForwardMsg)                    │
├─────────────────────────────────────────────────────────────────────┤
│                        前端渲染层                                   │
│  ElementNodeRenderer → Metric 组件 → formatNumber 格式化            │
│     → metricColors 颜色映射 → styled-components 样式应用            │
│     → vega-embed 渲染 sparkline 图表                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、Python 后端：数据构造与决策

### 2.1 入口方法

核心入口为 [metric.py:MetricMixin.metric()](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/lib/streamlit/elements/metric.py#L92-L423)，主要参数：

| 参数 | 类型 | 作用 |
|------|------|------|
| `value` | 数字/字符串/None | 指标主值 |
| `delta` | 数字/字符串/None | 变化值（差值） |
| `delta_color` | `normal`/`inverse`/`off`/命名色 | 差值颜色策略 |
| `delta_arrow` | `auto`/`up`/`down`/`off` | 箭头方向控制 |
| `format` | 字符串/None | 数值格式化规范 |
| `chart_data` | 数值序列/None | 迷你图数据 |
| `chart_type` | `line`/`bar`/`area` | 迷你图类型 |
| `delta_description` | 字符串/None | 差值旁的描述文本 |

### 2.2 数值字符串化（value/delta → string）

#### `_parse_value()` — [metric.py:456-461](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/lib/streamlit/elements/metric.py#L456-L461)

```python
def _parse_value(value: Value) -> str:
    if value is None:
        return "—"               # None 渲染为长破折号
    if isinstance(value, str):
        return value              # 字符串原样保留（可含 Markdown）
    return from_number(value)      # 数字类型 → 调用 from_number
```

#### `_parse_delta()` — [metric.py:464-469](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/lib/streamlit/elements/metric.py#L464-L469)

```python
def _parse_delta(delta: Delta) -> str:
    if delta is None or delta == "":
        return ""                   # 空差值不渲染
    if isinstance(delta, str):
        return dedent(delta)        # 字符串去缩进后保留
    return from_number(delta)       # 数字类型 → from_number
```

#### `from_number()` — [string_util.py:269-305](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/lib/streamlit/string_util.py#L269-L305)

后端的 `from_number` **不做任何本地化或精度格式化**，仅做类型转换：
- Python `numbers.Number`（int/float/Decimal 等）→ `str(value)`
- NumPy 标量（有 `item()` 方法）→ `str(value.item())`

> **关键设计**：后端仅将数字转为最简单的字符串表示（如 `1234.567`），**真正的格式化工作推迟到前端** 依据用户的 `format` 参数执行。这样能利用浏览器的 `Intl.NumberFormat` 做本地化。

### 2.3 差值颜色与方向决策

#### `_determine_delta_color_and_direction()` — [metric.py:472-523](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/lib/streamlit/elements/metric.py#L472-L523)

这是 **后端最核心的协同决策函数**，返回 `MetricColorAndDirection(color, direction)` 数据类。

**决策步骤**：

```
delta 为空（None/""）?
  ├─ 是 → color=GRAY, direction=NONE  （无差值状态）
  └─ 否
       ├─ 方向判定：_is_negative_delta(delta)
       │    ├─ True  → direction=DOWN  （字符串以 "-" 开头）
       │    └─ False → direction=UP
       │
       └─ 颜色判定（依据 delta_color 参数）：
            ├─ delta_color ∈ 命名色 ("red"/"green" 等)
            │    → 直接映射 _DELTA_COLOR_TO_PROTO，方向按正负
            │
            ├─ "normal"（默认）
            │    ├─ is_negative → RED
            │    └─ 否则        → GREEN
            │
            ├─ "inverse"（反向，负向变好看的绿色）
            │    ├─ is_negative → GREEN   （如成本下降是好事）
            │    └─ 否则        → RED
            │
            └─ "off"
                 → GRAY（灰显，不表达褒贬）
```

#### `_is_negative_delta()` — [metric.py:526-527](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/lib/streamlit/elements/metric.py#L526-L527)

```python
def _is_negative_delta(delta: Delta) -> bool:
    return dedent(str(delta)).startswith("-")
```

> **注意**：此函数用 **字符串前缀匹配** 而非数值比较，这意味着：
> - 用户传入的字符串如 `"-1.2 °F"` 会被判定为负向
> - 但 `"下降 5%"` 这种不含 "-" 的中文描述会被判定为正向（方向=UP）

#### `delta_arrow` 覆盖逻辑 — [metric.py:384-389](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/lib/streamlit/elements/metric.py#L384-L389)

颜色/方向决策之后，`delta_arrow` 参数可 **强制覆盖方向**：

```python
if parsed_delta_arrow in _DELTA_ARROW_TO_PROTO:
    metric_proto.direction = _DELTA_ARROW_TO_PROTO[parsed_delta_arrow]
```

即 `"up"` → UP，`"down"` → DOWN，`"off"` → NONE，这会影响箭头显示但 **不改变颜色**。

### 2.4 Protobuf 消息结构

序列化后的消息定义见 [Metric.proto](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/proto/streamlit/proto/Metric.proto)：

```protobuf
message Metric {
  string label = 1;              // 标签（标题）
  string body = 2;               // 主值（已转字符串）
  string delta = 3;              // 差值（已转字符串，可能为空）

  enum MetricDirection { DOWN=0; UP=1; NONE=2; }
  MetricDirection direction = 4; // 箭头方向

  enum MetricColor { RED=0; GREEN=1; GRAY=2; ORANGE=3;
                     YELLOW=4; BLUE=5; VIOLET=6; PRIMARY=7; }
  MetricColor color = 5;         // 差值颜色（后端已决策好的枚举）

  string help = 6;
  LabelVisibility label_visibility = 7;
  bool show_border = 8;
  repeated double chart_data = 9; // sparkline 数值序列
  ChartType chart_type = 10;
  string format = 11;             // 格式化字符串（原样传给前端）
  string delta_description = 12;  // 差值旁的辅助文本
}
```

> **关键协同点**：`color` 和 `direction` 已经是 **后端决策好的枚举值**，前端无需再次判断正负；`format` 和 `body`/`delta` 的 **原始数字字符串** 一起传给前端，由前端执行实际格式化。

### 2.5 入队发送

`_enqueue()` 方法 — [delta_generator.py:473-569](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/lib/streamlit/delta_generator.py#L473-L569)：

1. 将 `MetricProto` 填入 `ForwardMsg.delta.new_element.metric`
2. 附加 `height_config` / `width_config` 布局信息
3. 设置 `metadata.delta_path` 标识 DOM 中的位置
4. 通过 `_enqueue_message(msg)` 经 WebSocket 推送到前端

---

## 三、前端层：组件渲染与协同

### 3.1 组件挂载点

[ElementNodeRenderer.tsx:507-524](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/core/Block/ElementNodeRenderer.tsx#L507-L524) 根据 `node.element.type === "metric"` 挂载组件：

```typescript
case "metric": {
  const metricProto = node.element.metric as MetricProto
  const hasChart = metricProto.chartData && metricProto.chartData.length > 0
  return (
    <ElementContainer
      config={hasChart ? LARGE_ELEMENT : DEFAULT}  // 有图则分配更大垂直空间
      isStale={isStale}                             // 数据过时标记
    >
      <Metric element={metricProto} {...elementProps} />
    </ElementContainer>
  )
}
```

### 3.2 Metric 组件主流程

[Metric.tsx:257-435](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/elements/Metric/Metric.tsx#L257-L435)

#### 3.2.1 Props 解构（L264-L277）

从 proto 解构出所有字段，其中 **协同关键字段** 为：
- `format`：格式化字符串
- `color`：已决策的 `MetricColor` 枚举
- `direction`：已决策的 `MetricDirection` 枚举
- `body` / `delta`：原始数字字符串

#### 3.2.2 数值格式化（L279-L288）

```typescript
const formattedMetricValue =
  format && isNumericString(metricValue)
    ? safeFormatNumber(metricValue, format)
    : metricValue

const formattedDelta =
  format && delta && isNumericString(delta)
    ? safeFormatNumber(delta, format)
    : delta
```

**两步判定**：
1. 必须用户显式传了 `format` 参数（非空）
2. `isNumericString()` 判定字符串为纯数字（可被 `Number()` 解析为有限数）

> **设计意图**：如果用户传入 `"$1,234.56"` 这类非纯数字字符串，说明用户自己已经格式化好了，**前端不应再处理**，避免重复格式化造成 `$$1,234.56` 之类的错误。

#### `safeFormatNumber()` — [Metric.tsx:56-63](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/elements/Metric/Metric.tsx#L56-L63)

```typescript
function safeFormatNumber(value: string, format: string): string {
  try {
    return formatNumber(Number(value), format)
  } catch {
    return value  // 格式化失败（如 printf 格式错）→ 回退原始值
  }
}
```

#### `formatNumber()` — [formatNumber.ts:91-202](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/util/formatNumber.ts#L91-L202)

支持的 format 类型与实现库：

| format 值 | 实现方式 | 示例输入 → 输出 |
|-----------|---------|----------------|
| `undefined`/`""` | **numbro**（自动 mantissa） | `1234.56789` → `"1234.5679"` |
| `"plain"` | numbro（mantissa=20, trimMantissa） | `1234.567` → `"1234.567"` |
| `"localized"` | **Intl.NumberFormat** 本机语言 | 1234.567 → 中文环境 `"1,234.567"` |
| `"percent"` | Intl（style:"percent"） | `12.345` → `"1,234.50%"` |
| `"dollar"` | Intl（currency:"USD" narrowSymbol） | `1234.567` → `"$1,234.57"` |
| `"euro"` | Intl（currency:"EUR"） | `1234.567` → `"€1,234.57"` |
| `"yen"` | Intl（currency:"JPY", 0 位小数） | `1234.567` → `"¥1,235"` |
| `"accounting"` | numbro（千分位 + 括号负数） | `-1234` → `"(1,234.00)"` |
| `"bytes"` | Intl（compact+byte 单位，替换"BB"→"GB"） | `1234` → `"1.2KB"` |
| `"compact"` / `"scientific"` / `"engineering"` | Intl（notation） | `1234` → `"1.2K"` |
| printf 格式（`"%.2f"`, `"%,d"`） | **sprintf.js**（支持 `,` `_` 千分位标志） | `1234.567, "%.2f"` → `"1234.57"` |

> **与后端的协同边界**：后端不做任何本地化格式化，只转纯数字串；前端根据浏览器语言环境做本地化。**同一个 format 参数在中美用户浏览器上会显示不同的千分位样式**。

#### `isNumericString()` — [formatNumber.ts:215-221](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/util/formatNumber.ts#L215-L221)

```typescript
export function isNumericString(value: string): boolean {
  if (value.trim() === "") return false
  const parsed = Number(value)
  return !Number.isNaN(parsed) && Number.isFinite(parsed)
}
```

> **注意事项**：带单位的字符串（`"-1.2 °F"`、`"70 °F"`）会被判定为非纯数字，**format 会被忽略**，只按原始字符串显示。这是后端 `from_number` 只接受纯数字类型时才能触发格式化的前端镜像约束。

#### 3.2.2.1 未传入 format 参数时的默认行为

当用户调用 `st.metric()` **未指定 `format` 参数** 时（`format=None` / proto 中 `format=""` 或未设置），前端会直接使用后端传来的 **原始字符串**，不做任何数值格式化处理。

关键代码逻辑在 [Metric.tsx:279-288](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/elements/Metric/Metric.tsx#L279-L288)：

```typescript
// 当 format="" 或未设置时，format 为 falsy，短路求值直接返回 metricValue
const formattedMetricValue =
  format && isNumericString(metricValue)
    ? safeFormatNumber(metricValue, format)
    : metricValue   // ← 默认路径：直接使用后端原始字符串

const formattedDelta =
  format && delta && isNumericString(delta)
    ? safeFormatNumber(delta, format)
    : delta         // ← 默认路径：直接使用后端原始字符串
```

**实际效果示例**：

| Python 调用 | 后端 body 字符串 | 前端显示 (无 format) | 前端显示 (format="compact") |
|------------|----------------|---------------------|----------------------------|
| `st.metric("A", 1234567)` | `"1234567"` | `"1234567"` | `"1.2M"` |
| `st.metric("A", 12.3456789)` | `"12.3456789"` | `"12.3456789"` | `"12.3457"` (numbro 自动精度) |
| `st.metric("A", None)` | `"—"` | `"—"` | `"—"` |
| `st.metric("A", "70 °F")` | `"70 °F"` | `"70 °F"` | `"70 °F"` (非数字，format 忽略) |

> **重要区分**：`format` 参数的默认值 `None` ≠ `formatNumber(..., undefined)`。`formatNumber` 在 `format=undefined` 时会使用 numbro 做自动精度格式化，但 Metric 组件在 `format` 未传入时**根本不会调用 formatNumber**，直接走原始字符串分支。这是有意为之的设计：保证向后兼容，不改变老版本用户既有的显示效果。

最终展示时，`formattedMetricValue` / `formattedDelta` 会被传入 `<StreamlitMarkdown>` 组件渲染（参考 [Metric.tsx:367-373](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/elements/Metric/Metric.tsx#L367-L373) 和 [Metric.tsx:396-402](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/elements/Metric/Metric.tsx#L396-L402)），支持有限的 Markdown 语法（粗体、斜体、内联代码、链接等），但禁止原始 HTML。

#### 3.2.3 箭头方向映射（L290-L302）

```typescript
switch (direction) {
  case DOWN:   metricDirection = ArrowDownward; break
  case UP:     metricDirection = ArrowUpward;   break
  case NONE:   metricDirection = null           // 不显示箭头
}
```

方向是 **后端已经决策好的枚举**，前端仅做图标映射。此处 `metricDirection = null` 时后续渲染会完全跳过箭头 `<Icon>` 节点，并通过 `showArrow={metricDirection !== null}` 通知 styled-component 调整内边距。

#### 3.2.4 前端状态管理（Hooks 协同）

| Hook | 用途 | 影响范围 |
|------|------|---------|
| `useRef<HTMLDivElement>(null)` (chartRef) | 持有 vega-embed 挂载的 DOM 节点 | 图表生命周期控制 |
| `useCalculatedDimensions()` (chartWidth) | 实时监听容器宽度，给 vega spec 提供可用宽度 | 图表响应式重绘 |
| `useEffect(..., [chartData, color, theme, chartWidth, chartType])` | 依赖变化时重新 `embed()` 渲染图表 | 图表数据/颜色/类型/宽度/主题变化触发重绘 |
| `useId()` (deltaA11yId) | 为 delta 描述文本生成无障碍 `aria-describedby` id | 无障碍语义 |
| `useEmotionTheme()` | 获取当前主题色板供颜色函数使用 | 明暗主题切换 |
| `memo(Metric)` | 对 props 浅比较，避免同数据下的重复渲染 | 性能优化 |

> **图表渲染的关键协同**：`color` 字段同时驱动 **差值徽章颜色** 和 **sparkline 颜色**。在 `getMetricChartSpec()` 中，折线/柱状/面积图的主色和面积图背景色均通过 `getMetricColor(theme, color)` 和 `getMetricBackgroundColor(theme, color)` 取得。

### 3.3 颜色系统协同

颜色映射在 [metricColors.ts](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/elements/Metric/metricColors.ts) 中集中管理，对外暴露三个函数：

#### `getMetricColor(theme, color)` — 主色（L27-L50）

用于 **折线描边、柱体填充、面积边界**。直接映射 `theme.colors`：

| Proto 枚举 | theme.colors 键 |
|-----------|----------------|
| RED | `redColor` |
| GREEN | `greenColor` |
| GRAY（default） | `grayColor` |
| ORANGE / YELLOW / BLUE / VIOLET | 对应 `orangeColor` 等 |
| PRIMARY | `primary`（主题主色） |

#### `getMetricTextColor(theme, color)` — 文本色（L87-L110）

用于 **差值徽章内的文字和图标颜色**。映射到 `*TextColor` 系列键，保证文字在背景上的对比度。

#### `getMetricBackgroundColor(theme, color)` — 背景色（L56-L81）

用于 **差值徽章背景、面积图填充色**。映射到 `*BackgroundColor` 系列键。

特殊情况：**PRIMARY 色没有预定义背景色**，需要实时计算：
```typescript
case MetricProto.MetricColor.PRIMARY:
  return transparentize(theme.colors.primary, lightTheme ? 0.9 : 0.7)
```
- 浅色主题：primary 颜色 alpha=10%（透明化 90%）
- 深色主题：primary 颜色 alpha=30%（透明化 70%）

> **主题一致性保证**：整个 Metric 组件（差值徽章文字/背景 + 图表线/背景/填充）共用 **同一套 color → 主题色** 的映射函数，视觉上形成统一的指标语义色。

### 3.4 样式应用（styled-components）

[styled-components.ts](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/elements/Metric/styled-components.ts) 定义的样式与数据协同：

#### `StyledMetricDeltaText`（L87-L110）— 差值徽章

```typescript
color: getMetricTextColor(theme, metricColor),        // 文字色
backgroundColor: getMetricBackgroundColor(theme, metricColor),  // 徽章背景
...(showArrow && { paddingLeft: twoXS })              // 箭头存在时左内边距调小
```

这里的 `metricColor` 来自 **proto 的 color 枚举**，而非前端重新计算。`showArrow` 则由 direction 决定，用于精细控制边距（因为 ArrowUpward 图标自带 2px padding）。

#### `StyledMetricValueText`（L74-L80）

- 使用 `theme.fontSizes.metricValueFontSize`（大字号粗体）
- 颜色为中性的 `bodyText`，**不受 delta_color 影响**（始终是默认文字色）

#### `StyledMetricContainer`（L33-L42）

`showBorder` 控制是否展示卡片边框与圆角。此参数同样来自 proto。

### 3.5 Stale 状态机制（isStale）

Stale 状态用于在脚本 **重新运行期间**（如用户点击按钮、修改输入触发 rerun），视觉上标记那些来自 **上一轮 scriptRunId**、尚未被新数据替换的元素，提示用户这些内容可能已过时。

#### 3.5.1 状态计算逻辑

isStale 的布尔值由 [utils.ts:isElementStale()](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/core/Block/utils.ts#L43-L70) 函数计算：

```typescript
export function isElementStale(
  node: AppNode,
  scriptRunState: ScriptRunState,
  scriptRunId: string,          // 当前正在运行的 scriptRunId（来自 ScriptRunContext）
  fragmentIdsThisRun?: string[]
): boolean {
  if (scriptRunState === ScriptRunState.RERUN_REQUESTED) {
    return true                   // 只要刚请求 rerun，全部元素标记为 stale
  }
  if (scriptRunState === ScriptRunState.RUNNING) {
    if (fragmentIdsThisRun?.length) {
      // Fragment 部分 rerun：仅标记同一 fragmentId 下 scriptRunId 不同的元素
      return Boolean(
        node.fragmentId &&
        fragmentIdsThisRun.includes(node.fragmentId) &&
        node.scriptRunId !== scriptRunId
      )
    }
    return node.scriptRunId !== scriptRunId   // 全量 rerun：比较元素节点 scriptRunId
  }
  return false                    // 未运行状态下不标记为 stale
}
```

其中 `node.scriptRunId` 在 [ElementNode.ts](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/render-tree/ElementNode.ts#L33-L58) 构造函数中赋值——每个元素节点被创建时，记录其所属的后端 script run 标识。当前 `scriptRunId` 从 [ScriptRunContext](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/core/ScriptRunContext.tsx) 获取，随每次后端推送新的 ForwardMsg 更新。

#### 3.5.2 状态传入路径

从 `ScriptRunContext` → `ElementNodeRenderer` → `ElementContainer` → styled-components：

1. [ElementNodeRenderer.tsx:1249-1250](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/core/Block/ElementNodeRenderer.tsx#L1249-L1250) 读取上下文并计算：
   ```typescript
   const { scriptRunState, scriptRunId, fragmentIdsThisRun } = useContext(ScriptRunContext)
   ```
   然后对每个元素节点调用 `isElementStale(node, state, id, fragmentIds)`，将结果以 `isStale` prop 传入 `<ElementContainer>`。

2. [ElementContainer.tsx:88-89](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/core/Block/ElementContainer.tsx#L88-L89) 继续传递并加"全屏豁免"逻辑：
   ```typescript
   data-stale={isStale}
   isStale={isStale && !isFullScreen}   // 全屏模式下不减淡，保持可读性
   ```

3. [styled-components.ts:116](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/core/Block/styled-components.ts#L116) 应用样式：
   ```typescript
   ...(isStale && elementType !== "skeleton" && STALE_STYLES)
   ```
   其中 `skeleton`（骨架屏元素）不参与淡化，避免骨架屏加载中被进一步淡化。

#### 3.5.3 视觉效果

[consts.ts:STALE_STYLES](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/theme/consts.ts#L19-L24)：

```typescript
export const STALE_TRANSITION_PARAMS = "1s ease-in 0.5s"
export const STALE_STYLES = {
  opacity: opacities.stale,   // opacities.stale = 0.33（透明度 33%）
  transition: `opacity ${STALE_TRANSITION_PARAMS}`,
}
```

**实际视觉行为**：
- 脚本触发重新运行后，元素等待 **0.5 秒**（避免快速 rerun 时闪烁）
- 随后在 **1 秒内** 通过 ease-in 缓动淡出至 **33% 不透明度**
- 后端新数据到达后（元素 scriptRunId 与当前相等），立即恢复 100%
- Metric 组件 **没有独立的 hideIfStale 处理**（与 balloons 等动画元素不同），只受透明度淡化影响，DOM 节点仍然存在并占空间

### 3.6 Sparkline Hover 交互与高亮实现

当 metric 提供 `chart_data` 时，前端使用 vega-embed 渲染 Vega-Lite 图表，并内置三层图层结构实现 **hover 最近点检测 → 高亮圆点 → 显示 tooltip** 的交互体验。核心实现位于 [Metric.tsx:getMetricChartSpec()](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/elements/Metric/Metric.tsx#L76-L251)。

#### 3.6.1 Vega-Lite 三层图层架构

```
Layer 0 (chart_mark)        主视觉图层：实际线/柱/面积图
  ├─ 线型：strokeCap:round, strokeWidth:metricStrokeWidth
  ├─ 柱型：cornerRadius:full（圆角柱）
  └─ 面积：面积背景 + 顶部描边双线结构

Layer 1 (points)             透明点图层：用于 hover 事件捕获（视觉不可见）
  └─ mark: { type: "point", opacity: 0 }   ← 完全透明

Layer 2 (highlighted_points) 高亮图层：显示最近点（hover 时才可见）
  ├─ mark: { type: "point", filled:true, size:65, tooltip:true }
  └─ transform: [{ filter: { param: hover_selection, empty:false } }]
```

#### 3.6.2 Hover 选择参数（Layer 1）

Layer 1 中声明了一个名为 `${baseName}_hover_selection` 的 Vega 选择参数：

```typescript
params: [{
  name: `${baseName}_hover_selection`,
  select: {
    type: "point",       // 选择单个数据点
    encodings: ["x"],    // 按 x 轴方向匹配（最近横坐标点）
    nearest: true,       // 选择距离鼠标最近的点，而非精确命中
    on: chartData.length > LARGE_DATASET_POINT_THRESHOLD
      ? "mousemove{16}"  // 大数据 (>1000 点) 节流到 16ms 约 60fps
      : "mousemove",     // 普通数据实时响应
    clear: "mouseleave", // 鼠标离开图表时清除选择
  },
}]
```

**关键设计点**：
- **`nearest: true`**：鼠标不必精准落在数据线上，只要在图表范围内，Vega 会按距离自动选择最近的 x 坐标点，大幅提升可用性
- **大数据节流**：`mousemove{16}` 是 Vega 的事件节流语法，避免 1000+ 点时 hover 计算阻塞主线程
- **`encodings: ["x"]`**：仅按 x（时间索引）匹配，不考虑 y 值，使垂直方向的查找更宽容

#### 3.6.3 高亮圆点（Layer 2）

通过 `transform` 过滤，仅渲染当前被 hover 选中的点：

```typescript
{
  name: `${baseName}_highlighted_points`,
  transform: [
    { filter: { param: `${baseName}_hover_selection`, empty: false } }
  ],
  mark: {
    type: "point",
    filled: true,   // 实心圆点（非空心）
    size: 65,       // 像素面积（约直径 9px）
    tooltip: true,  // 启用 Vega 默认 tooltip 管线，后续被 vega-embed 覆盖
  },
  encoding: { /* 继承主层 x/y 编码 */ }
}
```

- **`empty: false`**：无选择时整个 filter 返回空数组，不渲染任何圆点 → 常态下无高亮
- **颜色继承**：高亮圆点的颜色从 `config.mark.color` 继承，即 `getMetricColor(theme, color)`，与图表主线/柱颜色一致，形成统一视觉语言

#### 3.6.4 Tooltip 自定义

在调用 `vega-embed` 时注入自定义 tooltip 配置 — [Metric.tsx:329-336](file:///d:/fz/0601/solo-dogfeeding/code/235-streamlit/frontend/lib/src/components/elements/Metric/Metric.tsx#L329-L336)：

```typescript
tooltip: {
  theme: "custom",   // 套用 Streamlit 主题化 tooltip 样式
  formatTooltip: (value: { y: number }) => `${value.y}`
}
```

由于 sparkline 的 x 值只是 `[0, 1, 2, ...]` 的数字索引（无业务含义），`formatTooltip` 函数剥离了 x 值，仅 **展示 y 方向的原始数值**。同时 `theme: "custom"` 触发 Streamlit 自定义 tooltip 样式（边框、字体、阴影）与整体 UI 一致。

#### 3.6.5 Hover 效果协同总览

```
鼠标移至图表区域
  │
  ▼ Vega 引擎
Layer 1 捕获 mousemove → nearest=true 找最近 x 点
  │
  ├─ 更新 ${baseName}_hover_selection 参数
  │
  ▼
Layer 2 transform.filter 收到参数匹配
  │
  ├─ 渲染 size=65 的实心圆点（与主线同色）
  │   （视觉高亮反馈）
  │
  ▼ vega-embed tooltip 插件
tooltip 命中 point 标记 → formatTooltip(y 值) → 自定义样式弹出
  │
  ▼
鼠标离开图表 → Layer 1 clear:mouseleave 清空选择
  │
  └─ Layer 2 filter 空数组 → 圆点消失 + tooltip 关闭
```

> **与 color 系统的协同**：整个 hover 高亮（高亮圆点颜色、tooltip 文字主题、线/柱颜色）全部由后端决策好的同一个 `color` 枚举驱动，经过 `getMetricColor()` 映射到实际主题色，保持差值徽章与图表交互的视觉一致性。

---

## 四、完整协同链路时序

以一次调用 `st.metric("销售", 12345.67, delta=-830, delta_color="inverse", format="dollar")` 为例：

```
用户代码
  │
  ▼
metric.py: metric("销售", 12345.67, -830, "inverse", format="dollar")
  │
  ├─ _parse_value(12345.67) → "12345.67"      [纯数字字符串]
  ├─ _parse_delta(-830)    → "-830"           [纯数字字符串，带 - 前缀]
  │
  ├─ _determine_delta_color_and_direction("inverse", -830)
  │    ├─ _is_negative_delta("-830") → True   [以 - 开头]
  │    ├─ direction = DOWN                    [负差值]
  │    └─ delta_color="inverse" + 负 → GREEN  [成本下降是好事 → 绿色]
  │    → MetricColorAndDirection(GREEN, DOWN)
  │
  ├─ MetricProto { body="12345.67", delta="-830",
  │                color=GREEN, direction=DOWN,
  │                format="dollar", ... }
  │
  ▼
delta_generator._enqueue("metric", proto)
  │  → ForwardMsg 通过 WebSocket 发送
  ▼
前端 ElementNodeRenderer 收到 type="metric"
  │
  ▼
Metric 组件渲染
  │
  ├─ 格式化阶段：
  │    ├─ format="dollar" && isNumericString("12345.67")
  │    │   → formatNumber(12345.67, "dollar") = "$12,345.67"   [按浏览器本地化]
  │    └─ format="dollar" && isNumericString("-830")
  │        → formatNumber(-830, "dollar") = "-$830.00"
  │
  ├─ 颜色阶段：
  │    ├─ color=GREEN
  │    │   ├─ 徽章文字 → theme.colors.greenTextColor
  │    │   ├─ 徽章背景 → theme.colors.greenBackgroundColor
  │    │   └─ 图表线色 → theme.colors.greenColor
  │    └─ direction=DOWN → ArrowDownward 图标
  │
  └─ 状态阶段：
       ├─ useCalculatedDimensions → 为图表算宽度
       └─ useEffect → (如有 chartData) 嵌入 vega chart
```

---

## 五、关键设计要点总结

1. **决策与执行分离**
   - 后端只做 **颜色/方向的逻辑决策**（将用户参数 + 差值正负 → proto 枚举）
   - 前端只做 **视觉执行**（枚举 → 实际 CSS 颜色 + 格式化字符串）
   - 两端通过 Protobuf 的强类型枚举解耦，避免重复判定逻辑。

2. **格式化的前后端边界与默认行为**
   - 后端只保证数字→可被解析的字符串，**不做本地化**
   - 前端利用浏览器 `Intl.NumberFormat` 按用户语言环境格式化
   - 通过 `isNumericString` 守门，避免对已格式化的字符串重复处理
   - **默认 `format=None` 时完全跳过格式化**，直接使用后端原始字符串（经 StreamlitMarkdown 渲染），保证向后兼容不破坏既有显示。这是一个"功能默认关闭"的渐进式设计。

3. **颜色的多重视觉通道**
   同一个 `color` 枚举统一驱动：差值徽章 **文字色 + 背景色** + sparkline 图表 **主线色/面积填充色** + **hover 高亮圆点颜色**。四处使用不同函数映射，既保证徽章的文本对比度，又保证图表视觉层次，还保持交互反馈的语义一致性。

4. **方向与箭头的独立控制**
   - `delta_arrow` 参数可强制覆盖 direction（影响箭头显示），但不影响 color
   - `delta_color="off"` 强制 GRAY，但不覆盖 direction（箭头仍显示）
   - 两者正交解耦，满足 "只变色不变方向" 或 "只变方向不变色" 的灵活需求

5. **前端状态轻量**
   Metric 是 **纯展示组件**（无内部交互状态），仅使用 Hooks 管理图表引用和响应式尺寸，所有语义状态（值、颜色、方向）来自 props，天然适合 `memo` 优化。

6. **Stale 机制的分层传递与豁免**
   - 通过 `ScriptRunContext → ElementNodeRenderer → ElementContainer → styled-components` 四层管道传递 isStale 信号
   - 内置 **三大豁免**：全屏模式不减淡（避免全屏报告内容被淡化）、骨架屏不减淡（避免加载态被二次弱化）、非 RUNNING/RERUN_REQUESTED 状态不减淡（保证稳定渲染）
   - 通过 0.5s 延迟 + 1s ease-in 过渡避免快速 rerun 的视觉闪烁，仅在长计算时才让用户感知到过时标记

7. **Sparkline Hover 的声明式三层架构**
   - 完全在 Vega-Lite 声明式 spec 中实现交互，不依赖 React 事件监听，避免 DOM 事件与 SVG 的坐标换算
   - **三层职责分离**：Layer 0 纯展示、Layer 1 纯捕获（视觉不可见）、Layer 2 纯反馈（按需渲染），互不干扰
   - 通过 `nearest + encodings:["x"]` 降低交互精度要求，1000+ 大数据集时自动节流 mousemove 到 16ms，在可用性和性能间取得平衡
