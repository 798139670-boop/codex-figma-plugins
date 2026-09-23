# 公共组件库 new：Variant 与 Toggle 导出规则分析

## Contents

- [1. 分析范围](#1-分析范围)
- [2. Page 的用途与主要结构](#2-page-的用途与主要结构)
- [3. Component Set 的通用导出流程](#3-component-set-的通用导出流程)
  - [3.1 为什么只允许真实存在的组合](#3-1-为什么只允许真实存在的组合)
  - [3.2 Variant 可以改什么](#3-2-variant-可以改什么)
- [4. Toggle 有两条不同的导出路径](#4-toggle-有两条不同的导出路径)
  - [4.1 普通 Toggle](#4-1-普通-toggle)
  - [4.2 Variant Toggle](#4-2-variant-toggle)
  - [4.3 状态名不是 Toggle Role](#4-3-状态名不是-toggle-role)
- [5. ToggleGroup 的处理](#5-togglegroup-的处理)
- [6. 页面典型组合结构点评](#6-页面典型组合结构点评)
  - [6.1 `Toggle_yeqian`](#6-1-toggle_yeqian)
  - [6.2 `BottomToggleS7`](#6-2-bottomtoggles7)
  - [6.3 `ContainerToggleListS7`](#6-3-containertogglelists7)
- [7. 哪些前缀可以省略](#7-哪些前缀可以省略)
- [8. 哪些前缀不能省略](#8-哪些前缀不能省略)
- [9. 需要谨慎省略的前缀](#9-需要谨慎省略的前缀)
  - [9.1 List / Container](#9-1-list-container)
  - [9.2 Image 子节点](#9-2-image-子节点)
  - [9.3 行为组件实例](#9-3-行为组件实例)
- [10. 对当前 Page 的整理建议](#10-对当前-page-的整理建议)
- [11. 最终判断原则](#11-最终判断原则)


## 1. 分析范围

本文分析 Figma 文件 公共组件库 new 的 `03-ToUGUI（common）` Page，重点说明：

- Component Set / Variant 如何转成一个可复用的 UGUI Prefab。
- Variant Toggle 与普通 Toggle 的处理差异。
- `ToggleGroup` 如何建立互斥关系。
- 哪些 Figma 节点可以省略 UGUI Role 前缀，哪些不能。


## 2. Page 的用途与主要结构

`03-ToUGUI（common）` 是面向 Common Prefab 导出的整理页，不是单纯的视觉展示页。当前可分为四类区域：

- `图片`：公共 Sprite 和准备烘图的视觉资源。
- `组件`：可复用的 Component、Component Set 和组合组件，是本文分析重点。
- `动效`：公共动效资源。
- `Welcome`：说明或封面内容，不属于主要 Common 组件导出结构。

页面中与 Variant / Toggle 相关的代表性节点包括：

| Figma 节点 | 状态轴 | 预期含义 |
| --- | --- | --- |
| `Toggle_yeqian` (`3700:765`) | `checked / unchecked` | 页签 Toggle |
| `Toggle_toggle` (`3795:2638`) | `yes / no` | 二态 Toggle |
| `toggle_denglu` (`4039:555`) | `checked / unchecked` | 登录 Toggle |
| `toggle_weitiao` (`5010:1112`) | `checked / unchecked` | 微调 Toggle |
| `Toggle_daojian` (`5254:1652`) | `checked / unchecked` | 刀剑样式 Toggle |
| `ToggleGroup_daojian` (`5254:1790`) | 组合结构 | 为多个 Toggle 建立互斥关系 |
| `lock1` (`3982:1848`) | `State=on / off` | 普通状态组件，不自动成为 Toggle |
| `Lunpan` (`3757:1174`) | `State=zhankai / shouqi` | 普通状态组件 |
| `jiangli` (`5698:1209`) | `lingqu / bukelingqu` | 普通状态组件 |

## 3. Component Set 的通用导出流程

Component Set 并不是把每个 Variant 分别导出为一个 Prefab。当前实现会把整个 Set 编译为一个带状态控制器的 Prefab：

```text
Figma COMPONENT_SET
  COMPONENT: Property 1=checked
  COMPONENT: Property 1=unchecked
            |
            v
一个 Unity Prefab
  FigmaVariantController
  Baseline 节点
  可合并的状态差异
  无法安全合并的 fallback branch
```

主要步骤如下：

1. `BuildVariantSets` 读取真实的 `COMPONENT_SET`。
2. 只把 Set 的直接 `COMPONENT` 子节点视为 Variant member。
3. 从 Figma Component Property 定义及 member 名称中解析属性和值。
4. 按文档中真实存在的状态组合建立签名，不补造不存在的笛卡尔积组合。
5. 选择与 Property 默认值匹配的 member 作为默认状态。
6. 以默认 member 为基线合并可安全对应的节点；不能安全合并的最小子树保留为 fallback branch。
7. 在 Prefab 根添加 `FigmaVariantController`。
8. Set id/key 及所有 member id/key 都映射到同一个 Prefab；member 通过 alias 带入对应状态值。

Controller 支持四种 Figma Component Property：

| Figma Property | Unity 中的处理 |
| --- | --- |
| `VARIANT` | 选择实际存在的 Variant Definition，并应用节点差异 |
| `BOOLEAN` | 控制绑定目标的显隐/状态 |
| `TEXT` | 修改绑定的 `TextMeshProUGUI` |
| `INSTANCE_SWAP` | 切换绑定的实例目标 |

非 `BOOLEAN`、`TEXT`、`INSTANCE_SWAP` 的类型按 `VARIANT` 处理。非 Variant 属性必须唯一解析到目标节点，否则会标记为未绑定并产生警告。

### 3.1 为什么只允许真实存在的组合

`FigmaVariantController.TrySetValues` 会先构造候选状态，再查找完整匹配的 Variant。只要 Set 中存在 Variant 属性，但候选组合在 Figma 中不存在，切换就会返回 `false`，不会臆造一个混合状态。

因此，多轴 Component Set 应在 Figma 中真正建立所有业务允许的组合；不希望出现的组合不要只依靠默认值掩盖。

### 3.2 Variant 可以改什么

Variant 可以改变 Prefab 内部的视觉、文本、节点激活状态、组件状态以及组件根的实际宽高。它不应覆盖业务实例放置时的外部布局意图，包括位置、Anchor、Pivot、Rotation 和 Scale。

这条边界保证同一个 Common Prefab 放在不同业务界面时，切换 Variant 不会把实例拉回组件定义页的位置。

## 4. Toggle 有两条不同的导出路径

### 4.1 普通 Toggle

普通 Toggle 是一个没有 `FigmaVariantController` 的 `Toggle_*` 节点：

```text
Toggle_music
  Image_bg
  Image_checkmark
  Label_text
```

其绑定逻辑为：

- `Image_bg`、`background`、`bg` 等 Graphic 作为 `Toggle.targetGraphic`。
- `checkmark`、`check`、`mark`、`on` 等 Graphic 作为 `Toggle.graphic`。
- 找不到背景时，在根上补可射线点击的空 Graphic。
- 找不到 checkmark 时仍保留 Toggle，但产生“缺少 Checkmark”警告。
- 视觉切换主要依赖 Unity Toggle 的 `graphic` 和 Transition。

所以普通 Toggle 即使根节点已有 `Toggle_` 前缀，内部背景和勾选图仍需要使用可识别名称。

### 4.2 Variant Toggle

`Toggle_yeqian`、`Toggle_toggle`、`toggle_denglu`、`toggle_weitiao`、`Toggle_daojian` 属于 Variant Toggle：Component Set 根先通过名称获得 `Toggle` Role，同时 Set 又被编译出 `FigmaVariantController`。

编译器只在以下条件成立时配置 `FigmaVariantToggleBinder`：

```text
Component Set 根 UguiRole == Toggle
```

Binder 会从 Controller 的所有 `VARIANT` 属性中寻找二态轴。目前只接受：

- `checked` 与 `unchecked`（也兼容 `uncheck`）；或
- `yes` 与 `no`。

必须恰好找到一个符合条件的属性轴：

- 找不到：不创建自动状态绑定，并输出警告。
- 找到多个：因含义不唯一，不创建自动状态绑定，并输出警告。
- 恰好一个：建立 `Toggle` 与 `FigmaVariantController` 的双向同步。

双向同步过程是：

```text
用户点击 Toggle
  -> Toggle.onValueChanged
  -> controller.TrySetValue(checked / unchecked)
  -> 应用对应 Variant

业务代码修改 Controller
  -> controller.StateChanged
  -> Binder 更新 Toggle.isOn
```

Variant Toggle 不使用传统 checkmark 控制整套视觉。Binder 会：

- 在根上建立 `FigmaHitArea` 作为 `targetGraphic`。
- 将 `Toggle.graphic` 设为 `null`。
- 由 `FigmaVariantController` 应用 checked/unchecked Variant 中的完整视觉差异。

因此，Variant member 可以整体更换背景、图标、文字或结构，而不必把所有差异压缩成一个 `Image_checkmark`。

### 4.3 状态名不是 Toggle Role

这是本页最重要的命名边界：

```text
checked / unchecked / yes / no
```

只负责在“已经是 Toggle 的 Component Set”中识别开关状态，不负责推断 UGUI Role。

例如 `lock1` 虽然有 `State=on/off`，但根没有 Toggle Role，因此只会得到普通 `FigmaVariantController`，不会自动添加 Unity `Toggle`。同理，普通 Set 即使改成 `checked/unchecked`，只要根没有被识别为 Toggle，也不会启用 `FigmaVariantToggleBinder`。

## 5. ToggleGroup 的处理

`ToggleGroup` 是独立的行为角色，不是由多个 Toggle 自动推断出来的。

页面中的 `ToggleGroup_daojian` 可理解为：

```text
ToggleGroup_daojian
  img_bg
  ToggleGroup
    Toggle_daojian Instance
    Toggle_daojian Instance
```

当某个节点获得 `ToggleGroup` Role 后，导出器会给它添加 Unity `ToggleGroup`，并把其后代所有 `Toggle.group` 指向该 Group。运行时某个 Toggle 被选中时，Unity ToggleGroup 会关闭同组其他 Toggle；其他 Toggle 各自的 Binder 再把自己的 Controller 切到 unchecked 状态。

因此：

- `ToggleGroup_*` 根或实际承担分组的中间节点不能省略 Role 名。
- 单纯在一个 `List` 或 `Container` 下放多个 Toggle，不会自动获得互斥关系。
- Group 可以位于组合组件内部，不一定非要与单个 Toggle 的 Component Set 合并。

## 6. 页面典型组合结构点评

### 6.1 `Toggle_yeqian`

推荐结构：

```text
Toggle_yeqian                 # Set 根：必须表达 Toggle Role
  Property 1=checked          # member：无需再写 Toggle 前缀
  Property 1=unchecked
```

Set 根的 Role 会传播到 Variant member 的组件引用映射中。因此 member 使用 Figma 自动生成的属性赋值名是正确做法，无需写成 `Toggle_checked`、`Toggle_unchecked`。

### 6.2 `BottomToggleS7`

其语义结构约为：

```text
BottomToggleS7
  List_toggle
    item
      Toggle_yeqian Instance
    ...
```

`BottomToggleS7` 不以 Toggle 开头，不会被误识别为 Toggle；它是普通组合容器。内部实例只有在用户提供的 Common 模块 URL 中动态解析到对应 Component identity 后，才能继承定义端的 Toggle 行为。

但当前结构只表达“List 中有多个 Toggle”，没有表达“这些 Toggle 属于同一个 ToggleGroup”。如果业务要求页签互斥，应在共同祖先处增加显式 `ToggleGroup_*` 节点或 Role，而不能依赖 `BottomToggleS7` 名称中的 `Toggle` 字样。

### 6.3 `ContainerToggleListS7`

`ContainerToggleListS7` 也不会因为名称中间包含 `Toggle` 而获得 Toggle Role。Role 的正则要求行为词位于名称开头，并满足分隔符、驼峰或数字边界；而 `prefix` 规则只检查第一个 `_`、`-` 或空格之前的完整 token。

因此，该名称不会命中 Toggle 或 ToggleGroup；如果它是普通 Frame/Group/Component，会再由节点类型回退为 `Container`。其内部 `List_toggle + item + Toggle Instance` 仍然不会自动建立互斥组。

## 7. 哪些前缀可以省略

这里的“可以省略”是指省略 UGUI Role 前缀后，当前代码仍能稳定推断目标结构；不是指任何节点都可以随意命名。

| 结构 | 能否省略 Role 前缀 | 原因或条件 |
| --- | --- | --- |
| Component Set 内的 Variant member | 可以 | Role 由 Set 根建立并传播；member 保留 `Property=value` 即可 |
| 普通纯状态 Component Set | 可以 | 只需要 `FigmaVariantController`，由业务主动切换状态，不要求 Unity Button/Toggle 行为 |
| Figma `INSTANCE` | 通常可以 | 节点类型可自动推断为 `Reference`，并通过 Component id/key 查 Common Prefab |
| Figma Text | 可以 | 节点类型自动推断为 `Label` |
| Rectangle / Image | 通常可以 | 节点类型自动推断为 `Image`；仍建议关键绑定节点显式命名 |
| 普通 Frame / Group / Component | 可以 | 默认推断为 `Container` |
| `界面/Views` 下的顶层 Frame | 可以 | 层级推断为 `UIRoot` |
| Auto Layout Frame | 技术上可以 | 会自动推断为 `List`，但存在误判风险 |

可不加行为前缀的页面实例包括 `lock1`、`Lunpan`、`jiangli`：前提是业务只通过 `FigmaVariantController` 切换状态，不需要用户点击后自动触发 Unity `Toggle` 或 `Button`。

## 8. 哪些前缀不能省略

| 结构 | 不能省略的原因 |
| --- | --- |
| 要生成 Unity Toggle 的 Component Set 根 | Variant 状态名不推断 Toggle；根必须匹配 `Toggle` Role |
| 要生成 Unity ToggleGroup 的节点 | 不会根据后代 Toggle 自动推断分组行为 |
| Button、Input、ProgressBar、Slider、ScrollBar 等行为根 | 节点形状和子节点不足以可靠推断交互组件 |
| `NotExport`、`UiMask` 等特殊导出语义 | 这些是导出指令，必须显式表达 |
| 普通 Toggle 的背景和 checkmark Graphic | 需要可识别名称来绑定 `targetGraphic` 和 `graphic` |

推荐使用明确形式：

```text
Toggle_xxx
ToggleGroup_xxx
Button_xxx
Input_xxx
ProgressBar_xxx
Slider_xxx
ScrollBar_xxx
NotExport_xxx
UiMask_xxx
```

名称匹配不要求只能使用下划线。当前规则也支持连字符、空格、驼峰和数字边界，例如 `toggleMusic`、`Toggle-Music`、`Toggle2`。不过团队文档建议统一使用 `Role_businessName`，这样最易阅读和排查。

## 9. 需要谨慎省略的前缀

### 9.1 List / Container

Auto Layout Frame 会被自动推断成 List，这对滚动列表方便，但也会把普通横排、竖排布局容器误判为 List。

建议：

- 真正的运行时列表或 ScrollView 使用 `List_*` / `ScrollView_*`。
- 仅用于静态排版的 Auto Layout 使用 `Container_*`。

### 9.2 Image 子节点

Rectangle 虽可自动成为 Image，但对 Toggle、Button、Slider 等控件，代码还会按名称查找背景、checkmark、fill、handle 等具体职责。关键 Graphic 应使用明确名称，不能只依靠节点类型。

### 9.3 行为组件实例

真实 Figma Instance 一般可以省略实例自身的 `Toggle_` 前缀，因为从用户提供的 Common 模块 URL 动态建立的当次索引能依据 Component id/key 传播定义端的 Button/Toggle/ToggleGroup 行为。前提是：

- 定义端 Component 或 Component Set 已正确获得行为 Role；
- 实例仍保留真实组件身份，没有被 detach；
- 当次动态 Common 索引能解析到对应组件。

若定义端本身未标 Role，实例名中的 `checked` 或 `on` 也不能补救。

## 10. 对当前 Page 的整理建议

1. 保留 `Toggle_yeqian`、`Toggle_toggle`、`toggle_denglu`、`toggle_weitiao`、`Toggle_daojian` 的 Toggle 根命名。
2. Variant member 保持 Figma 的 `Property=value` 命名，不重复添加 `Toggle_`。
3. 将所有需要自动绑定的 Toggle 状态轴统一为 `checked/unchecked`；`yes/no` 虽受支持，但统一术语更利于排查。
4. 确保每个 Variant Toggle 只有一个 checked/unchecked 型 Variant 轴；其他属性可以是 Boolean、Text、Instance Swap，或使用不会被误认作 Toggle 状态的 Variant 值。
5. 明确检查 `BottomToggleS7` 和 `ContainerToggleListS7` 是否要求互斥选择；若要求，补充真正的 `ToggleGroup_*` 祖先节点。
6. `lock1`、`Lunpan`、`jiangli` 若只是展示状态组件，可保持普通名称；若预期用户点击即切换，则应明确选择 Button 或 Toggle 行为并修改根命名。
7. 对普通非 Variant Toggle，保留明确的 `Image_bg` 和 `Image_checkmark`；不要把 Variant Toggle 的规则直接套到普通 Toggle 上。
8. 对静态 Auto Layout 组合组件显式使用 `Container_*`，避免被自动推断为 List。

## 11. 最终判断原则

判断一个前缀能否省略时，可按以下顺序检查：

```text
这个节点是否需要 Unity 行为组件？
  |
  +-- 是：Button / Toggle / ToggleGroup / Input / Slider ...
  |      -> 根节点显式写 Role，不能靠 Variant 值或子节点猜测
  |
  +-- 否：只需要容器、图片、文本、引用或普通 Variant Controller
         -> 可利用节点类型和层级推断，必要时显式命名以避免误判
```

一句话总结：**Variant 描述“有哪些状态”，Role 前缀描述“这个对象在 Unity 中是什么控件”。只有状态、没有 Role，不会自动产生 Toggle 行为。**
