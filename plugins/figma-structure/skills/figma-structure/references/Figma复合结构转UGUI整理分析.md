# Figma 复合结构转 UGUI 整理分析

## Contents

- [1. 背景与分析范围](#1-背景与分析范围)
- [2. 核心结论](#2-核心结论)
- [3. 转换器的主要识别逻辑](#3-转换器的主要识别逻辑)
  - [3.1 导出范围](#3-1-导出范围)
  - [3.2 Role 识别](#3-2-role-识别)
  - [3.3 结构与视觉分离](#3-3-结构与视觉分离)
- [4. 复合控件的标准结构](#4-复合控件的标准结构)
  - [4.1 Button](#4-1-button)
  - [4.2 Toggle](#4-2-toggle)
  - [4.3 ToggleGroup](#4-3-togglegroup)
  - [4.4 Input](#4-4-input)
  - [4.5 ProgressBar](#4-5-progressbar)
  - [4.6 Slider](#4-6-slider)
  - [4.7 ScrollBar](#4-7-scrollbar)
- [5. List 与 ScrollView](#5-list-与-scrollview)
  - [5.1 推荐结构](#5-1-推荐结构)
  - [5.2 Item 模板](#5-2-item-模板)
  - [5.3 ScrollBar 层级](#5-3-scrollbar-层级)
- [6. 选服原始结构分析](#6-选服原始结构分析)
- [7. ToUGUI 目标结构分析](#7-tougui-目标结构分析)
  - [7.1 ServerList 命名不够稳健](#7-1-serverlist-命名不够稳健)
  - [7.2 ServerTypeList 的 Item 不会自动成为 Toggle](#7-2-servertypelist-的-item-不会自动成为-toggle)
  - [7.3 ScrollBar_v 层级不正确](#7-3-scrollbar_v-层级不正确)
  - [7.4 server_status_list 可能被误判为 List](#7-4-server_status_list-可能被误判为-list)
  - [7.5 SecondPopView_S7 不是理想的公共组件引用](#7-5-secondpopview_s7-不是理想的公共组件引用)
- [8. 建议的选服最终结构](#8-建议的选服最终结构)
- [9. 建议实施步骤](#9-建议实施步骤)
- [10. 导出验收清单](#10-导出验收清单)
  - [10.1 Figma 文档](#10-1-figma-文档)
  - [10.2 Unity Prefab](#10-2-unity-prefab)
  - [10.3 SuperScrollView（如启用）](#10-3-superscrollview-如启用)


## 1. 背景与分析范围

本文用于记录：如何将普通 Figma 展示稿整理成可由 `com.matrix889.figma-to-ugui` 稳定导出为 Unity UGUI Prefab 的语义结构。

分析参考：

- 复合结构示例
- 选服原始结构
- 项目历史中与选服界面对应的 `ToUGUI（Login）/界面/ServerLlistView` 目标结构

本文只讨论 Figma 文档整理和导出结构，不讨论选服界面的业务数据加载、网络协议或运行时数据源实现。

复合组件除了层级和命名，还需要满足 RectTransform、点击区域、Auto Layout、Mask 和 Unity 驱动几何约束，详见：[Figma复合组件几何布局与UGUI注意事项](./Figma复合组件几何布局与UGUI注意事项.md)。

Figma Export 只应设置在最终静态视觉源上，不能作为普通节点导出开关使用，详见：[Figma节点PNG Export设置规则分析](./Figma节点PNG-Export设置规则分析.md)。

除 Button、Toggle、List 等复合控件外，插件还支持 UIRoot、Container、Label、Image、Reference、Mask、UiMask 和 NotExport 等节点语义；其中哪些 Role 前缀可以省略，详见：[FigmaToUGUI导出节点类型总览](./FigmaToUGUI导出节点类型总览.md)。

Page 模块名的解析方式，以及 Views、Coms、Res Section 的组织规范，详见：[FigmaToUGUI模块与Page-Section导出规范](./FigmaToUGUI模块与Page-Section导出规范.md)。

## 2. 核心结论

普通设计稿不能仅通过批量重命名变成 UGUI 结构。需要把以视觉展示为目的的图层树，重构为以运行时控件语义为目的的节点树：

```text
导出范围 Section
  View 根
    业务容器
      交互控件根
        约定名称的视觉和文本子节点
      List 根
        运行时 Item 模板
        ScrollBar
      公共组件 Instance
```

转换器会综合以下信息决定输出：

1. Section 名称决定节点是否属于界面、组件或图片资源。
2. 节点名称决定 Button、Toggle、List、Input 等 UGUI 角色。
3. Figma 节点类型决定节点是文本、图片、容器、组件定义还是组件实例。
4. Auto Layout 决定 Unity LayoutGroup、方向、间距和 Padding。
5. `Clip content` 决定 List 是否生成 ScrollRect。
6. PNG Export 决定复杂视觉是否烘成 Sprite。
7. Component/Instance 身份决定节点是否能替换为 prefab instance。

因此，整理工作的目标是在尽量保持视觉效果不变的前提下，为每个运行时节点补齐明确、稳定、可验证的语义。

## 3. 转换器的主要识别逻辑

### 3.1 导出范围

推荐建立独立的 `ToUGUI` Page，并使用以下 Section：

```text
Page ToUGUI(ServerList)
  Section 界面
  Section 组件
  Section 图片
```

- `界面`：放正式 View；其顶层 Frame 通常导出为独立 Prefab。
- `组件`：放当前模块可复用的 Figma Component 定义。
- `图片`：放只生成 Sprite、不生成界面节点的资源。

不建议依赖旧的 `Export`、`导出` Section。

### 3.2 Role 识别

角色优先由节点名称匹配。例如：

| UGUI 角色 | 推荐命名 |
| --- | --- |
| Image | `Image_bg`、`Image_icon` |
| Label | `Label_title`、`Label_text` |
| Button | `Button_confirm`、`Btn_close` |
| Toggle | `Toggle_serverType` |
| ToggleGroup | `ToggleGroup_serverTypes` |
| List | `List_server`、`ScrollView_server` |
| Input | `Input_account` |
| ProgressBar | `ProgressBar_loading` |
| Slider | `Slider_volume` |
| ScrollBar | `ScrollBar_v`、`ScrollBar_h` |
| Container | `Container_panel` |
| NotExport | `NotExport_guides`、`Ignore_demo` |

没有显式角色时：

- Text 自动解析为 Label。
- Rectangle/Image 自动解析为 Image。
- Instance 自动解析为 Reference。
- 普通 Frame/Group/Component 默认是 Container。
- Auto Layout Frame 会自动推断为 List。
- `界面/Views` 下的顶层 Frame 可自动推断为 UIRoot。

需要注意：普通排版容器如果使用 Auto Layout，也可能被推断成 List。若它只是静态布局容器，应显式使用 `Container_xxx`。

### 3.3 结构与视觉分离

控件根只负责行为、范围和布局，视觉尽量放到约定子节点：

```text
Button_confirm
  Image_bg
    button_confirm_bg       # PNG Export
  Label_text
```

结构或交互节点不应依赖根 Frame 的复杂 Fill。当前自动生成 `BgImage@bg` 的兼容功能默认关闭，正式结构应显式创建 `Image_bg`。

UIRoot 只承担界面根尺寸、锚点和层级边界，不能直接设置 Color Fill。普通结构/交互 Frame 若自身有可见 Color Fill、没有 PNG Export，应按以下方式分离：

1. 在根节点内创建铺满父节点的首子节点 `image_bg`，复制原 Fill，并移除根节点 Fill；`image_bg` 自身存在可见 Color Fill 时，必须设置一条 `PNG / 1x / contentsOnly / 无 suffix` Export。
2. `image_bg` 与父节点保持相同宽高，通常使用四向 Stretch；圆角按视觉需要复制到 `image_bg`。
3. 父节点未开启 Clip content 时，现有 Text、Image、Button 等节点保持为 `image_bg` 后面的兄弟节点，原有遮挡顺序整体后移一位。
4. 父节点开启 Clip content 时，仍由结构根保留裁剪语义；`image_bg` 只承载背景视觉。Text、Button、Toggle、Input 等可识别 Role 必须从背景视觉子树中拆出，作为 `image_bg` 的兄弟节点，并保持原相对位置和前后层级。
5. 不得把动态文本或交互节点放进显式 Image 节点。插件对显式 Image 容器会压制其子树，这些节点否则不会独立生成 UGUI 对象。
6. 描边、圆角同时承担裁剪边界时可以继续保留在结构根；避免为了移动 Fill 而改变原来的 Clip content、圆角裁剪和描边外扩效果。

若开启 `Clip content` 的节点包含越出自身边界、必须按当前裁剪结果烘焙的装饰图形，则优先按“活动 Role 与静态裁剪图分离”处理：

1. 先识别 Text、Button、Toggle、Input、ScrollView Item 等不能被烘进图片的非 Image 节点，将它们提升到裁剪图片节点之外，并保持绝对位置、前后顺序和原尺寸。
2. Rectangle、Vector、Boolean Operation、复杂 Fill、描边和被裁剪的装饰效果留在静态视觉子树中。
3. 确认静态视觉子树不再包含需要独立导出的活动 Role 后，才给该静态视觉根设置一条 `PNG / 1x / contentsOnly / 无 suffix` Export。
4. 不得直接给仍同时承担布局、行为或动态内容职责的原结构根设置 Export；必要时复制出同尺寸的英文业务名视觉根，例如 `DialogBackground`，将活动 Role 在视觉副本中隐藏，再由视觉副本负责 PNG Export。
5. 原结构根应移除 Fill，并隐藏已经转入视觉副本的静态分支，只保留活动 Role，避免 Unity 中重复生成图片和 Figma 画布重复叠加。

跨父级移动 Text 或包含 Text 的 Frame 前必须加载其现有字体。若源字体在当前 Figma 环境不可用，不能强行换字体或破坏文本；应采用上一条的“导出视觉副本 + 原结构保留活动 Role”方案。视觉副本与原结构重叠放置，副本位于原结构之前，最终画面保持不变。

若节点本身已有受支持的 PNG Export，则继续按图片源优先规则处理，不再额外拆 `image_bg`。正式导出树中的非 Text 节点只要自身保留可见 Color Fill，就必须作为图片源设置 PNG Export；行为/布局根不能直接 Export 时，应先把 Fill 分离到 `image_bg` 等静态视觉节点，再给视觉节点设置 Export。UIRoot、`uimask`、`NotExport` 子树以及已被祖先整体 PNG Export 覆盖的后代不进入这项审计。

复杂 Rectangle、Vector、Boolean Operation、描边和光效，应根据运行时需求决定：

- 需要动态控制的文本、图标和状态保留为真实节点。
- 纯装饰视觉包进稳定命名的 PNG Export Frame。
- 不同视觉资源不能都使用 `Rectangle`、`Vector`、`Image_bg` 等泛名作为最终导出名。

## 4. 复合控件的标准结构

复合结构示例表达的共同规则是：导出器先给根节点挂载行为组件，待子节点生成后，再查找并绑定 Graphic、文本和驱动区域。

### 4.1 Button

```text
Button_confirm
  Image_bg
  Label_text
```

导出结果：

- 根节点挂 `Button`。
- 点击范围由 Button 根 Frame 决定。
- `targetGraphic` 优先绑定 `targetgraphic`、`hitarea`、`target`、`Image_bg`、`background` 或 `bg`。
- 找不到有效背景时可能保留 Button，但产生缺少 targetGraphic 的警告。

### 4.2 Toggle

```text
Toggle_serverType @checked=on
  Image_bg
    server_type_normal
  Image_checked
    server_type_selected
  Label_text
```

导出结果：

- 根节点挂 `Toggle`。
- `Image_bg` 作为 `targetGraphic`。
- `Image_checked`、`Image_check`、`Image_on` 等作为 `graphic`。
- 选中态图片应与背景重叠，通常使用 Absolute positioning，不参与普通 Auto Layout 排列。

不能只创建名为 `checked` 的普通空容器，再把不可识别名称的 Graphic 放在其内部。最终需要有一个转换器可以识别并绑定的 Graphic。

### 4.3 ToggleGroup

```text
ToggleGroup_serverTypes
  Toggle_recommend
  Toggle_myServers
  Toggle_internal
```

ToggleGroup 会把后代 Toggle 绑定到同一个 Unity `ToggleGroup`。

只有业务明确需要互斥选中行为时才使用 ToggleGroup；普通入口按钮组仍应使用业务容器和 Button。

如果共同祖先开启了 `Clip content`，该节点应按 ScrollView/List 处理，即使直属或后代节点是 Toggle，也不能把裁剪容器命名为 `ToggleGroup_*`。此时只把容器改成稳定的英文业务名，保留 Auto Layout 与 Clip content，由结构推断滚动视图；各 Toggle 作为普通 Item 处理，互斥逻辑交给业务列表。

### 4.4 Input

```text
Input_account
  Image_bg
  Text Area
    Label_input_holder
    Label_text
```

导出结果：

- 根节点挂 `TMP_InputField`。
- `Image_bg` 作为 targetGraphic。
- `textarea`、`text area` 或 `viewport` 作为文本区域。
- `Label_input_holder`、`Label_placeholder`、`Label_holder` 作为提示文本。
- `Label_text`、`Label_value`、`Label_input` 优先作为输入内容文本。

也可以通过名称参数表达默认提示：

```text
Input_account @placeholder=请输入账号
```

### 4.5 ProgressBar

```text
ProgressBar_loading
  Image_bg
  Image_fill
```

`Image_fill` 会被配置为 Filled Image。若缺失，则退回根 Image；根 Image 也不存在时产生警告。

### 4.6 Slider

```text
Slider_volume
  Image_bg
  Fill Area
    Image_fill
  Handle Slide Area
    Image_handle
```

导出器会绑定：

- `Image_fill` 到 `Slider.fillRect`。
- `Image_handle`、`handle`、`thumb` 或 `knob` 到 `Slider.handleRect`。
- Handle Graphic 到 `Slider.targetGraphic`。

当前方向固定为 `LeftToRight`。

### 4.7 ScrollBar

选服文档中的 ScrollBar_huadong 是标准纵向 ScrollBar 样例：

```text
ScrollBar_huadong              10 × 1108
  Image_bg                     10 × 1108
    serverlist_img_cemianxian
  Image_handle                 10 × 232，贴顶
    serverlist_img_handle
```

导出器会根据该几何推导：

- 根高大于宽，方向为纵向。
- Handle 位于顶部，Direction 为 `TopToBottom`。
- 初始 `value = 0`。
- `size = 232 / 1108 ≈ 0.2094`。
- Handle 与 Track 同宽，左右 inset 均为 0。
- 自动在根与 Handle 之间增加 Stretch 铺满的 `Sliding Area`，Figma 不需要手工创建这一层。

ScrollBar 用在 List 时还有两项刚性要求：

1. 必须是对应 List/ScrollView 根的直接子节点，才能绑定到 `ScrollRect.verticalScrollbar` 或 `horizontalScrollbar`。
2. List 使用 Auto Layout 时，ScrollBar 应设置为 Absolute positioning，不能作为普通 Item 参与 Content 排列。

虽然 `ScrollBar_huadong` 可以通过宽高比正确推断为纵向，为避免后续改尺寸导致方向变化，推荐命名为：

```text
ScrollBar_serverList_v
```

完整的 Handle 比例、Viewport 占位和运行时变化规则见：[Figma复合组件几何布局与UGUI注意事项](./Figma复合组件几何布局与UGUI注意事项.md#101-选服-scrollbar_huadong-实例分析)。

## 5. List 与 ScrollView

### 5.1 推荐结构

```text
ScrollView_server
  Button_serverItem
    Image_bg
    Label_serverName
    Image_serverStatus

  NotExport_demoItems

  ScrollBar_v
    Image_bg
    Image_handle
```

List 根要求：

- 使用 `List_xxx` 或 `ScrollView_xxx` 显式命名。
- 使用 Auto Layout 表达方向、Padding 和 Item spacing。
- 开启 `Clip content`，根节点尺寸就是最终可见视口尺寸。
- 直接子节点主要是运行时 Item 模板。

不应在 Figma 中手写以下 Unity 层级：

```text
Viewport
  Content
```

开启 `Clip content` 后，导出器会自动生成：

```text
ScrollView_server           # ScrollRect
  Viewport                  # Image + Mask
    Content
      Button_serverItem
  ScrollBar_v
```

### 5.2 Item 模板

展示稿中常放置大量重复 Item 用于确认视觉效果。对于由 `Auto Layout + Clip content` 识别成 ScrollView/List 的节点，正式导出结构只保留显示顺序中的第一个普通流式 Item，其余展示 Item 必须增加 `NotExport_` 前缀：

```text
ScrollView_server
  Button_serverItem         # 正式模板
  NotExport_demoItems       # 其余展示实例
```

如果启用 SuperScrollView，List 的所有可导出直接子节点都会进入 Item 模板池。重复展示 Item 若没有放入 `NotExport`，可能产生大量模板、同名模板冲突或错误的运行时对象池配置。

“第一个”不能只按 Layer 面板肉眼判断，应结合 Auto Layout 计算：

1. 先排除 `visible=false`、已有 `NotExport` 前缀和 `Absolute positioning` 的子节点；
2. Vertical 按主轴上的 y/Auto Layout 顺序取第一项，Horizontal 按 x/Auto Layout 顺序取第一项；Grid/Wrap 先按行再按列；
3. Padding、Item spacing 和主轴对齐方式用于确认实际显示顺序，不能把位于 padding 前的绝对定位装饰当成 Item；
4. Scrollbar 不应放进承担 Item Auto Layout 的列表内容容器；若原稿中暂时位于其下，应先移到最近的 ScrollView/业务结构父节点。悬浮按钮、装饰等 Absolute 子节点也不参与“只保留第一个 Item”的筛选；
5. 后续 Item 改为 `NotExport_<原英文名>`，保留可见状态和 Auto Layout 位置，以便设计稿仍显示完整列表。

### 5.3 ScrollBar 层级

Figma 侧承担 Item Auto Layout 与 Clip content 的列表节点只保存 Item 模板，ScrollBar 不放在该列表内容节点里面。ScrollBar 应与列表节点同级，位于能够同时表达两者归属关系的最近业务结构父节点下：

```text
正确：
ServerPanel
  ServerItems
    Button_serverItem
  ScrollBar_v

错误：
ServerItems
  Button_serverItem
  ScrollBar_v
```

这样可以避免 ScrollBar 被 Auto Layout、首 Item 筛选或 SuperScrollView 模板池误当成列表内容。ScrollBar 使用英文业务名并保持独立复合结构；需要贴靠列表边缘时，按共同父节点坐标设置位置，不通过把它塞进 Content 来对齐。

## 6. 选服原始结构分析

原始节点 `1:970 服务器选择` 是典型的视觉展示结构：

- `1:1009 Frame 315` 中平铺了 10 个服务器 Item。
- `1:1042 Frame 316` 中平铺了 9 个服类型实例。
- `1:1008 通用弹窗`、背景、标尺、垫图、滚动条等都是按视觉位置并列。
- `item`、`Frame 315`、`Frame 316`、`状态` 等名称没有稳定 UGUI 语义。
- 结构没有明确表达哪些是运行时模板、哪些只是展示数据。

这种结构可以作为设计评审稿，但不适合直接转成可维护的 UGUI Prefab。

## 7. ToUGUI 目标结构分析

项目历史中的目标结构已经进行了以下关键整理：

```text
ToUGUI（Login）
  界面
    ServerLlistView
      NotExport
      uimask
      main
        SecondPopView_S7
        ServerList
          btn_item
          NotExport
        login_serverlist_img_cemianbg
        login_serverlist_img_cemianxian
        ServerTypeList
          item
          NotExport
        server_status_list
        ScrollBar_v

  图片
    login_serverlist_img_cemianbg
    ...
```

相对原始稿，它已经完成：

- 将正式界面放入 `界面` Section。
- 将设计辅助层和展示副本放入 `NotExport`。
- 将大量重复 Item 收敛为单个正式模板。
- 将独立图片放入 `图片` Section。
- 添加 `ScrollBar_v` 等可识别名称。
- 保持整理前后的视觉表现基本一致。

但按照当前转换器逻辑，仍有几项值得继续修正。

### 7.1 ServerList 命名不够稳健

`ServerList` 不以 `List` 或 `ScrollView` 开头。它只有在自身确实是 Auto Layout Frame 时才会被自动推断为 List。

为降低设计属性变更带来的角色漂移，建议改成：

```text
ScrollView_server
```

或：

```text
List_server
```

### 7.2 ServerTypeList 的 Item 不会自动成为 Toggle

当前正式模板名称是 `item`，内部是 `checked/uncheck` 容器。这不会自动生成 Unity Toggle。

如果左侧服类型需要互斥选择行为，建议改为：

```text
ScrollView_serverType
  Toggle_serverType
    Image_bg
    Image_checked
    Label_text
```

如果服类型固定且不需要滚动，可改成 ToggleGroup。

如果业务层计划把它当成普通列表模板并自行实现状态切换，则可以保留普通 Item，但需要明确这是有意的业务约定，而不是自动 UGUI Toggle。

### 7.3 ScrollBar_v 层级不正确

目标结构中的 `ScrollBar_v` 位于 `main` 下，与 `ServerList` 同级。当前自动绑定逻辑要求它位于 List 根下。

建议移动为：

```text
ScrollView_server
  Button_serverItem
  ScrollBar_v
```

### 7.4 server_status_list 可能被误判为 List

如果 `server_status_list` 是 Auto Layout Frame，它会被自动推断为 List。该区域实际是静态状态图例，不需要 ScrollRect 或动态模板语义。

建议显式命名：

```text
Container_serverStatusLegend
```

### 7.5 SecondPopView_S7 不是理想的公共组件引用

目标 metadata 中的 `SecondPopView_S7` 是本地 Component 定义，不是 Common 组件库的真实 Instance。

当前代码已经支持通过 ComponentKey、ComponentId、NodeId 和唯一名称向本地 Common prefab 降级匹配，但推荐做法仍是：

1. 先从公共组件文档导出 `SecondPopView_S7.prefab`。
2. 业务 Figma 文档中放置公共组件库的真实 Instance。
3. 不要把 Component 定义本体直接放进 View。
4. 不要把公共弹窗内部节点复制展开到业务 View。

普通业务导出不会因为 Common prefab 缺失而自动创建它；缺失时只能降级展开结构。

Figma 整理侧必须区分“确认命中”“完整索引确认无匹配”和“证据不足未解析”，并遵守主文件的 Common fallback 与授权边界：

- 节点自身已有受支持的 PNG Export：优先按图片源保留原节点，跳过 Common 匹配、`NotExport` 包装和 Clone/Detach；
- 能按 ComponentKey、ComponentId、本地化 NodeId 或唯一标准化名称命中 Common：已有正确真实 Instance 就保留；需要替换的本地节点才保留不可见备份，并在原位创建公共库真实 Instance；
- 本轮完整 Common 索引确认无匹配，且 Clone + Detach 在授权计划内：Clone 原 Instance，Detach 新副本，把原 Instance 放入名为 `NotExport` 且 `visible=false` 的 Frame；新副本按普通节点流程继续整理；
- 索引不可访问、身份缺失或匹配歧义：保留原节点并报告 unresolved，不以 Detach 代替身份核验；用户要求 Common 复用时也不能默默改走 Detach；
- Detach 后递归检查副本内部残留 Instance，每个仍按上述身份和授权条件处理；证据不足的引用保留并报告，不为清空警告而无限递归 Detach；
- 组件处理不能止于把原节点改名或包进 `NotExport`；除整棵子树本来就不导出的展示/说明内容外，隐藏原引用必须有正式替代节点；
- 各路径都必须保持父级、兄弟顺序、位置、尺寸和 Auto Layout sizing，不能按视觉相似度猜测公共组件。

## 8. 建议的选服最终结构

```text
Page ToUGUI(ServerList)
  Section 界面
    ServerListView
      NotExport_designGuides

      uimask

      MainPanel
        SecondPopView_S7             # Common Component Instance

        ScrollView_server            # Vertical Auto Layout + Clip content
          Button_serverItem
            Image_bg
              login_serverlist_btn_bg
            Label_serverName
            Image_serverStatus

          NotExport_demoItems

          ScrollBar_v                # Absolute positioning
            Image_bg
            Image_handle

        Image_serverTypePanel
          login_serverlist_img_cemianbg

        ScrollView_serverType
          Toggle_serverType
            Image_bg
            Image_checked
            Label_text

          NotExport_demoTypes

        Container_serverStatusLegend
          Status_full
            Image_icon
            Label_text
          Status_crowd
          Status_smooth
          Status_maintenance

  Section 组件
    Button_serverItem                # 仅在需要独立组件 Prefab 时放这里
    Toggle_serverType

  Section 图片
    login_serverlist_btn_bg
    login_serverlist_img_cemianbg
    login_serverlist_img_cemianxian
    server_status_full
    server_status_crowd
    server_status_smooth
    server_status_maintenance
```

这里的 `uimask` 沿用选服 ToUGUI 目标样例中的精确保留名称，只表示写入 `ServerListView` 根 `FigmaNode.UiMasks` 的纯色元数据，不会生成可见 Image，并且不能设置 PNG Export。若选服界面要求 Prefab 自带实际暗色蒙层，应将该节点改为普通 Image/Rectangle 视觉节点；若要求裁剪子节点，则改用 `Mask_*`。

`uimask` 同时也是“当前 View 可能是非全屏弹窗”的检查信号。应查找遮罩上方尺寸小于 UIRoot 的弹窗背景图片或组件，并将其下方仅用于展示场景的背景节点统一放入新建 `NotExport` Frame。参考结构为：

```text
ViewRoot
  NotExport                      # 弹窗后方的场景背景
  uimask
  PopupRoot                      # 非全屏图片或组件，以及正式弹窗内容
```

已有 NotExport 类前缀的节点无需重复处理；弹窗内部需要真实导出的控件也不能因为处于 PopupRoot 的覆盖范围内而被误归入 NotExport。

`Button_serverItem`、`Toggle_serverType` 是否放进 `组件` Section，应根据是否需要独立组件 Prefab 决定：

- 只作为当前 List 的模板：可以直接保留在 View 的 List 下。
- 被多个 View 重用或需要独立 prefab：制作成 Component，放入 `组件`，List 中使用 Instance。

## 9. 建议实施步骤

1. 将原始界面复制到独立 `ToUGUI(ServerList)` Page。
2. 若原结构已位于 `界面` Section，直接复用；否则创建 `界面`。`组件`、`图片` 只在确有对应产物时按需创建。
3. 将安全区、标尺、说明、垫图等辅助层放入 `NotExport_*`。
4. 公共弹窗能命中 Common 时保留或替换为真实 Instance；确认无匹配且已授权时才使用 Clone + Detach 的普通节点副本，证据不足时保留并报告。
5. 将服务器列表改为 `ScrollView_* + Auto Layout + Clip content`。
6. 只保留一个正式服务器 Item，其余展示 Item 放入 `NotExport`。
7. 左侧服类型已开启 Clip content 时固定整理为 ScrollView + Toggle Item，不得使用 ToggleGroup；只有无裁剪、无滚动语义时才考虑 ToggleGroup。
8. 将 ScrollBar 移动到对应 List 根下，并设置 Absolute positioning。
9. 将静态状态图例显式标记为 Container，避免被推断成 List。
10. 将复杂背景整理成 `Image_*` 语义容器和独立 PNG Export 节点。
11. 保留需要动态修改、本地化或数据绑定的真实 Label，不要烘进背景图。
12. 首次结构迁移使用 Rebuild 导出，并强制刷新 Figma 节点缓存。
13. 结构稳定后再使用 Incremental 或 Quarantine 更新。

不建议用 VisualOnly 完成此次迁移，因为它不会可靠处理 List、组件引用、层级和 ScrollView 后端等结构变化。

## 10. 导出验收清单

### 10.1 Figma 文档

- [ ] View 位于 `界面` Section 下。
- [ ] View 根是固定最终尺寸的 Frame。
- [ ] 所有正式节点使用稳定 ASCII 业务名或 Role 名。
- [ ] 辅助标注和展示副本位于 `NotExport` 子树。
- [ ] List 使用 Auto Layout，并在需要滚动时开启 `Clip content`。
- [ ] List 直接子节点主要是正式 Item 模板。
- [ ] ScrollBar 是 List 根的直接子节点。
- [ ] Toggle 有可识别的 `Image_bg` 和 `Image_checked` Graphic。
- [ ] 公共组件使用真实 Instance。
- [ ] PNG Export 节点名称唯一且具有业务语义。
- [ ] `Image_*` 语义节点下没有需要继续导出的 Button、Label 或复杂业务子树。
- [ ] Figma 源文档中没有手写 `BgImage@bg`。

### 10.2 Unity Prefab

- [ ] View 生成独立 Prefab。
- [ ] Button、Toggle、ToggleGroup 等组件挂在正确根节点。
- [ ] List 根生成 `ScrollRect`。
- [ ] 自动生成 `Viewport/Image/Mask` 和 `Content`。
- [ ] 正式 Item 位于 Content 下。
- [ ] Scrollbar 已绑定到 `ScrollRect.verticalScrollbar` 或 `horizontalScrollbar`。
- [ ] `SecondPopView_S7` 是本地 Common prefab instance。
- [ ] `NotExport` 展示节点未出现在最终 Prefab。
- [ ] 复杂背景使用正确 Sprite，尺寸和 RectTransform 一致。
- [ ] 导出日志没有未处理的“缺少 Image_bg”“缺少 Checkmark”“组件实例降级展开”等警告。

### 10.3 SuperScrollView（如启用）

- [ ] 只有 `List + Clip content` 节点进入 SuperScrollView 转换。
- [ ] 普通列表生成 `LoopListView2`，Grid/Wrap 生成 `LoopGridView`。
- [ ] 每个正式 Item 都进入模板池。
- [ ] 展示 Item 已被 `NotExport` 过滤。
- [ ] 模板名称唯一，业务代码使用最终模板名调用 `NewListViewItem`。
- [ ] 业务代码负责调用 `InitListView` 或 `InitGridView`，导出器不生成数据源。
