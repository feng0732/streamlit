# Streamlit 图表接入链路分析

本文档梳理 Streamlit 图表系统的数据对象接入链路、前端渲染边界和异常兜底机制。路径以项目根目录为基准，后端代码在 `lib/streamlit/`，前端代码在 `frontend/lib/`。

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
| st.pydeck_chart | `pydeck_obj is None` 时序列化为 `EMPTY_MAP`（仅 initialViewState，无 layers）；否则 `pydeck_obj.to_json()` 生成完整 Deck.GL JSON | `deck_gl_json_chart` |
| st.map | `data is None` 或 `data.empty` 时序列化为 `_DEFAULT_MAP`（同 EMPTY_MAP）；有数据时深拷贝 `_DEFAULT_MAP`，覆盖 initialViewState 为自动计算的居中坐标和缩放级别，并追加 `layers` 数组含 ScatterplotLayer | `deck_gl_json_chart` |

> **注意**：PyDeck 和 st.map 都**不使用 Arrow IPC** 传输数据，而是将图层数据序列化为 JSON 内嵌在 spec 中。st.map 的 ScatterplotLayer 数据通过 `df.to_dict(orient="records")` 转为对象数组，直接嵌入 Deck.GL JSON 的 `layers[0].data` 字段。
>
> **st.map 空数据 vs 有数据的关键差异**：
> - **空数据**（None / empty df）：`EMPTY_MAP` / `_DEFAULT_MAP` 只有 `{"initialViewState": {"latitude": 0, "longitude": 0, "pitch": 0, "zoom": 1}}`，**无 layers 字段**，前端渲染一个以 (0,0) 为中心 zoom=1 的空底图
> - **有数据**：深拷贝 `_DEFAULT_MAP` 后通过 `_get_viewport_details(df, lat, lon, zoom)` 自动计算居中坐标（经纬度中点）和缩放级别（基于经纬度范围查 `_ZOOM_LEVELS` 表），然后添加 ScatterplotLayer
>
> 代码位置：`lib/streamlit/elements/map.py` 中 `to_deckgl_json()` / `_get_viewport_details()` / `_get_zoom_level()`

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
状态管理：`frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx`
JSON 转换：`frontend/lib/src/components/elements/DeckGlJsonChart/utils/jsonConverter.ts`

```
DeckGlJsonChart
    │
    ├─► useDeckGl() Hook
    │     ├─► JSON5.parse(element.json) 解析 Deck.GL JSON spec
    │     │     ⚠️ 无 try/catch，解析失败直接抛错到祖先 ErrorBoundary
    │     ├─► 无 mapStyle 时按明/暗主题注入 Carto 底图（positron / dark-matter）
    │     ├─► Carto 底图未配 key → 自动注入 Streamlit 公用 cartoKey="x7g2plm9yq8vfrc"
    │     ├─► jsonConverter.convert() 将 @@type/@= 语法转为 Deck 图层实例
    │     │     注册 layers/aggregation-layers/geo-layers/mesh-layers/CARTO_LAYERS
    │     ├─► delete jsonCopy?.views 避免控制台警告
    │     ├─► 选择交互层（isSelectionModeActivated 时）：
    │     │     ├─► 无 pickable 定义 → 自动设 pickable=true
    │     │     ├─► 注入 selectedOpacity=255 / unselectedOpacity=102 (40%) 填充色
    │     │     └─► updateTriggers 绑定 selectedIndices / anyLayersHaveSelection 驱动颜色重算
    │     ├─► sanitizeSelection() 清理孤儿索引（见 4.5 节）
    │     │     ├─► filterValidIndicesForLayer() 过滤越界索引，重新拉取当前对象
    │     │     └─► 图层无 ID / 非数组数据（URL/GeoJSON）→ 保留但无法验证
    │     ├─► viewState 初始化：
    │     │     useState(null) → useEffect 监听 deck.initialViewState 变化 → setViewState
    │     │     首帧 viewState===null，<DeckGL> 不挂载（条件渲染）
    │     └─► useStWidthHeight 尺寸计算
    │           fallback: initialViewState.height → theme.sizes.defaultMapHeight
    │
    ├─► registerLoaders([CSVLoader, GLTFLoader])  注册数据加载器
    ├─► viewState === null ? null : <DeckGL>   （null 时不挂载，避免 deck.gl runtime assertion）
    ├─► isInitialized 延迟一帧注入 layers（修复 HexagonLayers 首帧不渲染 bug）
    ├─► usesMapbox 检测 → 注入 MapBoxCss
    ├─► 空数据场景（EMPTY_MAP）：
    │     deck.layers 为空数组，仅渲染底图瓦片，无数据图层
    │     initialViewState.latitude=0, longitude=0, zoom=1 → 非洲外海
    └─► Toolbar 提供全屏 / 清除选择等操作
```

### 4.5 PyDeck 选择状态的延续与清理机制

PyDeck 选择状态的生命周期由**后端 key_as_main_identity 延续** + **前端 sanitizeSelection 清理** 两段机制共同实现。

#### 后端：配置变更时的状态延续

```
st.pydeck_chart(chart, key="my_map", on_select="rerun")
    │
    ▼
[lib/streamlit/elements/deck_gl_json_chart.py] pydeck_chart()
    └─► compute_and_register_element_id(
          "deck_gl_json_chart",
          user_key=key,
          key_as_main_identity={"selection_mode"},    ← 关键
          dg=self.dg,
          is_selection_activated=is_selection_activated,
          selection_mode=selection_mode,
          use_container_width=use_container_width,
          spec=spec,
        )
```

