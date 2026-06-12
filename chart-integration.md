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

组件：[DeckGlJsonChart.tsx](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/DeckGlJsonChart.tsx)
状态管理：[useDeckGl.tsx](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx)
JSON 转换：[utils/jsonConverter.ts](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/utils/jsonConverter.ts)

```
DeckGlJsonChart
    │
    ├─► useDeckGl() Hook
    │     ├─► JSON5.parse(element.json) 解析 Deck.GL JSON spec
    │     ├─► 无 mapStyle 时按明/暗主题注入 Carto 底图（positron / dark-matter）
    │     ├─► Carto 底图未配 key → 自动注入 Streamlit 公用 cartoKey="x7g2plm9yq8vfrc"
    │     ├─► jsonConverter.convert() 将 @@type/@= 语法转为 Deck 图层实例
    │     │     注册 layers/aggregation-layers/geo-layers/mesh-layers/CARTO_LAYERS
    │     ├─► delete jsonCopy?.views 避免控制台警告
    │     ├─► 选择交互层（isSelectionModeActivated 时）：
    │     │     ├─► 无 pickable 定义 → 自动设 pickable=true
    │     │     ├─► 注入 selectedOpacity=255 / unselectedOpacity=102 (40%) 填充色
    │     │     └─► updateTriggers 绑定 selectedIndices / anyLayersHaveSelection 驱动颜色重算
    │     ├─► sanitizeSelection() 清理孤儿索引
    │     │     ├─► filterValidIndicesForLayer() 过滤越界索引，重新拉取当前对象
    │     │     └─► 图层无 ID / 非数组数据（URL/GeoJSON）→ 保留但无法验证
    │     └─► useStWidthHeight 尺寸计算
    │           fallback: initialViewState.height → theme.sizes.defaultMapHeight
    │
    ├─► registerLoaders([CSVLoader, GLTFLoader])  注册数据加载器
    ├─► viewState === null ? null : <DeckGL>   （避免 deck.gl runtime assertion）
    ├─► isInitialized 延迟一帧注入 layers（修复 HexagonLayers 首帧不渲染 bug）
    ├─► usesMapbox 检测 → 注入 MapBoxCss
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

### 5.4 PyDeck / Map 渲染边界

| 边界问题 | 处理方式 | 位置 |
|----------|---------|------|
| viewState 为 null | `viewState && <DeckGL>` 条件渲染，避免 deck.gl runtime assertion error | DeckGlJsonChart.tsx |
| HexagonLayers 首帧不渲染 | `useEffect` 延迟一帧 `setIsInitialized(true)`，`layers={isInitialized ? deck.layers : EMPTY_LAYERS}` | DeckGlJsonChart.tsx |
| 无 mapStyle 配置 | 按明/暗主题自动注入 Carto 公共底图（positron-gl-style / dark-matter-gl-style） | useDeckGl.tsx |
| Carto 底图未配 key | 自动注入 Streamlit 公用 cartoKey | useDeckGl.tsx |
| 未定义 `views` 字段引发控制台警告 | `delete jsonCopy?.views` 主动移除 | useDeckGl.tsx |
| 高度无配置 fallback | initialViewState.height → `theme.sizes.defaultMapHeight` | useDeckGl.tsx |
| **后端**：data=None 或 df.empty | 使用 `EMPTY_MAP = {initialViewState: {lat:0, lon:0, zoom:1}}` 渲染空白世界地图 | deck_gl_json_chart.py / map.py |
| **后端**：缺 lat/lon 列 | `StreamlitAPIException`，列出允许列名与现有列名 | map.py `_get_lat_or_lon_col_name()` |
| **后端**：lat/lon 列含 NaN/NaT/None | `StreamlitAPIException` 禁止空值 | map.py `_get_lat_or_lon_col_name()` |
| **后端**：color 列颜色格式非法 | `StreamlitAPIException: Column "X" does not appear to contain valid colors.` | map.py `_convert_color_arg_or_column()` |
| **后端**：pandas 3.x 兼容性 | `_prepare_pydeck_for_json()` 把 Layer DataFrame 弱引用转为 `list[dict]`（pydeck vars() 访问 DataFrame 在 pd3 失效） | deck_gl_json_chart.py |

### 5.5 通用尺寸边界

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
| GraphVizChart useEffect | graphviz.renderDot 渲染异常 → LOG.error，不影响其他组件 |
| useVegaEmbed.ts updateData view.remove | `view.remove(name, truthy)` 数据集已被删除 → 吞掉异常，继续执行 |
| arrowUtils.ts isFacetChart / hasNestedComposition | JSON.parse(spec) 失败 → 返回 false（视为非 facet/非嵌套，不阻断渲染） |

**WidgetStateManager 状态兜底**：

- Vega/Plotly 图表选择交互的反序列化异常会被捕获，返回默认空选择
- 表单清空时图表选择状态自动重置

### 6.4 Vega 视图创建失败的异常暴露路径

**关键发现**：`createView()` **没有 catch 异常**，且调用方是 fire-and-forget，Promise rejection 不会被 React ErrorBoundary 捕获。

完整暴露链路：

```
[ArrowVegaLiteChart.tsx] useLayoutEffect [L222-L241]
    │
    └─► createView(containerRef, spec)     // async，返回 Promise<VegaView|null>
          │  // eslint-disable-next-line @typescript-eslint/no-floating-promises
          │  // ⚠️ 无 await、无 .catch()、无 try/catch，fire-and-forget
          │
          └─► [useVegaEmbed.ts] createView [L111-L186]
                │
                ├─ if (containerRef.current === null) {
                │     throw new Error("Element missing.")   // 同步抛错
                │  }
                │
                ├─ try {
                │    await embed(container, spec, options)  // vegaEmbed [L137-L141]
                │    // 以下步骤均在 try 内：
                │    // maybeConfigureSelections、getDataArrays、insert data、runAsync、resize
                │  } finally {
                │    setIsCreatingView(false)   // 仅确保 loading state 复位 [L181-L183]
                │  }
                │  // ⚠️ 没有 catch 块！Promise rejection 直接向外抛出
                │
                ▼
          Promise rejection → 冒泡到浏览器全局 unhandledrejection 事件
          → 开发环境 React 控制台打印红色错误堆栈
          → 生产环境静默但 Vega 图表区域为空（view 未创建）
