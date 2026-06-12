# Streamlit 主题样式机制完整传递链路

## 一、总体架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 配置值层 (Config Values)                                                  │
│  config.toml → [theme] [theme.light] [theme.dark] [theme.sidebar] ...        │
│  Python config 系统注册与加载                                                  │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. 后端处理层 (Backend Processing)                                           │
│  _populate_theme_msg() → parse_fonts_with_source() → CustomThemeConfig Proto │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. 数据传输层 (Data Transport)                                               │
│  NewSession.custom_theme → ForwardMsg → WebSocket → 前端                     │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 前端解析层 (Frontend Parsing)                                             │
│  handleMessage → handleNewSession → processThemeInput → createCustomThemes   │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 主题构建层 (Theme Construction)                                           │
│  createTheme → createEmotionTheme → createEmotionColors + createShadows      │
│  + createBaseUiTheme (BaseWeb 桥接)                                           │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  6. 样式注入层 (Style Injection)                                              │
│  RootStyleProvider → BaseProvider + CacheProvider + EmotionThemeProvider     │
│  + Global(globalStyles) → 渲染到 <style> 标签                                 │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  7. 页面渲染层 (Page Rendering)                                               │
│  styled-components → props.theme.colors / spacing / radii ...                │
│  + CSS 自定义属性 (--st-*) 给 component-v2-lib                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、第1层：配置值层 — 主题配置定义与加载

### 2.1 配置注册机制

**核心文件**：[config.py](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/lib/streamlit/config.py)

主题配置通过 `_create_theme_options()` 函数批量注册，支持 6 个配置分类：

```python
# config.py L110-L117
class CustomThemeCategories(str, Enum):
    SIDEBAR = "sidebar"              # [theme.sidebar]
    LIGHT = "light"                  # [theme.light]
    DARK = "dark"                    # [theme.dark]
    LIGHT_SIDEBAR = "light.sidebar"  # [theme.light.sidebar]
    DARK_SIDEBAR = "dark.sidebar"    # [theme.dark.sidebar]
```

每个主题选项（如 `primaryColor`、`backgroundColor`）会同时注册到多个分类下，形成继承体系。

### 2.2 可配置的主题选项清单

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `base` | str | 继承的基础主题: `"light"` / `"dark"` / 本地TOML路径 / URL |
| `primaryColor` | str | 主色调（按钮、进度条等） |
| `backgroundColor` | str | 页面背景色 |
| `secondaryBackgroundColor` | str | 组件背景色（输入框、表格等） |
| `textColor` | str | 正文文字颜色 |
| `font` / `bodyFont` | str | 正文字体，支持 `字体名:URL` 格式 |
| `codeFont` | str | 代码字体 |
| `headingFont` | str | 标题字体 |
| `baseFontSize` | int | 基础字号（像素） |
| `baseFontWeight` | int | 基础字重 100-600 |
| `codeFontSize` | str | 代码字号 |
| `codeFontWeight` | int | 代码字重 |
| `headingFontSizes` | list[str] | h1-h6 标题字号数组（1-6项） |
| `headingFontWeights` | list[int] | h1-h6 标题字重数组（1-6项） |
| `baseRadius` | str | 组件圆角: `"none"/"small"/"medium"/"large"/"full"` 或像素值 |
| `buttonRadius` | str | 按钮圆角（同上） |
| `borderColor` | str | 组件边框颜色 |
| `linkColor` | str | 超链接颜色 |
| `linkUnderline` | bool | 超链接是否带下划线 |
| `showWidgetBorder` | bool | 是否显示组件边框 |
| `showSidebarBorder` | bool | 是否显示侧边栏分隔线 |
| `codeTextColor` | str | 代码文字颜色 |
| `codeBackgroundColor` | str | 代码块背景色 |
| `dataframeBorderColor` | str | 数据表边框颜色 |
| `dataframeHeaderBackgroundColor` | str | 数据表表头背景色 |
| `redColor` / `orangeColor` / `yellowColor` / `blueColor` / `greenColor` / `violetColor` / `grayColor` | str | 7 种语义色的主色 |
| `redBackgroundColor` / `...BackgroundColor` | str | 7 种语义色的背景色（可从主色自动派生） |
| `redTextColor` / `...TextColor` | str | 7 种语义色的文字色（可从主色自动派生） |
| `chartCategoricalColors` | list[str] | 图表分类调色板 |
| `chartSequentialColors` | list[str] | 图表连续调色板（10项） |
| `chartDivergingColors` | list[str] | 图表发散调色板（10项） |
| `metricValueFontSize` | str | st.metric 数值字号 |
| `metricValueFontWeight` | int | st.metric 数值字重 |
| `fontFaces` | list[dict] | @font-face 自定义字体声明 |