**`key_as_main_identity={"selection_mode"}` 的作用**：
- **有 key 时**：只有 `selection_mode` 会参与 element ID 计算。这意味着**数据变化、spec 变化、图层增删、尺寸参数变化都不会改变 element ID**，选择状态会在这些配置变更时延续。
- **无 key 时**：所有参数（包括 spec）都会参与 ID 计算，配置变更后 element ID 变化，选择状态被重置。
- 代码注释明确指出："This allows selection state to persist across data/spec changes. Note: This can lead to orphaned selections if data length shrinks, but the frontend handles this by sanitizing invalid indices."

#### 前端：spec 变化时的孤儿索引清理

```
[useDeckGl.tsx] 组件挂载 / spec 变化
    │
    ├─► useExecuteWhenChanged() 依赖 [parsedPydeckJson, isSelectionModeActivated]
    │     每次 parsedPydeckJson 变化（即每次 rerun 收到新 spec）时执行
    │
    ├─► getLayerDataInfo(parsedPydeckJson.layers)  构建当前图层信息
    │     ├─► 遍历所有图层，收集 {layerId → dataArray | undefined}
    │     ├─► 无 id 的图层 → hasUnknownLayerId = true
    │     └─► 非数组数据（URL 字符串 / GeoJSON 对象）→ 存 undefined，表示无法验证索引
    │
    └─► sanitizeSelection(currentSelection, layerDataInfo)
          │
          ├─► 遍历所有已选 layerId:
          │    │
          │    ├─► 图层已删除（layerData.has(layerId) === false）：
          │    │     ├─► 全部图层都无 ID 且 layerData 为空 → 保留选择（无法验证）
          │    │     └─► 否则 → 丢弃该图层的所有选择，changed=true
          │    │
          │    ├─► 图层存在但数据非数组（layerDataForId === undefined）：
          │    │     └─► 保留所有选择（无法验证索引），changed=false
          │    │
          │    └─► 图层存在且数据是数组：
          │          ├─► filterValidIndicesForLayer(indices, objects, layerDataForId)
          │          │     ├─► 遍历每个已选索引 idx
          │          │     ├─► idx < data.length → 保留，更新 object 为当前数据[idx]
          │          │     │     （如果对象变化了 → changed=true）
          │          │     └─► idx >= data.length → 丢弃，changed=true
          │          └─► 剩余索引 > 0 → 写入新 selection，否则清空
          │
          └─► changed === true → setSelection() 写回 widget 状态，触发一次同步
                fromUi: false，表示是程序清理而非用户操作
```

**清理规则速查表**：

| 场景 | 处理方式 | changed |
|------|---------|---------|
| 图层已删除 + 至少一个图层有 ID | 丢弃该图层所有选择 | true |
| 图层已删除 + 全部图层无 ID + 无数据数组 | 保留所有选择（无法验证） | false |
| 图层存在 + 数据是 URL/GeoJSON（非数组） | 保留所有选择（无法验证索引） | false |
| 图层存在 + 数据是数组 + 索引越界 | 丢弃越界索引 | true |
| 图层存在 + 数据是数组 + 索引有效但对象已变 | 保留索引，更新 object | true |
| 图层存在 + 数据是数组 + 索引有效 + 对象未变 | 保留 | false |

#### 典型场景示例

**场景 1：数据长度缩减**
```
初始数据: 100 条，已选索引 [50, 60, 70]
config 变更: df = df.head(55) → 剩 55 条
sanitizeSelection:
  70 >= 55 → 丢弃
  50, 60 中，60 >= 55 → 丢弃
  最终: [50]，changed=true → setSelection 回写
```

**场景 2：图层被删除**
```
初始: 两个图层 A(id="layer1") 和 B(id="layer2")，各选 [1,2]
config 变更: 删除图层 B
sanitizeSelection:
  layer1: 存在，保留 [1,2]
  layer2: layerData.has("layer2") === false → 丢弃
  最终: {"layer1": [1,2]}，changed=true
```

**场景 3：图层无 ID**
```
初始: 图层未设置 id，已选 [0, 1, 2]
config 变更: 数据从 100 条缩到 3 条
sanitizeSelection:
  hasUnknownLayerId = true，layerData.size = 0
  → 全部保留，无法验证
  最终: [0, 1, 2]，changed=false
```