```

**可能触发 createView 失败的场景**：

| 场景 | 来源 |
|------|------|
| Vega-Lite spec 语法非法（缺 mark/encoding 等） | 用户 `st.vega_lite_chart(spec)` 传入非法 spec |
| infinite extent 错误（嵌套 vconcat+hconcat + width=stretch） | 虽然后端已检测，但手工构造非法 spec 仍可触发 |
| vega-embed CSP ast 模式下表达式编译失败 | ast=true + vega-interpreter 不兼容的自定义 expr |
| Arrow data 与 spec 字段名不匹配 | spec encoding 引用不存在的列 |
| 容器 DOM 节点被意外卸载 | `containerRef.current === null` 时主动 `throw new Error("Element missing.")` |
| Vega 表达式运行时错误 | `datum.non_existent_field * 2` 等运行时抛错 |

**核心要点**：
1. 异步 Promise rejection **不会被 React ErrorBoundary 捕获**（ErrorBoundary 仅捕获同步渲染异常）
2. 图表组件自身**没有包裹 ErrorBoundary**，依赖上层 Element 组件（通常是 App 的全局 ErrorBoundary）
3. `containerRef.current === null` 的同步抛错会被最近的 ErrorBoundary 捕获
4. `finally` 块确保 `isCreatingView` 状态复位，不会导致组件永久 loading

### 6.5 Arrow 数据解析：空数组 vs 直接抛错的边界

**核心调用链**：

```
ArrowVegaLiteChart [L137-L151] useMemo
    ├─► new Quiver(elementProto.data)  [L139]
    │     └─► parseArrowIpcBytes(arrowData.data)  Quiver.ts [L150]
    │           └─► tableFromIPC(ipcBytes)  arrowParseUtils.ts [L352]
    │                 ⚠️ 无 try/catch！
    │
    └─► wrapDatasets(elementProto.datasets)  [L141]
          └─► new Quiver(dataset.data)  [L121]

getDataArray(quiver)  arrowUtils.ts [L138-L201]
    ├─► if (numDataRows === 0) return []   空数据 → 静默
    └─► for (row, col) quiver.getCell(row, col)  越界 → 抛错
