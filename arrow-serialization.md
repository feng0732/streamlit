# DataFrame/Arrow 序列化路径深度解析

本文档梳理 Streamlit 中 DataFrame/Arrow 从后端 Python 序列化到前端还原渲染的完整链路，涵盖表格转换、协议封装、传输边界和前端还原四个核心阶段。

---

## 一、整体架构概览

```
Python 用户数据
      │
      ▼
┌───────────────────────────┐
│  1. 数据格式识别与归一化   │  dataframe_util.py
│  (40+ 种输入类型统一)      │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  2. Pandas DataFrame →     │  convert_pandas_df_to_arrow_bytes()
│     PyArrow Table          │  fix_arrow_incompatible_column_types()
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  3. Arrow IPC Stream       │  RecordBatchStreamWriter
│     字节序列化              │  _maybe_truncate_table()
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  4. Protobuf 协议封装      │  ArrowData.proto
│  (ArrowData / Dataframe /  │  Dataframe.proto
│   Components / BidiComp)   │  Components.proto
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  5. 传输边界（WebSocket）  │  WebsocketConnection.tsx
│  消息接收 → 去重匹配       │  ForwardMessageCache.ts
│  → 引用还原 → 消息分发      │  App.tsx handleMessage()
│  → Delta 应用              │  AppRoot.ts applyDelta()
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  6. 前端 Arrow 还原        │  parseArrowIpcBytes()
│  IPC bytes → Arrow Table   │  tableFromIPC()
│  + Pandas Schema 解析      │  Quiver.ts
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  7. UI 渲染                │  DataFrame.tsx (glide-data-grid)
│  交互式表格 / 静态表格      │  Table.tsx (HTML Table)
└───────────────────────────┘
```

---

## 二、后端阶段：表格转换（Python → Arrow Bytes）

### 2.1 入口层：ArrowMixin.dataframe()