**场景 4：数据为 GeoJSON 对象**
```
图层 data 是 {type: "FeatureCollection", features: [...]}（非数组）
sanitizeSelection:
  layerDataForId = undefined（非数组）
  → 保留所有选择，无法验证索引
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
| 数据增量更新闪烁 | useVegaEmbed.updateData() 基于数据 hash 做 insert/remove/data 增量变更 | `frontend/lib/src/components/elements/ArrowVegaLiteChart/useVegaEmbed.ts` |
| CSP 合规（禁止 `new Function`） | vegaEmbed 选项 `ast: true` 将表达式编译为 AST 而非函数字符串 | `frontend/lib/src/components/elements/ArrowVegaLiteChart/ArrowVegaLiteChart.tsx` |

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
| viewState 为 null | `viewState` 初始为 `null`（`useDeckGl.tsx` 中 `useState<...|null>(null)`），由 `useEffect` 监听 `deck.initialViewState` 变化后才 `setViewState`。渲染侧 `viewState && <DeckGL>` 条件渲染，null 时不挂载 DeckGL，避免 deck.gl runtime assertion error | `DeckGlJsonChart.tsx` |
| HexagonLayers 首帧不渲染 | `useEffect` 延迟一帧 `setIsInitialized(true)`，`layers={isInitialized ? deck.layers : EMPTY_LAYERS}` | `DeckGlJsonChart.tsx` |
| 无 mapStyle 配置 | 按明/暗主题自动注入 Carto 公共底图（positron-gl-style / dark-matter-gl-style） | `useDeckGl.tsx` |
| Carto 底图未配 key | 自动注入 Streamlit 公用 cartoKey=`x7g2plm9yq8vfrc` | `useDeckGl.tsx` |
| 未定义 `views` 字段引发控制台警告 | `delete jsonCopy?.views` 主动移除 | `useDeckGl.tsx` |
| 高度无配置 fallback | initialViewState.height → `theme.sizes.defaultMapHeight` | `useDeckGl.tsx` |
| **后端**：data=None 或 df.empty | 序列化为 `EMPTY_MAP = {"initialViewState": {"latitude": 0, "longitude": 0, "pitch": 0, "zoom": 1}}`，**无 layers 字段**，仅渲染一个以 (0,0) 为中心、zoom=1 的空底图（无任何数据图层） | `lib/streamlit/elements/deck_gl_json_chart.py` EMPTY_MAP 定义 / `lib/streamlit/elements/map.py` `_DEFAULT_MAP = dict(EMPTY_MAP)` |
| **后端**：缺 lat/lon 列 | `StreamlitAPIException`，列出允许列名（`lat`, `latitude`, `LAT`, `LATITUDE`）与现有列名 | `lib/streamlit/elements/map.py` `_get_lat_or_lon_col_name()` |
| **后端**：lat/lon 列含 NaN/NaT/None | `StreamlitAPIException` 禁止空值 | `lib/streamlit/elements/map.py` `_get_lat_or_lon_col_name()` |
| **后端**：color 列颜色格式非法 | `StreamlitAPIException: Column 'X' does not appear to contain valid colors.` | `lib/streamlit/elements/map.py` `_convert_color_arg_or_column()` |
| **后端**：pandas 3.x 兼容性 | `_prepare_pydeck_for_json()` 把 Layer DataFrame 弱引用转为 `list[dict]`（pydeck vars() 访问 DataFrame 在 pd3 失效）。**注意：此修改是就地（in-place）修改 pydeck 对象** | `lib/streamlit/elements/deck_gl_json_chart.py` `_prepare_pydeck_for_json()` |
| **后端**：width/height 参数序列化 | 通过 `create_layout_config(width, height)` 写入 `LayoutConfig` proto，而不是写入 Deck.GL JSON spec。st.map 无 width/height 参数，st.pydeck_chart 默认 `width="stretch"`, `height=500` | `lib/streamlit/elements/deck_gl_json_chart.py` `pydeck_chart()` |

#### 尺寸参数序列化链路

```
用户代码: st.pydeck_chart(chart, width="stretch", height=500)
    │
    ▼
[Python] pydeck_chart()
    ├─ create_layout_config(width="stretch", height=500)
    │    └─ 写入 ForwardMsg.delta.metadata.layout_config
    │       (不是写入 element.json!)
    ├─ pydeck_obj.to_json() → Deck.GL JSON spec（不含 width/height）
    └─ _enqueue("deck_gl_json_chart", proto, layout_config=layout_config)
         │
         ▼
[前端] DeckGlJsonChart
    ├─ element.widthConfig (从 LayoutConfig 解析，不是从 element.json)
    ├─ shouldWidthStretch(widthConfig) → true/false
    └─ useStWidthHeight({
         element: {},
         shouldUseContainerWidth,
         container: { width: propsWidth, height: fullScreenHeight },
         heightFallback: viewState?.initialViewState?.height || defaultMapHeight
       })
         ├─► width = isFullScreen ? propsWidth : (shouldUseContainerWidth ? "100%" : undefined)
         └─► height = isFullScreen ? fullScreenHeight : heightFallback

注意：st.map() 函数签名中没有 width/height 参数，走默认 layout_config（width=stretch, height=500）。
Deck.GL JSON spec 本身不会包含 width/height 字段。
```

#### pandas 3.x 兼容性处理对对象复用的影响

**问题背景**：pandas 3.x 移除了 `__dict__` 属性访问方式，pydeck 的 `default_serialize` 通过 `vars(DataFrame)` 访问 DataFrame 属性会失败。

**处理方式**：`_prepare_pydeck_for_json(pydeck_obj)` 在 pandas >= 3.0.0 时自动调用：

```python
def _prepare_pydeck_for_json(pydeck_obj: Deck | None) -> None:
    # 遍历 pydeck_obj.layers 中的每个 layer:
    for layer in layers:
        data = getattr(layer, "data", None)
        # pydeck 对 DataFrame 用 weakref 包装，先解引用
        if isinstance(data, weakref.ref):
            data = data()
        # 如果是 DataFrame → 就地替换为 list[dict]
        if isinstance(data, pd.DataFrame):
            layer.data = data.to_dict(orient="records")  # ⚠️ in-place 修改!
