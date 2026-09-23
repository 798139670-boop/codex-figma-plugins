# 测试skill：FigmaToUGUI 结构整理记录

## Contents

- [1. 整理对象](#1-整理对象)
- [2. 原结构的主要问题](#2-原结构的主要问题)
- [3. 整理后的 Page / Section](#3-整理后的-page-section)
- [4. View 内最终结构](#4-view-内最终结构)
  - [4.1 前缀精简原则](#4-1-前缀精简原则)
  - [4.2 `uimask` 的实际含义](#4-2-uimask-的实际含义)
  - [4.3 弹窗覆盖背景的 NotExport 整理](#4-3-弹窗覆盖背景的-notexport-整理)
- [5. ScrollView / ScrollBar 处理](#5-scrollview-scrollbar-处理)
  - [5.1 列表根](#5-1-列表根)
  - [5.2 Item 模板](#5-2-item-模板)
  - [5.3 ScrollBar](#5-3-scrollbar)
- [6. Clip content / Toggle 处理](#6-clip-content-toggle-处理)
- [7. PNG Export 设置](#7-png-export-设置)
- [8. 英文命名检查](#8-英文命名检查)
- [9. 验证结果](#9-验证结果)
- [10. Unity 侧建议验证](#10-unity-侧建议验证)


## 1. 整理对象

- Figma 文件：测试skill
- 原 Page：`测试（Test）`
- 原 Section：`界面`
- 原界面根：`2:87 服务器选择`
- 整理日期：2026-07-29

本次按以下现有规范和导出器行为整理：

- [FigmaToUGUI 模块与 Page / Section 导出规范](./FigmaToUGUI模块与Page-Section导出规范.md)
- [Figma 复合结构转 UGUI 整理分析](./Figma复合结构转UGUI整理分析.md)
- [公共组件库 new：Variant 与 Toggle 导出规则分析](./公共组件库new-Variant与Toggle导出规则分析.md)
- [Figma 节点 PNG Export 设置规则分析](./Figma节点PNG-Export设置规则分析.md)

## 2. 原结构的主要问题

原稿是可用于视觉评审的选服界面，但没有完整表达 UGUI 导出语义：

- Page 和多数正式导出 Layer 缺少稳定的英文产物名；`界面` Section 本身应保留中文；
- 服务器列表只是 Auto Layout Frame，未显式表达 `ScrollView`；
- 十个服务器 Item 都是列表直属子节点，没有区分正式模板和展示副本；
- Track 和 Handle 与列表同级，未组成 `ScrollBar`，也不是列表的直属子节点；
- 左侧九个 Variant Instance 位于开启 Clip content 的 Auto Layout Frame 中，应导出为 ScrollView，而不是 ToggleGroup；
- 背景、位图和状态图标没有稳定的英文图片名，也未设置 PNG Export；
- 设计标尺和垫图位于正式 View 子树内，隐藏后仍可能按默认配置进入 Prefab。

## 3. 整理后的 Page / Section

```text
Page ToUGUI（Test）                 # 模块名解析为 Test
  Section 界面
    ServerList                     # 1080 × 2400；界面顶层自动推断 UIRoot
```

对应节点：

| 用途 | 节点 ID | 整理后名称 |
| --- | --- | --- |
| Page | `0:1` | `ToUGUI（Test）` |
| 界面 | `1:2` | `界面` |
| View 根 | `2:87` | `ServerList` |

`ServerList` 保持标准尺寸 `1080 × 2400`。它是“界面”的直属顶层 Frame，会由层级规则自动推断为 UIRoot，目标 Prefab 名为 `ServerList.prefab`。本模块没有独立组件或图片资源，因此不创建空的“组件”“图片” Section；原先误建的 `6:37`、`6:38` 已移除。

## 4. View 内最终结构

省略 `NotExport` 子树内部的设计说明和展示内容后，正式结构为：

```text
ServerList                          # 界面顶层 Frame -> UIRoot
  NotExport_DesignGuides
  NotExport                        # 弹窗覆盖的场景背景；整棵子树不导出
    Background
    SystemMenu
    ServerSelectPanel
  uimask                           # 精确特殊 Role；仅记录遮罩色元数据
  NotExport                        # visible=false；保留原 CommonDialog Instance
    CommonDialog
  DialogBackground                 # 1011 × 1625；Clip content；PNG Export 1x
  CommonDialog                     # 无 Fill、无 Export的结构容器
    Title                          # 当前唯一活动子节点；保持独立 Label

  ServerItems                      # Vertical Auto Layout -> List；Clip content
    Button_ServerItem              # 唯一正式 Item 模板
      image_bg                     # 首子节点；558 × 100；Color Fill + PNG Export 1x
      ServerName                   # Text -> Label
      ServerStatus                 # PNG Export 1x -> Image

    NotExport_DemoItem01
    NotExport_DemoItem02
    NotExport_DemoItem03
    NotExport_DemoItem04
    NotExport_DemoItem05
    NotExport_DemoItem06
    NotExport_DemoItem07
    NotExport_DemoItem08
    NotExport_DemoItem09

  ScrollBar_ServerListV            # ServerList 直属节点；10 × 1108
    Background                     # Rectangle -> Image；绑定 Track
    Handle                         # Rectangle -> Image；10 × 232，贴顶

  CategoryPanel                    # Rectangle + PNG Export 1x -> Image
  CategoryOutline                  # Rectangle + PNG Export 1x -> Image

  ServerCategories                 # Vertical Auto Layout + Clip content -> ScrollView
    NotExport + Toggle_Recommended
    Toggle_Recommended             # 显示顺序第一项，唯一正式分类 Item
      selected                     # Rectangle + PNG Export 1x
      CategoryText
    NotExport_Toggle_MyServersPrimary
    NotExport_Toggle_Internal01
    NotExport_Toggle_Internal02
    NotExport_Toggle_MyServersSecondary
    NotExport_Toggle_Internal03
    NotExport_Toggle_Internal04
    NotExport_Toggle_Internal05
    NotExport_Toggle_Internal06

  Container_ServerStatusLegend
    StatusFull                     # 普通 Frame -> Container
      StatusFullText               # Text -> Label
      StatusFullIcon               # PNG Export 1x -> Image
    StatusBusy
      StatusBusyText
      StatusBusyIcon               # PNG Export 1x -> Image
    StatusSmooth
      StatusSmoothText
      StatusSmoothIcon             # PNG Export 1x -> Image
    StatusMaintenance
      StatusMaintenanceText
      StatusMaintenanceIcon        # PNG Export 1x -> Image

  NotExport_ReferenceImage
```

### 4.1 前缀精简原则

本次继续按“能稳定自动推断就省略，行为和特殊导出语义必须显式保留”清理名称。

已去掉的前缀：

| 原名称 | 当前名称 | 自动推断依据 |
| --- | --- | --- |
| `UIRoot_ServerList` | `ServerList` | “界面”直属顶层 Frame 自动成为 UIRoot |
| `ScrollView_ServerList` | `ServerItems` | Vertical Auto Layout Frame 自动成为 List；该节点确实是运行时列表 |
| `Reference_CommonDialog` | `CommonDialog` | 无法命中 Common 后已 Detach，按普通 Frame 处理 |
| `Label_*` | 业务名或 `*Text` | Figma Text 自动成为 Label |
| `Image_*` | 业务名、`Background`、`Handle` 或 `*Icon` | Rectangle/Image 自动成为 Image；PNG Export 还会强制 Image Render Policy |
| `Container_Status*` | `Status*` | 这些 Frame 的 `layoutMode=NONE`，默认自动成为 Container |

仍保留的显式 Role 名称或前缀：

| 前缀 | 保留原因 |
| --- | --- |
| `Button_ServerItem` | 普通 Frame 和子节点外观不能自动推断 Button |
| `ScrollBar_ServerListV` | Background/Handle 层级不能自动推断 Scrollbar |
| `ServerCategories` | Clip content 优先表达 ScrollView，禁止增加 `ToggleGroup_` |
| `Toggle_Recommended` | ScrollView 显示顺序中的第一个普通 Item，作为唯一正式分类 Toggle 模板 |
| `Container_ServerStatusLegend` | 这是 Horizontal Auto Layout 的静态排版容器；去掉会被误推断为 List |
| `uimask` | UiMask 是特殊元数据语义，必须使用插件可识别的保留名称 |
| `NotExport_*` | 必须显式压制自身及整个子树 |

ScrollBar 内的 `Background / Handle` 虽去掉了 `Image_`，仍保留了复合控件绑定所需的职责名，因此不会影响 Track 和 Handle 查找。

### 4.2 `uimask` 的实际含义

`2:122` 已按参考目标结构恢复为精确名称 `uimask`。它保持一个可见纯色 Fill，且没有设置 PNG Export，这是符合插件完整加载流程的。导出器对保持为 `UiMask` Role 的节点使用特殊策略：

- 不生成 Sprite；
- 不生成 GameObject 或 Unity `Image`，并压制其子树；
- 将节点 ID、名称和 Fill 颜色写入最近可导出父节点的 `FigmaNode.UiMasks`；当前归属为 `ServerList` 根；
- Fill 必须恰好是一个可见纯色 Fill，否则跳过并给出警告；
- 最终 alpha 为 `color.a × fill.opacity × node.opacity`。当前节点的 Fill opacity 为 `0.75`、节点 opacity 为 `1` 时，记录的 alpha 为 `0.75`。

不能给 `uimask` 设置 PNG Export。加载器先按名称识别 Role，随后对所有“带 PNG 且 Role 不是 `Mask`”的节点统一改写为 `Image`；因此 `UiMask + PNG` 不会继续走元数据策略，而会被错误导出为普通图片。

因此，它只适合表达供业务代码读取的“遮罩颜色元数据”。插件仓库目前没有自动消费 `UiMasks` 并创建可见蒙层的运行时代码。若 `Backdrop` 需要在 Prefab 中直接显示为半透明暗层，应改用普通 Rectangle/Image 视觉节点，并视视觉复杂度选择纯色 UGUI Image 或 PNG Export；若要裁剪后代，则应使用 `Mask_*`，不能使用 `UiMask_*`。

### 4.3 弹窗覆盖背景的 NotExport 整理

发现 `uimask` 后，不能默认界面本身就是全屏正式 View。应继续寻找遮罩上方的弹窗背景根：优先识别尺寸小于 UIRoot 的图片节点，或承担弹窗背景的 Component Instance。本例中 `CommonDialog` 是 `1011 × 1625` 的非全屏组件，属于弹窗背景根。

参考选服目标结构，将位于弹窗层下方、只用于还原弹窗背后场景的视觉节点收进新建的全屏 `NotExport` Frame：

```text
NotExport                         # 16:37，1080 × 2400
  Background                     # 2:121
  SystemMenu                     # 2:123
  ServerSelectPanel              # 2:124，全屏参考视觉
uimask                            # 2:122，仍与 NotExport 同级
NotExport                         # 23:86，visible=false
  CommonDialog                    # 2:125，原远程 Instance
CommonDialog                      # 23:62，Detach 后的普通 Frame
```

`NotExport_DesignGuides`、`NotExport_ReferenceImage` 以及列表中的 `NotExport_DemoItem*` 已有压制前缀，不再额外包裹或改名。正式弹窗内容，例如服务器列表、分类 Toggle、Scrollbar 和状态图例，位于弹窗背景上方并需要生成 UGUI 节点，也不能因为几何上落在弹窗范围内而放入 NotExport。

## 5. ScrollView / ScrollBar 处理

### 5.1 列表根

`2:126` 已从 `Frame 315` 整理为 `ServerItems`：

- `layoutMode = VERTICAL`；
- `clipsContent = true`；
- 尺寸从 `558 × 1087` 调整为 `586 × 1108`，为右侧 ScrollBar 纳入根边界；
- Y 从 `607` 调整为 `586`；
- `paddingTop = 21`，因此第一个 Item 的屏幕绝对位置保持不变；
- 十个 Item 的宽度均固定回 `558`，避免列表扩宽后被 Stretch 到 `586`。

`ServerItems` 是 Vertical Auto Layout Frame，因此即使没有 `List_ / ScrollView_` 前缀也会自动推断为 List。配合 `Clip content`，导出器会自动创建 Unity 的 `Viewport / Content`，Figma 中不手工补这两层。

### 5.2 Item 模板

`2:127` 是唯一正式模板：

```text
Button_ServerItem
  ServerName
  ServerStatus
```

其余九个 Item 保持原视觉位置，但根名称改为 `NotExport_DemoItem01~09`。导出器会压制整棵副本子树，因此不会把十份视觉数据全部注册成运行时模板。

原稿文字使用 `FZCuYuan-M03S Regular`，当前 Figma MCP 环境没有该字体，跨父节点移动含这些文字的 Frame 会被 Figma 拒绝。因此没有强行把九个副本移动到一个新的父 Frame，而采用逐项 `NotExport`；两种结构对导出器的模板过滤结果相同，且当前方案不会破坏字体和视觉。

### 5.3 ScrollBar

新增 `7:37 ScrollBar_ServerListV`，并将原 Track `2:182`、Handle `2:183` 移入其中：

```text
ServerList
  ServerItems                       # 只保存 Item 模板
  ScrollBar_ServerListV             # 与 ServerItems 同级
    Background                       # 10 × 1108
    Handle                           # 10 × 232，y=0
```

预期 Unity 配置：

- `Scrollbar.Direction = TopToBottom`；
- 初始 `value = 0`；
- `size = 232 / 1108 ≈ 0.2094`；
- 自动生成 `Sliding Area`；
- 自动绑定到 `ScrollRect.verticalScrollbar`。

## 6. Clip content / Toggle 处理

左侧节点 `2:159` 本身是 `Vertical Auto Layout + clipsContent=true`，因此容器名称改为英文 `ServerCategories`，由结构自动推断为 ScrollView。即使它的子节点是 Toggle，也不能再使用 `ToggleGroup_*`；裁剪滚动语义优先于子节点的控件类型。

九个远程 Variant Instance 在当次用户提供的 Common 模块 URL 中无法按 key、id 或唯一标准化名称找到对应定义，均已先按普通引用流程 Clone + Detach。随后按 ScrollView 模板规则，只保留 Auto Layout 显示顺序第一项：

```text
ServerCategories
  NotExport                 # visible=false
    OriginalToggleInstance
  Toggle_Recommended        # y=0，唯一正式 Item
  NotExport_Toggle_*        # 后续八个 Detach 展示节点
```

顺序依据为 `layoutMode=VERTICAL`、`primaryAxisAlignItems=MIN`、`itemSpacing=31` 和各流式子节点的 y 顺序。隐藏的原 Instance 备份、已有 `NotExport` 节点以及 Absolute 子节点不参与排序。后八个 Detach Frame 保持可见和原 Auto Layout 位置，仅通过 `NotExport_` 前缀排除导出，因此设计稿展示不变。滚动容器只负责 ScrollView/Viewport/Content，不生成 Unity `ToggleGroup`，互斥选择如有需要应由业务列表逻辑处理。

## 7. PNG Export 设置

整理前曾给以下静态视觉叶节点增加一条 `PNG / 1x / 无 suffix` Export：

- `Background`；
- `SystemMenu`；
- `ServerSelectPanel`；
- `CategoryPanel`；
- `CategoryOutline`；
- `DialogHeaderLeft`、`DialogHeaderCenter`、`DialogHeaderRight`；
- `ScrollBar_ServerListV/Background`、`Handle`；
- `Toggle_Recommended/selected`；
- 正式服务器 Item 的 `ServerStatus`；
- 四个状态说明图标。

没有在 `UIRoot`、`ScrollView`、`ToggleGroup`、`Button`、`ScrollBar` 或其他行为/布局根上设置 PNG Export。Export 设置在 Rectangle 视觉叶节点上；ScrollBar 根保持行为结构，只有 Track/Handle Rectangle 设置 Export。

`uimask` 没有设置 PNG Export。它不是图片导出源，并且 PNG Export 的后处理会把它从 `UiMask` 改写为 `Image`，因此必须保持无 Export。`Background / SystemMenu / ServerSelectPanel` 目前已经位于 `NotExport` 子树，不再作为 Unity 图片产物；其 Export 设置仅保留 Figma 侧原始信息，不应计入正式导出图片清单。其他上述节点仍是最终静态视觉叶节点。

## 8. 英文命名检查

最终正式 View 树按以下边界检查：

- `NotExport_*` 子树不进入产物；
- 带 PNG Export 的叶节点作为一张 Sprite，不继续导出其内部实例层；
- Figma Instance 先检查 PNG Export：带 PNG 的原 Instance 直接作为图片源保留；只有无 PNG 且无法命中 Common 的引用，才保存 `NotExport` 备份并使用 Detach 后的普通结构。

按此边界检查，当前 View 的正式导出节点名不含中文。Text 的显示内容仍可使用中文，不影响 GameObject 命名。

所有“无 PNG Export 且未命中 Common”的活动引用都已处理：先 Clone，再 Detach 副本，把原 Instance 放入 `NotExport` 且设置 `visible=false`。递归处理范围包含 `CommonDialog`、九个分类 Toggle、弹窗副本内部九个嵌套引用，以及按钮内部继续发现的两个 `ActionIcon` 引用。`ServerStatus` 和四个状态图标本身已有 `PNG / 1x` Export，因此保留原 Instance 直接按图片导出，不参与 Common/Detach 流程。

早期 Detach 产生的 14 个 Page 顶层孤立 Frame 已全部归位，Page 顶层目前只有 `1:2 界面` Section。五个孤立 Frame 内使用的 `FZCuYuan-M03S Regular` 在当前 Figma MCP 环境不可用，无法直接跨父级移动；处理时保留隐藏原 Instance 作为无损备份，在普通副本归位后以 `Noto Sans SC Regular` 重建了“一键放入”“交付”和三个数量 `1` 的文字层，其他字号、颜色、尺寸和对齐参数沿用原值。

## 9. 验证结果

- Page 可解析模块名：`Test`；
- Page 直属 Section 只有中文 `界面`；空的 `Coms / Res` 已删除；
- Page 顶层除 `界面` Section 外没有散落 Frame；
- `ServerList` 位于“界面”顶层，可自动推断 UIRoot，尺寸为 `1080 × 2400`；
- `ServerList` UIRoot 的白色 Color Fill 已移除，UIRoot 只保留尺寸、裁剪和层级职责；
- `CommonDialog` 的裁剪静态视觉已复制为同位置、同尺寸的 `48:16 DialogBackground`，并设置 PNG Export 1x；副本中的 `Title` 已隐藏，避免文本烘进 Sprite；
- `DialogBackground` 是该视觉副本唯一的 PNG 源；克隆时继承的 `48:18~48:20 DialogHeader*` 后代 Export 已清除，避免父子重复出图；
- 原 `23:62 CommonDialog` 保持 `clipsContent=true`，已移除 Fill，并隐藏 Header、Vector、Divider 等静态分支，当前只保留活动的 `23:68 Title`；
- `Title` 使用当前环境缺失的 `FZCuYuan-M03S Regular`，无法安全跨父级移动，因此采用“导出视觉副本 + 原结构保留 Label”的等价方案，未替换字体；
- `Button_ServerItem` 的原 Color Fill 已移动到首子节点 `37:17 image_bg`；`ServerName` 与带 PNG Export 的 `ServerStatus` 仍独立导出；
- `Button_ServerItem/image_bg` 与父节点等大、四向 Stretch，并已设置 PNG Export 1x；`ServerList`、`CommonDialog` 两个结构根均无 Fill；
- 对 `ServerList` 正式活动树执行“非 Text + 可见 Color Fill + 无 PNG Export”审计后，唯一遗漏为 `37:17 image_bg`，现已补齐；`uimask`、`NotExport`/隐藏子树和 `DialogBackground` 整体 Export 的后代按规则排除；
- `ServerItems` 保持 Vertical Auto Layout 和 Clip content，可自动推断 List；
- 只有 `Button_ServerItem` 是正式列表模板；
- `ScrollBar_ServerListV` 已从 `ServerItems` 移出，现为 `ServerList` 的直属节点，不参与列表 Item 模板识别；
- ScrollBar Track/Handle 尺寸、对齐和比例保持原值，两个 Rectangle 均已设置 PNG Export 1x；
- `ServerCategories` 保持 Vertical Auto Layout 和 Clip content，导出为 ScrollView，不生成 ToggleGroup；
- 分类 ScrollView 只保留显示顺序第一项 `Toggle_Recommended`；后八个 Detach Frame 已改为 `NotExport_Toggle_*`，九个原 Instance 仍分别保存在不可见 `NotExport` 中；
- `23:92 selected` 及正式树中其他普通 Rectangle 均已设置 PNG Export 1x；`uimask` 和 `NotExport` 子树明确排除；
- 正式导出树中有 5 个活动 Instance，均为带 PNG Export 的图片源；活动的非图片 Instance 审计结果为 0；
- 正式导出树中的中文 Layer name 审计结果为 0；
- 弹窗覆盖的场景背景已归入新建的 `NotExport`，`uimask` 与弹窗正式结构保持为其后续同级节点；
- 正式导出边界内没有中文 Layer name；
- 整理前后截图已回归，最终列表 Item 宽度锁回原始 `558 px`，视觉保持一致。

## 10. Unity 侧建议验证

实际导出后重点检查：

1. 生成路径是否为 `Prefabs/Test/ServerList.prefab`；
2. `ScrollRect.verticalScrollbar` 是否绑定 `ScrollBar_ServerListV`；
3. ScrollBar 是否为 `TopToBottom`，`size` 是否约为 `0.2094`；
4. 列表模板池是否只有 `Button_ServerItem`；
5. `ServerCategories` 是否生成 ScrollView，且没有错误生成 `ToggleGroup`；
6. 分类 ScrollView 的模板池是否只有 `Toggle_Recommended`，后八个 `NotExport_Toggle_*` 是否完全未进入 Prefab；
7. `NotExport_*` 子树是否完全未进入 Prefab；
8. Prefab、GameObject 和 Sprite 文件名是否全部为英文；
9. 所有 `NotExport` 隐藏备份是否完全未进入 Prefab，正式树中是否不存在未解析的 Reference；
10. 若业务确实使用遮罩元数据，`ServerList` 根的 `FigmaNode.UiMasks` 是否包含 `uimask`，颜色及 alpha 是否正确；若期望直接显示暗层，应确认已改为普通 Image，而不是依赖 `UiMask` 自动显示。
