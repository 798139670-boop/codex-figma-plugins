# FigmaToUGUI 导出节点类型总览

## Contents

- [1. 结论](#1-结论)
- [2. Role 解析顺序](#2-role-解析顺序)
- [3. 非复合控件 Role](#3-非复合控件-role)
  - [3.1 UIRoot：界面根](#3-1-uiroot-界面根)
  - [3.2 Container：普通结构与布局容器](#3-2-container-普通结构与布局容器)
  - [3.3 Label：文本](#3-3-label-文本)
  - [3.4 Image：普通图片或可渲染视觉](#3-4-image-普通图片或可渲染视觉)
  - [3.5 Reference：Prefab 引用](#3-5-reference-prefab-引用)
  - [3.6 Mask：真正的 UGUI 裁剪节点](#3-6-mask-真正的-ugui-裁剪节点)
  - [3.7 UiMask：只记录遮罩元数据](#3-7-uimask-只记录遮罩元数据)
  - [3.8 NotExport：忽略整个子树](#3-8-notexport-忽略整个子树)
- [4. Section 也是一种产物类型](#4-section-也是一种产物类型)
- [5. Render Policy 会覆盖 Role 的表面结果](#5-render-policy-会覆盖-role-的表面结果)
- [6. 哪些 Role 前缀可以不加](#6-哪些-role-前缀可以不加)
  - [6.1 可以不加](#6-1-可以不加)
  - [6.2 不能不加](#6-2-不能不加)
  - [6.3 复合控件内部也不能随意省略](#6-3-复合控件内部也不能随意省略)
- [7. 创作检查清单](#7-创作检查清单)


## 1. 结论

导出插件中的“节点类型”不能只理解为 Button、Toggle、List 等复合控件。转换结果由三层共同决定：

1. **Role**：节点在 UGUI 中承担什么语义，决定主要 Unity 组件和特殊后处理。
2. **Section**：节点属于界面、公共组件还是图片资源，决定生成 Prefab、Sprite，还是忽略。
3. **Render Policy**：节点最终作为结构节点、图片节点、占位节点，还是完全不生成 GameObject。

Page 模块名及“界面（Views）/ 组件（Coms）/ 图片（Res）”的组织方式见：[FigmaToUGUI模块与Page-Section导出规范](./FigmaToUGUI模块与Page-Section导出规范.md)。

本文纳入 Figma 创作规范的 Role 为：

```text
Auto
UIRoot
Container
Label
Image
Reference
Button
Toggle
ToggleGroup
List
Input
ProgressBar
Slider
ScrollBar
NotExport
Mask
UiMask
```

此前已经重点分析的复合控件是：`Button / Toggle / ToggleGroup / List / Input / ProgressBar / Slider / ScrollBar`。除此之外，还需要关注本文介绍的界面根、普通容器、文本、图片、Prefab 引用、子界面和控制类节点。

`Auto` 只是解析前的内部默认值，不是建议在 Figma 中直接创作的最终导出类型。

插件还会保留 Figma 原生节点种类：`Document / Page / Frame / Group / Text / Rectangle / Ellipse / Line / Vector / Image / Component / ComponentSet / Instance / Section`。这些是输入形态，不等同于 UGUI 导出类型。例如 Ellipse、Line、Vector 如果没有明确 Role 或可用图片渲染策略，最终仍可能按 Container/结构节点处理；Component Set 和 Instance 则主要改变 Prefab/Reference 流程。

## 2. Role 解析顺序

插件按以下顺序解析节点 Role：

1. 节点名命中 `RoleMatchRules.json` 时，使用显式 Role。
2. 未显式命名的 Figma Instance 自动解析为 `Reference`。
3. 未显式命名的 Auto Layout Frame 自动解析为 `List`。
4. Text 自动解析为 `Label`。
5. Rectangle 和 Image 自动解析为 `Image`。
6. Frame、Group、Component、Component Set 默认解析为 `Container`。
7. 其他类型兜底为 `Container`。

这带来两个重要结论：

- 普通文本、矩形、实例和无 Auto Layout 的容器通常可以省略 Role 前缀。
- Auto Layout Frame 如果只是布局容器，不能完全依赖自动推断；否则会被当成 `List`，应显式命名为 `Container_*`。

当前 `PluginData Role Mapping` 是禁用状态，因此不能把插件私有数据当作稳定的 Role 来源。

## 3. 非复合控件 Role

### 3.1 UIRoot：界面根

推荐命名：`UIRoot_ServerList`；规则也接受精确 token `root`。

作用：

- 作为 View Prefab 的根语义。
- “界面”（内部类型 Views）Section 下的顶层 Frame 可在配置允许时自动推断为 `UIRoot`。
- 根节点至少生成 `RectTransform` 和插件的 `FigmaNode` 标记。
- 根节点的普通视觉、布局组件会经过专门清理；符合“单一纯色结构背景”条件时仍可能保留 `Image` 背景。

Prefab 根有两种模式：

| 根模式 | 结果 |
| --- | --- |
| Stretch/普通 Prefab 根 | 不自带 Canvas，供业务 Canvas 或其他界面层级挂载 |
| `StandaloneCanvas` | 根上增加 `Canvas + CanvasScaler + GraphicRaycaster`，Canvas 为 Screen Space Overlay，Scaler 使用 Scale With Screen Size |

注意：`UIRoot` 是产物根语义，不应为了“看起来像根容器”给普通内部 Frame 使用。

### 3.2 Container：普通结构与布局容器

推荐命名：`Container_panel`；精确 token `group`、`frame` 也能匹配。

典型结果：

- 基础组件为 `RectTransform`。
- Figma Auto Layout 可进一步生成对应的 LayoutGroup、Padding、Spacing 等布局设置。
- 满足结构纯色背景规则时，可能增加 `Image`，并非所有 Container 都是空节点。
- 子节点按普通层级继续导出。

可以省略前缀的情况：普通非 Auto Layout 的 Frame、Group、Component、Component Set 默认就是 Container。

不能省略前缀的情况：Auto Layout Frame 仅用于排版、不希望成为 List 时，必须显式使用 `Container_*`。

### 3.3 Label：文本

推荐命名：`Label_title`。普通 Figma Text 节点可省略前缀，因为会自动解析为 Label。

典型结果为：

```text
RectTransform
TextMeshProUGUI
```

插件会同步文本、字体映射、字号、颜色、对齐等属性；Text Auto Resize 和 Auto Layout 场景还可能影响尺寸、`ContentSizeFitter` 或 `LayoutElement`。不要把多段需要分别控制的业务文本烘成一张 PNG。

如果 Frame 只是包着文本，Frame 本身不会因为包含 Text 而自动成为 Label；需要让 Text 节点直接承担 Label，或者显式命名目标节点。

### 3.4 Image：普通图片或可渲染视觉

推荐命名：`Image_bg`、`Img_icon`；Rectangle/Image 类型可省略前缀。

常见结果为 `RectTransform + Image`，但 Sprite 来源可能不同：

- Figma PNG Export；
- 九宫格渲染；
- Res Section 强制服务端渲染；
- Local Common Image；
- 简单纯色结构；
- 特殊旧路径中的 `FigmaImage`。

如果一个 `Image_*` 容器包含 PNG Export 视觉子节点，插件可用子节点的合成图作为当前 Image 的 Sprite，并压制该视觉子树。Image 不适合作为需要保留独立交互、文本或动态布局子节点的父根。

PNG Export 的放置规则详见 [Figma节点PNG Export设置规则分析](./Figma节点PNG-Export设置规则分析.md)。

### 3.5 Reference：Prefab 引用

推荐命名：`Reference_xxx`；规则也接受精确 token `ref`、`prefab`。

存在两条来源：

1. 真实 Figma Component Instance 在未显式指定其他 Role 时自动成为 Reference。
2. 显式 `Reference_*` 节点从名称参数中解析目标 Prefab 路径或名称。

导出时先创建/保留定位用 `RectTransform`，随后把它链接或替换为 Nested Prefab instance，并在布局完成后恢复实例的尺寸、缩放和位置。名称中的 `@size` 参数可控制是否使用 Figma 实例尺寸覆盖 Prefab 原尺寸。

注意：

- 真实 Instance 通常可以省略 `Reference_` 前缀。
- 普通 Frame 若要引用指定 Unity Prefab，不能省略 `Reference_`/`Ref`/`Prefab` 语义和目标信息。
- 显式 Reference 未携带有效路径、目标 Prefab 缺失或实例化失败时，会警告并降级为普通容器，而不是自动拥有目标 Prefab 的内容。
- 不要给需要展开导出其内部子树的 Instance 强行改成 Reference；Reference 路径会优先走嵌套 Prefab 逻辑。

### 3.6 Mask：真正的 UGUI 裁剪节点

推荐命名：`Mask_viewport`。

典型结果为：

```text
RectTransform
Image
Mask
```

插件明确保留其子树，并设置 Unity `Mask.showMaskGraphic = false`。因此 Mask 的 Image 用于定义模板裁剪区域，而不是显示一张普通背景图。

注意：

- Mask 必须有可作为裁剪模板的有效视觉/形状。
- 被裁剪内容必须位于 Mask 子树内。
- 它与 List 的 `Clip content -> ScrollRect` 不是同一概念；滚动视口是否还需要显式 Mask，要看目标结构和插件生成路径。
- Mask 是 Unity UI 的 stencil 裁剪，会增加渲染与层级约束，不应把普通装饰背景都命名成 Mask。

### 3.7 UiMask：只记录遮罩元数据

参考选服 ToUGUI 目标结构，单个界面遮罩推荐使用精确保留名 `uimask`。插件也支持 `UiMask_xxx` 前缀形式，但这仍属于显式 Role 标记，并非可以省略的普通业务前缀。

`UiMask` 与 Unity `Mask` 完全不同：

- 自身不生成 GameObject，Render Policy 为 `NoNode`。
- 自身子树被压制。
- 插件把它的 fill 记录到最近的可导出父节点 `FigmaNode` 元数据中。
- 只接受“一个可见纯色 fill”；不满足时跳过并警告。
- 记录颜色的 alpha 为 `Figma color.a × paint opacity × node opacity`。
- 不得设置 PNG Export：加载器的 PNG 后处理会把除 `Mask` 外的 Role 改写为 `Image`，从而丢失 `UiMask` 语义。

它适用于界面/弹窗遮罩颜色等元信息，不用于裁剪子节点。当前插件只负责写入 `FigmaNode.UiMasks`，仓库内没有自动读取该字段并创建可见蒙层的运行时实现。若目标是裁剪，请使用 `Mask_*`；若目标是实际显示可点击的半透明背景，应根据运行时需求使用 Image/Button 等正常节点，不能默认用 UiMask 替代。

发现 `uimask` 时还要判断该 View 是否实际表达“非全屏弹窗叠加在场景上”：

1. 在遮罩上方寻找弹窗背景根，通常是小于 UIRoot 的图片节点或 Component Instance。
2. 结合层级顺序和几何范围，识别弹窗下方只用于展示场景的背景节点。
3. 将这些背景节点收进新建的 `NotExport` Frame，使结构形成 `NotExport（场景背景） / uimask / 弹窗正式结构`。
4. 已经带 `NotExport`、`NoExport` 或 `Ignore` 压制语义的节点不重复包裹。
5. 不能仅凭几何相交把弹窗内部的正式 Label、Button、List 等内容放进 NotExport；z-order 和业务职责必须同时满足。

### 3.8 NotExport：忽略整个子树

推荐命名：`NotExport_guides`。以下 token 也会匹配：

```text
NoExport
Ignore
Placeholder
PlaceLimiter
```

行为：

- 自身不生成 GameObject 或 Sprite。
- `SuppressChildren = true`，整个子树都不继续导出。
- 独立 Component 若为 NotExport，也不会生成组件 Prefab。

它适合说明、标尺、占位示意、设计注释和不应进入运行时的样例内容。它不同于隐藏节点：隐藏是 Figma 可见性策略；NotExport 是明确且稳定的导出语义。

## 4. Section 也是一种产物类型

插件内部 Section 枚举为：

```text
Unknown / Views / Coms / Vars / Res / Export
```

当前策略层的主要映射如下：

| Section | 当前策略 | 主要产物 |
| --- | --- | --- |
| Views | `ViewPrefab` | View/页面 Prefab；顶层 Frame 可推断 UIRoot |
| Coms | `ComponentPrefab` | Component、Component Set、Variant 组件 Prefab |
| Res | `ImageAssetSource` | Sprite 资源，不生成普通 UI GameObject |
| Vars | `Ignored` | 当前不生成 UI 节点 |
| Export | `Ignored` | 当前统一策略层忽略；代码中仍有部分历史兼容扫描入口，不建议新文档依赖 |
| Unknown | `Ignored` | 默认导出模式下不作为正式产物范围 |

Component Set/Variant 不是 Role，但会影响 Prefab 产物：一个变体集合通常编译为一个带 `FigmaVariantController` 的组件 Prefab；Instance 再通过 Reference/Nested Prefab 使用它。

## 5. Render Policy 会覆盖 Role 的表面结果

即使 Role 相同，最终 Unity 组件也可能被 Render Policy 改写：

| Render Policy | 结果 |
| --- | --- |
| `StructuralNode` | 通常为 RectTransform；纯色结构背景可增加 Image；Clip List 可增加 ScrollRect |
| `ImageNode` | RectTransform + Image，节点作为普通图片输出 |
| `SingleImageNode` | RectTransform + Image，整棵视觉子树合成为单图并压制子节点 |
| `FigmaImageNode` | RectTransform + FigmaImage，特殊/兼容渲染路径 |
| `PlaceholderFrame` | 一般只有 RectTransform；Role 为 Image 时可有 Image |
| `NoNode` | 不生成 GameObject，常见于 UiMask、资源 Section、顶层纯资源和被忽略范围 |

因此不能仅根据 Figma 节点是 Frame 或名称是 Container，就断言 Unity 中一定只有 RectTransform；PNG Export、九宫格、纯色背景、Section 和子图结构都会改变最终结果。

## 6. 哪些 Role 前缀可以不加

这里的“可以不加”是指：省略 Role 前缀后，当前插件仍能根据 Figma 原生节点类型或所在层级稳定推断目标 Role。业务名称仍应保留。

### 6.1 可以不加

| 目标 Role | 可以不加前缀的 Figma 结构 | 自动推断依据 | 示例 |
| --- | --- | --- | --- |
| Label | Text | Text 自动成为 Label | `title`、`serverName` |
| Image | Rectangle、Image | Rectangle/Image 自动成为 Image | `bg`、`icon` |
| Reference | 真实 Component Instance | Instance 自动成为 Reference，并按 component id/key 链接 Prefab | `ServerItem` 实例 |
| Container | 无 Auto Layout 的 Frame、Group、Component、Component Set | 这些类型默认成为 Container | `panel`、`content` |
| UIRoot | “界面”（内部类型 Views）Section 的顶层 Frame | 层级推断可把 View 顶层节点改成 UIRoot | `ServerList` |
| List | Auto Layout Frame | 未显式命名的 Auto Layout Frame 自动成为 List | `serverItems` |

其中 UIRoot 和 List 属于“条件可省略”：

- UIRoot 依赖节点确实位于“界面”顶层，并启用了 `InferUiRootFromViewsTopLevel`；跨项目规范中仍建议写 `UIRoot_*`。
- Auto Layout Frame 无论是否真的需要列表行为都会被推断为 List，所以只有确认它是列表/滚动布局时才适合省略 `List_*`。

### 6.2 不能不加

| 目标 Role | 必须显式表达的原因 | 推荐形式 |
| --- | --- | --- |
| Container（Auto Layout Frame） | 否则会被误推断为 List | `Container_panel` |
| Button | 不能从矩形、文本或点击外观自动推断交互行为 | `Button_confirm` / `Btn_confirm` |
| Toggle | Variant 状态和普通 Instance 不会自动推断 Unity Toggle | `Toggle_music` |
| ToggleGroup | 不会根据后代 Toggle 自动建立互斥组 | `ToggleGroup_tabs` |
| Input | 文本框外观不足以推断 TMP_InputField | `Input_account` |
| ProgressBar | bg/fill 层级不会自动把根识别为进度条 | `ProgressBar_loading` |
| Slider | bg/fill/handle 层级不会自动推断 Slider | `Slider_volume` |
| ScrollBar | bg/handle 层级不会自动推断 Scrollbar | `ScrollBar_vertical` |
| Reference（普通 Frame） | 只有真实 Instance 才会自动 Reference；普通 Frame 需要目标 Prefab 信息 | `Reference_xxx` / `Ref` / `Prefab` |
| Mask | 普通形状不会自动成为 Unity Mask | `Mask_viewport` |
| UiMask | 属于只记录 fill 的特殊元数据语义 | `uimask`；需要区分多个时可用 `UiMask_xxx` |
| NotExport | 属于压制自身和整个子树的导出指令 | `NotExport_guides` / `NoExport` / `Ignore` |

### 6.3 复合控件内部也不能随意省略

即使 Text、Rectangle 在普通场景可以自动成为 Label/Image，作为复合控件的关键绑定子节点时仍应使用插件能识别的职责名，例如 `Background / Checkmark / Fill / Handle / Label / Content / Viewport`。否则根 Role 虽然正确，`targetGraphic`、`graphic`、`fillRect`、`handleRect`、`textComponent` 等引用仍可能绑定失败。

建议团队统一使用 `Role_businessName`。前缀可省略主要用于理解插件兜底逻辑，不应成为牺牲结构可读性的理由。

## 7. 创作检查清单

导出前逐项确认：

- 当前节点位于 `Views / Coms / Res` 中正确的产物范围。
- 行为节点使用明确 Role；普通视觉节点没有误命中行为前缀。
- Auto Layout Frame 是 List 还是 Container 已明确。
- 真正 Component Instance 应链接为 Reference，目标 Prefab 可解析。
- Mask 与 UiMask 没有混用。
- NotExport 下没有仍需导出的正式资源或节点。
- PNG Export 没有设置在 UIRoot、行为根或布局根上。
- 最终同时核对 Role、Section 和 Render Policy，而不是只看节点名称。

