# FigmaToUGUI 模块与 Page / Section 导出规范

## Contents

- [1. 适用范围](#1-适用范围)
- [2. 总体推荐结构](#2-总体推荐结构)
- [3. 模块名称如何从 Figma 获取](#3-模块名称如何从-figma-获取)
  - [3.1 主来源是 Page 名](#3-1-主来源是-page-名)
  - [3.2 解析优先级](#3-2-解析优先级)
  - [3.3 推荐 Page 命名](#3-3-推荐-page-命名)
  - [3.4 多个 Page 是否可以属于同一模块](#3-4-多个-page-是否可以属于同一模块)
- [4. 公共模块的识别](#4-公共模块的识别)
- [5. Section 的识别规则](#5-section-的识别规则)
  - [5.1 必须使用真实 Figma Section](#5-1-必须使用真实-figma-section)
  - [5.2 Section 必须是 Page 的直属子节点](#5-2-section-必须是-page-的直属子节点)
  - [5.3 默认名称匹配](#5-3-默认名称匹配)
- [6. 界面（Views）Section 规范](#6-界面-views-section-规范)
  - [6.1 放置内容](#6-1-放置内容)
  - [6.2 根节点和产物名](#6-2-根节点和产物名)
  - [6.3 可见性](#6-3-可见性)
- [7. 组件（Coms）Section 规范](#7-组件-coms-section-规范)
  - [7.1 放置内容](#7-1-放置内容)
  - [7.2 组件要求](#7-2-组件要求)
- [8. 图片（Res）Section 规范](#8-图片-res-section-规范)
  - [8.1 放置内容](#8-1-放置内容)
  - [8.2 是否需要设置 Figma Export](#8-2-是否需要设置-figma-export)
  - [8.3 模块资源归属](#8-3-模块资源归属)
- [9. Section 外节点和其他 Section](#9-section-外节点和其他-section)
- [10. 模块名如何影响 Unity 输出](#10-模块名如何影响-unity-输出)
- [11. Unity 产物禁止中文命名](#11-unity-产物禁止中文命名)
  - [11.1 规则范围](#11-1-规则范围)
  - [11.2 为什么必须在 Figma 侧处理](#11-2-为什么必须在-figma-侧处理)
  - [11.3 统一字符范围](#11-3-统一字符范围)
  - [11.4 Prefab 文件名对应的 Figma 设置](#11-4-prefab-文件名对应的-figma-设置)
  - [11.5 Prefab 内所有节点对应的 Figma 设置](#11-5-prefab-内所有节点对应的-figma-设置)
  - [11.6 图片文件名对应的 Figma 设置](#11-6-图片文件名对应的-figma-设置)
  - [11.7 Variant 和状态名称](#11-7-variant-和状态名称)
  - [11.8 名称参数](#11-8-名称参数)
  - [11.9 Figma 侧检查范围](#11-9-figma-侧检查范围)
  - [11.10 规则检查表达式](#11-10-规则检查表达式)
- [12. 推荐模板](#12-推荐模板)
  - [12.1 普通业务模块](#12-1-普通业务模块)
  - [12.2 公共模块](#12-2-公共模块)
  - [12.3 不推荐结构](#12-3-不推荐结构)
- [13. 导出前检查清单](#13-导出前检查清单)


## 1. 适用范围

本文整理节点层级和 Role 以外的 Figma 导出规范，重点说明：

- 插件如何从 Figma 获取模块名称；
- Page 应如何命名；
- Page 下应如何组织 Section；
- `界面（Views） / 组件（Coms） / 图片（Res）` 分别放什么；
- 模块名、Section 和 Unity 输出目录之间的关系；
- 公共模块、可见性、重名和按需导出的注意事项。

相关文档：

- [FigmaToUGUI导出节点类型总览](./FigmaToUGUI导出节点类型总览.md)
- [Figma复合结构转UGUI整理分析](./Figma复合结构转UGUI整理分析.md)
- [Figma节点PNG Export设置规则分析](./Figma节点PNG-Export设置规则分析.md)

## 2. 总体推荐结构

一个业务模块推荐使用一个专门的 ToUGUI Page：

```text
Page: ToUGUI（ServerList）
  Section: 界面
    UIRoot_ServerList        # 1080 × 2400
    UIRoot_ServerRecommend   # 1080 × 2400

  Section: 组件              # 按需创建
    ServerItem                 # Component
    Toggle_ServerType         # Component Set

  Section: 图片              # 按需创建
    serverlist_bg
    serverlist_tab_normal
    serverlist_tab_selected
```

公共模块推荐：

```text
Page: ToUGUI（Common）
  Section: 界面              # 可选，公共模块一般可以没有正式 View
  Section: 组件
    CommonDialog
    Toggle_CommonTab
  Section: 图片
    common_btn_bg
    common_icon_close
```

核心规则是：

```text
Page 决定模块归属
Section 决定产物类别：界面（Views）/ 组件（Coms）/ 图片（Res）
Section 直属子节点决定独立导出单元
节点 Role 决定 Prefab 内的 UGUI 结构
```

## 3. 模块名称如何从 Figma 获取

### 3.1 主来源是 Page 名

插件调用 `FigmaModuleNameResolver.Resolve(page.Name, rulesPath, overrideValue)`。默认 `ModuleNameRules.json` 使用以下规则：

```regex
^[^()（）]*[（(]([^）)]+)[）)]
```

即取 Page 名中第一对中文或英文括号内的内容：

| Figma Page 名 | 解析出的模块名 |
| --- | --- |
| `ToUGUI（ServerList）` | `ServerList` |
| `ToUGUI(ServerList)` | `ServerList` |
| `主界面（Login）` | `Login` |
| `公共组件（Common）` | `Common` |
| `ServerList` | 未命中正则，回退为完整 Page 名 `ServerList` |

模块名不是从 Section、View Frame、Figma 文件名或 URL 参数读取的。它们可以影响导出选择和产物名，但不是默认模块名来源。

### 3.2 解析优先级

模块名优先级为：

1. Unity 导出设置中的 `ModuleNameOverride`；
2. `ModuleNameRules.json` 中第一条成功匹配且第一个捕获组非空的规则；
3. 去掉扩展名后的完整 Page 名。

规则按 JSON 中的顺序匹配，只取第一个有效捕获组。括号里的值会 `Trim`，但不会主动转换大小写。

`ModuleNameOverride` 是本次设置的全局覆盖值。导出多个 Page 时，如果设置了同一个 Override，这些 Page 都会被视为同一模块。因此批量导出多模块时应保持为空，依赖 Page 命名规则。

### 3.3 推荐 Page 命名

团队统一使用：

```text
ToUGUI（ModuleName）
```

推荐模块名只使用稳定的英文、数字和下划线，例如：

```text
ToUGUI（Login）
ToUGUI（ServerList）
ToUGUI（Activity2026）
ToUGUI（Common）
```

不建议：

- `ToUGUI（登录）`：可以工作，但 Unity 目录、代码生成和跨工具配置使用中文模块名更容易产生维护成本。
- `ToUGUI（Login / Server）`：路径非法字符在生成目录时会被替换，实际目录名可能与设计预期不同。
- `ToUGUI（Login）（Old）`：只取第一对括号，后续括号不会成为模块名。
- `ToUGUI-Login-v2`：未命中默认规则时整个 Page 名都会成为模块名。
- 多个含义不同的 Page 却解析成同一模块名：其资源和组件会进入同一模块作用域，容易重名。

### 3.4 多个 Page 是否可以属于同一模块

技术上可以。模块比较忽略大小写，多个 Page 解析到同一模块名时会被纳入同一模块视图；按选中 Frame 导出时，插件还会查找所有同模块 Page 的 `Res` 资源 Section。

但 Figma 创作侧优先采用“一模块一 ToUGUI Page”，原因是：

- 更容易看到该模块完整的界面、组件、图片；
- 避免跨 Page 的图片、组件和 Prefab 重名；
- 避免只移动 View、漏移动对应资源；
- 便于按模块检查和批量导出。

只有 Page 内容规模过大时再拆分，并保证所有拆分 Page 的括号模块名完全一致。

## 4. 公共模块的识别

公共模块不是靠 `FigmaModuleMapping.json` 判断，而是由 `CommonModuleConfig.json` 的 `commonModuleName` 定义，默认值为：

```json
"commonModuleName": "Common"
```

因此公共 Page 推荐命名：

```text
ToUGUI（Common）
```

Page 解析出的模块名必须与 `commonModuleName` 忽略大小写相等，公共图片、公共组件相关流程才会处理该 Page。

公共图片前缀由公共模块名派生：

```text
Common -> common_
```

例如：

```text
common_btn_bg
common_icon_close
```

`common_*` 资源不应放在普通业务模块的“图片” Section。插件发现公共前缀图片位于非公共模块的 Res/Export 范围时会跳过并警告。

公共模块名不允许包含空白、控制字符和路径非法字符。配置非法时会回退为 `Common`。

## 5. Section 的识别规则

### 5.1 必须使用真实 Figma Section

只有 Figma 节点类型为 `SECTION` 才会进入 Section 分类。以下做法无效：

- 把普通 Frame 命名为 `界面`；
- 把 Group 命名为 `组件`；
- 只在图层名称中包含“图片”但没有创建 Section。

### 5.2 Section 必须是 Page 的直属子节点

完整文档和轻量索引都只枚举 Page 的直属 children 来识别 Section。解析某个 Section 时，其直属子节点会作为顶层导出候选；如果 Section 内再嵌套 Section，内层 Section 会被跳过。

正确：

```text
Page
  Section 界面
    Frame ServerList
```

错误：

```text
Page
  Section ToUGUI
    Section 界面        # 内层 Section 不会按正式导出范围继续解析
      Frame ServerList
```

因此不要再建立一个总 Section 包住界面、组件、图片。需要的 Section 应直接并列放在 Page 下。

### 5.3 默认名称匹配

`SectionExportRules.json` 默认规则如下：

| Section 类型 | 可识别名称 | 匹配方式 |
| --- | --- | --- |
| 界面（Views） | `界面*`、兼容 `Views*` | Prefix，忽略大小写 |
| 组件（Coms） | `组件*`、兼容 `Coms*` | Prefix，忽略大小写 |
| 图片（Res） | `图片*`、兼容 `Res*` | Prefix，忽略大小写 |
| Export | 只接受 `Export` | Exact，忽略大小写 |

所以这些名称可以识别：

```text
界面
界面-弹窗
组件-列表项
图片-背景
```

Figma 创作侧必须使用对应中文名，英文名只作为旧文档兼容入口。普通业务模块默认只建立一个 `界面` Section；只有确实存在模块内可复用组件或独立图片资源时，才分别创建 `组件`、`图片`。待整理节点已经位于 `界面` Section 时，应直接复用，不能再创建新的“界面”，也不能把它改名为 `Views`。内容很多时才拆成 `界面-主界面`、`界面-弹窗` 等多个同类 Section。

生产规范不依赖 `Export` Section。当前代码对它保留了部分兼容入口，但统一策略层并不把它作为正式 View/Component Section；新文档应使用界面、组件、图片。

未识别的 Section 是 `Unknown`，默认生产模式下会被忽略。`Vars` 虽存在内部枚举，但默认 Section 规则没有 Vars 命名映射，统一策略也不生成 UI 产物。

## 6. 界面（Views）Section 规范

### 6.1 放置内容

界面 Section 中每个直属、可见且有有效边界的顶层 Frame 表示一个独立 View Prefab：

```text
Section 界面
  Frame UIRoot_ServerList         # 1080 × 2400
  Frame UIRoot_ServerRecommend    # 1080 × 2400
```

界面根节点（UIRoot）的标准设计尺寸为：

```text
Width  = 1080
Height = 2400
```

即 Figma 中“界面” Section 下的每个正式界面根 Frame 应设置为 `1080 × 2400 px`。插件会使用根 Frame 的实际边界计算 Prefab 根尺寸；使用 `StandaloneCanvas` 根模式时，该尺寸还会成为 `CanvasScaler.referenceResolution`。因此不能只让内部背景达到 1080 × 2400，而让 UIRoot 本身保持其他尺寸。

UIRoot Frame 不得直接设置 Color Fill。界面底色或背景必须使用独立的英文 Image/Rectangle 子节点，并放在 UIRoot 的最底层；简单纯色可使用铺满父节点的 `image_bg`，复杂静态视觉则按图片规则设置 PNG Export。这样 UIRoot 只负责根尺寸和结构，不会与背景 Graphic 混为同一个节点。

除非项目对横屏、平板或特殊二级画布另有明确规范，正式业务界面都应使用这一标准尺寸。需要例外时，应在模块规范中单独记录，不能因内容超出画布而随意拉长 UIRoot；滚动内容应放在 List/ScrollView 的 Content 中。

不要增加一层纯整理 Frame：

```text
Section 界面
  Frame 所有界面             # 会被当作一个导出根
    Frame ServerList         # 不再是独立 View 根
    Frame ServerRecommend
```

如果只是为了 Figma 画布排版，请用 Section 自身组织，而不是用一个父 Frame 包住多个正式 View。

### 6.2 根节点和产物名

默认开启 `InferUiRootFromViewsTopLevel`，“界面”顶层 Frame 会推断为 UIRoot。默认也开启 `StripUiRootPrefixFromExportNames`：

```text
UIRoot_ServerList -> ServerList.prefab
```

即使允许省略 UIRoot 前缀，团队仍可显式命名 `UIRoot_*` 来强调它是独立界面根。

同一 Page 中导出名重复的 Frame 会被跳过并警告，所以 View 名必须唯一。名称清洗后相同也应视为冲突，例如不同非法字符最终都被替换成 `_` 的情况。

### 6.3 可见性

- 隐藏的“界面”顶层根不会作为正式 View 导出。
- 默认 `includeHiddenChildrenInViews=true`，View 内部隐藏子节点仍可进入 Prefab，但对应 GameObject 保持 inactive。
- 不要通过隐藏正式 View 根来表达版本状态；需要导出的版本应保持可见并使用明确名称。

## 7. 组件（Coms）Section 规范

### 7.1 放置内容

“组件”用于当前模块可复用的组件定义：

```text
Section 组件
  Component ServerItem
  Component ServerTag
  Component Set Toggle_ServerType
```

推荐直属放置 Component 或 Component Set。插件在默认模式下要求组件能追溯到“组件”（内部类型 Coms）Section；放在未知 Section 或 Page 顶层的组件定义会跳过并提示移动到“组件”。

普通 Frame 也存在作为“组件”顶层组件候选的兼容路径，但无法提供稳定的 Figma Component id/key 和 Instance 关联。需要复用和引用的内容应创建为真正 Component。

### 7.2 组件要求

- Component/Component Set 名称在同一输出作用域内必须唯一；重复名称的后续组件会跳过。
- Variant member 放在对应 Component Set 内，不要散放到 Coms 顶层。
- 页面中使用组件时使用真实 Instance，插件才能通过 component id/key 链接 Nested Prefab。
- Coms 内的样例展示、说明文字和状态对照不要混在正式 Component 子树中；应放在不导出范围或移到另外的设计 Page。

Section 在 Figma 画布上的前后顺序不应作为依赖关系。公共组件目录会先建立组件清单，再处理资源，不能依赖“把 Coms 放在 Res 前面”来修复引用。

## 8. 图片（Res）Section 规范

### 8.1 放置内容

“图片”中每个直属节点应表示一个独立 Sprite 资源单元：

```text
Section 图片
  Frame serverlist_bg
  Component serverlist_tab_normal
  Group serverlist_icon_recommend
```

资源可以由内部多个可见子层组合，但外层直属节点必须有稳定名称和有效边界。插件会把资源单元作为整体请求 Figma 服务端渲染；Res 资源不生成普通 UI GameObject。

不要用一个总 Frame 包住所有资源：

```text
Section 图片
  Frame 所有图片             # 容易被识别成一个合成资源
    Frame bg
    Frame icon
```

应让 `bg`、`icon` 分别成为 Res 的直属资源节点。

### 8.2 是否需要设置 Figma Export

Res Section 会使用 `ForcedSectionRender` 等资源策略，普通资源节点不要求逐个勾选 Figma PNG Export。只有确实需要控制 Figma Export scale/suffix 或走特定 PNG 资源路径时才设置。

PNG Export 只支持 PNG；不要在行为根、布局根或完整 View 根上设置。详细规则见 [Figma节点PNG Export设置规则分析](./Figma节点PNG-Export设置规则分析.md)。

### 8.3 模块资源归属

按选中 Frame 导出时，插件先从该 Frame 所在 Page 解析模块名，再加载所有同模块 Page 的 Res/Export 图片 Section。因此：

- 资源应放在与使用方相同模块名的 Page；
- 同模块拆成多个 Page 时，各 Page 的 Res 会共同进入模块资源范围；
- 不同模块不要使用相同模块名来“共享”资源，应把真正共享资源放到 Common；
- 同模块资源名应唯一，避免不同 Section/不同 Page 输出到同一路径。

## 9. Section 外节点和其他 Section

在默认生产模式下：

- View Prefab 只接受 Views 类型；
- 组件 Prefab 只接受 Coms 类型；
- 图片资源使用 Res 类型；
- Unknown、Vars 等范围不生成正式 UI 产物；
- Page 顶层散放的 Frame/Component 不应作为正式导出结构依赖。

可以在同一个 Figma 文件中保留需求说明、流程图和原始设计 Page，但 ToUGUI Page 应保持干净。推荐将非导出内容放到单独 Page，而不是与导出 Section 混放。

`动效*` Section 由单独的动效图片导出工具按名称前缀识别，不属于常规 Views/Coms/Res 页面导出。只有使用该专用工作流时才建立，不能用它替代 Res。

## 10. 模块名如何影响 Unity 输出

默认输出根为：

```text
Assets/FigmaToUGUI/Generated/Prefabs
Assets/FigmaToUGUI/Generated/Components
Assets/FigmaToUGUI/Generated/Sprites
```

普通模块通常在对应根目录下增加模块子目录：

```text
ToUGUI（ServerList）
  -> Prefabs/ServerList/ServerList.prefab
  -> Components/ServerList/*.prefab
  -> Sprites/ServerList/*.png
```

公共模块的组件和图片路径优先使用 `CommonModuleConfig.json` 中的显式路径；为空时回退到：

```text
Components/Common
Sprites/Common
```

`FigmaModuleMapping.json` 是兼容/高级输出映射，不是 Page 模块识别的主来源。它按 View/Component 名精确匹配；当前主流程会使用其中的模块名、Prefab 路径和 Component 路径。配置结构虽然还声明了 `spritePath`，但当前 Editor 代码没有调用 `ResolveSpriteRoot`，不能把它当作有效的 Sprite 路径覆盖功能。除非项目确实需要历史目录兼容或逐 View 定制路径，否则不要维护该文件，以免 Page 模块名与实际输出目录不一致。

## 11. Unity 产物禁止中文命名

### 11.1 规则范围

导出到 Unity 后，以下名称一律不能包含中文：

1. 所有 `.prefab` 文件名；
2. Prefab 层级内所有 GameObject 名；
3. 所有导出的 Sprite/PNG 文件名；
4. 由模块名生成的 Unity 输出目录名；
5. Component、Component Set 和状态图资源生成的 Prefab/Sprite 名。

这条规则限制的是 Unity 对象名和资产文件名，不限制界面显示文本。Figma Text 图层可以显示中文，但该 Text 图层自身的 Layer name 必须是英文：

```text
Layer name: Label_ServerName    # 正确，导出节点名为英文
Text content: 推荐服务器         # 允许，属于界面显示内容
```

错误示例：

```text
Layer name: 推荐服务器           # 导出 GameObject 名包含中文
Text content: 推荐服务器
```

### 11.2 为什么必须在 Figma 侧处理

插件的名称清洗主要调用 `SanitizeFileName`，它只会处理 `/`、`\` 和系统非法文件名字符，不会：

- 检测中文；
- 将中文转换为拼音；
- 自动翻译中文；
- 给中文业务名生成稳定英文别名。

因此 `按钮背景` 仍可能导出为 `按钮背景.png` 或名为 `按钮背景` 的 GameObject。规则必须在 Figma 创作阶段执行，不能依赖导出后批量重命名；导出后改名还会破坏增量更新、节点标记、Prefab 引用和代码绑定路径。

### 11.3 统一字符范围

所有基础业务名推荐满足：

```regex
^[A-Za-z][A-Za-z0-9_]*$
```

即：

- 第一个字符使用英文字母；
- 后续只使用英文字母、数字、下划线；
- 不使用中文、空格、全角符号、斜杠、括号和标点；
- 不以数字开头；
- 不使用仅大小写不同的名称区分资源。

Variant 的 `Property=Value` 和必要的 `@key=value` 参数属于语法例外，但等号两侧及参数内容仍必须是英文、数字或下划线，不能包含中文。

各类名称建议：

| 类型 | 推荐形式 | 示例 |
| --- | --- | --- |
| 模块名 | PascalCase | `ServerList`、`Common` |
| View/Prefab | PascalCase，Figma 可带 Role 前缀 | `UIRoot_ServerList` |
| Component Prefab | PascalCase | `ServerItem`、`CommonDialog` |
| 普通 UGUI 节点 | `Role_BusinessName` | `Button_Confirm`、`Label_ServerName` |
| 容器 | `Container_BusinessName` | `Container_ServerInfo` |
| 图片资源 | lower_snake_case | `serverlist_bg_main` |
| 公共图片 | `common_` + lower_snake_case | `common_icon_close` |
| Variant 属性和值 | 英文 PascalCase 或固定英文状态 | `State=Normal`、`Checked=On` |
| Export suffix | 英文、数字、下划线 | `_tex`、`_2x` |

### 11.4 Prefab 文件名对应的 Figma 设置

View Prefab 名通常来自“界面”（内部类型 Views）Section 下的 UIRoot Frame 名。默认会去掉 `UIRoot_` 前缀：

```text
UIRoot_ServerList -> ServerList.prefab
UIRoot_选服       -> 选服.prefab      # 禁止
```

因此必须保证去掉 Role 前缀后的业务名仍是英文。

Component Prefab 名来自“组件”（内部类型 Coms）Section 下的 Component 或 Component Set 名：

```text
Component ServerItem        -> ServerItem.prefab
Component 服务器条目         -> 服务器条目.prefab   # 禁止
```

需要检查的 Figma 字段：

- UIRoot Frame 名；
- Component 名；
- Component Set 名；
- 可能作为独立导出根的普通 Frame 名；
- Page 括号内的模块名，因为它会参与输出目录命名；
- 高级模块映射中配置的模块名和路径段。

Page 前缀和 Section 名自身不直接成为 Prefab 文件名。Page 推荐采用 `ToUGUI（ServerList）`；Section 创作规范固定使用中文 `界面 / 组件 / 图片`，英文 `Views / Coms / Res` 仅保留为插件兼容名称。

### 11.5 Prefab 内所有节点对应的 Figma 设置

Prefab 内 GameObject 名主要来自 Figma Layer name。Role 识别不会普遍删除 Role 前缀，只有 UIRoot 等少量导出名有专门处理。因此导出树内所有这些节点都必须使用英文 Layer name：

- Frame、Group、Section 直属导出根；
- Text、Rectangle、Vector、Ellipse、Line；
- Component 和 Component Set 内的全部子层；
- 放入 Views 的 Component Instance；
- Button、Toggle、List、Input、Slider、ScrollBar 等复合控件根及其关键子节点；
- 默认隐藏但仍会导出的 Views 子节点；
- Image 容器中的视觉源节点。

特别注意 Instance：即使源 Component 名是英文，页面中的 Instance Layer name 也会成为嵌套 Prefab 实例节点名，所以每个实例也要单独检查。

推荐：

```text
Container_ServerInfo
  Image_Background
  Label_ServerName
  Label_ServerStatus
  Button_Enter
```

禁止：

```text
服务器信息
  背景
  服务器名称
  状态
  进入按钮
```

设计说明、标尺和注释如果完全位于非导出范围，可以保留中文；一旦位于正式导出树中，即使节点默认 hidden，也应按英文规则命名，因为默认配置会把 Views 内隐藏子节点导出为 inactive GameObject。

### 11.6 图片文件名对应的 Figma 设置

Sprite 文件名可能来自以下字段：

- “图片”（内部类型 Res）Section 下资源根节点名；
- “界面/组件”内设置 PNG Export 的视觉节点名；
- `Image_` 或 `Img_` 前缀去除后的业务名；
- Button/Toggle 状态视觉节点名；
- Variant 状态名；
- Figma Export 设置中的 suffix；
- 公共图片 key。

因此不能只检查 Res。以下都会产生中文图片名风险：

```text
Image_选服背景            -> 选服背景.png
Button_确认/背景          -> 状态图片可能使用“背景”
State=按下                -> 状态图片名可能拼接“按下”
server_bg + suffix “_高清” -> server_bg_高清.png
```

推荐：

```text
Image_serverlist_bg
Image_button_normal
Image_button_pressed
State=Normal
State=Pressed
suffix=_tex
```

`Image_`/`Img_` 前缀可能在图片命名阶段被去掉，所以不能只保证前缀是英文；前缀后的实际资源名也必须是英文。

### 11.7 Variant 和状态名称

Variant property name、property value 和 Variant member name 也统一使用英文：

```text
State=Normal
State=Highlighted
State=Pressed
State=Disabled
Checked=On
Checked=Off
```

原因是状态值除了用于 `FigmaVariantController`，还可能参与 Button/Toggle 状态识别和状态 Sprite 文件名拼接。不要使用：

```text
状态=正常
状态=按下
选中=是
```

中文状态展示文字应放在 Variant 内部的英文命名 Text 图层内容中，而不是 property/value 名中。

### 11.8 名称参数

节点名中使用的 `@参数` 也是 Layer name 的一部分。参数 key 和 value 都不得包含中文：

```text
Reference_ServerItem@size=source     # 可以
Input_Account@placeholder=Account    # 名称为英文
Input_账号@placeholder=请输入账号     # 禁止
```

如果需要中文占位符或其他本地化文字，应优先放在英文命名的 Text 子节点内容中，或交给 Unity 本地化系统，不应把中文文案编码进节点名。

### 11.9 Figma 侧检查范围

导出前至少检查以下位置：

| Figma 位置/字段 | 是否允许中文 | 原因 |
| --- | --- | --- |
| Page 括号内模块名 | 不允许 | 生成模块目录和模块作用域 |
| Page 括号外说明文字 | 可以，但不推荐 | 默认不进入产物名 |
| Section 名 | 必须使用 `界面 / 组件 / 图片` | Section 自身不生成 Prefab 节点 |
| Views 下 UIRoot 名 | 不允许 | 生成 View Prefab 和根节点名 |
| Coms 下 Component/Component Set 名 | 不允许 | 生成组件 Prefab 名 |
| 正式导出树全部 Layer name | 不允许 | 生成 GameObject 名或绑定路径 |
| Instance Layer name | 不允许 | 生成嵌套 Prefab 实例节点名 |
| Res 资源根及视觉子层名 | 不允许 | 可能生成 Sprite 名或资源记录名 |
| Variant 属性名/值/member name | 不允许 | 参与状态和图片命名 |
| Figma Export suffix | 不允许 | 直接追加到图片文件名 |
| Text characters/content | 允许 | 这是界面文案，不是节点名 |
| 完全不导出的设计说明 | 允许 | 不进入 Unity 产物 |

### 11.10 规则检查表达式

如果后续给插件增加自动校验，可分别使用：

```regex
# 检查是否含中文
[\u3400-\u4DBF\u4E00-\u9FFF\uF900-\uFAFF]

# 检查推荐的基础 ASCII 标识符
^[A-Za-z][A-Za-z0-9_]*$

# 检查带可选 @key=value 参数的节点名
^[A-Za-z][A-Za-z0-9_]*(?:@[A-Za-z][A-Za-z0-9_]*=[A-Za-z0-9_]+)*$
```

校验对象至少应包括：模块名、View 导出名、Component/Component Set 名、所有将生成节点的 `CleanName/ExportDisplayName`、图片 canonical name、Variant property/value 和 PNG Export suffix。

## 12. 推荐模板

### 12.1 普通业务模块

```text
Page ToUGUI（ModuleName）
  Section 界面
    UIRoot_ViewA             # 1080 × 2400
      ...正式界面节点
    UIRoot_ViewB             # 1080 × 2400
      ...正式界面节点

  Section 组件               # 有模块内复用组件时按需创建
    Component ComponentA
    Component Set Toggle_ComponentB

  Section 图片               # 有独立图片资源时按需创建
    module_bg_main
    module_icon_a
    module_icon_b
```

### 12.2 公共模块

```text
Page ToUGUI（Common）
  Section 组件
    Component CommonButton
    Component Set Toggle_CommonTab

  Section 图片
    common_btn_normal
    common_btn_pressed
    common_icon_close
```

### 12.3 不推荐结构

```text
Page ServerList设计稿               # 模块名回退为整个 Page 名
  Frame 界面                        # 不是 Section
  Section ToUGUI                    # Unknown
    Section 界面                    # 嵌套 Section 被跳过
  Frame ServerList                  # 散放在 Page 顶层
  Section 图片
    Frame 所有图片                  # 多个资源被包成一个导出单元
```

## 13. 导出前检查清单

- Page 使用 `ToUGUI（ModuleName）`，括号内容就是预期 Unity 模块名。
- Page 括号内模块名、UIRoot、Component/Component Set、全部导出 Layer、Variant 状态和图片 suffix 都不包含中文。
- 所有 Prefab、GameObject、Sprite 的基础业务名满足 `^[A-Za-z][A-Za-z0-9_]*$`；图片可进一步统一为 lower_snake_case，必要参数使用纯 ASCII 的 `@key=value`。
- 批量导出多模块时 `ModuleNameOverride` 为空。
- 普通模块默认只有 Page 直属的真实 `界面` Section；`组件`、`图片` 仅在确有内容时按需创建。
- 已位于 `界面` Section 的待整理结构复用原 Section，不重复创建，也不改成英文名。
- 没有用一个总 Section 再嵌套三类 Section。
- “界面”下每个正式 UIRoot Frame 的尺寸为 `1080 × 2400 px`。
- UIRoot Frame 没有直接 Color Fill；背景由最底层英文 Image/Rectangle 子节点承担。
- “界面”的每个直属 Frame 是一个独立 View，名称和清洗后的导出名唯一。
- “组件”的正式复用单元是 Component/Component Set，名称唯一。
- “图片”的每个直属节点是一张独立资源，名称和边界稳定。
- 公共 Page 解析出的模块名与 `CommonModuleConfig.commonModuleName` 一致。
- `common_*` 资源只位于公共模块。
- 正式导出内容没有散放在 Page 顶层或 Unknown Section。
- 同模块拆分多个 Page 时，跨 Page 的 View、Component、Sprite 名称不冲突。
- 没有依赖 Section 的画布顺序表达组件或资源依赖。