```

**对对象复用的影响**：

| 场景 | 行为 |
|------|------|
| **单次 run 内单次调用**（常见） | 正常工作，无副作用 |
| **单次 run 内同一 Deck 对象多次调用** | 第二次调用时 `layer.data` 已经是 `list[dict]`，不是 DataFrame。如果用户后续代码中依赖 layer.data 是 DataFrame，会出错 |
| **跨 rerun 调用** | Streamlit rerun 会重新执行整个脚本，Deck 对象重新创建，无影响 |
| **用户手动缓存 Deck 对象**（如 `@st.cache_data` 返回 Deck） | 被修改过的 Deck 对象（data 是 list[dict]）会被缓存复用，后续运行时跳过 DataFrame→list 转换，但渲染结果一致 |

> 函数 docstring 明确指出："This function modifies the pydeck object in place. If the same Deck object is passed to multiple st.pydeck_chart calls within a single script run, subsequent calls will see the converted list[dict] data instead of DataFrames."

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
[ArrowVegaLiteChart.tsx] useLayoutEffect
    │
    └─► createView(containerRef, spec)     // async，返回 Promise<VegaView|null>
          │  // eslint-disable-next-line @typescript-eslint/no-floating-promises
          │  // ⚠️ 无 await、无 .catch()、无 try/catch，fire-and-forget
          │
          └─► [useVegaEmbed.ts] createView
                │
                ├─ if (containerRef.current === null) {
                │     throw new Error("Element missing.")   // 同步抛错
                │  }
                │
                ├─ try {
                │    await embed(container, spec, options)  // vegaEmbed
                │    // 以下步骤均在 try 内：
                │    // maybeConfigureSelections、getDataArrays、insert data、runAsync、resize
                │  } finally {
                │    setIsCreatingView(false)   // 仅确保 loading state 复位
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
ArrowVegaLiteChart useMemo
    ├─► new Quiver(elementProto.data)
    │     └─► parseArrowIpcBytes(arrowData.data)  // Quiver.ts
    │           └─► tableFromIPC(ipcBytes)         // arrowParseUtils.ts
    │                 ⚠️ 无 try/catch！
    │
    └─► wrapDatasets(elementProto.datasets)
          └─► new Quiver(dataset.data)

getDataArray(quiver)  // arrowUtils.ts
    ├─► if (numDataRows === 0) return []   空数据 → 静默
    └─► for (row, col) quiver.getCell(row, col)  越界 → 抛错
```

**返回空数组 `[]` 的场景（静默降级，不抛错）**：

| 触发条件 | 位置 | 行为 |
|----------|------|------|
| Quiver 行数为 0（`numDataRows === 0`） | `arrowUtils.ts` `getDataArray()` | 返回 `[]`，Vega 渲染空图表 |
| `quiverData === null`（proto.data 为空） | `arrowUtils.ts` `getInlineData()` | 返回 null，不调用 `view.insert()` |
| updateData 中新数据为空（0 行） | `useVegaEmbed.ts` `updateData()` | 调用 `view.remove(name, truthy)`，dataset 已删除的异常被吞掉（catch 空） |
| datasets 为空数组 | `arrowUtils.ts` `getDataSets()` | 返回 null，不处理 |

**直接抛错的场景（向上抛出，可导致组件崩溃）**：

| 触发条件 | 错误信息 | 位置 | 抛出时机 |
|----------|---------|------|----------|
| 输入 Arrow IPC bytes 损坏或格式非法 | apache-arrow 内部抛错（如 "Invalid IPC stream"） | `arrowParseUtils.ts` `tableFromIPC()` | `new Quiver()` 同步抛，React 渲染阶段 |
| Pandas schema 中 index column 在 Arrow schema 中找不到 | `Index field ${indexCol} not found in arrow schema` | `arrowParseUtils.ts` `parseArrowIpcBytes()` | `new Quiver()` 同步抛，React 渲染阶段 |
| Quiver.getCell 行索引越界 | `Row index is out of range: ${rowIndex}` | `Quiver.ts` `getCell()` | `getDataArray()` 遍历抛，React 渲染阶段 |
| Quiver.getCell 列索引越界 | `Column index is out of range: ${columnIndex}` | `Quiver.ts` `getCell()` | `getDataArray()` 遍历抛，React 渲染阶段 |
| Arrow 数据类型不兼容（如 Union 类型） | apache-arrow `tableFromIPC()` 抛错 | `arrowParseUtils.ts` | `new Quiver()` 同步抛 |

**关键边界**：
1. **同步抛错都会被 React ErrorBoundary 捕获**（因为发生在渲染阶段的 useMemo 中）
2. **0 行数据永远静默降级**，不会抛错
3. `tableFromIPC(ipcBytes)` **无任何 try/catch**，Arrow bytes 损坏直接崩溃
4. `view.remove(name, truthy)` 有 **空 catch 块**（`useVegaEmbed.ts`），dataset 不存在时静默，不抛错
5. `isFacetChart()` 和 `hasNestedComposition()` 都有 **try/catch 返回 false**（`ArrowVegaLiteChart.tsx`），JSON 解析失败不阻断渲染

### 6.6 PyDeck / Map 的异常兜底

**后端兜底**：`lib/streamlit/elements/deck_gl_json_chart.py` / `lib/streamlit/elements/map.py`