```

**返回空数组 `[]` 的场景（静默降级，不抛错）**：

| 触发条件 | 位置 | 行为 |
|----------|------|------|
| Quiver 行数为 0（`numDataRows === 0`） | [arrowUtils.ts#L138-L141](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/ArrowVegaLiteChart/arrowUtils.ts#L138-L141) | 返回 `[]`，Vega 渲染空图表 |
| `quiverData === null`（proto.data 为空） | [arrowUtils.ts#L84-L92](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/ArrowVegaLiteChart/arrowUtils.ts#L84-L92) | `getInlineData()` 返回 null，不调用 `view.insert()` |
| updateData 中新数据为空（0 行） | [useVegaEmbed.ts#L195-L204](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/ArrowVegaLiteChart/useVegaEmbed.ts#L195-L204) | 调用 `view.remove(name, truthy)`，dataset 已删除的异常被吞掉（catch 空） |
| datasets 为空数组 | [arrowUtils.ts#L111-L129](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/ArrowVegaLiteChart/arrowUtils.ts#L111-L129) | `getDataSets()` 返回 null，不处理 |

**直接抛错的场景（向上抛出，可导致组件崩溃）**：

| 触发条件 | 错误信息 | 位置 | 抛出时机 |
|----------|---------|------|----------|
| 输入 Arrow IPC bytes 损坏或格式非法 | apache-arrow 内部抛错（如 "Invalid IPC stream"） | [arrowParseUtils.ts#L346-L352](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/dataframes/arrowParseUtils.ts#L346-L352) | `new Quiver()` 同步抛，React 渲染阶段 |
| Pandas schema 中 index column 在 Arrow schema 中找不到 | `Index field ${indexCol} not found in arrow schema` | [arrowParseUtils.ts#L259-L264](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/dataframes/arrowParseUtils.ts#L259-L264) | `new Quiver()` 同步抛，React 渲染阶段 |
| Quiver.getCell 行索引越界 | `Row index is out of range: ${rowIndex}` | [Quiver.ts#L242-L244](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/dataframes/Quiver.ts#L242-L244) | `getDataArray()` 遍历抛，React 渲染阶段 |
| Quiver.getCell 列索引越界 | `Column index is out of range: ${columnIndex}` | [Quiver.ts#L245-L247](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/dataframes/Quiver.ts#L245-L247) | `getDataArray()` 遍历抛，React 渲染阶段 |
| Arrow 数据类型不兼容（如 Union 类型） | apache-arrow `tableFromIPC()` 抛错 | arrowParseUtils.ts | `new Quiver()` 同步抛 |

**关键边界**：
1. **同步抛错都会被 React ErrorBoundary 捕获**（因为发生在渲染阶段的 useMemo 中）
2. **0 行数据永远静默降级**，不会抛错
3. `tableFromIPC(ipcBytes)` **无任何 try/catch**，Arrow bytes 损坏直接崩溃
4. `view.remove(name, truthy)` 有 **空 catch 块**（[useVegaEmbed.ts#L198-L202](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/ArrowVegaLiteChart/useVegaEmbed.ts#L198-L202)），dataset 不存在时静默，不抛错
5. `isFacetChart()` 和 `hasNestedComposition()` 都有 **try/catch 返回 false**（[ArrowVegaLiteChart.tsx#L52-L105](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/ArrowVegaLiteChart/ArrowVegaLiteChart.tsx#L52-L105)），JSON 解析失败不阻断渲染

### 6.6 PyDeck / Map 的异常兜底

**后端兜底**：[deck_gl_json_chart.py](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/deck_gl_json_chart.py) / [map.py](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/map.py)

| 场景 | 处理方式 | 位置 |
|------|---------|------|
| `st.pydeck_chart(None)` | `json.dumps(EMPTY_MAP)`，渲染空白世界地图（经纬度 0,0，zoom=1） | [deck_gl_json_chart.py#L65-L67](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/deck_gl_json_chart.py#L65-L67) |
| `st.map(None)` / `st.map(empty_df)` | 复用 `_DEFAULT_MAP`（同 EMPTY_MAP） | [map.py#L53](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/map.py#L53) |
| PyDeck selection_mode 非法值 | `StreamlitAPIException` 列出合法选项 `{"single-object", "multi-object"}` | [deck_gl_json_chart.py#L93-L97](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/deck_gl_json_chart.py#L93-L97) |
| PyDeck selection_mode 传集合（多值） | `StreamlitAPIException` 不支持多值 | [deck_gl_json_chart.py#L88-L91](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/deck_gl_json_chart.py#L88-L91) |
| PyDeck on_select 非 `ignore/rerun/callable` | `StreamlitAPIException` | [deck_gl_json_chart.py#L548-L552](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/deck_gl_json_chart.py#L548-L552) |
| pandas 3.x + pydeck DataFrame 序列化失败 | `_prepare_pydeck_for_json()` 将 DataFrame 弱引用转为 `list[dict]` | [deck_gl_json_chart.py#L642-L680](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/deck_gl_json_chart.py#L642-L680) |
| `st.map` 找不到经纬度列 | `StreamlitAPIException` 列出候选列名与现有列名 | [map.py `_get_lat_or_lon_col_name()`] |
| `st.map` 经纬度列含空值 | `StreamlitAPIException` 禁止 NaN/NaT/None | [map.py `_get_lat_or_lon_col_name()`] |
| `st.map` color 列值非法 | `StreamlitAPIException` 提示颜色格式不合法 | [map.py `_convert_color_arg_or_column()`] |
| Mapbox token 缺失 | 优先取 pydeck_obj.mapbox_key，fallback 到 `config.get_option("mapbox.token")` | [deck_gl_json_chart.py#L538-L543](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/deck_gl_json_chart.py#L538-L543) |

**前端兜底**：[useDeckGl.tsx](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx)

| 场景 | 处理方式 | 位置 |
|------|---------|------|
| Deck.GL JSON 解析失败（非法 JSON5） | `JSON5.parse()` 抛错 → useMemo 抛错 → 祖先 ErrorBoundary 捕获 | [useDeckGl.tsx#L363-L367](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx#L363-L367) |
| 图层 id 缺失导致选择状态不可追踪 | `hasUnknownLayerId` 标记，所有旧选择被丢弃但不崩溃 | [useDeckGl.tsx#L185-L197](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx#L185-L197) |
| 数据长度缩水、选择索引越界 | `filterValidIndicesForLayer()` 过滤越界索引，重新拉取当前数据对象 | [useDeckGl.tsx#L224-L247](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx#L224-L247) |
| 图层被移除后仍保留旧选择 | `sanitizeSelection()` 检测 layerId 不存在则丢弃（全部无 id 时例外保留） | [useDeckGl.tsx#L255-L302](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx#L255-L302) |
| 视图状态 viewState 未初始化 | `viewState && <DeckGL>` 条件渲染，避免 deck.gl runtime assertion | [DeckGlJsonChart.tsx#L239-L270](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/DeckGlJsonChart.tsx#L239-L270) |
| HexagonLayers 首帧不渲染 bug | `useEffect` 延迟一帧 `setIsInitialized(true)`，layers 延迟注入 | [DeckGlJsonChart.tsx#L101-L106](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/DeckGlJsonChart.tsx#L101-L106) |
| WebGL 上下文不足 | 浏览器自动回收最早的 WebGL 上下文，Streamlit 仅文档提示 ≤8 张图 | [deck_gl_json_chart.py#L346-L352](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/deck_gl_json_chart.py#L346-L352) |
| tooltip 模板变量不存在（`{col_not_exist}`） | `interpolate()` 未匹配则保留原模板字符串，不抛错 | [useDeckGl.tsx#L104-L121](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx#L104-L121) |
| 未定义 views 字段引发控制台警告 | `delete jsonCopy?.views` 主动移除 | [useDeckGl.tsx#L533](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx#L533) |

**Serde 兜底**：[PydeckSelectionSerde](file:///d:/fz/0601/solo-dogfeeding/code/223-streamlit/lib/streamlit/elements/deck_gl_json_chart.py#L246-L270)

```python
def deserialize(self, ui_value: str | None) -> PydeckState:
    empty_state = {"selection": {"indices": {}, "objects": {}}}
    # ui_value is None → return empty_state
    # ui_value is "{}" (empty dict) → "selection" not in dict → return empty_state
    # ui_value is valid JSON → return parsed result
```

接收到空 dict `{}` 或缺失 `selection` 键时，返回 `EMPTY_STATE` 而非抛错。

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
