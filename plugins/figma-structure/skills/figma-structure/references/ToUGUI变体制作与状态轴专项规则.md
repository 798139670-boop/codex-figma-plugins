# ToUGUI 变体制作与状态轴专项规则

## 1. 规则定位与观察边界

本文件补充 `公共组件库new-Variant与Toggle导出规则分析.md`，把已读取界面中的变体做法整理成可复用的制作思路。它只追加制作和审计规则，不替代 Component Set、Toggle、ToggleGroup、Common、布局或 Export 原规则。

既有只读批量审计覆盖 145 个 `界面` 直属根和 27,141 个节点；其中记录到 3,024 条带 source 身份的 Instance 记录、1,320 个 source 身份、551 个不同的 source/属性字符串。观察到的轴包括 `Property 1=checked/unchecked`、`Property 1=checked, Property 2=green`、`State=off`、`Property 1=CurrencyItem`、颜色、品质、资源类型等。这些是样本证据，不是要求所有组件都复制这些名称或状态。

Figma 只读时，只读取 Component Set、直接 Component member、Property 定义、实例属性、结构和布局；不得为了学习切换 Variant、调整实例属性、改尺寸、改约束或写入 Figma。当前数据可支持静态推演和规则学习，不能据此声称 Unity 运行时切换已经通过。

## 2. 变体的基本制作模型

推荐先建立一个真实 `COMPONENT_SET`，再在 Set 内建立直接的 `COMPONENT` members：

```text
Component Set Toggle_Profile
├─ Component State=checked
└─ Component State=unchecked
```

制作和导出按以下边界执行：

1. Component Set 是一个可复用 Prefab/Controller 单元；直接 member 是状态定义，不分别当作同一组件的多个独立 Prefab。
2. 只把 Set 的直接 `COMPONENT` 子节点视为正式 Variant member；普通 Frame、Instance、嵌套 Component 或说明节点不能凭位置冒充 member。
3. 从 Figma Component Property 定义和 member 名称解析 `Property=Value`。保留真实 property、value、member ID/key 和组合关系；不要只看可见文字或截图推断状态。
4. Set 根承担组件的业务语义和必要 UGUI Role。member 名称只表达属性和值，不能在 member 上重复添加 `Toggle_`、`Button_` 等 Role 前缀。
5. 一个 Set 编译成一个 `FigmaVariantController` 及其状态差异。以默认 Property 值对应的 member 为 baseline，能稳定对应的节点合并；无法安全对应的最小差异子树保留 fallback branch。
6. `VARIANT`、`BOOLEAN`、`TEXT`、`INSTANCE_SWAP` 属性分别对应状态选择、显隐/布尔状态、文本绑定和实例替换；其他属性类型按 Variant 语义读取。每个非 Variant 属性必须唯一绑定到目标，否则报告未绑定。

## 3. 状态轴如何设计

### 3.1 一个轴只表示一个业务维度

为每个真实业务维度建立一个语义 property，例如：

```text
State=checked / unchecked
Theme=green / yellow
Size=small / large
Mode=online / offline
Quality=common / rare / epic
```

Property 名和值必须使用英文语义名；显示给玩家的中文仍放在 Text `characters` 或本地化数据中。不要用 `Property 1`、`Property 2`、`Variant2` 作为新组件的最终语义命名；遇到历史文件中的这类名称，记录其原始值，并在授权命名整理时转换为稳定英文业务名，同时保持 `Property=Value` 语法和引用一致。

### 3.2 只建立真实存在的组合

多轴 Set 只建立业务允许的组合，不自动补全笛卡尔积：

```text
允许：State=checked + Theme=green
允许：State=unchecked + Theme=green
允许：State=checked + Theme=yellow
不自动假设：State=unchecked + Theme=yellow
```

如果运行时请求 Figma 中不存在的完整组合，Controller 应返回失败并报告；不能用默认值、截图或临时拼接两个 member 冒充不存在的 Variant。制作时应明确列出允许组合、默认组合和缺失组合。

### 3.3 默认状态与 baseline

- Property 默认值必须对应一个真实 member；没有默认对应关系时列入 `needsReview`。
- 选默认 member 作为 baseline，先合并所有状态都拥有且职责一致的节点，再记录只在特定状态出现的节点。
- 如果同名节点在不同 member 中几何、类型、绑定职责或层级不同，不能强行合并；保留最小 fallback branch 并报告差异。
- 需要状态逻辑的 Component Set 不应把状态差异烘成单张 PNG；静态视觉源可以在各 member 内分别提供，但 member 根、Controller 和动态职责必须保留。

## 4. 变体的结构与布局制作

### 4.1 默认保持同骨架

同一 Set 的 members 默认保持一致的层级骨架和职责名：根、背景、图标、Label、HitArea、内容插槽、Mask/Viewport 等应能按稳定路径对应。允许改变的是状态所需的视觉、文字绑定、显隐、实例替换和组件状态；不应因为只是换状态而改变无关的父子关系。

### 4.2 每个状态都要核对布局

逐个 member 读取并比较：

- 根宽高、Constraints、Auto Layout、FIXED/HUG/FILL、padding、spacing、alignment；
- 文本 Rect、基线、`textAutoResize`、换行和截断；
- 背景、图标、勾选图、动态插槽、HitArea 的相对位置和尺寸；
- Clip/Mask、越界装饰、旋转/变换和 `@size`；
- Export、Instance source component ID/key 和可见性。

