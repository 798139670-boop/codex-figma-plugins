# Figma 复合组件：几何布局与 UGUI 注意事项

## Contents

- [1. 分析目标](#1-分析目标)
- [2. Figma 样例中的几何关系](#2-figma-样例中的几何关系)
- [3. 所有复合组件的共同规则](#3-所有复合组件的共同规则)
  - [3.1 根节点尺寸同时是行为边界](#3-1-根节点尺寸同时是行为边界)
  - [3.2 普通节点默认使用中心 Anchor/Pivot](#3-2-普通节点默认使用中心-anchor-pivot)
  - [3.3 Auto Layout 会接管普通子节点位置](#3-3-auto-layout-会接管普通子节点位置)
  - [3.4 不要让两个布局系统同时控制同一轴](#3-4-不要让两个布局系统同时控制同一轴)
  - [3.5 背景尺寸和点击尺寸不是天然相同](#3-5-背景尺寸和点击尺寸不是天然相同)
  - [3.6 公共组件实例不要随意改成长宽比](#3-6-公共组件实例不要随意改成长宽比)
- [4. Button](#4-button)
- [5. Toggle 与 ToggleGroup](#5-toggle-与-togglegroup)
  - [5.1 普通 Toggle](#5-1-普通-toggle)
  - [5.2 Variant Toggle](#5-2-variant-toggle)
  - [5.3 ToggleGroup 初始状态](#5-3-togglegroup-初始状态)
- [6. Input](#6-input)
- [7. ProgressBar](#7-progressbar)
- [8. Slider](#8-slider)
- [9. List / ScrollView / Mask](#9-list-scrollview-mask)
  - [9.1 根尺寸就是 Viewport 尺寸](#9-1-根尺寸就是-viewport-尺寸)
  - [9.2 Content 尺寸与对齐](#9-2-content-尺寸与对齐)
  - [9.3 Mask 注意事项](#9-3-mask-注意事项)
- [10. Scrollbar](#10-scrollbar)
  - [10.1 选服 `ScrollBar_huadong` 实例分析](#10-1-选服-scrollbar_huadong-实例分析)
  - [10.2 Scrollbar 的 Figma 制作模板](#10-2-scrollbar-的-figma-制作模板)
- [11. Variant 组件的几何一致性](#11-variant-组件的几何一致性)
- [12. 推荐验收方式](#12-推荐验收方式)
  - [Button / Toggle / Input](#button-toggle-input)
  - [ProgressBar / Slider](#progressbar-slider)
  - [List / Scrollbar](#list-scrollbar)
  - [Common Instance / Variant](#common-instance-variant)
- [15. 最重要的设计检查清单](#15-最重要的设计检查清单)


## 1. 分析目标

本文是在“节点层级和命名正确”之外，对复合组件的几何、对齐、尺寸、点击区域和 Unity 驱动行为进行补充说明。

分析参考：

- 复合结构 Figma 样例
- UGUI uGUI 2.0 文档

本文涉及的主要组件：Button、Toggle、ToggleGroup、Input、ProgressBar、Slider、List/ScrollView、Scrollbar 和 Mask。

这些组件的静态视觉应在哪一层设置 Figma PNG Export，详见：[Figma节点PNG Export设置规则分析](./Figma节点PNG-Export设置规则分析.md)。

## 2. Figma 样例中的几何关系

样例不是只提供了命名规范，也表达了子节点相对父节点的尺寸和位置关系：

| 组件 | 根尺寸 | 关键子节点几何 |
| --- | --- | --- |
| `Button_button` | `220 × 72` | `Image_bg` 与根同尺寸；Label 左右各留 20，垂直居中 |
| `Toggle_toggle` | `240 × 64` | 背景 `36 × 36`，位于 x=0、垂直居中；checked 图 `20 × 20`，在背景内居中；Label 从 x=52 开始 |
| `Input_input` | `320 × 72` | `Image_bg` 铺满；Text Area 为 `288 × 48`，四周留 `16 / 12` |
| `ProgressBar_progress` | `360 × 40` | 背景与 Fill 都铺满根节点 |
| `Slider_slider` | `360 × 64` | Track 宽 360、高 10、垂直居中；Fill Area 宽 360；Handle Area 宽 360；Handle 为 `28 × 28` |

这些关系最终会转成 RectTransform 尺寸、Anchor、布局组件引用或 Unity 控件的驱动区域。层级正确但几何错误，仍可能得到“能导出但不好用”的 Prefab。

## 3. 所有复合组件的共同规则

### 3.1 根节点尺寸同时是行为边界

复合组件根尺寸不只是包围盒，它通常还代表：

- Button、Toggle、Input、Slider 的预期点击/拖拽区域。
- LayoutGroup 中该 Item 的占位尺寸。
- 公共组件实例的设计基准尺寸。
- Variant 切换时允许变化的组件内部尺寸。

根尺寸应覆盖所有需要交互和布局占位的内容。不要让可见子节点大量越出根 Rect；越界视觉可能仍能显示，但父级 Mask、LayoutGroup、射线命中和列表裁剪会按父级 Rect 工作。

### 3.2 普通节点默认使用中心 Anchor/Pivot

转换器对非 Auto Layout 普通子节点先设置：

```text
anchorMin = anchorMax = (0.5, 0.5)
pivot = (0.5, 0.5)
sizeDelta = Figma 节点实际尺寸
anchoredPosition = 子节点中心 - 父节点中心
```

随后再根据 Figma Constraints 改写 Anchor：

- Left / Right / Center、Top / Bottom / Center 转成相应固定 Anchor。
- Left & Right、Top & Bottom 转成 Stretch，并保留四边 inset。
- Figma `SCALE` 当前不转成比例 Anchor：水平回退 Left，垂直回退 Top，并产生警告。

因此，响应式组件不要使用 `SCALE` 期待随父级按比例变化。应选择：

- 固定尺寸、固定边距；或
- 水平/垂直 Stretch；或
- Figma Auto Layout 的 Fill container。

### 3.3 Auto Layout 会接管普通子节点位置

当父节点是 Figma Auto Layout，且子节点不是 Absolute positioning 时，转换器会把子节点 `anchoredPosition` 设为 0，实际排列交给 Unity 的 Horizontal/Vertical/Grid LayoutGroup。

以下 Figma 信息会映射到 Unity LayoutGroup：

- Auto Layout 方向 → Horizontal/Vertical LayoutGroup。
- Padding → LayoutGroup padding。
- Item spacing → LayoutGroup spacing。
- 主轴和交叉轴对齐 → childAlignment。
- Fill container / layoutGrow / Stretch → LayoutElement 的 flexible 或固定尺寸。
- Absolute positioning → `LayoutElement.ignoreLayout = true`，保留显式位置。

需要重叠的节点必须使用 Absolute positioning，例如：

- Toggle 的 checked 图覆盖在背景图上。
- Button 的背景铺底、Label 叠在背景上。
- ProgressBar 的 Fill 覆盖背景。
- Slider 的 Track、Fill Area、Handle Area 彼此重叠。
- List 内独立悬浮的 Scrollbar。

如果把这些节点都作为普通 Auto Layout 子节点，它们会被依次排开，而不是叠放。

### 3.4 不要让两个布局系统同时控制同一轴

UGUI Auto Layout behavior区分 Layout Element 和 Layout Controller。转换器也会避免在行为根或父 LayoutGroup 子节点上滥用 ContentSizeFitter。

特别注意：

- Button、Toggle、Input、Slider、ProgressBar、Scrollbar、List 根不会自动使用 ContentSizeFitter。
- 位于父级 Auto Layout 中的节点，尺寸主要由父 LayoutGroup + 自身 LayoutElement 决定。
- 需要 Hug content 的普通 Container 可使用 Auto Layout；行为控件根应给出明确尺寸。
- 不建议在同一轴上同时使用父 LayoutGroup、ContentSizeFitter 和业务代码改 RectTransform。

### 3.5 背景尺寸和点击尺寸不是天然相同

Unity 的 Graphic Raycaster 只会命中启用了 `raycastTarget` 且指针位于其 Rect 内的 Graphic。转换器会把 Button/Toggle/Input 的目标背景 Graphic 设为可射线命中，但不会自动把这个子 Graphic 拉伸到行为根大小。

因此：

- 若希望整个控件根都可点击，`Image_bg` 或 `HitArea` 应铺满根节点。
- 若背景只覆盖一个小图标，则标签或根节点空白区域可能不可点击。
- 纯装饰 Image、Text 通常应保持 `raycastTarget = false`，避免挡住父控件。
- Variant Toggle 是例外：Binder 会在根节点增加 `FigmaHitArea`，点击区域按根 Rect 工作。

### 3.6 公共组件实例不要随意改成长宽比

普通 Common Prefab 引用默认优先保留源 Prefab 根尺寸，再通过 `localScale` 适配 Figma 实例目标尺寸。如果目标宽高比例与源组件不同，会产生非等比缩放，文字、圆形、边框和九宫格都可能失真。

建议：

- 固定视觉组件保持源宽高比，只做等比整体缩放。
- 需要响应式拉伸的组件，内部必须使用 Stretch/LayoutGroup，并在实例名中使用精确参数 `@size`，让目标宽高直接驱动根尺寸。
- `@size` 不是“任意拉伸都安全”的开关；固定位置的内部子节点仍不会自动响应。

## 4. Button

推荐几何：

```text
Button_confirm                 220 × 72
  Image_bg                     铺满 220 × 72
  Label_text                   位于安全内边距内，水平/垂直居中
```

注意事项：

1. `Image_bg` 最好与 Button 根同尺寸或 Stretch 铺满，否则有效点击区域可能小于 Button 根。
2. Label 应保留左右安全边距，不能依赖文字当前长度刚好不溢出；本地化文案通常更长。
3. Label 的 Rect 应覆盖可排版区域，文本自身再用 TMP alignment 居中，不要靠空格或手工偏移居中。
4. 若 Button 使用 Auto Layout，根据设计选择：
   - 背景 Absolute + Stretch，文字作为普通布局内容；或
   - 根不用 Auto Layout，背景和文字均使用明确 Constraints。
5. Button 根不应只有一个尺寸很小的图标 Graphic，却期望整个大根区域都可点击；需要显式铺满的 `Image_bg`/`HitArea`。
6. Unity Button 的 On Click 在“按下并在按钮区域内释放”时触发；拖出有效射线区域再释放不会触发。

样例中 `Image_bg` 与 `Button_button` 同为 `220 × 72`，这是正确的命中面尺寸关系。

## 5. Toggle 与 ToggleGroup

### 5.1 普通 Toggle

样例结构：

```text
Toggle_Option                  240 × 64
  Image_bg                     36 × 36，垂直居中
  Image_checked                20 × 20，在 Image_bg 内居中
  Label_text                   x=52，垂直居中
```

视觉对齐注意：

- `Image_checked` 应与 `Image_bg` 共用视觉中心。
- checked 图应使用 Absolute positioning；转换器发现它参与 Toggle 根 Auto Layout 时会发出警告。
- Label 和图标之间应使用稳定间距，不要让 Label Rect 覆盖 checked 图。
- checked 图最好不参与射线检测，转换器会把绑定的 `Toggle.graphic.raycastTarget` 关闭。

点击区域注意：

- 当前样例 `Image_bg` 只有 `36 × 36`，而 Toggle 根为 `240 × 64`。
- 普通 Toggle 有背景时，插件会使用该背景作为 `targetGraphic`，并移除根 EmptyRaycast。
- 这意味着如果没有其他可命中的 Graphic，标签区域不一定属于有效点击区域。

若期望“点击文字也能选中”，推荐增加铺满根节点的透明 `HitArea`，或调整结构使可绑定背景 Graphic 覆盖整个根 Rect。不要把小图标背景同时当作整行命中面。

### 5.2 Variant Toggle

Variant Toggle 会由 `FigmaHitArea` 覆盖根 Rect，并把 `Toggle.graphic` 置空，视觉由 `FigmaVariantController` 切换。因此：

- 根 Rect 应准确表达期望点击范围。
- 各 Variant 的根尺寸最好一致；若必须变化，应确认 ToggleGroup 和外部 LayoutGroup 能接受运行时尺寸变化。
- 各 Variant 内对应节点应保持稳定语义和对齐基准，减少 fallback branch 和切换跳动。
- checked/unchecked 两态的文字基线、中心点和外边界应尽量一致，避免切换时“抖一下”。

### 5.3 ToggleGroup 初始状态

UGUI behavior requires：ToggleGroup 在场景加载或实例化时，如果多个成员初始为 On，不会立即自动纠正；只有后续某个 Toggle 被切换为 On 时才会关闭其他成员。

因此，Figma/Prefab 初始状态必须保证：

- `Allow Switch Off = false` 的互斥组：恰好一个成员初始 checked。
- 允许全不选的组：零个或一个初始 checked，但不能多个。
- 不要依赖 ToggleGroup 在 Awake 时替你消除多个 checked。

## 6. Input

推荐结构：

```text
Input_account                  320 × 72
  Image_bg                     铺满根
  Text Area                    四周留内边距
    Label_input_holder         与输入文本完全重叠
    Label_text                 与 placeholder 完全重叠
```

注意事项：

1. `Image_bg` 最好铺满根，用作 `TMP_InputField.targetGraphic` 和点击/聚焦区域。
2. Text Area 决定输入文字、光标、选择区域的可用空间。其边界必须避开边框、左侧图标、右侧清除按钮等视觉。
3. Placeholder 与实际 Text 应使用同一个 Rect、Anchor、Pivot、字体大小和 alignment；两者是替换显示关系，不是横向排列关系。
4. Placeholder/Text 需要重叠，不能作为普通 Auto Layout 兄弟依次排列。
5. 单行输入应预留上下空间，防止字体 ascender/descender、Caret 或选中框贴边裁切。
6. 动态文本应验证中文、英文、数字、超长文本和不同字体 fallback，而不仅检查 Figma 示例字符串。
7. 若缺少 Text Area，插件会回退根 Rect；若缺少 Text 节点，会自动生成一个四周 `10 / 6` inset 的 Text Area，但这只是兜底，不应代替正式设计。

样例的 Text Area 相对根为左右 16、上下 12，且 Placeholder 与 Text 完全重合，符合 Input 的基本几何要求。

## 7. ProgressBar

推荐结构：

```text
ProgressBar_loading            360 × 40
  Image_bg                     铺满
  Image_fill                   与背景同起点、同高度，通常铺满
```

插件会把 `Image_fill` 设置为：

```text
Image.Type = Filled
FillMethod = Horizontal
FillOrigin = Left
fillAmount = 1
```

注意事项：

1. ProgressBar 的 Fill Rect 应与完整进度轨道对齐；不要通过把 Figma Fill 画成半宽来表达初始 50%，当前插件仍会把 `fillAmount` 初始化为 1。
2. Fill 和背景应共用左边界与垂直中心；高度不同则是刻意的内缩视觉，而不是位置误差。
3. `Image.Type.Filled` 是按 Sprite 矩形裁切。若 Fill Sprite 两端都有圆角，部分进度时右端通常会被直接切断，不能天然保持双端圆角。
4. 需要始终保留圆形端帽时，应使用适合 Filled 裁切的 Sprite、独立端帽、Mask 或项目自定义 Progress 实现。
5. ProgressBar 是展示组件，Fill 默认关闭 raycast，不应挡住上层交互。
6. 当前方向固定为水平从左到右；纵向、反向或环形进度不能只靠 Figma 摆放表达。

## 8. Slider

推荐结构：

```text
Slider_volume                  360 × 64
  Image_bg                     Track：360 × 10，垂直居中
  Fill Area                    与 Track 使用相同水平范围
    Image_fill                 与 Track 左端对齐
  Handle Slide Area            与 Track 使用相同中心和运动范围
    Image_handle               固定视觉尺寸，例如 28 × 28
```

插件固定设置：

- Direction = LeftToRight。
- Min/Max = 0/1。
- `fillRect` 绑定 `Image_fill`。
- `handleRect` 绑定 `Image_handle`。
- 初始值优先由 `Fill 宽度 / Track 宽度` 计算。

几何注意：

1. Track 和 Fill 必须具有有效 Figma bounds，并在水平方向使用同一个量尺；否则初始值会回退 Unity Rect 并产生 warning。
2. Fill 左端应与 Track 左端一致。若 Fill 整体偏移，即使宽度比例正确，运行时仍可能出现视觉跳变。
3. Fill/Handle 是 Unity `Slider` 主动驱动的 RectTransform。导出后它们的 Anchor 和位置会被重新配置，不能期望保留普通图片的点 Anchor。
4. Handle Slide Area 应与 Track 对齐。插件会在可识别 Area 下把运动宽度调整为 `TrackWidth - HandleWidth`，使 Handle 中心到达两端时视觉不会超出半个 Handle。
5. Handle 的垂直中心应与 Track 中心一致；高度较大是正常的触点视觉，不应通过父 Auto Layout 将其挤到轨道上方或下方。
6. Track、Fill Area、Handle Area 应重叠，不应作为普通横向/纵向 Auto Layout 子节点排列。
7. 当前插件不从 Figma 推断 RightToLeft、BottomToTop、整数步进或业务数值范围；这些需要扩展命名规则、配置或业务代码。
8. 样例 Fill 与 Track 都是满宽，因此导出初始值为 1；若需要展示 60%，应让 Fill Figma 宽度为 Track 的 60%，并保持共同左边界。

UGUI behavior also requires Fill Rect 是填充区域、Handle Rect 是滑块区域，二者会随 Slider value 变化，不是静态装饰节点。

## 9. List / ScrollView / Mask

### 9.1 根尺寸就是 Viewport 尺寸

List 开启 `Clip content` 后，插件自动生成：

```text
List_xxx                      ScrollRect
  Viewport                    Stretch 铺满 List 根，Image + Mask
    Content                   左上 Anchor/Pivot
      Item...
  ScrollBar_*                 可选，仍留在 List 根
```

Viewport 被强制铺满 List 根，所以：

- List 根宽高就是可见窗口大小。
- Clip content 的边界就是实际裁剪边界。
- 若需要为 Scrollbar 预留空间，应在设计中缩小可视列表根，或接受 Scrollbar 叠在 Viewport 上；当前 Viewport 不会自动因 Scrollbar 缩窄。

### 9.2 Content 尺寸与对齐

Content 使用左上 Anchor/Pivot，初始位置为左上角。其尺寸计算规则：

- 横向列表：宽度 = 左右 Padding + 所有 Item 宽度 + 间距；高度至少等于 Viewport。
- 纵向列表：高度 = 上下 Padding + 所有 Item 高度 + 间距；宽度至少等于 Viewport。
- Content 不会小于 Viewport。

因此：

- Figma Auto Layout 的 Padding、spacing 和 Item 实际尺寸必须准确。
- 纵向 List 的 Item 若应铺满宽度，应使用 Fill container/Stretch；不要依赖当前示例宽度碰巧相等。
- 普通 Auto Layout Item 位置由 Content 的 LayoutGroup 接管，Figma 的手工 x/y 不再是权威数据。
- 装饰、悬浮按钮和 Scrollbar 应设 Absolute positioning，避免计入 Content 尺寸与模板列表。

### 9.3 Mask 注意事项

UGUI behavior requires Mask 通过父 Graphic 的形状限制子节点显示。当前插件生成的 List Viewport 使用 `Image + Mask`，并设置 `showMaskGraphic = false`。

需要注意：

- 子内容必须位于 Viewport/Mask 的后代层级才会被裁剪。
- 超出根 Rect 的 Item 会被裁掉，这是预期行为，不是导出偏移。
- Mask 的边界不会自动包含阴影、外发光；需要显示在裁剪区外的效果不能放在被 Mask 的子树里。
- 多层嵌套 Mask 会增加 stencil 关系复杂度，应避免无意义嵌套。
- List 背景通常应用到 Viewport Image，而不是作为 Content 中的第一个 Item。

## 10. Scrollbar

推荐 Figma 结构只需：

```text
ScrollBar_v                   根 Rect = 完整轨道
  Image_handle               Handle 的初始位置和尺寸
```

插件会自动补成 Unity 标准结构：

```text
ScrollBar_v
  Sliding Area               Stretch 铺满根
    Image_handle
```

几何推导规则：

- 根宽高比或 `_h`/`_v` token 决定横向/纵向。
- `Handle 主轴尺寸 / Track 主轴尺寸` 决定 `Scrollbar.size`。
- Handle 更靠近哪一端决定 Direction。
- Handle 在交叉轴相对 Track 的 inset 会保留。
- Handle 应放在希望代表 `value = 0` 的端点。

注意事项：

1. 不要把 Handle 画成与 Track 等长，除非确实希望 size=1、没有可滚动范围。
2. 纵向 Handle 若希望初始贴顶，应在 Figma 中贴顶；插件会据此选择 TopToBottom。
3. Handle 在交叉轴最好与 Track 居中或使用明确等距 inset，否则导出会忠实保留偏移。
4. Scrollbar 自己会通过 DrivenRectTransformTracker 重写 Handle 在滚动轴上的 Anchor；不要依赖 Handle 的普通 Figma Constraints 控制运行时位置。
5. Scrollbar 作为 List 的直接子节点才能自动绑定到 ScrollRect。
6. List 通常也是 Auto Layout，因此 Scrollbar 必须设为 Absolute positioning；否则可能被路由进 Content 或参与 Item 排列。
7. UGUI区分 Slider 和 Scrollbar：Slider 用于选择数值，Scrollbar 用于表达和控制可见内容比例，不能只因外形相似而混用。

### 10.1 选服 `ScrollBar_huadong` 实例分析

参考：选服 / ScrollBar_huadong。

Figma 实际结构和尺寸：

```text
ScrollBar_huadong              10 × 1108
  Image_bg                     10 × 1108，x=0，y=0
    serverlist_img_cemianxian  10 × 1108
  Image_handle                 10 × 232，x=0，y=0
    serverlist_img_handle      10 × 232
```

按当前插件逻辑，这个实例会得到：

| 项目 | 推导结果 | 依据 |
| --- | --- | --- |
| 角色 | `ScrollBar` | 根名以 `ScrollBar` 开头 |
| 朝向 | 纵向 | 根高 1108 远大于宽 10；名称没有 `_h`/`_v` 时按宽高比推断 |
| Direction | `TopToBottom` | Handle 顶部间距 0，小于底部间距 876 |
| 初始 Value | `0` | 插件固定初始化为 0，Handle 放在 Direction 的起点顶部 |
| Size | `232 / 1108 ≈ 0.2094` | Handle 主轴尺寸除以 Track 主轴尺寸 |
| 交叉轴 inset | 左右均为 0 | Handle 与 10px 宽 Track 完全同宽 |
| Sliding Area | 自动生成并铺满根 | Figma 无需手工创建 |

导出目标层级：

```text
ScrollBar_huadong              Image + Scrollbar
  Image_bg
    serverlist_img_cemianxian
  Sliding Area                自动生成，Stretch，零 inset
    Image_handle              Scrollbar.handleRect
      serverlist_img_handle
```

这个样例在 Scrollbar 自身几何上是正确的：

- `Image_bg` 完整覆盖根轨道。
- Handle 与轨道同宽，没有横向漂移。
- Handle 明确贴顶，能够稳定推导 `TopToBottom`。
- Handle 高度约占轨道 20.94%，可以表达当前可视区域约占完整内容的比例。

仍需在实际 List 中确认以下事项：

1. `ScrollBar_huadong` 必须是对应 `List_*`/`ScrollView_*` 根的直接子节点，才能自动写入 `ScrollRect.verticalScrollbar`。
2. 如果 List 根使用 Auto Layout，Scrollbar 必须设为 Absolute positioning；否则会参与 Item 排列或被路由到 Content。
3. List 的 Viewport 默认铺满根，不会自动因为 10px Scrollbar 缩窄。若 Scrollbar 不应覆盖内容，应在设计上给内容预留右侧空间。
4. `Scrollbar.size` 在绑定到 ScrollRect 后可能由 ScrollRect 按运行时 Content/Viewport 比例继续更新；Figma 的 0.2094 主要提供初始 Prefab 几何和未绑定时的默认表现。
5. 如果服务器数量、Item 高度或 Viewport 高度在运行时变化，不要把 232px Handle 高度当成永久业务值，应让 ScrollRect 驱动实际比例。
6. 当前根名虽可依靠宽高比正确识别纵向，但更稳定的业务命名是 `ScrollBar_serverList_v`，避免未来组件旋转或尺寸调整后方向漂移。

### 10.2 Scrollbar 的 Figma 制作模板

可将选服样例整理为通用模板：

```text
ScrollBar_<business>_v        固定轨道宽度，完整可滚动高度
  Image_bg                    Stretch 铺满根
    <track sprite>
  Image_handle                与轨道同宽或使用明确左右 inset
    <handle sprite>
```

制作时只需确定四件事：

- 根 Rect 是完整轨道范围。
- Handle 主轴尺寸表达初始可视比例。
- Handle 放在 `value=0` 对应端点。
- Handle 交叉轴位置表达最终居中或 inset。

不需要在 Figma 中创建 `Sliding Area`，也不应手工制作会与 Unity `Scrollbar` 驱动 Anchor 冲突的运动动画或 Constraints。

## 11. Variant 组件的几何一致性

Component Set 的不同 Variant 除了节点语义需要可匹配，还应遵循以下几何规则：

- 同一语义节点尽量保持相同父路径和名称。
- 同一位置的 Label 保持相同 Rect、Pivot 和 alignment。
- 仅颜色或 Sprite 改变时，不要无意改变节点尺寸。
- checked/unchecked 等状态根尺寸尽量相同。
- 若 Variant 根尺寸确实变化，外层 Auto Layout 会重新排版；固定坐标父容器则可能与邻居重叠。
- 不同 Variant 使用不同 Auto Layout 模式时，会切换 LayoutGroup 状态，风险高于单纯视觉变化，必须在 Unity 中逐状态验证。
- fallback branch 的范围越大，运行时整棵子树切换越明显，也更容易产生布局抖动和引用不稳定。

## 12. 推荐验收方式

仅看 Prefab 静态截图不足以验证复合组件。建议每个组件至少做以下测试：

### Button / Toggle / Input

- 点击根 Rect 的四角、文字区和图标区，确认命中范围符合预期。
- 切换 Highlighted、Pressed、Disabled、checked/unchecked。
- 使用长中文、长英文和字体 fallback 验证文本边界。
- ToggleGroup 初始化时确认最多一个 On。

### ProgressBar / Slider

- 测试值 0、0.01、0.5、0.99、1。
- 检查 Fill 两端、圆角、Handle 中心和运动边界。
- 改变父容器宽度后确认 Track、Fill Area、Handle Area 是否仍对齐。

### List / Scrollbar

- 测试 Content 小于、等于和大于 Viewport。
- 测试 0、1、多个 Item，以及不同 Item 高度。
- 检查 Padding、spacing、首尾 Item、裁剪边界和阴影。
- 拖动 Scrollbar 到两端，确认 Handle size、direction 和 Content 同步。
- 调整 List 根尺寸，确认 Item 的 Fill/Fixed 行为符合 Figma 设计。

### Common Instance / Variant

- 原始尺寸、等比缩放、非等比目标尺寸分别验证。
- 默认引用缩放与 `@size` 响应式尺寸分别验证。
- 切换所有真实 Variant 组合，检查根尺寸、文本基线和布局抖动。

## 15. 最重要的设计检查清单

- [ ] 复合组件根 Rect 覆盖全部布局占位和预期交互范围。
- [ ] Button/Input 的背景或 HitArea 铺满可点击根。
- [ ] 普通 Toggle 的文字区也有可命中的 Graphic；Variant Toggle 根尺寸正确。
- [ ] 需要重叠的背景、Fill、checked、Handle 使用 Absolute positioning 或明确 Constraints。
- [ ] Placeholder 与输入 Text 完全重叠且 alignment 一致。
- [ ] ProgressBar Fill 与完整轨道对齐，没有用半宽表达初始进度。
- [ ] Slider Track、Fill Area、Handle Area 共用同一水平量尺和中心。
- [ ] List 根尺寸就是 Viewport 尺寸，Clip content 已开启。
- [ ] List Padding、spacing、Item Fixed/Fill 设置明确。
- [ ] Scrollbar 是 List 直接子节点，并在 Auto Layout 中设为 Absolute。
- [ ] Scrollbar Handle 尺寸和 value=0 位置正确。
- [ ] Scrollbar 的 `_h`/`_v` 方向明确，或宽高比不会造成错误推断。
- [ ] Viewport 是否需要为 Scrollbar 预留空间已经明确。
- [ ] Variant 对应节点的尺寸、中心、文字基线和父路径稳定。
- [ ] Common Instance 不随意非等比缩放；响应式结构才使用 `@size`。
- [ ] Unity 中已经测试边界值、长文本、不同父尺寸和全部 Variant。