核心入口位于 [arrow.py](./lib/streamlit/elements/arrow.py#L476-L1105) 的 `ArrowMixin.dataframe()` 方法。

**两条分支路径：**

| 数据类型 | 处理方式 | 代码位置 |
|---------|---------|---------|
| `pa.Table` (PyArrow原生) | 直接序列化，跳过 Pandas | [arrow.py#L948-L954](./lib/streamlit/elements/arrow.py#L948-L954) |
| 其他所有类型 | → Pandas DataFrame → Arrow | [arrow.py#L955-L981](./lib/streamlit/elements/arrow.py#L955-L981) |

```python
# arrow.py L948-L981 核心逻辑
if isinstance(data, pa.Table):
    # 快速路径：PyArrow Table 直接序列化
    proto.arrow_data.data = dataframe_util.convert_arrow_table_to_arrow_bytes(data)
else:
    # 通用路径：先转 Pandas DataFrame
    data_format = dataframe_util.determine_data_format(data)
    if dataframe_util.is_pandas_styler(data):
        marshall_styler(proto.arrow_data, data, default_uuid)  # Styler 特殊处理
    data_df = dataframe_util.convert_anything_to_pandas_df(data, ensure_copy=False)
    proto.arrow_data.data = dataframe_util.convert_pandas_df_to_arrow_bytes(data_df)
```

### 2.2 数据格式识别：determine_data_format()

[dataframe_util.py](./lib/streamlit/dataframe_util.py#L1364-L1453) 中定义了 **40+ 种** 数据格式的枚举 `DataFormat`，包括：

- **Pandas 生态**: `PANDAS_DATAFRAME`, `PANDAS_SERIES`, `PANDAS_INDEX`, `PANDAS_STYLER`, `PANDAS_ARRAY`
- **PyArrow 生态**: `PYARROW_TABLE`, `PYARROW_ARRAY`
- **分布式/大对象**: `DASK_OBJECT`, `MODIN_OBJECT`, `SNOWPARK_OBJECT`, `SNOWPANDAS_OBJECT`, `PYSPARK_OBJECT`
- **Polars 生态**: `POLARS_DATAFRAME`, `POLARS_LAZYFRAME`, `POLARS_SERIES`
- **数据库相关**: `DBAPI_CURSOR`, `DUCKDB_RELATION`
- **集合类型**: `LIST_OF_RECORDS`, `LIST_OF_ROWS`, `KEY_VALUE_DICT`, `TUPLE_OF_VALUES`, `SET_OF_VALUES`
- **其他**: `NUMPY_MATRIX`, `XARRAY_DATASET`, `CUSTOM_DICT` 等

### 2.3 数据归一化：convert_anything_to_pandas_df()

[dataframe_util.py#L559-L814](./lib/streamlit/dataframe_util.py#L559-L814) 是核心归一化函数，按优先级顺序尝试：

**第一梯队：直接兼容（零拷贝或轻量转换）**
```python
if isinstance(data, pd.DataFrame):          return data  # 直接返回
if isinstance(data, (pd.Series, pd.Index)): return pd.DataFrame(data)
if is_pandas_styler(data):                  return data.data
if isinstance(data, np.ndarray):            return pd.DataFrame(data)
```

**第二梯队：有 .to_pandas() / .to_arrow() 方法的库**
```python
# Polars
if is_polars_dataframe(data):     return data.to_pandas()
if is_polars_series(data):        return data.to_pandas().to_frame()
# Xarray
if is_xarray_dataset(data):       return data.to_dataframe()
# Dask / Modin / Snowpark / PySpark
# ... 各自调用 limit(max_unevaluated_rows).to_pandas() / head()
```

**第三梯队：协议层兼容**
```python
# Arrow PyCapsule Interface (__arrow_c_stream__) - 优先于 DataFrame Interchange
if has_callable_attr(data, "__arrow_c_stream__"):
    table = pa.RecordBatchReader.from_stream(data).read_all()
    return table.to_pandas()

# DataFrame Interchange Protocol (__dataframe__) - 已标记 deprecated
if has_callable_attr(data, "__dataframe__"):
    return pd.api.interchange.from_dataframe(data)
```

**第四梯队：Python 原生集合兜底**
```python
# generator / Enum / deque / UserList / CustomDict / NamedTuple / Dataclass
# → dict-like → list-like → pd.DataFrame() 构造器兜底
```

**未评估数据行数限制：** 对于 LazyFrame、Dask、Snowpark 等 out-of-core 对象，默认只取前 `_MAX_UNEVALUATED_DF_ROWS = 10000` 行，并提示用户。

### 2.4 Pandas → Arrow 转换：convert_pandas_df_to_arrow_bytes()

[dataframe_util.py#L934-L975](./lib/streamlit/dataframe_util.py#L934-L975)：

```python
def convert_pandas_df_to_arrow_bytes(df, *, downcast_large_types=False):
    # 步骤1: Pandas DataFrame → PyArrow Table
    try:
        table = pa.Table.from_pandas(df)
    except (ArrowTypeError, ArrowInvalid, ArrowNotImplementedError):
        # 自动修复不兼容的列类型
        df = fix_arrow_incompatible_column_types(df)
        table = pa.Table.from_pandas(df)

    if downcast_large_types:
        table = _downcast_large_arrow_types(table)  # Pandas 3.x StringDtype 适配

    # 步骤2: PyArrow Table → Arrow IPC bytes
    return convert_arrow_table_to_arrow_bytes(table)
```

#### 类型自动修复：fix_arrow_incompatible_column_types()

[dataframe_util.py#L1302-L1361](./lib/streamlit/dataframe_util.py#L1302-L1361) 处理以下 Arrow 不兼容类型：

| 不兼容类型 | 修复策略 | 示例 |
|-----------|---------|------|
| 复数类型 (complex64/128/256) | → 字符串 | `1+2j` → `"(1+2j)"` |
| Period[B/N/ns/U/us] | → 字符串 | 前端未实现这些 period 类型 |
| geometry (GeoPandas) | → 字符串 | WKB/WKT 序列化 |
| object dtype + 混合类型 (mixed-integer/complex) | → 字符串 | |
| object dtype + dict-like | → 字符串 | Arrow JS 不支持 struct in object |
| object dtype + frozenset/ExtensionArray | → list | PyArrow 无法直接序列化 |
| object dtype + 其他 list-like | 保持不变 | |
| 混合类型的 Index | → 字符串 | |

#### Large 类型降级：_downcast_large_arrow_types()

[dataframe_util.py#L817-L892](./lib/streamlit/dataframe_util.py#L817-L892)：

**问题背景**：Pandas 3.x 默认将 StringDtype 存储为 Arrow 的 `large_string` (LargeUtf8)，而第三方自定义组件 v1 捆绑了较旧版本的 Arrow JS，无法解码 LargeUtf8/LargeBinary/LargeList 的 type code。

**降级映射**：
```
large_string  →  string       (Utf8)
large_binary  →  binary
large_list<T> →  list<T>      (递归嵌套降级)
```

### 2.5 Polars 快速路径：_convert_polars_to_arrow_bytes()

[dataframe_util.py#L1010-L1016](./lib/streamlit/dataframe_util.py#L1010-L1016)：

```python
def _convert_polars_to_arrow_bytes(data):
    table = data.to_arrow()          # Polars → PyArrow Table（零拷贝）
    return convert_arrow_table_to_arrow_bytes(table)
```

**性能优势**：跳过 Pandas 中转，性能提升 **100-400x**（官方注释）。适用于 Polars DataFrame / Series / LazyFrame。

### 2.6 Arrow Table → IPC Bytes：convert_arrow_table_to_arrow_bytes()

[dataframe_util.py#L894-L931](./lib/streamlit/dataframe_util.py#L894-L931)：

```python
def convert_arrow_table_to_arrow_bytes(table):
    # 步骤1: 可选的自动截断（超消息大小限制时）
    table = _maybe_truncate_table(table)

    # 步骤2: large_list → list 降级（Arrow JS 不支持 large_list）
    if _has_large_list_type(table.schema):
        table = table.cast(_downcast_large_list_schema(table.schema))

    # 步骤3: Arrow IPC Stream 格式序列化
    sink = pa.BufferOutputStream()
    writer = pa.RecordBatchStreamWriter(sink, table.schema)
    writer.write_table(table)
    writer.close()
    return sink.getvalue().to_pybytes()
```

**IPC 格式选择**：使用 `RecordBatchStreamWriter` (Stream Format) 而非 `RecordBatchFileWriter` (File Format)。Stream 格式只包含一个 schema + 若干 record batch，无 footer，适合流式传输；File 格式有 magic number 和 footer，适合文件存储。

### 2.7 自动截断机制：_maybe_truncate_table()

[dataframe_util.py#L1153-L1225](./lib/streamlit/dataframe_util.py#L1153-L1225)：

**触发条件**：`server.enableArrowTruncation = True`（实验性功能）

**算法逻辑**：
```
1. 读取 server.maxMessageSize (MB) → 转 bytes
2. 预估 table 大小 = table.nbytes + 1MB (protobuf 开销)
3. 如果超限制：
   targeted_rows = ceil(table_rows * (max_message_size / table_size))
   再额外多减 5% 预估误差 / 至少减 1% / 至少减 5 行
   → slice(0, targeted_rows)
   → 递归调用直到大小合规
4. 提示用户: "Showing X out of Y rows due to data size limitations."
```

### 2.8 Pandas Styler 处理：marshall_styler()

[arrow.py#L962-L968](./lib/streamlit/elements/arrow.py#L962-L968) 调用 [pandas_styler_utils.py](./lib/streamlit/elements/lib/pandas_styler_utils.py#L33-L71)：

```
Styler 对象处理流程：
  ├─ 1. 设置 UUID (用户未提供时用 delta_path hash)
  ├─ 2. 调用 styler._compute() 触发样式计算
  ├─ 3. styler._translate() 获取内部样式数据
  ├─ 4. 提取 caption (表格标题)
  ├─ 5. 提取 CSS styles 字符串
  └─ 6. display_values → 另一个 Arrow IPC bytes（格式化后的显示值表格）
```

**display_values 的意义**：原始数据用于排序/计算，display_values 用于单元格文本显示。两者维度必须相同。

---

## 三、协议封装层：Protobuf 消息结构

### 3.1 基础数据容器：ArrowData.proto

[ArrowData.proto](./proto/streamlit/proto/ArrowData.proto) 是所有 Arrow 数据的原子容器：

```protobuf
message ArrowData {
  bytes data = 1;                    // Arrow IPC Stream Format 字节
  PandasStyler styler = 2;           // 可选的 Pandas Styler 元数据

  message PandasStyler {
    string uuid = 1;                 // Styler 唯一标识 → CSS ID 为 T_${uuid}
    string caption = 2;              // 表格标题
    string styles = 3;               // 整表 CSS 字符串
    bytes display_values = 4;        // 格式化值的 Arrow Table (另一组 IPC bytes)
  }
}
```

**关键设计**：`data` 字段是扁平的 `bytes`，而非结构化字段。Arrow 的 schema/metadata 全部内嵌在 IPC 字节流内部。

### 3.2 交互式表格：Dataframe.proto

[Dataframe.proto](./proto/streamlit/proto/Dataframe.proto) 是 `st.dataframe` 和 `st.data_editor` 的容器：

```protobuf
message Dataframe {
  ArrowData arrow_data = 1;              // 核心数据
  string id = 2;                         // Widget ID（可编辑/可选择时必填）
  string columns = 3;                    // 列配置 JSON
  EditingMode editing_mode = 4;          // READ_ONLY / FIXED / DYNAMIC / ADD_ONLY / DELETE_ONLY
  bool disabled = 5;
  string form_id = 6;                    // 所属表单 ID
  repeated string column_order = 7;      // 列显示顺序
  repeated SelectionMode selection_mode = 8;  // 行/列/单元格选择模式
  optional uint32 row_height = 9;
  optional string placeholder = 10;      // 空值占位符文本
  optional string selection_state = 11;  // 编程式设置的选中态 JSON
  optional string selection_default = 12; // 默认选中态 JSON
  map<string, string> button_click_widgets = 13;  // 按钮列的 widget ID 映射
}
```

### 3.3 自定义组件 v1：Components.proto

[Components.proto](./proto/streamlit/proto/Components.proto) 使用了 **独立的 Arrow 体系**（早于内部 Arrow 序列化实现）：

```protobuf
message ArrowTable {
  bytes data = 1;       // 主体数据的 Arrow IPC bytes
  bytes index = 2;      // 索引列 → 独立的 Arrow IPC bytes！
  bytes columns = 3;    // 列名 → 独立的 Arrow IPC bytes！
  ArrowTableStyler styler = 5;
}
```

**与内部实现的差异**：
- 内部实现：index 和 columns 作为额外字段嵌入同一个 Arrow Table（通过 Pandas Schema metadata 识别）
- 组件 v1：index、columns、data 是 **三组独立的 IPC 字节**，需分别反序列化后重组
- 对应实现：[component_arrow.py](./lib/streamlit/components/v1/component_arrow.py)

```python
# 反序列化时重组
data = convert_arrow_bytes_to_pandas_df(proto.data)
index = convert_arrow_bytes_to_pandas_df(proto.index)
columns = convert_arrow_bytes_to_pandas_df(proto.columns)
return pd.DataFrame(
    data.to_numpy(),
    index=index.to_numpy().T.tolist(),    # index 转置后作为行索引
    columns=columns.to_numpy().T.tolist(),  # columns 转置后作为列名
)
```

### 3.4 自定义组件 v2：BidiComponent.proto

双向组件使用了 **混合序列化策略**：

```protobuf
message MixedData {
  string json = 1;                    // 主体 JSON 字符串
  map<string, ArrowData> arrow_blobs = 2;  // DataFrame 二进制 Blob 字典
}
```

**后端序列化逻辑** [serialization.py#L34-L139](./lib/streamlit/components/v2/bidi_component/serialization.py#L34-L139)：

```
输入 dict → 遍历第一层 key-value：
  ├─ 如果 value 是 dataframe-like：
  │   ├─ convert_anything_to_arrow_bytes(value) → bytes
  │   ├─ ref_id = calc_hash(arrow_bytes)   （内容寻址，天然去重）
  │   ├─ arrow_blobs[ref_id] = arrow_bytes
  │   └─ dict[key] = {"__arrow_ref__": ref_id}  占位符
  └─ 否则保持原样
输出：
  MixedData {
    json: json.dumps(处理后的 dict)
    arrow_blobs: {ref_id: ArrowData{data: bytes}, ...}
  }
```

**优势**：JSON 处理标量参数，Arrow 承载大表二进制，互不干扰。相同 DataFrame 的多次引用只传输一份。

### 3.5 Vega-Lite 图表：VegaLiteChart.proto

图表数据也复用 ArrowData：
```protobuf
message VegaLiteChart {
  ArrowData data = 2;                      // 主数据集
  repeated ArrowNamedDataSet datasets = 3; // 命名数据集（Vega datasets API）
}
```

[ArrowNamedDataSet.proto](./proto/streamlit/proto/ArrowNamedDataSet.proto)：
```protobuf
message ArrowNamedDataSet {
  string name = 1;
  bool has_name = 3;      // proto3 无 has_field，需手动标记
  ArrowData data = 2;
}
```

---

## 四、传输边界：ForwardMsg → WebSocket → 前端接收完整链路

### 4.1 Delta → Element → ForwardMsg 封装链

消息嵌套层级：
```
ForwardMsg
  └─ Delta
       └─ Element
            ├─ Dataframe { arrow_data: ArrowData{data: bytes, ...}, ... }
            ├─ Table { arrow_data: ArrowData{...}, ... }
            ├─ VegaLiteChart { data: ArrowData{...}, datasets: [...] }
            ├─ ComponentInstance { special_args: [SpecialArg{ArrowDataframe}], ... }
            └─ BidiComponent { mixed_data: MixedData{json, arrow_blobs:{...}}, ... }
```

[ForwardMsg.proto](./proto/streamlit/proto/ForwardMsg.proto#L36-L111) 核心结构：

```protobuf
message ForwardMsg {
  string hash = 1;                    // 消息哈希（用于去重缓存）
  ForwardMsgMetadata metadata = 2;    // 不参与哈希的元数据
  oneof type {
    Delta delta = 5;                  // 绝大多数 UI 元素通过此类型传输
    string ref_hash = 11;             // 缓存命中引用（关键优化！）
    ...
  }
}

message ForwardMsgMetadata {
  bool cacheable = 1;                 // 是否可被前端缓存
  repeated uint32 delta_path = 2;     // Delta 在渲染树中的位置路径
  string active_script_hash = 4;
}
```

### 4.2 WebSocket 连接建立与接收层

#### 连接管理器：ConnectionManager

[ConnectionManager.ts](./frontend/connection/src/ConnectionManager.ts) 是连接层的入口，负责创建 `WebsocketConnection` 实例。它作为工厂和状态协调者，不直接处理消息。

#### WebSocket 连接：WebsocketConnection

[WebsocketConnection.tsx](./frontend/connection/src/WebsocketConnection.tsx#L699-L723) 是实际的 WebSocket 接收层。这是一个状态机驱动的类，管理连接生命周期和消息接收。

**WebSocket 消息接收的完整代码路径：**

```typescript
// WebsocketConnection.tsx L552-L560
this.websocket.addEventListener("message", (event: MessageEvent) => {
  if (checkWebsocket()) {
    this.handleMessage(event.data).catch(reason => {
      const err = `Failed to process a Websocket message. ${reason}`
      LOG.error(err)
      this.stepFsm("FATAL_ERROR", err)
    })
  }
})
```

**关键要点**：
- `checkWebsocket()` 是防御性检查，确保当前处理的事件仍属于活跃的 WebSocket 实例（防止重连后的消息交叉）
- `event.data` 是 `ArrayBuffer` 类型（WebSocket `binaryType = "arraybuffer"`，见 L545）
- 整个处理是异步的，任何解析错误都会触发 FSM 进入 `DISCONNECTED_FOREVER` 状态

### 4.3 消息解码与去重缓存：handleMessage()

[WebsocketConnection.tsx](./frontend/connection/src/WebsocketConnection.tsx#L699-L723) 的 `handleMessage()` 是第一个处理函数，负责：

```typescript
private async handleMessage(data: ArrayBuffer): Promise<void> {
  // 步骤1: 分配消息序列号（保证顺序）
  const messageIndex = this.nextMessageIndex
  this.nextMessageIndex += 1

  // 步骤2: Protobuf 二进制解码 → ForwardMsg 对象
  const encodedMsg = new Uint8Array(data)
  const msg = ForwardMsg.decode(encodedMsg)

  // 步骤3: 交给缓存处理（去重 + 引用还原）
  this.messageQueue[messageIndex] = await this.cache.processMessagePayload(
    msg,
    encodedMsg
  )

  // 步骤4: 按顺序分发队列中的消息
  while (this.lastDispatchedMessageIndex + 1 in this.messageQueue) {
    const dispatchMessageIndex = this.lastDispatchedMessageIndex + 1
    this.args.onMessage(this.messageQueue[dispatchMessageIndex])
    delete this.messageQueue[dispatchMessageIndex]
    this.lastDispatchedMessageIndex = dispatchMessageIndex
  }
}
```

**顺序保证机制**：
- `nextMessageIndex` 单调递增分配给每个到来的消息
- `messageQueue` 是一个普通对象作为字典存储（`{[index: number]: ForwardMsg}`）
- `lastDispatchedMessageIndex` 跟踪最后一个已分发的索引
- 消息解码可能是异步的（不同消息大小差异导致解码耗时不同），但分发时严格按索引顺序

---

### 4.4 去重匹配与引用还原：ForwardMessageCache

[ForwardMessageCache.ts](./frontend/connection/src/ForwardMessageCache.ts) 是整个缓存机制的核心。它的 `processMessagePayload()` 方法处理两种消息类型：

#### 4.4.1 缓存入口：processMessagePayload()

```typescript
// ForwardMessageCache.ts L119-L146
public async processMessagePayload(
  msg: ForwardMsg,
  encodedMsg: Uint8Array
): Promise<ForwardMsg> {
  // 步骤1: 先尝试缓存（如果是 cacheable 且不是 refHash）
  this.maybeCacheMessage(msg, encodedMsg)

  // 步骤2: 如果不是引用消息，直接返回
  if (msg.type !== "refHash") {
    return msg
  }

  // 步骤3: 是 refHash 消息 → 从缓存中取出原始消息
  const cachedMessage = this.getCachedMessage(msg.refHash as string, true)
  if (isNullOrUndefined(cachedMessage)) {
    throw new Error(`Cached ForwardMsg MISS [hash=${msg.refHash}]...`)
  }
  LOG.info(`Cached ForwardMsg HIT [hash=${msg.refHash}]`)

  // 步骤4: 将引用消息的 metadata 合并到原始消息
  if (!msg.metadata) {
    throw new Error("Reference ForwardMsg has no metadata...")
  }
  cachedMessage.metadata = msg.metadata  // metadata 用最新的（可能包含不同的 delta_path）
  return cachedMessage
}
```

#### 4.4.2 缓存写入逻辑：maybeCacheMessage()

```typescript
// ForwardMessageCache.ts L151-L194
private maybeCacheMessage(msg: ForwardMsg, encodedMsg: Uint8Array): void {
  // 条件1: 永远不缓存 refHash 类型的消息
  if (msg.type === "refHash") {
    return
  }

  // 条件2: 只有服务器标记为 cacheable 的消息才缓存
  if (!msg.metadata?.cacheable) {
    return
  }

  // 条件3: 必须有 hash 字段
  if (!msg.hash) {
    LOG.error("ForwardMsg has no hash...")
    return
  }

  // 条件4: 如果已缓存，只更新 scriptRunCount（不重复存储）
  if (this.getCachedMessage(msg.hash, true) !== undefined) {
    return
  }

  // 满足所有条件 → 存入缓存
  LOG.info(`Caching ForwardMsg [hash=${msg.hash}]`)
  this.messages.set(
    msg.hash,
    new CacheEntry(
      encodedMsg,                          // 存原始二进制（不是解码后的对象）
      this.scriptRunCount,
      msg.delta?.fragmentId ?? undefined
    )
  )
}
```

**关键设计决策**：
- 缓存存储的是原始 `Uint8Array` 编码消息，而非解码后的 `ForwardMsg` 对象。这是为了节省内存（避免同时保留两份数据），同时保持精确性（反序列化可能不具有完全的往返一致性）。

#### 4.4.3 缓存读取逻辑：getCachedMessage()

```typescript
// ForwardMessageCache.ts L203-L216
private getCachedMessage(
  hash: string,
  updateScriptRunCount: boolean
): ForwardMsg | undefined {
  const cachedEntry = this.messages.get(hash)
  if (isNullOrUndefined(cachedEntry)) {
    return undefined
  }

  // 更新 LRU 时间戳（用 scriptRunCount 代替）
  if (updateScriptRunCount) {
    cachedEntry.scriptRunCount = this.scriptRunCount
  }

  // 动态解码：只有被引用时才解码
  return ForwardMsg.decode(cachedEntry.encodedMsg)
}
```

#### 4.4.4 缓存过期与清理：maxCachedMessageAge 的完整传递链

**Age 算法**：不是标准的 LRU，而是基于"脚本运行次数"的 Age 算法。`ForwardMessageCache` 维护一个全局的 `scriptRunCount` 计数器，每次脚本运行完成（成功完成或 fragment 完成）时递增 1。每个缓存条目在被访问（`maybeCacheMessage` 写入或 `getCachedMessage` 读取）时，会更新 `entry.scriptRunCount = this.scriptRunCount`。条目年龄 `age = currentRunCount - entry.scriptRunCount`。

`maxCachedMessageAge` 的值由 **四层链路** 确定：

```
第 1 层：配置文件
   .streamlit/config.toml → global.maxCachedMessageAge
   （默认值由后端 Server 配置决定）
         │
         ▼  proto
第 2 层：NewSession 消息
   NewSession.proto → Config.max_cached_message_age (int32, field 3)
   见 [NewSession.proto L92-L97](./proto/streamlit/proto/NewSession.proto#L92-L97)
         │
         ▼  SessionInfo.propsFromNewSessionMessage()
第 3 层：会话信息缓存
   SessionInfo.ts → Props.maxCachedMessageAge
   见 [SessionInfo.ts L40, L116](./frontend/lib/src/SessionInfo.ts#L40, L116)
   在 NewSession 消息到达时被初始化，会话生命周期内不变
         │
         ▼  handleScriptFinished()
第 4 层：调用时传入
   App.tsx → this.sessionInfo.current.maxCachedMessageAge
   作为 incrementMessageCacheRunCount 的第一个参数
```

**调用条件**（四层门禁）：
```typescript
// App.tsx L1641-L1656
if (
  this.connectionManager !== null &&
  status !== ForwardMsg.ScriptFinishedStatus.FINISHED_EARLY_FOR_RERUN &&  // ⚠️ 被 rerun 打断的脚本 → 不清理！
  this.sessionInfo.isSet &&                                                // 会话已初始化
  this.hasReceivedNewSession                                               // 当前 run 的 NewSession 已收到
) {
  this.connectionManager.incrementMessageCacheRunCount(
    this.sessionInfo.current.maxCachedMessageAge,  // ← 来自会话配置，不是硬编码的 1 或 2
    this.state.fragmentIdsThisRun                   // ← 直接使用 state.fragmentIdsThisRun 数组
  )
}
```

**关键修正（与旧版描述的差异）**：
1. ~~`FINISHED_EARLY_FOR_RERUN` 传入 `maxMessageAge = 2`~~ → **事实是 FINISHED_EARLY_FOR_RERUN 被直接排除，不会调用清理**。目的是避免 rerun 期间提前删除可能被新脚本需要的缓存消息。
2. ~~`maxMessageAge` 是硬编码的 1 或 2~~ → **事实是来自 `sessionInfo.current.maxCachedMessageAge`，由后端配置决定**。
3. ~~`fragmentIdsThisRun` 用 `currentFragmentId` 三元表达式构造~~ → **事实是直接使用 `this.state.fragmentIdsThisRun`**，该数组在 NewSession 消息到达时被赋值。

---

##### 完整清理逻辑：ForwardMessageCache.incrementRunCount()

```typescript
// ForwardMessageCache.ts L70-L100
public incrementRunCount(
  maxMessageAge: number,
  fragmentIdsThisRun: string[]
): void {
  // ── 全局计数器递增（无论是否 fragment run）──
  // ⚠️ 已知限制：fragment run 也会递增全局 scriptRunCount，
  // 导致非 fragment 消息的 age 同样增加。如果后续紧跟多次 fragment run
  // 后才 full rerun，非 fragment 消息可能因 age 超标被误删。
  // 注释说明："技术 overhead 大于收益，暂不做 per-fragment 计数"
  this.scriptRunCount += 1

  this.messages.forEach((entry, hash) => {
    // ── 过滤 1：fragment run 专属逻辑 ──
    if (
      fragmentIdsThisRun.length > 0 &&
      (!entry.fragmentId || !fragmentIdsThisRun.includes(entry.fragmentId))
    ) {
      // fragment run 期间：只处理属于本次 fragment 的消息
      // 非本次 fragment 的消息 → 直接跳过（不判断 age，不删除）
      return
    }

    // ── 过滤 2：年龄判断 ──
    if (entry.getAge(this.scriptRunCount) > maxMessageAge) {
      LOG.info(`Removing expired ForwardMsg [hash=${hash}]`)
      this.messages.delete(hash)
    }
  })
}
```

**Fragment run 期间哪些消息会被纳入老化处理？**

| 消息类型 | fragmentId 属性 | fragmentIdsThisRun 非空时 | 是否被老化判断 |
|---------|----------------|-------------------------|--------------|
| **主脚本完整 run 的消息** | `undefined` | `!entry.fragmentId` 为 true → 命中过滤 1 → return | ❌ 跳过 |
| **本次 fragment 的消息** | `"frag_abc"` | 在 fragmentIdsThisRun 中 → 不命中过滤 1 → 继续 | ✅ 参与老化 |
| **其他 fragment 的消息** | `"frag_xyz"` | 不在 fragmentIdsThisRun 中 → 命中过滤 1 → return | ❌ 跳过 |
| **无 fragmentId 的消息** | `undefined` | `!entry.fragmentId` → 命中过滤 1 → return | ❌ 跳过 |

**简而言之**：fragment run 期间，只有**属于本次 fragment 的消息**才会被纳入老化判断和可能的清理。非本次 fragment 的消息（包括主脚本消息、其他 fragment 消息）全部跳过。

**但注意全局计数器的副作用**：即使非 fragment 消息不被清理，它们的 `age`（= currentRunCount - entry.scriptRunCount）仍在不断增大（因为 scriptRunCount 每次都 +1）。如果连续进行 N 次 fragment run 后才执行 full rerun，而 N > maxCachedMessageAge，那么即使非 fragment 消息从未被"跳过老化"，在随后的 full rerun 中也会因为 age 超标而被删除。

---

**ScriptFinishedStatus 触发时机总结**：

| 状态值 | 含义 | 是否触发 clearStaleNodes | 是否触发 incrementMessageCacheRunCount |
|-------|------|-------------------------|--------------------------------------|
| `FINISHED_SUCCESSFULLY` (0) | 完整脚本运行成功 | ✅ 是 | ✅ 是（使用会话配置的 maxCachedMessageAge） |
| `FINISHED_WITH_COMPILE_ERROR` (1) | 编译错误 | ❌ 否 | ❌ 否 |
| `FINISHED_EARLY_FOR_RERUN` (2) | 被 rerun 打断 | ❌ 否（防止闪烁） | ❌ 否（防止误删） |
| `FINISHED_FRAGMENT_RUN_SUCCESSFUL` (3) | fragment 运行成功 | ✅ 是 | ✅ 是（带 fragmentIdsThisRun 过滤） |

**对于 Arrow 数据的意义**：几 MB 甚至几十 MB 的 Arrow bytes 不会在 rerun 时重复传输。当同一个 DataFrame 在多次 rerun 中出现时：
- 第 1 次：发送完整消息 + `metadata.cacheable = true` → 前端缓存，`entry.scriptRunCount = currentRunCount`
- 第 2 次：相同消息被访问 → `entry.scriptRunCount` 更新为新的 `currentRunCount` → age = 0
- 第 3 次及以后：如果 `elementHash` 相同且未被清理（age ≤ maxCachedMessageAge），后端只需发送 `ForwardMsg{ref_hash: "abc123", metadata: {...}}`（约 50 字节），前端还原成原始消息
- 超过 `maxCachedMessageAge` 次 rerun 未被访问 → 自动清理

---

### 4.5 消息分发：App.tsx handleMessage()

消息队列按顺序分发后，到达 [App.tsx](./frontend/app/src/App.tsx#L968-L1042) 的 `handleMessage()` 回调。这是消息类型路由的核心：

```typescript
handleMessage = (msgProto: ForwardMsg): void => {
  const dispatchProto = (
    obj: ForwardMsg,
    name: string,
    funcs: Record<string, (value: any) => void>
  ): void => {
    const whichOne = (obj as unknown as Record<string, unknown>)[
      name
    ] as string
    if (whichOne in funcs) {
      return funcs[whichOne](
        (obj as unknown as Record<string, unknown>)[whichOne]
      )
    }
    throw new Error(`Cannot handle ${name} "${whichOne}".`)
  }

  try {
    dispatchProto(msgProto, "type", {
      newSession: (newSessionMsg: NewSession) =>
        this.handleNewSession(newSessionMsg),
      sessionStatusChanged: (msg: SessionStatus) =>
        this.handleSessionStatusChanged(msg),
      sessionEvent: (evtMsg: SessionEvent) =>
        this.handleSessionEvent(evtMsg),
      delta: (deltaMsg: Delta) =>
        this.handleDeltaMsg(
          deltaMsg,
          msgProto.metadata as ForwardMsgMetadata,
          msgProto.hash  // elementHash 传递给下游做 memoization
        ),
      pageConfigChanged: (pageConfig: PageConfig) =>
        this.handlePageConfigChanged(pageConfig),
      scriptFinished: (status: ForwardMsg.ScriptFinishedStatus) =>
        this.handleScriptFinished(status),
      // ... 其他 10+ 种消息类型
    })
  } catch (e) {
    const err = ensureError(e)
    LOG.error(err)
    this.showError("Bad message format", { message: err.message })
  }
}
```

**关键点**：
- `msgProto.hash` 会传递给 `handleDeltaMsg`，最终成为 ElementNode 的 `elementHash`，用于前端组件的 memoization
- 对于 Arrow 数据来说，这个 `hash` 是避免重复解析的关键锚点

---

### 4.6 Delta 应用：AppRoot.applyDelta() — 三大分支与 Visitor 模式协作

`handleDeltaMsg` 调用 `AppRoot.applyDelta()` 将 Delta 应用到渲染树：

```typescript
// App.tsx L1757-L1771
handleDeltaMsg = (
  deltaMsg: Delta,
  metadataMsg: ForwardMsgMetadata,
  elementHash?: string
): void => {
  this.setState(prevState => ({
    elements: prevState.elements.applyDelta(
      prevState.scriptRunId,
      deltaMsg,
      metadataMsg,
      elementHash
    ),
  }))
}
```

`applyDelta` 是整个更新流程的调度器，它不直接修改树结构，而是：
1. 根据 `delta.type` 分派到三个处理分支
2. 每个分支完成节点准备（payload 复用判定、children 继承等）后
3. 统一调用 `SetNodeByDeltaPathVisitor` 执行不可变写入

#### 4.6.1 Delta 类型与三大分支概览

[Delta.proto](./proto/streamlit/proto/Delta.proto#L24-L39) 定义了三种 Delta 类型：

```protobuf
message Delta {
  oneof type {
    Element new_element = 3;      // 新增元素（st.dataframe, st.table 等）
    Block add_block = 6;          // 新增块节点（容器布局）
    Transient new_transient = 9;  // 新增临时节点（spinner, toast 等）
  }
}
```

对应 [AppRoot.ts](./frontend/lib/src/render-tree/AppRoot.ts#L238-L301) 的 `applyDelta()` 有三大分支，**处理差异对照表**：

| 特性 | newElement 分支 | addBlock 分支 | newTransient 分支 |
|------|-----------------|---------------|-------------------|
| **触发场景** | st.dataframe / st.text 等元素 | st.container / st.sidebar 等容器 | st.spinner / toast 等临时内容 |
| **创建的节点类型** | `ElementNode` | `BlockNode` | `TransientNode` (初始 `anchor=undefined`) |
| **需要定位 existingNode？** | 是（用于 payload 复用判断） | 是（用于 children 继承判断） | **否**（不需要任何现有节点信息） |
| **GetNodeByDeltaPathVisitor？** | 有 | 有 | 无 |
| **canReuseElementPayload？** | **有（唯一有此判断的分支）** | 无 | 无 |
| **children 继承？** | 无（叶子节点） | 有（同类型 Block 继承） | 无（由 transientNodes 列表承载） |
| **创建后写入方式** | `SetNodeByDeltaPathVisitor.setNodeAtPath()` | 同左 | 同左（由 Visitor 内部捕获 anchor） |

---

#### 4.6.2 分支 1：newElement — 唯一有 payload 复用的路径

```typescript
// AppRoot.ts L248-L271
case "newElement": {
  const nextElement = delta.newElement as Element

  // ── 阶段 1：定位现有节点（仅用于复用判断，不参与写入）──
  const existingNode = GetNodeByDeltaPathVisitor.getNodeAtPath(
    this.root,
    deltaPath
  )

  // ── 阶段 2：payload 复用判定（精确位置：applyDelta 内部，addElement 之前）──
  const canReuse = canReuseElementPayload(
    existingNode,
    elementHash,
    nextElement
  )

  // ── 阶段 3：创建 ElementNode 并写入 ──
  return this.addElement(
    deltaPath,
    scriptRunId,
    canReuse ? existingNode.element : nextElement,  // ← 关键决策点
    metadata,
    activeScriptHash,
    delta.fragmentId,
    elementHash
  )
}
```

##### 元素 payload 复用：canReuseElementPayload()

**精确代码位置**：[AppRoot.ts](./frontend/lib/src/render-tree/AppRoot.ts#L53-L66)

```typescript
function canReuseElementPayload(
  existingNode: AppNode | undefined,
  elementHash: string | undefined,
  nextElement: Element
): existingNode is ElementNode {
  return (
    Boolean(elementHash) &&                    // 条件1: 有 elementHash（后端设置）
    nextElement.type !== undefined &&          // 条件2: 新元素有类型字段
    existingNode instanceof ElementNode &&      // 条件3: 现有节点必须是 ElementNode
    existingNode.elementHash === elementHash && // 条件4: hash 完全匹配（内容一致）
    existingNode.element.type === nextElement.type && // 条件5: 元素类型相同（防跨类型误用）
    !nextElement.hasOneShotEffect              // 条件6: 非一次性效果（如弹窗类需重建）
  )
}
```

**复用的传导链路**：
```
canReuse = true
    │
    ▼  传入 addElement()
element = existingNode.element  （旧 element 对象引用，含 Arrow bytes）
    │
    ▼  new ElementNode(element, ..., elementHash)
ElementNode.element === 旧 ElementNode.element
    │
    ▼  React 组件
useMemo(() => new Quiver(element.arrowData), [elementHash, element.arrowData])
    │  依赖项 elementHash 相同 → 跳过执行
    ▼
不重新解析 Arrow IPC bytes ✅
```

> **注意**：即使 ForwardMsgCache 未命中（例如跨不同 rerun 的不同 ForwardMsg hash），只要内容一致（`elementHash` 相同），此复用机制仍生效。

##### addElement() 实际工作

[AppRoot.ts](./frontend/lib/src/render-tree/AppRoot.ts#L414-L441) — 极为简洁，仅做两件事：
1. 构造 `ElementNode`（`element` 参数可能是复用的旧对象，也可能是新对象）
2. 调用 `SetNodeByDeltaPathVisitor.setNodeAtPath()` 完成不可变写入

```typescript
private addElement(
  deltaPath: number[],
  scriptRunId: string,
  element: Element,          // ← 复用 or 新建，由调用方决定
  metadata: ForwardMsgMetadata,
  activeScriptHash: string,
  fragmentId?: string,
  elementHash?: string
): AppRoot {
  const elementNode = new ElementNode(
    element, metadata, scriptRunId, activeScriptHash, fragmentId, elementHash
  )
  return new AppRoot(
    this.mainScriptHash,
    SetNodeByDeltaPathVisitor.setNodeAtPath(
      this.root, deltaPath, elementNode, scriptRunId
    ) as BlockNode,
    this.appLogo
  )
}
```

---

#### 4.6.3 分支 2：addBlock — 有 children 继承的路径

```typescript
// AppRoot.ts L273-L283
case "addBlock": {
  const deltaMsgReceivedAt = Date.now()
  return this.addBlock(
    deltaPath, delta.addBlock as BlockProto,
    scriptRunId, activeScriptHash, delta.fragmentId, deltaMsgReceivedAt
  )
}
```

##### addBlock() 四步处理流程

[AppRoot.ts](./frontend/lib/src/render-tree/AppRoot.ts#L443-L503)

```typescript
private addBlock(...): AppRoot {
  // ═══ 步骤 1：定位现有节点（GetNodeByDeltaPathVisitor）═══
  const existingNodeAtPath = GetNodeByDeltaPathVisitor.getNodeAtPath(
    this.root, deltaPath
  )
  // 关键：TransientNode 透明穿透——子节点继承操作的是 anchor
  const existingNode = existingNodeAtPath instanceof TransientNode
    ? (existingNodeAtPath.anchor ?? existingNodeAtPath)
    : existingNodeAtPath

  // ═══ 步骤 2：同类型 Block 继承 children ═══
  let children: AppNode[] = []
  if (
    existingNode instanceof BlockNode &&
    existingNode.deltaBlock.type === block.type
  ) {
    // Dialog 例外：身份不同则不继承（防不同 dialog 的内容串台）
    const isDialogWithDifferentIdentity =
      block.dialog && existingNode.deltaBlock.dialog &&
      block.id !== existingNode.deltaBlock.id

    if (!isDialogWithDifferentIdentity) {
      children = existingNode.children  // 引用相等，下游 React 不重渲染
    }
  }

  // ═══ 步骤 3：创建新 BlockNode ═══
  const blockNode = new BlockNode(
    activeScriptHash, children, block,
    scriptRunId, fragmentId, deltaMsgReceivedAt
  )

  // ═══ 步骤 4：不可变写入 ═══
  return new AppRoot(
    this.mainScriptHash,
    SetNodeByDeltaPathVisitor.setNodeAtPath(
      this.root, deltaPath, blockNode, scriptRunId
    ) as BlockNode,
    this.appLogo
  )
}
```

**children 继承的意义**：当 `st.container` 在 rerun 中被重新创建时，其内部的 `st.dataframe` 等子节点保持引用相等，从而避免子组件重渲染、避免 Arrow 数据重新解析。

---

#### 4.6.4 分支 3：newTransient — 无需现有节点，anchor 在 Visitor 中捕获

```typescript
// AppRoot.ts L285-L295
case "newTransient": {
  const transient = delta.newTransient as TransientProto
  return this.addTransient(
    deltaPath, scriptRunId, transient, metadata,
    activeScriptHash, delta.fragmentId
  )
}
```

##### addTransient() 极简流程

[AppRoot.ts](./frontend/lib/src/render-tree/AppRoot.ts#L505-L540)

```typescript
addTransient(...): AppRoot {
  const transientNode = new TransientNode(
    scriptRunId,
    undefined,  // ═══ 关键：初始 anchor = undefined ═══
    transient.elements.map(e => new ElementNode(
      e as Element, metadata, scriptRunId, activeScriptHash, fragmentId
    )),
    deltaMsgReceivedAt
  )

  return new AppRoot(
    this.mainScriptHash,
    SetNodeByDeltaPathVisitor.setNodeAtPath(
      this.root, deltaPath, transientNode, scriptRunId
    ) as BlockNode,
    this.appLogo
  )
}
```

**为什么不在这里设置 anchor？**
因为 `addTransient()` 不知道目标位置上原来有什么节点。anchor 的捕获必须在 `SetNodeByDeltaPathVisitor` 沿路径找到目标位置后，**在将要替换目标节点的那一刻**执行。详见 4.6.7 节。

---

#### 4.6.5 节点访问器（Visitor 模式）基础

整个渲染树操作基于 **Visitor 设计模式**，核心接口：

- [AppNode.interface.ts](./frontend/lib/src/render-tree/AppNode.interface.ts#L98-L99) — `accept<T>(visitor: AppNodeVisitor<T>): T`
- [AppNodeVisitor.interface.ts](./frontend/lib/src/render-tree/visitors/AppNodeVisitor.interface.ts#L21-L24) — 三个 visit 方法

**双分派原理**：
```
调用方                          节点 N                        访问器 V
   │                              │                             │
   │   N.accept(V)                │                             │
   │ ───────────────────────────> │                             │
   │                              │                             │
   │                              │   V.visitXXX(N, ...)        │
   │                              │ ───────────────────────────>│
   │                              │                             │
   │                              │                             │ 处理逻辑
   │                              │         返回结果            │
   │         返回结果             │ <─────────────────────────── │
   │ <─────────────────────────── │                             │
```

三种节点的 `accept` 实现（完全对称，各调各的 visit）：

| 节点类型 | `accept` 实现 | 代码位置 |
|---------|-------------|---------|
| `BlockNode` | `return visitor.visitBlockNode(this)` | [BlockNode.ts L108-L109](./frontend/lib/src/render-tree/BlockNode.ts#L108-L109) |
| `ElementNode` | `return visitor.visitElementNode(this)` | [ElementNode.ts L69-L70](./frontend/lib/src/render-tree/ElementNode.ts#L69-L70) |
| `TransientNode` | `return visitor.visitTransientNode(this)` | [TransientNode.ts L74-L75](./frontend/lib/src/render-tree/TransientNode.ts#L74-L75) |

---

#### 4.6.6 读访问器：GetNodeByDeltaPathVisitor

[GetNodeByDeltaPathVisitor.ts](./frontend/lib/src/render-tree/visitors/GetNodeByDeltaPathVisitor.ts) — 根据 `deltaPath` 定位节点。返回类型为 `AppNode | undefined`，所有"找不到"的情况都返回 `undefined`（而非抛出异常）。

**逐方法精确语义分析**：

##### visitElementNode()：叶子节点，永远返回 undefined

```typescript
// GetNodeByDeltaPathVisitor.ts L42-L45
visitElementNode(_node: ElementNode): AppNode | undefined {
  // ElementNodes have no children, so there is nothing to check
  return undefined
}
```

**关键：不检查路径，不抛出异常，无论空路径还是剩余路径，一律返回 undefined。**

测试用例确认此行为：[GetNodeByDeltaPathVisitor.test.ts L27-L44](./frontend/lib/src/render-tree/visitors/GetNodeByDeltaPathVisitor.test.ts#L27-L44)
- 路径 `[0]` → `undefined` ✅
- 路径 `[]`（空路径）→ `undefined` ✅

**为什么不抛错？设计理由**：
1. **统一查找语义**：Visitor 的返回类型是 `AppNode | undefined`，`undefined` 统一表示"目标不存在"。查找失败是正常的业务情况（例如第一次写入某位置时就没有现有节点），不是异常。
2. **调用方是 AppRoot.ts 的现有节点定位**：在 `applyDelta()` 的 `newElement` / `addBlock` 分支中，`GetNodeByDeltaPathVisitor.getNodeAtPath()` 用于查找 `existingNode`，结果为 `undefined` 时表示"该位置还没有节点"（全新写入），直接走正常新建分支即可，不需要额外的 try-catch 处理。
3. **不会被错误调用**：正常调用链中，BlockNode 会在 `remainingPath.length === 0` 时直接返回 `node.children[currentIndex]`（不会再调用 `child.accept(childVisitor)`），只有当路径还有剩余层级时才会向下递归。如果 child 恰好是 ElementNode（说明路径写多了，层级过深），静默返回 `undefined` 比抛出更优雅——等同于"在这个过深的位置找不到节点"。

---

##### visitBlockNode()：递归消耗路径索引

```typescript
// GetNodeByDeltaPathVisitor.ts L47-L67
visitBlockNode(node: BlockNode): AppNode | undefined {
  // ── 情形 A：路径为空 ──
  // Block 本身不是查找目标（只有 Block 的 children 才是）
  // 空路径意味着"我已经到达 Block 节点自身的位置，但没有指定 child 索引"→ 返回 undefined
  if (this.deltaPath.length === 0) {
    return undefined
  }

  const [currentIndex, ...remainingPath] = this.deltaPath

  // ── 情形 B：索引越界 ──
  // children.length 是精确边界：0 <= index < children.length
  if (currentIndex < 0 || currentIndex >= node.children.length) {
    return undefined
  }

  // ── 情形 C：路径全部消耗完毕 ──
  // remainingPath 为空 → children[currentIndex] 就是目标节点，直接返回
  if (remainingPath.length === 0) {
    return node.children[currentIndex]   // ← 命中！此分支不调用 .accept()
  }

  // ── 情形 D：还有剩余路径 ──
  // 创建新 visitor（携带 remainingPath），继续向下递归消耗
  const childVisitor = new GetNodeByDeltaPathVisitor(remainingPath)
  return node.children[currentIndex].accept(childVisitor)
}
```

**关键细节**：情形 C 命中时不调用 `child.accept()`，直接返回 child。这意味着目标节点若是 ElementNode，`visitElementNode()` 在查找成功场景下根本不会被调用。

测试用例覆盖：[GetNodeByDeltaPathVisitor.test.ts L47-L146](./frontend/lib/src/render-tree/visitors/GetNodeByDeltaPathVisitor.test.ts#L47-L146)
- 空路径 `[]` → `undefined`
- 浅路径 `[0]` → 返回 `children[0]`
- 深路径 `[1, 0]` → 递归后返回嵌套 child
- 越界 `[-1]`/`[3]`/`[2]`（等于长度）→ `undefined`

---

##### visitTransientNode()：透明穿透，不消耗路径索引

```typescript
// GetNodeByDeltaPathVisitor.ts L69-L77
visitTransientNode(node: TransientNode): AppNode | undefined {
  // ── 情形 A：路径为空 ──
  // 到达 TransientNode 自身的位置，把 anchor 当作该位置的实际内容
  // ⚠️ 与 BlockNode 空路径行为不同！BlockNode 空路径返回 undefined，
  //    TransientNode 空路径返回 node.anchor（TransientNode 认为自己的"内容"就是 anchor）
  if (this.deltaPath.length === 0) {
    return node.anchor   // 可能为 undefined（anchor 未设置时）
  }

  // ── 情形 B：还有剩余路径 ──
  // 穿透到 anchor，使用 this（同一个 visitor，剩余路径保持不变！）
  // ✅ 关键：不 new 新 visitor，this 的 deltaPath 没有被 consume
  //         TransientNode 在 deltaPath 上是透明的，不占用任何索引位置
  // ✅ 使用 ?. 可选链：anchor 为 undefined 时整个表达式返回 undefined（不抛错）
  return node.anchor?.accept(this)
}
```

**与 BlockNode 的对比差异**：

| 维度 | BlockNode.visitBlockNode() | TransientNode.visitTransientNode() |
|------|---------------------------|-----------------------------------|
| 剩余路径非空时 | `new GetNodeByDeltaPathVisitor(remainingPath)` → **消耗 1 个索引** | `node.anchor?.accept(this)` → **不消耗索引** |
| 空路径时 | 返回 `undefined` | 返回 `node.anchor`（可能仍为 undefined） |
| 查找失败 | 索引越界/空块 → `undefined` | anchor 不存在 → `undefined` |

测试用例验证：[GetNodeByDeltaPathVisitor.test.ts L182-L221](./frontend/lib/src/render-tree/visitors/GetNodeByDeltaPathVisitor.test.ts#L182-L221)
- 空路径 `[]` + anchor = ElementNode → 返回该 anchor ✅
- 路径 `[1]` + anchor = BlockNode(`[t1, t2]`) → 返回 `t2`（路径 [1] 在穿透到 BlockNode 后才被消耗，找到 Block.children[1]）✅
- 越界路径 `[-1]`/`[5]` + anchor = ElementNode → `undefined`（穿透到 ElementNode 后路径仍有剩余，由 visitElementNode 返回 undefined）✅

---

##### 完整决策表（所有分支汇总）

| 方法 | deltaPath 状态 | 附加条件 | 返回值 |
|------|---------------|---------|-------|
| **visitElementNode** | 任意（空/非空） | 任意 | `undefined` |
| **visitBlockNode** | 空路径 `[]` | — | `undefined` |
| | 非空 | `currentIndex` 越界 | `undefined` |
| | 非空 | `remainingPath.length === 0` | `node.children[currentIndex]` |
| | 非空 | `remainingPath` 还有剩余 | 递归：`node.children[currentIndex].accept(new Visitor(remainingPath))` |
| **visitTransientNode** | 空路径 `[]` | — | `node.anchor`（可能为 undefined） |
| | 非空 | anchor 存在 | `node.anchor.accept(this)`（同一 visitor，路径不消耗） |
| | 非空 | anchor 不存在 | `undefined`（?. 短路） |

**`visitTransientNode` 的透明性总结**：
- TransientNode 在 `deltaPath` 的索引体系中**不占位置**——它的 wrapper 层对路径查找是不可见的
- 穿透时使用同一个 `this` visitor，保证剩余路径被传递给 anchor 时不被修改
- 空路径时直接暴露 anchor，这让上层调用 `GetNodeByDeltaPathVisitor.getNodeAtPath(root, deltaPath)` 在目标位置有 TransientNode 时也能拿到真实内容（如 `addBlock()` 中的 `existingNodeAtPath instanceof TransientNode ? ...anchor...` 分支）
- **但注意**：空路径时 BlockNode 返回 undefined，而 TransientNode 返回 anchor，两者语义不一致。这是因为 BlockNode 认为"自己是容器、children 才是内容"，而 TransientNode 认为"anchor 就是自己的内容、transientNodes 只承载附加的临时显示层"。

---

#### 4.6.7 写访问器：SetNodeByDeltaPathVisitor — 核心交互逻辑

[SetNodeByDeltaPathVisitor.ts](./frontend/lib/src/render-tree/visitors/SetNodeByDeltaPathVisitor.ts) — 最复杂的访问器，承担三项职责：
1. **不可变更新**：沿路径创建新 BlockNode
2. **Anchor 捕获**：当 TransientNode（`anchor=undefined`）替换目标节点时，捕获原节点作为 anchor
3. **插入 vs 替换判定**：处理"透明节点与非透明节点替换"的边界情况

三种 `visit` 方法逐条分析：

##### visitElementNode()：叶子节点替换

```typescript
// SetNodeByDeltaPathVisitor.ts L42-L59
visitElementNode(node: ElementNode): AppNode {
  if (this.deltaPath.length > 0) throw new Error("...")  // 不应有剩余路径

  // ═══ Anchor 捕获场景：用 TransientNode 替换 ElementNode ═══
  if (this.nodeToSet instanceof TransientNode && !this.nodeToSet.anchor) {
    return new TransientNode(
      this.nodeToSet.scriptRunId,
      node,        // ← 原 ElementNode 变成 anchor
      this.nodeToSet.transientNodes,
      this.nodeToSet.deltaMsgReceivedAt
    )
  }

  return this.nodeToSet  // 普通替换：直接用新节点覆盖
}
```

##### visitBlockNode()：递归处理 + 插入/替换判定

```typescript
// SetNodeByDeltaPathVisitor.ts L85-L168
visitBlockNode(node: BlockNode): AppNode {
  // ── 路径为空：到达目标位置 ──
  if (this.deltaPath.length === 0) {
    // Anchor 捕获场景（同 visitElementNode）
    if (this.nodeToSet instanceof TransientNode && !this.nodeToSet.anchor) {
      return new TransientNode(
        this.nodeToSet.scriptRunId,
        node,         // ← 原 BlockNode 变成 anchor
        this.nodeToSet.transientNodes,
        ...
      )
    }
    return this.nodeToSet
  }

  // ── 路径非空：继续向下递归 ──
  const [currentIndex, ...remainingPath] = this.deltaPath

  if (currentIndex < 0 || currentIndex > node.children.length) throw ...

  const childVisitor = new SetNodeByDeltaPathVisitor(
    remainingPath, this.nodeToSet, this.scriptRunId
  )

  let newChildren: AppNode[] = []

  // 情形 A：索引越界 → 末尾追加新节点
  if (!node.children[currentIndex]) {
    if (remainingPath.length > 0) throw ...
    newChildren = node.children.slice()
    newChildren[currentIndex] = this.nodeToSet

  // 情形 B：索引合法 → 遍历 children 处理目标位置
  } else {
    let index = 0
    while (index < node.children.length) {
      const child = node.children[index]

      if (index !== currentIndex) {
        newChildren.push(child)     // 非目标：直接引用，保持不变
        index++
        continue
      }

      // 对目标 child 递归调用（可能得到 TransientNode 或其他节点）
      const nextChild = child.accept(childVisitor)

      // ══════════════════════════════════════════════════
      // ══ 插入 vs 替换 的核心判定 ══
      // ══════════════════════════════════════════════════
      // 满足以下三个条件 → 执行【插入】（而非替换）：
      //   1. 原 child 不是 TransientNode（是真实内容节点）
      //   2. 处理结果 nextChild 是 TransientNode（被包装了）
      //   3. nextChild.anchor !== child（说明是"新包装"而非"在原 Transient 上修改"）
      // 效果：新 Transient 在前，原节点在后（anchor 独立存在于 children 中）
      if (
        !(child instanceof TransientNode) &&
        nextChild instanceof TransientNode &&
        nextChild.anchor !== child
      ) {
        newChildren.push(nextChild)   // Transient 先插入（浮在上层）
        newChildren.push(child)       // 原节点也保留（作为底层内容）
      } else {
        newChildren.push(nextChild)   // 正常替换
      }
      index++
    }
  }

  // 创建新 BlockNode（不可变更新的保证）
  return new BlockNode(
    node.activeScriptHash, newChildren, node.deltaBlock,
    this.scriptRunId, node.fragmentId, node.deltaMsgReceivedAt
  )
}
```

**为什么需要"插入"分支？**
当 spinner 等临时内容被放置到某个已有元素（如 DataFrame）的位置时，如果只做替换：
- `spinner` → 替换原 `dataframe` 节点 → 原节点丢失 → spinner 消失后无法恢复

通过插入：
- `spinner(Transient)` + `dataframe(anchor)` 两个节点并存 → 渲染时 spinner 在上、dataframe 在下 → `clearTransientNodes()` 后只删除 Transient，dataframe 完整保留

##### visitTransientNode()：透明穿透 + replaceTransientNodeWithSelf

```typescript
// SetNodeByDeltaPathVisitor.ts L61-L83
visitTransientNode(node: TransientNode): AppNode {
  // 路径为空：到达目标位置，且该位置原本就是 TransientNode
  if (this.deltaPath.length === 0) {
    // 交给 replaceTransientNodeWithSelf 协议处理
    return this.nodeToSet.replaceTransientNodeWithSelf(node)
  }

  // 路径非空：穿透 anchor（TransientNode 不占用路径索引）
  if (node.anchor) {
    const newAnchor = node.anchor.accept(this)  // 递归修改 anchor
    return new TransientNode(                    // 重新包装 TransientNode
      node.scriptRunId, newAnchor,
      node.transientNodes, node.deltaMsgReceivedAt
    )
  }

  throw new Error("TransientNode has no anchor to set node at")
}
```

**路径非空时的处理**：TransientNode 在 `deltaPath` 层级上是"不存在"的，修改其内部 anchor 的路径不经过 TransientNode 本身消耗索引。修改完成后重新套回 TransientNode 外壳。

---

#### 4.6.8 replaceTransientNodeWithSelf 协议：三节点各自的合并策略

当 `nodeToSet` 要替换一个已有的 `TransientNode` 时，不直接覆盖，而是调用 `nodeToSet.replaceTransientNodeWithSelf(oldTransientNode)`，让新节点自己决定合并策略。

三种实现对照：

| nodeToSet 类型 | 实现位置 | 合并策略 |
|---------------|---------|---------|
| **ElementNode** | [ElementNode.ts L83-L117](./frontend/lib/src/render-tree/ElementNode.ts#L83-L117) | 1. scriptRunId 不同 → 整体替换<br>2. 无 transientNodes → 返回自身<br>3. ClearStaleNodeVisitor 清理 stale 临时节点<br>4. 清理后为空 → 返回自身<br>5. 否则：新 TransientNode(anchor=self, transientNodes=过滤后列表) |
| **BlockNode** | [BlockNode.ts L64-L97](./frontend/lib/src/render-tree/BlockNode.ts#L64-L97) | 逻辑同上，只是 anchor 类型为 BlockNode |
| **TransientNode** | [TransientNode.ts L84-L102](./frontend/lib/src/render-tree/TransientNode.ts#L84-L102) | 比较 `deltaMsgReceivedAt` 时间戳：<br>• 新节点较新 → 优先用新节点的 transientNodes，anchor 优先取 `this.anchor ?? old.anchor`<br>• 旧节点较新 → 返回旧节点 |

**设计目的**：同一个位置在同一次 script run 内可能收到多个 transient 更新（如 progress 条多次更新）。通过时间戳 + stale 清理的合并策略，避免每一次小更新都丢失原有的临时内容或 anchor。

---

#### 4.6.9 不可变更新的传播路径

`SetNodeByDeltaPathVisitor` 保证每次修改都会创建路径上所有的新 BlockNode。以修改 `root → BlockA → ElementY` 为例：

```
修改前（引用关系）：           修改后（新创建节点标记为 '）：

    Root                        Root'  ← 新
     │                           │
     ├─ BlockA                   ├─ BlockA'  ← 新（children 数组新）
     │    ├─ ElementX            │    ├─ ElementX  ← 同引用（未变）
     │    └─ ElementY (目标)     │    └─ ElementY'  ← 新（Arrow bytes 可能复用）
     └─ BlockB                   └─ BlockB  ← 同引用（未变）
```

- `Root'`, `BlockA'`, `ElementY'` 是新创建的 → React 浅比较检测到变化 → 重渲染
- `BlockB`, `ElementX` 保持同一引用 → React 跳过重渲染
- **Arrow payload 复用场景**：若 `ElementY'.element === ElementY.element`（引用相等），则下游组件 `useMemo(() => new Quiver(...), [element.arrowData])` 依赖项未变 → 跳过解析

---

#### 4.6.10 Delta 应用完整流程（以 newElement 路径为例）

```
delta.type = "newElement"
       │
       ▼  AppRoot.ts L248-L271
 applyDelta() - newElement 分支
       │
       ▼  GetNodeByDeltaPathVisitor.getNodeAtPath(root, deltaPath)
       │    ├─ 递归 BlockNode.children：消耗路径索引
       │    └─ TransientNode：透明穿透，不消耗索引
       │   → 返回 existingNode（可能是 ElementNode / undefined）
       │
       ▼  canReuseElementPayload(existingNode, elementHash, nextElement)
       │    ├─ elementHash 存在？
       │    ├─ existingNode 是 ElementNode？
       │    ├─ hash 完全匹配？
       │    ├─ 元素类型相同？
       │    └─ !hasOneShotEffect？
       │
       ▼  决策：
       ├─ 全满足 → elementToUse = existingNode.element  (复用旧 Arrow bytes!)
       └─ 否则   → elementToUse = nextElement           (使用新 element)
       │
       ▼  addElement(deltaPath, scriptRunId, elementToUse, ...)
       │    └─ new ElementNode(elementToUse, metadata, ..., elementHash)
       │
       ▼  SetNodeByDeltaPathVisitor.setNodeAtPath(root, deltaPath, elementNode)
       │    ┌────────────────────────────────────────────────────┐
       │    │ 递归下降：                                          │
       │    │  • visitBlockNode：创建 childVisitor 处理目标 child   │
       │    │    ├─ 若在 Block 层命中（路径空）→ 替换/捕获 anchor │
       │    │    └─ 否则 while 循环 children                       │
       │    │         ├─ 非目标 index → 直接 push（保持引用）       │
       │    │         └─ 目标 index → child.accept(childVisitor)  │
       │    │                      ├─ visitElementNode → 替换     │
       │    │                      └─ 插入/替换判定（可选）        │
       │    │  • 每退出一层 Block → 创建新 BlockNode（不可变）       │
       │    └────────────────────────────────────────────────────┘
       │   → 返回新 root BlockNode
       │
       ▼  new AppRoot(mainScriptHash, newRoot, appLogo)  ← 整体不可变
       │
       ▼  React setState → 调度重渲染
       │
       ▼  DataFrame.tsx
 useMemo(() => new Quiver(element.arrowData), [elementHash, element.arrowData])
       └─ elementHash 不变 或 element.arrowData 引用不变 → 跳过重新解析 ✅
```

---

### 4.7 前端接收链路完整时序

```
WebSocket 二进制帧 (ArrayBuffer)
       │
       ▼  WebsocketConnection.tsx L552-L560
  websocket.onmessage 事件
       │
       ▼  L699-L723
  handleMessage(data: ArrayBuffer)
       ├─ 分配 messageIndex (保证顺序)
       ├─ ForwardMsg.decode(encodedMsg) ────┐
       ├─ cache.processMessagePayload()     │
       │    ├─ maybeCacheMessage()          │
       │    │    ├─ 检查 type != "refHash"  │
       │    │    ├─ 检查 metadata.cacheable │
       │    │    └─ 存入 Map<hash, CacheEntry>
       │    └─ 如果 type == "refHash":
       │         ├─ getCachedMessage(refHash)
       │         │    └─ ForwardMsg.decode(缓存的 encodedMsg)  ←──┘
       │         ├─ 合并 metadata（refMsg 的最新 metadata）
       │         └─ 返回还原后的完整 ForwardMsg
       ├─ 存入 messageQueue[messageIndex]
       └─ while (lastDispatched+1 in queue):
             └─ onMessage(ForwardMsg) ──┐
                                         │
       ┌─────────────────────────────────┘
       ▼  App.tsx L968-L1042
  handleMessage(msgProto: ForwardMsg)
       └─ dispatchProto(msgProto, "type", {
             delta: (deltaMsg) => handleDeltaMsg(deltaMsg, metadata, hash)
          })
       │
       ▼  L1757-L1771
  handleDeltaMsg()
       └─ setState(prev => ({
             elements: prev.elements.applyDelta(scriptRunId, deltaMsg, metadata, elementHash)
          }))
       │
       ▼  AppRoot.ts L238-L301
  applyDelta() - switch (delta.type)
       ├─ "newElement" 分支：
       │    ├─ GetNodeByDeltaPathVisitor.getNodeAtPath()
       │    ├─ canReuseElementPayload() 检查
       │    │    ├─ elementHash 匹配？
       │    │    ├─ 元素类型匹配？
       │    │    └─ 非 one-shot？
       │    └─ addElement()
       │         └─ new ElementNode(复用 or 新 element)
       │              └─ SetNodeByDeltaPathVisitor.setNodeAtPath()
       │
       ├─ "addBlock" 分支：
       │    ├─ GetNodeByDeltaPathVisitor.getNodeAtPath()
       │    ├─ 同类型 Block 继承子节点
       │    └─ addBlock()
       │         └─ new BlockNode(继承的 children)
       │              └─ SetNodeByDeltaPathVisitor.setNodeAtPath()
       │
       └─ "newTransient" 分支：
            └─ addTransient()
                 └─ new TransientNode(anchor=undefined)
                      └─ SetNodeByDeltaPathVisitor.setNodeAtPath()
                           (自动设置 anchor 为原节点)
       │
       ▼
  React setState → 重新渲染
       │
       ▼  DataFrame.tsx
  useMemo(() => new Quiver(element.arrowData), [elementHash, element.arrowData])
       └─ elementHash 不变 → 跳过重新解析！
```

### 4.8 WebSocket 传输层细节

- **二进制传输**：WebSocket `binaryType = "arraybuffer"`，Protobuf 的 `bytes` 字段在 WebSocket 二进制帧中直接传输，无需 base64 编码。Arrow IPC bytes 在整个链路中保持二进制形态，直到前端 Quiver 解析。
- **认证令牌传输**：通过 `Sec-WebSocket-Protocol` 头传递（浏览器 API 不允许设置自定义 HTTP 头），见 [WebsocketConnection.tsx L531-L544](./frontend/connection/src/WebsocketConnection.tsx#L531-L544)。
- **消息大小硬限制**：Protobuf 单个消息默认上限为 2GB（C++ 实现），但 Streamlit 通过 `server.maxMessageSize` 配置项（默认 ~200MB）进行更严格的限制。超过限制时：
  - 若开启 `server.enableArrowTruncation`：自动截断表格（见 2.7 节）
  - 否则：WebSocket 层报错，连接可能断开

---

## 五、前端还原流程：Arrow Bytes → UI 渲染

### 5.1 核心解析器：parseArrowIpcBytes()

[arrowParseUtils.ts](./frontend/lib/src/dataframes/arrowParseUtils.ts#L346-L408) 是前端所有 Arrow 数据解析的统一入口：

```typescript
export function parseArrowIpcBytes(ipcBytes: Uint8Array): ParsedTable {
  // 步骤1: Arrow JS 反序列化 IPC bytes → Table
  const table = tableFromIPC(ipcBytes)

  // 步骤2: 从 Arrow schema 提取 Pandas 元数据（如果存在）
  const pandasSchema = parsePandasSchema(table)
  //   → table.schema.metadata.get("pandas") → JSON.parse
  //   → 如果 table 不来自 Pandas（如原生 PyArrow），此字段为 undefined

  // 步骤3: 提取 Dictionary 类型列的分类选项
  const categoricalOptions = parseCategoricalOptionsForColumns(table)

  // 步骤4: 解析索引列类型（需要 Pandas schema）
  const pandasIndexColumnTypes = parsePandasIndexColumnTypes(
    arrowSchema, pandasSchema, categoricalOptions
  )

  // 步骤5: 解析数据列类型
  const dataColumnTypes = parseDataColumnTypes(
    arrowSchema, pandasSchema, categoricalOptions
  )

  // 步骤6: 数据列值切片（剔除索引列）
  const data = parseData(table, dataColumnTypes)

  // 步骤7: 索引列值提取（需要 Pandas schema，Range Index 需手动生成）
  const pandasIndexData = parsePandasIndexData(table, pandasSchema)

  // 步骤8: 列名矩阵解析（支持多级表头）
  const columnNames = parseColumnNames(
    dataColumnTypes, pandasIndexColumnTypes, pandasSchema
  )

  return { pandasIndexData, data, columnNames, pandasIndexColumnTypes, dataColumnTypes }
}
```

### 5.2 Pandas Schema 元数据

Pandas 在 `pa.Table.from_pandas()` 时会向 schema metadata 注入一个 `"pandas"` key，JSON 结构如下：

```json
{
  "index_columns": ["__index_level_0__", {"kind": "range", "name": null, "start": 0, "stop": 100, "step": 1}],
  "column_indexes": [...],   // 多级表头信息
  "columns": [
    {"field_name": "col1", "name": "col1", "pandas_type": "int64", "numpy_type": "int64", "metadata": null},
    ...
  ]
}
```

**Range Index 的特殊处理**：

RangeIndex `[0, 1, 2, ..., N-1]` 不作为数据列存在于 Arrow Table 中，而是以对象形式存在于 `index_columns` 数组中。前端需手动生成：

```typescript
// arrowParseUtils.ts L104-L108
if (isPandasRangeIndex(indexCol)) {
  const { start, stop, step } = indexCol
  return range(start, stop, step)  // lodash.range 生成数字序列
}
```

### 5.3 多级列名解析：parseHeaderName()

多级列名在 Arrow schema field name 中被序列化为 Python tuple 的字符串表示：
```
"('Level1', 'Level2')"   ← Pandas MultiIndex columns
```

前端解析逻辑 [arrowParseUtils.ts#L134-L146](./frontend/lib/src/dataframes/arrowParseUtils.ts#L134-L146)：
```
name = "('1','foo (bar)')"
  → trim()
  → replace(/^\(/, "[")   → "['1','foo (bar)')"
  → replace(/\)$/, "]")   → "['1','foo (bar)']"
  → replace(/'/g, '"')    → '["1","foo (bar)"]'
  → JSON.parse()          → ["1", "foo (bar)"]
```

### 5.4 Quiver：前端数据容器

[Quiver.ts](./frontend/lib/src/dataframes/Quiver.ts) 将 Arrow 的列式存储转换为前端易用的行列访问接口：

```typescript
class Quiver {
  private readonly _columnNames: string[][]        // 列名矩阵（多级表头）
  private readonly _pandasIndexColumnTypes: ArrowType[]   // 索引列类型
  private readonly _dataColumnTypes: ArrowType[]          // 数据列类型
  private readonly _columnTypes: ArrowType[]              // 全部列类型（拼接）
  private readonly _pandasIndexData: IndexData            // 索引列值
  private readonly _data: Table                           // 数据列值（Arrow Table）
  private readonly _styler?: PandasStylerData             // Styler 信息
  private readonly _num_bytes: number                     // 原始字节数（用于 hash）

  // 核心访问方法：按 (行, 列) 坐标取单元格
  public getCell(rowIndex: number, columnIndex: number): DataFrameCell {
    // columnIndex < numIndexColumns → 从 _pandasIndexData 取
    // 否则 → 从 _data (Arrow Table) 的对应列取
  }
}
```

**Quiver 是不可变对象**，不能被 Immer 的 `produce()` 修改。

**Styler 递归解析**：
```typescript
// Quiver.ts L292-L303
function parseStyler(pandasStyler: ArrowData.PandasStyler): PandasStylerData {
  return {
    uuid, caption, cssStyles,
    cssId: `T_${uuid}`,
    // display_values 本身也是 Arrow IPC bytes → 递归创建 Quiver
    displayValues: new Quiver({ data: pandasStyler.displayValues }),
  }
}
```

### 5.5 组件渲染链路

#### 路径 A：交互式 DataFrame（glide-data-grid）

[DataFrame.tsx](./frontend/lib/src/components/widgets/DataFrame/DataFrame.tsx#L80-L120)：
```
Dataframe Proto
  └─ useMemo(() => new Quiver(element.arrowData), [elementHash, element.arrowData])
       └─ Quiver → useColumnLoader() → Glide GridColumn[]
            └─ Quiver → useDataLoader() → Glide getCellContent()
                 └─ <GlideDataEditor columns={...} getCellContent={...} />
```

`elementHash` 是 memoization 的关键锚点，避免 payload 未变化时重复解析几 MB 的 Arrow 数据。

#### 路径 B：静态 Table（HTML Table）

[Table.tsx](./frontend/lib/src/components/elements/Table/Table.tsx#L80-L99)：
```
Table Proto
  └─ useMemo(() => new Quiver(element.arrowData), [elementHash, element.arrowData])
       └─ Quiver.dimensions 计算行列范围
            └─ Quiver.getCell(r, c) 逐个取 cell 内容
                 └─ formatArrowCell(cell) 格式化文本
                      └─ <table> DOM 渲染
```

#### 路径 C：双向组件 MixedData 还原

[reconstructMixedData.ts](./frontend/lib/src/components/widgets/BidiComponent/utils/reconstructMixedData.ts#L42-L92)：

```typescript
// 前端还原：遍历 JSON dict 第一层，将 {__arrow_ref__: refId} 占位符替换为实际 Arrow Table
reconstructMixedData(jsonData, arrowBlobs):
  if data is {__arrow_ref__: refId}:
    return tableFromIPC(arrowBlobs[refId])   // 顶层直接是 DataFrame
  else if data is object:
    for each [key, value] in data:
      if value is {__arrow_ref__: refId}:
        result[key] = tableFromIPC(arrowBlobs[refId])  // 属性是 DataFrame
      else:
        result[key] = value  // 非 DataFrame 原值保留
  return result
```

---

## 六、跨场景对比总结

| 场景 | 序列化入口 | Proto 容器 | 前端解析器 | 典型使用 |
|------|-----------|-----------|-----------|---------|
| `st.dataframe` | `convert_pandas_df_to_arrow_bytes()` | `Dataframe.arrow_data.data` | `Quiver` → Glide Grid | 交互式表格 |
| `st.table` | `marshall(ArrowDataProto, ...)` | `Table.arrow_data.data` | `Quiver` → DOM `<table>` | 静态打印 |
| `st.line_chart` 等 | 内部同上 | `ArrowNamedDataSet.data` | ArrowVegaLiteChart → Vega | 图表可视化 |
| 自定义组件 v1 | `component_arrow.marshall()` | `ArrowTable{data,index,columns}` | 前端分别反序列化后重组 | `components.v1.html()` |
| 自定义组件 v2 | `serialize_mixed_data()` | `MixedData{json, arrow_blobs}` | `reconstructMixedData()` | `components.v2.bidi()` |

---

## 七、关键设计决策与技术债

1. **Arrow IPC Stream vs File**：选择 Stream 格式（无 magic/footer）是正确的短连接/流式传输选择。

2. **Pandas metadata 耦合**：索引信息依赖 Pandas 注入的 schema metadata。原生 PyArrow Table（不含 Pandas）会丢失索引列概念——这是有意为之，还是需要完善的边界行为？

3. **组件 v1 vs 内部的 Arrow 双轨**：Components.proto 的 ArrowTable 设计（data/index/columns 分离）早于内部 ArrowData 设计，两者不统一是历史遗留，增加了维护成本。

4. **Large 类型降级**：为兼容第三方旧版 Arrow JS 而在后端强制降级，本质是向后兼容的技术债。未来要求自定义组件升级 Arrow 版本后可移除。

5. **ForwardMsgCache + Arrow 字节的协同**：这是大数据量 rerun 性能的关键优化。相同数据跨 run 不重复传输，是用户感知"快"的重要原因。

6. **缓存存储原始二进制而非解码对象**：ForwardMessageCache 存储 `Uint8Array` 而非 `ForwardMsg` 对象，节省内存且保证精确性。

7. **双重 memoization 防线**：ForwardMsgCache（传输层）+ canReuseElementPayload（渲染层）+ 组件 useMemo（渲染层）共同确保 Arrow 数据在 rerun 时不被重复传输、重复创建、重复解析。

8. **基于 scriptRunCount 的 Age 缓存过期**：不同于标准 LRU，用脚本运行次数作为时间维度，更符合 Streamlit rerun 的使用模式。

9. **Visitor 模式实现不可变树**：GetNodeByDeltaPathVisitor 和 SetNodeByDeltaPathVisitor 通过双重分派实现类型安全的树操作，配合完整不可变性确保 React 渲染性能。

10. **TransientNode 透明穿透**：临时节点在 delta_path 查找和修改时不消耗路径索引，确保 delta_path 只描述真实内容结构，不受临时元素（如 spinner）干扰。

11. **TransientNode 的 anchor 自动捕获**：当无 anchor 的 TransientNode 替换普通节点时，自动将原节点设为 anchor，确保临时内容消失后原始内容可完整恢复。

12. **Block 子节点继承**：同类型 Block 替换时继承所有子节点，保留 Widget State 和 React State，这是对话框、列布局等容器 rerun 时状态不丢失的关键。