### 2.3 配置继承关系

```
[theme]                         ← 通用基础配置
  ├─ [theme.light]              ← 浅色模式覆盖（继承 [theme]）
  │     └─ [theme.light.sidebar]  ← 浅色侧边栏覆盖（继承 [theme.sidebar] + [theme.light]）
  ├─ [theme.dark]               ← 深色模式覆盖（继承 [theme]）
  │     └─ [theme.dark.sidebar]   ← 深色侧边栏覆盖（继承 [theme.sidebar] + [theme.dark]）
  └─ [theme.sidebar]            ← 侧边栏通用覆盖（继承 [theme]）
```

前端继承逻辑实现在 [utils.ts L1410-L1448](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/utils.ts#L1410-L1448) 的 `handleSectionInheritance()`。

---

## 三、第2层：后端处理层 — 配置转 Protobuf

### 3.1 入口：NewSession 消息构建

**核心文件**：[app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/lib/streamlit/runtime/app_session.py)

```python
# app_session.py L806-L833
# 6 次调用分别填充 6 个配置层级
_populate_theme_msg(msg.new_session.custom_theme)                        # [theme]
_populate_theme_msg(msg.new_session.custom_theme.light, "theme.light")    # [theme.light]
_populate_theme_msg(msg.new_session.custom_theme.dark, "theme.dark")      # [theme.dark]
_populate_theme_msg(msg.new_session.custom_theme.sidebar, "theme.sidebar")# [theme.sidebar]
_populate_theme_msg(msg.new_session.custom_theme.light.sidebar, ...)     # [theme.light.sidebar]
_populate_theme_msg(msg.new_session.custom_theme.dark.sidebar, ...)      # [theme.dark.sidebar]
```

### 3.2 _populate_theme_msg 核心流程

**位置**：[app_session.py L1149-L1320](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/lib/streamlit/runtime/app_session.py#L1149-L1320)

处理步骤：

1. **读取配置字典**：`theme_opts = config.get_options_for_section(section)`
2. **遍历赋值**：将每个配置选项映射到 `CustomThemeConfig` protobuf 字段
3. **特殊处理**：
   - `base` → 枚举值映射 `LIGHT=0 / DARK=1`
   - `font` / `codeFont` / `headingFont` → 通过 `parse_fonts_with_source()` 解析
   - `headingFontSizes` / `headingFontWeights` → JSON 解析并验证长度
   - `chartCategoricalColors` / `chartSequentialColors` / `chartDivergingColors` → 数组验证
   - `fontFaces` → JSON 解析为 `FontFace` 对象列表

### 3.3 字体解析：parse_fonts_with_source()

**核心文件**：[theme_util.py](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/lib/streamlit/runtime/theme_util.py)

支持 `字体名:URL` 格式：

```python
# 示例 config.toml:
# font = "Inter:https://fonts.googleapis.com/css2?family=Inter&display=swap"

# 解析结果:
#   msg.body_font = "Inter"
#   msg.font_sources += { config_name: "font", source_url: "https://..." }
```

侧边栏的字体源会自动附加 `-sidebar` 后缀：`config_name = "font-sidebar"`。

---

## 四、第3层：数据传输层 — WebSocket Protobuf

### 4.1 消息结构

**核心文件**：[NewSession.proto](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/proto/streamlit/proto/NewSession.proto)

```protobuf
// L27-L66 NewSession 消息
message NewSession {
  Initialize initialize = 1;
  string script_run_id = 2;
  Config config = 6;
  CustomThemeConfig custom_theme = 7;   // ← 主题配置
  repeated AppPage app_pages = 8;
  string page_script_hash = 9;
  // ...
}

// L134-L209 CustomThemeConfig（完整字段清单见 proto 文件）
message CustomThemeConfig {
  enum BaseTheme { LIGHT = 0; DARK = 1; }
  string primary_color = 1;
  string background_color = 3;
  string text_color = 4;
  // ... 60+ 个字段
  CustomThemeConfig sidebar = 24;  // 嵌套结构
  CustomThemeConfig light = 60;
  CustomThemeConfig dark = 61;
}
```

### 4.2 传输流程

```
Python 后端:
  AppSession._create_new_session_message()
    → 构造 msg.new_session.custom_theme
    → 包装为 ForwardMsg(type = NEW_SESSION)
    → 通过 WebSocket 发送二进制 protobuf

前端 (connection 模块):
  ConnectionManager.onMessage
    → 反序列化为 ForwardMsg 对象
    → 回调 App.handleMessage()
```

---

## 五、第4层：前端解析层 — 接收并初始化主题

### 5.1 消息分发入口

**核心文件**：[App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/app/src/App.tsx)

```typescript
// App.tsx L968-L1036 handleMessage
dispatchProto(msgProto, "type", {
  newSession: (newSessionMsg: NewSession) =>
    this.handleNewSession(newSessionMsg),
  // ...
})
```

### 5.2 handleNewSession 主题处理

**位置**：[App.tsx L1349-L1442](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/app/src/App.tsx#L1349-L1442)

```typescript
const themeInput = newSessionProto.customTheme as CustomThemeConfig
this.processThemeInput(themeInput)
```

### 5.3 processThemeInput 核心逻辑

**位置**：[App.tsx L1522-L1587](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/app/src/App.tsx#L1522-L1587)

处理步骤：

1. **哈希去重**：`createThemeHash()` 对主题配置排序后哈希，避免重复处理
2. **创建自定义主题**：调用 `createCustomThemes(themeInput)`
   - 如果配置了 `[theme.light]` 或 `[theme.dark]` 段 → 返回 3 个主题：`Custom Theme Light`、`Custom Theme Dark`、`Custom Theme Auto`
   - 否则 → 返回 1 个主题：`Custom Theme`
3. **注册主题**：`theme.addThemes(customThemes, { keepPresetThemes: false })`
4. **应用主题**：优先匹配用户缓存偏好（Light/Dark），否则使用 Auto（跟随系统）
5. **字体加载**：调用 `theme.setFonts(themeInput)` 设置 `fontFaces` 和 `fontSources`

### 5.4 useThemeManager Hook

**核心文件**：[useThemeManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/app/src/util/useThemeManager.ts)

状态管理：

| 状态 | 类型 | 说明 |
|------|------|------|
| `theme` | `ThemeConfig` | 当前激活的主题 |
| `availableThemes` | `ThemeConfig[]` | 可选主题列表（预设 + 自定义） |
| `fontFaces` | `IFontFace[]` | 需声明的 @font-face |
| `fontSources` | `Record<string,string>` | 需加载的 `<link>` 字体源 URL |

---

## 六、第5层：主题构建层 — Emotion + BaseWeb 双主题系统

### 6.1 ThemeConfig 数据结构

**核心文件**：[types.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/types.ts)

```typescript
// types.ts L109-L121
export type ThemeConfig = {
  name: string                      // "Light" / "Dark" / "Custom Theme" 等
  displayName?: string              // 菜单中显示的名称
  emotion: EmotionTheme             // Emotion 样式系统用的主题
  basewebTheme: BaseWebTheme        // BaseWeb 组件库用的主题
  primitives: ThemePrimitives       // BaseWeb 原始色板
  themeInput?: Partial<CustomThemeConfig>  // 原始配置输入（用于重建）
}
```

### 6.2 createTheme 构建流程

**核心文件**：[utils.ts L1118-L1170](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/utils.ts#L1118-L1170)

```typescript
export const createTheme = (themeName, themeInput, baseThemeConfig?, inSidebar?) => {
  // 1. 补全默认值: 把 themeInput 与 base 主题的默认值合并
  completedThemeInput = completeThemeInput(themeInput, baseThemeConfig)

  // 2. 选择起始主题: 根据 backgroundColor 亮度智能选择 light/dark 基底
  startingTheme = merge(cloneDark/lightTheme, { emotion: { inSidebar } })

  // 3. 构建 Emotion 主题（核心）
  emotion = createEmotionTheme(completedThemeInput, startingTheme)

  // 4. 构建 BaseWeb 主题（给 BaseWeb 组件用）
  basewebTheme = cloneDeep(createBaseUiTheme(emotion, startingTheme.primitives))

  return { name, emotion, basewebTheme, themeInput, ...startingTheme }
}
```

### 6.3 createEmotionTheme 详细步骤

**位置**：[utils.ts L685-L1065](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/utils.ts#L685-L1065)

#### 步骤 A：颜色解析与验证

```typescript
// 1. 解析所有颜色配置（自动补全 # 前缀、验证有效性）
parsedColors = Object.entries(customColors).reduce((acc, [key, color]) => {
  validatedColor = parseColor(color, key)  // 尝试原样，再加 #
  if (validatedColor) acc[key] = validatedColor
}, {})
```

#### 步骤 B：GenericColors 构建

GenericColors = PrimitiveColors + RequiredThemeColors + OptionalThemeColors

```typescript
newGenericColors = {
  ...colors,                           // 基础调色板（所有 gray/blue/red 渐变）
  primary: primary ?? colors.primary,  // 用户配置覆盖
  bodyText: bodyText ?? colors.bodyText,
  bgColor: bgColor ?? colors.bgColor,
  secondaryBg: secondaryBg ?? colors.secondaryBg,
  redColor/orangeColor/... : ... ,
  secondary: primary ?? colors.primary,  // secondary 跟随 primary
}
```

**PrimitiveColors 来源**：[primitives/colors.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/primitives/colors.ts)
- 含完整色阶：`gray10` ~ `gray100`、`red10` ~ `red100`、`blue10` ~ `blue100` 等

**RequiredThemeColors 来源**：
- 浅色：[emotionBaseTheme/themeColors.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/emotionBaseTheme/themeColors.ts)
- 深色：[emotionDarkTheme/themeColors.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/emotionDarkTheme/themeColors.ts)

#### 步骤 C：EmotionThemeColors 计算

**核心文件**：[getColors.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/getColors.ts)

```typescript
// getColors.ts L73-L102
EmotionThemeColors = {
  ...GenericColors,
  ...computeDerivedColors(GenericColors),  // ← 派生颜色
  link: blueTextColor,
  codeTextColor: greenTextColor,
  codeBackgroundColor: bgMix,
  borderColor: fadedText10,
  borderColorLight: fadedText05,
  dataframeBorderColor: fadedText05,
  dataframeHeaderBackgroundColor: bgMix,
  headingColor: bodyText,
  chartCategoricalColors: [...],
  chartSequentialColors: [...],
  chartDivergingColors: [...],
}
```

**computeDerivedColors 计算逻辑**（[getColors.ts L27-L63](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/getColors.ts#L27-L63)）：

```typescript
fadedText05 = transparentize(bodyText, 0.9)  // 最浅描边
fadedText10 = transparentize(bodyText, 0.8)  // 细边框
fadedText20 = transparentize(bodyText, 0.7)
fadedText40 = transparentize(bodyText, 0.6)  // 禁用态文字
fadedText60 = transparentize(bodyText, 0.4)  // 次要文字

bgMix = mix(bgColor, secondaryBg, 0.5)      // 代码块背景
darkenedBgMix100 = darken/lighten(bgMix)     // 图标颜色
darkenedBgMix25  = transparentize(darkenedBgMix100, 0.75) // 边框
lightenedBg05 = lighten(bgColor, 0.025)     // 按钮/复选框背景
```

#### 步骤 D：语义色自动派生

如果用户配置了 `redColor` 但没配置 `redBackgroundColor`，则自动派生：
- 浅色主题：`transparentize(redColor, 0.9)`
- 深色主题：`transparentize(redColor, 0.8)`

同理处理文本颜色：
- 浅色主题：`darken(redColor, 0.15)`
- 深色主题：`lighten(redColor, 0.15)`

**位置**：[utils.ts L298-L410](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/utils.ts#L298-L410) 的 `setBackgroundColors()` 和 `setTextColors()`

#### 步骤 E：圆角、字号、字重处理

| 配置项 | 处理函数 | 说明 |
|--------|----------|------|
| `baseRadius` | `parseRadius()` | none→0 / small→0.35rem / medium→0.5rem / large→1rem / full→1.4rem |
| `buttonRadius` | `parseRadius()` | 覆盖按钮专用圆角 |
| `baseFontSize` | 直接赋值 | px 单位，用于 html { font-size } |
| `codeFontSize` | `parseFontSize()` | 验证 rem/px |
| `headingFontSizes` | `convertHeadingFontSizeToRem()` | px→rem 转换 |
| `baseFontWeight` | `setFontWeights()` | normal=N, semiBold=N+100, bold=N+200, extrabold=N+300 |

#### 步骤 F：字体族解析

```typescript
// utils.ts L179-L212 parseFont()
parseFont("Inter")
  → "Inter, \"Source Sans\", sans-serif"  // 自动附加兜底字体

parseFont("sans-serif")
  → "\"Source Sans\", sans-serif"         // 别名映射
```

#### 步骤 G：阴影生成

**核心文件**：[getShadows.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/getShadows.ts)

根据 `bgColor` 亮度自动判断 light/dark，生成：
- `xs` / `sm` / `md` / `lg` / `xl` 五级阴影
- `widgetShadow` 组件阴影
- `sidebarShadow` 侧边栏阴影

### 6.4 createBaseUiTheme — BaseWeb 桥接

**核心文件**：[createBaseUiTheme.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/createBaseUiTheme.ts)

Streamlit 使用 **BaseWeb** 组件库（checkbox、select、datepicker 等），需要将 Emotion 主题映射为 BaseWeb 的主题格式：

```typescript
// Primitives 映射
primitives = {
  primaryFontFamily:  emotion.genericFonts.bodyFont,
  primary100-700:     emotion.colors.primary,  // 全色阶统一为 primary
  mono100:            emotion.colors.bgColor,
  mono200:            emotion.colors.secondaryBg,
  mono300-1000:       对应 gray 色阶,
}

// Overrides 映射
overrides = {
  borders: { radius100-500: emotion.radii.default, ... },
  typography: { font100-font600: widgetFontStyles, ... },
  colors: {
    backgroundPrimary: emotion.colors.bgColor,
    backgroundSecondary: emotion.colors.secondaryBg,
    contentPrimary: emotion.colors.bodyText,
    inputFill: widgetBackgroundColor,
    borderOpaque: emotion.colors.darkenedBgMix25,
    // ... 50+ 项 BaseWeb 专用颜色映射
  }
}
```

---

## 七、第6层：样式注入层 — React Provider 体系

### 7.1 RootStyleProvider 组件树

**核心文件**：[RootStyleProvider.tsx](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/RootStyleProvider.tsx)

```
<RootStyleProvider theme={activeTheme}>
  │
  ├─ <BaseProvider theme={theme.basewebTheme} zIndex={popup}>
  │   └─ 给 BaseWeb 组件（checkbox/select/datepicker...）提供主题
  │
  └─ <CacheProvider value={cache}>    // Emotion 样式缓存（nonce 支持 CSP）
      └─ <EmotionThemeProvider theme={theme.emotion}>
          │
          ├─ <Global styles={globalStyles} />
          │   └─ 注入全局 CSS（html/body/滚动条/基础排版）
          │
          └─ {children}
              └─ 所有 styled-components 可访问 props.theme
```

### 7.2 全局样式 globalStyles

**核心文件**：[globalStyles.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/globalStyles.ts)

注入的关键样式：

```css
html { font-size: ${baseFontSize}px; }        /* 动态根字号 */

body {
  margin: 0;
  font-family: ${genericFonts.bodyFont};
  font-weight: ${fontWeights.normal};
  color: ${colors.bodyText};
  background-color: ${colors.bgColor};          /* 页面背景 */
  -webkit-font-smoothing: auto;
}

p, ol, ul, dl {
  margin: 0 0 1rem 0;
  font-size: 1rem;
  font-weight: ${fontWeights.normal};
}

/* 滚动条样式 */
::-webkit-scrollbar { width: 6px; height: 6px; }
:hover::-webkit-scrollbar-thumb { background: ${colors.fadedText40}; }
```

### 7.3 字体源注入

在 [ThemedApp.tsx](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/app/src/ThemedApp.tsx) 中：

```tsx
<RootStyleProvider theme={activeTheme}>
  {fontFaces.length > 0 && <FontFaceDeclaration fontFaces={fontFaces} />}
  {fontSources && <FontSources fontSources={fontSources} />}
  <AppWithScreencast />
</RootStyleProvider>
```

- **FontSources**：生成 `<link rel="stylesheet" href="...">`（Google Fonts 等）
- **FontFaceDeclaration**：生成 `<style>@font-face { ... }</style>`（自定义字体文件）

---

## 八、第7层：页面渲染层 — 组件消费主题

### 8.1 styled-components 消费方式

**示例文件**：[styled-components.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/styled-components.ts)

```typescript
export const StyledErrorMessage = styled.small(({ theme }) => ({
  color: theme.colors.redTextColor,
  fontSize: theme.fontSizes.sm,
  marginTop: theme.spacing.twoXS,
}))
```

组件中可用的 `theme` 属性（完整 `EmotionTheme` 结构）：

| 属性分类 | 示例字段 |
|----------|----------|
| **colors** | `bgColor`, `bodyText`, `primary`, `secondaryBg`, `borderColor`, `fadedText40`, `blueTextColor`, `greenBackgroundColor`, `chartCategoricalColors[]`, ... |
| **spacing** | `none`, `twoXS`, `xs`, `sm`, `md`, `lg`, `xl`, `twoXL`, `threeXL`, `fourXL` |
| **sizes** | `full`, `headerHeight`, `sidebar`, `contentMaxWidth`, ... |
| **radii** | `sm`, `default`, `md2`, `xl`, `xxl`, `button`, `full`, `maxCheckbox` |
| **shadows** | `xs`, `sm`, `md`, `lg`, `xl`, `widgetShadow`, `sidebarShadow` |
| **fontSizes** | `baseFontSize`, `sm`, `md`, `lg`, `xl`, `h1FontSize`~`h6FontSize`, `codeFontSize` |
| **fontWeights** | `normal`, `semiBold`, `bold`, `extrabold`, `code`, `h1FontWeight`~`h6FontWeight` |
| **genericFonts** | `bodyFont`, `codeFont`, `headingFont`, `iconFont` |
| **lineHeights** | `none`, `tight`, `base`, `headings`, `inputWidget` |
| **iconSizes** | `xs`, `sm`, `md`, `lg`, `xl` |
| **zIndices** | `hide`, `base`, `sidebar`, `popup`, `modals`, `tablePortal` |
| **breakpoints** | `sm: 640`, `md: 768`, `lg: 992`, `xl: 1200` |
| **opacities** | `default`, `disabled`, `quiet` |
| **flags** | `inSidebar`, `showSidebarBorder`, `linkUnderline` |

### 8.2 自定义组件 (component-v2-lib) CSS 变量

**核心文件**：[component-v2-lib/src/theme.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/component-v2-lib/src/theme.ts)

主题属性映射为 `--st-*` CSS 自定义属性，供外部自定义组件使用：

```typescript
// CSS 变量命名规则: --st- + kebab-case(属性名)
--st-primary-color: #ff4b4b;
--st-background-color: #ffffff;
--st-text-color: #1a1a1a;
--st-base-radius: 0.5rem;
--st-heading-font-size-1: 2.75rem;
--st-red-color: #ff4b4b;
--st-red-background-color: rgba(255, 75, 75, 0.1);
/* ... 80+ 个 CSS 变量 */
```

### 8.3 侧边栏独立主题

侧边栏通过 `createSidebarTheme()` 从主主题派生：

**位置**：[utils.ts L1562-end](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/utils.ts#L1562)

```typescript
// 默认: 侧边栏背景 = main.secondaryBg
// 但如果 [theme.sidebar] 或 [theme.light.sidebar] 有配置，则应用覆盖
// 背景色、文字色、字体、圆角、所有语义色均可独立配置
```

渲染时在侧边栏容器外再包一层 `RootStyleProvider`（使用 sidebarTheme），实现内外主题隔离。

---

## 九、关键文件速查表

| 层级 | 文件路径 | 关键函数/类 |
|------|----------|-------------|
| 配置定义 | [config.py](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/lib/streamlit/config.py) | `_create_theme_options()`, `_create_section()` |
| 后端处理 | [app_session.py](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/lib/streamlit/runtime/app_session.py) | `_populate_theme_msg()`, `_create_new_session_message()` |
| 后端字体 | [theme_util.py](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/lib/streamlit/runtime/theme_util.py) | `parse_fonts_with_source()`, `_parse_font_config()` |
| Protobuf 定义 | [NewSession.proto](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/proto/streamlit/proto/NewSession.proto) | `message CustomThemeConfig` (L134-L209) |
| 前端入口 | [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/app/src/App.tsx) | `handleNewSession()`, `processThemeInput()` |
| 主题状态 | [useThemeManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/app/src/util/useThemeManager.ts) | `useThemeManager()`, `ThemeManager` 接口 |
| 预设主题 | [themeConfigs.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/themeConfigs.ts) | `baseTheme`, `lightTheme`, `darkTheme`, `customTheme` |
| 主题构建 | [utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/utils.ts) | `createTheme()`, `createEmotionTheme()`, `createCustomThemes()`, `handleSectionInheritance()` |
| 颜色计算 | [getColors.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/getColors.ts) | `createEmotionColors()`, `computeDerivedColors()` |
| 阴影计算 | [getShadows.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/getShadows.ts) | `createShadows()` |
| BaseWeb 桥接 | [createBaseUiTheme.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/createBaseUiTheme.ts) | `createBaseUiTheme()`, `createBaseUiThemePrimitives()`, `createBaseUiThemeOverrides()` |
| 类型定义 | [types.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/types.ts) | `EmotionTheme`, `EmotionThemeColors`, `ThemeConfig`, `DerivedColors` |
| 样式注入 | [RootStyleProvider.tsx](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/RootStyleProvider.tsx) | `RootStyleProvider` 组件 |
| 全局样式 | [globalStyles.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/globalStyles.ts) | `globalStyles` css 模板 |
| 浅色基底色 | [emotionBaseTheme/themeColors.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/emotionBaseTheme/themeColors.ts) | `requiredThemeColors` |
| 深色基底色 | [emotionDarkTheme/themeColors.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/emotionDarkTheme/themeColors.ts) | 深色模式 requiredThemeColors |
| 原语（基础） | [primitives/typography.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/lib/src/theme/primitives/typography.ts) | `fonts`, `fontSizes`, `fontWeights`, `lineHeights` |
| CSS 变量导出 | [component-v2-lib/src/theme.ts](file:///d:/fz/0601/solo-dogfeeding/code/224-streamlit/frontend/component-v2-lib/src/theme.ts) | `StreamlitTheme` 接口定义 |

---

## 十、完整调用链时序图

```
用户修改 config.toml  [theme] primaryColor = "#123456"
        │
        ▼
Python 启动 / 热重载
  config.get_options_for_section("theme")
  → { primaryColor: "#123456", backgroundColor: null, ... }
        │
        ▼
  AppSession._create_new_session_message()
    → _populate_theme_msg() 遍历赋值
    → new CustomThemeConfig({ primaryColor: "#123456", ... })
    → 嵌套赋值: .light / .dark / .sidebar / .light.sidebar / .dark.sidebar
        │
        ▼
  ForwardMsg(type=NEW_SESSION).SerializeToString()
    → WebSocket 发送二进制数据 ─────────────────────────┐
                                                         │
前端接收 ◄───────────────────────────────────────────────┘
  ConnectionManager.onMessage()
    → ForwardMsg.deserializeBinary()
        │
        ▼
  App.handleMessage()
    → dispatchProto → "newSession" → handleNewSession()
        │
        ▼
  App.processThemeInput(themeInput)
    ├─ createThemeHash() → 对比 state.themeHash，未变更则跳过
    └─ createCustomThemes(themeInput)
        ├─ hasLightConfigs || hasDarkConfigs ?
        │   ├─ YES: 创建 3 个主题 (Light/Dark/Auto)
        │   │   handleSectionInheritance(themeInput, "light")
        │   │   handleSectionInheritance(themeInput, "dark")
        │   └─ NO: 创建 1 个主题 ("Custom Theme")
        │
        └─ 对每个主题调用 createTheme(name, themeInput)
            ├─ 选择 startingTheme (light/dark 基于 bgColor 亮度)
            ├─ ▶ createEmotionTheme(themeInput, startingTheme)
            │   ├─ parseColor() 验证所有颜色
            │   ├─ 构建 GenericColors（用户值 → 默认值 fallback）
            │   ├─ ▶ createEmotionColors()
            │   │   └─ ▶ computeDerivedColors()
            │   │       └─ 计算 fadedText*, bgMix, darkenedBgMix*, lightenedBg*
            │   ├─ setBackgroundColors() → 语义背景色派生
            │   ├─ setTextColors() → 语义文字色派生
            │   ├─ parseRadius() → 圆角解析
            │   ├─ parseFont/parseFontSize/setFontWeights() → 字体处理
            │   ├─ setHeadingFontSizes / setHeadingFontWeights
            │   ├─ validateChartColors()
            │   └─ ▶ createShadows() → 生成 5 级阴影
            │
            └─ ▶ createBaseUiTheme(emotion, primitives)
                ├─ createBaseUiThemePrimitives() → Emotion→BaseWeb 色板映射
                └─ createBaseUiThemeOverrides() → 颜色/字体/圆角/边框 详细映射
        │
        ▼
  themeManager.addThemes(customThemes, { keepPresetThemes: false })
  themeManager.setTheme(mappedTheme) → 触发 React state 更新
        │
        ▼
  ThemedApp 重新渲染
    ├─ useThemeManager() 返回新的 activeTheme
    ├─ <RootStyleProvider theme={activeTheme}>
    │   ├─ <BaseProvider>         → BaseWeb 组件重渲染
    │   ├─ <CacheProvider>
    │   │   └─ <EmotionThemeProvider>
    │   │       ├─ <Global styles={globalStyles(theme)}>
    │   │       │   → 生成新的 <style data-emotion="st-emotion-cache-global">
    │   │       │   → html { font-size: Npx }
    │   │       │   → body { background-color, color, font-family... }
    │   │       │   → 滚动条、链接、列表等基础样式
    │   │       └─ App 子树
    │   │           └─ 所有 styled-components 用新 theme 计算样式
    │   │              → 生成 emotion hash class 名
    │   │              → 插入 <style data-emotion="st-emotion-cache-css">
    │   │
    ├─ <FontSources> → <link href="https://fonts.googleapis.com/..." rel="stylesheet">
    └─ <FontFaceDeclaration> → <style> @font-face { font-family: "..."; src: url("...") } </style>
        │
        ▼
用户看到新主题渲染的页面 ✅
```