| 场景 | 处理方式 | 位置 |
|------|---------|------|
| `st.pydeck_chart(None)` | `json.dumps(EMPTY_MAP)`，`EMPTY_MAP` **仅含 initialViewState、无 layers 字段**，前端渲染一个以 (0,0) 为中心、zoom=1 的空底图（只有 Carto 底图瓦片，无数据图层） | `deck_gl_json_chart.py` EMPTY_MAP 定义 / `marshall()` 内 `pydeck_obj is None` 分支 |
| `st.map(None)` / `st.map(empty_df)` | 复用 `_DEFAULT_MAP = dict(EMPTY_MAP)`，同样无 layers，渲染空底图 | `map.py` `to_deckgl_json()` 中 `data is None` / `data.empty` 分支 |
| `st.map(df)` 有数据时 | 深拷贝 `_DEFAULT_MAP` 后**覆盖 initialViewState 为自动计算的居中坐标和缩放级别**，并添加 `layers` 数组含 ScatterplotLayer，数据通过 `df.to_dict("records")` 内嵌 | `map.py` `to_deckgl_json()` 中 `_get_viewport_details()` 自动居中 + `_get_zoom_level()` 自动缩放 |
| PyDeck selection_mode 非法值 | `StreamlitAPIException` 列出合法选项 `{"single-object", "multi-object"}` | `deck_gl_json_chart.py` `parse_selection_mode()` |
| PyDeck selection_mode 传集合（多值） | `StreamlitAPIException` 不支持多值 | `deck_gl_json_chart.py` `parse_selection_mode()` |
| PyDeck on_select 非 `ignore/rerun/callable` | `StreamlitAPIException` | `deck_gl_json_chart.py` `pydeck_chart()` |
| pandas 3.x + pydeck DataFrame 序列化失败 | `_prepare_pydeck_for_json()` 将 DataFrame 弱引用转为 `list[dict]` | `deck_gl_json_chart.py` `_prepare_pydeck_for_json()` |
| `st.map` 找不到经纬度列 | `StreamlitAPIException` 列出候选列名与现有列名 | `map.py` `_get_lat_or_lon_col_name()` |
| `st.map` 经纬度列含空值 | `StreamlitAPIException` 禁止 NaN/NaT/None | `map.py` `_get_lat_or_lon_col_name()` |
| `st.map` color 列值非法 | `StreamlitAPIException` 提示颜色格式不合法 | `map.py` `_convert_color_arg_or_column()` |
| Mapbox token 缺失 | 优先取 pydeck_obj.mapbox_key，fallback 到 `config.get_option("mapbox.token")` | `deck_gl_json_chart.py` `marshall()` |

**前端兜底**：`frontend/lib/src/components/elements/DeckGlJsonChart/useDeckGl.tsx`

| 场景 | 处理方式 | 位置 |
|------|---------|------|
| Deck.GL JSON 解析失败（非法 JSON5） | `JSON5.parse()` 抛错 → useMemo 抛错 → 祖先 ErrorBoundary 捕获 | `useDeckGl.tsx` `parsedPydeckJson` useMemo |
| 图层 id 缺失导致选择状态不可追踪 | `hasUnknownLayerId` 标记，所有旧选择被丢弃但不崩溃 | `useDeckGl.tsx` `getLayerDataInfo()` |
| 数据长度缩水、选择索引越界 | `filterValidIndicesForLayer()` 过滤越界索引，重新拉取当前数据对象 | `useDeckGl.tsx` `filterValidIndicesForLayer()` |
| 图层被移除后仍保留旧选择 | `sanitizeSelection()` 检测 layerId 不存在则丢弃（全部无 id 时例外保留） | `useDeckGl.tsx` `sanitizeSelection()` |
| 视图状态 viewState 未初始化 | `viewState` 初始值为 `null`（`useDeckGl.tsx` `useState(null)`），通过 `useEffect` 监听 `deck.initialViewState` 变化时才 `setViewState`。渲染侧 `viewState && <DeckGL>` 条件渲染，null 时不挂载 DeckGL 避免其内部 assertion error | `DeckGlJsonChart.tsx` |
| HexagonLayers 首帧不渲染 bug | `useEffect` 延迟一帧 `setIsInitialized(true)`，layers 延迟注入 | `DeckGlJsonChart.tsx` |
| WebGL 上下文不足 | 浏览器自动回收最早的 WebGL 上下文，Streamlit 仅文档提示 ≤8 张图 | `lib/streamlit/elements/deck_gl_json_chart.py` docstring |
| tooltip 模板变量不存在（`{col_not_exist}`） | `interpolate()` 未匹配则保留原模板字符串，不抛错 | `useDeckGl.tsx` `interpolate()` |
| 未定义 views 字段引发控制台警告 | `delete jsonCopy?.views` 主动移除 | `useDeckGl.tsx` `deck` useMemo |
| **尺寸参数** | width/height 不写入 Deck.GL JSON，通过 `create_layout_config()` 写入 LayoutConfig proto。st.map 无 width/height 参数，默认 `width="stretch"`, `height=500` | `lib/streamlit/elements/deck_gl_json_chart.py` `pydeck_chart()` |
| **pandas 3.x 就地修改** | `_prepare_pydeck_for_json()` 就地修改 pydeck 对象，layer.data 从 DataFrame 转为 `list[dict]`。同一 Deck 对象在一次 run 内被多次调用时，第二次调用看到的是转换后的数据 | `lib/streamlit/elements/deck_gl_json_chart.py` `_prepare_pydeck_for_json()` |

**Serde 兜底**：`lib/streamlit/elements/deck_gl_json_chart.py` `PydeckSelectionSerde`

```python
def deserialize(self, ui_value: str | None) -> PydeckState:
    empty_state = {"selection": {"indices": {}, "objects": {}}}
    # ui_value is None → return empty_state
    # ui_value is "{}" (empty dict) → "selection" not in dict → return empty_state
    # ui_value is valid JSON → return parsed result
```

接收到空 dict `{}` 或缺失 `selection` 键时，返回 `EMPTY_STATE` 而非抛错。

---

### 6.7 PyDeck/Map 配置变更时的状态管理机制

#### 6.7.1 选择状态的延续与清理（sanitizeSelection 触发时机）

选择状态清理**不只是组件初始化时执行一次**，而是在**每次 spec 配置变更时**都会触发。

完整触发链路：