状态根尺寸、布局模式或文本基线不一致时，不能只用当前展示状态推断其它状态自适应。若尺寸差异是业务设计的一部分，要记录它是“组件根内部状态差异”；不要把状态切换导致的差异误写成外部实例的位置、Anchor、Pivot、Rotation 或 Scale。

### 4.3 运行时外部布局边界

Variant 可以改变 Prefab 内部视觉、文本、节点激活、组件状态和组件根实际宽高；不能覆盖业务实例放置时的外部位置、Anchor、Pivot、Rotation、Scale。这样同一 Common Prefab 在不同界面切换状态时不会被拉回组件定义页的位置。

`@size` 只说明实例尺寸参数，不证明任意状态都可安全拉伸。若每个状态需要不同外部尺寸或完全不同布局，优先拆成语义不同的组件或明确记录尺寸策略，不把多个不相关结构强塞进一个 Set。

## 5. Variant Toggle 与普通 Toggle

### 5.1 Variant Toggle

Variant Toggle 必须同时满足：

1. Component Set 根有明确英文 `Toggle_*` Role；
2. 所有 VARIANT 属性中恰好一个轴包含 `checked/unchecked`（兼容 `uncheck`）或 `yes/no`；
3. 该轴的两种值都映射到真实 member。

零个候选轴或多个候选轴都不能自动绑定，必须报告歧义。状态名本身不能推断 Toggle Role；普通 `State=on/off` Set 只产生 Variant Controller，不自动变成 Unity Toggle。

Variant Toggle 的完整视觉由 Controller 切换：根建立可点击的 `FigmaHitArea` 作为 `targetGraphic`，`Toggle.graphic` 保持为空；member 可以整体替换背景、图标、文字或结构。不要再套用普通 Toggle 的 checkmark 方案来覆盖整个状态差异。

### 5.2 普通 Toggle

没有 Variant Controller 的 `Toggle_*` 使用普通路径：保留可识别的 `Image_bg`/`background` 和 `Image_checkmark`/`checkmark` 等 Graphic，分别绑定 `targetGraphic` 和 `graphic`。不能因为普通 Toggle 名称包含 `checked` 或 `on`，就把它当成 Variant Toggle。

### 5.3 ToggleGroup

只有真实互斥选择才建立显式 `ToggleGroup_*`。多个 Toggle 放在 List、Container、Auto Layout 或 ScrollView 下不会自动互斥；列表语义、点击语义和互斥分组是三个独立判断。若业务要求互斥，Group 需要位于能统一绑定后代 Toggle 的共同祖先，并验证实例来源确实解析到定义端 Role。

## 6. 变体与 Common/Instance、Export 的关系

- 真实 Instance 的 source component ID/key 是身份唯一证据。不能用同名、同尺寸、视觉相似或 `@size` 推定它属于某个 Variant/Common。
- 保留 Instance 身份和 Component Set 关系；不能因为要学习或统一状态而 Detach、替换、迁移到 Common，来源不可读或身份歧义时列 `unresolved`。
- Component Set 根、Variant member 根、Toggle/行为根、动态 Text、Input、ProgressBar、Slider、ScrollBar、List/ScrollView 和运行时插槽不作为普通 PNG 根。
- 只对每个状态内能独立复现的静态视觉源处理 Export；不能把动态文本、数量、进度、输入、HitArea、Mask/Viewport 或整套状态逻辑烘进单张图。
- 状态专用视觉源可带合法 PNG，但父子重复、祖先 PNG 覆盖、NotExport 备份和 NineSlice 仍按既有 Export 分类处理。

## 7. 英文命名与属性保真

所有 layer name、Component Set/member name、property name/value、Variant member 和导出后缀都使用英文、数字或下划线，遵循入口中的“中文层名统一改为英文”优先规则。Text `characters` 可以保留中文。

改名时：

- 保留 `Property=Value` 的语法，不把 member 改成普通业务句子而丢失状态；
- 保留 Role、`@size`、Variant 属性键和值之间的引用关系；
- 不在 member 重复 `Toggle_`/`Button_` 前缀；
- 检查同一 Set 内 property/value 唯一性、同级名称、Common canonical name、Prefab/Sprite 名称和实例 override；
- Component 定义在范围外时不直接改远程主组件，不能用 Detach 掩盖命名阻塞。

## 8. 只读审计与制作报告

每个 Component Set 至少报告：

```text
setId / setName / setRole
members: [{memberId, memberName, properties, sourceKey, width, height}]
propertyAxes: [{name, type, values, defaultValue, realCombinations}]
baselineMember / fallbackBranches / missingCombinations
sameSkeleton / layoutDifferences / textDifferences / visibilityDifferences
toggleAxis / toggleBinding / toggleGroupEvidence
instanceIdentity / exportSources / unresolved
```

审计结果区分：`validVariantSet`、`validVariantToggle`、`ordinaryVariantController`、`needsReview`、`missingCombination`、`unresolvedIdentity`。完整读取、静态布局推演和 Unity 实际切换验证分别报告；只完成前两项时，不得声称运行时切换通过。

## 9. 与现有规则的关系

本文件只追加变体制作思路和审计字段。Component Set 的既有导出流程、Toggle/ToggleGroup 识别、Common 动态匹配、布局适配、Export 和英文命名规则继续有效；冲突时遵循用户当前范围、入口优先规则和更具体的既有安全约束。
