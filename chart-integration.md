# Streamlit 图表接入链路分析

本文档梳理 Streamlit 图表系统的数据对象接入链路、前端渲染边界和异常兜底机制。代码以 `lib/streamlit/`（后端 Python）和 `frontend/lib/`（前端 React）为根。

## 1. 总体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户代码 (Python)                              │
│  st.line_chart(df) / st.plotly_chart(fig) / st.altair_chart(chart)  │
└──────────────────────────────────────┬──────────────────────────────┘
                                       │
┌──────────────────────────────────────▼──────────────────────────────┐
│                  后端图表 Mixin 层 (elements/*.py)                    │
│  - VegaChartsMixin / PlotlyMixin / PyplotMixin / GraphvizMixin 等    │
│  - 数据预处理: 列解析、类型推断、宽表转长表、颜色转换                   │
│  - 序列化: Arrow IPC (Vega/DataFrame) / JSON (Plotly/Graphviz)       │
└──────────────────────────────────────┬──────────────────────────────┘
                                       │ Protobuf (ForwardMsg.delta)
┌──────────────────────────────────────▼──────────────────────────────┐
│                       前端 React 组件层                               │
│  ArrowVegaLiteChart / PlotlyChart / GraphVizChart / PydeckChart 等  │
│  - Quiver 解析 Arrow bytes → 行对象数组                               │
│  - useVegaEmbed / react-plotly.js / d3-graphviz / deck.gl 渲染       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 后端数据接入链路

### 2.1 DeltaGenerator 与 Mixin 组合模式

图表 API 通过多重继承挂载到 `DeltaGenerator`：

- `lib/streamlit/delta_generator.py` 中 `class DeltaGenerator` 继承了所有图表 Mixin
- 每个图表类型独立实现为 Mixin（如 `VegaChartsMixin`、`PlotlyMixin`、`PydeckMixin`、`GraphvizMixin`）
- Mixin 中的方法最终调用 `self.dg._enqueue(element_type, proto)` 发送消息

### 2.2 Vega-Lite / Altair 图表链路（核心路径）

Vega-Lite 是 Streamlit 内置图表（line/bar/area/scatter）和 Altair 图表的底层渲染引擎。

#### 调用链

```
st.line_chart / st.bar_chart / st.area_chart / st.scatter_chart
    │
    └─► built_in_chart_utils.generate_chart()
          │  ├─ _prep_data() 数据预处理
          │  ├─ 构造 Altair Chart 对象 (mark_line/mark_bar/...)
          │  └─ 返回 Altair Chart
    └─► VegaChartsMixin._altair_chart()
          │
          ├─► _convert_altair_to_vega_lite_spec()
          │     将 Altair Chart → Vega-Lite spec dict
          │     - 注册 Altair data_transformer: _to_arrow_dataset()
          │     - 序列化为 Arrow bytes，以 data hash 作为 dataset name
          │     - chart.to_dict(format="vega")
          │
          └─► VegaChartsMixin._vega_lite_chart()
                │
                ├─► 参数校验（theme / on_select / width / height）
                ├─► _prepare_vega_lite_spec(spec)   渲染兼容处理
                ├─► _marshall_chart_data()          数据序列化
                ├─► _stabilize_vega_json_spec()     spec 稳定性处理
                ├─► selection_mode 解析与注册
                └─► self.dg._enqueue("vega_lite_chart", proto)
```

#### 关键文件与函数

| 组件 | 位置 | 说明 |
|------|------|------|
| VegaChartsMixin | `lib/streamlit/elements/vega_charts.py` | line_chart/bar_chart/area_chart/scatter_chart/altair_chart/vega_lite_chart 全部实现 |
| generate_chart | `lib/streamlit/elements/lib/built_in_chart_utils.py` | 内置图表主入口，返回 Altair Chart |
| _vega_lite_chart | `lib/streamlit/elements/vega_charts.py` | Vega-Lite 最终入队方法 |
| _marshall_chart_data | `lib/streamlit/elements/vega_charts.py` | 数据序列化为 Arrow 并填充 proto |

#### 数据预处理流水线（generate_chart 内部）

```
_prep_data()
  ├─► _maybe_reset_index_in_place()       将 DataFrame Index 转为普通列
  ├─► _drop_unused_columns()              删除 x_column/y_column/color_column 之外的列
  ├─► _maybe_convert_color_column_in_place()  颜色列映射: #RRGGBB / 调色板名称
  ├─► _convert_col_names_to_str_in_place()    列名强制转 string（Vega-Lite 要求）
  └─► _maybe_melt()                       宽表 → 长表（多列 y 时）
        ├─► _melt_data()
        │     ├─ mixed dtype + >100 unique → StreamlitAPIException
        │     └─ fix_arrow_incompatible_column_types() 修复 Arrow 类型兼容
        └─► _infer_vegalite_type()       pandas dtype → Vega-Lite type
              int/float → quantitative
              datetime → temporal
              bool/object/str → nominal
              category/ordered → ordinal
```

### 2.3 Plotly 图表链路

```
st.plotly_chart(figure_or_data)
    │
    └─► PlotlyMixin.plotly_chart()
          │
          ├─► plotly.io.to_json(figure)  → spec JSON string
          ├─► _resolve_content_width/height()  从 figure.layout 提取自然尺寸
          ├─► 支持 matplotlib Figure → plotly.tools.mpl_to_plotly() 转换
          ├─► selection_mode 解析
          ├─► PlotlyChartSelectionSerde 注册
          └─► self.dg._enqueue("plotly_chart", proto)
```

位置：`lib/streamlit/elements/plotly_chart.py`

### 2.4 Matplotlib / Pyplot 链路

Pyplot 不走原生 Web 渲染，而是渲染为 PNG 图片，复用 Image 组件传输：

```
st.pyplot(fig)
    │
    └─► PyplotMixin.pyplot()
          ├─► fig.savefig(buf, format="png", dpi=...)  Matplotlib 后端渲染
          ├─► marshall_images() 复用 Image proto
          └─► self.dg._enqueue("imgs", proto)
```

位置：`lib/streamlit/elements/pyplot.py`

### 2.5 Graphviz / PyDeck / Map 链路

| 图表类型 | 后端序列化方式 | proto |
|----------|---------------|-------|
| st.graphviz_chart | DOT source string | `graphviz_chart` (spec, engine) |
| st.pydeck_chart | Deck.GL JSON via pydeck.Deck.to_json() | `deck_gl_json_chart` |
| st.map | 内部调用 `_map()`，转 Arrow bytes + Deck.GL JSON | `deck_gl_json_chart` |

### 2.6 Bokeh 状态

Bokeh 已废弃，仅显示 deprecation warning 不做任何渲染：
`lib/streamlit/elements/bokeh_chart.py`

---

## 3. 数据序列化与传输

### 3.1 统一数据转换入口：convert_anything_to_arrow_bytes

`lib/streamlit/dataframe_util.py` 中 `convert_anything_to_arrow_bytes()` 是 Arrow 型图表数据序列化的统一入口，支持：

- `pa.Table`
- Polars DataFrame / LazyFrame
- Pandas DataFrame / Series / Index
- NumPy ndarray
- Python list / dict
- 以及 `pd.DataFrameStyler`、`Snowpark DataFrame` 等扩展类型

流程：
```
任意输入类型 → convert_anything_to_pandas_df() 归一化
             → pa.Table.from_pandas(df, preserve_index=...)
             → table.to_batches() → IPC StreamWriter → bytes
```

### 3.2 Protobuf 消息封装

图表数据封装在 `ForwardMsg.delta.new_element.<element_type>` 中：

```
ForwardMsg
└── delta
    ├── metadata.delta_path (元素在 DOM 树中的路径)
    └── new_element
        ├── vega_lite_chart
        │   ├── spec (Vega-Lite JSON string, stabilized)
        │   ├── data (Arrow bytes, 内联数据表)
        │   ├── datasets[] (多命名数据集 Arrow bytes)
        │   ├── use_container_width
        │   ├── theme ("streamlit" / "")
        │   ├── id / form_id (选择交互 widget 用)
        │   └── selection_mode[]
        ├── plotly_chart
        │   ├── spec (Plotly JSON string)
        │   ├── config
        │   ├── theme
        │   └── (同上 widget 相关字段)
        ├── graphviz_chart (spec, engine)
        └── deck_gl_json_chart (json, tooltip)
```

---

## 4. 前端渲染链路

### 4.1 Vega-Lite / Altair 前端渲染

核心组件：`frontend/lib/src/components/elements/ArrowVegaLiteChart/ArrowVegaLiteChart.tsx`

```
ArrowVegaLiteChart (接收 element: VegaLiteChartProto)
    │
    ├─► Quiver(element.data.data)  解析 Arrow IPC bytes
    │     → _columnNames, _data, _pandasIndexData, _dataColumnTypes
    │
    ├─► WrappedNamedDataset[] 解析 datasets 字段
    │
    ├─► useVegaElementPreprocessor()
    │     ├─► 注入 Streamlit 主题配置 (colors, fonts)
    │     ├─► autosize 处理（pad / fit-x / fit）
    │     ├─► isFullScreen 全屏模式适配
    │     └─► finalizeDataTransforms 修正数据路径
    │
    ├─► getDataArray(quiver, datasetsMap)  arrowUtils.ts
    │     ├─► Quiver → 行优先对象数组 {col: value, ...}[]
    │     ├─► Date 时区偏移修正（Arrow 存 UTC，Vega 解析为本地时的修正）
    │     ├─► BigInt → Number（Vega 不支持 BigInt，可能丢精度）
    │     └─► 注入 DataFrame Index 列
    │
    ├─► useVegaEmbed(spec, dataArray, datasets)  useVegaEmbed.ts
    │     ├─► createView()
    │     │     vegaEmbed(container, spec, {
    │     │       ast: true,        // CSP 合规: expr 编译为 AST 而非 Function
    │     │       theme: undefined, // 使用我们注入的主题而非 vega 默认主题
    │     │       defaultStyle: false
    │     │     })
    │     ├─► updateView()  增量更新
    │     │     ├─► 数据 hash 不变 → 跳过
    │     │     ├─► insert/remove/data 三种增量策略
    │     │     └─► view.change(dataset, changeset)
    │     └─► Vega 交互事件 → WidgetStateManager 回传
    │
    └─► 渲染:
          - showChart === true  → Vega 视图
          - showChart === false → ReadOnlyGrid（"Show data" 数据表格视图）
```

### 4.2 Plotly 前端渲染

核心组件：`frontend/lib/src/components/elements/PlotlyChart/PlotlyChart.tsx`

```
PlotlyChart
    │
    ├─► PlotlyWidthCheck
    │     └─► 宽度未就绪时返回 null，避免布局错乱
    │
    ├─► applyStreamlitTheme()  根据 Streamlit theme 覆盖 layout.colorway 等
    ├─► computeSelectionMode()  clickmode/dragmode/hovermode 动态配置
    ├─► FormClearHelper 集成：表单清空时重置选择
    └─► react-plotly.js <Plot> 组件渲染
          - MIN_WIDTH = 150, DEFAULT_HEIGHT = 450 兜底尺寸
          - Math.max(width, MIN_WIDTH) 防止 width=-1
```

### 4.3 Graphviz 前端渲染

组件：`frontend/lib/src/components/elements/GraphVizChart/GraphVizChart.tsx`

```
GraphVizChart
    │
    ├─► useCalculatedDimensions() 获取容器尺寸
    ├─► d3-graphviz library 渲染
    │     graphviz(#id).fit(true).engine(dot/neato/...).renderDot(spec)
    │     .width/height 显式设置（stretch 模式时）
    └─► try/catch 包裹 renderDot，异常仅 LOG.error 不崩溃
```

### 4.4 Deck.GL (PyDeck / Map) 前端渲染

组件：`frontend/lib/src/components/elements/DeckGlJsonChart/DeckGlJsonChart.tsx`

```
DeckGlJsonChart
    │
    ├─► useDeckGl() Hook 管理视图状态、选择交互、图层
    ├─► registerLoaders([CSVLoader, GLTFLoader])  注册数据加载器
    ├─► <DeckGL> + <StaticMap> (react-map-gl) 渲染
    └─► Toolbar 提供全屏 / 清除选择等操作
```

---

## 5. 前端渲染边界

### 5.1 Vega-Lite 渲染边界处理

| 边界问题 | 处理方式 | 位置 |
|----------|---------|------|
| facet / hconcat / repeat 图表 width=stretch 崩溃 | 后端 `_vega_lite_chart()` 检测这些类型，自动用 `width="content"` | `lib/streamlit/elements/vega_charts.py` |
| 嵌套 vconcat+hconcat infinite extent 错误 | `_has_nested_composition(spec)` 前后端双重检测，禁用强制宽度 | 前后端各一份实现 |
| null legend title 导致无图例 | `_patch_null_legend_titles()` 把 `null` 替换为 `" "`（空格字符串） | `lib/streamlit/elements/vega_charts.py` |
| autosize 配置冲突 | `_prepare_vega_lite_spec()` 统一处理：pad 类型 / fit 类型 / step 类型分别适配 | `lib/streamlit/elements/vega_charts.py` |
| Altair 全局计数器导致每次 rerun spec 变化 | `_stabilize_vega_json_spec()` 正则替换 `param_\d+` / `view_\d+` 后缀 | `lib/streamlit/elements/vega_charts.py` |
| 数据增量更新闪烁 | useVegaEmbed.updateData() 基于数据 hash 做 insert/remove/data 增量变更 | useVegaEmbed.ts |
| CSP 合规（禁止 `new Function`） | vegaEmbed 选项 `ast: true` 将表达式编译为 AST 而非函数字符串 | ArrowVegaLiteChart.tsx |

### 5.2 Plotly 渲染边界

| 边界问题 | 处理方式 |
|----------|---------|
| width=-1（容器未测量完成） | MIN_WIDTH=150 兜底，`Math.max(width, MIN_WIDTH)` |
| 布局缓存 fallback | `layout.width || MIN_WIDTH` 双重兜底 |
| 选择交互重置 | FormClearHelper 订阅表单清空事件 |
| 选择模式配置冲突 | `computeSelectionMode()` 基于 selectionMode 动态配置 clickmode/dragmode/hovermode |

### 5.3 Graphviz 渲染边界

| 边界问题 | 处理方式 |
|----------|---------|
| containerWidth / containerHeight < 0（测量前） | 渲染时 `width(containerWidth < 0 ? 0 : containerWidth)` 兜底 |
| DOT 语法错误 | useEffect 内 try/catch + LOG.error |

### 5.4 通用尺寸边界

- `useCalculatedDimensions()` Hook 统一处理容器尺寸测量
- `shouldWidthStretch(widthConfig)` / `shouldHeightStretch(heightConfig)` 判断拉伸模式
- Fullscreen 模式由 `ElementFullscreenContext` 提供尺寸覆盖

---

## 6. 异常兜底机制

### 6.1 后端异常类型

定义于 `lib/streamlit/errors/`，图表相关：

| 异常类 | 触发场景 |
|--------|---------|
| `StreamlitAPIException` | 参数非法（theme 不支持、on_select 非法值、width/height 非法等） |
| `StreamlitColumnNotFoundError` | 指定的 x_column / y_column / color_column 不在 DataFrame 中 |
| `StreamlitInvalidColorError` | color_column 值不是合法颜色格式（非 #RRGGBB、非已知调色板） |
| `StreamlitColorLengthError` | 自定义颜色数组长度与 Series 不匹配 |
| `StreamlitAPIException`（melt 场景） | 宽表转长表遇到 mixed dtype object 列且 unique > 100，Arrow 无法统一类型 |

### 6.2 数据兼容兜底

**Arrow 类型兼容修复**：`fix_arrow_incompatible_column_types()` 处理 melt 后列 dtype 不一致问题，将列转为 object 或统一类型。

**空数据占位符**：内置图表中使用 `_NON_EXISTENT_COLUMN_NAME = "__streamlit_non_existent_column__"` 标记不存在的列，避免 Altair 解析崩溃。

**空数据集处理**：数据为空时，前端 getDataArray 返回空数组，Vega 渲染空图表而非崩溃。

### 6.3 前端异常兜底

**React ErrorBoundary**：`frontend/lib/src/components/shared/ErrorBoundary/ErrorBoundary.tsx`

- `getDerivedStateFromError` 捕获 React 渲染异常，降级显示 `ErrorElement`
- `ChunkLoadError` 特殊处理：提示用户刷新页面（Streamlit 升级时前端代码版本不一致）
- 其他异常：显示错误名称、消息、堆栈
- ErrorBoundary 在 StreamlitMarkdown 等组件中广泛使用，但图表组件自身未直接包裹，依赖上层 Element 组件

**局部 try/catch**：

| 位置 | 捕获内容 |
|------|---------|
| GraphVizChart useEffect | graphviz.renderDot 渲染异常 → LOG.error |
| useVegaEmbed.ts createView | vegaEmbed 创建视图异常 → 回调 error，不崩溃 |
| arrowUtils.ts getDataArray | Quiver → 行对象数组转换异常 → 返回空数据 |

**WidgetStateManager 状态兜底**：

- Vega/Plotly 图表选择交互的反序列化异常会被捕获，返回默认空选择
- 表单清空时图表选择状态自动重置

---

## 7. 完整端到端示例：st.line_chart(df)

```
用户代码: st.line_chart(df, x="date", y=["sales", "profit"], color="region")
    │
    ▼
[Python] VegaChartsMixin.line_chart
    ├─ generate_chart(df, x, y, color, ChartType.LINE)
    │    ├─ _prep_data: reset_index, convert colors, melt wide→long
    │    └─ 返回 alt.Chart(mark="line").encode(x=..., y=..., color=...)
    │
    ├─ _altair_chart(altair_chart)
    │    ├─ Altair data_transformer: df → Arrow bytes, hash-based dataset name
    │    └─ chart.to_dict(format="vega") → Vega-Lite spec dict
    │
    └─ _vega_lite_chart(spec=vega_spec)
         ├─ _prepare_vega_lite_spec: autosize, null legend title patch
         ├─ _marshall_chart_data: spec/datasets → VegaLiteChartProto (Arrow bytes)
         ├─ _stabilize_vega_json_spec: 去除 Altair 计数器后缀
         └─ dg._enqueue("vega_lite_chart", proto)
              │
              ▼
[WebSocket] ForwardMsg { delta: { new_element: { vega_lite_chart: {...} } } }
              │
              ▼
[React] ArrowVegaLiteChart
    ├─ new Quiver(element.data.data) → Arrow 解析
    ├─ useVegaElementPreprocessor → spec 注入主题、全屏适配
    ├─ getDataArray(quiver) → [{date: ..., sales: ..., region: ...}, ...]
    ├─ useVegaEmbed → vegaEmbed(view, spec, data) → DOM 渲染 SVG/Canvas
    └─ Vega 视图交互事件 → WidgetStateManager → 回传 Python on_select 回调
```

---

## 8. 扩展图表类型的接入范式

新增一种图表库 X 的典型步骤：

1. **后端**：在 `lib/streamlit/elements/x_chart.py` 实现 `XMixin.x_chart()`
   - 接收用户图表对象，序列化为可传输格式（Arrow / JSON / bytes）
   - 构造对应 Protobuf message
   - 调用 `self.dg._enqueue("x_chart", proto, layout_config)`

2. **Proto**：在 protobuf 定义中新增 `XChart` message，注册到 `Element.new_element` oneof

3. **前端**：在 `frontend/lib/src/components/elements/XChart/` 新建组件目录
   - `XChart.tsx`：接收 Protobuf element，调用前端图表库渲染
   - `styled-components.ts`：样式
   - 可选：`useX.ts` Hook 管理状态
   - 集成 FullscreenWrapper、Toolbar、ErrorBoundary

4. **异常与边界**：
   - 后端参数校验 → StreamlitAPIException
   - 数据序列化异常 → 友好错误信息
   - 前端最小尺寸兜底、try/catch 渲染异常、空数据降级