```
[Python] st.pydeck_chart(..., on_select="rerun")
    │  配置变更（图层增删、数据变化、初始视口变化、颜色变化等）
    ▼
[WebSocket] ForwardMsg 更新 element.json
    │
    ▼
[React] ArrowVegaLiteChart useMemo 解析新 parsedPydeckJson
    │
    ├─► useExecuteWhenChanged(parsedPydeckJson)   监听配置变更
    │     │
    │     ├─► 非选择模式或无 layers → 跳过
    │     ├─► getLayerDataInfo(parsedPydeckJson.layers)  重建图层数据索引
    │     └─► sanitizeSelection(data.selection, layerDataInfo)
    │           │
    │           ├─► 清理规则（见 6.6 节 sanitizeSelection 清理规则速查表）
    │           │    ├─► 图层删除 → 丢弃该图层选择
    │           │    ├─► 数据缩水 → 过滤越界索引，重新拉取当前对象
    │           │    ├─► URL/GeoJSON/无 ID 图层 → 保留（无法验证）
    │           │    └─► 数据未变 → 不变
    │           │
    │           └─► changed === true → setSelection({
    │                       fromUi: false,    // ⚠️ 关键：非用户操作，不触发 rerun
    │                       value: { selection: sanitized }
    │                    })
    │                       只同步到 WidgetStateManager，不触发 Python 回调
    │
    └─► deck useMemo 依赖 data.selection.indices，重新构造 DeckObject
          ├─► anyLayersHaveSelection 重新计算（避免全局 dimming）
          └─► 图层 fillFunction / updateTriggers 重建
```

**关键细节**：
- `setSelection({ fromUi: false })` 的 `fromUi=false` 标记表示这是**程序自动清理**，不是用户交互，**不会触发 rerun 或 on_select 回调**，仅同步前端 widget 状态
- `isInitialized` 状态仅用于延迟图层注入（修复 HexagonLayers 首帧不渲染 bug），**与选择状态清理无关**，`isInitialized` 变化不会触发 sanitizeSelection
- 视口配置（initialViewState）变更走**单独的 diff 更新链路**（见下一小节），不经过 sanitizeSelection

**典型触发场景**：

| 操作 | 是否触发 sanitize | changed |
|------|------------------|---------|
| 数据行数从 100 缩到 50 | 是 | true（越界索引被丢弃） |
| 新增大图层但未改动现有图层数据 | 是 | false（现有选择全部保留） |
| 删除一个已选图层 | 是 | true（该图层选择被丢弃） |
| 修改图层颜色/大小但不改数据 | 是 | false（选择索引全部保留） |
| 数据从 DataFrame 转为 list[dict]（内容完全相同） | 是 | 索引未越界 → false |
| 用户手动平移/缩放地图（改 viewState） | 否（不改变 parsedPydeckJson） | - |
| initialViewState 从后端变了 | 否（走单独的 diff 链路） | - |

---

#### 6.7.2 initialViewState 配置变更的 diff 更新机制

视口配置变更（如后端重新计算了 zoom/center）走**独立的 useEffect 链路**，不会触发选择状态清理，且只做增量 diff 更新：

```
useEffect(deck.initialViewState)
    │
    ├─► isEqual(deck.initialViewState, initialViewStateRef.current)  深比较
    │     └─► 无变化 → 直接返回
    │
    ├─► 计算 diff：遍历所有 initialViewState 键
    │     ├─► 键值与 old 相同 → 跳过
    │     └─► 键值不同 → 加入 diff 对象
    │
    ├─► setViewState(existing => ({ ...existing, ...diff }))
    │     ⚠️ 只覆盖变化的字段，保留用户手动平移/缩放的其他状态
    │
    └─► initialViewStateRef.current = deck.initialViewState  更新引用
```

**关键特性**：
- **保留用户手动交互状态**：例如后端只改了 `zoom`，用户手动平移的 `latitude/longitude` 会被保留
- **避免无限循环**：用户手动拖动地图会触发 `onViewStateChange` 更新 `viewState`，但 `initialViewStateRef` 记录的是后端上次下发的值，用户拖动不会触发此 useEffect
- **diff 只针对 initialViewState 自身**：latitude/longitude/zoom/pitch/bearing/height 字段独立比较

**示例**：
```
用户状态: {latitude: 39.9, longitude: 116.4, zoom: 10}  (用户手动平移到北京)
后端变更: initialViewState = {latitude: 0, longitude: 0, zoom: 5}
diff 计算: 三个字段都变了 → 全部覆盖
最终: {latitude: 0, longitude: 0, zoom: 5}  (用户手动状态被覆盖)

用户状态: {latitude: 39.9, longitude: 116.4, zoom: 10}
后端变更: initialViewState = {latitude: 0, longitude: 0, zoom: 10}  (只改经纬度)
diff 计算: latitude/longitude 变了，zoom 没变
最终: {latitude: 0, longitude: 0, zoom: 10}  (zoom 保留用户值? 不，因为 latitude/longitude 变了，但...
       ⚠️ 实际逻辑是 diff 只包含变化的字段，但 setViewState 时 ...existing 会保留用户状态
       最终：{latitude: 0, longitude: 0, zoom: 10})  ✅ zoom 保留
```

---

### 6.8 地图尺寸参数的序列化与前端应用

#### 6.8.1 后端序列化流程

`st.pydeck_chart(pydeck_obj, width="stretch", height=500)` 的尺寸参数**不写入 Deck.GL JSON spec**，而是走独立的 LayoutConfig 通道：

```
Python 调用: st.pydeck_chart(..., width="stretch", height=500, use_container_width=True)
    │
    ├─► use_container_width 处理（已废弃）
    │     ├─► show_deprecation_warning
    │     └─► use_container_width=True → width="stretch"
    │
    ├─► create_layout_config(width=width, height=height)
    │     └─► 构造 LayoutConfig proto，写入 ForwardMsg.delta.metadata.layout_config
    │
    └─► _get_pydeck_width(pydeck_obj)  // 尝试从 pydeck 对象读取 width
          └─► 返回值被丢弃，最终以 LayoutConfig 为准
```

