# Figma 节点 PNG Export 设置规则分析

## Contents

- [1. 先明确 Export 的含义](#1-先明确-export-的含义)
- [2. 插件实际识别规则](#2-插件实际识别规则)
  - [2.1 只识别 PNG](#2-1-只识别-png)
  - [2.2 Export scale](#2-2-export-scale)
  - [2.3 Export 会改变节点策略](#2-3-export-会改变节点策略)
- [3. 推荐的语义容器与视觉源分离](#3-推荐的语义容器与视觉源分离)
  - [3.1 Clip content 与越界视觉](#3-1-clip-content-与越界视觉)
- [4. 必须或强烈建议设置 Export 的节点](#4-必须或强烈建议设置-export-的节点)
  - [4.1 Rectangle 与复杂静态视觉](#4-1-rectangle-与复杂静态视觉)
  - [4.2 控件中的真实图片子节点](#4-2-控件中的真实图片子节点)
  - [4.3 独立 Sprite 资源](#4-3-独立-sprite-资源)
  - [4.4 Common 图片源](#4-4-common-图片源)
- [5. 不需要设置 Export 的节点](#5-不需要设置-export-的节点)
  - [5.1 不承担图片源职责的节点](#5-1-不承担图片源职责的节点)
  - [5.2 图片 Section 中的整节点资源](#5-2-图片-section-中的整节点资源)
  - [5.3 九宫格源](#5-3-九宫格源)
- [6. 不能设置 Export 的结构节点](#6-不能设置-export-的结构节点)
  - [6.1 Text 的特殊规则](#6-1-text-的特殊规则)
- [7. 各复合组件应设置 Export 的位置](#7-各复合组件应设置-export-的位置)
- [8. 选服 ScrollBar 的具体设置](#8-选服-scrollbar-的具体设置)
- [9. 复合结构 Demo 的具体建议](#9-复合结构-demo-的具体建议)
  - [Button](#button)
  - [Toggle](#toggle)
  - [Input](#input)
  - [ProgressBar](#progressbar)
  - [Slider](#slider)
- [10. 常见错误](#10-常见错误)
  - [10.1 Export 层级过高](#10-1-export-层级过高)
  - [10.2 Export 层级过低或切得太碎](#10-2-export-层级过低或切得太碎)
  - [10.3 动态文字被烘图](#10-3-动态文字被烘图)
  - [10.4 同时给语义容器和视觉子节点 Export](#10-4-同时给语义容器和视觉子节点-export)
  - [10.5 只写 `Image_xxx`，没有任何图片来源](#10-5-只写-image_xxx-没有任何图片来源)
  - [10.6 使用非 PNG Export](#10-6-使用非-png-export)
- [11. 最终判断流程](#11-最终判断流程)
- [12. 检查清单](#12-检查清单)
- [13. 自动勾选与漏勾复查](#13-自动勾选与漏勾复查)


> 本文讨论 Figma 右栏 PNG Export 的资源烘焙规则。它不同于插件的 `Image` Role 和 `Res` Section；相关节点类型见 [FigmaToUGUI导出节点类型总览](./FigmaToUGUI导出节点类型总览.md)。

## 1. 先明确 Export 的含义

Figma 右侧面板中的 Export 不是“让该节点参与 UGUI 导出”的通用开关。

对当前插件而言，设置 PNG Export 表示：

```text
把这个节点及其可见子树烘成一张 PNG
  -> 生成或复用 Sprite 资源
  -> 当前节点按 Image 处理
  -> 当前节点的子树不再生成独立 UGUI 节点
```

所以判断是否设置 Export 的核心问题不是“这个节点是否要出现在 Prefab 中”，而是：

> 这个节点及其后代，是否应该合并为一张不可再拆分的静态 Sprite？

如果答案是否定的，就不应设置 Export。

## 2. 插件实际识别规则

### 2.1 只识别 PNG

插件遍历节点的 `exportSettings`，只要存在一个 `format=PNG` 的设置，`HasPngExport` 就为 true。

- PNG：受支持。
- SVG、JPG、PDF 等：当前不会按对应格式导出，会产生“不支持非 PNG”警告。
- 同一节点存在多个 PNG 设置时，插件主要使用第一个 PNG 设置作为 `PrimaryPngExportSetting`。

推荐每个视觉源节点只保留一个明确的 PNG Export 设置。

### 2.2 Export scale

插件支持 Figma Export 的：

- `SCALE`：直接作为导出倍率。
- `WIDTH`：目标宽度 / 节点布局宽度，换算成倍率。
- `HEIGHT`：目标高度 / 节点布局高度，换算成倍率。

最终倍率限制在 `0.01–4`。通常 UI Sprite 使用 `1x`；确有高清资源规范时统一使用 `2x`，不要在同一模块随意混用倍率。

### 2.3 Export 会改变节点策略

除显式 `Mask` 外，节点一旦带 PNG Export，加载阶段会把它的 Role 改为 `Image`。这一步发生在名称 Role 解析之后，因此也会覆盖先前识别出的 `NotExport` 和 `UiMask`。

| Export 节点位置 | 导出结果 |
| --- | --- |
| Section 直接子节点/页面顶层 | 只导出 Sprite，不生成该节点 GameObject |
| View/Component 内的嵌套节点 | 生成一个 Image GameObject，绑定 Sprite，压制其后代 UI 节点 |
| 显式 `Image_*` 容器的 PNG 子节点 | 子节点仅作为 Sprite 源；最终由外层 `Image_*` 消费，子节点不单独生成 GameObject |
| 显式 `Mask` 节点 | 保留 Mask 结构，PNG 作为遮罩 Graphic 的 Sprite 来源 |
| `NotExport` / `UiMask` 节点误设 PNG | Role 会被改写为 Image，原有压制/元数据语义丢失；禁止这样设置 |

其中 `UiMask` 不是一种图片设置：在不带 PNG Export 时，它不生成 Sprite 或 GameObject，只把单一可见纯色 Fill 记录到最近可导出父节点的 `FigmaNode.UiMasks`。一旦增加 PNG Export，它会被改写成普通 Image。若需要 Prefab 中直接可见的蒙层，应一开始就使用正常 Rectangle/Image 名称，不要借用 `UiMask` 名称。

Component/Instance 的处理必须先检查 PNG Export，再检查 Common 组件身份。只要节点本身已有受支持的 PNG Export，它就是最终图片源：

- 保留原节点、原位置和图片源身份；已有合规 Export 跳过，配置 Export 的任务可纠正倍率、后缀或重复设置，但不转换组件身份；
- 不执行 Common key/id/名称匹配；
- 不创建 `NotExport` 备份，不 Clone/Detach，也不额外生成一个相同的图片副本；
- 节点内部即使来自远程组件，也会随整棵可见子树烘成 Sprite，不再需要 Prefab Reference。

若组件整理确实需要隐藏原引用，必须同时保留合法正式替代；只有设计说明、展示副本、弹窗背后的场景等整棵子树本来就不应导出的内容，才允许单独使用 `NotExport`。是否引用 Common、保留玩法内组件或使用已授权的 Detach 方案，遵循 `SKILL.md` 的组件归属与身份流程。未命中 Common 不等于允许 Detach；仅设置 Export 时不进入替换或 Detach 流程。

## 3. 推荐的语义容器与视觉源分离

最稳定的图片结构是：

```text
Image_bg                       # UGUI 语义和 RectTransform；自身无 Fill 时不设置 Export
  login_panel_bg               # 真正视觉源，设置 PNG Export
```

导出后：

```text
Image_bg                       # Unity Image，绑定 login_panel_bg.png
```

内层视觉源不会再生成独立 GameObject。

这样做的优点：

- `Image_bg`、`Image_checked`、`Image_fill`、`Image_handle` 等名字稳定服务于控件绑定。
- Sprite 文件名可以使用真实业务资源名。
- 外层 Rect 表达显示和交互尺寸；内层节点表达 PNG 的真实导出边界。
- 更容易区分布局边界和阴影、描边等可见边界。
- 后续替换视觉资源时，不必改变控件层级和引用名称。

如果 `Image_bg` 自身设置了可见 Color Fill，则必须直接给 `Image_bg` 设置 PNG Export；仅当它自身无 Fill、只作为语义容器并由内部视觉源提供图片时，才保持无 Export。无论采用哪种方式，都不要在父容器和视觉子节点上重复设置 Export。

正式 View/Component 导出树中的非 Text 节点只要自身保留可见 Color Fill，就纳入 PNG Export 审计。Button、Toggle、ScrollView 等行为/布局根若带 Fill，先将 Fill 拆到独立 `image_bg`，再给 `image_bg` 设置 Export，行为根本身保持无 Export。以下情况例外：

- UIRoot：禁止保留 Color Fill，应先拆出背景节点；
- `uimask`：禁止 PNG Export，否则会丢失 UiMask 元数据语义；
- `NotExport`、隐藏备份和展示说明子树；
- 已被祖先整体 PNG Export 覆盖的静态视觉后代，避免父子重复出图；
- 其他必须保留特殊 Role、不能转成普通 Image 的节点，按对应 Role 规则处理，不能机械增加 Export。

### 3.1 Clip content 与越界视觉

当一个开启 `Clip content` 的节点确实依靠裁剪边界形成最终静态外观，且其 Rectangle、Vector、阴影或装饰图形存在越界区域时，可以把“裁剪后的整体”作为 PNG 源，但必须先分离活动节点：

- Text、Button、Toggle、Input、动态 Icon、列表 Item 等可识别且需要独立生成 UGUI 对象的节点，优先移到图片源之外；
- 图片源中只保留静态图形以及需要被裁剪的装饰效果；
- 只有完成分离后的静态视觉根才能设置 PNG Export；仍承担行为或布局职责的根节点不能直接变成 Image；
- 若缺失字体导致 Text 或含 Text 的 Frame 无法跨父级移动，复制原裁剪节点作为同尺寸视觉副本，将副本命名为英文业务图片名、隐藏其中的活动 Role，并给副本设置 Export；原节点移除 Fill、隐藏静态分支，只保留活动 Role；
- 视觉副本放在原结构之前且位置、尺寸完全重合，既保持 Figma 显示不变，也避免动态节点被烘焙进 Sprite。

例如弹窗可以整理为：

```text
DialogBackground              # Clip content；只含静态视觉；PNG Export 1x
CommonDialog                  # 无 Fill、无 Export；结构容器
  Title                       # 独立 Label
```

## 4. 必须或强烈建议设置 Export 的节点

这里的“必须”是指：在默认配置 `enableVectorGraphicRendering=false` 下，若希望该视觉稳定成为 Sprite，就需要明确的 PNG Export 或放入专门的图片 Section 走整节点服务端渲染。

### 4.1 Rectangle 与复杂静态视觉

当前 Figma 转 UGUI 制作规范要求：正式 View/Component 导出树中的 Rectangle 视觉节点应设置一条 `PNG / 1x / contentsOnly / 无 suffix` Export。即使它只是纯色 Rectangle，也应使用明确的图片导出身份，避免依赖不同导出路径对简单 Fill 的隐式还原差异。

以下 Rectangle 不进入这条规则：

- `uimask`：必须保持无 Export，否则 Role 会被覆盖为普通 Image；
- `NotExport_*` 自身及其整棵子树；
- 已由祖先的完整静态 PNG 覆盖，或已识别为 NineSlice 专用源的内部切片；
- 完全透明、仅用于点击命中的 `HitArea`；
- 运行时插槽的 `placeholder` / `PlaceLimiter` 预览及其配套遮罩着色层；
- 纯结构 Frame 中不存在的空占位，不应为了满足规则临时改成 Rectangle。

命名整理也在范围内时，多个 Rectangle 的导出名应为唯一英文业务名，避免重复的 `Rectangle`、`Rectangle 1`。仅配置 Export 或要求保留名称时，不为补勾顺带改名；已有命名、重复名或 Common 冲突应单独报告。

以下视觉若位于 View/Component 内，建议在最内层视觉源设置 PNG Export：

这里的“最内层视觉源”是能独立完整复现最终外观的最小静态视觉子树，不是最深叶子。完整 Boolean 图案在 Boolean 根出图；静态遮罩与 tint/fillcolor 合成应在包含双方的视觉父节点出图。Fill 关闭但仍有可见 Stroke 或效果的节点，也必须纳入图片源检查。

- Vector、Boolean Operation、Line、Ellipse。
- 圆角、描边、渐变、多重 Fill 的 Rectangle。
- 阴影、内阴影、Blur、复杂透明叠加。
- 设计师绘制的图标、纹理、光效和装饰。
- Figma Image Fill、贴图或外部位图。
- 需要像素级保持 Figma 效果的组合图形。

原因：当前默认关闭自动 Vector 服务端栅格化；复杂 Vector/Shape 在普通 View/Component 中可能降级为 Container。Image Fill 元数据也不能作为普通图片来源稳定依赖。

### 4.2 控件中的真实图片子节点

以下是常见的 Export 视觉源：

```text
Button_confirm
  Image_bg
    button_confirm_bg          # PNG Export

Toggle_option
  Image_bg
    option_normal              # PNG Export
  Image_checked
    option_checked             # PNG Export

Input_account
  Image_bg
    input_account_bg           # PNG Export

ProgressBar_loading
  Image_bg
    loading_track              # PNG Export
  Image_fill
    loading_fill               # PNG Export

Slider_volume
  Image_bg
    volume_track               # PNG Export
  Fill Area
    Image_fill
      volume_fill              # PNG Export
  Handle Slide Area
    Image_handle
      volume_handle            # PNG Export
```

Export 应放在最终视觉源上，不应放在 Button、Toggle、Input、ProgressBar 或 Slider 根上。

### 4.3 独立 Sprite 资源

如果某个节点只需要生成 PNG/Sprite，不需要生成 GameObject，应放入 `图片/Res` Section。

`图片/Res` 有两条合法路径：

1. 节点设置 PNG Export：按 Figma Export 设置和 scale 导出。
2. 节点不设置 Export：Section 策略使用 `ForcedSectionRender`，把整个节点服务端渲染为 Sprite。

因此，图片 Section 中不是所有节点都必须勾 Export。但以下情况建议显式设置：

- 需要明确的 1x/2x 导出倍率。
- 需要 Figma Export 的可见性和裁剪结果。
- 需要稳定的 Export identity 参与缓存和 Artifact 校验。
- 同一个复杂组件中只想导出某个内部视觉节点，而不是渲染整个外层节点。

### 4.4 Common 图片源

跨模块复用的 Common Sprite 建议在公共组件库的 `图片` Section 中建立唯一源节点。

- 可使用 PNG Export，获得明确的导出 scale 和 Export identity。
- 也可依赖 `图片/Res` 的 ForcedSectionRender。
- 业务 View 只消费 Common Sprite，不应复制一份同名本地 Export 节点。

如果使用“语义容器 + Common 视觉子节点”：

```text
Image_checked
  common_img_gou1              # PNG Export / Common 图片源身份
```

真正的资源身份在内层 `common_*` 节点，不在 `Image_checked` 容器。

## 5. 不需要设置 Export 的节点

### 5.1 不承担图片源职责的节点

通常不需要 Export：

- TMP 文本。
- 自身无 Fill 的空 `Image_*` 占位节点，Sprite 由业务运行时设置。
- 已从 Common Sprite/Prefab 获得权威资源来源的消费实例。

`Image_*` 前缀只保证生成 Unity Image 组件，并不保证自动找到同名 Sprite。空占位节点不设置 Export 时，最终 `sprite` 可以为空，这是正常的运行时插槽语义。

### 5.2 图片 Section 中的整节点资源

如前所述，`图片/Res` Section 会对可见节点使用 ForcedSectionRender，因此普通整图资源可以不设置 Export。

但要避免误解：这一自动行为只属于图片资源 Section。不能因为图片 Section 中不勾 Export 也能出图，就认为 View/Component 内任意复杂 Vector 都能自动出图；默认配置下两者策略不同。

### 5.3 九宫格源

被插件数据或规范 3×3/1×3/3×1 结构识别为 NineSlice 的节点，会使用专用 NineSlice 策略，通常不依赖 PNG Export。

九宫格父节点勾 Export 时，Figma REST Export 可能失败，插件虽保留 ServerRender 回退，但不建议把普通 PNG Export 当作九宫格识别开关。应先按九宫格规则正确建立 border 和结构。

## 6. 不能设置 Export 的结构节点

以下节点设置 Export 会破坏 UGUI 结构或烘掉动态内容：

| 节点 | 为什么不能设置 |
| --- | --- |
| `Button_*` 根 | 会变成单张 Image，Label/Icon/状态子节点被烘平，Button Role 丢失 |
| `Toggle_*` 根 | 会丢失 Toggle 行为和 checked Graphic/Variant 结构 |
| `ToggleGroup_*` | 会丢失分组行为和后代 Toggle |
| `Input_*` 根 | Text Area、Placeholder、动态输入文本会被烘成图片 |
| `ProgressBar_*` 根 | `Image_fill` 无法作为运行时 Filled Image 独立控制 |
| `Slider_*` 根 | Track、Fill、Handle 无法绑定为 Slider 的驱动 Rect |
| `Fill Area` | 会把 Fill 子节点烘平，破坏 `Slider.fillRect` 的运动区域 |
| `Handle Slide Area` | 会把 Handle 烘平，破坏 `Slider.handleRect` 的运动区域 |
| `ScrollBar_*` 根 | Track 和 Handle 合成一张图，无法生成可拖动 Scrollbar |
| `Sliding Area` | Unity 需要用它作为 Handle 的驱动父级 |
| `List_*` / `ScrollView_*` | Item、Viewport、Content、Scrollbar 会被合成图片 |
| 普通 Auto Layout 容器 | 子节点不再独立布局，动态文本和 Item 被烘平 |
| Component Set 根 | Variant member 不再编译为 `FigmaVariantController` |
| Variant member 根 | 整个状态结构可能退化为静态 Image，无法安全合并差异 |
| 需要保留引用的 Component Instance | 可能从 Reference/Prefab 语义变成图片消费节点 |
| `NotExport_*` / `UiMask_*` / `placeholder` | PNG 会覆盖原压制或元数据语义，禁止给自身设置 Export |

### 6.1 Text 的特殊规则

以下 Text 不能设置 Export：

- 需要本地化。
- 需要数据绑定。
- Input 的 Placeholder/Text。
- 数量、名称、状态、倒计时等运行时文案。
- 需要响应字体 fallback、字号、颜色或排版变化的文本。

只有完全静态、确定不需要修改、且必须与复杂视觉一起烘图的装饰文字，才可以作为某个外层 PNG Export 视觉节点的后代被合并。

## 7. 各复合组件应设置 Export 的位置

| 复合组件 | 建议设置 Export | 不设置 Export |
| --- | --- | --- |
| Button | 背景、静态图标的最内层视觉源 | Button 根、Label |
| 普通 Toggle | 背景和 checked 图的最内层视觉源 | Toggle 根、Label |
| Variant Toggle | 各 Variant 内需要 Sprite 的视觉源 | Component Set 根、member 根、动态 Label |
| ToggleGroup | Group 背景视觉源（如有） | ToggleGroup 根、后代 Toggle 根 |
| Input | 背景和静态装饰图标 | Input 根、Text Area、Placeholder、Text |
| ProgressBar | Track/Fill 的 Sprite 源 | ProgressBar 根、Image_fill 语义容器 |
| Slider | Track/Fill/Handle 的 Sprite 源 | Slider 根、Fill Area、Handle Slide Area |
| Scrollbar | Track/Handle 的 Sprite 源 | ScrollBar 根、自动或自建 Sliding Area |
| List | Item 内静态背景/图标的视觉源 | List 根、Item 结构根、动态 Label、Scrollbar 根 |
| Mask | 复杂遮罩形状可在 Mask 根设置 PNG Export | 普通矩形 Mask 无需 Export |

## 8. 选服 ScrollBar 的具体设置

选服节点：ScrollBar_huadong。

当前结构：

```text
ScrollBar_huadong              # 不设置 Export
  Image_bg                     # 语义容器，不设置 Export
    serverlist_img_cemianxian  # 建议 PNG Export
  Image_handle                 # 语义容器，不设置 Export
    serverlist_img_handle      # 建议 PNG Export
```

原因：

- `ScrollBar_huadong` 必须保留 Scrollbar Role。
- `Image_bg` 必须保留为轨道 Graphic 的语义节点。
- `Image_handle` 必须保留为 `Scrollbar.handleRect`。
- 内层 `serverlist_img_*` 只负责提供 Sprite，适合 PNG Export 后被外层容器消费。

不推荐：

```text
ScrollBar_huadong              # PNG Export，错误
  Image_bg
  Image_handle
```

这会把轨道和 Handle 合成一张图片，Scrollbar 不再拥有独立可拖动 Handle。

可接受但不如容器/视觉源分离清晰的方案：

```text
ScrollBar_huadong              # 不 Export
  Image_bg                     # 直接 PNG Export，子树必须只有静态轨道视觉
  Image_handle                 # 直接 PNG Export，子树必须只有静态 Handle 视觉
```

父容器和内层 `serverlist_img_*` 不要同时设置 Export，否则会产生重复或被压制的资源请求。

## 9. 复合结构 Demo 的具体建议

### Button

```text
Button_button                 不 Export
  Image_bg                    只要自身有可见 Color Fill，就设置 PNG Export
  Label_text                  不 Export
```

### Toggle

```text
Toggle_toggle                 不 Export
  Image_bg                    视觉源按需 Export
  Image_checked               视觉源按需 Export
  Label_text                  不 Export
```

### Input

```text
Input_input                   不 Export
  Image_bg
    demo_input_bg             复杂圆角/描边背景建议 Export
  Text Area                   不 Export
    Label_input_holder        不 Export
    Label_text                不 Export
```

### ProgressBar

```text
ProgressBar_progress          不 Export
  Image_bg
    demo_progressbar_bg       建议 Export
  Image_fill
    demo_progressbar_fill     建议 Export
```

注意 `demo_progressbar_fill` 应导出完整 Fill Sprite；不要用缩短 PNG 节点表达当前进度。

### Slider

```text
Slider_slider                不 Export
  Image_bg
    demo_slider_bg           建议 Export
  Fill Area                  不 Export
    Image_fill
      demo_slider_fill       建议 Export
  Handle Slide Area          不 Export
    Image_handle             Handle 视觉源按需 Export
```

## 10. 常见错误

### 10.1 Export 层级过高

现象：Prefab 中 Label、Icon、Handle、Item 全部消失，只剩一张背景图。

原因：Export 节点会压制整个子树。

处理：把 Export 下移到最内层纯视觉节点。

### 10.2 Export 层级过低或切得太碎

现象：一个图标被拆成大量小 Sprite 和 GameObject，资源数、DrawCall 和维护成本上升。

处理：不会独立变化、材质和层级固定的一组纯视觉可以包进一个视觉 Frame，对该 Frame 设置一次 PNG Export。

### 10.3 动态文字被烘图

现象：Unity 中没有 TMP，业务无法修改文本。

处理：把 Text 移出 Export 子树，保留真实 Label。

### 10.4 同时给语义容器和视觉子节点 Export

现象：重复资源、命名冲突、某个 Export 被父级压制或缓存身份混乱。

处理：外层消费、内层出图；或外层直接出图，二选一。

### 10.5 只写 `Image_xxx`，没有任何图片来源

现象：Unity 有 Image 组件但 `sprite=null`。

原因：`Image_` 只声明 Role，不会按去前缀后的名字猜同名 Sprite。

处理：提供 PNG Export 视觉源、Common 图片引用、图片 Section 资源，或明确接受运行时赋值。

### 10.6 使用非 PNG Export

现象：日志提示“不支持非 PNG”。

处理：Figma Export format 统一选择 PNG。

## 11. 最终判断流程

```text
这个节点是否需要保留独立 UGUI 行为、布局或动态内容？
  |
  +-- 是 -> 节点本身不能 Export
  |          再检查它的纯视觉子节点是否需要 Export
  |
  +-- 否 -> 它及后代是否应合并成一张静态 Sprite？
             |
             +-- 是 -> 设置一个 PNG Export
             |
             +-- 否 -> 不设置 Export
```

一句话总结：

> **Export 应标在“最终不可再拆的静态视觉源”上，而不是标在 UGUI 控件根、布局容器或动态内容上。**

## 12. 检查清单

- [ ] Export format 是 PNG。
- [ ] 一个视觉源通常只有一个 PNG Export 设置。
- [ ] Button/Toggle/Input/Slider/Scrollbar/List 根未设置 Export。
- [ ] Fill Area、Handle Slide Area、Text Area 未设置 Export。
- [ ] 动态 Label、Placeholder、输入 Text 未进入 Export 子树。
- [ ] 复杂 Vector、圆角、描边、阴影和 Image Fill 有明确 Sprite 来源。
- [ ] `Image_*` 语义容器和视觉源的职责已经分开。
- [ ] 父容器与视觉子节点没有重复设置 Export。
- [ ] 图片 Section 中明确选择 PNG Export 或 ForcedSectionRender。
- [ ] Common 图片只有一个权威源，没有在业务模块重复导出。
- [ ] 九宫格使用专用识别规则，而不是依靠普通 Export。
- [ ] Scrollbar Track/Handle 分别出图，没有把整个 Scrollbar 烘平。
- [ ] `UiMask_*` 未被当作图片导出源；需要直接显示的半透明蒙层使用普通 Image。
- [ ] 非 PNG Instance 内部已展开检查，未因组件边界漏掉图片源。
- [ ] 仅有 Stroke 的图形、Boolean 根、完整 mask+tint 视觉已检查。
- [ ] 自动勾选已写入并回读验证，重复运行不会新增重复 Export 设置。
- [ ] 正式 PNG、排除子树内的既有 PNG、NineSlice 专用源和动态插槽分开统计。
- [ ] 仅 Export 任务未改变名称、层级、几何、可见性、画面或组件身份。

## 13. 自动勾选与漏勾复查

### 13.1 执行范围

用户要求“自动勾选、补勾、配置图片 Export、修正漏勾”，或继续复查这些修改时，检查后直接执行已确定的 Export 修改并验证，不只输出建议，不再为这些设置请求一次确认。用户明确要求“仅检查／不修改”时只输出审计结果。

本模式只修改目标内的 `exportSettings`。不顺带改名、移动、增加节点、拆组件、迁移玩法内组件或写入公共库主组件。若其他任务范围已明确授权结构修正，再按对应结构流程处理。仅勾 Export 不表示下载了 PNG，也不表示已经验证 Unity 导出的 Sprite 或 Prefab。

### 13.2 先完整遍历，再判定出图边界

从精确目标节点递归读取全部后代，包括非 PNG Instance 的所有内部节点；不能在实例外层停止。记录每个节点的父子关系、现有设置、组件身份、静态视觉依据，以及祖先 PNG、排除语义、NineSlice 等边界。不可只搜索 `Image_*`、只查 Image Fill，或只看当前可见画面。

全树枚举与普通补勾候选是两件事：已被祖先 PNG 覆盖的后代仍需检查重复勾选和动态内容，但不各自新增 PNG。有效 `NotExport` / `NoExport` / `Ignore` / `UiMask` / `placeholder` / `PlaceLimiter` 排除树不新增 PNG。确认的隐藏状态图仍按实际职责判断，不能仅因 `visible=false` 就当作备份。

| 观察到的节点 | 自动处理 |
| --- | --- |
| 正式静态位图、纹理、光效、装饰图形，没有其他合法来源 | 给完整图片源补一条 PNG |
| Fill 关闭，但 Stroke 可见，例如圆形描边 | 根据完整可见外观补 PNG，不能因没有 Fill 漏掉 |
| Boolean Operation 组成一个完整静态图案 | 勾 Boolean 根，不逐个勾内部运算图形 |
| 静态 mask 与 fillcolor/tint 共同形成图案 | 勾包含双方的完整视觉根，不单独勾遮罩或纯色矩形 |
| 非 PNG 的组件实例 | 保留实例，继续检查内部视觉源；支持时只写目标实例内的 Export override |
| 已有正确 PNG 的完整静态源 | 跳过写入；检查内部重复 PNG 和不该被烘焙的动态内容 |
| 透明 HitArea、动态 Image 插槽、placeholder 与配套 tint | 不补 PNG；配套 tint 不是独立静态资源 |
| 已按结构或插件数据确认的 NineSlice 源及内部切片 | 走九宫格专用规则，不按普通漏勾处理；不为统一格式机械增删其现有 PNG |
| 行为根、布局根、动态文字、有效排除节点 | 不新增 PNG；已有错误 PNG 按可证明的修正规则处理 |

静态合成中只要有必须独立变化的部分，就保留独立图片源；不得为了减少 PNG 数量把动态内容合并。名称只提供线索：例如 `btn_*bg` 可能只是既有纯背景名称，应结合图片内容、交互和结构职责确认，不能只按字符串批量勾选或取消。

若行为根误勾 PNG，只有在现有静态视觉源能完整承接外观、撤掉根 PNG 后每个静态分支都有合法来源、动态及控件节点能保持独立时，才自动将 Export 从根迁移到这些视觉源。仅发现一个背景子节点不够证明迁移安全。需要新增、拆分或移动节点才能修正时，在 Export-only 范围保留现状并报告阻塞，继续处理其他确定项。

### 13.3 写入行为与幂等

没有明确用户或项目覆盖规则时，接受的每个正式图片源设置为：

```js
node.exportSettings = [{
  format: "PNG",
  constraint: { type: "SCALE", value: 1 },
  contentsOnly: true,
  suffix: ""
}];
```

通过可用的 Figma Plugin API 执行，不能用选中图层、改名、报告“建议勾选”代替实际写入。沿用 `figma-use` 的调用和批次要求：

1. 形成带节点 ID、当前设置、目标设置、原因的修改清单，区分补勾、参数修正、冗余取消和 Export 迁移。
2. 每批至多约十项逻辑修改。写入前回读节点名称、父级、顺序、几何、现有设置及相关组件身份，确认没有过期；保留不含 Export 的前置快照。
3. 已是单条 PNG、SCALE 1、contentsOnly true、空 suffix 的源跳过。比较这些语义字段，不因 API 返回额外默认字段而重复写入。
4. 对漏勾或参数错误的确定源直接赋目标设置，不向现有数组反复追加。移除冗余 PNG 前确认祖先确实代表同一个不可分静态视觉，且没有独立动态职责。
5. 回读每个修改节点的 `exportSettings`，返回所有修改 ID 和前后值。API 失败后停止该批、检查实际状态，再决定恢复；不得通过 Detach 或改主组件绕过不支持的实例 override。

### 13.4 收尾复核与报告

写入后重新展开整个目标，包括实例内部；不能只复查本批节点。确认：

- 正式图片源的实际格式、倍率、contentsOnly、suffix 和设置条数正确；
- 没有有效父子重复 PNG，没有动态文字、占位预览或独立控件被烘入；
- 所有剩余有绘制内容的非 Text 节点，都已有合法图片来源、处于完整 PNG 下、属于专用策略，或明确列为未解决；
- 名称、节点类型、父子顺序、几何、布局、可见性、绘制属性、文字和组件 key 与修改前一致；
- 整体和关键复合节点的画面检查通过。Export-only 设置核验和 Unity 运行时验证分别报告。

结果记录 `added`、`corrected`、`removed`、`movedExport`、`unchanged`、`intentionallyUnexported`、`unresolved` 的节点 ID 与原因。另给出完整遍历节点数、正式 PNG 设置节点数、排除子树内既有 PNG 数、专用切片源情况与验证结果。一次迁移记为一项 `movedExport`，不能把迁入节点重复算成漏勾图片。

这里的正式 PNG 数是当前目标内有效设置节点数，不是去重文件数或已下载资源数。`NotExport` 备份内部的既有 PNG 单列并保持不动；它们不因子节点有 PNG 就成为正式图片。排除节点自身误设 PNG 则会覆盖排除 Role，应单独作为错误处理。所有数量从当前目标计算，不能沿用上次任务的计数作为完成标准。

## 14. 图片 Export 覆盖完整性门槛（追加规则）

用户反馈“应该导出的图片仍未导出”时，不把一次补勾的数量当成完成标准。对目标完整树中的每个独立静态视觉源建立覆盖表，并且只能归入以下一种结论：

| 覆盖结论 | 通过条件 |
| --- | --- |
| `exported` | 最终不可再拆的静态视觉根自身有且只有合规 PNG，完整外观已覆盖 |
| `coveredByAncestor` | 合规 PNG 祖先完整覆盖该视觉，后代没有需独立替换的职责，也没有重复导出 |
| `specialPolicy` | NineSlice、ForcedSectionRender、复杂 Mask、UiMask/NotExport/placeholder/PlaceLimiter 等专用规则已被确认，并记录来源 |
| `intentionallyUnexported` | 动态文字、运行时插槽、布局/行为根、交互 HitArea 或必须保留的结构职责已确认不烘图 |
| `needsReview` | 读取不完整、静态/动态边界、Instance override、Mask/Clip 或结构迁移条件仍不确定 |

有可见绘制内容、却没有上述任何结论的节点必须进入 `missingCoverage`，不能静默跳过或归入“已复核”。`missingCoverage` 是导出不完整的失败信号；只有补齐、明确专用策略或列入有 ID 和理由的 `needsReview/unresolved` 后，才能结束审计。对确定的静态源直接补写 Export；若修正需要移动、拆分、Detach、改主组件或改变结构，则保留原状并报告阻塞。

收尾时必须重新递归展开全部后代（包括非 PNG Instance 内部），回读每个正式源的 Export 参数，检查父子重复、动态文字/控件烘入、排除树误导出和静态源遗漏。报告至少包含 `exported`、`coveredByAncestor`、`specialPolicy`、`intentionallyUnexported`、`needsReview`、`missingCoverage` 的节点 ID 与原因；不能只报告新增 PNG 的数量。截图检查、Figma Export 设置检查和 Unity 运行时验证仍分别记录。

覆盖闭环的绘制起点包括 `paint.visible && opacity > 0` 的 Rectangle、Vector、Boolean、Line、Ellipse，Fill 关闭但 Stroke 可见的节点、Effect，以及 Mask+tint 合成。`coveredByAncestor` 只有在祖先自身可见且 `opacity > 0`、其所有可见静态后代都被完整覆盖，且没有动态 Text、控件、运行时插槽、独立替换分支或排除边界时才成立；祖先仅有 Export 设置不够。

Component Set、Variant member 或普通 Component 根不 PNG，不等于其内部静态像素可以跳过；成员根承载完整像素但没有可导出的静态子源时，列入 `needsReview/unresolved`。View Instance 不能凭实例名推断已有图片，必须查 canonical source 与实际 Export。先执行 safe-candidate pass，再执行 coverage-closure pass；报告 `coverageTotals={painted,exported,coveredByAncestor,specialPolicy,intentionallyUnexported,needsReview,missingCoverage}`。只要 `missingCoverage > 0` 或仍有未解释的 `needsReview/unresolved`，就只能报告 Export 覆盖未闭环。

`ForcedSectionRender` 只有在节点确实位于 `Res/图片` Section、插件策略可读且来源 Section 已记录时，才可归入 `specialPolicy`；否则仍是 `missingCoverage` 或 `needsReview`。 