**序列化结果**：
- `width` / `height` → 写入 `ForwardMsg.delta.metadata.layout_config.width/height`
- Deck.GL JSON spec 中**不含** width/height 字段（pydeck 对象自身的 width 属性也会被忽略）
- `use_container_width` 不会写入 proto，仅在 Python 端转换为 `width="stretch"` 后丢弃

#### 6.8.2 前端尺寸计算（useStWidthHeight）

前端通过 `useStWidthHeight` Hook 计算最终尺寸，完整优先级：

```
useStWidthHeight({
  element: {},                              // 无 element 级别的 width/height 配置
  container: { height, width },             // 来自 ElementFullscreenContext
  isFullScreen,
  shouldUseContainerWidth: shouldWidthStretch(widthConfig),
  heightFallback: initialViewState.height || theme.sizes.defaultMapHeight
})
    │
    ├─► width 计算:
    │     if (shouldUseContainerWidth || isFullScreen) → "100%"
    │     else → element.width || container.width || widthFallback ("auto")
    │
    └─► height 计算:
          if (isFullScreen && container.height) → container.height  (全屏优先)
          else → element.height || container.height || heightFallback
                  heightFallback 优先级:
                    initialViewState.height（来自 Deck.GL JSON spec）
                    → theme.sizes.defaultMapHeight（500px）
```

**完整优先级速查**：

| 参数 | 优先级（高→低） |
|------|----------------|
| **width** | 1. `isFullScreen=true` → "100%"<br>2. `width="stretch"` → "100%"<br>3. element.width（PyDeck 无）<br>4. container.width（父容器宽度）<br>5. widthFallback "auto" |
| **height** | 1. `isFullScreen=true` + container.height 存在 → container.height<br>2. element.height（PyDeck 无）<br>3. container.height（父容器高度）<br>4. initialViewState.height（Deck spec 内嵌，如 EMPTY_MAP 无 height）<br>5. theme.sizes.defaultMapHeight（500px） |

**典型场景**：
```
st.pydeck_chart(deck, height=600)  # 传入 height
  → LayoutConfig.height = 600
  → 前端 container.height = 600
  → 最终 height = 600  ✅

st.pydeck_chart(None)  # 空数据走 EMPTY_MAP，无 height 配置
  → LayoutConfig 为默认（height 由 width/height 参数默认值 500）
  → 前端 container.height = 500
  → 最终 height = 500  ✅  不是 EMPTY_MAP 里的值（EMPTY_MAP 没有 height）

st.pydeck_chart(deck, height="stretch")
  → LayoutConfig.height = "stretch"
  → shouldHeightStretch = true
  → heightFallback = initialViewState.height 可能不存在
  → 最终 height = 父容器高度（由 Flex 布局决定）
```

---

### 6.9 pandas 3.x 兼容性处理对对象复用的影响

#### 6.9.1 问题背景

pandas 3.0 起 `DataFrame` 不再支持通过 `vars(df)` 访问 `__dict__` 属性，而 pydeck 的 `default_serialize()` 依赖 `vars()` 遍历对象属性，导致序列化失败。

#### 6.9.2 修复方案：`_prepare_pydeck_for_json()`

```python
def _prepare_pydeck_for_json(pydeck_obj: Deck | None) -> None:
    if pydeck_obj is None:
        return

    layers = getattr(pydeck_obj, "layers", None)
    if layers is None:
        return

    for layer in layers:
        data = getattr(layer, "data", None)
        if data is None:
            continue

        # pydeck 将 DataFrame 包装在 weakref 中以避免循环引用
        if isinstance(data, weakref.ref):
            data = data()
            if data is None:
                continue

        # 就地替换 DataFrame 为 list[dict]
        if isinstance(data, pd.DataFrame):
            layer.data = data.to_dict(orient="records")
```

**关键特性**：
- **就地修改**（in-place）：直接修改 `layer.data` 属性，不返回新对象
- **weakref 处理**：pydeck 内部用 `weakref.ref(DataFrame)` 包装数据以避免循环引用，需要先解引用
- **只转换 DataFrame**：其他数据类型（list/GeoJSON/URL 字符串）不处理

#### 6.9.3 对对象复用的影响

**核心问题**：函数是**原地修改** pydeck_obj，对同一个 Deck 对象的多次调用会有副作用。

**场景 1：单次运行内复用同一 Deck 对象（问题场景）**

```python
# 用户代码
deck = pdk.Deck(layers=[pdk.Layer("ScatterplotLayer", data=df)])
st.pydeck_chart(deck)  # 第一次调用
# _prepare_pydeck_for_json 将 deck.layers[0].data 从 DataFrame 转为 list[dict]
st.pydeck_chart(deck)  # 第二次调用，同一个 deck 对象
# layer.data 已经是 list[dict]，isinstance(data, pd.DataFrame) 为 False
# 跳过转换，但序列化结果一致（list[dict] 同样可被 json.dumps）
```

**影响**：
- ✅ 功能正常：两次调用都能正确序列化
- ⚠️ 副作用：第二次调用时 `layer.data` 已不是 DataFrame，如果用户后续代码需要访问原始 DataFrame（如 `deck.layers[0].data.sum()`）会失败
- ⚠️ 性能：第一次转换有开销，后续调用无额外开销

**场景 2：跨 rerun 复用 Deck 对象（不常见，但可能）**

```python
@st.cache_data
def create_deck():
    # 用户手动缓存 Deck 对象（不推荐，推荐缓存原始数据）
    return pdk.Deck(layers=[pdk.Layer("ScatterplotLayer", data=df)])

deck = create_deck()
st.pydeck_chart(deck)
```

**影响**：
- ✅ 功能正常：第一次 rerun 时 DataFrame 被转为 list[dict]，后续 rerun 看到 list[dict] 也能正常序列化
- ⚠️ 隐式类型变化：缓存的 deck 对象被永久修改，数据类型从 DataFrame 变为 list[dict]
- ⚠️ 缓存膨胀：如果用户在 `@st.cache_data` 中缓存 Deck 对象，缓存值会被修改（但不影响缓存本身，因为 Streamlit 缓存是 immutable 写入后只读）

**场景 3：每次 rerun 重新创建 Deck 对象（Streamlit 推荐模式）**

```python
# 用户代码（标准写法）
deck = pdk.Deck(layers=[pdk.Layer("ScatterplotLayer", data=df)])
st.pydeck_chart(deck)
```

**影响**：
- ✅ 无副作用：每次 rerun 都是新的 Deck 对象，转换不影响其他代码
- ✅ 推荐模式：符合 Streamlit rerun 模型的最佳实践

**docstring 已明确的警告**：
> "This function modifies the pydeck object in place. If the same Deck object is passed to multiple st.pydeck_chart calls within a single script run, subsequent calls will see the converted list[dict] data instead of DataFrames. In Streamlit's rerun-based execution model, this is typically not an issue since Deck objects are usually recreated on each run."

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

## 8. 完整端到端示例：st.map(df)

```
用户代码: st.map(df, latitude="lat", longitude="lon", size=100, color="#ff0000")
    │
    ▼
[Python] MapMixin.map()
    │
    ├─ to_deckgl_json(df, lat="lat", lon="lon", size=100, color="#ff0000", zoom=None)
    │    ├─ data is None? No → 不走 EMPTY_MAP
    │    ├─ convert_anything_to_pandas_df(df)
    │    ├─ _get_lat_or_lon_col_name(df, "latitude", "lat", {"lat","latitude","LAT","LATITUDE"})
    │    ├─ _get_lat_or_lon_col_name(df, "longitude", "lon", {"lon","longitude","LON","LONGITUDE"})
    │    ├─ _get_value_and_col_name(df, 100, _DEFAULT_SIZE=100) → ("100", None)
    │    ├─ _convert_color_arg_or_column(df, "#ff0000", None) → (255, 0, 0, 255)
    │    ├─ _get_viewport_details(df, "lat", "lon", None)
    │    │    ├─ center_lat = (max_lat + min_lat) / 2.0
    │    │    ├─ center_lon = (max_lon + min_lon) / 2.0
    │    │    └─ zoom = _get_zoom_level(max(range_lat, range_lon))  查 _ZOOM_LEVELS 表
    │    │
    │    └─ 构建 JSON:
    │         default = copy.deepcopy(_DEFAULT_MAP)
    │         default["initialViewState"]["latitude"] = center_lat    ← 覆盖默认 0
    │         default["initialViewState"]["longitude"] = center_lon   ← 覆盖默认 0
    │         default["initialViewState"]["zoom"] = zoom              ← 覆盖默认 1
    │         default["layers"] = [{
    │             "@@type": "ScatterplotLayer",
    │             "getPosition": "@@=[lon, lat]",
    │             "getRadius": "100",
    │             "radiusMinPixels": 3,
    │             "radiusUnits": "meters",
    │             "getFillColor": [255, 0, 0, 255],
    │             "data": df.to_dict("records")    ← 内嵌 JSON
    │         }]
    │
    └─ marshall(map_proto, deck_gl_json) → _enqueue("deck_gl_json_chart", ...)
         │
         ▼
[WebSocket] ForwardMsg { delta: { new_element: { deck_gl_json_chart: { json: "..." } } } }
         │
         ▼
[React] DeckGlJsonChart
    ├─ useDeckGl: JSON5.parse(element.json) → ParsedDeckGlConfig
    │    ├─ 无 mapStyle → 注入 Carto 底图 (positron/dark-matter)
    │    ├─ Carto 底图 → 注入 cartoKey="x7g2plm9yq8vfrc"
    │    ├─ jsonConverter.convert() → DeckObject (layers 解析为 Deck 层实例)
    │    └─ useEffect: deck.initialViewState 变化 → setViewState
    │
    ├─ viewState !== null → <DeckGL viewState={...} layers={deck.layers}>
    │    └─ <StaticMap mapStyle={deck.mapStyle} mapboxApiAccessToken={...} />
    │
    └─ 渲染: Carto 底图 + ScatterplotLayer 散点
```

### 对比：st.map(None) 空数据链路

```
用户代码: st.map(None)
    │
    ▼
[Python] to_deckgl_json(data=None, ...)
    └─ data is None → return json.dumps(_DEFAULT_MAP)
         _DEFAULT_MAP = {"initialViewState": {"latitude": 0, "longitude": 0, "pitch": 0, "zoom": 1}}
         ⚠️ 无 layers 字段！
    │
    ▼
[React] DeckGlJsonChart
    ├─ useDeckGl: JSON5.parse → { initialViewState: {...} }，无 layers
    │    ├─ 无 mapStyle → 注入 Carto 底图
    │    └─ jsonConverter.convert() → deck.layers 为空数组 []
    │
    └─ 渲染: Carto 底图瓦片，以 (0,0) 为中心 zoom=1 → 非洲外海
         无任何数据图层、无散点
```

---

## 9. 扩展图表类型的接入范式

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
